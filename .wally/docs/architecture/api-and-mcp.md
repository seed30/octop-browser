---
kind: component
scope: api-and-mcp
status: draft
origin: generated
source_paths: [src/octop_browser/mcp_server.py, src/octop_browser/tool_interface.py, src/octop_browser/__init__.py]
symbols: [app, list_tools, call_tool, browser_tool, _ACTION_MAP]
upstream: [architecture/index.md, INDEX.md]
downstream: [contracts/api/mcp-tools.md, contracts/api/python-sdk.md, architecture/session-core.md]
---

# 组件架构：Python 公共 API 与 MCP 接入层

本组件把同一套浏览器动作以两种协议对外暴露：包根导出的 Python 异步 API，以及 stdio/SSE 上的 MCP Tool 列表。它只做参数分发与结果格式化，不持有 CDP 连接，真正的动作实现落在 [会话核心引擎](session-core.md)。

## 1. 组件定位与框架选型

| 维度 | 实现事实 | 源码锚点 |
|---|---|---|
| 框架模型 | `API Service`（MCP `Server` + 包级 SDK 门面） | `app = Server("octop-browser")` (`src/octop_browser/mcp_server.py:22`) |
| 包根导出面 | 12 个符号，`__all__` 固定排序 | `src/octop_browser/__init__.py:15-28` |
| 无状态分发器 | `_registry: dict[str, BrowserSession]` 按 profile 复用 | `src/octop_browser/tool_interface.py:17` |
| 合法动作白名单 | 20 个动作字面量，`frozenset` | `src/octop_browser/tool_interface.py:20-43` |
| MCP Tool 数 | 25 个 `Tool` 声明 | `src/octop_browser/mcp_server.py:24-347` |
| 工具名映射 | 14 条 `browser_* → action` 映射 | `src/octop_browser/mcp_server.py:349-364` |

## 2. 入向装配与分发调用树

```text
[Python SDK] from octop_browser import browser_tool -> src/octop_browser/tool_interface.py:91 browser_tool(action, profile, ...)
  ├─ [校验] src/octop_browser/tool_interface.py:120 action in _KNOWN_ACTIONS 否则 _error_result()
  ├─ [生命周期] src/octop_browser/tool_interface.py:125 close_session 分支：pop registry + sess.close(kill=)
  ├─ [建会话] src/octop_browser/tool_interface.py:57 _get_or_create_session() -> src/octop_browser/session.py:199 BrowserSession.create()
  ├─ [多标签补偿] src/octop_browser/tool_interface.py:172 urls 已在册时逐个 navigate/new_tab(force_new=True)
  └─ [分发] src/octop_browser/tool_interface.py:180 getattr(sess, action)(**kwargs) -> ToolResult
       └─ [断线重连] src/octop_browser/tool_interface.py:186 _is_disconnected_error() -> _drop_session() -> 重试一次

[MCP stdio] src/octop_browser/mcp_server.py:527 stdio_server() -> app.run()
[MCP SSE] src/octop_browser/mcp_server.py:491 MCP_PORT 存在 -> uvicorn + Starlette /sse + /messages/
  └─ call_tool() src/octop_browser/mcp_server.py:391
       ├─ install_browser -> _handle_install_browser() :372 -> install_chromium_stream()
       ├─ browser_record_* -> RecordController / daemon.send_request() :395-457
       ├─ browser_open -> browser_tool("list_tabs", urls=...) :464
       └─ 其它 -> _ACTION_MAP[name] -> browser_tool(action, **arguments) :475
```

## 3. 参数组装字典与返回解包契约

MCP `arguments` 字典原样透传给 `browser_tool`，没有二级转换层；因此工具 Schema 的每个属性名必须与 `BrowserSession` 方法签名逐字一致。

