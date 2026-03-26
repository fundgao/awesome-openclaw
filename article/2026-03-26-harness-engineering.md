# Harness Engineering：AI Agent 时代的系统工程实践全整理

2026 年 2 月，**Harness Engineering** 这个词突然在 AI 工程圈子里火了起来。Mitchell Hashimoto 在博客里提了这个说法，OpenAI 紧接着发了百万行代码的实验报告，Martin Fowler 也跟进写了深度分析——几周之内，这个术语就成了讨论 AI Agent 开发绕不开的话题。

本文把 Harness Engineering 的来龙去脉、核心方法和实战经验做了一次系统整理，重点拆解 OpenAI、Anthropic、Stripe 等团队踩过的坑和沉淀下来的做法。最后，我们对 8 个独立信息源进行交叉对比，梳理出业界已形成共识的六大方面、仍存分歧的四个议题，以及三个最值得探索的空白区。

---

# 第一部分：什么是 Harness Engineering

**Harness Engineering** 是指围绕 AI Agent（特别是 Coding Agent）设计和构建**约束机制、反馈回路、工作流控制和持续改进循环**的系统工程实践。

它解决的核心问题是：

> 当 AI Agent 拥有了强大的代码生成能力后，如何确保其输出的可靠性、一致性和长期可维护性。

“Harness” 本意是马具——缰绳、鞍具那一套东西，把马的力气引到正确方向上。拿来类比 AI Agent 挺合适：LLM 就像一匹蛮力十足但方向感不太行的马，跑得快但容易跑偏。

## 1.1 三层工程概念的关系

Harness Engineering 并不是凭空出现的，它是 **Prompt Engineering** 和 **Context Engineering** 的自然延伸。三者构成嵌套关系。

Phil Schmid 打了个比方：

> **模型是 CPU，Harness 是操作系统** —— CPU 再强，OS 拉胯也白搭。

mtrajan 的区分更直接：

- Context Engineering 管的是“给 Agent 看什么”
- Harness Engineering 管的是“系统怎么防崩、怎么量化、怎么修”

## 1.2 这个词是怎么火起来的

2025 年底已经有人零星提到过这个说法，但真正结晶成术语是 2026 年 2 月的事：

- **Mitchell Hashimoto** 在博客中首次明确命名了这一实践，提出核心理念：  
  > 每当发现 Agent 犯错，就花时间设计一个解决方案，确保 Agent 永远不会再犯同样的错误。
- **OpenAI** 数天后发布了  
  *Harness engineering: leveraging Codex in an agent-first world*
- **Ethan Mollick** 围绕 “Models, Apps, and Harnesses” 三个概念重组了他的 AI 指南框架
- **Martin Fowler** 发表深度分析文章，将 OpenAI 的 Harness 分类为三个领域：
  - Context Engineering
  - Architecture Constraints
  - Garbage Collection

---

# 第二部分：为什么需要 Harness Engineering

## 2.1 模型能力不是瓶颈

这一判断得到了量化验证：

- **Can.ac 实验**：仅改变 Harness 的工具格式（编辑接口），就在 16 个模型上显著提升了编码基准分数。效果最显著的 Grok Code Fast 1 从 **6.7%** 跃升至 **68.3%**，而模型权重完全没变。
- **LangChain 实验**：仅通过 Harness 改进，在 Terminal Bench 2.0 上从第 **30 名**跃升至第 **5 名**，同一模型提升了 **13.7 分**。

这些结果表明：

> 在纠结模型选择之前，先审视 Harness 设计，往往能获得更高的投资回报率。

OpenAI 团队说得很直接：

> 真正卡你的不是 Agent 写代码的能力，而是围绕它的结构、工具和反馈机制跟不上。

## 2.2 Agent 的典型失败模式

Anthropic 在做长时间运行 Agent 的过程中，总结了 Agent 常见的失败模式：

### 失败模式 1：试图一步到位（One-shotting）

Agent 倾向于一次做完所有事情，结果在实现进行到一半时上下文窗口就耗尽了。

### 失败模式 2：过早宣布胜利

项目后期，Agent 会看到已有进展就宣布任务完成，即使还有大量功能未实现。

### 失败模式 3：过早标记功能完成

没有明确提示时，Agent 写完代码就标记为“完成”，却没有做端到端测试。

