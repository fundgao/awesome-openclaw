# OpenSpec vs Spec Kit：同为 SDD，但一个更“产品化”，一个更“方法论”

我看了两个项目的官方仓库和文档，结论先放前面：

- **OpenSpec** 更像一个“可直接拿来跑”的多助手 spec 工作流产品。
- **Spec Kit** 更像 GitHub 推出的 **SDD（Spec-Driven Development）方法论模板/脚手架**。

如果你想尽快在 **Cursor / Claude Code / Codex CLI** 等环境里落地，**OpenSpec 更偏实战集成**。如果你想采用一套更“规范、分阶段、治理感更强”的 Spec-Driven Development 流程，**Spec Kit 更偏方法学基座**。

参考：
- OpenSpec：https://github.com/Fission-AI/OpenSpec
- Spec Kit：https://github.com/github/spec-kit
- Spec Kit 站点：https://github.github.com/spec-kit/

---

## 一句话对比

- **Spec Kit**：强调“先原则、再规格、再计划、再任务、再实现”的阶段化流程，把 spec 当成开发主轴。
- **OpenSpec**：强调“用更轻量的 spec 层约束 AI 编码助手”，支持变更并行、工件可随时更新、工作流更灵活。

---

## 核心差异

### 1）产品定位

**Spec Kit**
- 官方定位：帮助团队开始实践 Spec-Driven Development，目标是“更快构建高质量软件”。
- 叙事更像：开发哲学 + 标准流程 + 模板。文档强调 intent-driven development、multi-step refinement、rich specification creation。

**OpenSpec**
- 官方定位：给 AI coding assistants 增加一层轻量 spec，解决“需求只存在聊天记录里导致结果不可预测”的问题。
- 更像：围绕 AI 编码助手做的工作流系统，重点是“对齐意图、管理变更、让产物可审查”。

---

### 2）流程设计

**Spec Kit：更线性、更分阶段**
- 推荐流程明确：
  1. constitution
  2. specify
  3. plan
  4. tasks
  5. implement
- 有 branch-based context awareness：命令会根据当前 Git 分支识别活跃 feature。

**OpenSpec：更灵活、更工件驱动**
- 强调每个 change 拥有独立文件夹：proposal/specs/design/tasks，可并行推进。
- 可以随时更新任意 artifact，不走 rigid phase gates。
- CLI 围绕状态与工件：status、instructions、templates、schemas、archive 等。

---

### 3）产物模型

**Spec Kit**
- 很重视规范化文档模板。
- spec template 里包含：
  - User Story（P1/P2/P3）
  - Independent Test
  - Acceptance Scenarios
  - Functional Requirements
  - Edge Cases
- 适合把需求写得完整、可验证、可拆解。

**OpenSpec**
- 核心模型：
  - **Specs**：系统当前行为的 source of truth
  - **Changes**：提议中的修改，独立存放，完成后再 merge / archive 回 specs
- 更偏“持续变更管理”，适合多个 feature/改动同时推进。

---

### 4）与 AI 工具的关系

**Spec Kit**
- 支持通过 `/speckit.*` 命令在多种 agent 中使用；Codex CLI 的 skills 模式用 `$speckit-*`。
- 更像“让 agent 遵循这套方法”。

**OpenSpec**
- 更直接强调“works with 20+ AI assistants via slash commands”。
- 更像“你已有工具上的一层 workflow”，而不是单纯的方法模板。

---

### 5）上手与依赖感受

**Spec Kit**
- 初始化方式偏脚手架（例如 uvx from git+...），并提供 Bash/PowerShell 双脚本。
- 从命令设计看更有“规定流程”的感觉。

**OpenSpec**
- `openspec init` 初始化；CLI 提供浏览、校验、归档、schema 管理等能力。
- 近期 1.0 重构为 action-based system，强调 AI 能理解项目状态和下一步可执行动作。

---

### 6）方法学气质

**Spec Kit 的气质**
- 更像“把传统产品/需求/架构/任务拆解流程，用 AI 重新执行一遍”。
- 更适合：
  - 想建立统一研发规约
  - 想先治理团队流程
  - 想把需求文档写得标准、完整、可审查

**OpenSpec 的气质**
- 更像“给 AI 编码加护栏，但不要太重”。
- 更适合：
  - 已经在大量使用 AI 编码助手
  - 想保留灵活迭代，不想被固定阶段卡住
  - 想把多个 change 并行管理起来

---

## 我的选型建议

选 **Spec Kit**，如果你更在意：
- 规范化流程
- 团队治理
- 从 principles → spec → plan → tasks 的严格链路
- spec 更完整、更像正式工程文档

选 **OpenSpec**，如果你更在意：
- 与现有 AI 编码工具快速结合
- 轻量但可审查的变更管理
- 多变更并行
- 工件随时迭代，不想被 rigid phase gates 束缚

---

## 一个简洁结论

你可以把它们理解成：
- **Spec Kit = GitHub 风格的 SDD 方法论框架**
- **OpenSpec = 面向 AI 编码助手的 SDD 工作流实现**

两者不是完全对立，很多理念同源；但体验上，**Spec Kit 更“制度化”**，**OpenSpec 更“产品化/工具化”**。

另外，OpenSpec README 里也把自己和 Spec Kit 对比为“Spec Kit 更 thorough 但更 heavyweight；OpenSpec 更轻、更自由迭代”。这个对比带有作者立场，但确实能反映两者定位差异。

---

## 可选后续

如果需要，我可以继续补一份：
- “在 Cursor / Claude Code / Codex CLI 里落地，二者目录结构与命令流怎么对应”的对照表
