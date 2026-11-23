---
kind: operations
scope: pitfalls-and-adr
status: draft
origin: generated
source_paths: [src/octop_browser/tool_interface.py, src/octop_browser/hooks.py, src/octop_browser/session.py, src/octop_browser/cdp/launcher.py, src/octop_browser/cdp/launch_prep.py, src/octop_browser/actions/capture.py, src/octop_browser/install.py, src/octop_browser/linux_deps.py]
symbols: [_global_hooks, page_navigated, send_passive, crop, _have_npx, _user_namespaces_available]
upstream: [INDEX.md, architecture/module-dep-matrix.md]
downstream: [operations/boundary.md, contracts/error-codes.md]
---

# ADR、高危雷区与存量未挂载代码

本页记录不可从代码结构直接看出的约束与历史包袱。修改相关模块前必须逐条核对。

## 1. 关键架构决策（ADR 摘录）

| 编号 | 决策 | 理由 / 证据 | 影响范围 |
|---|---|---|---|
| ADR-01 | 运行时不依赖 Playwright，直接手写 CDP over websockets | README `Core Technology` 表；`pyproject.toml:24-31` 依赖仅 websockets/pydantic/aiohttp/mcp | 所有动作页；禁止引入 Selenium/Playwright 运行时调用 |
| ADR-02 | Playwright 仅作一次性二进制下载器 | `src/octop_browser/install.py:1-41`；`src/octop_browser/cdp/launcher.py:84-123` 直接 `Popen` 缓存二进制 | 安装器可 `pip install playwright`，运行时不可 import |
| ADR-03 | 登录态完全靠 Chrome user-data-dir，不另建 Cookie 存储 | `src/octop_browser/profile.py:165-174`；`src/octop_browser/settings.py:78-92` | 同名 profile 必须映射同一目录 |
| ADR-04 | `cdp_host` 唯一来源为配置，禁止硬编码 `localhost` | `AGENTS.md:127`；`src/octop_browser/cdp/launcher.py:369`、`:677` | 所有 HTTP 拼接点 |
| ADR-05 | ref 导航即失效，禁止跨跳转复用 nodeId | `src/octop_browser/actions/navigate.py:28`、`src/octop_browser/session.py:141` | 新增导航类动作必须清缓存 |
| ADR-06 | token 估算统一 `len(content)//4` | `src/octop_browser/session.py:327`、`src/octop_browser/actions/js_eval.py:49`、`src/octop_browser/session.py:840` | 文本类动作保持一致 |
| ADR-07 | 录制产物存本机 JSON/JSONL，不引入数据库 | `src/octop_browser/record/store.py:28-43` | 所有录制/回放模型 |

## 2. 高危雷区（Danger Zones）

| 雷区 | 触发方式 | 后果 | 正确做法 / 锚点 |
|---|---|---|---|
| 导航后忘记清 ref | 新增跳转动作未调 `invalidate()` | 把旧 `nodeId` 发给新文档，点击落到错误节点 | `src/octop_browser/actions/navigate.py:28` 是模板 |
| `Input` 域调用 `enable` | 仿照 `Page.enable` 写 `Input.enable` | CDP 无该方法，报错 | `src/octop_browser/session.py:67-68` 注释明确 |
| 直连 `cdp_ws_url` 下切标签 | 调 `switch_tab` | 抛 `Cannot switch tabs...` | `src/octop_browser/session.py:107-110` |
| 截图返回 base64 给 agent | 把 `result["data"]` 直接写进 `content` | token 爆炸 | 必须落盘返回路径，见 `src/octop_browser/actions/capture.py:1-29` |
| `full_page=True` 常态化 | 每次截图都全页 | 多屏 PNG 数 MB，淹没 agent | `src/octop_browser/actions/capture.py:105-111` 明确默认 False |
| 远程 host 执行 kill | `cdp_host` 非 loopback 时终止浏览器 | 误杀远端进程或无效 | `src/octop_browser/cdp/launcher.py:452` 直接返回 False |
| SSE 安装器等待 sudo 密码 | 去掉 `sudo -n` | 请求挂死等 TTY | `src/octop_browser/linux_deps.py:190-206` |
| 旧 dnf 使用 `--skip-unavailable` | 为图省事合并字体步骤 | TencentOS 3/RHEL 8 报 `unrecognized arguments`，字体全跳过 | `src/octop_browser/linux_deps.py:269-271` |
| 密钥进日志 | 把 `nonce`/录制明文值打到 stdout | 敏感信息泄漏 | `src/octop_browser/record/injected/recorder.js:243-249` + `src/octop_browser/record/privacy.py:74-81` 双端脱敏 |
| 把 `status: draft` 页当权威 | 不回读源码直接改实现 | 弱命中幻觉 | INDEX 读取规程第 2 条 |