### 失败模式 4：环境启动困难

每次新会话启动时，Agent 需要花费大量 token 搞清楚如何运行应用，而不是把时间花在开发上。

## 2.3 上下文窗口利用率的甜蜜区间

Dex Horthy 提出了一个很实用的经验观察：

> **上下文填得越满，LLM 输出质量越差。**

以 168K token 的上下文窗口为例，大约用到 40% 就开始走下坡路：

- **Smart Zone（前约 40%）**
  - 聚焦
  - 准确推理
  - 信息相关且精炼

- **Dumb Zone（超过约 40%）**
  - 幻觉
  - 循环
  - 格式错误的工具调用
  - 低质量代码

换句话说：

> 给 Agent 塞一堆 MCP 工具、冗长文档和累积对话历史，不会让它更聪明，反而会让它变笨。

---

# 第三部分：Harness Engineering 的四大支柱

综合 OpenAI、Anthropic、Carlini、Huntley、Horthy 等多个团队的实践，四种模式反复出现并形成收敛。这就是 Harness Engineering 的四大支柱。

## 3.1 支柱一：上下文架构（Context Architecture）

**核心原则：** Agent 应当恰好获得当前任务所需的上下文——不多不少。

每个团队都独立发现，将所有指令塞进一个文件无法扩展。解决方案是：

> **分层上下文与渐进式披露**

### 典型实践

- **OpenAI**：使用 `AGENTS.md` 作为动态反馈循环文件
- **Anthropic**：使用大量 README 和频繁更新的进度文件
- **Horthy**：倡导 “Frequent Intentional Compaction”
- **Vasilopoulos（2026 论文）**：提出三层上下文
  - Hot Memory
  - Domain Experts
  - Cold-Memory Knowledge

### 三层上下文体系

| 层级 | 加载时机 | 内容示例 | 上下文占用 |
|---|---|---|---|
| Tier 1：会话常驻 | 每次会话自动加载 | AGENTS.md / CLAUDE.md，项目结构概览 | 最小 |
| Tier 2：按需加载 | 特定子 Agent 或技能被调用时 | 专业化 Agent 的上下文、领域知识 | 中等 |
| Tier 3：持久化知识库 | Agent 主动查询时 | 研究文档、规格说明、历史会话 | 按需 |

## 3.2 支柱二：Agent 专业化（Agent Specialization）

**核心原则：** 专注于特定领域、拥有受限工具的 Agent，优于拥有全部权限的通用 Agent。

### 典型实践

- **Carlini**：将 Agent 专业化为
  - 编译器核心
  - 去重
  - 性能优化
  - 文档
- **Vasilopoulos**：部署了 19 个领域特定 Agent
- **Huntley**：用子 Agent 保持主 Agent 上下文清洁

### 典型角色分工

| Agent 角色 | 职责范围 | 工具权限 |
|---|---|---|
| 研究 Agent | 探索代码库、分析实现细节 | 只读 |
| 规划 Agent | 将需求分解为结构化任务 | 只读 |
| 执行 Agent | 实现单个具体任务 | 限定范围的读写 |
| 审查 Agent | 审计完成的工作，标记问题 | 只读 + 标记 |
| 调试 Agent | 修复审查发现的问题 | 限定范围修复 |
| 清理 Agent | 对抗熵积累，清理低质量代码 | 读写 |

## 3.3 支柱三：持久化记忆（Persistent Memory）

**核心原则：** 进度持久化在文件系统上，而非上下文窗口中。

每次新 Agent 会话从零开始，通过文件系统制品重建上下文。

### Anthropic 的经典方案

#### 初始化 Agent

首次会话使用专门 prompt，要求模型建立初始环境，包括：

- `init.sh`
- `claude-progress.txt`
- 初始 git 提交

#### 编码 Agent

后续每次会话要求模型在做出增量进展的同时，留下结构化更新。

### 典型启动流程

1. 运行 `pwd` 查看工作目录
2. 读取 `git log` 和进度文件
3. 读取 feature list，选择最高优先级任务
4. 启动开发服务器，跑基础端到端测试
5. 确认基本功能正常后开始新开发

### 关键发现

> 使用 JSON 格式追踪 feature 状态比 Markdown 更有效，因为 Agent 更不容易不恰当地修改结构化数据。

## 3.4 支柱四：结构化执行（Structured Execution）

