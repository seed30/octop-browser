---
kind: domain-model
scope: toolresult-and-settings
status: draft
origin: generated
source_paths: [src/octop_browser/models.py, src/octop_browser/settings.py, src/octop_browser/mode.py, src/octop_browser/session.py, src/octop_browser/tool_interface.py]
symbols: [ToolResult, ActionMetrics, TabInfo, OctopSettings, settings, BrowserMode]
upstream: [business/domain-models/index.md, architecture/api-and-mcp.md]
downstream: [contracts/configuration.md, contracts/api/python-sdk.md]
---

# 领域模型字段字典：`ToolResult` 与 `OctopSettings`

本页展开两条最热的调用数据：所有动作的返回值 `ToolResult`，以及全仓唯一配置源 `OctopSettings`。前者决定 CLI/MCP/SDK 的成功语义，后者决定端口、目录、超时与启动行为。

## 1. `ToolResult` 字段字典

定义于 `src/octop_browser/models.py:20-33`，每个 `BrowserSession` 方法与 `browser_tool()` 都返回它。

| 字段 | 类型 | 默认值 | 构造与消费规则 | 锚点 |
|---|---|---|---|---|
| `success` | `bool` | 必填 | 失败时 CLI 写 stderr 并返回退出码 1；MCP 改写成 `ERROR: {error}` | `src/octop_browser/models.py:29`、`src/octop_browser/cli.py:506-534`、`src/octop_browser/mcp_server.py:477-478` |
| `content` | `str \| dict[str,object]` | 必填 | DOM 文本、截图路径、JS 返回值；MCP 统一 `str(...)` | `src/octop_browser/models.py:30`、`src/octop_browser/mcp_server.py:476` |
| `error` | `str \| None` | `None` | 成功时必须是 `None`；`_error_result()` 总是填错误串 | `src/octop_browser/models.py:31`、`src/octop_browser/tool_interface.py:46-54` |
| `metrics` | `ActionMetrics` | 必填 | 不允许缺省；错误结果用全 0 的 metrics 占位 | `src/octop_browser/models.py:32`、`src/octop_browser/tool_interface.py:51-53` |
| `metadata` | `dict[str,object] \| None` | `None` | 截图带 `path/url/title/full_page/element_ref/width/height/size_kb`；`list_tabs` 带 `tabs[]` | `src/octop_browser/models.py:33`、`src/octop_browser/actions/capture.py:206-215`、`src/octop_browser/session.py:846-856` |

`ActionMetrics` 五个字段（`src/octop_browser/models.py:10-17`）：`action`、`duration_ms`、`dom_nodes_scanned`、`estimated_tokens`、`screenshot_size_kb=0`。交互类动作 `_metrics()` 固定 `dom_nodes_scanned=1`、`estimated_tokens=10` (`src/octop_browser/actions/interact.py:12-18`)；导航类固定 `0`/`20` (`src/octop_browser/actions/navigate.py:16-22`)；文本类用 `len(content)//4` 动态计算。

## 2. `ToolResult` 构造路径与不变量

```text
动作成功 -> ToolResult(success=True, content=..., metrics=...)     例 src/octop_browser/session.py:334
动作失败 -> ToolResult(success=False, content="", error=..., metrics=...) 例 src/octop_browser/session.py:342-344
参数/动作非法 -> _error_result() errors 串 + 全 0 metrics   src/octop_browser/tool_interface.py:46-54
记账 -> _record() 累加 action_count/duration/tokens/errors  src/octop_browser/session.py:895-903
汇总 -> metrics_summary() -> {total_actions, total_duration_ms, total_estimated_tokens, errors}  :875-881
```

不变量：不要绕过 `_record()` 直接返回 `ToolResult`，否则 `metrics_summary()` 会漏计；新增动作若自行构造结果，必须在返回前调用 `await self._record(result)`。

## 3. `TabInfo` 与标签元数据

