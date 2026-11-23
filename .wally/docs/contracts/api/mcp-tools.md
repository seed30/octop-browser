---
kind: api-contract
scope: mcp-tools
status: draft
origin: generated
source_paths: [src/octop_browser/mcp_server.py, src/octop_browser/tool_interface.py]
symbols: [_TOOLS, _ACTION_MAP, list_tools, call_tool, _main]
upstream: [contracts/api-reference.md, architecture/api-and-mcp.md]
downstream: [contracts/error-codes.md, integrations/downstream/record-daemon.md]
---

# MCP Tool 全集与 inputSchema 字典

`src/octop_browser/mcp_server.py:24-347` 静态声明 25 个 `Tool`，`list_tools()` 直接返回该列表 (`src/octop_browser/mcp_server.py:367-369`)。工具之间无优先级，全部由 `call_tool()` 分发。默认 stdio，`MCP_PORT` 存在时切换 SSE。

## 1. 传输与全局约定

| 维度 | 实现 | 锚点 |
|---|---|---|
| 服务名 | `Server("octop-browser")` | `src/octop_browser/mcp_server.py:22` |
| 默认传输 | `stdio_server()` → `app.run()` | `src/octop_browser/mcp_server.py:527-530` |
| SSE 传输 | `MCP_PORT` + `MCP_HOST`（默认 `0.0.0.0`），Starlette 路由 `/sse` 与 `/messages/` POST | `src/octop_browser/mcp_server.py:491-525` |
| 通用参数 | 多数工具含 `profile`（默认 `default`） | 各 Tool 的 `properties` |
| 配置传递 | 环境变量经客户端 `mcpServers.<name>.env` 注入，**不写进 inputSchema** | `src/octop_browser/mcp_server.py:24-347` |
| 响应 | `TextContent(type="text", text=...)` | `src/octop_browser/mcp_server.py:388`、`:479` |

## 2. 浏览器动作类 Tools（14 个）

| Tool 名 | 必填 | inputSchema 属性键（原文） | 落点 | 声明锚点 |
|---|---|---|---|---|
| `browser_navigate` | `url` | `url`、`profile` | `navigate` | `:25-36` |
| `browser_dom_tree` | 无 | `level`（enum 四档，默认 `interactive`）、`profile` | `dom_tree` | `:37-51` |
| `browser_screenshot` | 无 | `profile`、`element_ref`、`full_page`（默认 false）、`path` | `screenshot` | `:52-75` |
| `browser_click` | 无 | `ref`、`selector`、`x`、`y`、`profile` | `click` | `:76-89` |
| `browser_type` | `text` | `text`、`ref`、`selector`、`profile` | `type` | `:90-103` |
| `browser_eval_js` | `expression` | `expression`、`profile` | `eval_js` | `:104-115` |
| `browser_fill` | `text` | `text`、`ref`、`selector`、`profile` | `fill` | `:116-129` |
| `browser_press` | `key` | `key`、`profile` | `press` | `:130-141` |
| `browser_wait` | 无 | `url_contains`、`text`、`selector`、`timeout_ms`（默认 10000）、`profile` | `wait` | `:142-155` |
| `browser_new_tab` | 无 | `url`、`profile` | `new_tab` | `:241-251` |
| `browser_switch_tab` | `tab_id` | `tab_id`、`profile` | `switch_tab` | `:252-263` |
| `browser_close_tab` | 无 | `tab_id`、`profile` | `close_tab` | `:264-274` |
| `browser_list_tabs` | 无 | `profile` | `list_tabs` | `:275-284` |
| `browser_close_session` | 无 | `profile`、`kill`（默认 false） | `close_session` | `:285-303` |

这 14 个经 `_ACTION_MAP`（`src/octop_browser/mcp_server.py:349-364`）转为 action 名后调用 `browser_tool(action=action, **arguments)` (`:475`)。`browser_open` 是特例，见下节。

## 3. 录制、回放、安装类 Tools（11 个）

