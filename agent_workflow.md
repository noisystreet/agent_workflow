# 使用 Agent 编写新项目时的准备工作（工作流备忘）

本文整理从「仅想法」到「可协作骨架」的落地流程，供使用 **编程 Agent** 开新项目时对照。顺序可按团队习惯微调，但 **文档 → 构建 → 协作 → 提交规范** 不宜倒置。

---

## 1. 产品与架构先行

- **一句话定位**：面向谁、解决什么问题（如：面向开发者的 CLI 工具、SaaS 平台、跨平台桌面应用、嵌入式系统等）。  
- **架构总览文档**：目标/非目标、分层图、演进路线、开放决策、仓库目录规划（可先写规划路径，代码后补）。  
- **关键子系统专文**：核心模块单独成文（如 `docs/API.md`、`docs/DATA_MODEL.md`），避免总览过长。  
- **目标平台与运行环境**：若已定目标（如浏览器、移动端、特定 OS / 硬件），写进架构文档并约束工具链与依赖选型。  
- **命名**：项目代号 / 包名 / 模块名统一后写入 README、文档与包管理配置，避免后期全文替换。

---

## 2. 法律与贡献

- **许可证**：选定 SPDX 标识（如 MIT、Apache-2.0 或双许可），放置对应 LICENSE 文件，源码头部按需添加 `SPDX-License-Identifier`。  
- **CONTRIBUTING**：如何贡献、许可证对贡献的默认含义、**不要**在公开 Issue 发安全问题的指引链到 `SECURITY.md`。  
- **SECURITY.md**：漏洞上报方式（若暂无公开渠道，写清「待补充」也可）。  
- **CHANGELOG.md**：从 `[Unreleased]` 起记变更习惯。

---

## 3. 最小可构建工程

- **包管理与项目骨架**：初始化项目结构（如 `cargo init`、`npm init`、`go mod init` 等），至少包含一个占位模块，保证构建/类型检查命令能通过。  
- **工具链锁定**：固定语言版本、格式化工具版本（如 `rust-toolchain.toml`、`.node-version`、`go.mod` 的 go 版本），与文档描述一致。  
- **项目配置**：构建脚本、路径别名、环境变量占位等（如 `.cargo/config.toml`、`tsconfig.json`、`Makefile`），注释指向相关文档。  
- **环境变量管理**：提供 `.env.example` 列出所有需配置的环境变量及示例值，`.env` 本身加入 `.gitignore`，避免密钥与配置入库。  
- **依赖锁定文件策略**：明确锁文件是否提交——可执行项目（binary / 应用）应提交（保证构建可复现）；库项目（library）不应提交（避免下游版本冲突）。  
- **EditorConfig**：放置 `.editorconfig`，统一缩进风格、换行符（LF / CRLF）、字符编码（UTF-8），确保跨编辑器与 Agent 输出一致。  
- **代码格式化配置**：放置格式化工具配置（如 `rustfmt.toml`、`.prettierrc`、`.golangci.yml`），明确规则而非依赖工具默认值，与 CI 格式检查对齐。  
- **统一命令入口**：提供 `Makefile` / `Justfile` / `package.json scripts` 等，将常用命令（build / test / lint / clean）收敛为单一入口，降低 Agent 与新成员猜测命令的成本。  
- **最小可运行示例**：至少一个能实际运行的占位代码（如 `fn main() {}`、健康检查端点 `/ping`），而非空目录，确保构建通过 ≠ 运行通过，尽早暴露链接错误、端口冲突等运行时问题。  
- **`.gitignore`**：忽略构建产物、依赖目录、IDE 配置、环境文件（`.env`）等。

---

## 4. 编码约定

