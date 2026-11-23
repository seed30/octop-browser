---
kind: operations
scope: deployment-and-runtime
status: draft
origin: generated
source_paths: [pyproject.toml, Makefile, .github/workflows/release.yml, .github/workflows/ci.yml, src/octop_browser/settings.py, src/octop_browser/record/settings.py, src/octop_browser/cdp/launcher.py]
symbols: [state_dir, socket_path, pid_path, launch_or_attach, serve_forever]
upstream: [INDEX.md, contracts/configuration.md]
downstream: [operations/boundary.md, operations/runbook-and-observability.md]
---

# 构建产物、运行期目录与进程边界

本仓发布形态是 PyPI 上的 Python 库 + CLI，不是常驻服务。唯一长期运行的进程是可选的记录守护进程与它启动的 Chrome。

## 1. 构建与发布产物

| 产物 | 生成命令 | 内容 | 锚点 |
|---|---|---|---|
| wheel + sdist | `make build` → `uv build --out-dir dist .` | 打包 `src/octop_browser`，含 `py.typed` | `Makefile:64-77`、`pyproject.toml:52-53` |
| 控制台脚本 | 安装 wheel 后生成 `octop-browser` | 入口 `octop_browser.cli:main` | `pyproject.toml:49-50` |
| PyPI 发布 | `make publish`（`PYPI_TOKEN` 或交互输入） | twine 上传 `dist/*` | `Makefile:81-102` |
| TestPyPI 发布 | `make publish-test` | `PYPI_REPO=testpypi` | `Makefile:104-106` |
| 发版流水线 | `.github/workflows/release.yml` | `secrets.PYPI_API_TOKEN`，`environment: pypi` | `.github/workflows/release.yml` |
| 分支流 | `feature/* → develop → release/x.y.z → main → tag v*` | 见 `CONTRIBUTING.md` | `AGENTS.md:130-138` |

仓库无 Dockerfile/Compose、无部署脚本、无 CI 部署 job；`.wally/config.json` 的 `deploy.readiness` 为 `pending`，`gaps` 已逐条说明，不臆造部署命令。

## 2. 运行期磁盘目录树契约

```text
~/.octop-browser/                    src/octop_browser/settings.py:74-92（旧名 ~/.harness-browser 回退）
├── profiles/
│   ├── .octop-browser-profiles.json src/octop_browser/profile.py:25 端口分配表
│   └── <profile>/
│       ├── .octop-browser-target.json src/octop_browser/profile.py:24 粘滞页面 target
│       └── Chrome user-data-dir（Cookie / localStorage）
├── screenshots/                     src/octop_browser/actions/capture.py:46-59 octop-<ms>.png
└── record-replay/                    src/octop_browser/record/settings.py:24-29
    ├── daemon.sock                   src/octop_browser/record/daemon.py:132-133
    ├── daemon.pid                    src/octop_browser/record/daemon.py:136-137
    └── recordings/<id>/              src/octop_browser/record/store.py:45-62
        ├── manifest.json / events.jsonl / steps.json
        ├── draft.skill.md
        └── screenshots/ snapshots/ replay-reports/
```

权限约束：`profiles_dir` 与录制目录 `0o700`，`events.jsonl` `0o600` (`src/octop_browser/record/store.py:22-25`、`:58-61`)。若 `~/.octop-browser` 不存在而旧目录存在则沿用旧目录，避免登录态与录制产物失联 (`src/octop_browser/settings.py:78-92`)。Linux `/root/...` 路径不可被 Chrome 写入时，启动器会把 profile 迁移到 `/tmp/octop-browser-profiles-<uid>` 并写日志 (`src/octop_browser/cdp/launch_prep.py:144-186`)。

## 3. 网络与 IPC 边界

| 边界 | 地址 / 协议 | 谁监听 | 源码锚点 |
|---|---|---|---|
| CDP HTTP 发现 | `http://{cdp_host}:{cdp_port}/json*` | Chrome | `src/octop_browser/cdp/launcher.py:369`、`:677`、`:695` |
| CDP WebSocket | `ws://{cdp_host}:{cdp_port}/devtools/page/...` | Chrome | `src/octop_browser/cdp/client.py:67` |
| 录制 daemon | Unix socket `<base_dir>/daemon.sock`，单行 JSON | Python 守护进程 | `src/octop_browser/record/daemon.py:114`、`:132` |
| MCP stdio | 标准输入输出 | 本进程 | `src/octop_browser/mcp_server.py:527-530` |
| MCP SSE（可选） | `MCP_HOST:MCP_PORT` + `/sse`、`/messages/` | Starlette/uvicorn | `src/octop_browser/mcp_server.py:491-525` |

`localhost` 永不硬编码，全部经 `cfg.cdp_host`。`cdp_host` 非 loopback 时禁止本仓启动 Chrome，只能连接已有浏览器或设置 `BROWSER_USE_CDP_WS_URL` (`src/octop_browser/cdp/launcher.py:516-523`)。远程 host 不执行 `Browser.close` 与 `pkill` (`src/octop_browser/cdp/launcher.py:452`)。

## 4. 进程生命周期与停机

| 场景 | 行为 | 锚点 |
|---|---|---|
| CLI 每次调用 | attach 到 profile 已占用的 CDP 端口，命令结束进程退出但 Chrome 存活 | `src/octop_browser/cdp/launcher.py:508-514` |
| `close-session` 不带 `--kill` | 只断 WebSocket，Chrome 继续运行 | `src/octop_browser/tool_interface.py:125-161` |
| `close-session --kill` | `Browser.close` → 3s 轮询 → POSIX `pkill` 兜底 → 清 Singleton 锁 | `src/octop_browser/cdp/launcher.py:440-494` |
| 空闲回收 | 本地 profile 空闲 `idle_timeout_minutes` 后由 60s 轮询控制器终止 | `src/octop_browser/idle.py:111-129` |
| daemon 停机 | `shutdown` 先落盘当前录制，再 `call_soon` 关闭 server | `src/octop_browser/record/daemon.py:84-90` |
| 启动脱离终端 | POSIX 下 `start_new_session=True`；macOS headed 不脱离以保持窗口前台 | `src/octop_browser/cdp/launcher.py:566-576` |

## 5. 安装前置与共变清单

运行环境需 Python 3.11+；浏览器二进制由 `octop-browser install-browser` 一次性引导，Playwright 仅作下载器不参与运行时 (`src/octop_browser/install.py:1-41`)。Linux 无 root 时系统库安装前缀 `sudo -n --`，不会等待 TTY (`src/octop_browser/linux_deps.py:190-206`)。

- 改目录规则 → 同步 [配置字典](../contracts/configuration.md)、[会话生命周期](../business/wf-session-lifecycle.md)、`tests/unit/test_profile.py`。
- 改发布流程 → 同步 `Makefile`、`release.yml` 与本页产物表；凭证只从环境变量/secret 读取，不落库不回显。
- 必跑 `make build` 与 `uv run pytest tests/unit/test_profile.py tests/unit/test_settings.py -q`。
