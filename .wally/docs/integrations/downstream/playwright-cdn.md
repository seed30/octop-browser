---
kind: downstream-contract
scope: playwright-cdn-and-pip
status: draft
origin: generated
source_paths: [src/octop_browser/install.py, src/octop_browser/playwright_download.py, src/octop_browser/cdp/launcher.py]
symbols: [build_download_candidates, probe_source, race_download_sources, env_for_candidate, _pip_install_candidates]
upstream: [integrations/index.md, architecture/installer.md]
downstream: []
---

# 下游出向接口契约：`Playwright CDN 与 pip 引导`

安装阶段有两个外部出向目标：Python 包索引（引导 `playwright` 包）与 Playwright Chromium 下载 CDN（官方源、npmmirror 镜像、自定义源）。二者只服务于一次性引导，运行时不产生任何 Playwright 调用，缓存中的二进制由 [启动器](../../architecture/cdp-and-launcher.md) 直接 `subprocess.Popen` 使用。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点 |
|---|---|---|
| 本仓封装文件 | `src/octop_browser/playwright_download.py`（竞速与环境）、`src/octop_browser/install.py`（命令与流水线） | — |
| 官方 CDN | `https://cdn.playwright.dev` | `src/octop_browser/playwright_download.py:16` |
| 镜像 CDN | `https://cdn.npmmirror.com/binaries/playwright` | `src/octop_browser/playwright_download.py:17` |
| 自定义源配置键 | `PLAYWRIGHT_DOWNLOAD_HOST` | `src/octop_browser/playwright_download.py:18` |
| 官方别名集合 | 三个 URL：官方根、`dbazure/download/playwright`、`playwright.download.prss.microsoft.com/dbazure/download/playwright` | `src/octop_browser/playwright_download.py:21-27` |
| 探测超时 | 4.0s，1 KiB Range GET | `src/octop_browser/playwright_download.py:20`、`:83-105` |
| 二进制缓存发现 | `$PLAYWRIGHT_BROWSERS_PATH` / `~/.cache/ms-playwright` / macOS/Windows 对应目录，取最高 `chromium-<rev>` | `src/octop_browser/cdp/launcher.py:84-123` |

## 2. 本仓上游调用方反查表

| 封装方法 | 上游调用模块 / 函数 | 所属业务流 | 调用点 |
|---|---|---|---|
| `build_download_candidates()` | `ensure_chromium`、`install_chromium_stream` | [WF-04 安装引导](../../business/index.md) | `src/octop_browser/install.py:332`、`:576` |
| `race_download_sources()` | 同上 | WF-04 | `src/octop_browser/install.py:332`、`:575-577` |
| `env_for_candidate()` | 同步/流式下载子进程 | WF-04 | `src/octop_browser/install.py:344`、`:588` |
| `_pip_install_candidates()` | `_run_pip_install`、流式 pip 步骤 | WF-04 | `src/octop_browser/install.py:154`、`:549` |
| `_find_playwright_chromium()` | `chromium_executable`、`find_chrome` 末位兜底 | WF-02 / WF-04 | `src/octop_browser/install.py:100`、`src/octop_browser/cdp/launcher.py:187` |

## 3. 候选组装字典与探测规则

`build_download_candidates()` 产出 `DownloadCandidate(kind, host, probe_url)` 三元组，顺序固定为 custom → npmmirror → official，用 `seen` 集合去重 (`src/octop_browser/playwright_download.py:46-80`)。

| kind | host 值 | probe_url | 进入下载命令时的 env 处理 |
|---|---|---|---|
| `custom` | `PLAYWRIGHT_DOWNLOAD_HOST.rstrip("/")` | `{custom}/` | 写回 `PLAYWRIGHT_DOWNLOAD_HOST={host}` |
| `npmmirror` | `https://cdn.npmmirror.com/binaries/playwright` | `{mirror}/` | 写回该 host |
| `official` | 空串 | `https://cdn.playwright.dev/` | `env.pop("PLAYWRIGHT_DOWNLOAD_HOST", None)` |

`probe_source(url, timeout_s)` 的请求字典：

```text
Request(url, headers={"Range": "bytes=0-1023"}, method="GET")
  ├─ urlopen(timeout=4.0) -> read(16) -> 返回 monotonic 差值（TTFB）
  ├─ status >= 500 -> OSError（候选淘汰）
  ├─ HTTPError.code >= 500 -> 抛出（候选淘汰）
  └─ HTTPError 4xx -> 视为可达，返回 TTFB
```

`race_download_sources()` 用 `ThreadPoolExecutor(max_workers=min(n, 3))` 并行探测，按 TTFB 升序返回；全部失败时 `logger.info` 后返回原候选列表，保证下载仍被尝试 (`src/octop_browser/playwright_download.py:121-148`)。

## 4. 下载与 pip 命令字典、返回判定

| 命令 | argv 组装 | env | 成功判定 | 锚点 |
|---|---|---|---|---|
| pip（uv 优先） | `["uv","pip","install","playwright","--python", sys.executable]` | 继承 | `returncode == 0` 且 `_playwright_available()` | `src/octop_browser/install.py:131-137`、`:168` |
| pip（兜底） | `[sys.executable,"-m","pip","install","playwright"]` | 继承 | 同上 | `src/octop_browser/install.py:137` |
| 下载 Chromium | `[sys.executable,"-m","playwright","install","chromium"]` | `env_for_candidate(cand)` | `returncode == 0` | `src/octop_browser/install.py:178-179`、`:339-351` |
| 验证 | `[chromium_exe, "--version"]` | 继承 | `returncode == 0`，取 stdout 版本串 | `src/octop_browser/install.py:241-254` |

`plain_cli_env()` 在 env 中强制写入 `NO_COLOR=1`、`FORCE_COLOR=0`、`NODE_DISABLE_COLORS=1`，避免 Node 安装器把 zip 名包成 ANSI 粗体而污染流式日志 (`src/octop_browser/playwright_download.py:151-162`)。多个 pip 候选是否全部尝试由 `OCTOP_MULTI_PIP_BOOTSTRAP` / `HARNESS_MULTI_PIP_BOOTSTRAP` 决定，缺省仅第一条 (`src/octop_browser/install.py:145-156`)。

## 5. 错误翻译与共变清单

| 失败场景 | 本仓表现 | 锚点 |
|---|---|---|
| Playwright 未安装且 `OCTOP_SKIP_PLAYWRIGHT_PIP` 命中 | 返回 `False` / 流式 `done=False,error=<手工安装提示>` | `src/octop_browser/install.py:317-325`、`:538-545` |
| 所有 CDN 探测失败 | 回退 catalog 顺序继续下载，不直接报错 | `src/octop_browser/playwright_download.py:138-140` |
| 所有下载候选失败 | 同步返回 `False`；流式发 `{"done":True,"success":False,"error":"Chromium download failed"}` | `src/octop_browser/install.py:357-358`、`:600-602` |
| 二进制存在但缺系统库 | `verify_chromium()` 提取首个 `.so` 行 + `manual_hint()` | `src/octop_browser/install.py:257-267` |
| 验证超时 | `Chromium verification timed out`（10s 上限） | `src/octop_browser/install.py:247-248` |

- **共变**：改探测头/超时必须同步 `tests/unit/test_playwright_download.py`。
- **共变**：改 pip 候选顺序或跳过开关必须同步 [配置字典](../../contracts/configuration.md) 与 `tests/unit/test_install.py`。
- **安全**：下载主机仅来自配置或代码常量；不得把 token 拼入 probe_url 或日志。
