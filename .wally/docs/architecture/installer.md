---
kind: component
scope: chromium-installer
status: draft
origin: generated
source_paths: [src/octop_browser/install.py, src/octop_browser/playwright_download.py, src/octop_browser/linux_deps.py]
symbols: [ensure_chromium, install_chromium_stream, verify_chromium, race_download_sources, build_install_steps]
upstream: [architecture/index.md, architecture/cdp-and-launcher.md]
downstream: [integrations/downstream/playwright-cdn.md, integrations/downstream/os-package-manager.md, contracts/configuration.md]
---

# 组件架构：Chromium 引导安装器

安装器是一次性引导工具，不是运行时依赖：它用 Playwright 的下载能力与包管理器把 Chromium 装进标准缓存，随后由 [CDP 启动器](cdp-and-launcher.md) 接管。运行时永远不 import Playwright，公共导出在 `src/octop_browser/install.py:68-74`。

## 1. 两种 flavours 与统一流水线

| 入口 | 形态 | 适用场景 | 锚点 |
|---|---|---|---|
| `ensure_chromium(quiet=True)` | 同步阻塞，返回 `bool` | CLI `install-browser`、脚本 | `src/octop_browser/install.py:280` |
| `install_chromium_stream(is_disconnected=)` | 异步生成器，逐行产出 `InstallEvent` | MCP `install_browser`、进度 UI | `src/octop_browser/install.py:486` |
| `chromium_executable()` | 查询已安装二进制路径 | 启动兜底、CLI 幂等判断 | `src/octop_browser/install.py:94` |
| `verify_chromium()` | 运行 `<exe> --version` 验证 | 流水线最终门禁 | `src/octop_browser/install.py:230` |

五步流水线在两条路径上一致：① 已有二进制且验证通过则短路；② 缺 `playwright` 包则 pip 引导；③ CDN 竞速后下载 Chromium；④ Linux 系统库与 CJK 字体（尽力而为）；⑤ `--version` 验证决定成败。

## 2. 同步安装调用树

```text
ensure_chromium() src/octop_browser/install.py:280
  ├─ [快路径] chromium_executable() :294 -> verify_chromium() :296 -> True 则直接返回
  │    └─ 已存在但损坏 -> _run_install_deps() :305 -> 再验证 :306
  ├─ _playwright_available() :316（find_spec + invalidate_caches :119）
  │    └─ 缺失且 OCTOP_SKIP_PLAYWRIGHT_PIP 命中 -> 返回 False 并给出手工安装提示 :317-325
  ├─ _run_pip_install() :152 -> uv pip install --python <exe> 优先，python -m pip 兜底 :128-138
  ├─ race_download_sources(build_download_candidates()) :332 -> 按 TTFB 排序的候选列表
  ├─ 逐个候选执行 python -m playwright install chromium :339-351
  ├─ _run_install_deps() :361 -> build_install_steps() :202
  └─ verify_chromium() :364
```

`_multi_pip_bootstrap_enabled()` 由 `OCTOP_MULTI_PIP_BOOTSTRAP` / `HARNESS_MULTI_PIP_BOOTSTRAP` 控制，缺省只试第一条 pip 命令 (`src/octop_browser/install.py:145-156`)。流式版本在每一步 `yield` 日志行，仅最终步骤发 `{"done": True, "success": ...}`，并支持 `is_disconnected` 回调杀掉子进程 (`src/octop_browser/install.py:377-420`)。

## 3. CDN 竞速与下载请求字典

`build_download_candidates()` 按固定顺序产出最多 3 个 `DownloadCandidate`（`src/octop_browser/playwright_download.py:30-80`）：

| kind | host | probe_url | 触发条件 |
|---|---|---|---|
| `custom` | `PLAYWRIGHT_DOWNLOAD_HOST` 去掉首尾斜杠 | `{custom}/` | 设置了该变量且不等于官方别名集合 |
| `npmmirror` | `https://cdn.npmmirror.com/binaries/playwright` | `{mirror}/` | 默认加入，去重后写入 |
| `official` | 空串（由 Playwright 自身默认） | `https://cdn.playwright.dev/` | 永远加入，位于末位 |

探测请求字典（`probe_source()`，`src/octop_browser/playwright_download.py:83-105`）：`Request(url, headers={"Range": "bytes=0-1023"}, method="GET")`，读 16 字节计算 TTFB；HTTP ≥500 抛异常；`HTTPError` 4xx 视为可达。并行度 `min(len(candidates), 3)`，按 TTFB 升序返回；全部失败时回退原始目录顺序 (`:121-148`)。下载子进程环境变量由 `env_for_candidate()` 构造：命中候选时写 `PLAYWRIGHT_DOWNLOAD_HOST`，官方候选则删除该键，并统一附加 `NO_COLOR=1`/`FORCE_COLOR=0`/`NODE_DISABLE_COLORS=1` 防止 ANSI 污染日志 (`:151-172`)。

## 4. 系统依赖步骤字典与验证分支

`build_install_steps()` 依 `detect_package_manager()` 的探测顺序 `apt-get → dnf → yum → zypper → pacman → apk` 生成 `InstallStep`（`src/octop_browser/linux_deps.py:25`、`:182-248`）：

| 管理器 | 库步骤 | CJK 字体步骤 | 特殊处理 |
|---|---|---|---|
| `apt-get` | 先 `playwright install-deps chromium`，再原生 apt 兜底 | `fonts-noto-cjk` | 加 `env DEBIAN_FRONTEND=noninteractive` (`:256`) |
| `dnf` / `yum` | 原生批量安装 16 个库 | 3 个候选字体逐个独立安装 | 不用 `--skip-unavailable`，兼容旧 dnf (`:269-271`) |
| `zypper` / `pacman` / `apk` | 原生批量安装 | 单字体包 | `with_privileges()` 前缀 `sudo -n --`，永不交互 (`src/octop_browser/linux_deps.py:190-206`) |

`install.py:_run_install_deps()` 的规则：必需库步骤首个成功后跳过后续必需步骤；字体步骤仅在库成功后执行，失败不改变返回值 (`:186-222`)。`verify_chromium()` 的返回分支：`rc==0` 成功；`rc==127` 或 stderr 含 `shared libraries` 时提取首个 `.so` 行与 `manual_hint()`；其余错误回显退出码与 stderr 尾部 (`src/octop_browser/install.py:252-272`)。

## 5. 安装事件契约与共变清单

`InstallEvent` 是 `TypedDict(total=False)`，四种键：`log`、`done`、`success`、`error`；终态事件带 `done`，日志事件带 `log`，两者不共存 (`src/octop_browser/install.py:81-91`)。MCP `_handle_install_browser()` 汇总所有 `log` 行，终态拼成 `OK` 或 `FAILED: {error}` (`src/octop_browser/mcp_server.py:372-388`)。

- **共变**：改 CDN 候选或探测头 → 同步 [CDN 契约](../integrations/downstream/playwright-cdn.md) 与 `tests/unit/test_playwright_download.py`。
- **共变**：改包列表或步骤顺序 → 同步 [包管理器契约](../integrations/downstream/os-package-manager.md)、`tests/unit/test_linux_deps.py`、`tests/unit/test_install.py`。
- **共变**：改事件键 → 同步 `src/octop_browser/mcp_server.py:372` 汇总逻辑与 `tests/unit/test_install.py`。
- 必跑命令：`uv run pytest tests/unit/test_install.py tests/unit/test_playwright_download.py tests/unit/test_linux_deps.py -q`。
