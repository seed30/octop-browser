---
kind: operations
scope: dev-and-test
status: draft
origin: generated
source_paths: [Makefile, pyproject.toml, .github/workflows/ci.yml, tests/unit/conftest.py, tests/integration/conftest.py, tests/integration/test_session.py, scripts/record_replay_smoke.py]
symbols: [make_all, find_chrome, run_smoke]
upstream: [INDEX.md, architecture/session-core.md]
downstream: [operations/deployment-and-runtime.md, operations/runbook-and-observability.md]
---

# 构建、运行与单测映射

所有开发命令经 Makefile 转发到 `uv`；质量门禁 `make all` 是 format + lint + typecheck + test 的超集，也是 git hook 的预提交动作。

## 1. 环境准备与命令全景

| 命令 | 实际动作 | 锚点 |
|---|---|---|
| `make install` / `install-dev` | `uv sync --extra dev`，无 uv 时 `pip install -e ".[dev]"` | `Makefile:156-163` |
| `make install-hooks` | `git config core.hooksPath .githooks`，提交前自动跑 `make all` | `Makefile:148-154` |
| `make all` | `format lint typecheck test` | `Makefile:120-121` |
| `make format` | `ruff check --fix src tests` + `ruff format src tests`（会改写文件） | `Makefile:130-134` |
| `make lint` | `ruff check src tests` + `ruff format --check src tests` | `Makefile:123-128` |
| `make typecheck` | `mypy src`（strict） | `Makefile:136-139` |
| `make test` | `pytest --cov=octop_browser --cov-report=term-missing` | `Makefile:141-144` |
| `make build` | `uv build --out-dir dist .`，产物 wheel + sdist | `Makefile:64-77` |
| `make mcp` | `python -m octop_browser.mcp_server` | `Makefile:114-116` |
| `make clean` | 清 `dist/`、`build/`、`.mypy_cache`、`.ruff_cache`、`htmlcov` 等 | `Makefile:170-180` |
| `make publish` / `publish-test` | twine 上传 PyPI / TestPyPI，需 `PYPI_TOKEN` | `Makefile:81-106` |

工具约束：Python ≥3.11 (`pyproject.toml:14`)，ruff `line-length=88`、lint 选 `E F I N W` (`:55-59`)，mypy `strict=true`、目标 3.11 (`:73-76`)，pytest `asyncio_mode="auto"`、`testpaths=["tests"]` (`:78-83`)。不要为提高行宽通过而放宽 88。

## 2. 源码模块 → 必跑单测映射

| 源码模块 | 对应单测文件 | 用例数 | 专属关注点 |
|---|---|---|---|
| `cli.py` | `tests/unit/test_cli.py` | 24 | 解析、kwargs 翻译、退出码 |
| `session.py` | `tests/unit/test_session.py`（集成 3） | 3 | 真浏览器会话 |
| `tool_interface.py` | `tests/unit/test_tool_interface.py` | 13 | registry、白名单、重试 |
| `mcp_server.py` | `tests/unit/test_mcp_server.py` | 5 | Tool 列表与分发 |
| `models.py` / `settings.py` / `mode.py` | `test_models.py` / `test_settings.py` / `test_mode.py` | 5 / 16 / 8 | 模型、环境变量、三态 |
| `profile.py` / `idle.py` / `hooks.py` | `test_profile.py` / `test_idle.py` / `test_hooks.py` | 19 / 6 / 5 | 端口、回收、事件 |
| `cdp/launcher.py` / `launch_prep.py` / `client.py` | `test_launcher.py` / `test_launch_prep.py` | 35 / 12 | 启动字典、自愈 |
| `dom/builder.py` / `refs.py` | `test_dom_builder.py` / `test_refs.py` | 6 / 5 | 四级视图、ref 失效 |
| `actions/*` | `test_capture.py` / `test_navigate_actions.py` | 5 / 3 | 截图与导航动作 |
| `install.py` / `playwright_download.py` / `linux_deps.py` | `test_install.py` / `test_playwright_download.py` / `test_linux_deps.py` | 33 / 7 / 11 | 安装流水线 |
| `record/*` | `test_record_replay_semantic.py` / `_replay_runner.py` / `_store_skill.py` / `test_record_daemon_client.py` | 13 / 10 / 5 / 2 | 折叠、回放、产物、IPC |
| 冒烟 | `scripts/record_replay_smoke.py` | 1 流程 | 真 Chrome 端到端 |

## 3. 测试分层与 Mock 隔离规范

| 层级 | 运行方式 | Chrome 依赖 | 锚点 |
|---|---|---|---|
| 单元测试 | `uv run pytest tests/unit/ -q` | 无；全 mock CDPClient/子进程 | `tests/unit/conftest.py` |
| 集成测试 | `uv run pytest tests/integration/ -v` | 真 headless Chrome + CDP，缺失时整组 skip | `tests/integration/conftest.py` |
| CI | PR 触发，Python 3.11/3.12 矩阵，先 `apt-get install chromium-browser` | 有 | `.github/workflows/ci.yml:6-44` |
| 本地冒烟 | `python scripts/record_replay_smoke.py` | 真 Chrome，data:/临时页面，无外部凭证 | `scripts/record_replay_smoke.py:1-7` |

`find_chrome()` 返回 None 时集成测试整组跳过（`tests/integration/conftest.py`），因此无 Chrome 机器上的 `make test` 仍应绿。CI 仅对 PR 运行，并跳过 `chore/sync-develop-after-*` 分支的同步 PR (`.github/workflows/ci.yml:18`)。

## 4. 窄命令与失败定位

```bash
uv run pytest tests/unit/test_mode.py -q
uv run pytest tests/unit/test_settings.py::test_env_override_browser_mode -q
uv run mypy src/octop_browser/session.py
uv run ruff check src/octop_browser/cdp/
```

`make format` 会改写源文件；提交钩子也会自动重新 `git add` 被改写的暂存文件。不要用 `SKIP_PRECOMMIT=1` 或 `--no-verify` 绕过红灯套件。更细的错误串定位见 [错误码字典](../contracts/error-codes.md) 与 [排障 Runbook](runbook-and-observability.md)。

## 5. 共变清单

新增源码模块：建对应 `tests/unit/test_<module>.py` → 更新本页映射表 → 确认 `make all` 全绿。修改环境变量：同步 `tests/unit/test_settings.py` 与 [配置字典](../contracts/configuration.md)。必跑 `make all`；不得提交测试日志、向量结果或会话记录。
