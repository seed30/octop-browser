---
kind: panorama
scope: business-index
status: draft
origin: generated
source_paths: [src/octop_browser/tool_interface.py, src/octop_browser/session.py, src/octop_browser/record/recorder.py, src/octop_browser/record/replay.py, src/octop_browser/cli.py]
symbols: [browser_tool, BrowserSession, RecordSession, ReplayRunner]
upstream: [INDEX.md, overview/business-architecture.md]
downstream: [business/wf-browser-action.md, business/wf-session-lifecycle.md, business/wf-record-replay.md, business/domain-models/index.md]
---

# 业务流程总表与交叉索引

本仓共有 3 条主业务流程：`WF-01` 单次动作执行、`WF-02` 会话生命周期、`WF-03` 录制回放。三者共享同一动作入口 `browser_tool()`，但触发角色与产物不同。

## 1. 全量工作流清单

| 流程 ID | 名称 | 触发角色 / 触发条件 | 执行形态 | 最终产物 | 详情 |
|---|---|---|---|---|---|
| `WF-01` | 单次浏览器动作执行 | CLI 子命令 / MCP Tool / Python SDK 调用 | 同步，单次 | `ToolResult`，可选 PNG 文件 | [wf-browser-action.md](wf-browser-action.md) |
| `WF-02` | 会话生命周期与登录态持久化 | 首次 `browser_tool` 调用或 `BrowserSession.create()` | 同步建连 + 异步空闲回收 | Chrome 进程、profile 目录、粘滞 target 文件 | [wf-session-lifecycle.md](wf-session-lifecycle.md) |
| `WF-03` | 录制 → 语义折叠 → 技能引导回放 | `record start` / `replay run` | 异步会话 + 守护进程可选 | `manifest.json`/`events.jsonl`/`steps.json`/`draft.skill.md`/`replay_*.json` | [wf-record-replay.md](wf-record-replay.md) |

## 2. 流程间调用关系

```text
WF-03 回放阶段 ──调用 browser_tool()──> WF-01（每一步都复用动作执行链）
WF-03 录制阶段 ──复用 launch_or_attach()──> WF-02 的连接装配
WF-01 / WF-03 ──首次动作触发──> WF-02（profile → 端口 → Chrome → CDP）
WF-02 空闲回收 ──终止 Chrome──> WF-01 后续调用需重连
```

不变量：WF-03 不自行实现点击/输入，必须通过 WF-01 的 `browser_tool()` 执行；WF-01 不管理 Chrome 生命周期，必须经 WF-02 的连接装配。

## 3. 接口、错误码与数据模型交叉索引

| 流程 | 主要入向接口 | 关键错误串 | 相关数据模型 |
|---|---|---|---|
| `WF-01` | [CLI 动作类](../contracts/api/cli-commands.md) · [MCP 动作类](../contracts/api/mcp-tools.md) · [Python SDK](../contracts/api/python-sdk.md) | `Unknown action`、`Ref not found`、`Timeout waiting for condition` | [ToolResult / ActionMetrics](domain-models/toolresult-and-settings.md) |
| `WF-02` | `BrowserSession.create()`、`close-session` | `Chrome did not start`、`Cannot launch Chrome on remote host`、`No open page found` | [OctopSettings / BrowserMode](domain-models/toolresult-and-settings.md) |
| `WF-03` | `record *`（10 子命令）、`replay run`、`browser_record_*` | `Record daemon is not running`、`matchedBy=not-found`、`Unsupported verification` | [录制产物](domain-models/recording-artifacts.md) |

完整错误字面量见 [错误码字典](../contracts/error-codes.md)；术语映射见 [领域术语表](../overview/glossary.md)。

## 4. 全局业务不变量

1. 同一 `profile` 名必须映射到同一 user-data-dir 与 CDP 端口；登录态不另建存储。
2. 任何跨文档导航必须先清 `RefCache`；ref 失效后必须重新 `dom_tree`。
3. 敏感值在 JS 与 Python 双端脱敏为 `{{sensitive_value}}`，不落明文。
4. 回放的是语义意图，不是原始坐标序列；目标匹配阈值 45。
5. `page_navigated` 钩子不发射，不得用于导航感知。

## 5. 共变清单

新增业务流：建 `business/<workflow>.md` → 本页第 1 节登记 → 更新 [业务架构](../overview/business-architecture.md) 的 `WF-*` 总表 → 更新根 INDEX 路由 → 补对应单测。必跑 `make all`。
