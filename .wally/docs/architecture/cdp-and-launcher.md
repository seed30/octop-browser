---
kind: component
scope: cdp-and-launcher
status: draft
origin: generated
source_paths: [src/octop_browser/cdp/client.py, src/octop_browser/cdp/launcher.py, src/octop_browser/cdp/launch_prep.py, src/octop_browser/profile.py, src/octop_browser/idle.py]
symbols: [CDPClient, CDPSessionError, find_chrome, launch_or_attach, get_page_ws_url, terminate_browser, ProfileManager, BrowserIdleController]
upstream: [architecture/index.md, architecture/session-core.md]
downstream: [integrations/downstream/chrome-cdp.md, contracts/configuration.md, operations/runbook-and-observability.md]
---

# 组件架构：CDP 传输底座与进程启动器

本组件负责把「一个 profile 名」变成「一个可用的页面级 CDP WebSocket」，并在空闲或显式关闭时精确回收进程。它是全仓唯一的 `subprocess`/`websockets`/`aiohttp` 出向边界，外部域名与端口一律经 `OctopSettings` 注入。

## 1. 组件构成与框架选型

| 子模块 | 角色 | 核心符号 | 锚点 |
|---|---|---|---|
| `cdp/client.py` | DAO / 传输底座 | `CDPClient`、`CDPSessionError`、`ActivityTracker` | `:30`、`:19`、`:23` |
| `cdp/launcher.py` | 进程启动与发现 | `find_chrome`、`launch_or_attach`、`get_page_ws_url`、`terminate_browser` | `:150`、`:497`、`:618`、`:440` |
| `cdp/launch_prep.py` | 启动前自愈 | `ensure_xdg_runtime_dir`、`ensure_profile_writable`、`clear_profile_locks` | `:63`、`:144`、`:82` |
| `profile.py` | 目录/端口/目标绑定 | `Profile`、`ProfileManager` | `:34`、`:95` |
| `idle.py` | 空闲回收定时 | `BrowserIdleController`、`idle_controller_for` | `:22`、`:132` |

## 2. 连接装配调用树

```text
_InternalCDPSession.connect() src/octop_browser/session.py:49
  └─ launch_or_attach(profile, headless) src/octop_browser/cdp/launcher.py:497
       ├─ [探测] _port_in_use() :433 -> _port_in_use_at() :420 -> GET http://{cdp_host}:{port}/json/version
       ├─ [命中] _get_ws_url() :350 -> 返回 browser 级 WebSocket
       ├─ [拒绝] cdp_host 非 localhost/127.0.0.1 -> RuntimeError("Cannot launch Chrome on remote host") :517-523
       ├─ [查找] find_chrome() :150 -> 显式 BROWSER_USE_CHROME_BIN > 系统路径 > PATH > Playwright 缓存
       ├─ [自愈] prepare_chrome_launch() src/octop_browser/cdp/launch_prep.py:207
       │    ├─ ensure_xdg_runtime_dir() :63 -> /tmp/runtime-octop-browser-<uid>
       │    ├─ ensure_profile_writable() :144 -> /root 下或不可写时迁移到 /tmp/octop-browser-profiles-<uid>
       │    ├─ pkill_chrome_profile() :189（POSIX pkill -9 -f user-data-dir=...）
       │    └─ clear_profile_locks() :82 -> 删除 SingletonLock/Cookie/Socket
       ├─ [构造参数] _build_flags() :190 -> --remote-debugging-port / --user-data-dir / --headless=new / --no-sandbox ...
       ├─ [启动] subprocess.Popen(cmd, start_new_session=...) :578
       └─ [等待] _get_ws_url(process=proc, stderr_file=...) :580 -> 快失败读出 stderr tail :374
  └─ get_page_ws_url(port, profile) src/octop_browser/cdp/launcher.py:618
       ├─ 粘滞 target（profile.load_target()）优先 :653
       ├─ 首个非 about:blank 页面 :663
       ├─ 无页面则 /json/new?about:blank（PUT 优先、GET 兜底） :685-712
       └─ 回写 profile.save_target() :669
```

## 3. CDP 帧协议与活动记账

`CDPClient.send()` 是全仓唯一的指令出口，四件事在同一次调用里完成：

