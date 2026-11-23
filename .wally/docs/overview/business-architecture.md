---
kind: panorama
scope: business-architecture
status: draft
origin: generated
source_paths: [src/octop_browser/cli.py, src/octop_browser/mcp_server.py, src/octop_browser/tool_interface.py, src/octop_browser/record/recorder.py, src/octop_browser/record/replay.py, src/octop_browser/session.py]
symbols: [browser_tool, RecordSession, ReplayRunner, BrowserSession]
upstream: [INDEX.md]
downstream: [business/index.md, overview/callchain-panorama.md]
---

# 业务架构全景

本仓面向三类触发角色：终端用户（CLI）、AI Agent（MCP）、Python 调用方（SDK）。业务域只有两个——「浏览器即时操作」与「录制回放资产生产」；前者同步返回，后者异步落盘并可在另一时刻重放。

## 1. 触发角色与业务域

| 触发角色 | 入口形态 | 主要业务诉求 | 入口锚点 |
|---|---|---|---|
| 终端用户 | `octop-browser <cmd>`（33 子命令） | 导航、取 DOM、点击、截图、管理会话与标签 | `src/octop_browser/cli.py:96-404` |
| AI Agent | MCP Tool（25 个，stdio/SSE） | 通过工具调用驱动浏览器，获取结构化页面状态 | `src/octop_browser/mcp_server.py:24-347` |
| Python 调用方 | 包根 12 个导出 | 在应用内嵌浏览器自动化 | `src/octop_browser/__init__.py:15-28` |
| 录制操作者 | `record *` / `replay run` | 沉淀可复用的浏览器工作流 | `src/octop_browser/cli.py:320-402` |

| 业务域 | 范围 | 核心对象 | 产出 |
|---|---|---|---|
| 即时操作域 | 导航、DOM、交互、JS、标签、等待、截图 | `BrowserSession`、`ToolResult` | 页面状态变化 + 返回值 |
| 录制资产域 | 采集、脱敏、语义折叠、技能生成、回放 | `RecordSession`、`StepsDocument` | 本机录制目录与报告 |

## 2. 全量工作流总表（WF-*）

| 流程 ID | 名称 | 同步/异步 | 业务不变量 | 文档 |
|---|---|---|---|---|
| `WF-01` | 单次浏览器动作执行 | 同步 | 20 个动作白名单；ref 导航即失效 | [wf-browser-action.md](../business/wf-browser-action.md) |
| `WF-02` | 会话生命周期与登录态持久化 | 建连同步、回收异步 | 同名 profile 对应同一 data_dir 与端口 | [wf-session-lifecycle.md](../business/wf-session-lifecycle.md) |
| `WF-03` | 录制 → 语义折叠 → 技能引导回放 | 异步会话 + 守护进程 | 回放意图而非坐标；双端脱敏 | [wf-record-replay.md](../business/wf-record-replay.md) |

## 3. 全局不变量与红线

1. **动作收口**：所有浏览器动作必须经 `browser_tool()` 的 `_KNOWN_ACTIONS` 白名单（`src/octop_browser/tool_interface.py:20-43`），禁止在 CLI/MCP 直接调 `CDPClient.send()`。
2. **登录态收口**：Cookie/localStorage 只存于 profile 的 user-data-dir（`src/octop_browser/profile.py:169-174`），禁止另建认证文件。
3. **ref 时效**：跨文档导航与标签切换必须清 `RefCache`（`src/octop_browser/actions/navigate.py:28`、`src/octop_browser/session.py:141`），否则会把旧 nodeId 发给新页面。
4. **敏感信息**：录制值经 JS `readValue()` 与 Python `redact_event()` 双端脱敏为 `{{sensitive_value}}`（`src/octop_browser/record/injected/recorder.js:243-249`、`src/octop_browser/record/privacy.py:74-81`），不写明文、不回显、不进日志。
5. **回放语义化**：`match_target()` 先在结构化 DOM 上按分数选 ref（阈值 45，`src/octop_browser/record/matcher.py:60-62`），失败才退 selector/xpath 与 bbox。
6. **配置单一源**：所有主机、端口、目录、超时来自 `OctopSettings`/`RecordReplaySettings`，无内部平台地址硬编码。

## 4. 业务能力到代码的映射

| 用户说法 | 对应能力 | 实现入口 | 关联流程 |
|---|---|---|---|
| "打开某个网页" | `navigate` / `open` | `src/octop_browser/session.py:262`、`src/octop_browser/cli.py:141`、`:147` | WF-01 / WF-02 |
| "看看页面上有什么可点的" | `dom_tree`（四档） | `src/octop_browser/session.py:300`、`src/octop_browser/dom/builder.py:160` | WF-01 |
| "点这个按钮 / 填这个框" | `click`/`type`/`fill`/`select` | `src/octop_browser/actions/interact.py:63`、`:92`、`:167`、`:226` | WF-01 |
| "等页面跳转完成" | `wait` | `src/octop_browser/session.py:491` | WF-01 / WF-03 |
| "把登录状态留到下次" | named profile | `src/octop_browser/profile.py:165` | WF-02 |
| "录一遍操作以后自动跑" | record + replay | `src/octop_browser/record/recorder.py:350`、`src/octop_browser/record/replay.py:25` | WF-03 |
| "把录制变成 Agent Skill" | `generate-skill` | `src/octop_browser/record/skill_generator.py:32` | WF-03 |

## 5. 共变清单

业务规则变更必须同时更新 [业务流程总表](../business/index.md)、对应 `WF-*` 页、[术语表](glossary.md) 与单测。必跑 `make all`。
