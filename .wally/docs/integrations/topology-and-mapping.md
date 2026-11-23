---
kind: mapping
scope: inbound-to-downstream-mapping
status: draft
origin: generated
source_paths: [src/octop_browser/cli.py, src/octop_browser/mcp_server.py, src/octop_browser/tool_interface.py, src/octop_browser/session.py, src/octop_browser/actions/interact.py, src/octop_browser/actions/capture.py, src/octop_browser/cdp/client.py, src/octop_browser/record/replay.py]
symbols: [_action_kwargs, call_tool, browser_tool, _resolve_coords, CDPClient.send, ReplayRunner.run]
upstream: [integrations/index.md, INDEX.md]
downstream: [integrations/downstream/chrome-cdp.md, integrations/downstream/record-daemon.md, contracts/api-reference.md]
---

# 上游入参 DTO → 中间转换器 → 下游请求字典键映射

本页展开「三明治夹心层」：最上层是 CLI/MCP 的入参字典，中间是 `tool_interface`/`session`/`actions` 的转换，最下层是发往 Chrome 的 CDP 字典。所有 Key 以源码字面为准。

## 1. 三段式映射总表

| 上游 DTO Key | 中间转换函数 | 中间形态 | 下游字典键路径 | 锚点链 |
|---|---|---|---|---|
| CLI `args.level` | `_action_kwargs()` (`src/octop_browser/cli.py:435-436`) | `kwargs["level"]` | DOMBuilder 的 `level` 参数 → `DOM.getDocument({depth:-1})` | `src/octop_browser/cli.py:436` → `src/octop_browser/session.py:306` |
| CLI `args.xy` | `_action_kwargs()` (`:449-450`) | `kwargs["x"]`,`kwargs["y"]` | `Input.dispatchMouseEvent({x,y})` | `src/octop_browser/cli.py:450` → `src/octop_browser/actions/interact.py:75-83` |
| CLI `args.ref` | `_action_kwargs()` (`:445-446`) | `kwargs["ref"]` | `RefCache.lookup(ref)` → `DOM.getBoxModel({nodeId})` → `{x,y}` | `src/octop_browser/cli.py:446` → `src/octop_browser/actions/interact.py:42-45` |
| CLI `args.text` | `_action_kwargs()` (`:452`、`:457`、`:463`) | `kwargs["text"]` | `Input.dispatchKeyEvent({type:"char", text})` | `src/octop_browser/cli.py:452` → `src/octop_browser/actions/interact.py:104`、`:198` |
| CLI `args.timeout_ms` | `_action_kwargs()` (`:472`) | `kwargs["timeout_ms"]` | `wait()` 的 deadline 与 `asyncio.sleep(0.25)` | `src/octop_browser/cli.py:472` → `src/octop_browser/session.py:500-544` |
| MCP `arguments["element_ref"]` | 原样透传 (`:475`) | `kwargs["element_ref"]` | `DOM.getBoxModel({nodeId})` → `Page.captureScreenshot({clip})` | `src/octop_browser/mcp_server.py:475` → `src/octop_browser/actions/capture.py:130-149` |
| MCP `arguments["full_page"]` | 原样透传 | `kwargs["full_page"]` | `Page.getLayoutMetrics({})` → `clip` + `captureBeyondViewport=True` | `src/octop_browser/actions/capture.py:150-168` |
| MCP `arguments["path"]` | 原样透传 | `kwargs["path"]` | `_resolve_path()` → 绝对路径原样/相对路径落 `screenshots_dir` | `src/octop_browser/actions/capture.py:46-59` |
| MCP `arguments["inputs"]` | `arguments.get("inputs") or {}` (`:455`) | `inputs` dict | `ReplayRunner._resolve_value()` 替换 `{{key}}` | `src/octop_browser/mcp_server.py:455` → `src/octop_browser/record/replay.py:566-570` |
| MCP `arguments["mode"]`（browser_open） | 显式取 `get("mode")` (`:466`) | `mode` 参数 | `BrowserSession.create()` 的模式仲裁 | `src/octop_browser/mcp_server.py:466-472` → `src/octop_browser/session.py:231-235` |
| SDK `profile` | `_get_or_create_session()` (`:57-68`) | `_registry[profile]` 键 | `ProfileManager.get_or_create(name)` → `data_dir`/`cdp_port` | `src/octop_browser/tool_interface.py:63-65` → `src/octop_browser/profile.py:165-174` |
| SDK `kill=True`（close_session） | `kwargs.pop("kill")` (`:126`) | 布尔 | `Browser.close` + `pkill` + 清锁 | `src/octop_browser/tool_interface.py:135-151` → `src/octop_browser/cdp/launcher.py:469-494` |

