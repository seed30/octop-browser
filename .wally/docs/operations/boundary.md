---
kind: operations
scope: knowledge-boundary
status: draft
origin: generated
source_paths: [pyproject.toml, .github/workflows/release.yml, src/octop_browser/mcp_server.py, src/octop_browser/cdp/launcher.py, src/octop_browser/record/replay.py]
symbols: [find_chrome, race_download_sources, _main, ReplayRunner.run]
upstream: [INDEX.md, architecture/module-dep-matrix.md]
downstream: []
---

# 认知边界声明与仓库外黑盒

本页明确本知识库的快照范围、无法从本仓验证的外部系统，以及动态分发能力的局限。超出边界的结论必须标注为外部黑盒，不得推测实现。

## 1. 文档快照基线

| 项目 | 取值 | 说明 |
|---|---|---|
| 仓库 | `octop-browser`（单仓模式） | `pyproject.toml:9` |
| 分支 | `chore/wally-init-20261124-c81` | 生成本站时的当前分支 |
| 快照提交 | `81b847fc00dd3d44ac3519f6c9489c7ca7a32d0e` | 仅代表源码基准；本站文档未提交 |
| 源码范围 | `src/octop_browser/**`、`tests/**`、`scripts/**` | 40 个已跟踪源码文件 |
| 排除范围 | `.wally/docs/**` | 避免生成循环；SOP 外部页只链接不改写 |
| 语言 | 简体中文（`zh-CN`） | 与 `.wiki-meta.json` 一致 |

## 2. 仓库外黑盒清单

| 外部系统 | 本仓可见部分 | 不可见部分 | 本仓证据锚点 |
|---|---|---|---|
| Chrome / Chromium 进程 | CDP 指令与 HTTP 发现端点、启动参数 | 浏览器内部渲染、下载、Cookie 加密实现 | `src/octop_browser/cdp/launcher.py:190-232`、`:618-712` |
| Playwright CDN | 候选主机、Range 探测、下载 argv | CDN 内容、版本目录实际结构、镜像一致性 | `src/octop_browser/playwright_download.py:16-80` |
| PyPI / TestPyPI | Makefile 上传命令、CI secret 名 | 索引服务、包审核与可见性策略 | `Makefile:81-106`、`.github/workflows/release.yml` |
| 系统包管理器 | 命令前缀、包名列表、探测顺序 | 仓库可用性、依赖解析、sudo 策略 | `src/octop_browser/linux_deps.py:25-25`、`:182-206` |
| MCP 客户端 | stdio/SSE 工具声明与响应 | 客户端如何展示/调用工具、env 注入细节 | `src/octop_browser/mcp_server.py:24-347`、`:482-530` |
| 目标网站页面 | 录制时的 DOM 描述、回放时的 DOM 快照 | 服务端业务逻辑、登录/风控策略 | `src/octop_browser/record/injected/recorder.js:201-232`、`src/octop_browser/record/replay.py:112-117` |
| Harness / Octop 上层项目 | README 中列出的相关项目名 | 其调度、记忆与网关实现 | `README.md:386-393` |

结论：上述系统的内部行为一律写「外部黑盒：本仓未包含服务端实现」，不得根据命名推测字段或错误码。

## 3. 动态分发与覆盖局限

- 元素定位依赖运行时 `dom_tree` 输出；静态文档无法保证任意页面的 ref 稳定，匹配阈值 45 是启发式而非语义完备 (`src/octop_browser/record/matcher.py:60-62`)。
- 回放的 `{{var}}` 值由调用方注入；本仓不保存明文密码，录制时已替换为 `{{sensitive_value}}` (`src/octop_browser/record/privacy.py:74-81`)。
- `BROWSER_USE_CDP_WS_URL` 直连模式跳过 launcher 与切标签逻辑，本站关于端口分配、粘滞 target、空闲回收的结论在该模式下不适用 (`src/octop_browser/session.py:55-57`、`src/octop_browser/idle.py:138`)。
- `record` 守护进程仅支持本地 Unix socket，无跨主机会话能力 (`src/octop_browser/record/daemon.py:132-165`)。
- 回放验证只实现 6 种类型；其它类型返回 `Unsupported verification: {typ}` (`src/octop_browser/record/replay.py:412-415`)。

## 4. 已知未覆盖与待验证项

| 项目 | 状态 | 说明 |
|---|---|---|
| 测试环境部署 | `BLOCKED` / `readiness: pending` | `.wally/config.json` 无部署 command、无目标环境、无凭证来源与回滚判据 |
| 生产部署自动化 | 不在仓内 | 仅有 PyPI 发版流水线，无服务部署 job |
| 数值错误码体系 | 未找到 | 全仓无 `degraded` 字面量、无错误码枚举，详见 [错误码字典](../contracts/error-codes.md) |
| `degraded` 状态字段 | 未找到 | 降级以「超时仍成功/仅日志/跳过」实现 |

## 5. 维护规则

本页随仓库结构与外部依赖变化更新；发现来自弱命中检索的结论必须先 `Read` 源码原文核验字面存在再落笔。跨仓依赖边若无法给出真实 `repo/path:line` 证据，一律不写入 `module-dep-matrix.md` 的 `repoDeps`。

- 关联阅读：[系统技术架构](../overview/system-architecture.md)、[业务流程总表](../business/index.md)。
