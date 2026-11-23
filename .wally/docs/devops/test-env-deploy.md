# 测试环境部署

以项目已有部署文档为权威，在这里记录引用、实际命令与验证方式。

## 环境与执行

- 环境类型、地址、资源归属、访问和部署授权边界：
- 创建/查询/代码同步/部署命令：
- 依赖、配置和凭证来源（不记录明文）：
- 就绪与生效检查命令：
- 缺陷修复前保留基线的方法：
- 回滚命令与触发条件：
- 清理/释放及保留条件：

## 项目自定义

补充组件清单、常用环境和同步限制。已有文件升级不覆盖。


初始化时从项目实际文件补齐执行指引，记录来源、工作目录、命令、前置依赖、结果判据及清理/回滚。必要信息缺失时在 `.wally/config.json` 设可选 `readiness: pending` 并列出具体 `gaps`，向用户询问并等待补齐；阶段标为 BLOCKED，不能自动跳过或因指南路径非空而启用。静态核验完整后才设 ready，不代表执行通过。未带 readiness 的旧配置仍兼容 command/guide 判断。

## 静态核验结论（未实际执行）

核验对象：README.md / README_CN.md、Makefile、pyproject.toml、.github/workflows/*.yml、CONTRIBUTING.md、tests/、scripts/。结论：`readiness: pending`。

- 仓库是 Python 库/CLI（hatchling 打包，pyproject.toml；`octop-browser` 入口），本地启动 Chromium 经 CDP 驱动，不是需部署到测试环境的服务。
- 未发现 Dockerfile / Compose / k8s / 部署脚本 / CI 部署 job；CI（ci.yml、release.yml）仅做 install → lint → typecheck → make test → make build → PyPI 发布。
- 现有最贴近的发布入口：`make publish`（PyPI）、`make publish-test` = `make publish PYPI_REPO=testpypi`（TestPyPI，Makefile）、`release.yml` 的 `pypa/gh-action-pypi-publish`（`environment: pypi`、`secrets.PYPI_API_TOKEN`）。这些是发版/上传制品，不是测试环境部署，且无部署后生效检查、无回滚/释放、无测试环境凭证来源，故不作为 deploy.command。
- 具体缺口见 `.wally/config.json` 的 `deploy.gaps`；command / verify / rollback 保持为空，等待提供真实测试环境信息后再补齐，禁止臆造命令。
