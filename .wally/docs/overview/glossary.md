---
kind: glossary
scope: domain-glossary
status: draft
origin: generated
source_paths: [src/octop_browser/settings.py, src/octop_browser/mode.py, src/octop_browser/dom/refs.py, src/octop_browser/record/models.py]
symbols: [OctopSettings, BrowserMode, RefCache, PrivacySettings, StepsDocument]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [business/index.md, contracts/configuration.md]
---

# 领域术语与代码符号映射表

本页锁定「自然语言需求词 → 源码精确符号」的唯一对应关系。所有页面沿用本表命名，禁止同义词混写。运行时形态与装配顺序见 [系统技术架构](system-architecture.md)，配置取值见 [配置项全集](../contracts/configuration.md)。

## 1. 浏览器会话与连接术语

| 需求侧说法 | 精确符号 / 字面量 | 定义锚点 | 易混点澄清 |
|---|---|---|---|
| 会话 | `BrowserSession` (`src/octop_browser/session.py:174`) | 面向调用方的异步 API 门面，持有 `_InternalCDPSession` 与动作计数 | 不等于 Chrome 进程；`close()` 默认只断开 WebSocket |
| 底层会话 | `_InternalCDPSession` (`src/octop_browser/session.py:33`) | 每个页面一个 `CDPClient` + 一个 `RefCache` | `cfg.cdp_ws_url` 存在时直接连接，完全绕过 launcher |
| 无状态工具 | `browser_tool` (`src/octop_browser/tool_interface.py:91`) | 以 `profile` 为键从 `_registry` 复用会话的动作分发器 | 名称叫无状态，但进程内注册表有状态 |
| 配置对象 | `OctopSettings` (`src/octop_browser/settings.py:110`) | 每个字段用 `default_factory` 在实例化时重读环境变量 | 模块级 `settings` 单例在导入时读取一次 |
| 启动模式 | `BrowserMode = Literal["auto", "headed", "headless"]` (`src/octop_browser/mode.py:21`) | 三态枚举，`auto` 依据 `DISPLAY`/`WAYLAND_DISPLAY` 判定 | `headless=` 布尔参数是旧版逃生舱，优先级最高 |
| 桌面探测 | `has_desktop_environment()` (`src/octop_browser/mode.py:43`) | macOS/Windows 恒真；Linux 需要显示环境变量 | SSH 无 X 转发时返回假，走 headless |
| 空闲回收器 | `BrowserIdleController` (`src/octop_browser/idle.py:22`) | 按真实 CDP 活动刷新截止时间，超时终止本地 profile | 仅本地 host 且未设 `cdp_ws_url` 时创建 |

## 2. DOM 与元素定位术语

| 需求侧说法 | 精确符号 / 字面量 | 定义锚点 | 易混点澄清 |
|---|---|---|---|
| DOM 视图级别 | `DOMLevel = Literal["minimal","interactive","full","structured"]` (`src/octop_browser/dom/builder.py:10`) | 四档 token 开销递减输出 | `structured` 返回 JSON 字符串，不是 dict |
| 元素引用 | `ref` (`src/octop_browser/dom/builder.py:124`) | `_make_ref` 生成 `btn_2`/`inp_search` 形式短引用 | 导航后全部失效，必须重新 `dom_tree` |
| 引用缓存 | `RefCache` (`src/octop_browser/dom/refs.py:6`) | `ref → CDP nodeId` 字典，含 `invalidate()` | `RefCache` 不缓存 DOM 文本，只存节点 ID |
| 选择器候选 | `selectorCandidates` (`src/octop_browser/dom/builder.py:97`) | 由 `data-testid`/`id`/`name`/`aria-label`/`type` 拼出的定位串 | 录制器另有版本，最多 8 条并附带 CSS 路径 |
| 可访问名 | `accessibleName` (`src/octop_browser/dom/builder.py:240`) | `aria-label` → `aria-labelledby` → `label` → `placeholder` → 子树文本的优先链 | 与 `label` 字段不是同一来源 |

## 3. 录制回放与产物术语

