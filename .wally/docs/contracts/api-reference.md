---
kind: api-contract
scope: inbound-api
status: draft
origin: generated
source_paths: [src/octop_browser/cli.py, src/octop_browser/mcp_server.py, src/octop_browser/tool_interface.py, src/octop_browser/__init__.py, src/octop_browser/session.py, src/octop_browser/record/daemon.py]
symbols: [_build_parser, _TOOLS, browser_tool, list_tools, call_tool, main]
upstream: [INDEX.md]
downstream: [contracts/api/cli-commands.md, contracts/api/mcp-tools.md, contracts/api/python-sdk.md, contracts/error-codes.md]
---

# 入向接口契约全集（SDK / CLI / MCP）

本仓没有 HTTP 业务路由；对外入口共三类：包根 Python SDK、33 个 CLI 子命令、25 个 MCP Tool。三者最终都汇入 `browser_tool()` 的 20 个动作白名单，或录制/安装的专用分支。接口总数超过 20，按协议拆为三个子页，本页保留公共约定与全量路由总表。

## 1. 公共协议、鉴权与响应包络

| 维度 | 精确实现规则 | 源码锚点 |
|---|---|---|
| 鉴权 | 无内置鉴权；进程级能力，依赖调用方 OS 权限 | `src/octop_browser/cli.py:631`、`src/octop_browser/mcp_server.py:482` |
| profile 归属 | 所有动作以 `profile` 字符串映射到 Chrome user-data-dir，默认 `default` | `src/octop_browser/tool_interface.py:93`、`src/octop_browser/profile.py:165` |
| 统一返回 | 动作类返回 `ToolResult`（`success`/`content`/`error`/`metrics`/`metadata`） | `src/octop_browser/models.py:20-33` |
| CLI 输出 | 默认人类文本；`--json` 输出 `model_dump(mode="json")` | `src/octop_browser/cli.py:502-511` |
| MCP 输出 | `TextContent(type="text", text=...)`；失败前缀 `ERROR: ` | `src/octop_browser/mcp_server.py:476-479` |
| 录制/回放输出 | JSON 字典，`ok` 或 `status` 表示成功 | `src/octop_browser/cli.py:606-607`、`:622-623` |
| 退出码 | 动作/record 成功 0、失败 1；replay 仅 `status=="passed"` 为 0 | `src/octop_browser/cli.py:534`、`:607`、`:623` |
| 传输 | CLI 进程内 `asyncio.run`；MCP stdio 默认，`MCP_PORT` 存在时 SSE | `src/octop_browser/cli.py:645-649`、`src/octop_browser/mcp_server.py:491-530` |

## 2. 全量入向接口路由总表

### 2.1 CLI（33 个）

| 分组 | 子命令 | 落点 action | 详情 |
|---|---|---|---|
| 安装 | `install-browser` | `ensure_chromium()` | [CLI 子命令全集](api/cli-commands.md) |
| 导航 | `navigate`、`open` | `navigate`、`list_tabs`+`urls` | 同上 |
| 页面状态 | `dom-tree`、`screenshot` | `dom_tree`、`screenshot` | 同上 |
| 交互 | `click`、`type`、`fill`、`press`、`wait`、`select`、`scroll`、`hover` | 同名下划线形式 | 同上 |
| JS / 历史 | `eval-js`、`go-back`、`go-forward`、`reload` | `eval_js`、`go_back`、`go_forward`、`reload` | 同上 |
| 标签 | `new-tab`、`switch-tab`、`close-tab`、`list-tabs` | 同名下划线形式 | 同上 |
| 生命周期 | `close-session` | `close_session` | 同上 |
| 录制 | `record doctor`、`daemon-start`、`daemon-stop`、`status`、`start`、`stop`、`list`、`show`、`generate-steps`、`generate-skill` | `RecordController` / daemon IPC | 同上 |
| 回放 | `replay run` | `ReplayRunner.run()` | 同上 |

### 2.2 MCP Tools（25 个）

浏览器动作 14 个（`browser_navigate`/`browser_dom_tree`/`browser_screenshot`/`browser_click`/`browser_type`/`browser_eval_js`/`browser_fill`/`browser_press`/`browser_wait`/`browser_new_tab`/`browser_switch_tab`/`browser_close_tab`/`browser_list_tabs`/`browser_close_session`），录制 8 个，回放 1 个，`browser_open` 1 个，`install_browser` 1 个。逐 Schema 见 [MCP 工具清单](api/mcp-tools.md)。

### 2.3 Python SDK（12 个包根导出）

`BrowserSession`、`browser_tool`、`BrowserMode`、`OctopSettings`、`settings`、`ToolResult`、`ActionMetrics`、`TabInfo`、`InstallEvent`、`chromium_executable`、`ensure_chromium`、`install_chromium_stream`，定义于 `src/octop_browser/__init__.py:15-28`。`BrowserSession` 另暴露 20 个动作方法，见 [Python API](api/python-sdk.md)。

## 3. 动作白名单与特殊分支

`_KNOWN_ACTIONS` 的 20 个字面量（`src/octop_browser/tool_interface.py:20-43`）：`navigate`、`dom_tree`、`screenshot`、`click`、`type`、`fill`、`press`、`wait`、`select`、`scroll`、`hover`、`eval_js`、`go_back`、`go_forward`、`reload`、`new_tab`、`switch_tab`、`close_tab`、`list_tabs`、`close_session`。

三个不经过普通分发的特殊入口：

| 特殊入口 | 行为 | 锚点 |
|---|---|---|
| `close_session` | 从 registry 移除并 `sess.close(kill=)`；无会话但 `kill=True` 时走 `terminate_browser()` | `src/octop_browser/tool_interface.py:125-161` |
| `open` / `browser_open` | 传 `urls` 列表，实际 action 改为 `list_tabs`，创建时开多个标签 | `src/octop_browser/cli.py:520-521`、`src/octop_browser/mcp_server.py:464-473` |
| `install_browser` | 流式安装器，不走 `browser_tool` | `src/octop_browser/mcp_server.py:372-394` |

## 4. 参数校验与错误响应

所有入向参数最终以 `**kwargs` 传给 `BrowserSession` 方法；参数名错误由 Python `TypeError` 捕获并转为 `Invalid parameters for '{action}': {exc}` (`src/octop_browser/tool_interface.py:183-184`)。CLI 侧 `argparse` 自身处理必填/互斥校验，如 `click` 的 `--ref/--selector/--xy` 互斥必填组 (`src/octop_browser/cli.py:202-207`)。MCP 侧依赖 `inputSchema`，无额外服务端参数校验。错误字典详见 [错误码字典](error-codes.md)。

## 5. 共变清单

新增或修改任一入向接口必须同步：实现文件 → 本页总表 → 对应分组子页 → [术语表](../overview/glossary.md) → [API/MCP 组件](../architecture/api-and-mcp.md) → 对应单测。必跑 `uv run pytest tests/unit/test_cli.py tests/unit/test_mcp_server.py tests/unit/test_tool_interface.py -q`。
