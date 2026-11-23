---
kind: domain-model
scope: recording-artifacts
status: draft
origin: generated
source_paths: [src/octop_browser/record/models.py, src/octop_browser/record/store.py, src/octop_browser/record/semantic.py, src/octop_browser/record/replay.py, src/octop_browser/record/settings.py]
symbols: [RecordingManifest, StepsDocument, SemanticStep, TargetDescriptor, PrivacySettings, RecordingStore]
upstream: [business/domain-models/index.md, business/wf-record-replay.md]
downstream: [architecture/record-engine.md, contracts/api/cli-commands.md]
---

# 领域模型字段字典：录制与回放产物

录制链路把浏览器事件逐层压缩成四类磁盘产物：原始事件、清单、语义步骤、技能草案，再加回放报告。所有文件由 `RecordingStore` 统一读写，路径规则集中且可枚举。

## 1. 磁盘目录树契约

```text
<base_dir>/recordings/<recording_id>/          src/octop_browser/record/store.py:28-29
├── manifest.json                              :31-32  原子替换写
├── events.jsonl                               :34-35  追加写 + flush
├── steps.json                                 :37-38
├── draft.skill.md | final.skill.md            :40-43
├── screenshots/                               :49
├── snapshots/                                 :49
└── replay-reports/
    ├── replay_<YYYYmmdd_HHMMSS>.json           src/octop_browser/record/replay.py:647-657
    └── failure_<ts>_<step_id>/                 src/octop_browser/record/replay.py:434-466
        ├── reason.txt
        ├── dom_snapshot.json
        ├── screenshot.png | screenshot_error.txt
```

`base_dir = BROWSER_RECORD_REPLAY_DIR or state_dir()/record-replay` (`src/octop_browser/record/settings.py:24-29`)。目录权限 `0o700`，`events.jsonl` 权限 `0o600` (`src/octop_browser/record/store.py:22-25`、`:58-61`)。

## 2. `manifest.json` 字段字典

`RecordingManifest` 定义于 `src/octop_browser/record/models.py:32-43`，全部驼峰别名写盘。

| 磁盘 Key | Python 字段 | 类型 | 默认值 | 写入方 / 规则 | 锚点 |
|---|---|---|---|---|---|
| `schemaVersion` | `schema_version` | `str` | `"1.0"` | 固定值 | `src/octop_browser/record/models.py:33` |
| `recordingId` | `recording_id` | `str` | 必填 | `rec_<时间>_<hex6>` | `src/octop_browser/record/recorder.py:43-44` |
| `status` | `status` | `recording\|stopped\|failed` | `"recording"` | stop 时置 `stopped` | `src/octop_browser/record/recorder.py:110` |
| `createdAt` | `created_at` | ISO8601 | 必填 | UTC `_now()` | `src/octop_browser/record/recorder.py:39-40` |
| `endedAt` | `ended_at` | ISO8601 \| None | `None` | stop 时写入 | `src/octop_browser/record/recorder.py:111` |
| `profile` | `profile` | `str` | `"default"` | CLI/MCP 传入 | `src/octop_browser/record/recorder.py:371` |
| `target` | `target` | `RecordingTarget` | 空对象 | `targetId`/`startUrl`/`endUrl`/`title` | `src/octop_browser/record/models.py:20-24` |
| `privacy` | `privacy` | `PrivacySettings` | 见下 | 三轴策略 | `src/octop_browser/record/models.py:14-17` |
| `stats` | `stats` | `RecordingStats` | `{events:0, steps:0}` | 每事件 +1；折叠后写 steps | `src/octop_browser/record/recorder.py:250`、`:402` |

`PrivacySettings` 三轴：`input_policy` 默认 `mask-sensitive`、可选 `none|mask-sensitive|mask-all`；`url_query_policy` 默认 `mask-sensitive`；`screenshot_policy` 默认 `off`，可选 `on-error|on-step-or-error|every-event` (`src/octop_browser/record/models.py:10-17`)。

