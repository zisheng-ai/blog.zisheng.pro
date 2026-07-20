---
title: Agentic System Engineer 成长路线图与学习指南
description: 从这张能力架构图出发，结合全栈开发背景，规划一条系统化的 Agent 工程师成长路径。
date: 2026-06-08 22:00:00
categories:
  - AI
tags:
  - Agent
  - 学习路线
  - AI
  - 职业规划
cover: https://img.alicdn.com/imgextra/i2/O1CN01cbuFjJ1RXTCX2ovap_!!6000000002121-2-tps-1672-941.png
---

最近在整理 AI Agent 方向的知识体系时，我和 GPT 反复聊了几个来回，生成了一张非常系统的 **Agentic System Engineer 专家能力架构图**。这张图把从底层工程基础到顶层专家影响力，拆成了七个层次（L1-L7），同时辅以横向的工程素养和业务素养两条维度。

作为全栈开发者，我大部分时间在 JavaScript/TypeScript 和 Node.js 生态里工作，正在往 AI Agent 方向拓展。这张图恰好给了我一个清晰的坐标系——知道自己现在在哪儿，接下来该往哪儿走。这篇文章既是学习笔记，也是一份面向全栈开发者的成长计划。

<!-- more -->

## 能力架构总览

这张图的核心路径是：**LLM 理解 → Agent 核心 → Runtime / Infra → Grounding & Execution → Evolution → 战略影响力**。七个层级层层递进，每个层级下面都有具体的能力点。

| 层级 | 名称 | 核心关键词 |
|------|------|-----------|
| L1 | 通用工程与系统基础 | 编程语言、算法、OS、网络、数据库、云原生 |
| L2 | AI / LLM 能力基础 | Transformer、Prompt Engineering、Function Calling、多模态、模型评估 |
| L3 | Agent 核心认知 | Planning、Memory、Tool Use、State/Context、长期任务、安全约束 |
| L4 | Runtime 与基础设施 | Agent Runtime、工作流、调度、Sandbox、Checkpoint、可观测性 |
| L5 | 环境交互与执行能力 | Browser Agent、Vision/DOM Grounding、Tool/API 编排、RAG、多 Agent 协同、Human-in-the-loop |
| L6 | 自进化与优化系统 | Eval 评测、Trajectory/Trace、反馈闭环、Skill Library、Self-Evolve、RL |
| L7 | 专家影响力与战略 | 技术战略、系统架构、方法论、开源影响力、团队带教 |

两个横向维度同样关键：
- **横向工程素养**：可靠性、可维护性、性能、成本、安全
- **横向业务素养**：产品理解、领域建模、沟通表达、文档写作、业务落地

下面结合我的实际情况，逐层拆解学习计划。

## L1：通用工程与系统基础（已具备，持续巩固）

作为全栈开发者，L1 的大部分能力已经覆盖。这里需要重点补充的是：**面向 Agent 系统的工程思维**。

- **编程语言**：JavaScript/TypeScript 是主力，正在强化 Python（FastAPI、AI Agent 生态），Java 也在学习中。Agent 系统开发中，Python 是 AI 侧的主力，JS/TS 适合前端集成和工具链。
- **数据结构与算法**：基础已具备，需要补充的是**图遍历、状态空间搜索**等 Agent Planning 相关算法。
- **操作系统与网络**：已有基础，重点看**进程隔离、容器化**等与 Sandbox 安全执行相关的内容。
- **数据库/存储**：关系型和非关系型都有使用经验，需要补充的是**向量数据库**（如 Milvus、Pinecone）和 Agent 记忆持久化方案。
- **云原生/分布式系统**：Docker、K8s 有基础，需要深入的是**Serverless（FaaS）**和**事件驱动架构**，这对 Agent Runtime 的弹性调度很重要。

**行动项**：
1. 系统学习向量数据库原理与选型
2. 深入 Docker 安全隔离机制，为后续 Sandbox 设计打基础
3. 补充图算法（DFS/BFS、A*）在 Agent 路径规划中的应用

## L2：AI / LLM 能力基础（正在强化）

