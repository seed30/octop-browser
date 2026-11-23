---
kind: api-contract
scope: cli-commands
status: draft
origin: generated
source_paths: [src/octop_browser/cli.py, src/octop_browser/record/daemon.py]
symbols: [_build_parser, _ACTION_ALIAS, _action_kwargs, _run_record, _run_replay, main]
upstream: [contracts/api-reference.md, architecture/cli.md]
downstream: [integrations/downstream/record-daemon.md, contracts/error-codes.md]
---

# CLI 子命令全集与参数字典

全部 33 个子命令由 `_build_parser()` 在 `src/octop_browser/cli.py:96-404` 注册。下表逐一列清命令、参数与落点，无省略。

## 1. 公共参数与解析约定

| 参数 | 适用命令 | 默认值 | 说明 | 锚点 |
|---|---|---|---|---|
| `--profile` | 所有动作类、record doctor/start、replay run | `default` | profile 名，映射 user-data-dir | `src/octop_browser/cli.py:55-61` |
| `--json` | 所有动作类与 record/replay 子命令 | `False` | 动作类输出完整 `ToolResult`；record/replay 控制缩进 | `src/octop_browser/cli.py:62-67`、`:606` |
| `--headed`/`--headless`/`--auto` | 所有动作类 | `None`（走配置） | 互斥组，仅在 profile 首次调用时生效 | `src/octop_browser/cli.py:70-93` |
| `--version` | 顶层 | — | 打印安装包版本，`PackageNotFoundError` 时显示 `unknown` | `src/octop_browser/cli.py:47-52`、`:111-116` |

## 2. 动作类子命令（22 个）

| 子命令 | 落点 action | 位置参数 / 选项 | 关键默认值 | 注册锚点 |
|---|---|---|---|---|
| `install-browser` | `ensure_chromium()`（非 browser_tool） | `--quiet` | 不静默 | `src/octop_browser/cli.py:125-138` |
| `navigate` | `navigate` | `url` | — | `src/octop_browser/cli.py:141-144` |
| `open` | `list_tabs` + `urls` | `urls...`（`nargs="+"`） | — | `src/octop_browser/cli.py:147-164` |
| `dom-tree` | `dom_tree` | `--level {minimal,interactive,full,structured}` | `interactive` | `src/octop_browser/cli.py:167-185` |
| `screenshot` | `screenshot` | `--full-page`、`--element-ref`、`--path` | 均关闭 | `src/octop_browser/cli.py:188-198` |
| `click` | `click` | `--ref` \| `--selector` \| `--xy X Y`（互斥必填） | — | `src/octop_browser/cli.py:201-209` |
| `type` | `type` | `text` + `--ref` \| `--selector` | 无目标则直接键入 | `src/octop_browser/cli.py:212-218` |
| `fill` | `fill` | `text` + `--ref` \| `--selector` | 同上 | `src/octop_browser/cli.py:221-227` |
| `press` | `press` | `key` | — | `src/octop_browser/cli.py:230-233` |
| `wait` | `wait` | `--url-contains`、`--text`、`--selector`、`--timeout-ms` | `10000` | `src/octop_browser/cli.py:236-242` |
| `select` | `select` | `value` + `--ref` \| `--selector`（互斥必填） | — | `src/octop_browser/cli.py:245-251` |
| `scroll` | `scroll` | `--direction {up,down,left,right}`、`--amount` | `down` / `300` | `src/octop_browser/cli.py:254-260` |
| `hover` | `hover` | `ref` | — | `src/octop_browser/cli.py:263-266` |
| `eval-js` | `eval_js` | `expression` | — | `src/octop_browser/cli.py:269-272` |
| `go-back` | `go_back` | 仅公共参数 | — | `src/octop_browser/cli.py:275-282` |
| `go-forward` | `go_forward` | 仅公共参数 | — | `src/octop_browser/cli.py:275-282` |
| `reload` | `reload` | 仅公共参数 | — | `src/octop_browser/cli.py:275-282` |
| `new-tab` | `new_tab` | `--url` | `None`（about:blank） | `src/octop_browser/cli.py:285-288` |
| `switch-tab` | `switch_tab` | `tab_id` | — | `src/octop_browser/cli.py:290-293` |
| `close-tab` | `close_tab` | `--tab-id` | `None`（不关闭） | `src/octop_browser/cli.py:295-298` |
| `list-tabs` | `list_tabs` | 仅公共参数 | — | `src/octop_browser/cli.py:300-302` |
| `close-session` | `close_session` | `--kill` | `False`（仅断开 WS） | `src/octop_browser/cli.py:305-317` |