**核心原则：** 将思考与执行分离。

所有团队都施加了刻意的执行序列：

> **理解 → 规划 → 执行 → 验证**

### 典型实践

- **OpenAI**：声明式 prompt + 反馈回路
- **Huntley**：将规划模式与构建模式分离
- **Horthy**：Research-Plan-Implement 工作流

### 人工检查点的价值

> 审查计划远比审查代码快。

当规格正确时，实现自然更可靠；当规格有误时，可以在 500 行代码生成前及时纠正。

---

# 第四部分：先进团队的实战案例

## 4.1 OpenAI：百万行代码的零手写实验

### 实验概况

| 指标 | 数值 |
|---|---|
| 团队规模 | 3 名工程师 |
| 持续时间 | 5 个月（2025.8 起） |
| 代码规模 | 约 100 万行 |
| 手写代码 | 0 行 |
| 合并 PR 数 | 约 1,500 个 |
| 日均 PR / 人 | 3.5 个 |
| 效率提升 | 约 10 倍 |

### 五大 Harness 原则

1. **设计环境，而非编写代码**  
   当 Agent 卡住时，不是“更努力”，而是诊断“缺少什么能力”。

2. **机械化地执行架构约束**  
   依赖方向：  
   `Types → Config → Repo → Service → Runtime → UI`

3. **将代码仓库作为唯一事实源**  
   不要把关键知识放在 Slack 或 Google Docs。

4. **将可观测性连接到 Agent**  
   连接 Chrome DevTools、日志、指标、DOM 快照。

5. **对抗熵**  
   用后台任务自动清理 “AI Slop”。

### 自定义 Linter 的巧妙设计

当 Agent 违反架构约束时，错误消息不仅标记违规，还会告诉 Agent **如何修复**。

## 4.2 Anthropic：16 个 Agent 构建 C 编译器

### 项目数据

| 指标 | 数值 |
|---|---|
| 持续时间 | 约 2 周 |
| 并行 Agent 数 | 16 个 Claude Opus 4.6 实例 |
| Claude Code 会话数 | 约 2,000 |
| Rust 代码量 | 100,000 行 |
| GCC torture test 通过率 | 99% |
| 可编译真实项目 | 150+ |
| 总 API 成本 | 约 \$20,000 |

### 关键 Harness 设计

- **上下文窗口污染缓解**
  - 最小化控制台输出
  - 日志写入文件
  - grep 友好的错误格式
  - 预计算聚合统计

- **Agent 时间盲区**
  - 模型没有时间感
  - 解决方案：确定性测试子采样

- **专业化而非通用化**
  - 核心编译器
  - 去重
  - 性能优化
  - 代码质量
  - 文档

- **CI 作为 Harness**
  - 用 Harness 层面的解决方案应对模型层面的问题

## 4.3 Anthropic：长时间运行 Agent 的有效 Harness

### 核心痛点

长时间运行的 Agent 必须在多个独立会话里工作，而每次新会话对上一轮进展一无所知。

### 两阶段解决方案

#### 初始化 Agent

建立初始环境，包括：

- `init.sh`
- `claude-progress.txt`
- 初始 git 提交

#### 编码 Agent

每次后续会话要求模型做出增量进展，并留下结构化更新。

### 四大失败模式与对策

| 问题 | 初始化 Agent 的行为 | 编码 Agent 的行为 |
|---|---|---|
| 过早宣布项目完成 | 建立结构化 JSON 功能列表 | 启动时读取功能列表，选择单个功能 |
| 环境中遗留 bug 或未文档化进度 | 编写初始 git 仓库和进度文件 | 开始时读取进度和日志，结束时提交更新 |
| 过早标记功能完成 | 建立功能列表文件 | 自我验证后再标记为 passing |
| 不知道如何运行应用 | 编写 `init.sh` | 会话开始时读取 `init.sh` |

### 浏览器自动化测试

通过 Puppeteer MCP 等工具，Agent 能识别和修复仅从代码层面看不到的 bug。

## 4.4 Stripe：千级 PR 的 Minions 系统

Stripe 的 **Minions** 是较成熟的**无人值守并行化**实践：

开发者在 Slack 里发任务，Agent 从写代码到跑通 CI 再到提 PR 全程包办，人只在最后审查。

### 关键架构要素

