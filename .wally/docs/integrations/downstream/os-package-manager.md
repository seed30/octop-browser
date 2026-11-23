---
kind: downstream-contract
scope: linux-package-manager
status: draft
origin: generated
source_paths: [src/octop_browser/linux_deps.py, src/octop_browser/install.py]
symbols: [detect_package_manager, build_install_steps, with_privileges, manual_hint, _run_install_deps]
upstream: [integrations/index.md, architecture/installer.md]
downstream: []
---

# 下游出向接口契约：`Linux 系统包管理器与 CJK 字体`

Chromium 二进制下载完成后，缺失的 `.so` 会让 `--version` 验证以 127 退出。本封装在 Linux 上探测本机包管理器，安装共享库与 CJK 字体；macOS/Windows 直接判定不适用并返回成功。

## 1. 封装定位与探测契约

| 契约项 | 精确值 / 规则 | 源码锚点 |
|---|---|---|
| 本仓封装文件 | `src/octop_browser/linux_deps.py` | 步骤构造器 |
| 探测顺序 | `apt-get → dnf → yum → zypper → pacman → apk`，取 `PATH` 上首个命中 | `src/octop_browser/linux_deps.py:25`、`:182-187` |
| 非 Linux 行为 | `_run_install_deps()` 返回 `True`；流式直接 `done/success=True` | `src/octop_browser/install.py:198-199`、`src/octop_browser/install.py:435-437` |
| 提权方式 | 非 root 时前缀 `sudo -n --`，`-n` 保证不等待 TTY | `src/octop_browser/linux_deps.py:190-206` |
| 步骤模型 | `InstallStep(label, cmd, env, optional)`，`frozen=True` | `src/octop_browser/linux_deps.py:172-180` |

## 2. 本仓上游调用方反查表

| 封装方法 | 上游调用模块 | 所属业务流 | 调用点 |
|---|---|---|---|
| `build_install_steps()` | `install._run_install_deps`、`install._stream_install_deps` | WF-04 安装引导 | `src/octop_browser/install.py:202`、`:439` |
| `manual_hint()` | `verify_chromium`、安装步骤失败日志 | WF-04 | `src/octop_browser/install.py:266`、`src/octop_browser/install.py:441`、`:477` |
| `with_privileges()` | `_native_batch_step`、`_font_steps` | WF-04 | `src/octop_browser/linux_deps.py:259`、`:281` |

## 3. 步骤组装字典与命令键

`build_install_steps(playwright_cmd)` 的返回规则（`src/octop_browser/linux_deps.py:220-248`）：

| 管理器 | 步骤序列 | 必需/可选 | 命令前缀 |
|---|---|---|---|
| `None`（未知） | 仅 `playwright install-deps chromium` | 必需 | 调用方传入的 argv |
| `apt-get` | Playwright 步骤 → 原生 apt 库步骤 → `fonts-noto-cjk` | 必需 / 必需 / 可选 | `["env","DEBIAN_FRONTEND=noninteractive","apt-get","install","-y",...]` |
| `dnf` / `yum` | 原生库步骤 → 3 个字体包各自一步 | 必需 / 可选 | `["dnf","install","-y",...]` |
| `zypper` | 原生库步骤 → `noto-sans-cjk-fonts` | 必需 / 可选 | `["zypper","--non-interactive","install","-y",...]` |
| `pacman` | 原生库步骤 → `noto-fonts-cjk` | 必需 / 可选 | `["pacman","-S","--noconfirm","--needed",...]` |
| `apk` | 原生库步骤 → `font-noto-cjk` | 必需 / 可选 | `["apk","add","--no-cache",...]` |

库包集合固定（16 个）：`apt-get` 用 Debian 名如 `libnss3`/`libatk-bridge2.0-0`/`libasound2`；RPM 系用 `nss`/`nspr`/`at-spi2-atk`/`mesa-libgbm`/`alsa-lib` 等 (`src/octop_browser/linux_deps.py:28-137`)。字体候选在 RPM 系为 `google-noto-sans-cjk-ttc-fonts`/`google-noto-cjk-fonts`/`wqy-zenhei-fonts` 三个独立步骤，刻意不用 dnf 的 `--skip-unavailable`，因为旧 dnf 会拒绝该参数导致全部字体静默跳过 (`src/octop_browser/linux_deps.py:263-285`)。

## 4. 成功判定与错误翻译

`install._run_install_deps()` 的状态机（`src/octop_browser/install.py:186-222`）：

| 变量 | 初值 | 转移条件 | 终态影响 |
|---|---|---|---|
| `libs_ok` | `False` | 任一必需步骤 `returncode == 0` | 决定函数返回值 |
| `fonts_ok` | `False` | 任一可选步骤 `returncode == 0` | 仅决定是否重试字体步骤 |
| 必需步骤 | — | `libs_ok` 已真则 `continue` | 首个成功即锁定 |
| 可选步骤 | — | `fonts_ok` 已真或 `libs_ok` 仍假则 `continue` | 库失败不装字体 |

流式版本 `_stream_install_deps()` 额外产出日志：`Installing {system dependencies|CJK fonts} via {label} ...`、`{label} failed; trying native package manager ...`、`System dependencies installed.` 或 `System deps installation failed. {manual_hint()}`（`src/octop_browser/install.py:423-478`）。字体步骤失败只记录日志，不改变最终 `success`。

`manual_hint(manager)` 生成的手工命令取该管理器前 6 个库名加省略号；未识别管理器时固定返回 `Run: sudo playwright install-deps chromium`（`src/octop_browser/linux_deps.py:209-217`）。

## 5. 共变清单与安全约束

- 改包集合或探测顺序 → 同步 [安装器架构](../../architecture/installer.md)、`tests/unit/test_linux_deps.py` 与 `tests/unit/test_install.py`。
- `with_privileges()` 必须使用 `sudo -n`，禁止改成交互式 sudo；安装器可能挂在 SSE/MCP 请求上，等待 TTY 会挂死。
- 包名与命令来自代码常量，不接受外部输入拼接，避免命令注入。
- 必跑命令：`uv run pytest tests/unit/test_linux_deps.py tests/unit/test_install.py -q`。

- 关联阅读：[配置字典](../../contracts/configuration.md)、[安装器架构](../../architecture/installer.md)。