## 3. `events.jsonl` 行字典

每行由 `RecordSession._append()` 统一补 4 个键后追加：`seq`（进程内自增）、`recordingId`、`receivedAt`（UTC）、以及 JS 事件自身字段 (`src/octop_browser/record/recorder.py:240-249`)。`kind` 全集 16 种见 [录制引擎](../../architecture/record-engine.md) 第 2、4 节；其中 `recorder_installed` 是注入完成的哨兵，`navigation`/`history.pushState`/`popstate`/`hashchange` 由 URL 变化检测产生。

`target` 段由 JS `describeElement()` 生成 19 个键（`src/octop_browser/record/injected/recorder.js:201-232`），Pydantic `TargetDescriptor` 只显式建模其中 14 个，剩余通过 `extra="allow"` 原样保留 (`src/octop_browser/record/models.py:46-66`)。

## 4. `steps.json` 字段字典

`StepsDocument` 定义于 `src/octop_browser/record/models.py:80-88`。

| 磁盘 Key | 类型 | 生成规则 | 锚点 |
|---|---|---|---|
| `schemaVersion` | `str` | 固定 `"1.0"` | `src/octop_browser/record/models.py:81` |
| `recordingId` | `str` | 透传 manifest id | `src/octop_browser/record/semantic.py:346` |
| `startUrl` | `str` | 优先 `recorder_installed(seq<=1)` 的 page.url | `src/octop_browser/record/semantic.py:372-394` |
| `inputs[]` | `list[dict]` | `{name, kind, required, example?}`；`kind ∈ secret/email/string/date` | `src/octop_browser/record/semantic.py:122-170` |
| `steps[]` | `list[SemanticStep]` | 步骤 id 为 `step_001` 递增 | `src/octop_browser/record/semantic.py:104-105`、`:646-648` |
| `verification[]` | `list[dict]` | 通常取最后一个带 `urlContains` 的 expect，生成 `url_contains` | `src/octop_browser/record/semantic.py:665-669` |

`SemanticStep` 九个字段（`src/octop_browser/record/models.py:69-77`）：`id`、`kind`、`description`、`url`、`target`、`value`、`key`、`expect`。`kind` 实际取值包括 `open`/`navigate`/`new_tab`/`click`/`fill`/`select`/`set_checked`/`press`/`scroll_to`。`target` 是压缩后的 dict，最多保留 5 条 `selectorCandidates` (`src/octop_browser/record/semantic.py:68-79`)。

## 5. 回放报告与共变清单

`ReplayRunner.run()` 构造的报告字典键（`src/octop_browser/record/replay.py:35-43`）：`recordingId`、`startedAt`、`status`（`running/passed/failed`）、`inputs`、`stepsLoaded`、`steps[]`、`verification[]`、`endedAt`；失败时追加 `error`、`failureArtifacts`，成功落盘后追加 `reportPath`。每个 step 条目含 `id`/`kind`/`status` 及按类型附加的 `ref`、`matchedBy`、`urlAfter`、`error`、`switchTab`、`wait`、`scroll` 等键 (`src/octop_browser/record/replay.py:48-271`)。

- **共变**：加持久化字段必须同时改 Pydantic 模型、`store.write_manifest()/write_steps()` 的 `by_alias=True` 序列化、Reader 的 `.get()` 兜底与 `tests/unit/test_record_replay_store_skill.py`。
- **共变**：改步骤 `kind` 必须同步 `semantic._build_steps()`、`replay.run()` 的分支、`skill_generator._workflow_md()` 与 [回放引擎](../../architecture/record-engine.md)。
- **禁止**：把 `{{sensitive_value}}` 之外的明文密码、token 写入任何产物；脱敏在 JS 与 Python 双端执行。
- 必跑：`uv run pytest tests/unit/test_record_replay_store_skill.py tests/unit/test_record_replay_semantic.py tests/unit/test_record_replay_replay_runner.py -q`。