连字符命令名经 `_ACTION_ALIAS` 映射为下划线 action，共 9 条：`dom-tree`、`eval-js`、`go-back`、`go-forward`、`new-tab`、`switch-tab`、`close-tab`、`list-tabs`、`close-session` (`src/octop_browser/cli.py:413-423`)。其余命令名本身即合法 action。

## 3. record 子命令（10 个）

| 子命令 | 参数 | 后端调用 | 锚点 |
|---|---|---|---|
| `record doctor` | `--profile`、`--json` | `RecordController.doctor()` | `src/octop_browser/cli.py:323-325`、`:540-541` |
| `record daemon-start` | `--json` | `ensure_daemon()` | `src/octop_browser/cli.py:327-330`、`:542-543` |
| `record daemon-stop` | `--json` | `send_request({"command":"shutdown"})` | `src/octop_browser/cli.py:332-335`、`:544-545` |
| `record status` | `--json` | `send_request({"command":"status"})` | `src/octop_browser/cli.py:337-338`、`:546-547` |
| `record stop` | `--recording-id`、`--name`、`--no-generate-steps`、`--json` | daemon `stop` 请求 | `src/octop_browser/cli.py:340-344`、`:548-556` |
| `record start` | `--profile`、`--name`、`--duration`、`--privacy {none,mask-sensitive,mask-all}`、`--json` | 有 duration 走进程内；否则 daemon `start` | `src/octop_browser/cli.py:346-366`、`:557-585` |
| `record list` | `--json` | `RecordingStore.list_recordings()` | `src/octop_browser/cli.py:368-369`、`:586-592` |
| `record show` | `recording_id`、`--json` | `RecordingStore.recording_summary()` | `src/octop_browser/cli.py:371-375`、`:593-594` |
| `record generate-steps` | `recording_id`、`--json` | `RecordController.generate_steps()` | `src/octop_browser/cli.py:377-381`、`:595-596` |
| `record generate-skill` | `recording_id`、`--name`、`--json` | `RecordController.generate_skill()` | `src/octop_browser/cli.py:383-388`、`:597-598` |

`--privacy` 仅影响进程内模式：构造 `PrivacySettings(input_policy=args.privacy, url_query_policy="mask-sensitive", screenshot_policy="off")` (`src/octop_browser/cli.py:559-563`)；daemon 模式把值作为 request 的 `privacy` 字段透传 (`src/octop_browser/cli.py:582`)。

## 4. replay 子命令（1 个）

`replay run <recording_id>`：`--profile` 默认 `default`，`--input key=value` 可重复，按首个 `=` 切分为 `inputs` dict (`src/octop_browser/cli.py:390-402`、`:610-616`)。落点为 `ReplayRunner().run(recording_id, profile=..., inputs=...)` (`src/octop_browser/cli.py:617-619`)，报告 `status` 为 `passed` 时退出码 0。

## 5. 输出与退出码契约

| 命令族 | 成功输出 | 失败输出 | 退出码判据 |
|---|---|---|---|
| 动作类 | `content` 字符串或 JSON；`--json` 为完整 `ToolResult` | `ERROR: {error}` 写 stderr | `0 if result.success else 1` (`src/octop_browser/cli.py:534`) |
| `install-browser` | `Chromium already installed: {path}` 或静默成功 | `ensure_chromium()` 返回 False | `0/1` (`src/octop_browser/cli.py:637-643`) |
| `record` | `json.dumps(data, ensure_ascii=False, indent=None if --json else 2)` | 同结构 `ok=False` | `0 if data.get("ok", True) else 1` (`src/octop_browser/cli.py:607`) |
| `replay` | 报告 JSON | `{"status":"failed","error":...}` | `0 if status=="passed" else 1` (`src/octop_browser/cli.py:623`) |

## 6. 共变清单

新增子命令：`_build_parser()` → 必要时 `_ACTION_ALIAS` → `_action_kwargs()` → `main()` 分支 / `_run_record()` 分支 → 本页表格与 [总索引](../api-reference.md) → `tests/unit/test_cli.py`。必跑 `uv run pytest tests/unit/test_cli.py -q`。

- 关联阅读：[API 与 MCP 接入层](../../architecture/api-and-mcp.md)、[错误码字典](../error-codes.md)。