`TabInfo` 四字段 `tab_id`/`url`/`title`/`active` (`src/octop_browser/models.py:36-43`)。`list_tabs()` 从 `GET /json` 过滤 `type == "page"`，active 判定优先取 profile 粘滞 target，若该 id 不在页面列表则回退第一个页面 (`src/octop_browser/session.py:810-819`)。顶层 `content` 是 `[{tab_id}] {title} — {url}` 逐行文本，结构化副本放在 `metadata["tabs"]` (`src/octop_browser/session.py:830-856`)。

## 4. `OctopSettings` 字段与覆盖优先级

`OctopSettings` 是 dataclass，每个字段用 `field(default_factory=lambda: _env_*())`，因此每次实例化都重读环境变量；模块级 `settings = OctopSettings()` 在导入时读取一次 (`src/octop_browser/settings.py:109-185`)。

| 字段 | 类型 | 环境变量 | 默认值 | 主要 Reader |
|---|---|---|---|---|
| `profiles_dir` | `Path` | `BROWSER_USE_PROFILES_DIR` | `state_dir()/profiles` | `src/octop_browser/profile.py:116` |
| `screenshots_dir` | `Path` | `BROWSER_USE_SCREENSHOTS_DIR` | `state_dir()/screenshots` | `src/octop_browser/actions/capture.py:51` |
| `cdp_host` | `str` | `BROWSER_USE_CDP_HOST` | `localhost` | `src/octop_browser/cdp/launcher.py:369`、`:677`；`src/octop_browser/session.py:121` |
| `cdp_port_start` | `int` | `BROWSER_USE_CDP_PORT_START` | `9222` | `src/octop_browser/profile.py:158` |
| `chrome_bin` | `str \| None` | `BROWSER_USE_CHROME_BIN` | `None`（自动探测） | `src/octop_browser/cdp/launcher.py:163` |
| `cdp_timeout` | `float` | `BROWSER_USE_CDP_TIMEOUT` | `30.0` | `src/octop_browser/cdp/client.py:48` |
| `launch_retries` | `int` | `BROWSER_USE_LAUNCH_RETRIES` | `20` | `src/octop_browser/cdp/launcher.py:367` |
| `launch_delay` | `float` | `BROWSER_USE_LAUNCH_DELAY` | `0.25` | `src/octop_browser/cdp/launcher.py:368` |
| `cdp_ws_url` | `str \| None` | `BROWSER_USE_CDP_WS_URL` | `None` | `src/octop_browser/session.py:55` |
| `idle_timeout_minutes` | `float` | `BROWSER_USE_IDLE_TIMEOUT_MINUTES` | `0.0` | `src/octop_browser/idle.py:137` |
| `viewport_width` / `viewport_height` | `int` | `BROWSER_USE_VIEWPORT_*` | `1440` / `900` | `src/octop_browser/session.py:83-84` |
| `cdp_max_message_size` | `int \| None` | `BROWSER_USE_CDP_MAX_MESSAGE_SIZE` | `33554432`，`0`→`None` | `src/octop_browser/cdp/client.py:69` |
| `browser_mode` | `BrowserMode` | `BROWSER_USE_MODE` | `auto` | `src/octop_browser/mode.py:26-40` |

覆盖优先级：显式传给 `BrowserSession.create(settings=...)` 的对象 > 实例化 `OctopSettings()` 时读取的环境变量 > 模块级单例（导入时快照）> 代码默认值。`monkeypatch.setenv` 在测试中依赖「每次实例化重读」这一行为。

## 5. 共变清单

- 改 `ToolResult` 字段 → 同步 CLI `_format_result()`、MCP `call_tool()`、回放报告构造与 `tests/unit/test_models.py`。
- 改 `OctopSettings` 字段 → 同步 [配置全集](../../contracts/configuration.md)、`tests/unit/test_settings.py` 与 README 配置表。
- 改 token 估算口径 → 同步 `dom_tree`、`eval_js`、`list_tabs` 三处 `//4`。
- 必跑：`uv run pytest tests/unit/test_models.py tests/unit/test_settings.py tests/unit/test_tool_interface.py -q`。

- 关联阅读：[入向接口契约全集](../../contracts/api-reference.md)、[系统技术架构](../../overview/system-architecture.md)。
