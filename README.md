# LCDD: Lean Contract-Driven Documentation for Rust Agents

**LCDD (Lean Contract-Driven Documentation)** 是一套专为 AI Coding Agent（如 Claude Code, OpenCode, Cursor, Codex）在 Rust 项目中设计的**高质量注释与文档治理方法论**。

它彻底根治 Agent 常见的四大垃圾注释顽疾：
1. **啰嗦复述（Narrator / Translator Slop）**：边写代码边打草稿，用自然语言逐行翻译简单代码。
2. **自造名词（Hallucinated Terminology）**：凭空捏造不存在的架构概念与角色名词。
3. **格式非标（Style Mismatch）**：跨语言混淆，使用 Javadoc `@param` 或残缺的 Markdown 格式。
4. **领域错位（Scope Confusion）**：在模块级记流水账，在函数级泄露内部私有容器，在行内替代重构。

---

## 快速安装 (Install via Skills CLI)

本仓库已完全适配标准 [Skills CLI](https://skills.sh/) (`npx skills`)，支持一键安装到 Claude Code, Cursor, OpenCode, Codex 等 Agent 环境：

```bash
# 从 GitHub 一键安装全套 3 个 Skills (推荐)
npx skills add catitw/lcdd-skill -g -y

# 或按需安装单个 Skill
npx skills add catitw/lcdd-skill@rust-silent-coding -g -y
npx skills add catitw/lcdd-skill@rust-contract-docs -g -y
npx skills add catitw/lcdd-skill@rust-doc-deslop -g -y

# 本地路径安装 (当前开发机直接可用)
npx skills add /home/catitw/mypros/lcdd-skill -g -y
```

---

## 核心架构：三位一体闭环（The Triad Architecture）

LCDD 覆盖了软件工程的完整生命周期，将职责精准切分为三个各司其职的 Skill：

```
                ┌──────────────────────────────────────────────┐
                │             LCDD 三位一体体系                │
                └──────────────────────┬───────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         ▼                             ▼                             ▼
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
│  1. 防增量 (New) │          │  2. 立标准 (PR)  │          │ 3. 治存量 (Legacy)│
│rust-silent-coding│          │rust-contract-docs│          │ rust-doc-deslop  │
├──────────────────┤          ├──────────────────┤          ├──────────────────┤
│• 静默编码实现    │          │• 契约审计与文档  │          │• 逐模块渐进式清洗│
│• 零行内废话注释  │          │• RFC 1574 强结构 │          │• 废话注释物理删除│
│• 提炼私有函数自证│          │• 类型系统锚定术语│          │• 掩盖烂代码微重构│
│• 仅留 SAFETY 证明│          │• 真实 Doctest 门禁│          │• 散装安全升级标准│
└──────────────────┘          └──────────────────┘          └──────────────────┘
```

---

## Skills 职责与调用指南

### 1. `rust-silent-coding`（阶段 1：静默编码员）
* **触发时机**：编写新功能、重构内部实现、修复 Bug 时。
* **核心军规**：
  - **Zero Narrative Comments**：函数体内部严格禁止出现 `// Loop over`、`// Step 1`、`i += 1 // increment` 等解说或翻译类注释。
  - **代码自证（Self-Documenting）**：凡是想要写注释的地方，必须通过**提取具名私有辅助函数（Private Helper）**或**有意义的局部不可变变量绑定**来替代。
  - **极窄白名单**：唯一允许的行内注释仅限 `// SAFETY:` 健全性证明、外部缺陷 workaround（必须附带 issue URL）、复杂数学公式引用、跨线程内存序理由。

### 2. `rust-contract-docs`（阶段 2：契约审计员）
* **触发时机**：对外暴露公共 API、模块重构、准备提交 PR / 发布 Release 前。
* **核心选项：`--diff [base]`**：
  - 支持 `/skill:rust-contract-docs --diff`（或 `--diff main`），自动根据 git diff 扫描改动项。
  - **严格边界锁定**：**只对本次改动中新增或修改签名的 `pub` 项补全契约**。严禁擅自扩展到同文件中未改动的老函数，彻底防范 PR 膨胀。
* **核心军规**：
  - **术语类型锚定（Anti-Hallucination）**：所有出现的概念必须且只能使用 Rustdoc Intra-doc 语法（如 `[`SessionManager`]`）锚定到代码中的真实类型。任何胡乱编造的名词都会触发编译报错。
  - **四级注释分水岭（4-Tier Scope Hierarchy）**：
    - `//!` 模块级：专注心智模型与状态机拓扑，严禁列举函数清单。
    - `///` 项级：专注黑盒调用契约，严禁泄露内部私有实现。
  - **严格 RFC 1574 规范**：单行主动语态摘要、`# Errors`、`# Panics`、`# Safety`、以及完全可运行的 `# Examples`（用 `?` 处理错误，严禁伪代码）。
  - **硬性验收验证**：运行 `cargo test --doc` 与 `cargo doc --no-deps` 保证零文档腐败。
### 3. `rust-doc-deslop`（阶段 3：存量净化官）
* **触发时机**：接手历史遗留老项目、清理被 AI 注释污染的模块时。
* **核心军规**：
  - **必须给定模块或文件路径（Mandatory Path Scope）**：拒绝全库漫灌，严格以模块为单位推进（如 `/rust-doc-deslop src/codec`），不限单次变更文件数，保障模块整体性。
  - **五类语义分类（5-Class Triage）**：
    1. **Cat A (纯 AI 废话)** ➔ 物理删除。
    2. **Cat B (以注释掩盖烂代码)** ➔ 顺手微重构为具名私有辅助函数，然后删除注释。
    3. **Cat C (散装安全说明)** ➔ 规范化升级为标准 `// SAFETY:` 或 `/// # Safety`。
    4. **Cat D (真实 Why/不变量/Workaround)** ➔ 严格保留与精炼。
    5. **Cat E (脱节陈旧文档)** ➔ 重新与真实代码签名对齐。
  - **双模式支持**：支持 `--review`（仅出诊断报告）与默认 Apply（安全修改）。
  - **回归安全锁**：执行前后目标模块测试 `cargo test --lib <module>` 必须 100% 通过。

---

## 项目配套接入指引

为了让这套方法论在你的 Rust 项目中发挥最大威力，建议在项目中做如下极简配置：

### 1. `Cargo.toml` 编译器门禁（Rust 1.74+）
```toml
[lints.rust]
missing_docs = "warn"

[lints.clippy]
doc_markdown = "warn"
missing_errors_doc = "warn"
missing_panics_doc = "warn"
missing_safety_doc = "warn"
undocumented_unsafe_blocks = "deny"
```

### 2. 根目录 `AGENTS.md` / `CLAUDE.md` 约束（极简 18 行）
```markdown
## Documentation Discipline (LCDD)
1. Coding Phase: Use `rust-silent-coding`. ZERO inline explanation comments inside functions. Self-documenting naming and helper functions only.
2. Contract Phase: Use `rust-contract-docs --diff`. Ground all terms via intra-doc links [`Type`]. Touched public items must satisfy RFC 1574 (# Errors, # Panics, # Safety, runnable # Examples).
3. Remediation Phase: Use `rust-doc-deslop <path>`. Remediate module-by-module. Strip AI slop, refactor code deodorant, formalize safety proofs.
4. Verification: All changes must pass `cargo clippy`, `cargo test --doc`, and `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps`.
```

### 3. 本地验证一条龙
```bash
# 检查文档死链与造词
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps

# 运行所有文档示例测试
cargo test --doc

# 检查 unsafe 注释与 RFC 1574 结构
cargo clippy -- -D clippy::undocumented_unsafe_blocks -W clippy::missing_errors_doc -W clippy::missing_safety_doc
```

---

## 目录结构
```
/home/catitw/mypros/lcdd-skill/
├── README.md               # LCDD 方法论全貌与工程落地指南
├── rust-silent-coding/     # 阶段 1: 静默编码 Skill (防增量)
│   └── SKILL.md
├── rust-contract-docs/     # 阶段 2: 契约补全与审计 Skill (立标准)
│   └── SKILL.md
└── rust-doc-deslop/        # 阶段 3: 存量净化与微重构 Skill (治存量)
    └── SKILL.md
```
