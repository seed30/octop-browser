---
kind: api-contract
scope: python-sdk
status: draft
origin: generated
source_paths: [src/octop_browser/__init__.py, src/octop_browser/session.py, src/octop_browser/tool_interface.py, src/octop_browser/install.py, src/octop_browser/models.py]
symbols: [BrowserSession, browser_tool, OctopSettings, ensure_chromium, install_chromium_stream]
upstream: [contracts/api-reference.md, architecture/api-and-mcp.md]
downstream: [business/domain-models/toolresult-and-settings.md, contracts/error-codes.md]
---

# Python SDK 入向契约

包根导出 12 个符号（`src/octop_browser/__init__.py:15-28`），全部可从 `from octop_browser import ...` 获取。SDK 是异步优先接口，所有浏览器动作返回 `ToolResult`，不会抛业务错误（参数错误也被包装成 `success=False`）。

## 1. 包根导出与签名字典

| 符号 | 类型 | 签名 / 形态 | 定义锚点 |
|---|---|---|---|
| `BrowserSession` | class | `await BrowserSession.create(profile, headless, mode, urls, profile_manager, settings)` | `src/octop_browser/session.py:174`、`:199` |
| `browser_tool` | async fn | `browser_tool(action, profile="default", headless=None, mode=None, urls=None, **kwargs) -> ToolResult` | `src/octop_browser/tool_interface.py:91` |
| `BrowserMode` | `Literal` | `"auto" \| "headed" \| "headless"` | `src/octop_browser/mode.py:21` |
| `OctopSettings` | dataclass | 13 个字段，见 [配置模型](../../business/domain-models/toolresult-and-settings.md) | `src/octop_browser/settings.py:110` |
| `settings` | 实例 | 导入时读取环境的默认配置单例 | `src/octop_browser/settings.py:185` |
| `ToolResult` | Pydantic | `success`/`content`/`error`/`metrics`/`metadata` | `src/octop_browser/models.py:20` |
| `ActionMetrics` | Pydantic | `action`/`duration_ms`/`dom_nodes_scanned`/`estimated_tokens`/`screenshot_size_kb` | `src/octop_browser/models.py:10` |
| `TabInfo` | Pydantic | `tab_id`/`url`/`title`/`active` | `src/octop_browser/models.py:36` |
| `InstallEvent` | TypedDict | `log`/`done`/`success`/`error` | `src/octop_browser/install.py:81` |
| `chromium_executable` | fn | `() -> str \| None` | `src/octop_browser/install.py:94` |
| `ensure_chromium` | fn | `(quiet=True) -> bool` | `src/octop_browser/install.py:280` |
| `install_chromium_stream` | async gen | `(is_disconnected=None) -> AsyncGenerator[InstallEvent]` | `src/octop_browser/install.py:486` |

## 2. `BrowserSession` 动作方法全集（20 个）