## 3. 存量未挂载 / 历史兼容代码警示

以下符号**存在但当前没有调用路径**，不要误以为它们参与主流程：

| 符号 | 位置 | 当前状态 | 备注 |
|---|---|---|---|
| `_global_hooks` | `src/octop_browser/tool_interface.py:18` | 声明后无任何读写 | 全局钩子不可用，勿据此设计 |
| `page_navigated` | `src/octop_browser/hooks.py:15`、`:33-34` | `on()` 接受注册但从不发射 | 导航感知请用 `before_action` 或轮询 |
| `crop` | `src/octop_browser/actions/capture.py:90`、`:101` | 保留参数，无独立效果 | 实际裁剪靠 `element_ref`/`full_page` |
| `_have_npx()` | `src/octop_browser/install.py:623-625` | 测试辅助，主流程不调用 | 仅为与旧安装器探测对称保留 |
| `_LEGACY_*` 状态文件 / 目录名 | `src/octop_browser/settings.py:75`；`src/octop_browser/profile.py:29-30` | 仅作为读取回退 | 新写入统一用 `.octop-browser-*` |
| `__hbr_*` / `__HBR_*` 命名 | `recorder.js`、`recorder.py` | 历史命名前缀 | 与 Octop 品牌无关，改动会破坏已有录制 |
| `send_passive()` | `src/octop_browser/cdp/client.py:118-122` | 已实现但当前无业务调用点 | 可用于不续期的健康探测 |

## 4. 版本与协议兼容陷阱

| 陷阱 | 源码约束 | 锚点 |
|---|---|---|
| hatchling 版本上限 | `hatchling>=1.25,<1.33`，且显式 Core Metadata 2.4 覆盖，防止新版本静默产出 2.5 被 twine 拒绝 | `pyproject.toml:2-6` |
| mcp 版本上限 | `mcp>=1.9,<3`；mcp 2.x 移除了 `Server.list_tools()`/`call_tool()` 装饰器 | `pyproject.toml:28-30` |
| CDP 字段新旧命名 | `cssContentSize`/`contentSize`、`cssVisualViewport`/`visualViewport` 双读 | `src/octop_browser/actions/capture.py:155-159`、`:195-199` |
| `/json/new` 方法 | 先 PUT 后 GET，兼容旧Chromium派生版 | `src/octop_browser/cdp/launcher.py:695-712` |
| Playwright 目录布局 | `chrome-win64`/`chrome-linux64` 优先，`chrome-win`/`chrome-linux` 兜底 | `src/octop_browser/cdp/launcher.py:51-81` |
| macOS headed 不脱离进程组 | 否则窗口无法获得前台焦点 | `src/octop_browser/cdp/launcher.py:561-576` |

## 5. 共变清单

- 新增 ADR：在本页追加编号并指向源码证据，不要只写结论。
- 移除历史兼容代码前：确认没有旧 profile / 旧录制文件依赖 `.harness-browser-*` 或 `__hbr_*`。
- 必跑：`make all`；涉及启动器时另跑 `uv run pytest tests/unit/test_launcher.py tests/unit/test_launch_prep.py -q`。

- 关联阅读：[模块依赖矩阵](../architecture/module-dep-matrix.md)、[排障 Runbook](runbook-and-observability.md)。
