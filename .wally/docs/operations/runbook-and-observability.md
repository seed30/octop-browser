---
kind: operations
scope: runbook-and-observability
status: draft
origin: generated
source_paths: [src/octop_browser/cdp/launcher.py, src/octop_browser/cdp/client.py, src/octop_browser/cdp/launch_prep.py, src/octop_browser/tool_interface.py, src/octop_browser/record/daemon.py, src/octop_browser/record/replay.py, src/octop_browser/idle.py]
symbols: [_get_ws_url, _tail_stderr, send_request, reap_now, _capture_failure_artifacts]
upstream: [INDEX.md, contracts/error-codes.md]
downstream: [operations/pitfalls-and-adr.md, operations/boundary.md]
---

# 排障 Runbook 与可观测信号

本仓没有指标端口或审计数据库；可观测性来自 `logging` 输出、`ToolResult.error`、录制报告 JSON 与磁盘目录。以下按症状给出确定性决策树，每条都指向源码检测点。

## 1. 信号源清单

| 信号 | 形态 | 产生位置 |
|---|---|---|
| `logger.debug` | CDP 连接/关闭、配置解析、锁清理 | `src/octop_browser/cdp/client.py:73`、`:150`；`src/octop_browser/mode.py:69-74`；`src/octop_browser/cdp/launch_prep.py:97` |
| `logger.info` | CDN 竞速胜出、空闲回收、锁清成功 | `src/octop_browser/playwright_download.py:142-147`；`src/octop_browser/idle.py:104-108`；`src/octop_browser/cdp/launch_prep.py:223-227` |
| `logger.warning` | 自愈重试、profile 迁移、下载源失败 | `src/octop_browser/cdp/launcher.py:591-595`；`src/octop_browser/cdp/launch_prep.py:180-184`；`src/octop_browser/install.py:352-356` |
| `logger.exception` | 钩子失败、browser_tool 失败、空闲回收异常 | `src/octop_browser/hooks.py:53`；`src/octop_browser/tool_interface.py:205`、`:211`；`src/octop_browser/idle.py:122-125` |
| Chrome stderr tail | 启动失败时拼入 `RuntimeError`，最多 4096 字节 | `src/octop_browser/cdp/launcher.py:401-417` |
| `ToolResult.error` | 动作失败的机器可读字符串 | `src/octop_browser/models.py:31` |
| `replay_*.json` | 逐步状态与验证结果 | `src/octop_browser/record/replay.py:647-657` |
| `failure_<ts>_<step>/` | 失败原因、DOM 快照、截图 | `src/octop_browser/record/replay.py:425-467` |

## 2. 启动失败决策树

```text
报错 "Chrome did not start on host:port after N attempts"
  ├─ stderr tail 含 "returncode=21"/"ProcessSingleton"/"SingletonLock"
  │    -> Singleton 冲突：recover_profile() 已自动改名旧目录并重试一次；仍失败则手工删 <profile>/.octop-browser-target.json 与 Singleton* 锁
  ├─ stderr 含 "Failed to move to new namespace"
  │    -> 用户命名空间受限：检查 BROWSER_USE_NO_SANDBOX=1；root 下启动器已自动加 --no-sandbox
  ├─ stderr 含 "shared libraries"/"error while loading"
  │    -> 系统库缺失：执行 install-browser 触发 install-deps，或按 manual_hint() 手工安装
  ├─ stderr 含 "no DISPLAY"/"cannot open display"
  │    -> BROWSER_USE_MODE=headless，或设置 DISPLAY/WAYLAND_DISPLAY
  └─ stderr 为空且端口不通
       -> 核对 cfg.cdp_host 与 BROWSER_USE_CDP_PORT_START；端口占用时启动器会直接 attach 而非新起
```

关键判定：`_get_ws_url()` 每轮先查 `process.poll()`，进程已退出立即抛错并附 stderr tail，不会等满 `launch_retries` 次 (`src/octop_browser/cdp/launcher.py:370-398`)。`_tail_stderr()` 任何异常都返回空串，避免掩盖原始 `RuntimeError` (`src/octop_browser/cdp/launcher.py:401-417`)。

## 3. 动作失败决策树

