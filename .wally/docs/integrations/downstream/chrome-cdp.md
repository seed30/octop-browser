---
kind: downstream-contract
scope: chrome-cdp
status: draft
origin: generated
source_paths: [src/octop_browser/cdp/client.py, src/octop_browser/cdp/launcher.py, src/octop_browser/session.py, src/octop_browser/actions/navigate.py, src/octop_browser/actions/interact.py, src/octop_browser/actions/capture.py, src/octop_browser/actions/js_eval.py]
symbols: [CDPClient.send, _build_flags, get_page_ws_url, navigate, click, screenshot, eval_js]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# 下游出向接口契约：`Chrome DevTools Protocol`

本仓不通过 Playwright/Selenium 驱动浏览器，而是自己与 Chrome 的两个出向通道通信：HTTP `/json*` 发现端点，以及页面级 WebSocket CDP 指令通道。公共成功判定为 WebSocket 响应不含 `error` 键且 `result` 可被解包（`src/octop_browser/cdp/client.py:113-116`）。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点 |
|---|---|---|
| 本仓业务封装文件 | `src/octop_browser/cdp/client.py`、`src/octop_browser/cdp/launcher.py` | 指令与进程双封装 |
| 底层传输 | `websockets.asyncio.client.ClientConnection`，非线程、单连接 | `src/octop_browser/cdp/client.py:11-12`、`:50` |
| HTTP 发现主机 | `cfg.cdp_host`，默认 `localhost`；禁止硬编码 | `src/octop_browser/settings.py:139-143` |
| HTTP 探测超时 | `/json/version` 2s、`/json` 5s、`/json/version`（端口判断）1s | `src/octop_browser/cdp/launcher.py:384`、`:678`、`:426` |
| WebSocket 帧上限 | `settings.cdp_max_message_size`，默认 33554432；`0` 转 `None` 表示不设上限 | `src/octop_browser/settings.py:171-177`、`src/octop_browser/cdp/client.py:67-70` |
| 页面级 WS 选择 | 粘滞 target → 首个非 `about:blank` → 任意 page → 新建 blank | `src/octop_browser/cdp/launcher.py:618-671` |

## 2. 本仓上游调用方反查表

| 封装方法 | 上游调用模块 / 函数 | 所属业务流 | 调用点 |
|---|---|---|---|
| `CDPClient.send("Page.navigate", ...)` | `actions.navigate.navigate` | [WF-01](../../business/wf-browser-action.md) | `src/octop_browser/actions/navigate.py:37` |
| `CDPClient.send("DOM.getDocument", ...)` | `session.BrowserSession.dom_tree`、`interact._resolve_coords` | WF-01 | `src/octop_browser/session.py:306`、`src/octop_browser/actions/interact.py:48` |
| `CDPClient.send("Input.dispatchMouseEvent", ...)` | `interact.click/scroll/hover` | WF-01 | `src/octop_browser/actions/interact.py:75`、`:124`、`:152` |
| `CDPClient.send("Input.dispatchKeyEvent", ...)` | `interact.type_text/fill_text/press_key` | WF-01 | `src/octop_browser/actions/interact.py:104`、`:181`、`:215` |
| `CDPClient.send("Page.captureScreenshot", ...)` | `capture.screenshot` | WF-01 | `src/octop_browser/actions/capture.py:170` |
| `CDPClient.send("Runtime.evaluate", ...)` | `js_eval.eval_js`、`session.wait`、`session.dom_tree` | WF-01 | `src/octop_browser/actions/js_eval.py:19`、`src/octop_browser/session.py:516`、`:308` |
| `get_page_ws_url()` | `_InternalCDPSession.connect`、`RecordController._attach` | [WF-02](../../business/wf-session-lifecycle.md)、[WF-03](../../business/wf-record-replay.md) | `src/octop_browser/session.py:60`、`src/octop_browser/record/recorder.py:339` |
| `launch_or_attach()` | `_InternalCDPSession.connect`、`RecordController._attach` | WF-02 / WF-03 | `src/octop_browser/session.py:59`、`src/octop_browser/record/recorder.py:336` |
| `terminate_browser()` | `_InternalCDPSession.close`、`tool_interface.close_session`、`idle.reap_now` | WF-02 | `src/octop_browser/session.py:160`、`src/octop_browser/tool_interface.py:148`、`src/octop_browser/idle.py:80` |