| 协议 | 入向字典 Key（原文） | 转换规则 | 出向解包 | 源码锚点 |
|---|---|---|---|---|
| MCP Tool | `profile` | 缺省 `"default"`，作为 `_registry` 键 | 无 | `src/octop_browser/mcp_server.py:32`、`:475` |
| MCP Tool | `mode`（仅 `browser_open`） | 透传 `BrowserMode`，仅建会话时生效 | 无 | `src/octop_browser/mcp_server.py:322-327`、`:466-472` |
| MCP Tool | `inputs`（仅 `browser_replay_run`） | `arguments.get("inputs") or {}` 兜底空字典 | `ReplayRunner.run()` 返回 report dict | `src/octop_browser/mcp_server.py:451-457` |
| MCP Tool | `generate_steps`（仅 `browser_record_stop`） | 默认 `True`，写入 daemon 请求字典 | daemon 响应 dict | `src/octop_browser/mcp_server.py:415-424` |
| MCP 成功 | `result.content` | `str` 原样；否则 `str(...)` | `TextContent(type="text", text=...)` | `src/octop_browser/mcp_server.py:476-479` |
| MCP 失败 | `result.success == False` | 文本替换为 `ERROR: {result.error}` | 同上 | `src/octop_browser/mcp_server.py:477-478` |
| Python SDK | `ToolResult` | Pydantic 模型，字段见 [领域模型](../business/domain-models/toolresult-and-settings.md) | 调用方自行判 `success` | `src/octop_browser/models.py:20-33` |
| CLI `--json` | `result.model_dump(mode="json")` | 全量 JSON 序列化 | 输出含 `metrics`/`metadata` | `src/octop_browser/cli.py:502-505` |

## 4. 错误、降级与重试边界

| 故障场景 | 检测条件 | 处理动作 | 锚点 |
|---|---|---|---|
| 未知动作 | `action not in _KNOWN_ACTIONS` | `_error_result()` 返回 `success=False`，错误串含合法动作排序表 | `src/octop_browser/tool_interface.py:120-123` |
| 参数名拼写错误 | `getattr(sess, action)(**kwargs)` 抛 `TypeError` | 转为 `Invalid parameters for '<action>': ...` | `src/octop_browser/tool_interface.py:183-184` |
| CDP 连接丢失 | `CDPSessionError` 文本含 `not connected`/`connection lost` | 丢弃注册表会话并原参数重试一次 | `src/octop_browser/tool_interface.py:83-88`、`:186-210` |
| 会话创建失败 | `_get_or_create_session()` 抛任意异常 | 返回 `Failed to create session: {exc}` | `src/octop_browser/tool_interface.py:167-168` |
| MCP 未知工具 | `_ACTION_MAP.get(name) is None` | 返回纯文本 `Unknown tool: {name}`，非异常 | `src/octop_browser/mcp_server.py:459-461` |
| 记录守护进程未启动 | unix socket 连接失败 | 返回标准 `{"ok": False, "error": ...}` 包络 | `src/octop_browser/record/daemon.py:152-159` |

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 新增一个动作：先在 `session.py` 加方法 → 加入 `src/octop_browser/tool_interface.py:20` 白名单 → 在 `src/octop_browser/mcp_server.py:24` 声明 `Tool` 并补 `_ACTION_MAP` → 在 `src/octop_browser/cli.py:96` 加子命令 → 同步更新 [入向接口总表](../contracts/api-reference.md) 与 [MCP 工具清单](../contracts/api/mcp-tools.md)。
2. 修改动作参数签名：必须同步 MCP `inputSchema.properties`、CLI `_action_kwargs()` 分支与单测 `tests/unit/test_mcp_server.py`、`tests/unit/test_tool_interface.py`。
3. 修改 SDK 导出：同步 `src/octop_browser/__init__.py:15` 的 `__all__` 与两份 Skill 文件 `skills/octop-browser/SKILL.md`、`skills/octop-browser-zh/SKILL.md`。
4. 必跑命令：`uv run pytest tests/unit/test_tool_interface.py tests/unit/test_mcp_server.py tests/unit/test_cli.py -q`。