这是从传统工程师转向 AI 工程师的关键一层。

- **Transformer / 推理基础**：理解 Attention 机制、KV Cache、Token 生成过程。不需要推导所有公式，但要理解**模型为什么会产生幻觉、为什么长文本会丢失上下文**。
- **Prompt / Context Engineering**：已经大量使用，但要体系化。需要掌握 Chain-of-Thought、ReAct、Self-Consistency 等 Prompt 范式。
- **Function Calling**：这是 Agent 调用工具的基石。要深入理解不同模型（GPT、Claude、Llama）的 Function Calling 实现差异。
- **多模态理解**：Vision 能力正在快速发展，需要了解 GPT-4V、Claude 3 的视觉输入处理机制。
- **模型评估**：掌握 BLEU、ROUGE、BERTScore 等传统指标，更要关注**Agent 专用评估方法**（如成功率、任务完成率）。
- **推理成本意识**：Token 计费、上下文窗口成本、模型选型策略。生产环境中成本意识决定方案可行性。

**行动项**：
1. 精读《Attention Is All You Need》，理解 Transformer 核心机制
2. 系统学习 Prompt Engineering 体系，整理一份 Prompt 模式库
3. 动手实现一个支持 Function Calling 的 Agent 原型
4. 对比 GPT-4、Claude 3.5、Llama 3 在相同任务上的成本与效果

## L3：Agent 核心认知（重点突破层）

这是 Agent 系统的"内核"。如果说 L2 是理解模型，L3 就是理解 Agent 的本质。

- **Planning**：Agent 如何拆解任务、制定计划。关键概念包括 Task Decomposition（任务分解）、Hierarchical Planning（层次规划）、ReAct（推理+行动循环）。
- **Memory**：短期记忆（上下文窗口管理）、长期记忆（向量存储、知识图谱）。需要理解**记忆的检索、压缩、遗忘机制**。
- **Tool Use**：Agent 如何发现、选择、调用工具。MCP（Model Context Protocol）是新兴标准，值得关注。
- **State / Context 管理**：Agent 的状态机设计、上下文窗口的滑动窗口策略、多轮对话的状态保持。
- **长期任务执行**：多步骤任务的持久化执行、错误恢复、断点续传。这是从 Demo 走向生产的关键。
- **安全与约束**：输入过滤、输出审查、权限控制、沙箱隔离。Agent 的自主性越高，安全风险越大。

**行动项**：
1. 用 Python 实现一个 ReAct Agent，包含 Planning + Tool Use + Memory
2. 研究 LangChain、Mastra 的 Memory 实现，对比不同方案
3. 学习 MCP 协议，实现一个自定义 MCP Server
4. 设计一个带状态持久化的 Agent，支持任务中断和恢复

## L4：Runtime 与基础设施（从原型到生产）

L3 关注的是单个 Agent 的认知能力，L4 关注的是**如何让 Agent 系统在生产环境中稳定运行**。

- **Agent Runtime**：Agent 的执行引擎。需要了解 LangChain、Mastra、AutoGen 等框架的 Runtime 设计。
- **工作流 / 状态机**：用状态机或 DAG 编排 Agent 的执行流程。如 LangGraph、Temporal 等。
- **调度与队列**：多 Agent 任务的排队调度、优先级管理、并发控制。
- **Sandbox / 隔离**：代码执行的安全沙箱（如 E2B、Docker Sandbox）、文件系统隔离、网络隔离。
- **Checkpoint / Durable Execution**：执行状态持久化，支持故障恢复。Temporal、Durable Objects 等技术。
- **可观测性 / 监控**：Agent 的调用链路追踪、成本监控、效果评估指标收集。OpenTelemetry、LangSmith 等工具。

**行动项**：
1. 学习 LangGraph，用状态机方式重构 L3 的 ReAct Agent
2. 研究 E2B 或 Docker 的代码执行沙箱方案，实现安全代码运行环境
3. 集成 LangSmith 或 Helicone，建立 Agent 的监控和追踪体系
4. 学习 Temporal 或类似框架，实现 Durable Execution

