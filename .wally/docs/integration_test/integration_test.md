# 集成测试规范

## 环境

引用实际部署文档，填写测试环境、依赖、鉴权来源、健康检查、数据隔离及清理方式。禁止使用真实用户凭证或把测试默认指向生产资源。

## 用例

填写 runner、用例目录、fixture、正向/逆向/边界与异常恢复设计。断言覆盖业务结果和副作用，不只检查响应码；涉及中文/JSON 时覆盖 Unicode。稳定回归用例放项目测试目录，一次性用例放本任务 cases/，避免共享可变状态。

## 执行与判定

填写实际命令、逐用例及汇总报告位置、失败/跳过判定与 CI 门禁。核实实际执行数量、断言/环境/执行错误，退出码不能替代业务结果。必要的请求标识、输入、代码版本与摘要写入任务 IT 产物，不提交凭证和临时运行日志。
项目扩展可补充具体工具和更严格门禁。

执行前读取本任务 execution.json 的 it 配置；command/guide 均为空则自动 SKIPPED，保留未验证边界。配置后的失败或环境/权限/工具缺失应阻塞，不得冒充未配置。


初始化时从项目实际文件补齐执行指引，记录来源、工作目录、命令、前置依赖、结果判据及清理/回滚。必要信息缺失时在 `.wally/config.json` 设可选 `readiness: pending` 并列出具体 `gaps`，向用户询问并等待补齐；阶段标为 BLOCKED，不能自动跳过或因指南路径非空而启用。静态核验完整后才设 ready，不代表执行通过。未带 readiness 的旧配置仍兼容 command/guide 判断。

## 静态核验结论（未实际执行）

真实入口（均见 Makefile / pyproject.toml / CI）：

- `make test` → `uv run pytest --cov=octop_browser --cov-report=term-missing`（`testpaths = ["tests"]`，含 `tests/integration/`）；CI 门禁：.github/workflows/ci.yml、release.yml 的 Test 步骤均运行 `make test`。
- 前置：`make install`（`uv sync --extra dev`，见 README.md / CONTRIBUTING.md）；CI 另执行 `sudo apt-get install -y chromium-browser`。
- 仅集成用例：`uv run pytest tests/integration/ -v`（CODEBUDDY.md）。

环境与用例：`tests/integration/` 用真实 headless Chrome + CDP（test_session.py、test_record_replay_e2e.py），页面为 `data:` URL / 本地临时 HTML，自包含、无需外部测试环境地址与凭证；pyproject.toml 定义 `integration` marker。`tests/integration/conftest.py` 在 `find_chrome()` 为 None 时整组 skip。

验收判据：以 pytest 逐用例汇总为准（退出码 0、无 failed/error；skip 不计失败），不以退出码替代业务断言；完整描述见 `.wally/config.json` 的 `it.success_criteria`。结论：`readiness: ready`，`gaps: []`（不代表已实际执行通过）。
