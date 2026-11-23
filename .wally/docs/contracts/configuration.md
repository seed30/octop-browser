---
kind: config-dict
scope: configuration
status: draft
origin: generated
source_paths: [src/octop_browser/settings.py, src/octop_browser/mode.py, src/octop_browser/cdp/launcher.py, src/octop_browser/cdp/launch_prep.py, src/octop_browser/install.py, src/octop_browser/linux_deps.py, src/octop_browser/playwright_download.py, src/octop_browser/record/settings.py, src/octop_browser/mcp_server.py]
symbols: [OctopSettings, _env_tristate, state_dir, RecordReplaySettings, build_download_candidates, _main]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [operations/deployment-and-runtime.md, architecture/installer.md]
---

# 配置项、环境变量与动态开关全集

全仓配置只有一个 dataclass 源 `OctopSettings`（浏览器）与一个 `RecordReplaySettings`（录制），其余是安装器与 MCP 的进程级开关。所有进程环境变量均为可选，未设置时走代码默认值。

## 1. 覆盖优先级链

`显式传给 BrowserSession.create(settings=...)` > `实例化 OctopSettings()/RecordReplaySettings() 时读环境变量` > `模块级单例（导入时快照）` > `代码硬编码默认值`。

关键点：每个字段用 `field(default_factory=lambda: _env_*())`，所以每次实例化都会重读环境，这正是测试中 `monkeypatch.setenv` 生效的原因；模块级 `settings = OctopSettings()` 在导入时读取一次 (`src/octop_browser/settings.py:109-185`)。录制侧同理 (`src/octop_browser/record/settings.py:22-38`)。

## 2. 浏览器核心环境变量（17 项）

| 环境变量 | 类型 | 默认值 | 控制行为 / 边界 | 读取锚点 |
|---|---|---|---|---|
| `BROWSER_USE_PROFILES_DIR` | path | `state_dir()/profiles` | Chrome user-data-dir 根目录 | `src/octop_browser/settings.py:128-132` |
| `BROWSER_USE_SCREENSHOTS_DIR` | path | `state_dir()/screenshots` | 截图 PNG 输出根目录 | `src/octop_browser/settings.py:133-138` |
| `BROWSER_USE_CDP_HOST` | str | `localhost` | HTTP/WS 发现主机；非 loopback 禁止启动进程 | `src/octop_browser/settings.py:139-143`、`src/octop_browser/cdp/launcher.py:517-523` |
| `BROWSER_USE_CDP_PORT_START` | int | `9222` | profile 起始调试端口 | `src/octop_browser/settings.py:144-146` |
| `BROWSER_USE_CHROME_BIN` | path | 自动探测 | 显式二进制路径，优先于系统路径搜索 | `src/octop_browser/settings.py:147-149`、`src/octop_browser/cdp/launcher.py:163-164` |
| `BROWSER_USE_CDP_TIMEOUT` | float | `30.0` | 单条 CDP 指令等待秒数 | `src/octop_browser/settings.py:150-152` |
| `BROWSER_USE_LAUNCH_RETRIES` | int | `20` | `/json/version` 轮询次数 | `src/octop_browser/settings.py:153-155`、`src/octop_browser/cdp/launcher.py:367` |
| `BROWSER_USE_LAUNCH_DELAY` | float | `0.25` | 轮询间隔秒 | `src/octop_browser/settings.py:156-158`、`src/octop_browser/cdp/launcher.py:368` |
| `BROWSER_USE_CDP_WS_URL` | str | 未设置 | 直连远程 Chrome，完全绕过 launcher 与切标签 | `src/octop_browser/settings.py:159-161`、`src/octop_browser/session.py:55-57` |
| `BROWSER_USE_IDLE_TIMEOUT_MINUTES` | float | `0.0`（关闭） | 本地 profile 空闲回收阈值 | `src/octop_browser/settings.py:162-164`、`src/octop_browser/idle.py:137-139` |
| `BROWSER_USE_VIEWPORT_WIDTH` | int | `1440` | `Emulation.setDeviceMetricsOverride` 宽度 | `src/octop_browser/settings.py:165-167` |
| `BROWSER_USE_VIEWPORT_HEIGHT` | int | `900` | 同上高度 | `src/octop_browser/settings.py:168-170` |
| `BROWSER_USE_CDP_MAX_MESSAGE_SIZE` | int | `33554432`；`0`→`None` | WebSocket 帧上限；`None` 表示不设限 | `src/octop_browser/settings.py:171-177`、`src/octop_browser/cdp/client.py:67-70` |
| `BROWSER_USE_NO_SANDBOX` | 三态 | 自动探测 | 强制/禁止 `--no-sandbox` | `src/octop_browser/cdp/launcher.py:302-309` |
| `BROWSER_USE_DISABLE_GPU` | 三态 | headless 时开 | 强制/禁止 `--disable-gpu` | `src/octop_browser/cdp/launcher.py:263-266` |
| `BROWSER_USE_DISABLE_DEV_SHM_USAGE` | 三态 | Linux 时开 | 强制/禁止 `--disable-dev-shm-usage` | `src/octop_browser/cdp/launcher.py:277-280` |
| `BROWSER_USE_MODE` | enum | `auto` | `auto`/`headed`/`headless`，经 `normalize_mode()` 校验 | `src/octop_browser/settings.py:178-180`、`src/octop_browser/mode.py:26-40` |