## 2. 点击定位的三分支转换字典

`interact._resolve_coords()` 是最高频的转换点，按 `ref → selector → (x,y)` 顺序短路 (`src/octop_browser/actions/interact.py:32-60`)：

```text
[ref]     ref -> RefCache.lookup() -> nodeId -> DOM.getBoxModel({nodeId}) -> model.content[8点] -> cx=(p0+p4)/2, cy=(p1+p5)/2
[selector] selector -> DOM.getDocument({depth:0}) -> root.nodeId -> DOM.querySelector({nodeId, selector}) -> nodeId -> 同上取中心
[(x,y)]  直接 float(x), float(y)
[全无]   raise ValueError("Must provide ref, selector, or (x, y)")
```

nodeId=0 时抛 `Selector '{selector}' matched no elements.`；ref 未命中时抛 `Ref '{ref}' not found. Call dom_tree() first.`。

## 3. 各动作的完整下游字典键（逐条展开）

| 动作 | 下游 method | params 字典键（含条件键） | 返回解包键 | 锚点 |
|---|---|---|---|---|
| `navigate` | `Page.navigate` | `url` | `errorText`、`frameId` | `src/octop_browser/actions/navigate.py:37-53` |
| `go_back` / `go_forward` | `Runtime.evaluate` | `expression="history.back()"/"history.forward()"` | 无 | `src/octop_browser/actions/navigate.py:64`、`:74` |
| `reload` | `Page.reload` | `ignoreCache=False` | 无 | `src/octop_browser/actions/navigate.py:84` |
| `dom_tree` | `DOM.getDocument` | `depth=-1` | `root` | `src/octop_browser/session.py:306-307` |
| `dom_tree` | `Runtime.evaluate` | `expression`（JSON.stringify url/title）、`returnByValue=True` | `result.value` → `url`/`title` | `src/octop_browser/session.py:308-319` |
| `screenshot` | `DOM.getBoxModel`（仅 element_ref） | `nodeId` | `model.content` | `src/octop_browser/actions/capture.py:136-142` |
| `screenshot` | `Page.getLayoutMetrics`（仅 full_page 或补尺寸） | `{}` | `cssContentSize`/`cssVisualViewport` | `src/octop_browser/actions/capture.py:154-159`、`:194-201` |
| `screenshot` | `Page.captureScreenshot` | `format="png"`、条件 `clip{x,y,width,height,scale}`、条件 `captureBeyondViewport=True` | `data`（base64） | `src/octop_browser/actions/capture.py:125-170` |
| `click` | `Input.dispatchMouseEvent` ×2 | `type`、`x`、`y`、`button="left"`、`clickCount=1` | 无 | `src/octop_browser/actions/interact.py:74-84` |
| `scroll` | `Input.dispatchMouseEvent` | `type="mouseWheel"`、`x=400`、`y=300`、`deltaX`、`deltaY` | 无 | `src/octop_browser/actions/interact.py:124-132` |
| `hover` | `Input.dispatchMouseEvent` | `type="mouseMoved"`、`x`、`y` | 无 | `src/octop_browser/actions/interact.py:152-159` |
| `type` / `fill` | `Input.dispatchKeyEvent` | `type="char"`、`text`；fill 另有 `keyDown/keyUp` + `modifiers∈{2,4}` + `key="Backspace"` | 无 | `src/octop_browser/actions/interact.py:104`、`:180-198` |
| `press` | `Input.dispatchKeyEvent` ×2 | `type`、`key`、`code` | 无 | `src/octop_browser/actions/interact.py:215-220` |
| `select`（ref） | `DOM.resolveNode` + `Runtime.callFunctionOn` | `nodeId`；`objectId`、`functionDeclaration`、`arguments=[{"value": value}]`、`returnByValue=True` | `result.value` | `src/octop_browser/actions/interact.py:239-257` |
| `select`（selector） | `Runtime.evaluate` | `expression`（含 `repr(selector)`/`repr(value)` 拼接）、`returnByValue=True` | 无 | `src/octop_browser/actions/interact.py:259-268` |
| `eval_js` | `Runtime.evaluate` | `expression`、`returnByValue=True`、`awaitPromise=True` | `result.value`/`result.subtype`/`result.description` | `src/octop_browser/actions/js_eval.py:19-48` |
| `wait` | `Runtime.evaluate` | `expression`（含 url/text/selector 探测）、`returnByValue=True` | `result.value` → JSON | `src/octop_browser/session.py:504-519` |
| `new_tab` | `Target.createTarget` | `url`（或 `about:blank`） | `targetId` | `src/octop_browser/session.py:594-597` |
| `switch_tab` | `Target.activateTarget` | `targetId` | 无 | `src/octop_browser/session.py:113-115` |
| `list_tabs` / `switch_to_target` | HTTP `GET /json` | `timeout=5s` / 无 | 数组元素 `id`/`url`/`title`/`webSocketDebuggerUrl` | `src/octop_browser/session.py:807-809`、`:121-124` |
| `close_tab` | `Target.closeTarget` | `targetId` | 无 | `src/octop_browser/session.py:773-775` |
| 连接 | `Emulation.setDeviceMetricsOverride` | `width`、`height`、`deviceScaleFactor=1`、`mobile=False` | 无 | `src/octop_browser/session.py:86-94` |

