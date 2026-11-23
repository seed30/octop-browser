# 架构与上下游（octop-browser）

> Source: `Makefile`、`pyproject.toml`、`CODEBUDDY.md`、`src/octop_browser/*.py`、`.github/workflows/*.yml`、`tests/integration/conftest.py`
> Last-verified: 2026-11-24（静态核验，未执行构建 / 部署 / 测试命令）

## 1. 组件定位与职责

`octop-browser` 是一个 Python 库 / CLI：直接启动 Chromium，并通过 **Chrome DevTools Protocol（CDP，WebSocket）** 驱动浏览器，运行时不依赖 Playwright / Selenium（`install-browser` 仅一次性借用 Playwright 下载 Chromium 二进制）。运行依赖只有 `websockets`、`aiohttp`、`pydantic`、`mcp`（`pyproject.toml`）。

对外三类入口（同一套 `BrowserSession` 内核）：

| 入口 | 位置 | 说明 |
|------|------|------|
| Python 异步 API | `src/octop_browser/session.py` | `BrowserSession`，混入 `HooksMixin` |
| 无状态工具函数 | `src/octop_browser/tool_interface.py` | `browser_tool(action, profile=..., mode=...)`，按 profile 复用 `_registry: dict[str, BrowserSession]` |
| CLI / MCP | `src/octop_browser/cli.py`、`src/octop_browser/mcp_server.py` | `octop-browser`（`[project.scripts]` → `octop_browser.cli:main`）、MCP stdio 服务（`Server("octop-browser")`，`make mcp`） |

模块分层：`tool_interface` / `cli` / `mcp_server`（入口）→ `session`（`BrowserSession`、`_InternalCDPSession`）→ `cdp`（`CDPClient`、`launcher`）→ Chrome；动作分散在 `actions/`（`navigate`、`interact`、`capture`、`js_eval`），`dom/`（`DOMBuilder`、`RefCache`），`record/`（录制 / 回放 / 技能生成）。

## 2. 关键调用链

```
browser_tool() / cli.py / mcp_server.py
        └─► BrowserSession (session.py)
              └─► _InternalCDPSession（每页一个 CDPClient + RefCache）
                    ├─► cdp.client.CDPClient (websockets) ──► Chrome：Page / DOM / Runtime / Input
                    └─► cdp.launcher：find_chrome → launch_or_attach → get_page_ws_url
```

- `_InternalCDPSession.connect()`（`session.py`）：若设置了 `cfg.cdp_ws_url`（`BROWSER_USE_CDP_WS_URL`），**跳过 launcher**，直接连接指定 WebSocket，用于远程 / 容器内 Chrome。
- `cdp/launcher.py`：管理 Chrome 进程，`find_chrome()` 探测二进制，`launch_or_attach()` 在 CDP 端口已服务时复用已有进程；通过 `aiohttp` 轮询 `http://{cdp_host}:{port}/json/version` 取 `type==page` 目标。HTTP 探测的 host **始终取自 `settings` 的 `cdp_host`，不硬编码 `localhost`**。
- `tool_interface.browser_tool`：动作名经 `getattr` 映射到 `BrowserSession` 方法，`_KNOWN_ACTIONS` 列出 20 个动作（navigate / dom_tree / screenshot / click / … / close_session）。

## 3. 配置与数据边界

- `settings.OctopSettings`（`settings.py`）是唯一配置源，字段均从环境变量读取；常用变量：`BROWSER_USE_CDP_HOST`（默认 localhost）、`BROWSER_USE_CDP_PORT_START`（9222）、`BROWSER_USE_MODE`（auto/headed/headless）、`BROWSER_USE_CHROME_BIN`、`BROWSER_USE_CDP_TIMEOUT`（30.0）、`BROWSER_USE_CDP_WS_URL`、`BROWSER_USE_IDLE_TIMEOUT_MINUTES`（0）。MCP 侧通过客户端 `mcpServers.<name>.env` 传入，不写进工具 schema。
- `profile.ProfileManager`（`profile.py`）：每个 profile 分配一个 Chrome `--user-data-dir`，`cdp_port = cdp_port_start + index`。登录态持久化完全依赖「同名 profile → 同一数据目录」，不另设 cookie / 凭证路径。
- DOM 输出 4 档（`dom/builder.py`：minimal / interactive / full / structured）；`dom/refs.py` 的 `RefCache` 维护 `ref → CDP nodeId`，**任何导航类动作（navigate / go_back / go_forward / reload）必须清空缓存**，否则向调用方返回过期节点。
- `HooksMixin`（`hooks.py`）在每次动作前后触发 `before_action` / `after_action` / `action_error`（`page_navigated` 预留未触发），`models.ActionMetrics` 记录耗时、扫描节点数、估算 token、截图大小。

## 4. 上下游协议与失败影响

- 上游：AI 框架 / CLI / MCP 客户端传入动作与 profile；环境变量决定浏览器启动方式。
- 下游：本机或远程 Chromium 的 CDP WebSocket；无外部数据库、HTTP 服务或第三方账号依赖。
- 失败/退化影响：
  - 缺少 Chrome/Chromium → `tests/integration/conftest.py` 的 `find_chrome()` 为 None 时**整组集成用例 skip**，单元测试保持可用；CI（`ci.yml`、`release.yml`）显式 `apt-get install -y chromium-browser` 后运行 `make test`。
  - CDP 命令超时 / 大截图帧超限 → 由 `BROWSER_USE_CDP_TIMEOUT`、`BROWSER_USE_CDP_MAX_MESSAGE_SIZE`（默认 32 MiB）控制。
  - 远程连接走 `BROWSER_USE_CDP_WS_URL` 直连，绕过本地启动器；此时 launcher 的 Chrome 探测 / `--user-data-dir` 逻辑不生效。
  - 导航后未清 `RefCache` → 动作基于过期 ref 定位元素（已知回归风险）。
- 质量门禁（`Makefile`、`pyproject.toml`、`ci.yml`）：`make all` = format + lint + typecheck + test；`make test` = `uv run pytest --cov=octop_browser --cov-report=term-missing`（`testpaths=["tests"]`，`asyncio_mode="auto"`，`integration` marker）；ruff `line-length=88`、mypy `strict`。发布链路为 `release.yml`：`v*` 标签 → build/test → PyPI（`environment: pypi`、`secrets.PYPI_API_TOKEN`）。

## 5. 维护约定

- 本页正文由源码静态核对得到；`.wally/docs/overview/system-architecture.md` 仍是 `wally-init` 生成的 thin 骨架（`status: draft`，`source_paths: []`），按该文件说明应由 `wally-codewiki` Mode B / 完整 SOP 生成正文，本页不覆盖它。
- 改动启动 / 连接 / DOM 路径时，须同步核对本页「调用链」与 `CODEBUDDY.md` 的 Architecture 一节。
