---
kind: error-dict
scope: error-codes
status: draft
origin: generated
source_paths: [src/octop_browser/tool_interface.py, src/octop_browser/cdp/client.py, src/octop_browser/cdp/launcher.py, src/octop_browser/actions/navigate.py, src/octop_browser/actions/interact.py, src/octop_browser/actions/capture.py, src/octop_browser/actions/js_eval.py, src/octop_browser/session.py, src/octop_browser/mode.py, src/octop_browser/hooks.py, src/octop_browser/dom/builder.py, src/octop_browser/record/daemon.py, src/octop_browser/record/replay.py, src/octop_browser/install.py]
symbols: [CDPSessionError, _error_result, RecordDaemon.dispatch, ReplayRunner.run]
upstream: [INDEX.md, contracts/api-reference.md]
downstream: [operations/runbook-and-observability.md, business/wf-browser-action.md]
---

# 错误码与异常翻译字典

本仓没有数值错误码枚举；错误以三类载体出现：`ToolResult.error` 字符串、`CDPSessionError`/`ValueError`/`RuntimeError` 异常、以及 daemon/replay 的结构化 `{"ok": false, "error": ...}` / `status="failed"`。下文按来源逐条列出字面串与触发行号，禁止按语义猜测。

## 1. 入向分发层错误

| 字面错误串 / 条件 | 触发条件 | 返回形态 | 锚点 |
|---|---|---|---|
| `Unknown action '{action}'. Valid: [...]` | `action not in _KNOWN_ACTIONS` | `ToolResult(success=False)` | `src/octop_browser/tool_interface.py:120-123` |
| `Invalid parameters for 'close_session': [...]` | close_session 除 `kill` 外仍有 kwargs | `ToolResult(success=False)` | `src/octop_browser/tool_interface.py:127-131` |
| `Failed to create session: {exc}` | `_get_or_create_session()` 抛异常 | `ToolResult(success=False)` | `src/octop_browser/tool_interface.py:167-168` |
| `Invalid parameters for '{action}': {exc}` | `getattr(sess, action)(**kwargs)` 抛 `TypeError` | `ToolResult(success=False)` | `src/octop_browser/tool_interface.py:183-184` |
| `Invalid browser mode {value!r}. Expected one of: auto, headed, headless` | `normalize_mode()` 收到非法值 | `ValueError` | `src/octop_browser/mode.py:35-39` |
| `Unknown event {event!r}. Valid: [...]` | `HooksMixin.on()` 注册非法事件 | `ValueError` | `src/octop_browser/hooks.py:36-37` |
| `Unknown DOM level: {level!r}` | `DOMBuilder.build()` 收到四档之外的值 | `ValueError` | `src/octop_browser/dom/builder.py:184` |
| `Unknown record command {cmd}` | CLI `_run_record()` 未匹配子命令 | `{"ok": False, "error": ...}` | `src/octop_browser/cli.py:599-603` |
| `Unknown tool: {name}` | MCP 工具名不在 `_ACTION_MAP` 且非特殊分支 | 纯文本响应，非异常 | `src/octop_browser/mcp_server.py:459-461` |

## 2. CDP 传输与浏览器启动错误

