# octop-browser · Agent 知识库总路由表（INDEX）

> **Agent 读取规程（必读）**：
> 1. **禁止一次性全量读取 `.wally/docs/`**。先查下方「任务意图路由表」或「源码路径反向索引」，只打开命中的 1–3 篇目标文档（可用 `Read(limit=15)` 先看顶部 YAML 头确认 `source_paths`、`symbols` 与 `status`）。
> 2. **弱命中与草稿核验纪律**：标记 `status: draft` 的文档为源码提取草稿；从文档拿到 `路径:L起-L止`（`符号名`）、上下游字典结构与「共变耦合清单」后，执行修改前必须调用 `Read(path, offset, limit)` 复核源码字面实现（近邻词线索不等于原词覆盖）。
> 3. **人工精修保护**：标记 `origin: human` 或 `status: verified` 的页面为人工确认权威结论，后续重跑知识库严禁覆盖。本次快照无此类页面。基准提交：`chore/wally-init-20261124-c81` @ `81b847fc00dd3d44ac3519f6c9489c7ca7a32d0e`。
> 4. **本页为根路由，不写 YAML 头**；其余 codewiki 管理页均带 8 键 YAML 头。SOP 专用页与外部权威页仅链接，不参与 `pageOrder`。

`octop-browser` 是 Python 3.11+ 的轻量浏览器自动化库/CLI：启动真实 Chromium，通过手写 CDP over websockets 直接驱动（无 Playwright 运行时），以 Python SDK、CLI、MCP 三种入口暴露同一套 20 个动作；录制回放链路把浏览器事件折叠成语义步骤后重新执行。全局形态见 [系统技术架构](overview/system-architecture.md)，业务域见 [业务架构全景](overview/business-architecture.md)。

---

## 1. 任务意图最短路由表（Agent Intent Router）

| Agent 当前待办任务类型 | 按序必读文档（1 → 2 → 3） |
|---|---|
| **新增/修改浏览器动作、参数或 CLI/MCP Schema** | [入向接口总表](contracts/api-reference.md) → [API/MCP 组件](architecture/api-and-mcp.md) → [参数组装映射](integrations/topology-and-mapping.md) |
| **对接或修改下游 CDP 指令字典** | [Chrome CDP 契约](integrations/downstream/chrome-cdp.md) → [CDP 与启动器](architecture/cdp-and-launcher.md) → [错误码字典](contracts/error-codes.md) |
| **修改会话、profile、端口或登录态持久化** | [会话生命周期 WF-02](business/wf-session-lifecycle.md) → [CDP 与启动器](architecture/cdp-and-launcher.md) → [配置字典](contracts/configuration.md) |
| **修改 DOM 视图、ref 规则或点击定位** | [会话核心引擎](architecture/session-core.md) → [术语表](overview/glossary.md) → [主干调用链](overview/callchain-panorama.md) |
| **修改录制/回放/技能生成** | [业务流程总表](business/index.md) → [WF-03](business/wf-record-replay.md) → [录制产物模型](business/domain-models/recording-artifacts.md) |
| **新增/修改录制守护进程命令** | [守护进程契约](integrations/downstream/record-daemon.md) → [录制引擎](architecture/record-engine.md) → [CLI 命令全集](contracts/api/cli-commands.md) |
| **修改安装器、CDN 竞速或系统依赖** | [Playwright CDN 契约](integrations/downstream/playwright-cdn.md) → [包管理器契约](integrations/downstream/os-package-manager.md) → [安装器](architecture/installer.md) |
| **定位报错 / 超时 / 降级原因** | [错误码字典](contracts/error-codes.md) → [排障 Runbook](operations/runbook-and-observability.md) → [主干调用链](overview/callchain-panorama.md) |
| **跨模块重构 / 评估联动修改与跑单测** | [模块依赖矩阵](architecture/module-dep-matrix.md) → [高危雷区与 ADR](operations/pitfalls-and-adr.md) → [构建与单测映射](operations/dev-and-test.md) |
| **理解外部边界与不可验证项** | [认知边界](operations/boundary.md) → [部署与运行时](operations/deployment-and-runtime.md) → [组件总表](architecture/index.md) |