- **Toolshed MCP 服务器**
  - 提供近 500 个工具
- **隔离的预热 Devbox**
  - 与人类工程师相同开发环境
  - 与生产和互联网隔离

### 关键洞察

> Agent 需要和人类工程师一样的上下文和工具，不是事后补上的集成，而是一开始就得是一等公民。

---

# 第五部分：Harness 的核心组件详解

## 5.1 AGENTS.md —— Agent 的活文档

AGENTS.md 是一个新兴的开放约定，本质上是给 AI Agent 的 README。

### 关键特性

- 不是一次性写完的静态文档
- 每当 Agent 犯错时都要更新
- 文档成为反馈循环，而非静态制品
- 简单问题通过更新 AGENTS.md 解决
- 复杂问题需要工具层面的解决方案

### OpenAI 的进阶实践

不是维护一个巨大的指令文件，而是维护一个小型 `AGENTS.md`，指向更深层的事实源：

- 设计文档
- 架构图
- 执行计划
- 质量评级

并由后台 Agent 定期扫描过期文档并提交清理 PR。

## 5.2 架构约束与自动化执行

### 分层架构依赖方向强制执行

```text
Types → Config → Repo → Service → Runtime → UI
```

任何违反这一方向的代码都由自定义 Linter 自动检测和阻止。

### Linter 错误消息即修复指令

传统 Linter 只标记违规，OpenAI 的自定义 Linter 会在错误消息中直接包含修复方法。

### 结构测试

Martin Fowler 提到 ArchUnit 等结构测试框架的潜力，它们可以验证代码库的架构约束是否被遵守。

## 5.3 可观测性集成

OpenAI 团队将可观测性接入 Agent 工作流：

- 浏览器自动化（Puppeteer MCP）
- Chrome DevTools 集成
- 日志和指标查询
- 遥测驱动的 bug 修复

## 5.4 熵管理与“垃圾回收”

Agent 生成的代码会以不同于人类编写的方式积累技术债。OpenAI 将其称为 **熵（Entropy）**。

### 解决方案：定期运行的“垃圾回收” Agent

- 扫描文档不一致
- 检测架构约束违规
- 清理冗余或低质量代码
- 让清理吞吐量与代码生成吞吐量成比例

---

# 第六部分：工程师角色的转变

## 6.1 从写代码到设计环境

OpenAI 和 Anthropic 的实践都指向一个结论：工程师的工作正在分成两块：

### 第一部分：构建环境

当 Codex 卡住时，团队将其视为环境设计问题——诊断 Agent 缺少什么才能可靠继续工作。

### 第二部分：管理工作

Greg Brockman 建议每个团队指定一名 **“Agent 队长”**，负责思考 Agent 如何融入团队工作流。

> Agent 的失败告诉你环境缺少什么；更好的环境又让管理更顺畅。

## 6.2 规划是新的编码

越来越多开发者强调与 AI 合作时**前期规划的广度**。

Boris Tane 的总结：

> 永远不要让 Agent 在你审查和批准书面计划之前写代码。

Anthropic 的做法更进一步：初始化 Agent 就生成超过 200 个功能的结构化列表，且全部初始标记为 failing。

## 6.3 并行编排与“两种管理风格”

### 两种并行工作模式

| 模式 | 描述 | 优势 | 要求 |
|---|---|---|---|
| 有人值守并行化 | 主动管理多个 Agent 会话，按需重定向 | 控制更多，问题发现更早 | 认知负担高 |
| 无人值守并行化 | 发布任务后离开，Agent 自主完成到 PR | 可扩展性更强 | Harness 足够成熟 |

Stripe 能做到无人值守并行化，是因为已经建立了 Toolshed、预热 Devbox 和紧密的 CI 集成。

---

# 第七部分：业界趋势与前瞻

## 7.1 Harness 将成为新的服务模板

未来，团队可能会像今天使用 Service Template 一样，从一组预制 Harness 中选择。

### Harness 模板可能包含

- 自定义 Linter 规则
- 结构测试
- 基础上下文和知识文档
- 额外上下文提供者
- 预配置的 CI/CD 管道

## 7.2 技术栈和代码拓扑的收敛

AI 可能推动团队走向：

- 更少的技术栈选择
- 更稳定的数据结构
- 更清晰的模块边界

## 7.3 Harness 应趋向简化而非复杂化

