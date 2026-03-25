# Harness Engineering 是什么？一文讲清 AI 里的“运行外壳工程”

在 AI 里，**harness engineering** 不是一个单一、严格统一的标准术语，它通常指的是：为模型或 agent 搭建“运行外壳 / 执行框架 / 评测框架”的工程工作。

最近这个词更多被用在 **agent 场景**，表示把模型包在一层可控的系统里，让它能分解任务、调用工具、读写环境、做检查、重试、记录轨迹，并把复杂任务稳定跑起来。OpenAI 把它描述为一种 **agent-first** 的工程方式：把大目标拆成更小的设计、编码、测试、审查等构件，再通过 prompt 和流程把 agent 组织起来完成工作。Anthropic 也明确区分了 **agent harness/scaffold** 和 **evaluation harness**：前者负责让模型“能做事”，后者负责把评测任务端到端跑起来、打分、汇总结果。([OpenAI](https://openai.com/index/harness-engineering/))

## 一句话理解

可以把它理解成：

> **Prompt engineering** 是“怎么跟模型说话”，而 **harness engineering** 是“怎么把模型放进一个可执行、可观测、可验证的系统里”。

评测领域常见的 **lm-evaluation-harness** 也是这个思路：不是只问一个 prompt，而是要有任务加载、批量运行、后处理、评分、结果聚合这些完整基础设施。([GitHub](https://github.com/EleutherAI/lm-evaluation-harness))

## 它通常包含什么

一个完整的 harness，常见会有这几层：

### 1. 任务入口

把用户目标转换成结构化任务，比如“修复 bug”“总结文档”“跑 benchmark”“给仓库加测试”。OpenAI 提到的做法就是先把大目标拆成更小的 building blocks，再逐层放权给 agent。([OpenAI](https://openai.com/index/harness-engineering/))

### 2. 模型调用层

负责选模型、组消息、设系统提示词、控制温度、上下文窗口、工具调用格式等。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

### 3. 工具与环境层

给模型提供 shell、代码执行、搜索、数据库、浏览器、文件系统、测试框架等工具，并限制权限边界。Anthropic 对 agent harness 的定义就包括“处理输入、编排工具调用并返回结果”。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

### 4. 控制流

包括计划、执行、检查、回滚、重试、分支、超时、中断恢复、多步循环等。这部分往往是 harness engineering 的核心，因为真正难的是把长链路任务稳定跑完。([OpenAI](https://openai.com/index/harness-engineering/))

### 5. 验证与反馈

不是只看模型“说它完成了”，而是看环境里是否真的发生了预期结果。Anthropic 特别强调：评估 agent 时，最终要看环境中的 **outcome**，而不只是最后一句自然语言输出。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

### 6. 日志与可观测性

记录每一步 prompt、工具调用、输入输出、错误、耗时、token、状态转移，用于调试和回放。Anthropic 对 evaluation harness 的描述中就包括记录步骤与聚合结果。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

## 为什么它重要

很多人一开始做 AI 应用，关注点全在 prompt 上；但系统一复杂，瓶颈常常不在 prompt，而在 harness：

- 模型不知道什么时候该查资料、什么时候该运行代码
- 工具调用顺序混乱
- 失败后不会恢复
- 输出看起来合理，但环境状态其实没变
- 多轮之后上下文污染，任务越跑越偏

所以在 agent 系统里，**模型能力 × harness 设计** 才是真正的产品能力。Anthropic 甚至直接说，评估“一个 agent”时，本质上是在评估 **模型 + harness** 的组合。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

## 它和几个相近概念的区别

### 1）和 prompt engineering 的区别

Prompt engineering 主要研究“如何写提示词”；harness engineering 研究“如何设计整个执行系统”。前者更像话术，后者更像编排层、运行时和治理层。这个区别在 OpenAI 对 harness engineering 的描述里很明显：重点不只是提示，而是把设计、代码、测试、审查等环节组织成可复用流程。([OpenAI](https://openai.com/index/harness-engineering/))

### 2）和 agent engineering 的区别

很多时候两者几乎重叠。更细一点讲：

- **agent engineering** 偏产品/系统整体
- **harness engineering** 偏“运行外壳”和“控制框架”

也就是：agent 是“智能体”，harness 是“让它稳定工作的那套骨架和轨道”。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

### 3）和 evaluation harness 的区别

**evaluation harness** 是 harness engineering 的一个重要子类，专门用于评测。比如 EleutherAI 的 **lm-evaluation-harness**，它解决的是 benchmark 任务管理、运行、打分和复现问题。([GitHub](https://github.com/EleutherAI/lm-evaluation-harness))

## 常见应用场景

### A. AI Coding Agent

这是现在最典型的场景。一个 coding harness 往往会做这些事：

- 接收 issue / 需求
- 扫描代码库
- 生成计划
- 修改文件
- 运行测试 / lint / build
- 根据报错重试
- 生成 PR 说明
- 请求人工 review

OpenAI 的文章就是在讲这种 agent-first 开发方式；近期也有终端型 coding agent 论文把 harness、scaffolding、context engineering 放在一起讨论。([OpenAI](https://openai.com/index/harness-engineering/))

### B. LLM 评测与 benchmark

比如你要比较多个模型在 MMLU、GSM8K、代码任务上的表现，就需要 harness 负责数据集读取、prompt 模板、批处理执行、答案抽取、打分、汇总。EleutherAI 的 **lm-evaluation-harness** 就是这个方向的代表工具。([GitHub](https://github.com/EleutherAI/lm-evaluation-harness))

### C. 长时任务 agent

当任务要跑几小时甚至几天时，harness 需要处理 checkpoint、记忆压缩、阶段性总结、人工接管点、失败恢复。Anthropic 2025 年关于 long-running agents 的文章专门讨论了更有效的 harness。([Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents))

### D. 自动测试 / fuzzing / harness 生成

在软件测试研究里，harness 还可能指“测试驱动代码”本身。近期也有工作研究用 LLM 自动生成 testing harness 或 fuzzing harness。这个语境跟 agent harness 不完全一样，但本质上仍然是在构建“让能力可执行、可检验”的外壳。([arXiv](https://arxiv.org/html/2511.01104v1))

## 实际上怎么用

如果你是做 AI 应用，建议按下面顺序理解和落地。

### 第一层：先定义任务边界

先回答 4 个问题：

- 输入是什么
- 输出是什么
- 可以调用哪些工具
- 什么算成功

例子：

“让 AI 帮我修一个 Python bug”

不要只写成一句模糊 prompt，而要把它明确成：

- 输入：仓库路径、issue 描述、相关测试
- 工具：读文件、改文件、执行 pytest、grep、git diff
- 约束：不能联网、不能删测试、只能改 src/
- 成功标准：新增/原有测试全部通过，diff 不超过某阈值

这一步其实已经进入 harness engineering 了，因为你在定义运行环境而不是单轮对话。

### 第二层：设计执行循环

最常见的是这种 loop：

**Plan → Act → Observe → Verify → Retry / Finish**

示意：

```text
1. 读任务
2. 生成计划
3. 调工具执行一步
4. 读取结果
5. 判断是否继续
6. 完成后做最终验证
7. 输出结果和证据
```

关键点不是让模型“一次答对”，而是让它每一步都可检查。

### 第三层：把“说完成了”改成“被系统证明完成了”

这是很多人忽略的地方。

比如让 agent “预订航班”，最终判定不应是它回复“已完成预订”，而应是数据库里真的出现一条 reservation。Anthropic 在 agent eval 里明确强调了这种 outcome-based 判断。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

在工程里对应做法是：

- 代码任务：看测试结果
- 数据任务：看目标表是否生成
- 文件任务：看文件是否存在且内容符合 schema
- 工单任务：看 ticket 是否被正确更新
- 检索任务：看引用是否真实存在

### 第四层：为失败设计恢复机制

一个像样的 harness，至少要有：

- 超时
- 工具调用重试
- 语法错误修复回路
- 上下文太长时的总结压缩
- 达到风险动作前的人类确认
- 中途崩溃后的断点续跑

长时 agent 场景下，这些能力尤其重要。([Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents))

### 第五层：记录完整轨迹

至少记录：

- 系统提示词版本
- 每轮上下文
- 工具调用参数
- 工具返回结果
- 关键中间产物
- 最终打分与失败原因

没有这层，你几乎无法真正调 harness。

## 一个最小可用示例

假设你要做一个“文档问答 + 引用”的 agent harness，最简单的版本可以这样设计：

### 输入

- 用户问题
- 文档库

### 工具

- `search_docs(query)`
- `open_doc(id)`
- `answer_with_citations()`

### 控制流程

1. 先把问题改写为检索 query
2. 检索 top-k 文档
3. 读取最相关段落
4. 让模型基于证据回答
5. 检查每个结论是否都有引用
6. 没证据的句子删掉或标明不确定

这其实就是一个小型 harness：模型不是直接回答，而是在你规定的流程里回答。

## 一个 coding harness 的简化伪代码

```python
def run_bugfix_agent(issue, repo):
    context = load_repo_context(repo)
    plan = model.plan(issue, context)

    for step in plan:
        action = model.choose_action(step, context)

        if action.type == "read":
            result = read_file(action.path)
        elif action.type == "edit":
            result = apply_patch(action.path, action.patch)
        elif action.type == "test":
            result = run_pytest(action.target)
        else:
            result = {"error": "unknown action"}

        log_step(step, action, result)
        context = update_context(context, action, result)

        if result_failed(result):
            repair = model.repair(step, context, result)
            context = update_context(context, None, repair)

    final = run_pytest("all")
    return verify_and_summarize(final)
```

这里真正有价值的部分，不是 `model.plan()` 本身，而是：

- 能调用哪些动作
- 每个动作的权限边界
- 如何记录日志
- 如何判断失败
- 如何做最终验证

这些都是 harness engineering。

## 做 harness engineering 时的设计原则

### 1. 先流程化，再智能化

不要一开始就追求“超强自主 agent”。先把最稳定的固定流程搭起来，再让模型接管其中少数决策点。

### 2. 把自由度关进笼子

给模型越多权限，不一定越强，通常只是更不稳定。

好的 harness 会让模型在有限动作集合里选择，而不是无限制自由发挥。

### 3. 成功标准必须外部可验证

尽量用测试、数据库状态、文件校验、规则引擎，而不是自然语言自评。([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

### 4. 让每步都能回放

没有可观测性，就没有真正的工程化。

### 5. 先优化 harness，再怪模型

很多失败案例，本质上是流程、上下文、工具接口、反馈回路设计差，不是模型本身不行。OpenAI 和 Anthropic 的工程文章都在强调：agent 表现高度取决于外层 scaffold / harness。([OpenAI](https://openai.com/index/harness-engineering/))

## 常用工具和生态

如果你问“怎么上手”，可以按两个方向：

### 1）做评测 harness

最典型的是 **lm-evaluation-harness**。它适合：

- 跑标准 benchmark
- 比较不同模型
- 做可复现评测
- 管理 prompt 模板和打分流程

([GitHub](https://github.com/EleutherAI/lm-evaluation-harness))

### 2）做 agent harness

常见会基于：

- 工作流框架
- tool-calling API
- 沙箱执行环境
- 日志 / trace 系统
- 测试框架
- 人工审批节点

这类系统没有唯一标准库，但核心模式很一致：**模型 + 工具 + 控制流 + 验证 + 观测**。([OpenAI](https://openai.com/index/harness-engineering/))

## 一个实操落地路线

你可以这样练：

### 路线 1：从问答 harness 开始

做一个“先检索、再回答、必须带引用”的系统。目标是学会：

- 工具编排
- 证据约束
- 输出校验

### 路线 2：做一个代码修复 harness

让 agent 只能：

- 读文件
- 改文件
- 跑测试
- 看报错

目标是学会：

- 执行循环
- 环境验证
- 自动重试

### 路线 3：做一个 eval harness

选 50~200 条你自己的任务集，统一跑不同模型，看通过率、成本、延迟、失败模式。这样你会很快理解 harness 和 eval 的关系。([GitHub](https://github.com/EleutherAI/lm-evaluation-harness))

## 容易踩的坑

最常见的坑有这些：

- 把 harness engineering 误解成“高级 prompt engineering”
- 没有成功标准，只看模型最后一句话
- 工具太多、权限太大，导致 agent 乱跑
- 不记录中间过程，出了错无法定位
- 没有 recovery 机制，第一次失败就全盘崩
- 把 benchmark 分数当成真实业务表现，忽略环境交互

## 结论

如果要给一个最实用的定义，我会这样说：

> AI 里的 harness engineering，就是围绕模型搭建一套**可执行、可观测、可验证、可恢复**的运行框架，让模型从“会回答”升级为“能可靠完成任务”。

它既可以指：

- **agent harness**：让模型能工作
- **evaluation harness**：让模型能被系统性评测

而在今天的 AI 工程里，真正拉开效果差距的，往往不只是模型本身，而是这层 harness 设计。

---

如果你愿意，我可以下一篇继续写：**《从 0 到 1 搭一个 AI coding harness：架构图、模块拆分与代码骨架》**。