| Tool 名 | 必填 | inputSchema 属性键（原文） | 分发分支 | 声明 / 处理锚点 |
|---|---|---|---|---|
| `browser_record_doctor` | 无 | `profile` | `RecordController.doctor()` | `:156-163`、`:395-399` |
| `browser_record_start` | 无 | `profile`、`name`、`privacy`（默认 `mask-sensitive`） | `ensure_daemon()` + daemon `start` | `:164-175`、`:400-414` |
| `browser_record_stop` | 无 | `recording_id`、`name`、`generate_steps`（默认 true） | daemon `stop` | `:176-187`、`:415-424` |
| `browser_record_status` | 无 | 空 `properties` | daemon `status` | `:188-192`、`:425-427` |
| `browser_record_list` | 无 | 空 `properties` | `RecordingStore.list_recordings()` | `:193-197`、`:428-436` |
| `browser_record_show` | `recording_id` | `recording_id` | `RecordingStore.recording_summary()` | `:198-206`、`:437-442` |
| `browser_record_generate_steps` | `recording_id` | `recording_id` | `RecordController.generate_steps()` | `:207-215`、`:443-445` |
| `browser_record_generate_skill` | `recording_id` | `recording_id`、`name` | `RecordController.generate_skill()` | `:216-227`、`:446-450` |
| `browser_replay_run` | `recording_id` | `recording_id`、`profile`、`inputs`（object） | `ReplayRunner.run()` | `:228-240`、`:451-457` |
| `browser_open` | `urls` | `urls`（array of string）、`profile`、`mode`（enum 三档，默认 `auto`） | `browser_tool("list_tabs", urls=..., mode=...)` | `:304-331`、`:464-473` |
| `install_browser` | 无 | 空 `properties` | `_handle_install_browser()` → `install_chromium_stream()` | `:332-346`、`:372-394` |

`browser_open` 的 `mode` 属性是 MCP 侧独有的显式参数，SDK `browser_tool()` 也支持但 CLI 用互斥开关表达。`inputs` 对象原样作为 `ReplayRunner.run(inputs=...)` 的参数，缺省 `{}`。

## 4. 请求参数字典到下游的完整映射

| MCP 参数 | 组装去向 | 转换规则 | 锚点 |
|---|---|---|---|
| `arguments` 全体 | `browser_tool(action, **arguments)` | 无二级转换；参数名必须与 `BrowserSession` 方法签名一致 | `:475` |
| `arguments["profile"]` | `_registry` 键 | `arguments.get("profile", "default")` 仅特殊分支显式兜底；普通分支靠 `browser_tool` 默认值 | `:397`、`:407`、`:475` |
| `arguments["privacy"]` | daemon request 的 `privacy` 键 | 默认 `mask-sensitive` | `:410` |
| `arguments["inputs"]` | `ReplayRunner.run(inputs=)` | `arguments.get("inputs") or {}` | `:455` |
| `arguments["generate_steps"]` | daemon request 的 `generate_steps` | 默认 `True` | `:421` |
| `arguments["urls"]` + `mode` | `browser_tool("list_tabs", urls=..., mode=...)` | 创建会话时开多标签，返回标签列表 | `:465-472` |

## 5. 响应解包与错误分支

| 分支 | 成功文本 | 失败文本 | 锚点 |
|---|---|---|---|
| 动作类 | `result.content`（dict 时 `str(...)`） | `ERROR: {result.error}` | `:476-479` |
| `browser_record_*` | `str(data)`，data 为 dict | 仍 `str(data)`，由 `ok` 字段区分 | `:395-450` |
| `browser_replay_run` | `str(report)` | 同上，由 `status` 区分 | `:451-457` |
| `install_browser` | `OK\n\n{聚合日志}` | `FAILED: {error}\n\n{聚合日志}` | `:386-388` |
| 未知工具 | — | `Unknown tool: {name}` | `:460-461` |

## 6. 共变清单

新增工具：在 `_TOOLS` 声明 `Tool` → 若属动作类则补 `_ACTION_MAP`（若新增 action 还需 `src/octop_browser/tool_interface.py:20` 白名单）→ 在 `call_tool()` 加专用分支 → 更新 [总索引](../api-reference.md) → 补 `tests/unit/test_mcp_server.py`。注意环境变量不得写进 `inputSchema`。必跑 `uv run pytest tests/unit/test_mcp_server.py tests/unit/test_tool_interface.py -q`。

- 关联阅读：[入向接口契约全集](../api-reference.md)、[CLI 命令全集](cli-commands.md)。
