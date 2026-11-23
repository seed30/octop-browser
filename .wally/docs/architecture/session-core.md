---
kind: component
scope: session-core
status: draft
origin: generated
source_paths: [src/octop_browser/session.py, src/octop_browser/actions/navigate.py, src/octop_browser/actions/interact.py, src/octop_browser/actions/capture.py, src/octop_browser/actions/js_eval.py, src/octop_browser/dom/builder.py, src/octop_browser/dom/refs.py, src/octop_browser/hooks.py]
symbols: [BrowserSession, _InternalCDPSession, DOMBuilder, RefCache, HooksMixin]
upstream: [architecture/index.md, architecture/api-and-mcp.md]
downstream: [architecture/cdp-and-launcher.md, business/wf-browser-action.md, contracts/api/python-sdk.md]
---

# 组件架构：会话核心与 DOM 引擎

本组件是全部浏览器动作的编排中枢：`BrowserSession` 把 20 个动作统一包成「钩子 → 底层动作 → 计数」三段式，DOM 引擎把 CDP 原始树折叠成四档 token 视图。底层连接与进程管理交给 [CDP 与启动器](cdp-and-launcher.md)。

## 1. 分层职责与不变量

| 层 | 源码 | 职责 | 修改红线 |
|---|---|---|---|
| 公共门面 | `src/octop_browser/session.py:174` | 动作方法、指标汇总、生命周期 | 禁止在此拼 CDP 指令字典 |
| 内部会话 | `src/octop_browser/session.py:33` | 持一个 `CDPClient` 与一个 `RefCache`，负责连接/切目标/视口 | 禁止把 `cdp_host` 写死为 `localhost` |
| 动作叶子 | `src/octop_browser/actions/navigate.py:25` 等 | 把动作翻译为具体 CDP 指令 | 禁止跳过 `ref_cache.invalidate()` |
| DOM 构建 | `src/octop_browser/dom/builder.py:137` | 四级视图与 ref 发放 | 禁止修改 `len(content)//4` 的 token 估算口径 |
| 事件钩子 | `src/octop_browser/hooks.py:19` | `on()` 注册与 `_fire()` 异常隔离 | 钩子异常只能记日志，不得中断动作 |

## 2. 会话装配调用树

```text
BrowserSession.create() src/octop_browser/session.py:199
  ├─ cfg = settings or _default_settings :226；pm = profile_manager or _default_profile_manager :227
  ├─ p = pm.get_or_create(profile) :228 -> src/octop_browser/profile.py:165
  ├─ [模式仲裁] :231 headless 显式值优先 > mode 显式值 > cfg.browser_mode :234
  ├─ internal = _InternalCDPSession(p, cfg) :236 -> idle_controller_for() :39
  ├─ internal.connect(headless=is_headless) :237 -> src/octop_browser/session.py:49
  │    ├─ [直连分支] cfg.cdp_ws_url 存在 :55 -> 直接 CDPClient.connect()
  │    ├─ [启动分支] launch_or_attach() :59 -> get_page_ws_url() :60
  │    ├─ enable_domain("Page"/"DOM"/"Runtime") :64-66
  │    ├─ _apply_viewport() :69 -> Emulation.setDeviceMetricsOverride :86
  │    └─ idle_controller.start() :72
  └─ [urls] 首个 navigate，其余 new_tab(force_new=True) :241-248
```

## 3. 动作分发与 ref 失效铁律