Phil Schmid 的分析指出：

> Manus 团队半年内重写了五次 Harness，但每次方向都是简化而不是加复杂度。

经验教训是：

> 随着模型能力提升，Harness 应该越做越薄。

## 7.4 更好的模型让 Harness 更重要而非更不重要

Carlini 的 C 编译器项目给了直接证据：

> 模型越强，能给的自主权越大，护栏就得越好。

## 7.5 棕地项目的改造挑战

所有公开成功案例大多涉及绿地项目或从零构建的 Harness。

而如何把这些方法应用到：

- 十年历史代码库
- 无架构约束
- 测试不一致
- 文档残缺

的棕地项目上，仍是一个更复杂的问题。

---

# 第八部分：最佳实践总结

## 8.1 立即可行的行动清单

- 创建并维护 `AGENTS.md`
- 在仓库中建立单一事实源
- 构建自定义 Linter，并在错误消息中嵌入修复指令
- 为 Agent 提供端到端测试工具
- 实施增量执行策略
- 分层管理上下文
- 上下文利用率保持在 40% 以下
- 建立定期“垃圾回收”机制

## 8.2 Harness 成熟度评估模型

| 阶段 | 特征 | 工程师角色 |
|---|---|---|
| Level 0：无 Harness | 直接给 Agent prompt，无结构化约束 | 手动写代码 + 偶尔用 AI |
| Level 1：基础约束 | AGENTS.md + 基础 Linter + 手动测试 | 主要写代码，AI 辅助 |
| Level 2：反馈回路 | CI/CD + 自动化测试 + 进度追踪 | 规划 + 审查为主 |
| Level 3：专业化 Agent | 多 Agent 分工 + 分层上下文 + 持久化记忆 | 环境设计 + 管理为主 |
| Level 4：自治循环 | 无人值守并行化 + 自动熵管理 + 自修复 | 架构师 + 质量把关者 |

## 8.3 关键 Harness 组件检查清单

| 组件 | 用途 | 优先级 |
|---|---|---|
| AGENTS.md / CLAUDE.md | 会话常驻上下文，动态反馈循环 | P0 |
| 自定义 Linter + 结构测试 | 机械化执行架构约束 | P0 |
| CI/CD 管道 | 自动化测试和验证反馈 | P0 |
| 进度文件 | 跨会话持久化记忆 | P1 |
| 功能列表文件 | 结构化完成标准 | P1 |
| 浏览器自动化 | 端到端测试验证 | P1 |
| 可观测性集成 | Agent 可查询日志 / 指标 | P2 |
| 熵管理 Agent | 定期清理低质量代码 | P2 |
| 专业化子 Agent | 分工协作减少上下文污染 | P2 |
| MCP 工具集成 | 连接外部工具和数据 | P2 |

---

# 第九部分：开放问题与挑战

## 9.1 代码可维护性的长期隐患

Greg Brockman 抛出了一个尚无答案的问题：

> 怎么防止“功能没问题但维护性很差”的代码渗透进代码库？

## 9.2 规模化验证

Birgitta Böckeler 对 OpenAI 报告的批评非常关键：

- 报告缺乏对功能和行为的验证
- 即使有浏览器自动化，也可能遗漏某些 bug

## 9.3 棕地项目改造

如何为已有十年历史的代码库引入 Harness Engineering，而不是被警报淹没，仍是开放问题。

## 9.4 文化采纳

所有成功案例都指向一个事实：

> 这些东西不会自己出现，得有人去建。

但好消息是，这些投入具备复利效应。

---

# 第十部分：业界共识与分歧全景

综合 OpenAI、Anthropic、Stripe、Martin Fowler、Mitchell Hashimoto、Charlie Guo、Alex Lavaee、SmartScope 等 8 个独立信息源的交叉对比，目前可归纳如下。

## 10.1 六大共识

1. **瓶颈在基础设施，不在模型智能**
2. **文档必须是活的反馈循环**
3. **思考与执行必须分离**
4. **上下文不是越多越好**
5. **约束必须机械化执行**
6. **工程师角色正在从“写代码”转向“设计环境 + 管理工作”**

## 10.2 四大分歧

### 分歧 1：Harness 应该越做越复杂还是越做越简单？

- Manus 倾向于简化
- OpenAI 倾向于深度定制
- 场景不同，方法不同