| 步骤 | 实现细节 | 锚点 |
|---|---|---|
| 连接 | `websockets.connect(ws_url, max_size=settings.cdp_max_message_size)`，避免 1009 消息过大 | `src/octop_browser/cdp/client.py:67-70` |
| 编号 | 自增 `_id` 写入 `{"id", "method", "params"}` 并登记 `Future` 到 `_pending` | `src/octop_browser/cdp/client.py:89-96` |
| 超时 | `asyncio.wait_for(future, timeout=self._timeout)`，超时抛 `CDPSessionError("Timeout waiting for response to ...")` | `src/octop_browser/cdp/client.py:99-104` |
| 接收 | 后台 `_recv_loop()` 按 `id` 回填 Future，按 `method` 派发监听器并注入 `__client` | `src/octop_browser/cdp/client.py:152-171` |
| 活动 | `track_activity=True` 时成对调用 `activity_started()/activity_finished()` | `src/octop_browser/cdp/client.py:85-87`、`:110-112` |
| 探测 | `send_passive()` 传 `track_activity=False`，健康探测不续期空闲 deadline | `src/octop_browser/cdp/client.py:118-122` |

`enable_domain("Input")` 不存在对应 CDP 方法，输入域通过 `Input.dispatch*` 直接发指令；`src/octop_browser/session.py:67-68` 对此有明确注释。

## 4. 启动标志与三态开关的精确规则

| 标志 / 行为 | 判定逻辑 | 源码锚点 |
|---|---|---|
| `--remote-debugging-address` | `cdp_host not in ("localhost","127.0.0.1")` 时追加 | `src/octop_browser/cdp/launcher.py:219-220` |
| `--headless=new` | 仅 `headless=True` 时追加 | `src/octop_browser/cdp/launcher.py:221-222` |
| `--no-sandbox` | `BROWSER_USE_NO_SANDBOX` 强制值；否则 Linux 且 (`getuid()==0` 或用户命名空间探测失败) | `src/octop_browser/cdp/launcher.py:283-309`、`:315-347` |
| `--disable-gpu` | `BROWSER_USE_DISABLE_GPU` 强制值；否则等于 `headless` | `src/octop_browser/cdp/launcher.py:252-266` |
| `--disable-dev-shm-usage` | `BROWSER_USE_DISABLE_DEV_SHM_USAGE` 强制值；否则 `sys.platform.startswith("linux")` | `src/octop_browser/cdp/launcher.py:269-280` |
| 自愈重试 | stderr 含 `returncode=21`/`ProcessSingleton`/`SingletonLock` 时调用 `recover_profile()` 重建并重试一次 | `src/octop_browser/cdp/launcher.py:583-613` |
| 空闲回收 | `idle_timeout_minutes>0` 且无 `cdp_ws_url` 且 `cdp_host ∈ {localhost,127.0.0.1,::1}` 才建控制器，60s 轮询 | `src/octop_browser/idle.py:132-151`、`:17-18` |
| 优雅停机 | 先发 `Browser.close`，3s 内端口未释放再 `pkill_chrome_profile()` 并清锁 | `src/octop_browser/cdp/launcher.py:440-494` |

## 5. 端口与目录契约、共变清单

- 端口分配：`cdp_port_start` 起始，跳过已占用值，结果持久化到 `<profiles_dir>/.octop-browser-profiles.json` (`src/octop_browser/profile.py:151-163`)。
- 目标粘滞：页面 target id 写入 `<profile.data_dir>/.octop-browser-target.json`，跨 CLI 进程复用同一标签页 (`src/octop_browser/profile.py:69-84`)。
- 兼容旧名：`.harness-browser-*` 文件作为回退读取，保证历史 profile 不丢端口与目标绑定 (`src/octop_browser/profile.py:29-30`)。
- **共变**：改 CDP 指令字典必须同步 [下游契约](../integrations/downstream/chrome-cdp.md) 与 `tests/unit/test_launcher.py`。
- **共变**：改端口/目录规则必须同步 [配置字典](../contracts/configuration.md)、`tests/unit/test_profile.py`、`tests/unit/test_idle.py`。
- 必跑命令：`uv run pytest tests/unit/test_launcher.py tests/unit/test_launch_prep.py tests/unit/test_profile.py tests/unit/test_idle.py -q`。