三态解析 `_env_tristate()`：`1/true/yes/on` → True，`0/false/no/off` → False，未设置或不可识别 → `None`（回退默认值）(`src/octop_browser/cdp/launcher.py:235-249`)。`BROWSER_USE_NO_SANDBOX` 单独用简化解析，未设置时在 Linux 下走 `getuid()==0 || !_user_namespaces_available()` (`src/octop_browser/cdp/launcher.py:302-309`)。

## 3. 录制与 MCP 环境变量（5 项）

| 环境变量 | 类型 | 默认值 | 控制行为 | 读取锚点 |
|---|---|---|---|---|
| `BROWSER_RECORD_REPLAY_DIR` | path | `state_dir()/record-replay` | 录制产物根目录，同时决定 daemon.sock/pid 位置 | `src/octop_browser/record/settings.py:24-29`、`src/octop_browser/record/daemon.py:132-137` |
| `BROWSER_RECORD_MAX_EVENT_BYTES` | int | `65536` | 单条事件 UTF-8 字节上限，超限静默丢弃 | `src/octop_browser/record/settings.py:30-32`、`src/octop_browser/record/recorder.py:265` |
| `BROWSER_RECORD_MAX_EVENTS_PER_SECOND` | int | `100` | 每秒事件数上限 | `src/octop_browser/record/settings.py:33-35`、`src/octop_browser/record/recorder.py:232-238` |
| `MCP_PORT` | int | 未设置 | 设置后 MCP 从 stdio 切换为 SSE HTTP | `src/octop_browser/mcp_server.py:491-525` |
| `MCP_HOST` | str | `0.0.0.0` | SSE 监听主机，仅 `MCP_PORT` 存在时生效 | `src/octop_browser/mcp_server.py:499` |

## 4. 安装器与间接环境变量（11 项）

| 环境变量 | 读取 / 写入 | 默认值 | 作用 | 锚点 |
|---|---|---|---|---|
| `OCTOP_SKIP_PLAYWRIGHT_PIP` | 读 | 未设置 | 为真时跳过 pip 引导 playwright | `src/octop_browser/install.py:76`、`:123-125` |
| `HARNESS_SKIP_PLAYWRIGHT_PIP` | 读（旧名兼容） | 未设置 | 同上 | `src/octop_browser/install.py:78`、`:124` |
| `OCTOP_MULTI_PIP_BOOTSTRAP` | 读 | 未设置 | 为真时依次尝试所有 pip 候选 | `src/octop_browser/install.py:145-149` |
| `HARNESS_MULTI_PIP_BOOTSTRAP` | 读（旧名兼容） | 未设置 | 同上 | `src/octop_browser/install.py:146-148` |
| `PLAYWRIGHT_DOWNLOAD_HOST` | 读 + 子进程写 | 未设置 | 自定义 CDN 主机；官方候选时从 env 中删除 | `src/octop_browser/playwright_download.py:18`、`:51`、`:165-172` |
| `PLAYWRIGHT_BROWSERS_PATH` | 读 | 未设置 | Playwright 浏览器缓存根 | `src/octop_browser/cdp/launcher.py:95-98` |
| `DISPLAY` / `WAYLAND_DISPLAY` | 读 | 未设置 | `auto` 模式的桌面环境判定 | `src/octop_browser/mode.py:52`、`:72-73` |
| `XDG_RUNTIME_DIR` | 写 | 由启动器强制覆盖 | Linux 启动前指向 `/tmp/runtime-octop-browser-<uid>` | `src/octop_browser/cdp/launch_prep.py:63-79`、`:217-218` |
| `NO_COLOR` / `FORCE_COLOR` / `NODE_DISABLE_COLORS` | 写 | `1`/`0`/`1` | 子进程禁用 ANSI 输出 | `src/octop_browser/playwright_download.py:158-162` |
| `DEBIAN_FRONTEND` | 写 | `noninteractive` | apt-get 安装不交互 | `src/octop_browser/linux_deps.py:256`、`:277` |
| `PROGRAMFILES` / `PROGRAMFILES(X86)` / `LOCALAPPDATA` / `USERNAME` / `USER` | 读 | 平台相关 | Windows 路径探测与临时目录 scope token | `src/octop_browser/cdp/launcher.py:102-104`、`:137-147`；`src/octop_browser/cdp/launch_prep.py:49-52` |

## 5. 凭据、下游端点与安全规则

| 目标 | 配置来源 | 凭据处理 | 未配置时行为 | 锚点 |
|---|---|---|---|---|
| Chrome CDP | `cdp_host`/`cdp_port_start`/`cdp_ws_url` | 无凭据；远程访问靠外部保障 | 本地自动启动；远程 host 必须自带 `cdp_ws_url` | `src/octop_browser/cdp/launcher.py:497-523` |
| Playwright CDN | `PLAYWRIGHT_DOWNLOAD_HOST` | 不含 token | 官方源兜底 | `src/octop_browser/playwright_download.py:46-80` |
| PyPI | Makefile `PYPI_TOKEN`（交互式输入）/ CI `secrets.PYPI_API_TOKEN` | 不落库、不回显、不写日志 | 提示输入后继续 | `Makefile:88-101`；`.github/workflows/release.yml` |
| MCP SSE | `MCP_PORT`/`MCP_HOST` | 无内置鉴权 | 不设 `MCP_PORT` 时走 stdio | `src/octop_browser/mcp_server.py:491-527` |

安全铁律：本仓不包含任何 `*.woa.com` 内部地址硬编码；所有主机与端口都从上述配置项读取。密钥不明文落库、不回显、不写日志。

- 关联阅读：[部署与运行时](../operations/deployment-and-runtime.md)、[领域术语与符号映射](../overview/glossary.md)。
