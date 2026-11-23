---
kind: downstream-contract
scope: record-daemon-ipc
status: draft
origin: generated
source_paths: [src/octop_browser/record/daemon.py, src/octop_browser/cli.py, src/octop_browser/mcp_server.py, src/octop_browser/record/settings.py]
symbols: [RecordDaemon.dispatch, send_request, ensure_daemon, socket_path, pid_path]
upstream: [integrations/index.md, architecture/record-engine.md]
downstream: []
---

# 下游出向接口契约：`录制守护进程（Unix Socket JSON IPC）`

`record` 命令需要跨多次 CLI 调用持有同一个 `RecordSession`，因此通过本地 Unix socket 与一个独立 Python 进程通信。协议是「一行 JSON 请求 → 一行 JSON 响应」，无鉴权，仅绑定本地文件系统路径。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点 |
|---|---|---|
| 本仓封装文件 | `src/octop_browser/record/daemon.py` | 服务端与客户端同文件 |
| Socket 路径 | `settings.base_dir / "daemon.sock"` | `src/octop_browser/record/daemon.py:132-133` |
| PID 路径 | `settings.base_dir / "daemon.pid"` | `src/octop_browser/record/daemon.py:136-137` |
| base_dir 默认值 | `BROWSER_RECORD_REPLAY_DIR` 或 `state_dir()/record-replay` | `src/octop_browser/record/settings.py:24-29` |
| 传输 | `asyncio.start_unix_server` + `open_unix_connection`，`readline()` 定帧 | `src/octop_browser/record/daemon.py:114`、`:149-162` |
| 请求超时 | 5.0s（客户端默认） | `src/octop_browser/record/daemon.py:140` |
| 进程启动 | `[sys.executable, "-m", "octop_browser.record.daemon"]`，POSIX 下 `start_new_session=True` | `src/octop_browser/record/daemon.py:176-187` |

## 2. 本仓上游调用方反查表

| 客户端入口 | 发出的 request 字典 | 所属业务流 | 锚点 |
|---|---|---|---|
| `cli._run_record` `daemon-start` | `ensure_daemon()`（内部 ping） | [WF-03](../../business/wf-record-replay.md) | `src/octop_browser/cli.py:543` |
| `cli._run_record` `daemon-stop` | `{"command": "shutdown"}` | WF-03 | `src/octop_browser/cli.py:545` |
| `cli._run_record` `status` | `{"command": "status"}` | WF-03 | `src/octop_browser/cli.py:547` |
| `cli._run_record` `stop` | `{"command":"stop","recording_id":...,"generate_steps":not --no-generate-steps,"name":...}` | WF-03 | `src/octop_browser/cli.py:549-556` |
| `cli._run_record` `start` | `{"command":"start","profile":...,"name":...,"privacy":...,"screenshots":"off"}` | WF-03 | `src/octop_browser/cli.py:577-585` |
| `mcp.call_tool` `browser_record_start/stop/status` | 同结构，由 `arguments.get()` 组装 | WF-03 | `src/octop_browser/mcp_server.py:400-427` |
| `mcp.call_tool` `browser_record_doctor` | 不走 IPC，直接 `RecordController.doctor()` | WF-03 | `src/octop_browser/mcp_server.py:395-399` |

## 3. 请求字典与响应解包全表

`RecordDaemon.dispatch()` 支持的命令字共 5 个（`src/octop_browser/record/daemon.py:39-91`）：

| command | 完整请求 Key | 成功响应 Key | 失败响应 | 锚点 |
|---|---|---|---|---|
| `ping` | `command` | `ok=True`、`active=_active_info()` | — | `src/octop_browser/record/daemon.py:41-42` |
| `start` | `command`、`profile`、`name`、`privacy`、`screenshots` | `ok` + `recordingId`/`store`/`startUrl`/`name` + `daemon=True` | 已有会话时 `ok=False,error="A recording is already active"` 并附 `active` | `src/octop_browser/record/daemon.py:43-63` |
| `stop` | `command`、`recording_id`、`generate_steps`、`name` | `ok=True` + `recordingId`/`events`/`steps`/`stepsPath`/`skillDraft` | 无会话 `No active recording`；ID 不符时回报实际 ID | `src/octop_browser/record/daemon.py:64-81` |
| `status` | `command` | `ok=True`、`active=_active_info()` | — | `src/octop_browser/record/daemon.py:82-83` |
| `shutdown` | `command` | `ok=True`、`shutdown=True` | — | `src/octop_browser/record/daemon.py:84-90` |
| 其它 | 任意 | — | `ok=False,error="Unknown command: {command}"` | `src/octop_browser/record/daemon.py:91` |

`_active_info()` 的返回字典固定为 `{recordingId, profile, startUrl, events}` 或 `None` (`src/octop_browser/record/daemon.py:93-101`)。`start` 把 `privacy` 字符串包成 `PrivacySettings(input_policy=request.privacy, url_query_policy="mask-sensitive", screenshot_policy=request.screenshots)` (`src/octop_browser/record/daemon.py:50-54`)。

## 4. 生命周期、降级与错误包络

```text
ensure_daemon() src/octop_browser/record/daemon.py:190
  ├─ is_running() :168 -> send_request({"command":"ping"}, timeout=1.0)
  ├─ [已运行] 返回 {"ok":True,"alreadyRunning":True,"socket":...}
  └─ [未运行] start_daemon_process() :176 -> 轮询 30×0.1s ping :194-197
       └─ 超时返回 {"ok":False,"error":"Daemon did not start", pid, socket}

RecordDaemon.run() :107
  ├─ sock.parent.mkdir() -> unlink 旧 socket :108-113
  ├─ async with start_unix_server(handle) -> serve_forever() :114-118
  └─ finally 删除 socket 与 pid 文件 :121-129
```

客户端不可达时不泄漏原始 `Errno 2`，而是返回固定包络：`{"ok": False, "error": "Record daemon is not running. Start it with \`octop-browser record daemon-start\`."}` (`src/octop_browser/record/daemon.py:152-159`)。服务端 `handle()` 把任意异常统一转成 `{"ok": False, "error": str(exc)}` 后写回 (`src/octop_browser/record/daemon.py:28-37`)。`shutdown` 会先停止并落盘当前录制，再通过 `call_soon(_close_server)` 关闭监听 (`src/octop_browser/record/daemon.py:85-90`)。

## 5. 共变清单与边界

- 新增命令字：必须同时改 `dispatch()`、`cli._run_record()` 分支、`mcp_server.call_tool()` 分支与 [CLI 命令全集](../../contracts/api/cli-commands.md)。
- 改响应键：同步 `tests/unit/test_record_daemon_client.py` 与 [回放引擎](../../architecture/record-engine.md) 的事件消费逻辑。
- 边界：socket 仅限本地，不做鉴权；不要把它暴露到网络，也不要在响应中回显密钥或凭证。
- 必跑命令：`uv run pytest tests/unit/test_record_daemon_client.py -q`。