- **错误处理范式**：根据技术栈统一错误处理方式，避免 Agent 混用多种风格——  
  - Rust：优先 `Result<T, E>` + 自定义错误类型（推荐 `thiserror` / `anyhow`），明确可恢复错误与不可恢复错误的边界（`Result` vs `panic` / `unwrap`）。  
  - Go：`error` 接口 + `fmt.Errorf("...: %w", err)` 错误包装，明确 `panic` 仅用于不可恢复场景。  
  - JavaScript / TypeScript：区分可预期错误（`Result` / `Either` 模式或自定义 Error 子类）与意外异常（`try/catch`），避免裸 `catch(e)` 吞掉错误。  
  - Python：自定义异常层级，`raise` 携带上下文（`raise ... from err`），避免裸 `except:` 吞异常。  
- **日志与可观测性**：统一日志方案，防止 Agent 随意使用 `print` / `println!`——  
  - 日志框架：Rust `tracing` / `log`、JS `pino` / `winston`、Go `slog` / `zap`、Python `structlog` / `logging`。  
  - 日志级别约定：`ERROR`（需人工介入）、`WARN`（可自动恢复的异常）、`INFO`（关键业务事件）、`DEBUG`（开发调试）、`TRACE`（详细追踪），各级别写什么应示例说明。  
  - 结构化日志：输出 JSON 格式（便于 ELK / Grafana Loki 等采集），包含 trace_id / span_id（若为分布式系统）。  
  - 日志输出目标：开发环境 stderr，生产环境可配置（文件、远程采集等）。  
- **配置管理层次**：明确配置值的优先级与来源——  
  - 优先级（从高到低）：命令行参数 → 环境变量 → 配置文件 → 代码内默认值。  
  - 配置文件格式统一（如 TOML / YAML / JSON），位置约定（项目根目录 / `config/` 目录）。  
  - 敏感配置（密钥、Token）仅通过环境变量或密钥管理服务注入，不得写入配置文件或代码。  
- **依赖版本策略**：  
  - 版本范围约定：明确依赖版本约束写法（如 Rust `^` / `~` / 精确版本、JS `^` / `~` / 锁定、Go 最小版本语义）。  
  - dev-dependencies 划分原则：测试、基准、文档工具等仅开发期依赖，不得泄漏到正式构建。  
  - 安全审计：集成依赖审计工具（如 `cargo audit`、`npm audit`、`pip audit`、`govulncheck`），CI 中定期执行。

---

## 5. 测试策略

- **框架选型**：根据技术栈选择测试框架并写入文档——  
  - Rust：内置 `#[test]` + `#[cfg(test)]` 模块；集成 / 属性测试可选 `proptest`、`quickcheck`；裸机或 `no_std` 场景需评估 `custom_test_frameworks` 或 QEMU 仿真。  
  - JavaScript / TypeScript：`vitest`、`jest` 等；E2E 可选 `playwright`、`cypress`。  
  - Go：内置 `testing` 包 + `testify` 扩展。  
  - Python：`pytest` + `pytest-cov`。  
- **测试分层**：明确单元测试、集成测试、端到端测试的边界与职责——  
  - 单元测试：与源码同文件（如 Rust `#[cfg(test)]` 模块、Go `xxx_test.go` 同包），测试单个函数 / 结构体。  
  - 集成测试：放在项目级目录（如 Rust `tests/`、Go 同包 `test/`、JS `tests/integration/`），测试模块间交互。  
  - 端到端测试：独立目录或独立 crate / package，测试完整用户流程；若依赖外部服务，需提供 mock 或容器化环境。  
- **目录约定**：统一测试文件位置，避免散落各处（如 `tests/` 顶层放集成测试、`tests/fixtures/` 放测试数据、`tests/common/` 放共享辅助）。  
- **覆盖率**：  
  - 确定是否要求覆盖率门控（如 CI 中覆盖率低于阈值则失败）。  
  - 工具选型：Rust `cargo-llvm-cov` / `tarpaulin`、JS `c8` / `istanbul`、Go `go test -cover`、Python `pytest-cov`。  
  - 覆盖率报告输出位置与 CI 集成方式（如 Codecov、Coveralls 上传，或仅本地生成）。  