| 字面错误串 | 语义 | 本仓处理 | 锚点 |
|---|---|---|---|
| `Not connected` | `_ws is None` 时调用 `send()` | 抛 `CDPSessionError` | `src/octop_browser/cdp/client.py:83-84` |
| `Timeout waiting for response to {method}` | `asyncio.wait_for` 超时；默认 30s | 清理 pending 后抛错 | `src/octop_browser/cdp/client.py:100-104` |
| `CDP error for {method}: {error}` | 响应含 `error` 键 | 抛 `CDPSessionError` | `src/octop_browser/cdp/client.py:113-114` |
| `CDP send failed for {method}: {exc}` | 其它发送异常 | 清理 pending 后抛错 | `src/octop_browser/cdp/client.py:107-109` |
| `Connection lost: {exc}` | 接收循环异常 | 批量给所有 pending Future 设异常 | `src/octop_browser/cdp/client.py:172-177` |
| `Chrome exited immediately (returncode={n}) before binding to {host}:{port}` | 启动期子进程已退出 | 附 stderr tail 后立即抛 `RuntimeError` | `src/octop_browser/cdp/launcher.py:374-380` |
| `Chrome did not start on {host}:{port} after {n} attempts` | 轮询耗尽 | 附 stderr tail 抛 `RuntimeError` | `src/octop_browser/cdp/launcher.py:393-398` |
| `Cannot launch Chrome on remote host '{host}'` | `cdp_host` 非 loopback 且端口空闲 | 要求远端手动启动或设 `cdp_ws_url` | `src/octop_browser/cdp/launcher.py:517-523` |
| `Chrome/Chromium not found` | `find_chrome()` 返回 None | 抛错并给出 apt/brew 安装提示 | `src/octop_browser/cdp/launcher.py:525-531` |
| `No open page found in Chrome` | 没有任何 page target 且创建失败 | 抛 `RuntimeError` | `src/octop_browser/cdp/launcher.py:648-649` |
| `No page target found for tab {target_id}` | `switch_to_target()` 找不到匹配 target | 抛 `RuntimeError` | `src/octop_browser/session.py:133-134` |
| `Cannot switch tabs when connected via BROWSER_USE_CDP_WS_URL` | 直连模式下请求切标签 | 抛 `RuntimeError` | `src/octop_browser/session.py:107-110` |

## 3. 动作层错误与降级

| 字面错误串 | 触发条件 | 返回形态 | 锚点 |
|---|---|---|---|
| `Navigation failed for {url}: {errorText}` | `Page.navigate` 返回 `errorText` | `ToolResult(success=False)`，不再等 load | `src/octop_browser/actions/navigate.py:38-45` |
| 导航 load 事件 5s 超时 | 无 `Page.loadEventFired` | **降级为成功**，内容 `Navigated to {url}` | `src/octop_browser/actions/navigate.py:46-49` |
| `Ref '{ref}' not found. Call dom_tree() first.` | `RefCache.lookup()` 返回 None | `CDPSessionError` | `src/octop_browser/actions/interact.py:44`、`src/octop_browser/actions/capture.py:133` |
| `Selector '{selector}' matched no elements.` | `DOM.querySelector` 返回 nodeId=0 | `CDPSessionError` | `src/octop_browser/actions/interact.py:54-55` |
| `Must provide ref, selector, or (x, y)` | click 无任何定位参数 | `ValueError` | `src/octop_browser/actions/interact.py:60` |
| `Must provide ref or selector` | `select_value()` 无定位参数 | `ValueError` | `src/octop_browser/actions/interact.py:270` |
| `Could not resolve ref '{ref}'.` | `DOM.resolveNode` 无 objectId | `CDPSessionError` | `src/octop_browser/actions/interact.py:241-242` |
| `Page.captureScreenshot returned empty data` | 截图 `data` 为空 | `ToolResult(success=False)` | `src/octop_browser/actions/capture.py:171-183` |
| `Unknown JS error` | JS 结果 `subtype=="error"` 且无 description | `ToolResult.error` 兜底串 | `src/octop_browser/actions/js_eval.py:28-40` |
| `Timeout waiting for condition` | `wait()` 10s 内条件未满足 | `ToolResult(success=False)` | `src/octop_browser/session.py:501`、`:545-557` |
| `setDeviceMetricsOverride failed` | 视口设置异常 | 仅 `logger.debug`，**降级为继续运行** | `src/octop_browser/session.py:95-96` |
| `Target.activateTarget failed` | 切标签激活异常 | 仅 `logger.debug`，继续走 WS 重连 | `src/octop_browser/session.py:116-117` |
| Hook 回调异常 | `_fire()` 中任意回调抛错 | `logger.exception`，**不中断动作** | `src/octop_browser/hooks.py:48-53` |

## 4. 安装与录制守护进程错误

