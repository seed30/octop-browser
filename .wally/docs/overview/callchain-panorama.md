---
kind: panorama
scope: callchain-panorama
status: draft
origin: generated
source_paths: [src/octop_browser/cli.py, src/octop_browser/mcp_server.py, src/octop_browser/tool_interface.py, src/octop_browser/session.py, src/octop_browser/cdp/client.py, src/octop_browser/cdp/launcher.py, src/octop_browser/record/recorder.py, src/octop_browser/record/replay.py, src/octop_browser/record/daemon.py, src/octop_browser/install.py]
symbols: [browser_tool, BrowserSession, CDPClient.send, launch_or_attach, ReplayRunner.run, RecordDaemon.dispatch]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [business/index.md, contracts/error-codes.md]
---

# 主干调用链全景

本页把 CLI、MCP、SDK、录制回放四条入口收敛到同一棵符号调用树，并标出线程/子进程/Unix socket 边界。图中每个节点都是真实符号，锚点为 `path:line`。

## 1. 主干符号调用树

```text
[入口 A] src/octop_browser/cli.py:631 main()
  ├─ install-browser :637 -> src/octop_browser/install.py:280 ensure_chromium() -> :332 race_download_sources() -> :364 verify_chromium()
  ├─ record :644 -> src/octop_browser/record/daemon.py:140 send_request() ──Unix Socket──> src/octop_browser/record/daemon.py:39 RecordDaemon.dispatch()
  │                                                                  └─ start -> src/octop_browser/record/recorder.py:350 start_in_process()
  ├─ replay :646 -> src/octop_browser/record/replay.py:25 ReplayRunner.run() -> 每步 browser_tool()
  └─ 动作 :649 -> _run_action() :514
        └─ src/octop_browser/tool_interface.py:91 browser_tool(action, profile, mode, **kwargs)
[入口 B] src/octop_browser/mcp_server.py:391 call_tool()
  ├─ 特殊分支 :393-457（install/record/replay/browser_open）
  └─ 动作分支 :475 -> browser_tool(action=action, **arguments)
[入口 C] Python SDK -> 直接 browser_tool() / BrowserSession.create()

公共主干：
browser_tool() src/octop_browser/tool_interface.py:91
  ├─ [guard] :120 action in _KNOWN_ACTIONS -> 否则 _error_result()
  ├─ [guard] :125 close_session -> registry.pop + sess.close(kill=) + 可选 terminate_browser()
  ├─ [registry] :57 _get_or_create_session(profile) -> src/octop_browser/session.py:199 BrowserSession.create()
  │    ├─ src/octop_browser/profile.py:165 get_or_create() -> _assign_port() :151
  │    ├─ src/octop_browser/mode.py:55 resolve_headless()（经 :231-235 仲裁）
  │    └─ src/octop_browser/session.py:49 _InternalCDPSession.connect()
  │         ├─ cfg.cdp_ws_url ? 直连 client.connect() :55
  │         ├─ 否则 src/octop_browser/cdp/launcher.py:497 launch_or_attach() ──subprocess.Popen──> Chrome :578
  │         ├─ src/octop_browser/cdp/launcher.py:618 get_page_ws_url() ──HTTP /json──> Chrome
  │         ├─ src/octop_browser/cdp/client.py:56 CDPClient.connect(ws) ──WebSocket──> Chrome
  │         └─ src/octop_browser/session.py:74 _apply_viewport() -> Emulation.setDeviceMetricsOverride
  ├─ [dispatch] :180 getattr(sess, action)(**kwargs)
  │    ├─ src/octop_browser/session.py:262 navigate -> src/octop_browser/actions/navigate.py:25 -> Page.navigate + Page.loadEventFired(5s)
  │    ├─ src/octop_browser/session.py:300 dom_tree -> DOM.getDocument -> DOMBuilder.build(level) -> tokens=len//4
  │    ├─ src/octop_browser/session.py:377 click -> src/octop_browser/actions/interact.py:63 -> DOM.getBoxModel -> Input.dispatchMouseEvent ×2
  │    ├─ src/octop_browser/session.py:396/416 type/fill -> Input.dispatchKeyEvent 逐字符
  │    ├─ src/octop_browser/session.py:485 eval_js -> Runtime.evaluate(awaitPromise=True)
  │    ├─ src/octop_browser/session.py:491 wait -> 0.25s 轮询 Runtime.evaluate（默认 10s）
  │    ├─ src/octop_browser/session.py:563 new_tab -> Target.createTarget + switch_to_target
  │    ├─ src/octop_browser/session.py:799 list_tabs ──HTTP /json──> Chrome
  │    └─ src/octop_browser/session.py:348 screenshot -> Page.captureScreenshot -> 写 PNG -> ToolResult.metadata
  ├─ [metrics] src/octop_browser/session.py:895 _record() -> before/after_action 钩子
  └─ [retry] :186 断线 -> _drop_session() -> 重建会话 -> 重试一次
```