## 4. 回放专用的变量与 URL 转换

| 输入 Key | 转换函数 | 输出 | 锚点 |
|---|---|---|---|
| `inputs["sensitive_value"]` | `_resolve_value()` | 替换 `{{sensitive_value}}` 为运行时密钥 | `src/octop_browser/record/replay.py:566-570` |
| `inputs["query"]` | `_resolve_url()` → `_replace_known_search_query()` | 百度 `wd=`、Google/Bing `q=` 换成新查询并 `quote_plus` | `src/octop_browser/record/replay.py:606-645` |
| `step.target.selectorCandidates` | `_selector_xpath_js()` | JS 中 `__SELECTORS__` / `__XPATH__` / `__KIND__` / `__TEXT__` 占位符由 `json.dumps` 注入 | `src/octop_browser/record/replay.py:499-557` |
| `step.target.bbox` | `_fallback_dom_action()` | 中心点 `x + width/2`, `y + height/2` → `browser_tool("click", x=, y=)` | `src/octop_browser/record/replay.py:491-496` |

## 5. 共变清单

任一字典键变更必须依次核对：上游 Schema（CLI/MCP）→ `_action_kwargs()`/`call_tool()` → `tool_interface` 白名单 → `session` 方法签名 → `actions` 字典拼装 → [Chrome CDP 契约](downstream/chrome-cdp.md) → 对应单测。必跑 `uv run pytest tests/unit/test_cli.py tests/unit/test_mcp_server.py tests/unit/test_tool_interface.py tests/unit/test_navigate_actions.py -q`。

- 关联阅读：[组件总表](../architecture/index.md)、[入向接口契约全集](../contracts/api-reference.md)。