### 分歧 2：单 Agent 还是多 Agent 架构？

- 大项目偏多 Agent
- 小项目单 Agent 也可能够用
- 仍无统一结论

### 分歧 3：人类应该介入到什么程度？

- 从 Hashimoto 的深度参与
- 到 Stripe / Huntley 的高度自动化
- 取决于 Harness 成熟度和信任水平

### 分歧 4：术语边界怎么画？

- Harness ⊇ Context ⊇ Prompt
- 或者 Context 与 Harness 互补
- 术语本身仍在演化

## 10.3 三大空白区

1. **棕地项目的 Harness 改造**
2. **功能和行为验证的系统化方案**
3. **AI 生成代码的长期可维护性**

## 10.4 共识与分歧速查表

| 领域 | 共识程度 | 核心结论 |
|---|---|---|
| 基础设施 > 模型智能 | ★★★★★ | 全面共识 |
| 思考与执行分离 | ★★★★★ | 全面共识 |
| 文档作为活反馈循环 | ★★★★☆ | 强共识 |
| 上下文不是越多越好 | ★★★★☆ | 强共识 |
| 约束必须机械化执行 | ★★★★☆ | 强共识 |
| 工程师角色转变 | ★★★★☆ | 强共识 |
| 人类介入程度 | ★★★☆☆ | 部分共识 |
| Harness 简化 vs 精细化 | ★★☆☆☆ | 存在分歧 |
| 单 Agent vs 多 Agent | ★★☆☆☆ | 存在分歧 |
| 术语边界定义 | ★★☆☆☆ | 存在分歧 |
| 功能验证方法 | ★☆☆☆☆ | 严重缺失 |
| 棕地项目改造 | ★☆☆☆☆ | 完全空白 |
| AI 代码长期可维护性 | ★☆☆☆☆ | 完全空白 |

---

# 总结

Harness Engineering 标志着 AI 辅助开发从“让模型写代码”到“设计让模型可靠工作的系统”的范式转变。

这不是等更强模型出来就能自动解决的事——**模型越强，Harness 反而越重要。**

六大共识已经比较清晰：

- 基础设施是瓶颈
- 文档要活
- 思考与执行分离
- 上下文不是越多越好
- 约束必须自动化
- 工程师角色正在转变

但仍有三大空白区尚待填补：

- 棕地项目改造
- 功能验证体系
- AI 代码长期可维护性

如果只记一句话：

> **瓶颈不在智能，而在基础设施。**

---

# 参考文献与资料链接

## 核心文章与报告

1. **OpenAI** — *Harness engineering: leveraging Codex in an agent-first world*
2. **Anthropic** — *Effective harnesses for long-running agents*
3. **Anthropic** — *Demystifying evals for AI agents*
4. **Nicholas Carlini (Anthropic)** — *Building a C Compiler with Claude*
5. **Martin Fowler** — *Harness Engineering*
6. **Martin Fowler** — *Context Engineering for Coding Agents*

## 深度分析与综述

1. **Charlie Guo** — *The Emerging "Harness Engineering" Playbook*
2. **SmartScope** — *What Is Harness Engineering*
3. **Alex Lavaee** — *How to Harness Coding Agents with the Right Infrastructure*
4. **Addy Osmani** — *Agentic Engineering*

## 实践者博客与案例

1. **Mitchell Hashimoto** — *My AI Adoption Journey*
2. **Geoffrey Huntley** — *Ralph Methodology*
3. **Dex Horthy** — *Advanced Context Engineering for Coding Agents*
4. **Stripe** — *Minions: Stripe's one-shot, end-to-end coding agents*
5. **Decision (Substack)** — *Harness Engineering: How to Supervise Code You Can't Read*

## 学术论文

1. **Vasilopoulos et al. (2026)** — *Codified Context: Three-Tier Context Infrastructure for Agent-Driven Development*

## 工具与框架

1. **Atomic CLI**
2. **Harbor Framework**
3. **Claude Agent SDK**

## 新闻报道与社区讨论

1. **InfoQ** — *Sixteen Claude Agents Built a C Compiler without Human Intervention*
2. **Ars Technica** — *Sixteen Claude AI agents working together created a new C compiler*
3. **01.me** — *Claude's Context Engineering Secrets: Best Practices Learned from Claude*