| 需求侧说法 | 精确符号 / 字面量 | 定义锚点 | 易混点澄清 |
|---|---|---|---|
| 录制会话 | `RecordSession` (`src/octop_browser/record/recorder.py:54`) | 注入 JS 采集器并把事件落盘的生命周期对象 | 与 `BrowserSession` 互不持有，可跨进程通过 daemon 持有 |
| 录制清单 | `RecordingManifest` (`src/octop_browser/record/models.py:32`) | `manifest.json`，记录状态、profile、隐私策略与计数 | JSON 键用别名 `recordingId`/`createdAt` 等驼峰形式 |
| 原始事件 | `events.jsonl` (`src/octop_browser/record/store.py:34`) | 每行一个 JSON 事件，含 `seq`/`kind`/`page`/`target` | 追加写后即 `flush`，供跨进程读取 |
| 语义步骤 | `SemanticStep` (`src/octop_browser/record/models.py:69`) | 由事件折叠出的 `open`/`click`/`fill` 等可重放意图 | `value` 中的 `{{var}}` 是输入变量占位符 |
| 步骤文档 | `StepsDocument` (`src/octop_browser/record/models.py:80`) | `steps.json`，含 `inputs`、`steps`、`verification` | 别名 `schemaVersion`/`startUrl` 为驼峰 |
| 目标匹配 | `match_target()` (`src/octop_browser/record/matcher.py:19`) | 用选择器、id、role、名称加权打分选当前 DOM 的 `ref` | 分数阈值 45，低于阈值返回 `not-found` |
| 回放运行器 | `ReplayRunner` (`src/octop_browser/record/replay.py:21`) | 逐步骤调用 `browser_tool` 并写 `replay_*.json` 报告 | 回放的是意图，不是原始坐标序列 |
| 隐私策略 | `PrivacySettings` (`src/octop_browser/record/models.py:14`) | `input_policy` / `url_query_policy` / `screenshot_policy` 三轴 | `mask-sensitive` 的脱敏字面量为 `{{sensitive_value}}`，实现见 `src/octop_browser/record/privacy.py:61` |
| 草稿技能 | `draft.skill.md` (`src/octop_browser/record/skill_generator.py:32`) | 由步骤文档生成的 agent Skill 草案 | 生成器固定 `allowed-tools: Bash` |

## 4. 安装与传输术语

| 需求侧说法 | 精确符号 / 字面量 | 定义锚点 | 易混点澄清 |
|---|---|---|---|
| 托管 Chromium | `chromium_executable()` (`src/octop_browser/install.py:94`) | 查找 Playwright 缓存中版本最高的 Chromium | 运行时不依赖 Playwright，仅安装阶段引导 |
| CDN 竞速 | `race_download_sources()` (`src/octop_browser/playwright_download.py:108`) | 并行探测自定义源、npmmirror、官方源，按 TTFB 排序 | 全部探测失败时回退目录顺序而非报错 |
| 系统依赖步骤 | `build_install_steps()` (`src/octop_browser/linux_deps.py:220`) | 按 `apt-get/dnf/yum/zypper/pacman/apk` 生成安装命令 | CJK 字体步骤为 `optional=True`，失败不回滚 |
| CDP 客户端 | `CDPClient` (`src/octop_browser/cdp/client.py:30`) | 手写 WebSocket 帧协议，`send()` 以 id 匹配 Future | `Input` 域无 `enable` 方法，直接发指令 |
| 下游传输封装 | `launch_or_attach()` (`src/octop_browser/cdp/launcher.py:497`) | 端口占用则重连，否则启动本地 Chrome | 远程 host 禁止启动，只能直连 `cdp_ws_url` |
| 进程回收 | `terminate_browser()` (`src/octop_browser/cdp/launcher.py:440`) | 先发 `Browser.close`，超时后 POSIX `pkill` 兜底 | 远程 host 直接返回 `False`，不做任何终止 |

## 5. 严禁混用的命名铁律

- 源码符号、字典 Key、环境变量、文件名一律保留原样，不翻译、不改大小写。
- 文档标题、表头与说明文字统一使用简体中文，与 `.wiki-meta.json` 的 `language` 一致。
- 引用源码一律写「仓库根相对路径:行号」，禁止 `L` 前缀与裸文件名。
- 更细的会话状态迁移见 [主干调用链全景](callchain-panorama.md)，业务能力分组见 [业务流程总表](../business/index.md)。
