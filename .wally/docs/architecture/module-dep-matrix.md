---
kind: panorama
scope: module-dep-matrix
status: draft
origin: generated
source_paths: [src/octop_browser/cli.py, src/octop_browser/mcp_server.py, src/octop_browser/tool_interface.py, src/octop_browser/session.py, src/octop_browser/record/recorder.py, src/octop_browser/record/replay.py, src/octop_browser/cdp/launcher.py, src/octop_browser/install.py]
symbols: [_ACTION_ALIAS, _ACTION_MAP, BrowserSession.create, ReplayRunner.run, launch_or_attach]
upstream: [architecture/index.md, INDEX.md]
downstream: [operations/pitfalls-and-adr.md, integrations/index.md]
---

# 模块依赖矩阵、高扇入底座与环检测

本仓为单仓工作区，`repoDeps` 为空：所有依赖边都在 `src/octop_browser` 内，仓库外只连 Chrome CDP、Playwright CDN、包管理器与本地磁盘。

## 1. 模块级依赖边（上游 → 下游）

| 上游模块 | 下游模块 | 边类型 | 证据锚点 |
|---|---|---|---|
| `cli.py` | `tool_interface.py` | 函数调用 | `browser_tool()` (`src/octop_browser/cli.py:523`) |
| `cli.py` | `record/*` | 函数调用 | `RecordController`/`ReplayRunner`/`RecordingStore` (`src/octop_browser/cli.py:35-39`) |
| `cli.py` | `install.py` | 函数调用 | `ensure_chromium()` (`src/octop_browser/cli.py:642`) |
| `mcp_server.py` | `tool_interface.py` | 函数调用 | `browser_tool()` (`src/octop_browser/mcp_server.py:475`) |
| `mcp_server.py` | `record/*` | 函数调用 | `RecordController`/`ReplayRunner` (`src/octop_browser/mcp_server.py:395-457`) |
| `mcp_server.py` | `install.py` | 函数调用 | `install_chromium_stream()` (`src/octop_browser/mcp_server.py:377`) |
| `tool_interface.py` | `session.py` | 函数调用 | `BrowserSession.create()` (`src/octop_browser/tool_interface.py:64`) |
| `tool_interface.py` | `cdp/launcher.py` | 函数调用 | `terminate_browser()` (`src/octop_browser/tool_interface.py:148`) |
| `session.py` | `actions/*` | 函数调用 | `navigate`/`interact`/`capture`/`js_eval` (`src/octop_browser/session.py:13-14`) |
| `session.py` | `dom/*` | 函数调用 | `DOMBuilder`/`RefCache` (`src/octop_browser/session.py:17-18`) |
| `session.py` | `cdp/client.py`、`cdp/launcher.py` | 函数调用 | `CDPClient`/`launch_or_attach`/`get_page_ws_url` (`src/octop_browser/session.py:15-16`) |
| `session.py` | `profile.py`、`idle.py`、`hooks.py`、`mode.py` | 函数调用 | `src/octop_browser/session.py:19-25` |
| `actions/*` | `cdp/client.py`、`dom/refs.py` | 函数调用 | 各 action 入参签名 |
| `record/recorder.py` | `record/*`（privacy/semantic/store/skill_generator/settings/models） | 函数调用 | `src/octop_browser/record/recorder.py:24-36` |
| `record/recorder.py` | `cdp/launcher.py`、`profile.py` | 函数调用 | `launch_or_attach`/`get_page_ws_url`/`ProfileManager` (`src/octop_browser/record/recorder.py:21-23`) |
| `record/replay.py` | `tool_interface.py` | 函数调用 | `browser_tool()` (`src/octop_browser/record/replay.py:14`、`:56`) |
| `record/daemon.py` | `record/recorder.py` | 函数调用 | `RecordController`/`RecordSession` (`src/octop_browser/record/daemon.py:14`) |
| `install.py` | `playwright_download.py`、`linux_deps.py` | 函数调用 | `src/octop_browser/install.py:58-64` |
| `install.py` | `cdp/launcher.py` | 函数调用 | `_find_playwright_chromium` (`src/octop_browser/install.py:57`) |
| `cdp/launcher.py` | `cdp/launch_prep.py` | 函数调用 | `prepare_chrome_launch`/`recover_profile` (`src/octop_browser/cdp/launcher.py:536-541`、`:599`) |
| `cdp/launcher.py` | `profile.py` | 类型/字段依赖 | `src/octop_browser/cdp/launcher.py:18`、`:653` |
| `cdp/client.py` | `settings.py` | 字段依赖 | `_settings.cdp_timeout`/`cdp_max_message_size` (`src/octop_browser/cdp/client.py:48`、`:69`) |

## 2. 高扇入底座（修改前必须全量回归）

| 底座模块 | 被依赖方数量 | 受影响页面 | 回归命令 |
|---|---|---|---|
| `tool_interface.py` | 4（cli、mcp、replay、idle 间接） | [API/MCP](api-and-mcp.md)、[回放](record-engine.md) | `uv run pytest tests/unit/test_tool_interface.py -q` |
| `cdp/client.py` | 5（session、actions、recorder、idle、replay 间接） | [CDP 底座](cdp-and-launcher.md)、[下游契约](../integrations/downstream/chrome-cdp.md) | `uv run pytest tests/unit/test_launcher.py -q` |
| `settings.py` | 8+ | [配置字典](../contracts/configuration.md)、全组件页 | `uv run pytest tests/unit/test_settings.py -q` |
| `record/store.py` | 5（cli、mcp、recorder、replay、daemon） | [录制引擎](record-engine.md)、[产物模型](../business/domain-models/recording-artifacts.md) | `uv run pytest tests/unit/test_record_replay_store_skill.py -q` |
| `profile.py` | 3（session、tool_interface、recorder） | [会话生命周期](../business/wf-session-lifecycle.md) | `uv run pytest tests/unit/test_profile.py -q` |
| `models.py` | 6+ | [工具结果模型](../business/domain-models/toolresult-and-settings.md)、契约页 | `uv run pytest tests/unit/test_models.py -q` |

## 3. 环检测与反向依赖约束

```text
无编译期强环：依赖方向为 接入层 -> 编排层 -> 核心引擎 -> 传输底座
受控反向边（函数内延迟 import，避免导入期环）：
  src/octop_browser/idle.py:75-76      -> cdp.launcher.terminate_browser + tool_interface._drop_session
  src/octop_browser/tool_interface.py:141 -> cdp.launcher.terminate_browser（close_session kill 分支）
  src/octop_browser/session.py:156     -> cdp.launcher.terminate_browser（close kill 分支）
  src/octop_browser/cdp/launcher.py:536    -> cdp.launch_prep.prepare_chrome_launch（运行时延迟）
  src/octop_browser/record/replay.py:14 -> tool_interface.browser_tool（回放复用动作入口）
```

铁律：新增跨层反向依赖必须使用函数内延迟 import（`# noqa: PLC0415` 风格），不得在模块顶层 import 造成导入期环。

## 4. 跨仓依赖矩阵

`repoDeps = []`。本仓 README 列出 `harness-agent`、`harness-memory`、`harness-gateway`、`Octop` 四个相关项目（`README.md:386-393`），但源码中没有任何指向它们的 import、HTTP 调用或 RPC 客户端；按零幻觉规则不生成跨仓边。

## 5. 共变清单

- 新增模块依赖：在本页第 1 节补边并附 `path:line` 证据；无证据不记。
- 修改高扇入底座：按第 2 节逐项回归，并更新 [高危雷区](../operations/pitfalls-and-adr.md)。
- 必跑 `make all`。