---

## 2. 源码目录/核心文件 → 文档反向索引表（Source-to-Doc Reverse Index）

| 仓库源码路径（前缀或关键文件） | 架构模型 / 角色 | 首选关联文档 | 关联上下游/业务文档 |
|---|---|---|---|
| `src/octop_browser/cli.py` | UI / CLI | [architecture/cli.md](architecture/cli.md) | [CLI 命令全集](contracts/api/cli-commands.md) |
| `src/octop_browser/mcp_server.py`、`tool_interface.py`、`__init__.py` | API Service | [architecture/api-and-mcp.md](architecture/api-and-mcp.md) | [MCP 工具清单](contracts/api/mcp-tools.md)、[Python API](contracts/api/python-sdk.md) |
| `src/octop_browser/session.py`、`actions/`、`dom/`、`hooks.py` | Core Engine | [architecture/session-core.md](architecture/session-core.md) | [WF-01](business/wf-browser-action.md)、[Chrome CDP 契约](integrations/downstream/chrome-cdp.md) |
| `src/octop_browser/cdp/`、`profile.py`、`idle.py` | DAO / 传输底座 | [architecture/cdp-and-launcher.md](architecture/cdp-and-launcher.md) | [配置字典](contracts/configuration.md)、[WF-02](business/wf-session-lifecycle.md) |
| `src/octop_browser/record/` | 异步 Worker / Core Engine | [architecture/record-engine.md](architecture/record-engine.md) | [WF-03](business/wf-record-replay.md)、[守护进程契约](integrations/downstream/record-daemon.md) |
| `src/octop_browser/install.py`、`playwright_download.py`、`linux_deps.py` | 存储与适配底座 | [architecture/installer.md](architecture/installer.md) | [CDN 契约](integrations/downstream/playwright-cdn.md)、[包管理器契约](integrations/downstream/os-package-manager.md) |
| `src/octop_browser/settings.py`、`mode.py`、`models.py` | 配置与数据模型 | [工具结果与配置模型](business/domain-models/toolresult-and-settings.md) | [配置字典](contracts/configuration.md)、[术语表](overview/glossary.md) |
| `tests/`、`scripts/`、`Makefile`、`pyproject.toml` | 验证资产 | [构建与单测映射](operations/dev-and-test.md) | [部署与运行时](operations/deployment-and-runtime.md) |
| `.github/`、`assets/`、`skills/` | 流水线 / 资产 / Skill | [认知边界](operations/boundary.md) | [组件总表](architecture/index.md) |
| `.wally/`（SOP 资产，非源码） | 外部引用 | 见下方第 6 节 | 不参与 codewiki `pageOrder` |

---

## 3. 模块技术能力清单（Component Registry）

| 组件标识 | 框架模型（Archetype） | 核心源码路径 | 核心入口符号（`路径:行号`） | 上游调用方 | 下游依赖 | 文档 |
|---|---|---|---|---|---|---|
| `api-and-mcp` | `API Service` | `mcp_server.py`、`tool_interface.py` | `call_tool` (`src/octop_browser/mcp_server.py:391`)、`browser_tool` (`src/octop_browser/tool_interface.py:91`) | CLI、MCP 客户端、SDK | `session-core`、`record-engine`、`installer` | [查看](architecture/api-and-mcp.md) |
| `cli` | `UI / CLI` | `cli.py` | `main` (`src/octop_browser/cli.py:631`) | 终端用户、脚本 | `api-and-mcp` | [查看](architecture/cli.md) |
| `session-core` | `Core Engine` | `session.py`、`actions/`、`dom/` | `BrowserSession` (`src/octop_browser/session.py:174`) | `api-and-mcp` | `cdp-and-launcher` | [查看](architecture/session-core.md) |
| `cdp-and-launcher` | `DAO / 传输底座` | `cdp/`、`profile.py`、`idle.py` | `CDPClient.send` (`src/octop_browser/cdp/client.py:75`)、`launch_or_attach` (`src/octop_browser/cdp/launcher.py:497`) | `session-core`、`record-engine` | Chrome CDP | [查看](architecture/cdp-and-launcher.md) |
| `record-engine` | `Async Worker / Engine` | `record/` | `RecordSession` (`src/octop_browser/record/recorder.py:54`)、`ReplayRunner` (`src/octop_browser/record/replay.py:21`) | CLI、MCP | CDP、Unix socket、本地磁盘 | [查看](architecture/record-engine.md) |
| `installer` | `DAO / 适配底座` | `install.py`、`playwright_download.py`、`linux_deps.py` | `ensure_chromium` (`src/octop_browser/install.py:280`) | CLI、MCP | Playwright CDN、包管理器 | [查看](architecture/installer.md) |

