---
kind: component
scope: record-engine
status: draft
origin: generated
source_paths: [src/octop_browser/record/recorder.py, src/octop_browser/record/daemon.py, src/octop_browser/record/semantic.py, src/octop_browser/record/matcher.py, src/octop_browser/record/replay.py, src/octop_browser/record/store.py, src/octop_browser/record/privacy.py, src/octop_browser/record/url_filter.py, src/octop_browser/record/skill_generator.py, src/octop_browser/record/injected/recorder.js]
symbols: [RecordSession, RecordController, RecordDaemon, SemanticProcessor, match_target, ReplayRunner, RecordingStore, SkillGenerator]
upstream: [architecture/index.md, architecture/cli.md]
downstream: [business/wf-record-replay.md, integrations/downstream/record-daemon.md, business/domain-models/recording-artifacts.md]
---

# 组件架构：录制、语义折叠与回放引擎

本组件是独立的异步流水线：浏览器内注入 JS 采集原始事件 → 本地磁盘落盘 → 语义折叠成可重放意图 → 通过 `browser_tool` 重新驱动浏览器。它不是常驻 Worker，而是由 CLI/MCP 触发、可由 Unix socket 守护进程跨命令持有的短生命周期会话，详见 [业务流程 WF-03](../business/wf-record-replay.md)。

## 1. 组件分层与职责边界

| 阶段 | 源码 | 核心符号 | 输出 |
|---|---|---|---|
| 采集注入 | `src/octop_browser/record/recorder.py:152`、`src/octop_browser/record/injected/recorder.js:1` | `_recorder_source()`、`emit()` | `events.jsonl` 原始行 |
| 会话控制 | `src/octop_browser/record/recorder.py:54`、`:304` | `RecordSession`、`RecordController` | `manifest.json` + 步骤 + 草稿 Skill |
| 跨进程守护 | `src/octop_browser/record/daemon.py:18` | `RecordDaemon`、`send_request()` | Unix socket JSON 包络 |
| 折叠与变量推断 | `src/octop_browser/record/semantic.py:333` | `SemanticProcessor` | `steps.json` |
| 目标匹配 | `src/octop_browser/record/matcher.py:19` | `match_target()` | `(ref, why)` 或 `(None, "not-found")` |
| 回放执行 | `src/octop_browser/record/replay.py:21` | `ReplayRunner` | `replay_*.json` + 失败产物 |
| 技能生成 | `src/octop_browser/record/skill_generator.py:19` | `SkillGenerator` | `draft.skill.md` |
| 脱敏与过滤 | `src/octop_browser/record/privacy.py:61`、`src/octop_browser/record/url_filter.py:44` | `redact_event()`、`is_replayable_page_url()` | 脱敏事件 / 可回放 URL 判定 |

## 2. 采集端注入与事件字典

`RecordSession._install_on_client()` 注入流程固定为四步：

```text
src/octop_browser/record/recorder.py:165 _install_on_client(client, target_id)
  ├─ enable_domain("Runtime"/"Page"/"DOM") :166-168
  ├─ Runtime.addBinding({name: binding_name}) :170  （随机 __hbr_emit_<hex>，src/octop_browser/record/recorder.py:72）
  ├─ Page.addScriptToEvaluateOnNewDocument({source: config + recorder.js}) :172
  ├─ Runtime.evaluate({expression: source}) :174  （覆盖已打开页面）
  └─ on("Runtime.bindingCalled") / on("Page.frameNavigated") :176-180
```

JS 侧 `emit()` 发往 Python 的完整字典键（`src/octop_browser/record/injected/recorder.js:55-73`）：

| Key | 类型 | 来源 / 转换 | 锚点 |
|---|---|---|---|
| `nonce` | `str` | `secrets.token_urlsafe(32)` 生成，防伪造注入 (`src/octop_browser/record/recorder.py:73`) | `src/octop_browser/record/injected/recorder.js:61` |
| `schemaVersion` | 字面量 `"1.0"` | 固定值 | `src/octop_browser/record/injected/recorder.js:62` |
| `event.localSeq` | `int` | JS 端自增序号 | `src/octop_browser/record/injected/recorder.js:59` |
| `event.kind` | `str` | `click`/`dblclick`/`input`/`change`/`submit`/`keydown`/`scroll`/`navigation`/`recorder_installed` 等 | `src/octop_browser/record/injected/recorder.js:252-348` |
| `event.ts` | ISO8601 | `new Date().toISOString()` | `src/octop_browser/record/injected/recorder.js:66` |
| `event.targetId` | `str` | 注入配置中的 `targetId` | `src/octop_browser/record/injected/recorder.js:67` |
| `event.page` | dict | `{targetId, url(脱敏), title, viewport{width,height,scrollX,scrollY}}` | `src/octop_browser/record/injected/recorder.js:41-53` |
| `event.target` | dict | `describeElement()` 的 19 个键 | `src/octop_browser/record/injected/recorder.js:201-232` |
| `event.value` | `str` | `readValue()`；敏感字段替换 `{{sensitive_value}}`，超长或命中敏感词用 `{{redacted_value}}` | `src/octop_browser/record/injected/recorder.js:243-249` |
| `event.mouse` / `modifiers` / `scroll` | dict | 鼠标坐标、按键修饰位、滚动位置 | `src/octop_browser/record/injected/recorder.js:254`、`:286`、`:294` |