## 3. HTTP 发现端点字典

| 端点与方法 | 请求参数 | 返回解包键 | 调用锚点 |
|---|---|---|---|
| `GET http://{host}:{port}/json/version` | `timeout=2s`（启动等待）/ `1s`（端口判断） | `webSocketDebuggerUrl` → browser 级 WS；缺失则重试 | `src/octop_browser/cdp/launcher.py:369-389`、`:420-430` |
| `GET http://{host}:{port}/json` | `content_type=None` | 数组，过滤 `type == "page"`；元素取 `id`/`url`/`title`/`webSocketDebuggerUrl` | `src/octop_browser/cdp/launcher.py:674-682`、`src/octop_browser/session.py:808-829` |
| `PUT /json/new?about:blank`，失败回退 `GET` | `timeout=5s` | 新建 target 的 dict，需 `type == "page"` | `src/octop_browser/cdp/launcher.py:685-712` |
| WebSocket browser 级 `Browser.close` | `{"id": 1, "method": "Browser.close"}` | 不等待响应，轮询端口释放 | `src/octop_browser/cdp/launcher.py:469-483` |

## 4. WebSocket 指令组装字典与返回解包

`CDPClient.send()` 统一信封为 `{"id": int, "method": str, "params": dict}`，`id` 自增并以 Future 登记 (`src/octop_browser/cdp/client.py:89-96`)；返回取 `result["result"]`，若含 `error` 则抛 `CDPSessionError` (`src/octop_browser/cdp/client.py:113-116`)。

