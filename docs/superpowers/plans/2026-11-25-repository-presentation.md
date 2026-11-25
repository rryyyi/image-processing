# Repository Presentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 docrestore 整理为内容准确、容易上手、能展示工具工程与评测能力的专业项目仓库。

**Architecture:** 首页提供价值、效果、架构、入口和证据；架构与评测细节放入专题文档。GitHub Actions 运行现有测试，模板约束问题报告和验证记录。不改变图像算法。

**Tech Stack:** Markdown / Mermaid、GitHub Actions YAML、Python 3.10–3.12、pytest。

**Spec:** `docs/superpowers/specs/2026-11-25-repository-presentation-design.md`（用户已确认）。

## Global Constraints

- 中文 README 为主，不宣称已实现 LLM 调度、Agent 编排、MCP 服务或 OCR 服务。
- 不改图像处理算法，不新增 Agent 运行时或服务。
- 不擅自决定开源许可证，不使用虚构覆盖率和性能承诺。
- 数据、模型和第三方代码不入库；历史实验与本次验证分开说明。
- 文件应留在用户指定目录；本次在当前目录的维护分支执行，避免产生另一份需要用户寻找的交付目录。

## Review Focus

- Windows 与 POSIX 安装命令：分别列出激活方法，核实 Python 版本范围。
- 场景参数：通过实际 `run --help` 和端到端调用核实 `--config`。
- API 契约：BGR/uint8 图像、零起始 index、manifest 字段与磁盘输出一致。
- 复现实验：明确第三方目录、模型依赖、汇总文件覆盖行为及区域 SSIM 限制。
- 本地链接与 CI：检查所有新增文档的文件链接和 YAML；不将本地测试冒充远程 CI 成功。

---

### Task 1: 首页与使用入口

**Files:** Rewrite `README.md`.

**Interfaces:** Consumes `DocumentPipeline.process_path(input_path, output_dir, save_intermediates)`、`PipelineResult.manifest()`、Typer CLI 与已有结果图片；produces 新的仓库首页及相对链接。

- [x] 核实安装与 CLI：创建 Python 3.12 环境，安装 `.[dev]`，执行现有 pytest 和 CLI 帮助。
- [x] 写入项目定位、效果图、架构、跨平台安装、CLI、API、输出、基准表、目录与边界。
- [x] 执行 README 使用路径：现有图像单图修复、场景参数、评估、API manifest。
- [x] 记录验证结果。

### Task 2: 架构和评测文档

**Files:** Create `docs/architecture.md`, `docs/evaluation.md`.

**Interfaces:** Consumes 主包配置/输出与历史 CSV；produces README 中的深度阅读入口。

- [x] 写清模块职责、处理顺序、告警码、运行时局限及 Agent 工具接入建议。
- [x] 写清样本规模、指标与区域 SSIM 实现限制、第三方版本、复现命令、输出覆盖行为。
- [x] 检查示例和模块链接，核实历史汇总统计来自 CSV。

### Task 3: 协作和自动检查

**Files:** Create `CONTRIBUTING.md`, `.github/workflows/tests.yml`, `.github/ISSUE_TEMPLATE/bug_report.yml`, `.github/ISSUE_TEMPLATE/feature_request.yml`, `.github/pull_request_template.md`; modify `.gitignore` if needed.

**Interfaces:** Consumes `pip install -e ".[dev]"` 与 `python -m pytest`；produces Linux/Windows × Python 3.10/3.12 测试矩阵和简洁协作入口。

- [x] 配置只读权限、push/PR/手动触发与标准安装测试流程，不依赖 GPU、未入库权重或真实数据。
- [x] 写贡献指南、Bug/功能表单及 PR 验证清单。
- [x] 忽略本地凭据、构建产物和日常运行输出，保留已有结果证据。
- [x] 解析 YAML，检查工作流语义和所有新增本地文档链接。

### Task 4: 最终验证与交付

**Files:** Update 本计划和设计的执行清单。

- [x] 独立复查整体 diff（当前 harness 无 subagent 工具，采用作者二次复查并说明）。
- [x] 运行适合本次修改的完整测试、CLI/API smoke、链接/YAML 检查与 `git diff --check`。
- [ ] 完成可审核的本地提交；核实远程是否变化、身份和推送权限，不强推或改写历史。
- [ ] 如共享主分支发布需额外确认，保留完整本地成果并仅在最后询问远程发布。

## 执行记录

- 用户已确认设计。无 subagent 工具，使用 executing-plans 在本会话连续执行。
- Ruling: 本次仅文档与配置维护，在当前目录新建维护分支；不创建额外 worktree，以保持用户指定交付路径。若后续用户在此目录同时切换分支，需要先检查未提交修改。
- Ruling: 不新增测试业务逻辑；复用现有测试并执行真实文档示例和 YAML/链接检查，避免为低风险 Markdown 和静态模板编写镜像测试。
- Task 1: complete — CLI help、默认处理、phone 预设、参考评估和 Python API 实际执行成功。
- Task 2: complete — 76 个本地文件/图片/锚点链接有效；20 个样本 ID、展示均值和 PSNR 差值与 CSV 一致。
- Task 3: complete — 工作流触发、权限、四组矩阵和 Issue 表单检查通过；四个场景预设全部执行成功。
- Final review: self-review (no subagent tool)。对照代码核查入口、配置、manifest、告警、历史结果与复现脚本；没有修改主包算法或历史数据。
- Final review: 区域 SSIM 回退、非独立测试集、计时范围、配置部分校验、CLI 中间结果开关与预设锐化依赖均已在文档说明。未将本次维护扩为算法修复。
- Verification: Python 3.12，pytest 16 passed；CLI/API/scenarios、Markdown 链接、YAML 和 git diff --check 均通过。远程 CI 还未运行。
- Remote: 初始 clone 成功；最终多次读取远程 refs 因 TLS 握手失败，无法确认远程最新状态或发布完成。未关闭 TLS 证书校验，未强推。