- **裸机 / 特殊环境**（若适用）：`no_std` 目标无法直接 `cargo test`，需文档说明替代方案（QEMU runner、硬件在环测试、Host 端 mock 测试等）。  
- **测试与构建的同步**：测试依赖（dev-dependencies）应与项目依赖同版本策略；CI 中测试命令应与本地开发一致。

---

## 6. CI 与本地钩子对齐

- **CI 流水线**（GitHub Actions / GitLab CI 等）：至少包含 lint（格式检查）、类型检查 / 静态分析、构建、测试。静态分析建议开启警告即失败。测试阶段应执行[测试策略](#5-测试策略)定义的全部测试层级（单元 → 集成 → E2E），并将覆盖率报告上传或归档。
- **pre-commit**：与 CI **同一套** lint / 检查命令；`pre-commit install` 且 `default_install_hook_types` 含 `pre-commit` 与 **`commit-msg`**（若需校验提交说明）。  
- **提交说明规则**：若需 Conventional Commits 或**自定义规则**（如 **英文第 1 行 + 中文第 2 行**），用 **仓库内脚本**（如 `scripts/*.py`）+ `commit-msg` 钩子实现；复杂规则勿强依赖仅校验首行的现成插件。  
- **`.gitmessage`**：`git config commit.template .gitmessage` 作为编辑器提示（可选）。

---

## 7. 面向 Agent 与团队的入口文档

- **README**：项目是什么、文档索引、如何构建、许可证一句、指向 CONTRIBUTING / AGENTS。  
- **AGENTS.md**：给 AI 的「必读缩略版」，至少包含以下内容：  
  - **项目身份**：项目是什么、技术栈、目录结构概览。  
  - **硬约束**：明确 Agent 的行为边界——  
    - 依赖方向：哪些模块可以依赖哪些模块（如 `core → adapter → infra`，禁止反向依赖）。  
    - 禁止引入的库：列出不允许添加的依赖及原因（如体积、许可证冲突、安全问题）。  
    - 文档修改权限：Agent 是否可以修改架构文档（`docs/ARCHITECTURE.md` 等）、是否需要人工审批后才能改动。  
    - 代码标注：Agent 生成的代码是否需要特定标注（如 `// AI-generated`、`<!-- agent:generated -->`），以便后续审计。
    - 安全红线：不得在代码或提交中包含密钥、Token、证书等敏感信息；不得绕过权限检查逻辑。  
    - 测试要求：Agent 生成新功能时必须同时编写测试，测试策略与分层遵循[测试策略](#5-测试策略)约定；最低覆盖率要求见该节覆盖率门控。
  - **验证命令**：Agent 完成修改后必须运行的检查命令（如 `cargo test`、`npm run lint`、`go vet`）。  
  - **文档与语言约定**：代码注释语言、提交信息格式、多语言文档同步规则。  
  - **协作入口**：安全政策与 PR 模板的位置。  
- **IDE / Agent 规则**（如 `.cursor/rules/*.mdc`、`.trae/rules/`、`.github/copilot-instructions.md`）：`alwaysApply` 核心一条；按路径拆分模块级规则；与 AGENTS.md 及书面规范互相链接避免漂移。规则中应引用 AGENTS.md 的具体章节，而非重复编写，确保单一信息源。

---

## 8. 代码托管平台配置（若使用）

- **`git init`** 后才有本地 pre-commit 钩子。  
- **Issue 模板**（如 `.github/ISSUE_TEMPLATE/`）+ **`config.yml`**。  
- **PR 模板**（如 `.github/pull_request_template.md`），检查项与 CI、文档约定一致。

---

## 9. 国际化与文档维护（按需）

- **目录约定**：如 `docs/*.md` 中文 + `docs/en/*.md` 同名英文；总览页成对维护（如 `docs/README.md` ↔ `docs/en/README.md`）。  
- **Agent 规则**：单独一条「中英同步」：改中文必改英文（或允许骨架 + 占位注释）。  
- **提交信息双语**：若要求「英文前、中文后、不同行」，需 **commit-msg 脚本** 落实结构；**语义一致**仍靠人工评审。  
- **文档维护策略**：代码变更时明确哪些文档必须同步更新——  
  - API 变更 → 更新 `docs/API.md` 及对应多语言版本。  
  - 架构调整 → 更新 `docs/ARCHITECTURE.md`，重大变更需人工审批。  
  - 新增依赖 → 更新架构文档的依赖清单与选型理由。  
  - PR 模板中可包含「文档同步」检查项，防止文档与代码漂移。

---

## 10. 版本发布与分支策略

- **语义化版本**：遵循 semver 规范（`MAJOR.MINOR.PATCH`），明确何为破坏性变更、何为向后兼容的新功能、何为修复。  
- **分支策略**：选定模型（如 GitHub Flow、Git Flow、Trunk-Based Development）并写入文档——  
  - 主干分支受保护，禁止直接推送；所有变更通过 PR 合入。  
  - 发布分支（若使用）：从主干切出，仅接受缺陷修复 cherry-pick。  
  - 功能分支命名约定（如 `feat/xxx`、`fix/xxx`、`docs/xxx`），便于 Agent 自动生成合规分支名。  
- **Tag 与 Release**：  
  - Tag 命名约定（如 `v1.2.3`、`release/v1.2.3`）。  
  - Release 自动化：CI 在 tag 推送后自动构建产物、生成 Release Notes（工具如 `release-please`、`cargo release`、`semantic-release`）。  
  - Release Notes 与 CHANGELOG 的关系：自动生成 vs 手动编写的边界。  
- **发布检查清单**：PR 模板中可包含发布前检查项（CHANGELOG 已更新、版本号已递增、迁移指南已编写等）。  
- **回滚策略**：发现严重问题时的回滚路径（重新部署上一版本、 revert commit、hotfix 分支等），提前约定比临时决策更可靠。

---

## 11. 开题前自检清单

- [ ] 架构 + 子系统文档可指向具体路径与开放项  
- [ ] LICENSE / CONTRIBUTING / SECURITY / CHANGELOG  
- [ ] 项目骨架 + 工具链锁定 + 环境变量模板 + 格式化配置 + 统一命令入口  
- [ ] 编码约定（错误处理、日志、配置管理、依赖策略）  
- [ ] CI + pre-commit（含 commit-msg 若需要）  
- [ ] 测试框架选型 + 分层约定 + 覆盖率门控  
- [ ] README + AGENTS.md + IDE/Agent 规则  
- [ ] 分支策略 + 版本发布 + Tag 约定  
- [ ] Issue/PR 模板（若上代码托管平台）  
- [ ] 文档多语言与维护策略  

---

## 12. 常见关键路径参考

```
docs/ARCHITECTURE.md          架构总览
docs/API.md                   API 设计（若适用）
docs/CONTRIBUTING.md          贡献指南
docs/SECURITY.md              安全政策
AGENTS.md                     Agent 入口文档
README.md                     项目入口
LICENSE                       许可证
CHANGELOG.md                  变更日志
.env.example                  环境变量模板
.editorconfig                 编辑器风格统一
rustfmt.toml / .prettierrc    格式化配置（按技术栈选择）
Makefile / Justfile           统一命令入口
config/                       配置文件目录（若适用）
tests/                        集成测试与测试辅助
tests/fixtures/               测试数据
.github/workflows/ci.yml      CI 流水线
.pre-commit-config.yaml       pre-commit 配置
scripts/commit_msg_check.py   提交说明校验脚本
.github/ISSUE_TEMPLATE/       Issue 模板
.github/pull_request_template.md  PR 模板
.cursor/rules/*.mdc           Agent 规则目录（示例）
```

将本文件随项目迭代更新即可。