| 方法 | 关键参数（`session.py` 行号） | 返回特征 |
|---|---|---|
| `navigate` | `url: str` (`:262`) | 清 ref；等 load 5s |
| `go_back` / `go_forward` | 无 (`:272`/`:280`) | `history.back()/forward()` + 清 ref |
| `reload` | 无 (`:288`) | `Page.reload({ignoreCache:False})` + 清 ref |
| `dom_tree` | `level="interactive"` (`:300`) | `content` 为文本或 `structured` JSON |
| `screenshot` | `crop=False`、`element_ref=None`、`full_page=False`、`path=None` (`:348`) | `content` 为 PNG 路径，metadata 带尺寸 |
| `click` | `ref`、`selector`、`x`、`y` 四选一 (`:377`) | 两次 `Input.dispatchMouseEvent` |
| `type` | `text`、`ref`、`selector` (`:396`) | 可选先点击再逐字符键入 |
| `fill` | `text`、`ref`、`selector` (`:416`) | Ctrl/Meta+A + Backspace 后键入 |
| `press` | `key: str` (`:436`) | `keyDown` + `keyUp` |
| `select` | `value`、`ref`、`selector` (`:442`) | `Runtime.callFunctionOn` 派发 input/change |
| `scroll` | `direction="down"`、`amount=300` (`:462`) | 滚轮事件 |
| `hover` | `ref: str` (`:473`) | `mouseMoved` |
| `eval_js` | `expression: str` (`:485`) | `awaitPromise=True`，结果 JSON 字符串 |
| `wait` | `url_contains`、`text`、`selector`、`timeout_ms=10000` (`:491`) | 0.25s 轮询 |
| `new_tab` | `url=None`、`force_new=False` (`:563`) | 空标签复用；metadata 带 `target_id` |
| `switch_tab` | `tab_id: str` (`:735`) | 重连页面级 CDP 并写粘滞目标 |
| `close_tab` | `tab_id=None` (`:766`) | 无 id 时返回 `No tab_id supplied; nothing to close.` |
| `list_tabs` | 无 (`:799`) | content 文本 + `metadata["tabs"]` |
| 生命周期 | `metrics_summary()` (`:875`)、`close(kill=False)` (`:887`) | 非动作方法，不触发钩子 |

`__aenter__`/`__aexit__` 支持 `async with await BrowserSession.create(...) as session`，退出时调用 `close()`（不杀进程）(`src/octop_browser/session.py:252-256`)。

## 3. `browser_tool` 调用契约与重试

| 阶段 | 规则 | 锚点 |
|---|---|---|
| 动作白名单 | 20 个合法值，非法直接 `_error_result()` | `src/octop_browser/tool_interface.py:20-43`、`:120-123` |
| 会话缓存 | `profile` 命中 `_registry` 则复用；首次调用才应用 `headless`/`mode`/`urls` | `src/octop_browser/tool_interface.py:57-68` |
| kwargs 分发 | `getattr(sess, action)(**kwargs)` | `src/octop_browser/tool_interface.py:180-181` |
| 多标签补偿 | 已存在会话时按 `urls` 顺序 navigate + new_tab(force_new=True) | `src/octop_browser/tool_interface.py:172-177` |
| 断线重试 | 检测 `not connected`/`connection lost` → `_drop_session()` → 原参数重试一次 | `src/octop_browser/tool_interface.py:83-88`、`:186-210` |
| 全局钩子槽 | `_global_hooks: dict[str, list[Any]]` 已声明但当前无读写路径 | `src/octop_browser/tool_interface.py:18` |

`_global_hooks` 是存量未挂载变量：声明后没有任何代码向其中注册或触发回调，不要据此实现全局钩子。

## 4. 安装类 API 与结果形态

`ensure_chromium()` 同步返回 `bool`，适合脚本；`install_chromium_stream()` 逐行 `yield` `InstallEvent`，终态事件为 `{"done": True, "success": bool, "error"?: str}` (`src/octop_browser/install.py:486-620`)。两者共享 `verify_chromium()` 作为最终门禁。`chromium_executable()` 是 `_find_playwright_chromium()` 的薄封装 (`src/octop_browser/install.py:94-100`)。

## 5. 典型调用与共变清单

```python
async with await BrowserSession.create(profile="work") as session:
    await session.navigate("https://example.com")
    tree = await session.dom_tree(level="interactive")
    result = await session.click(ref="btn_1")
    print(result.success, result.metrics.estimated_tokens)
```

- 修改任一方法签名：同步 `src/octop_browser/tool_interface.py:20` 白名单、MCP `inputSchema`、CLI `_action_kwargs()`、本页方法表与 `tests/unit/test_session.py`。
- 新增导出符号：同步 `src/octop_browser/__init__.py:15` `__all__` 与两份 Skill 文件。
- 必跑：`uv run pytest tests/unit/test_models.py tests/unit/test_tool_interface.py tests/unit/test_navigate_actions.py -q`。

- 关联阅读：[会话核心引擎](../../architecture/session-core.md)、[CLI 命令全集](cli-commands.md)。