`describeElement()` 的 19 个键为 `tag`/`role`/`name`/`accessibleName`/`text`/`label`/`placeholder`/`ariaLabel`/`id`/`nameAttr`/`type`/`testId`/`selectorCandidates`/`xpath`/`bbox`/`visible`/`enabled`/`isContentEditable`/`framePath`/`shadowPath`。`selectorCandidates` 最多 8 条并含 `stableCssPath()` 结果 (`src/octop_browser/record/injected/recorder.js:156-174`)。

## 3. 落盘、限流与脱敏规则

`RecordSession._on_binding_called()` 是唯一入库口（`src/octop_browser/record/recorder.py:256-286`）：

| 门禁 | 精确条件 | 不通过时行为 | 锚点 |
|---|---|---|---|
| 会话状态 | `self.active` 且 binding 名匹配 | 直接返回 | `src/octop_browser/record/recorder.py:257` |
| 事件大小 | `len(raw.encode("utf-8")) > max_event_bytes`（默认 65536） | 静默丢弃 | `src/octop_browser/record/recorder.py:265`、`src/octop_browser/record/settings.py:31` |
| 频率 | 每秒计数 `> max_events_per_second`（默认 100） | 静默丢弃 | `src/octop_browser/record/recorder.py:232-238`、`src/octop_browser/record/settings.py:34` |
| 伪造 | `payload["nonce"] != self.nonce` | 静默丢弃 | `src/octop_browser/record/recorder.py:273` |
| 类型 | `payload["event"]` 非 dict 或缺 `kind` 字符串 | 静默丢弃 | `src/octop_browser/record/recorder.py:275-285` |

通过门禁后 `_append()` 统一补 `seq`/`recordingId`/`receivedAt`，调用 `redact_event()`，再追加 `events.jsonl` 并回写 manifest 计数 (`src/octop_browser/record/recorder.py:240-254`)。Python 侧 `redact_event()` 对 `page.url`/`url`/`urlBefore`/`urlAfter` 走 `redact_url()`，对 `value` 按 `input_policy` 决定 `{{sensitive_value}}` 替换 (`src/octop_browser/record/privacy.py:27-81`)；敏感词正则为 `password|token|secret|key|code|otp|auth|session|jwt` (`src/octop_browser/record/privacy.py:11`)。

## 4. 语义折叠与回放执行规则

| 规则 | 精确实现 | 锚点 |
|---|---|---|
| 事件白名单 | 16 种 `kind` 保留，其余丢弃 | `src/octop_browser/record/semantic.py:353-370` |
| 起始 URL | `recorder_installed` 且 `seq<=1` 优先；否则首个带 URL 的事件 | `src/octop_browser/record/semantic.py:372-394` |
| 低信号点击过滤 | 非交互 role/tag 且文本 ≥160 字符，或选择器全含 `nth-of-type` | `src/octop_browser/record/semantic.py:277-316` |
| 变量推断 | `{{sensitive_value}}` → `sensitive_value/secret`；邮箱/搜索词/日期分别推断 `email`/`query`/`date` | `src/octop_browser/record/semantic.py:122-170` |
| 导航期望 | `navigate` 前若紧接 `click`/`press`/`submit`，把 `urlAfter` 与路径提示挂到前一步 `expect` | `src/octop_browser/record/semantic.py:626-648` |
| 目标匹配打分 | selector 80、id 70、nameAttr 60、name 50、role 25、partial-name 20；阈值 45 | `src/octop_browser/record/matcher.py:33-62` |
| 回放动作 | `open`→navigate；`new_tab`→new_tab(force_new)；`click/fill/select/set_checked`→先 `dom_tree(structured)` 再匹配 ref | `src/octop_browser/record/replay.py:53-188` |
| 兜底链 | selector/xpath JS → bbox 坐标点击 → 失败写 `failure_*` 目录并终止 | `src/octop_browser/record/replay.py:469-497` |
| 验证类型 | `url_contains`/`text_contains`/`selector_exists`/`title_contains`/`element_text_contains`/`input_value_equals` | `src/octop_browser/record/replay.py:329-423` |

## 5. 目录产物与共变清单

`RecordingStore` 为每个 `recording_id` 建立固定目录树（`src/octop_browser/record/store.py:45-62`）：`manifest.json`、`events.jsonl`、`steps.json`、`draft.skill.md`/`final.skill.md`，以及 `screenshots/`、`snapshots/`、`replay-reports/` 三个子目录。

- **共变**：改注入事件键 → 同步 `redact_event()`、`semantic.py` 字段读取、`matcher.py` 匹配字段与 `tests/unit/test_record_replay_semantic.py`。
- **共变**：改匹配阈值 → 同步 `tests/unit/test_record_replay_replay_runner.py` 与 [回放契约](../contracts/api/mcp-tools.md)。
- **共变**：改 daemon 命令字 → 同步 `src/octop_browser/cli.py:537` 分支、`src/octop_browser/mcp_server.py:395-457` 与 [守护进程契约](../integrations/downstream/record-daemon.md)。
- 必跑命令：`uv run pytest tests/unit/test_record_replay_semantic.py tests/unit/test_record_replay_replay_runner.py tests/unit/test_record_replay_store_skill.py tests/unit/test_record_daemon_client.py -q`。