| method | 完整 params 键 | 值来源 / 条件 | 返回提取键 | 组装锚点 |
|---|---|---|---|---|
| `Page.navigate` | `url` | 入参原样 | `errorText`、`frameId` | `src/octop_browser/actions/navigate.py:37-53` |
| `Page.reload` | `ignoreCache` | 固定 `False` | 无 | `src/octop_browser/actions/navigate.py:84` |
| `DOM.getDocument` | `depth` | `dom_tree` 用 `-1`（全树）；`_resolve_coords` 用 `0` | `root.nodeId`、整棵 root | `src/octop_browser/session.py:306`、`src/octop_browser/actions/interact.py:48` |
| `DOM.querySelector` | `nodeId`、`selector` | `root.nodeId` + CSS 选择器 | `nodeId`（0 表示未命中） | `src/octop_browser/actions/interact.py:50-55` |
| `DOM.resolveNode` | `nodeId` | ref 查表所得 | `object.objectId` | `src/octop_browser/actions/interact.py:239-242` |
| `DOM.getBoxModel` | `nodeId` | ref / 控件节点 | `model.content` 8 点坐标 | `src/octop_browser/actions/interact.py:23-29`、`src/octop_browser/actions/capture.py:136-142` |
| `Runtime.evaluate` | `expression`、`returnByValue`、`awaitPromise` | 各动作自造 JS 字符串 | `result.value`、`result.subtype`、`result.description` | `src/octop_browser/actions/js_eval.py:20-25`、`src/octop_browser/session.py:311-317` |
| `Runtime.callFunctionOn` | `objectId`、`functionDeclaration`、`arguments`、`returnByValue` | select 操作的 setter JS | `result.value` | `src/octop_browser/actions/interact.py:249-257` |
| `Input.dispatchMouseEvent` | `type`、`x`、`y`、`button`、`clickCount` | click 时 `mousePressed`/`mouseReleased` 各一次 | 无 | `src/octop_browser/actions/interact.py:74-84` |
| `Input.dispatchMouseEvent`（滚轮） | `type="mouseWheel"`、`x=400`、`y=300`、`deltaX`、`deltaY` | direction/amount 换算，默认 300px | 无 | `src/octop_browser/actions/interact.py:120-132` |
| `Input.dispatchMouseEvent`（移动） | `type="mouseMoved"`、`x`、`y` | `_get_box_center` 结果 | 无 | `src/octop_browser/actions/interact.py:152-159` |
| `Input.dispatchKeyEvent`（字符） | `type="char"`、`text` | 逐字符发送 | 无 | `src/octop_browser/actions/interact.py:104`、`:198` |
| `Input.dispatchKeyEvent`（按键） | `type`、`key`、`code`、`modifiers` | `press_key` 映射表 + fill 的 Ctrl/Meta+A | 无 | `src/octop_browser/actions/interact.py:180-196`、`:209-220` |
| `Emulation.setDeviceMetricsOverride` | `width`、`height`、`deviceScaleFactor=1`、`mobile=False` | `cfg.viewport_width/height`，下限 1 | 无 | `src/octop_browser/session.py:86-94` |
| `Page.captureScreenshot` | `format="png"`，可选 `clip{x,y,width,height,scale}`、`captureBeyondViewport` | `element_ref` 优先，其次 `full_page` | `data`（base64） | `src/octop_browser/actions/capture.py:125-170` |
| `Page.getLayoutMetrics` | `{}` | full_page 取 `cssContentSize`/`contentSize`；否则取 `cssVisualViewport`/`visualViewport` | `width`/`height`/`clientWidth`/`clientHeight` | `src/octop_browser/actions/capture.py:154-159`、`:194-201` |
| `Page.addScriptToEvaluateOnNewDocument` | `source` | JS 采集器配置 + 源码 | 无 | `src/octop_browser/record/recorder.py:172` |
| `Runtime.addBinding` | `name` | `__hbr_emit_<hex>` 随机绑定名 | 无 | `src/octop_browser/record/recorder.py:170` |
| `Target.createTarget` | `url` | 新标签 URL 或 `about:blank` | `targetId` | `src/octop_browser/session.py:594-597` |
| `Target.activateTarget` | `targetId` | 切标签目标 | 无（失败仅记日志） | `src/octop_browser/session.py:113-115` |
| `Target.closeTarget` | `targetId` | 关闭标签 | 无 | `src/octop_browser/session.py:773-775`、`:665-667` |

## 5. 错误翻译、超时与共变清单

| 下游错误 / 条件 | 本仓转换 | 锚点 |
|---|---|---|
| WebSocket 请求超时 | `CDPSessionError("Timeout waiting for response to {method}")`，默认 30s | `src/octop_browser/cdp/client.py:99-104` |
| 响应含 `error` 键 | `CDPSessionError("CDP error for {method}: {error}")` | `src/octop_browser/cdp/client.py:113-114` |
| 接收循环异常 | 所有 pending Future 设 `CDPSessionError("Connection lost: ...")` | `src/octop_browser/cdp/client.py:172-177` |
| `Page.navigate` 的 `errorText` 非空 | `ToolResult(success=False, error="Navigation failed for {url}: {errorText}")` | `src/octop_browser/actions/navigate.py:38-45` |
| ref 未找到 | `CDPSessionError("Ref '{ref}' not found. Call dom_tree() first.")` | `src/octop_browser/actions/interact.py:44`、`src/octop_browser/actions/capture.py:133` |
| Chrome 启动即退出 | 读 stderr tail 拼入 `RuntimeError`，含 `returncode=` | `src/octop_browser/cdp/launcher.py:374-380` |
| 端口占用 | 直接 attach，不重复启动 | `src/octop_browser/cdp/launcher.py:508-514` |

- **共变**：新增 CDP method 必须在第 4 节补字典键表，并同步 `tests/unit/test_launcher.py`/`tests/unit/test_navigate_actions.py`。
- **共变**：改 `cdp_max_message_size` 语义必须同步 [配置字典](../../contracts/configuration.md) 与 `tests/unit/test_settings.py`。
- **禁止**：任何 `localhost` 字面量进入 HTTP URL 拼接，一律使用 `cfg.cdp_host`。
