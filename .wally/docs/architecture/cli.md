---
kind: component
scope: cli
status: draft
origin: generated
source_paths: [src/octop_browser/cli.py, pyproject.toml, Makefile]
symbols: [_build_parser, _action_kwargs, _run_action, _run_record, _run_replay, main]
upstream: [architecture/index.md, INDEX.md]
downstream: [contracts/api/cli-commands.md, architecture/api-and-mcp.md, architecture/record-engine.md]
---

# 组件架构：CLI 命令入口

CLI 是 `pyproject.toml:49-50` 注册的 `octop-browser` 可执行入口，基于 `argparse` 单层子命令加两层嵌套子命令。它不实现业务逻辑，只负责参数翻译、输出格式化与退出码判定，全部动作最终汇入 [API 与 MCP 接入层](api-and-mcp.md)。

## 1. 组件边界与解析结构

| 契约项 | 精确实现 | 源码锚点 |
|---|---|---|
| 可执行入口 | `octop-browser = "octop_browser.cli:main"` | `pyproject.toml:50` |
| 主解析器 | `_build_parser()` 创建 `ArgumentParser`，`subparsers(required=True)` | `src/octop_browser/cli.py:96-122` |
| 公共参数 | `--profile` 默认 `default`，`--json` 控制完整 `ToolResult` 输出 | `src/octop_browser/cli.py:55-67` |
| 模式互斥组 | `--headed` / `--headless` / `--auto` 写同一个 `dest="mode"` | `src/octop_browser/cli.py:70-93` |
| 命令组 | `record` 10 个子命令、`replay run` 1 个子命令 | `src/octop_browser/cli.py:320-402` |
| 退出码 | 动作类成功 `0`/失败 `1`；record 依 `data["ok"]`；replay 依 `status == "passed"` | `src/octop_browser/cli.py:534`、`:607`、`:623` |

## 2. 启动与分发调用树

```text
main() src/octop_browser/cli.py:631
  ├─ logging.basicConfig(level=INFO) :633
  ├─ _build_parser() :634 -> parse_args() :635
  ├─ [branch] install-browser :637 -> chromium_executable() 命中则直接 0；否则 ensure_chromium() :642
  ├─ [branch] record :644 -> asyncio.run(_run_record()) :537
  │    ├─ doctor -> RecordController.doctor() :541
  │    ├─ daemon-start/stop/status -> ensure_daemon()/send_request() :543-547
  │    ├─ start -> 有 --duration 走 start_in_process :564；否则经 daemon :573-585
  │    └─ list/show/generate-steps/generate-skill -> RecordingStore/RecordController :586-598
  ├─ [branch] replay :646 -> asyncio.run(_run_replay()) :610
  │    └─ --input 重复项以首个 "=" 切分 -> ReplayRunner.run() :617
  └─ [default] _run_action() :514
       ├─ action = _ACTION_ALIAS.get(cmd, cmd) :515
       ├─ open 命令强制改走 list_tabs :520-521
       ├─ kwargs = _action_kwargs(args) :516 -> browser_tool(action, profile, mode, **kwargs) :523
       └─ _format_result() :502 -> stdout（成功）/stderr（失败）
```

## 3. 子命令到动作的参数翻译字典

`_ACTION_ALIAS` 只处理连字符到下划线的 9 个特例，其余子命令名本身即合法动作名。

| CLI 子命令 | 翻译后的 action | 关键入参字典 Key（`_action_kwargs`） | 锚点 |
|---|---|---|---|
| `navigate` | `navigate` | `url` | `src/octop_browser/cli.py:433-434` |
| `open` | `list_tabs`（并在 `urls` 中开多标签） | `urls` | `src/octop_browser/cli.py:431-432`、`:520-521` |
| `dom-tree` | `dom_tree` | `level`（默认 `interactive`） | `src/octop_browser/cli.py:435-436` |
| `screenshot` | `screenshot` | `full_page` / `element_ref` / `path`（条件写入） | `src/octop_browser/cli.py:437-443` |
| `click` | `click` | `ref` 或 `selector` 或 `x`+`y`（互斥必填） | `src/octop_browser/cli.py:200-209`、`:444-450` |
| `type` / `fill` | `type` / `fill` | `text` + 可选 `ref`/`selector` | `src/octop_browser/cli.py:451-462` |
| `wait` | `wait` | `url_contains`/`text`/`selector` + `timeout_ms`（默认 10000） | `src/octop_browser/cli.py:465-472` |
| `select` | `select` | `value` + `ref` 或 `selector` | `src/octop_browser/cli.py:473-478` |
| `scroll` | `scroll` | `direction`（默认 down）+ `amount`（默认 300） | `src/octop_browser/cli.py:479-481` |
| `close-session` | `close_session` | `kill`（仅 `--kill` 时写入） | `src/octop_browser/cli.py:494-496` |

完整子命令清单（含 10 个 `record` 子命令）见 [CLI 命令全集](../contracts/api/cli-commands.md)。

## 4. 输出契约与失败语义

| 输出模式 | 成功 | 失败 |
|---|---|---|
| 默认文本 | `content` 为字符串则原样打印；dict 走 `json.dumps(ensure_ascii=False)` | `ERROR: {error}`，写 stderr，退出码 1 |
| `--json` | `result.model_dump(mode="json")` 全量结构，含 `metrics`/`metadata` | 仍是同一 JSON 结构，`success=false` |
| `record` | `json.dumps(data, ensure_ascii=False, indent=2)`（非 `--json` 带缩进） | `data["ok"]=False` 时退出码 1 |
| `replay` | 报告 `status=passed` 退出码 0 | 任意非 passed 退出码 1 |

`--json` 只影响动作类命令；record/replay 始终输出 JSON，区别仅在缩进。`install-browser` 是唯一不经过 `asyncio.run()` 的分支，命中已有二进制时打印路径并返回 0。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 新增 CLI 子命令：`_build_parser()` 注册 → 必要时补 `_ACTION_ALIAS` → `_action_kwargs()` 增加翻译分支 → 更新 [命令全集](../contracts/api/cli-commands.md) → 补 `tests/unit/test_cli.py`。
2. 新增 `record` 子命令：除解析器外必须在 `_run_record()` 增加 `elif` 分支，否则落入 `Unknown record command` 兜底。
3. 改动 `--profile`/`--json` 公共参数会影响全部动作类命令，必须回归 `tests/unit/test_cli.py` 全部用例。
4. 必跑命令：`uv run pytest tests/unit/test_cli.py -q`；改动发版入口时另跑 `make build` 校验 `pyproject.toml:49-50` 脚本仍可装载。