## 2. 线程、队列、子进程与跨进程桥接

| 边界类型 | 实现 | 生命周期 | 锚点 |
|---|---|---|---|
| 子进程：Chrome | `subprocess.Popen(cmd, start_new_session=True)` | 跨 CLI 调用存活，直到 `--kill` 或空闲回收 | `src/octop_browser/cdp/launcher.py:566-578` |
| 子进程：用户命名空间探测 | `subprocess.run([python,-c,unshare], timeout=2)` | 进程级缓存 `_USER_NS_CACHE` | `src/octop_browser/cdp/launcher.py:315-347` |
| 子进程：pip / 下载 / 系统库 | `subprocess.run` 与 `asyncio.create_subprocess_exec` | 安装期 | `src/octop_browser/install.py:159`、`:339`、`:385` |
| 线程池 | `ThreadPoolExecutor(max_workers=3)` CDN 竞速；`asyncio.to_thread` 验证/终止 | 调用期 | `src/octop_browser/playwright_download.py:123`、`src/octop_browser/install.py:514` |
| 异步任务：空闲回收 | `asyncio.create_task(BrowserIdleController._run())`，60s 轮询 | 会话期 | `src/octop_browser/idle.py:43-48`、`:111-129` |
| 异步任务：CDP 接收 | `asyncio.create_task(CDPClient._recv_loop())` | 连接期 | `src/octop_browser/cdp/client.py:72`、`:152-177` |
| 异步任务：目标轮询 | `asyncio.create_task(RecordSession._poll_targets())`，0.5s | 录制期 | `src/octop_browser/record/recorder.py:99`、`:220-230` |
| IPC：Unix socket | `RecordDaemon.run()` 服务，`send_request()` 客户端 | 显式 daemon-start 后长期 | `src/octop_browser/record/daemon.py:107-137`、`:140-165` |
| 文件桥接 | `events.jsonl` 追加写 + flush，CLI 进程间读取 | 录制后持久 | `src/octop_browser/record/store.py:82-97` |

## 3. 录制与回放的分支调用树

```text
[录制] src/octop_browser/cli.py:557 record start
  -> src/octop_browser/record/daemon.py:39 dispatch("start") -> src/octop_browser/record/recorder.py:350 start_in_process()
  -> src/octop_browser/record/recorder.py:165 _install_on_client() ──CDP──> Chrome 注入 recorder.js
  -> src/octop_browser/record/injected/recorder.js:333 事件监听 -> emit() :55 ──Runtime.bindingCalled──> src/octop_browser/record/recorder.py:256 _on_binding_called()
  -> src/octop_browser/record/recorder.py:240 _append() -> src/octop_browser/record/privacy.py:61 redact_event() -> src/octop_browser/record/store.py:82 append_event()
  -> src/octop_browser/record/recorder.py:101 stop() -> src/octop_browser/record/semantic.py:338 process() -> src/octop_browser/record/skill_generator.py:32 generate()

[回放] src/octop_browser/cli.py:610 _run_replay() -> src/octop_browser/record/replay.py:25 run()
  -> src/octop_browser/record/store.py:111 read_steps() -> 每步 browser_tool()（复用主干）
  -> src/octop_browser/record/matcher.py:19 match_target() 选 ref（阈值 45）
  -> 失败兜底 src/octop_browser/record/replay.py:469 _fallback_dom_action() -> selector/xpath JS 或 bbox 坐标
  -> src/octop_browser/record/replay.py:647 _write_report() -> replay-reports/replay_<ts>.json
```

## 4. 异常短路出口汇总

| 短路点 | 条件 | 出口形态 | 锚点 |
|---|---|---|---|
| 动作白名单 | 非法 action | 立即 `_error_result()` | `src/octop_browser/tool_interface.py:120-123` |
| 会话创建 | 任意异常 | `Failed to create session: ...` | `src/octop_browser/tool_interface.py:167-168` |
| 参数不匹配 | `TypeError` | `Invalid parameters for ...` | `src/octop_browser/tool_interface.py:183-184` |
| CDP 错误 | 响应含 `error`/超时/断线 | `CDPSessionError` → `ToolResult.error` | `src/octop_browser/cdp/client.py:99-114` |
| 录制门禁 | 超字节/超频/nonce 不符 | 静默丢弃事件 | `src/octop_browser/record/recorder.py:256-286` |
| 回放失败 | 步骤失败或验证失败 | 写失败产物后 `break`，报告 `status=failed` | `src/octop_browser/record/replay.py:272-280`、`:417-422` |
| 安装失败 | 任一阶段失败 | 流式 `done/success=false` 或 `False` | `src/octop_browser/install.py:357-369`、`:600-620` |

## 5. 共变清单

修改主干任一节点必须回溯本页调用树、[系统技术架构](system-architecture.md) 的分层表与 [模块依赖矩阵](../architecture/module-dep-matrix.md)。必跑 `make all` 与相应单测；调用链证据必须来自真实源码，不得按弱命中推测。