## L5：环境交互与执行能力（从对话到行动）

这是 Agent "落地"的关键层。Agent 不仅要会思考，还要能与真实世界交互。

- **Browser Agent**：用 Agent 控制浏览器完成网页操作。browser-use、Playwright、Puppeteer 是核心工具。UI-TARS 是新兴的视觉驱动方案。
- **Vision / DOM Grounding**：让 Agent 理解网页视觉内容和 DOM 结构。结合多模态模型的视觉能力。
- **Tool / API Orchestration**：编排多个 API 调用，处理依赖关系、错误回退。MCP 协议在这里扮演重要角色。
- **RAG / 检索增强**：结合向量检索和 LLM，让 Agent 访问外部知识。Sirchmunk（Embedding-Free RAG）是值得关注的新方向。
- **多 Agent 协同**：多个 Agent 分工协作，如经理 Agent 分配任务、专家 Agent 执行。AutoGen、CrewAI 等框架。
- **Human-in-the-loop**：关键决策点引入人工确认，平衡自动化和可控性。

**行动项**：
1. 用 browser-use 实现一个自动化的网页数据抓取 Agent
2. 集成 RAG 能力，让 Agent 能查询文档知识库（尝试 Sirchmunk）
3. 用 AutoGen 或 CrewAI 搭建一个多 Agent 协作系统
4. 实现 Human-in-the-loop 机制，设计关键决策的人工确认流程

## L6：自进化与优化系统（从静态到动态）

让 Agent 系统能够持续学习和改进，而不是每次都从零开始。

- **Eval 评测体系**：建立 Agent 任务的评估数据集和指标。这是优化的前提。
- **Trajectory / Trace**：记录 Agent 的完整执行轨迹，用于分析和调试。
- **反馈闭环**：将执行结果反馈给 Agent，形成学习循环。成功/失败案例的积累。
- **Skill Library**：Agent 学习的技能库，可复用的工具组合和策略模板。
- **Self-Evolve**：Agent 自主改进自身 Prompt、工具选择策略。
- **策略优化 / RL**：用强化学习优化 Agent 的决策策略。DPO、PPO 等算法在 Agent 优化中的应用。

**行动项**：
1. 为自己的 Agent 项目建立 Eval 数据集和评估流程
2. 设计反馈收集机制，记录每次任务的执行轨迹和结果
3. 实现 Skill Library，积累可复用的 Agent 技能模板
4. 了解 DPO（Direct Preference Optimization）在 Agent 优化中的应用

## L7：专家影响力与战略（长期目标）

这是从工程师到专家、从技术执行到技术领导的跨越。

- **技术战略**：Agent 技术的演进方向判断、技术选型决策、团队技术路线规划。
- **系统架构**：设计大规模 Agent 系统的整体架构，考虑扩展性、可靠性、成本。
- **方法论输出**：沉淀 Agent 开发的工程方法论，形成可复用的框架和最佳实践。
- **开源影响力**：参与或主导开源 Agent 项目，建立技术影响力。FastGPT、Mastra 等。
- **团队带教**：培养团队成员的 Agent 开发能力，建立知识传承机制。
- **跨部门协同**：与产品、运营、业务团队协作，推动 Agent 技术在实际业务中落地。

**行动项**：
1. 在 GitHub 上持续贡献 Agent 相关开源项目
2. 沉淀 Agent 开发方法论，整理成内部文档或开源项目
3. 主动承担团队 Agent 方向的技术规划和选型工作
4. 结合飞猪业务场景，探索 Agent 在旅游/出行领域的落地机会

## 横向素养：贯穿始终的两条隐线

### 横向工程素养
Agent 系统对工程素养的要求比传统应用更高：
- **可靠性**：Agent 的自主性意味着错误可能被放大，需要有完善的熔断、降级机制。
- **可维护性**：Agent 的 Prompt 和策略需要版本管理，配置变更需要可追踪。
- **性能**：LLM 调用是瓶颈，需要优化调用次数、使用缓存、并行化非依赖步骤。
- **成本**：Token 消耗是持续成本，需要在效果和成本间做权衡。
- **安全**：Agent 能执行代码、访问外部系统，安全风险成倍放大。