| 症状 | 检测点 | 处置 |
|---|---|---|
| `Ref 'xxx' not found. Call dom_tree() first.` | `src/octop_browser/actions/interact.py:44`、`src/octop_browser/actions/capture.py:133` | 先 `dom_tree`；若已取过则检查其间是否发生导航清缓存 |
| `Navigation failed for ...: {errorText}` | `src/octop_browser/actions/navigate.py:38-45` | 检查 URL 可达性；导航不触发 load 不算失败（5s 超时降级成功） |
| `Timeout waiting for response to <method>` | `src/octop_browser/cdp/client.py:99-104` | 调高 `BROWSER_USE_CDP_TIMEOUT`；页面卡死时考虑 `close-session --kill` |
| `Not connected` / `Connection lost` | `src/octop_browser/cdp/client.py:84`、`:176` | `browser_tool` 已自动重建一次；反复出现查 Chrome 是否被外部杀掉 |
| `Timeout waiting for condition` | `src/octop_browser/session.py:501` | 放宽 `timeout_ms`，或改用 `eval_js` 直接读状态 |
| `Unknown action` / `Invalid parameters` | `src/octop_browser/tool_interface.py:120`、`:183` | 对照 [动作白名单](../contracts/api-reference.md) 与方法签名 |
| 截图很小 | `src/octop_browser/session.py:74-96` | 检查 `BROWSER_USE_VIEWPORT_WIDTH/HEIGHT`；`full_page=True` 仅在归档时使用 |
| 截图报 message too big | `src/octop_browser/cdp/client.py:67-70` | 调高 `BROWSER_USE_CDP_MAX_MESSAGE_SIZE`；`0` 可关闭上限 |

## 4. 录制与回放排障

| 症状 | 检测点 | 处置 |
|---|---|---|
| `Record daemon is not running` | `src/octop_browser/record/daemon.py:152-159` | 执行 `octop-browser record daemon-start` |
| `A recording is already active` | `src/octop_browser/record/daemon.py:44-49` | 先看 `record status`，再 `record stop` |
| 事件发生数远少于操作 | `src/octop_browser/record/recorder.py:265`、`:232-238` | 检查 `BROWSER_RECORD_MAX_EVENT_BYTES`/`MAX_EVENTS_PER_SECOND` 是否过小 |
| `steps.json` 为空 | `src/octop_browser/record/semantic.py:353-370` | 事件 `kind` 不在 16 项白名单；确认录制期间确有交互 |
| `matchedBy=not-found` | `src/octop_browser/record/matcher.py:60-62` | 查看 `failure_*/dom_snapshot.json`，确认页面是否已改版；必要时补 `selectorCandidates` |
| `FAILED: {error}`（MCP 安装） | `src/octop_browser/mcp_server.py:386` | 看聚合日志末尾；区分 pip / CDN / 系统库 / 验证四阶段 |
| `Cannot switch tabs when connected via BROWSER_USE_CDP_WS_URL` | `src/octop_browser/session.py:107-110` | 直连模式不支持切标签，改用独立的 `cdp_ws_url` 指向目标页面 |

回放失败时优先读 `reportPath` 指向的 JSON，再看 `failureArtifacts`：`reasonPath` 给原因，`domSnapshotPath` 给当时结构化 DOM，`screenshotPath` 给视觉证据 (`src/octop_browser/record/replay.py:425-467`)。

## 5. 空闲回收与资源泄漏

`idle_controller_for()` 只在本地 host、无 `cdp_ws_url`、`idle_timeout_minutes>0` 时创建控制器 (`src/octop_browser/idle.py:132-151`)。回收失败时 `reap_now()` 把 `_last_activity` 回拨到 `timeout - 60s`，下一轮再试 (`src/octop_browser/idle.py:86-89`)。若 Chrome 长期残留：确认 `cdp_host` 是否为 loopback、超时是否设置为 0、以及是否有真实 CDP 活动持续刷新 deadline（`send_passive()` 不刷新）。

## 6. 共变清单

新增日志或错误串：同步 [错误码字典](../contracts/error-codes.md) 与本页决策树。改回收/超时逻辑：同步 `tests/unit/test_idle.py`、`tests/unit/test_launcher.py`。生产排障不得打印 `__HBR_RECORDER_CONFIG__` 的 nonce 或任何密钥。