| 字面错误串 | 触发条件 | 返回形态 | 锚点 |
|---|---|---|---|
| `Chromium binary not found` | `chromium_executable()` 为 None | `verify_chromium() -> (False, msg)` | `src/octop_browser/install.py:237-239` |
| `Chromium verification timed out` | `--version` 超 10s | 同上 | `src/octop_browser/install.py:247-248` |
| `missing system library...` + `manual_hint()` | rc=127 或 stderr 含 `shared libraries` | 提取首个 `.so` 行 | `src/octop_browser/install.py:256-267` |
| `playwright package install failed` | 所有 pip 候选失败 | 同步 False / 流式 `done=False` | `src/octop_browser/install.py:568`、`:600-602` |
| `Chromium download failed` | 所有 CDN 候选下载失败 | 同上 | `src/octop_browser/install.py:581` |
| `Record daemon is not running. Start it with \`octop-browser record daemon-start\`.` | unix socket 连接失败 | 标准 `{"ok": False, "error": ...}` | `src/octop_browser/record/daemon.py:152-159` |
| `Daemon did not start` | 启动后 30×0.1s 内 ping 不通 | `{"ok": False, ...}` | `src/octop_browser/record/daemon.py:198` |
| `A recording is already active` | `start` 时已有活动会话 | `{"ok": False, "active": ...}` | `src/octop_browser/record/daemon.py:44-49` |
| `No active recording` | `stop` 时无活动会话 | `{"ok": False, ...}` | `src/octop_browser/record/daemon.py:65-66` |
| `Active recording is {id}, not {id}` | stop 的 recording_id 不匹配 | `{"ok": False, ...}` | `src/octop_browser/record/daemon.py:67-72` |
| `Unknown command: {command}` | daemon 命令字非法 | `{"ok": False, ...}` | `src/octop_browser/record/daemon.py:91` |

## 5. 回放状态机与降级字面量

回放报告的状态字面量只有 `running`/`passed`/`failed`，步骤级额外有 `skipped`。

| 状态 / 字面量 | 出现条件 | 锚点 |
|---|---|---|
| `matchedBy="not-found"` | `match_target()` 分数低于 45 且无兜底 | `src/octop_browser/record/replay.py:165`、`src/octop_browser/record/matcher.py:60-62` |
| `matchedBy="selector/xpath"` | 选择器或 XPath JS 兜底成功 | `src/octop_browser/record/replay.py:489` |
| `matchedBy="bbox"` | 录制含 bbox 时坐标兜底成功 | `src/octop_browser/record/replay.py:496` |
| `status="skipped", reason="transient_auth_or_risk_redirect"` | 回放导航命中瞬时风控 URL | `src/octop_browser/record/replay.py:210-219` |
| `status="skipped"` | 未知 kind 的步骤 | `src/octop_browser/record/replay.py:269-270` |
| `Recording has no replayable steps` | `steps_doc.steps` 为空 | `src/octop_browser/record/replay.py:45-46` |
| `Recording has no executable steps` | 全部步骤均为 skipped | `src/octop_browser/record/replay.py:282-284` |
| `Could not match target for {id}: {target}` | 目标不可达 | `src/octop_browser/record/replay.py:159-168` |
| `Unsupported verification: {typ}` | 验证类型不在 6 种之内 | `src/octop_browser/record/replay.py:412-415` |
| `Verification failed: {typ}={value}` | 任一验证项失败 | `src/octop_browser/record/replay.py:417-422` |

诚实声明：源码中**未找到** `degraded` 字面量（`Grep` 全仓 `src/octop_browser` 零命中）。本仓的降级语义通过「超时仍成功」「仅记日志」「跳过瞬时 URL」「兜底坐标点击」实现，没有 `degraded` 状态字段。

## 6. 排障联动

按错误串定位决策树见 [排障 Runbook](../operations/runbook-and-observability.md)；调用链定位见 [主干全景](../overview/callchain-panorama.md)。修改错误串必须同步 `tests/unit/test_cli.py`、`test_tool_interface.py`、`test_mcp_server.py` 与本页表格。