---

## 4. 业务流程与下游依赖速查

- **全景认知**：[系统技术架构](overview/system-architecture.md) · [业务架构全景](overview/business-architecture.md) · [主干调用链全景](overview/callchain-panorama.md) · [领域术语与符号映射](overview/glossary.md)
- **上下游与契约**：[上下游集成总览](integrations/index.md) · [参数组装映射](integrations/topology-and-mapping.md) · [入向接口参考](contracts/api-reference.md) · [错误码字典](contracts/error-codes.md) · [配置手册](contracts/configuration.md)
- **业务与数据**：[业务流程总表](business/index.md) · [核心领域与数据模型](business/domain-models/index.md)
- **验证、雷区与边界**：[构建与单测映射](operations/dev-and-test.md) · [部署与运行时](operations/deployment-and-runtime.md) · [排障 Runbook](operations/runbook-and-observability.md) · [高危雷区与 ADR](operations/pitfalls-and-adr.md) · [认知边界声明](operations/boundary.md)

---

## 5. 下游系统独立契约

- [Chrome DevTools Protocol](integrations/downstream/chrome-cdp.md) — HTTP `/json*` 与 WebSocket 指令字典
- [Playwright CDN 与 pip 引导](integrations/downstream/playwright-cdn.md) — CDN 竞速与下载 env
- [Linux 包管理器与 CJK 字体](integrations/downstream/os-package-manager.md) — 步骤字典与提权规则
- [录制守护进程 IPC](integrations/downstream/record-daemon.md) — Unix socket 请求/响应字典

---

## 6. SOP 专用与外部权威页（不参与 codewiki 管理）

以下页面由 `wally-init` 建立，属于 SOP 流程与外部权威资产；按 `subagent-and-meta.md`「本地增量范围」第 5 条，只链接、不复制正文、不自动改写为 codewiki 模板，也不计入 `.wiki-meta.json` 的 `pageOrder`/`pageCount`。

| 外部页 | 用途 | 未纳入 codewiki 管理的原因 |
|---|---|---|
| [apis/index.md](apis/index.md)、[apis/_template.md](apis/_template.md) | 对外接口文档索引与模板 | SOP 专用空模板，正文本站已在 `contracts/` 独立覆盖 |
| [devops/env.md](devops/env.md)、[devops/test-env-deploy.md](devops/test-env-deploy.md) | 环境搭建与测试环境部署 | SOP 专用；仓库无测试环境部署入口，`readiness: pending` |
| [integration_test/integration_test.md](integration_test/integration_test.md) | 集成测试规范 | SOP 专用；真实执行信息见 [构建与单测映射](operations/dev-and-test.md) |
| [unittest/unittest.md](unittest/unittest.md) | 单元测试规范 | SOP 专用；实际用例映射见 [构建与单测映射](operations/dev-and-test.md) |
| [sqls/README.md](sqls/README.md) | 数据库/Schema 权威目录 | 本仓无数据库，仅保留目录约定 |