### 横向业务素养
技术最终要服务于业务：
- **产品理解**：理解 Agent 产品的用户场景和价值主张。
- **领域建模**：将业务领域知识结构化，供 Agent 理解和使用。
- **沟通表达**：Agent 系统的复杂性要求清晰的文档和沟通能力。
- **文档写作**：Prompt 工程本质上就是"面向模型的文档写作"。
- **业务落地**：从 POC 到生产环境，推动 Agent 在真实业务中产生价值。

## 个人学习路线与时间安排

基于以上分析，结合日常工作节奏，制定如下学习计划：

### 第一阶段（1-3 个月）：夯实基础
**重点：L2 + L3**
- 周 1-2：系统学习 Transformer 原理，精读 Attention Is All You Need
- 周 3-4：Prompt Engineering 体系化学习，整理 Prompt 模式库
- 周 5-6：Function Calling 实现，动手写 Agent 原型
- 周 7-8：Memory 系统设计，实现短期+长期记忆
- 周 9-10：ReAct / Planning 模式深入
- 周 11-12：安全与约束机制设计

**产出物**：一个具备 Planning + Memory + Tool Use 的 Agent 原型

### 第二阶段（4-6 个月）：走向生产
**重点：L4 + L5**
- 月 4：学习 LangGraph / Mastra，用状态机重构 Agent
- 月 5：集成 Sandbox 安全执行环境
- 月 6：建立可观测性体系（追踪 + 监控 + 日志）
- 月 7：Browser Agent 实战（browser-use / UI-TARS）
- 月 8：RAG 系统集成（向量检索 + Embedding-Free 方案）
- 月 9：多 Agent 协同系统搭建

**产出物**：一个具备完整 Runtime + 环境交互能力的生产级 Agent 系统

### 第三阶段（7-9 个月）：持续进化
**重点：L6 + 横向素养**
- 月 10：建立 Eval 评测体系
- 月 11：实现反馈闭环和 Skill Library
- 月 12：成本优化与性能调优
- 持续：文档写作、方法论沉淀、开源贡献

**产出物**：可自进化的 Agent 系统 + 一套完整的方法论文档

### 第四阶段（10-12 个月）：建立影响力
**重点：L7**
- 主导团队 Agent 技术方向
- 开源项目持续贡献
- 结合业务场景推动 Agent 落地

## 学习资源推荐

### 基础理论
- 《Attention Is All You Need》— Transformer 起源
- 《ReAct: Synergizing Reasoning and Acting in Language Models》— Agent 基础范式
- 《Reflexion: Self-Reflective Agents》— 自我反思 Agent
- 《Let’s Verify Step by Step》— 推理过程验证

### 框架与工具
- **LangChain / LangGraph**：Agent 开发主流框架
- **Mastra**：新兴的 TypeScript-first Agent 框架
- **AgentScope**：阿里巴巴开源的多 Agent 框架，支持多模态、多智能体协同，内置丰富的对话管理和工具编排能力
- **browser-use**：浏览器自动化 Agent
- **MCP（Model Context Protocol）**：工具调用标准协议
- **E2B**：代码执行沙箱
- **Temporal**：Durable Execution 框架

### GitHub 课程与实战仓库

以下仓库按学习阶段分类，从入门到进阶，覆盖理论与实战：

#### 系统入门课程