```text
BrowserSession.<action>() -> _fire("before_action", {...}) -> actions.<fn>() -> _record(result)
  ├─ navigate :262 -> src/octop_browser/actions/navigate.py:25 ref_cache.invalidate() :28 -> Page.navigate :37 -> Page.loadEventFired 等 5s :47
  ├─ go_back :272 / go_forward :280 -> history.back()/forward() + invalidate
  ├─ reload :288 -> Page.reload({ignoreCache:False}) + invalidate
  ├─ dom_tree :300 -> DOM.getDocument({depth:-1}) :306 -> DOMBuilder.build(level) :326
  ├─ screenshot :348 -> src/octop_browser/actions/capture.py:87 -> Page.captureScreenshot :170 -> 写 PNG :187
  ├─ click :377 -> src/octop_browser/actions/interact.py:63 -> _resolve_coords() :32 -> Input.dispatchMouseEvent ×2 :74
  ├─ type :396 / fill :416 -> 先 click 再逐字符 Input.dispatchKeyEvent :104/:198
  ├─ select :442 -> DOM.resolveNode + Runtime.callFunctionOn 派发 input/change :249-257
  ├─ scroll :462 -> Input.dispatchMouseEvent(mouseWheel) :124
  ├─ hover :473 -> Input.dispatchMouseEvent(mouseMoved) :152
  ├─ eval_js :485 -> Runtime.evaluate(awaitPromise=True) :19
  └─ wait :491 -> 轮询 Runtime.evaluate，每 0.25s 一次，默认 10s 截止 :500-544
```

`RefCache.invalidate()` 的调用点是导航类动作的强制不变量：新增任何跨文档跳转动作都必须在发指令前清空缓存，否则会把旧文档的 `nodeId` 交给新页面。`switch_to_target()` 内部重建 `CDPClient` 后同样调用 `invalidate()` (`src/octop_browser/session.py:141`)。

## 4. DOM 四级视图与字段字典

| 级别 | 构造入口 | 输出结构与 token 口径 |
|---|---|---|
| `minimal` | `src/octop_browser/dom/builder.py:298` | 两行文本：URL、标题、交互元素计数与 `Scroll: yes` |
| `interactive` | `src/octop_browser/dom/builder.py:304` | 按标签分别渲染 `input`/`button`/`a`/`select`/`textarea`，每行带 `[ref=...]` |
| `full` | `src/octop_browser/dom/builder.py:337` | 结构标签行 + 交互元素行，`_STRUCTURAL_TAGS` 共 16 个 |
| `structured` | `src/octop_browser/dom/builder.py:359` | JSON：`{url, title, elements[]}`，元素含 `ref`/`role`/`node_id`/`selectorCandidates`/`framePath`/`shadowPath` |

交互判定来自 `_INTERACTIVE_TAGS`（7 个标签）与 `_INTERACTIVE_ROLES`（11 个 role），`a` 标签无 `href` 时强制降为非交互 (`src/octop_browser/dom/builder.py:225-226`)。`_SKIP_TAGS` 列出 10 个跳过标签但仍递归 `#document` (`src/octop_browser/dom/builder.py:203-215`)。iframe 与 Shadow DOM 通过 `framePath`/`shadowPath` 记录上下文 (`src/octop_browser/dom/builder.py:274-296`)。

## 5. 钩子、指标与共变清单

| 事件 | 触发点 | payload |
|---|---|---|
| `before_action` | 各动作方法首行，如 `src/octop_browser/session.py:263` | `{"action": str, "params": dict}` |
| `after_action` | `_record()` 成功分支 `src/octop_browser/session.py:903` | `ActionMetrics` |
| `action_error` | `_record()` 失败分支 `src/octop_browser/session.py:901` | `ToolResult` |
| `page_navigated` | `src/octop_browser/hooks.py:33` 注明保留但未发射 | 无 |

- **共变清单**：改 DOM 字段 → 同步 `src/octop_browser/dom/builder.py:359` 的 JSON 键、`src/octop_browser/record/matcher.py:19` 的匹配字段、`src/octop_browser/record/semantic.py:44` 的压缩字段、两份 Skill 文案。
- **共变清单**：改动作参数 → 同步 `src/octop_browser/tool_interface.py:20` 白名单、`src/octop_browser/mcp_server.py:24` Schema、[CLI 翻译表](cli.md)、`tests/unit/test_navigate_actions.py` 与 `tests/unit/test_dom_builder.py`。
- **必跑**：`uv run pytest tests/unit/test_hooks.py tests/unit/test_refs.py tests/unit/test_dom_builder.py tests/unit/test_navigate_actions.py -q`。