| 仓库 | 层级对应 | 简介 |
|------|---------|------|
| [datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents) | L2-L5 | Datawhale 出品的系统教程（6.3k+ Star），中文社区最完善的 Agent 入门课程。5 部分 16 章：Agent 基础 → 手写 Agent 框架（ReAct/Plan-and-Solve）→ 低代码平台（Coze/Dify/n8n）→ 进阶主题（Memory/RAG、MCP/A2A/ANP、Agentic-RL）→ 实战项目（智能出行助手、DeepResearch Agent、Cyber Town）。含自研 `hello_agents` 教学框架，从第一性原理构建 Agent |
| [microsoft/ai-agents-for-beginners](https://github.com/microsoft/ai-agents-for-beginners) | L2-L4 | 微软官方 12 课入门课程，含视频、文档和代码。覆盖 Tool Use、Agentic RAG、多 Agent 设计模式、生产部署、MCP/A2A 协议。使用 Azure AI Foundry、Semantic Kernel、AutoGen |
| [GeneArnold/AI-Agent-Engineering-Course](https://github.com/GeneArnold/AI-Agent-Engineering-Course) | L2-L5 | 7 周渐进式课程，专为 Claude Code 辅助学习设计。从单 Agent → Memory Agent → 多工具 Agent → 多 Agent 系统 → 视觉识别 Agent，含 MCP 覆盖 |
| [zkzkGamal/Agentic-AI-Tutorial](https://github.com/zkzkGamal/Agentic-AI-Tutorial) | L2-L4 | 5 章综合教程：LLM Provider 基础 → LangChain 编排 → RAG/Memory 基础设施 → Agent Graph 模式（ReAct、Human-in-the-loop）→ LangGraph + MCP Runtime |
| [NirDiamant/GenAI_Agents](https://github.com/NirDiamant/GenAI_Agents) | L3-L5 | 45+ 个 Jupyter Notebook 项目，从基础到高级，每个项目含详细说明和可运行代码。覆盖 RAG、多 Agent、工具调用等场景 |

#### 实战工作坊（项目驱动）

| 仓库 | 层级对应 | 简介 |
|------|---------|------|
| [iusztinpaul/designing-real-world-ai-agents-workshop](https://github.com/iusztinpaul/designing-real-world-ai-agents-workshop) | L4-L5 | 生产级多 Agent 系统实战：构建 Deep Research Agent + LinkedIn 写作工作流，以 MCP Server 方式提供服务。含代码、PPT、视频和自学路径 |
| [antonacio/intro-to-ai-agents](https://github.com/antonacio/intro-to-ai-agents) | L3-L4 | 手把手构建和编排 Agent 系统：Research Agent、RAG Agent、Web Search Agent、Math Agent，适合快速上手 |
| [coleam00/ai-agents-masterclass](https://github.com/coleam00/ai-agents-masterclass) | L3-L5 | YouTube 视频系列的配套代码，逐步构建实用 Agent，适合跟着视频边做边学 |
| [ed-donner/agents](https://github.com/ed-donner/agents) | L2-L4 | 6 周 Agent 工程课程，从编码到部署的完整流程 |

#### 从零实现（不依赖框架）

| 仓库 | 层级对应 | 简介 |
|------|---------|------|
| [pguso/ai-agents-from-scratch](https://github.com/pguso/ai-agents-from-scratch) | L2-L3 | 用 Node.js + 本地 LLM（node-llama-cpp）从零实现 Agent，理解底层原理。无需 API Key，完全本地运行。配套网站 [agentsfromscratch.com](https://agentsfromscratch.com) |
| [proflead/how-to-build-ai-agents-from-scratch](https://github.com/proflead/how-to-build-ai-agents-from-scratch) | L3 | 基于 Google ADK 和 Gemini 2.5 Flash Lite，构建 Research → Summarizer → Coordinator 多 Agent 系统 |
| [SRafi007/ai-tutorial-crew](https://github.com/SRafi007/ai-tutorial-crew) | L3-L4 | CrewAI + Ollama 本地 LLM 实现多 Agent 协作：自动研究、写作、审阅 Python 教程。零 API 成本 |

#### Awesome Lists（资源导航）

| 仓库 | 用途 |
|------|------|
| [artnitolog/awesome-agent-learning](https://github.com/artnitolog/awesome-agent-learning) | Agent 学习资源汇总，含课程、论文、框架教程、评测基准 |
| [e2b-dev/awesome-ai-agents](https://github.com/e2b-dev/awesome-ai-agents) | AI Agent 框架、库、研究论文的精选列表，区分开源和闭源方案 |
| [punkpeye/awesome-mcp-servers](https://github.com/punkpeye/awesome-mcp-servers) | 按类别整理的 MCP Server 集合（浏览器自动化、代码执行、云服务等） |
| [Shubhamsaboo/awesome-llm-apps](https://github.com/Shubhamsaboo/awesome-llm-apps) | LLM 应用集合，含 Agent、RAG、MCP 等实际项目 |

### 评估与优化
- **LangSmith**：Agent 监控和评估平台
- **Helicone**：LLM 可观测性工具
- **OpenAI Evals**：模型评估框架

### 博主与频道推荐

持续跟踪 Agent 领域的最新动态，除了文档和论文，关注一线从业者的分享同样重要。以下按方向分类推荐：

#### X (Twitter)

| 博主 | 方向 | 简介 |
|------|------|------|
| [@hwchase17](https://x.com/hwchase17) | Agent 框架 | LangChain / LangGraph 创始人，Agent 框架演进的权威声音 |
| [@karpathy](https://x.com/karpathy) | AI 基础 | Andrej Karpathy，AI 教育内容标杆，讲解深入浅出 |
| [@swyx](https://x.com/swyx) | AI 工程化 | Shawn Wang，Latent Space 播客主理人，AI Engineering 趋势追踪 |
| [@goodside](https://x.com/goodside) | Prompt Engineering | Riley Goodside，OpenAI Prompt Engineering 专家 |
| [@jxnlco](https://x.com/jxnlco) | LLM 工程化 | Jason Liu，OpenAI 工程师，LLM 工程实践最佳 |
| [@gregpr07](https://x.com/gregpr07) | Browser Agent | browser-use 联合创始人，第一手更新和实现思路 |
| [@karpulix](https://x.com/karpulix) | 多 Agent 协同 | CrewAI 创始人，多 Agent 协作的实践者 |
| [@rlancemartin](https://x.com/rlancemartin) | RAG | LangChain RAG 负责人，RAG 技术演进第一手资料 |
| [@BeibinLi](https://x.com/BeibinLi) | 多 Agent 框架 | AgentScope（阿里开源）核心作者 |
| [@simonw](https://x.com/simonw) | 工具集成 | Simon Willison，AI 工程化和工具集成领域的多产作者 |

#### YouTube

| 频道 | 方向 | 简介 |
|------|------|------|
| [Andrej Karpathy](https://www.youtube.com/@AndrejKarpathy) | AI 基础 | 从 Transformer 到 GPT 的底层原理讲解，适合夯实 L2 |
| [DeepLearning.AI](https://www.youtube.com/@Deeplearningai) | 系统课程 | Andrew Ng 的短课程系列，Prompt Engineering、RAG、Agent 都有 |
| [Sam Witteveen](https://www.youtube.com/@SamWitteveen) | LangChain 实战 | Google Developer Expert，大量 LangGraph 实战教程 |
| [James Briggs](https://www.youtube.com/@jamesbriggs) | RAG / 向量检索 | Pinecone 开发者，向量数据库和 RAG 最佳实践 |
| [Mastra](https://www.youtube.com/@mastra-ai) | TypeScript Agent | 官方频道，TypeScript-first Agent 框架，技术栈匹配度高 |
| [Latent Space](https://www.youtube.com/@LatentSpacePodcast) | 播客访谈 | swyx 主理，每期访谈一线从业者，了解行业真实进展 |
| [AI Explained](https://www.youtube.com/@aiexplained-official) | 行业动态 | 每日 AI 新闻速递，适合快速了解行业热点 |

**关注策略建议**：日常刷 X 追踪框架更新和行业热点（@hwchase17、@gregpr07、@swyx），周末看 YouTube 深度内容夯实基础（Andrej Karpathy、DeepLearning.AI），项目驱动时跟着实战教程边做边学（Sam Witteveen、James Briggs）。

## 每日2小时学习规划

以上路线图覆盖 12 个月，但大多数在职开发者每天能抽出的时间有限。以下是一份基于**每天 2 小时**、以**工作日为主**的执行方案，核心原则是**小步快跑、项目驱动、周末做整合**。

### 时间块分配

| 时段 | 时长 | 内容 |
|------|------|------|
| **理论学习** | 40 min | 读文档、课程章节或论文，理解概念并记关键笔记 |
| **动手实践** | 60 min | 写代码、跑示例、调 Prompt，必须产出可运行的代码 |
| **复盘整理** | 20 min | 写学习日志、整理代码片段、画思维导图，沉淀为可复用知识 |

建议固定时间段（如早 7:00-9:00 或晚 21:00-23:00），降低启动成本。

### 周末弹性安排

周末不强制加量，但建议完成两件事：
1. **周项目冲刺**（2-3 小时）：把本周零散实践整合成一个完整功能点
2. **补进度/深度阅读**：处理本周遗留，或精读一篇论文

### 第一阶段（1-3 月）：L2 + L3，打地基

**目标**：产出一个具备 Planning + Memory + Tool Use 的 Agent 原型。

| 周次 | 理论学习（40 min） | 动手实践（60 min） | 关键产出 |
|------|-------------------|-------------------|---------|
| 1-2 | 跟读 `hello-agents` Part 1（Agent & LLM 基础），配合 Andrej Karpathy 的 Transformer 视频 | 用 Python 复现简单的 Attention 计算；搭建本地开发环境（uv + Python 3.11） | 环境就绪 + Attention 草稿 |
| 3-4 | 系统学习 Prompt Engineering，整理 Chain-of-Thought、ReAct、Self-Consistency 模板 | 给同一个任务写 3 种不同 Prompt，对比输出效果；记录到个人 Prompt 模式库 | Prompt 模式库（10+ 模板） |
| 5-6 | 精读 `hello-agents` Part 2（手写 Agent 框架），理解 ReAct 循环 | 实现一个最简单的 ReAct Agent：输入 → Thought → Action → Observation → 输出 | ReAct Agent v0.1 |
| 7-8 | 学习 Memory 设计：短期记忆（上下文窗口）+ 长期记忆（向量存储） | 给 Agent 加短期对话历史；接入 ChromaDB/FAISS 做长期记忆检索 | 带记忆的 Agent v0.2 |
| 9-10 | 学习 Tool Use 和 MCP 协议；读 `hello-agents` 的 MCP/A2A 章节 | 给 Agent 注册 2-3 个工具（计算器、搜索、文件读取）；尝试写一个简易 MCP Server | 带工具调用的 Agent v0.3 |
| 11-12 | 学习安全约束：输入过滤、输出审查、权限控制 | 给 Agent 加输入校验、输出敏感词过滤、工具调用权限白名单 | 带安全约束的 Agent v0.4 |

**阶段产出**：一个可运行的 ReAct Agent（Python），支持 Memory + Tool Use + 基础安全。

### 第二阶段（4-6 月）：L4 + L5，走通生产

**目标**：把原型升级为具备 Runtime + 环境交互能力的系统。

| 月次 | 理论学习（40 min） | 动手实践（60 min） | 关键产出 |
|------|-------------------|-------------------|---------|
| 4 | 学习 LangGraph 状态机；读 `microsoft/ai-agents-for-beginners` 的 Design Patterns | 用 LangGraph 重写第一阶段的 Agent，用状态机管理执行流程 | LangGraph 版 Agent |
| 5 | 研究 E2B/Docker Sandbox；学习 Temporal/Durable Execution | 给 Agent 加一个代码执行沙箱；实现任务断点续传 | 带 Sandbox 的 Agent |
| 6 | 学习可观测性：OpenTelemetry、LangSmith/Helicone | 接入 LangSmith，记录每次调用的 Token 消耗、延迟、执行轨迹 | 可监控的 Agent 系统 |
| 7 | 学习 browser-use/UI-TARS；读 `hello-agents` 的实战项目章节 | 用 browser-use 实现一个自动登录 + 抓数据的 Agent | Browser Agent |
| 8 | 学习 RAG：向量检索 + Sirchmunk（Embedding-Free） | 给 Agent 接入文档知识库，支持基于自有文档的问答 | RAG-enabled Agent |
| 9 | 学习多 Agent 协同：AutoGen/CrewAI；读 `hello-agents` 的 Cyber Town | 搭建 2-3 个 Agent 协作：Planner + Worker + Critic | 多 Agent 协作系统 |

**阶段产出**：一个生产级 Agent 系统，具备状态机、沙箱、可观测性、Browser 能力、RAG、多 Agent 协同。

### 第三阶段（7-9 月）：L6 + 横向素养，持续进化

**目标**：让系统能自我改进，沉淀方法论。

| 月次 | 理论学习（40 min） | 动手实践（60 min） | 关键产出 |
|------|-------------------|-------------------|---------|
| 10 | 学习 Eval 体系：成功率、任务完成率、轨迹质量 | 为自己的 Agent 设计 10-20 个测试用例，建立自动化评估 | Eval 数据集 + 评估脚本 |
| 11 | 学习反馈闭环和 Skill Library 设计 | 实现执行轨迹记录；把常用工具组合抽象成可复用的 Skill 模板 | Skill Library（5+ 模板） |
| 12 | 学习成本优化：缓存策略、模型降级、并行化 | 给系统加 Redis 缓存、非关键步骤用便宜模型、并行调用无依赖工具 | 成本优化后的系统 |

**阶段产出**：可自进化的 Agent 系统 + 一套内部方法论文档。

### 第四阶段（10-12 月）：L7，建立影响力

**目标**：从个人学习转向团队价值和开源贡献。

| 行动 | 时间投入 | 具体做法 |
|------|---------|---------|
| 开源贡献 | 每天 60 min | 给 `hello-agents`、`Mastra`、`browser-use` 等项目提 PR（文档改进、Bug 修复、小功能） |
| 方法论沉淀 | 每天 40 min | 把学习笔记整理成博客文章或内部技术分享 |
| 业务探索 | 周末 3-4 小时 | 结合飞猪业务，设计一个 Agent Demo（如智能行程规划、客服辅助） |

### 关键执行原则

1. **先跑起来再完美**：第一周就写出能运行的代码，不要花两周只读理论
2. **一课一代码**：每学完一个概念（如 ReAct、Memory），当天就必须有代码产出
3. **用 Git 管理**：从第一天就建一个 `agent-learning` 仓库，每周末 commit
4. **输出倒逼输入**：每两周写一篇学习笔记（哪怕只是 README），强迫自己消化
5. **卡住先跳过**：2 小时很宝贵，卡超过 30 分钟就记 TODO，先往下走

### 各阶段推荐资源组合

| 阶段 | 主力课程 | 辅助资源 | 备用 |
|------|---------|---------|------|
| 第一阶段 | `hello-agents` Part 1-2 | Andrej Karpathy Transformer 视频 | `microsoft/ai-agents-for-beginners` 前 5 课 |
| 第二阶段 | `microsoft/ai-agents-for-beginners` 后 7 课 + LangGraph 官方文档 | `iusztinpaul/designing-real-world-ai-agents-workshop` | `zkzkGamal/Agentic-AI-Tutorial` |
| 第三阶段 | `hello-agents` Part 3-4 | LangSmith 官方文档 | OpenAI Evals 源码 |
| 第四阶段 | 开源项目源码阅读 | Latent Space 播客 | X 上追踪 @hwchase17、@gregpr07 |

按这个节奏，12 个月后你会有一条清晰的成长轨迹：**从"能写 Agent"到"能设计 Agent 系统"再到"能带团队做 Agent 落地"**。每天 2 小时，关键是**不中断**。

## 写在最后

成为 Agentic System Engineer 不是一蹴而就的，需要从工程基础到 AI 认知，再到系统架构，层层递进。这张图的价值在于它提供了一个完整的地图——你不需要一开始就掌握所有内容，但你需要知道每个能力点的位置和重要性。

作为全栈开发者，最大的优势是工程基础扎实、理解系统全貌。最大的挑战是从"确定性编程"转向"概率性系统"的思维转变。Agent 系统的行为不是完全可控的，需要在不确定性中做工程化。

这条路上没有捷径，但有地图。这张图就是我的地图。接下来，就是一步步走。
