---
title: 工业界 Agent Loop 全景报告：推理时架构范式与横向框架工程实现（2024–2026）
date: 2026-05-20
tags:
  - Agentic
  - Long-Horizon Task
publish: true
---

## TL;DR
- **工业界已就"Agent = LLM 在循环中自主调用工具"这一极简定义达成共识**（Anthropic / Simon Willison）。推理时的工程重心从"提示词"转向"上下文工程"与"循环控制"——即在有限注意力预算内,决定每一步喂给模型什么、何时停止、如何恢复。ReAct 仍是默认范式;Plan-and-Execute / ReWOO / Reflexion / ToT 是针对特定失败模式（token 浪费、刚性、延迟、搜索成本）的变体;Anthropic 的五种 workflow 模式（prompt chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer）是生产级编排蓝图。
- **横向框架已分化为四大抽象阵营**:图/状态机（LangGraph）、会话/消息（OpenAI Agents SDK、AutoGen）、事件驱动（LlamaIndex Workflows、Microsoft Agent Framework）、代码即行动（Smolagents、CodeAct）。LangGraph 凭借 Pregel 超步执行 + checkpoint 持久化 + interrupt 成为有状态长程任务的事实标准;OpenAI Agents SDK 以 handoffs / guardrails / sessions 四原语主打轻量;Anthropic Claude Agent SDK 把驱动 Claude Code 的同款 agent loop + 自动 compaction + subagents 开放为通用 harness。
- **两个标准重塑了工具层**:MCP（2024 年 11 月由 Anthropic 推出,2025 年 3 月被 OpenAI、4 月被 Google 采用,2025 年 12 月 9 日捐给 Linux 基金会下的 AAIF）统一了 agent-工具连接;A2A 统一了 agent 间通信。横切关注点中,**上下文工程（compaction / note-taking / sub-agents）、循环终止（max_iterations + 预算 + 超时三重护栏）、可观测性（OpenTelemetry 成为事实标准,LangSmith / Langfuse 领跑）** 是当前生产实践的三大支柱。

---

## 关键发现

1. **范式收敛到"工具循环"+ 上下文工程**。Anthropic 在 2025 年 9 月《Effective context engineering for AI agents》中明确:随着模型变强,工程不再是写完美提示,而是"在每一步管理整个上下文状态"。该文核心论断是"good context engineering means finding the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome"(好的上下文工程就是找到最小的高信号 token 集以最大化期望结果的概率)。研究发现"context rot"——随 token 增多模型召回精度下降,根源在于 transformer 的 n² 注意力关系使长上下文下注意力被稀释。

2. **单 agent 范式各有明确的失败模式权衡**。ReAct 易陷入无限循环/目标漂移且 token 消耗随轨迹增长;Plan-and-Execute 用更便宜的模型执行、可在执行前审查;ReWOO 据其原论文（Xu et al., arXiv:2305.18323, 2023）在 HotpotQA 上实现约 **5× token 效率**并提升约 4% 准确率;Reflexion 增加延迟但能自我纠错。

3. **多 agent 编排呈现明确拓扑学**:supervisor（中心路由）、swarm（去中心化直接 handoff）、network（任意互联）、hierarchical（supervisor 嵌套）。据 Anthropic 官方工程博客《How we built our multi-agent research system》,其多 agent 研究系统（Claude Opus 4 作 lead、Claude Sonnet 4 作 subagent）在内部研究评测上比单 agent Opus 4 **高 90.2%**;在 BrowseComp 评测上三因素解释了 95% 的性能方差,其中 **token 用量本身解释 80%**、工具调用数约 10%、模型选择约 5%;代价是多 agent 系统消耗约为普通 chat 的 **15 倍 token**。

4. **MCP 在一年内成为行业标准**。据 MCP 官方博客 2025 年 12 月 9 日《MCP joins the Agentic AI Foundation》,MCP 已有"**超过 9700 万次月度 SDK 下载、1 万个活跃 server**,并在 ChatGPT、Claude、Cursor、Gemini、Microsoft Copilot、Visual Studio Code 等主要平台获得一流客户端支持"。

5. **代码即行动（CodeAct）在复杂任务上比 JSON 工具调用成功率高约 20%、步数少约 30%**。据 CodeAct 论文（Wang et al., arXiv:2402.01030, ICML 2024）摘要,在 API-Bank 与新建的 M3ToolEval 基准上对 17 个 LLM 测试,"CodeAct outperforms widely used alternatives (up to 20% higher success rate)";因为代码天然支持控制流、数据流与工具组合。Anthropic 已将其产品化为 Programmatic Tool Calling。

6. **循环终止是生产可靠性的核心**。业界共识是三重护栏:max_iterations（LangChain 默认需手动设、典型 15–25）、token/成本预算、超时;并辅以显式 task_complete 信号与循环检测（去重）。

---

## 详解一:Agent Loop 架构范式与控制循环模式

### ReAct（Reasoning + Acting）
**核心循环**:Thought → Action → Observation 交替,每步一次 LLM 调用,直到模型不再产生工具调用、输出 Final Answer。**解决的问题**:让推理与工具使用交织,适应不可预测环境。**终止条件**:模型输出最终答案,或达到 max_iterations。**工具调用**:每轮选一个工具,观测结果回填到 scratchpad。**上下文/记忆**:整段 thought/action/observation 历史累积进上下文(易膨胀)。**错误恢复**:下一轮 thought 中自然纠偏——但实践中工具失败时 ReAct 容易陷入"无限动作循环或重复"(toolA 失败转 toolB、再循环回 toolA 直到撞 token 上限)。**优缺点**:适应性强、是行业标准;但长链上 token 成本高、易无限循环和目标漂移。适用于探索性数据分析、实时排障。

### Plan-and-Execute / Plan-and-Solve
**核心循环**:Planner 先生成完整多步计划（To-Do 列表）,Executor（可用更小更快模型）逐步执行,全部完成后 Planner 可选 replan。**解决的问题**:长程任务中单步 ReAct 容易丢失全局目标。**优点**:LLM 调用更少、成本更低、执行前可审查计划;**缺点**:遇到意外结果需显式 replan,不如 ReAct 灵活。有第三方博客称其在复杂长程工作流中准确率显著更高(该数字来自单一第三方来源,未经一手验证,应谨慎对待)。

### ReWOO（Reasoning WithOut Observation）
**核心循环**:三阶段解耦——Planner 一次性推理生成带占位符（如 #E1）的完整计划;Worker 并行执行所有工具调用;Solver 汇总生成最终答案。**关键机制**:用变量替换（#E）使后续步骤无需看到完整历史,LLM 上下文最小化。**优点**:据原论文（arXiv:2305.18323）在 HotpotQA 上实现约 5× token 效率并提升约 4% 准确率,延迟低。**适用**:可预测的多跳检索。**关于鲁棒性的来源分歧**:LangGraph 教程等工程实践普遍认为 ReWOO 因执行期无观测、工具返回异常时不如 ReAct,常需回退到 ReAct 恢复;但 ReWOO 原论文附录称其"在工具失败场景下表现稳健"。这一分歧本身值得注意——工程上仍建议为 ReWOO 加失败检测与 ReAct 回退逻辑。

### Reflexion / Self-Reflection
**核心循环**:在 ReAct/CoT 的 Actor 之上,加 Evaluator（打分）和 Self-Reflection（生成语言反馈）,将反馈存入记忆,在下一次尝试中作为上下文。**解决的问题**:让 agent 从失败轨迹中"口头强化学习"。Reflexion（Shinn et al., 2023）建立在 Self-Refine 之上;LATS 进一步把 MCTS 引入 Reflexion,代价是更高 token 与成本。**优缺点**:能自我纠错、提升准确率;但增加延迟,且研究发现 Reflexion/LATS 会产生冗余自反思。

### Tree of Thoughts（ToT）/ Graph of Thoughts
**核心循环**:把推理分解为离散"thought"单元,通过采样/顺序提议生成多个候选,用自评启发式（打分或投票）评估,再用 BFS/DFS 等搜索算法做前瞻、回溯、剪枝。**解决的问题**:用"System 2"式深思替代 CoT 的单一线性生成。**优缺点**:在 Game of 24 等需搜索的任务上大幅提升;但树搜索计算开销大,且依赖模型自评,对提示设计敏感。

### Anthropic 五种 workflow / agent 模式
Anthropic《Building Effective Agents》区分了"workflow"（LLM 与工具通过预定义代码路径编排）与"agent"（LLM 动态主导自身流程）。五种 workflow 模式:
- **Prompt chaining**:顺序分解为多步,每步基于上一步。
- **Routing**:对输入分类后路由到专门模型（如客服分流）,避免单提示过载。
- **Parallelization**:sectioning（分工）或 voting（多次投票提升置信度）,用于速度关键场景。
- **Orchestrator-Workers**:orchestrator 动态分解任务并委派,适合复杂度不可预测的任务（Anthropic 编码 agent 用此处理跨文件 GitHub issue）。
- **Evaluator-Optimizer**:一个 LLM 生成、另一个评估并迭代改进。

Anthropic 核心建议:**从直接用 LLM API 开始,只在确实需要时引入框架**,因为框架的抽象层会掩盖底层提示与响应、增加调试难度。

### CodeAct / 代码即行动
不再每次发一个 JSON 工具调用,而是让 agent 生成可执行 Python "code action",在沙箱中调用多个工具、处理结果、应用控制流。据 CodeAct 论文（Wang et al., arXiv:2402.01030, ICML 2024）,复杂任务成功率比 JSON/text 高约 20%、步数少约 30%。Smolagents 以此为默认 agent 类型(约 1000 行实现)。代价:需要沙箱执行环境、容器生命周期管理、更大的安全面(prompt injection 触及代码执行器是不同量级的风险)。HuggingFace 进一步发现"结构化生成的 CodeAgent"(JSON 包裹 thought+code)能再提升性能。

---

## 详解二:工业界主流横向 Agent 框架工程实现

### LangGraph / LangChain
**核心抽象**:基于图的状态机。节点（函数）+ 边 + 共享 State（TypedDict）。**执行引擎**:受 Google Pregel 启发的"super-step"（超步）/ Bulk Synchronous Parallel 模型。官方文档:"程序以离散的'超步'推进。并行运行的节点属于同一超步,顺序运行的节点属于不同超步。"每个超步分三阶段:**Plan**（确定本步执行哪些 actor）、**Execution**（并行执行,通道更新对 actor 不可见）、**Update**（用本步写入更新通道）。当所有节点 inactive 且无消息传递时终止。多出边的节点会在下一超步并行执行。recursion limit 限制最大超步数,1.0.6 起默认 1000。

**循环控制**:`create_agent` / `create_react_agent` 构建 agent+tools 两节点循环:agent 节点调模型,若 AIMessage 含 tool_calls 则路由到 tools 节点执行并回填 ToolMessage,再调模型,直到无 tool_calls。路由由 `tools_condition` 实现——官方定义为"若最后 AIMessage 含工具调用则路由到 'tools',否则路由到 '__end__'"。

**状态合并**:reducer 机制。每个 State key 有独立 reducer,默认覆盖;用 `Annotated[list, operator.add]` 可累加。`add_messages` 处理消息 ID 去重/更新（支持 human-in-the-loop 手动改消息,而非简单追加）。并行分支写同一 key 时**必须**定义 reducer,否则 last-write-wins 冲突。底层由 Pregel 的 `BinaryOperatorAggregate` 通道支撑。

**持久化/可恢复**:checkpointer 在每个超步读写 checkpoint;MemorySaver（开发）、SqliteSaver（单机）、PostgresSaver（多实例）。支持 durable execution 三模式(约 2025 年 7 月引入,取代旧的 `checkpoint_during` 布尔):`durability="exit"`（仅退出时持久化,性能最好但崩溃不可恢复）、`"async"`（异步持久化,性能与持久兼顾,崩溃时有小概率丢 checkpoint）、`"sync"`（同步持久化,最高持久性但有性能开销）。恢复时不从原代码行继续,而是重放至断点,故节点须确定性。

**human-in-the-loop**:`interrupt()` 函数暂停图、把输入存入持久层、标记线程 interrupted,之后用 `Command(resume=...)` 恢复;`interrupt_before` / `interrupt_after` 设置暂停点。支持 time travel——回到历史 checkpoint、注入修正、fork 执行路径。`HumanInTheLoopMiddleware` 提供 approve/edit/reject/respond 四种决策。

**多 agent**:supervisor（中心路由）、swarm（handoff 工具 + `Command(goto=..., graph=Command.PARENT)`）、hierarchical（supervisor 嵌套）。LangGraph 现推荐直接用 tools 实现 supervisor 而非旧的 langgraph-supervisor 库。**可观测性**:LangSmith 深度集成。**生产用户**:据第三方称包括 LinkedIn、Uber 等 400+ 公司(该数字为第三方来源)。

### OpenAI Agents SDK（前身 Swarm）
**四原语**:Agents（指令+工具的 LLM 实体）、Tools（函数工具,自动 schema 生成 + Pydantic 校验）、Handoffs（agent 间转移）、Guardrails（输入/输出安全检查）。2025 年 3 月发布,是实验性 Swarm 的生产级继任者,基于 Responses API。**执行循环**:Runner 自动处理 call-tool-respond 循环,包括多步工具链。**Handoffs**:实现为工具（如 `transfer_to_refund_agent`）,可带 input_filter 过滤历史。**Guardrails**:tripwire 触发即抛异常停止;可并行（默认,低延迟但可能已消耗 token）或阻塞（先校验再执行,省成本）。**Sessions**:agent loop 内维护工作上下文的持久记忆层。**Tracing**:每次 Runner.run 自动产生 trace（模型调用、工具调用、handoff、guardrail、计时）,在 platform.openai.com 查看。**MCP**:支持 HostedMCPTool（OpenAI 基础设施侧）与本地 stdio/SSE/StreamableHTTP transport。**模型支持**:默认 OpenAI,通过 LiteLLM/AnyLLM 支持 100+ LLM。据某来源,2026 年 4 月更新加入了原生 sandbox 执行(该具体日期/特性属较新表述,需谨慎)。

### Anthropic Claude Agent SDK（原 Claude Code SDK）
2025 年 9 月从"Claude Code SDK"改名,反映其已通用化。**核心循环**:gather context → take action → verify work → repeat。**关键卖点**:SDK 替你写 agent loop——不用检查 stop_reason、执行工具、回填结果。**内置工具**:14+ 个（Read/Write/Bash/Glob/Grep/WebSearch/WebFetch 等）,按名引用而非 JSON schema。**上下文管理**:自动 compaction——接近上下文上限时总结旧消息、用摘要重启(保留架构决策、未解决 bug、实现细节 + 最近 5 个文件）;tool result clearing（清除可重取的旧工具结果,已作为 Claude Developer Platform 特性发布）;memory 工具（文件式持久笔记）。**Subagents**:每个有独立上下文窗口、自定义系统提示、受限工具、独立权限;中间工具调用留在 subagent 内,只把最终消息（常 1000–2000 token）返回父 agent;可并行。fork 模式让 subagent 继承完整会话并复用 prompt cache。**agentic search**:用 grep/find/glob 按需搜索,优先于语义检索（RAG）。**持久记忆**:CLAUDE.md / .claude/CLAUDE.md 跨会话。**长程任务**:initializer agent + coding agent,每会话留清晰 artifacts 给下一会话（《Effective harnesses for long-running agents》《Scaling Managed Agents》）。

### AutoGen / AG2 / Microsoft Agent Framework
**AutoGen**:微软多 agent 会话框架。核心是 ConversableAgent（可对话）、AssistantAgent、UserProxyAgent。通过 `initiate_chat` 启动多 agent 对话;`human_input_mode`（ALWAYS/NEVER）控制人类参与;`max_consecutive_auto_reply` 限制循环。**v0.4（2025 年 1 月）**:完全重写为异步、事件驱动的 actor 模型,分层架构（Core/AgentChat/Extensions）,支持 OpenTelemetry。**Magentic-One**:通用多 agent 系统,Orchestrator 实现 outer loop（管理含事实/猜测/计划的 task ledger）和 inner loop。**AG2**:社区主导的 fork（从 AutoGen 0.2.34 改名）,保持旧 GroupChat 风格。**Microsoft Agent Framework（MAF,2026 年 4 月 1.0 GA）**:融合 Semantic Kernel（线程式状态管理、企业稳定性）与 AutoGen（多 agent 编排）。提供 AI Agents、Agent Threads（Redis/Cosmos DB 持久化）、Workflows（图式编排、checkpointing/hydration）。编排模式:sequential、concurrent、handoff、group chat、Magentic-One,均支持 streaming、checkpointing、HITL、pause/resume。可观测性:微软与 Cisco Outshift 合作向 OpenTelemetry 贡献了 agent 追踪标准,同款 instrumentation 可用于 MAF、LangChain、LangGraph、OpenAI Agents SDK。

### LlamaIndex Workflows
**核心抽象**:事件驱动。Steps（`@step` 装饰的函数）+ Events（typed Pydantic 类）。step 声明监听哪种 event、emit 哪种 event;特殊的 StartEvent/StopEvent 管理生命周期。**为何事件驱动**:克服 DAG 无法处理循环和条件逻辑（自我纠错/重试）的限制——是 Digraph 而非 DAG,一个 validation 步骤可 emit ValidationErrorEvent 触发 extraction 重跑形成自纠错循环。**特性**:全异步（asyncio）、可 start/pause/resume（stateful）、支持嵌套/并行。**AgentWorkflow**:更高层的多 agent 编排,init_run 初始化 Context 和 ChatMemory,handoff 作为工具集成进 run_agent_step。Workflows 1.0 已成独立包（`pip install llama-index-workflows`）,月下载据官方称 2500 万+。

### CrewAI
**核心抽象**:角色化（role-based）。Agents 有 role/goal/backstory。**双模式**:**Crews**（高自治,Process.sequential/hierarchical）和 **Flows**（确定性、可审计,`@start`/`@listen` 装饰器的事件驱动编排）。hierarchical process 中 manager 按能力分配任务、审查输出。独立于 LangChain 的轻量框架。适合角色明确、装配线式工作流;不适合自由的"全员对话"。

### Google ADK 与 A2A
**ADK（Agent Development Kit）**:编排原语 SequentialAgent、ParallelAgent、LoopAgent（像 while 循环,重复运行直到条件满足或达 max 迭代）、LlmAgent（动态路由）。Local sub-agents（进程内,内存通信）vs Remote agents（A2A,跨网络）。Java 1.0 引入 event compaction（总结+保留策略管理上下文）、ToolConfirmation（HITL）、App 容器管理全局 Plugins（日志/guardrails）。新 Interactions API 提供 inner loop（API 内）/outer loop（agent 代码）控制、用 previous_interaction_id 卸载历史管理。**A2A 协议**:50+ 技术伙伴构建的开放标准,补充 MCP。每个 agent 暴露 AgentCard（描述能力的 JSON,在 well-known 端点）,可被发现和调用,无需了解实现。`to_a2a()` 把 ADK agent 转为 A2A server（自动生成 agent card）。

### Pydantic AI、Smolagents、Strands Agents
- **Pydantic AI**:类型安全优先,由 Pydantic 团队打造（Pydantic 是 OpenAI SDK、Google ADK、Anthropic SDK、LangChain 等的校验层）。每个输入/工具签名/输出都用 Python 类型校验,输出校验失败会重新提示 agent 重试。依赖注入（RunContext）提供服务/数据。集成 Logfire（OpenTelemetry）。支持 durable execution、HITL 工具批准、graph、MCP/A2A,可纯 YAML/JSON 定义 agent。
- **Smolagents**（HuggingFace）:极简、代码优先。CodeAgent 是默认类型,agent 写并执行 Python 代码而非选预定义工具,遵循 ReAct 的 MultiStepAgent 抽象。约 1000 行实现。代码执行用 E2BSandbox 或 LocalPythonInterpreter。弱点:对小模型（<7B）代码质量下降明显;auth/限流/日志需自建。
- **Strands Agents**（AWS）:模型驱动哲学——给模型工具和目标,让 LLM 决定执行路径,不定义显式工作流。多 agent 模式:Graph、Swarm、Workflow,支持 A2A。模型无关（Bedrock/Anthropic/OpenAI/Gemini/Ollama）。原生 OpenTelemetry,部署到 Lambda/Fargate/EKS/Bedrock AgentCore。AWS 多团队（Amazon Q Developer、AWS Glue、VPC Reachability Analyzer 等)生产使用。据某来源 2025 年 5 月开源后 SDK 下载量超千万次（第三方数字）。

---

## 详解三:横切关注点

### 上下文工程（Context Engineering）
Anthropic 定义为"prompt engineering 的自然演进"——在 LLM 推理时管理整个上下文状态（系统指令、工具、MCP、外部数据、消息历史）。核心约束:**context rot**（随 token 增多召回精度下降）、有限"注意力预算"（源于 transformer n² 注意力）。三大长程技术:
- **Compaction**:接近上限时总结、用摘要重启新上下文窗口（Claude Code 保留架构决策、未解决 bug、实现细节 + 最近 5 个文件）。最轻量形式是 tool result clearing。
- **Structured note-taking（agentic memory）**:agent 把笔记写到上下文外的持久存储（如 NOTES.md、to-do list）,后续拉回。Anthropic 多 agent 研究系统的 LeadResearcher 会在上下文超 20 万 token 被截断前把计划存入 Memory。Claude 玩 Pokémon 跨数千步维持记忆。
- **Sub-agent 架构**:专门 subagent 用干净上下文处理聚焦任务,只返回浓缩摘要（1000–2000 token）。

还有 **just-in-time 检索**:维护轻量标识符（文件路径、查询、链接）,运行时按需加载,而非预先全量检索（Claude Code 的 grep/glob + CLAUDE.md 混合策略）。

### 工具调用机制与 MCP
**Function calling**:模型生成工具名+参数,开发者解析执行回填。**MCP（Model Context Protocol）**:2024 年 11 月 Anthropic 推出（创造者 David Soria Parra 与 Justin Spahr-Summers）,基于 JSON-RPC 2.0、借鉴 LSP。三原语:Tools（可调用函数）、Resources（可读数据,类似 GET）、Prompts（可复用模板）。client-server-host 架构。采用时间线:2025 年 3 月 OpenAI 采用（Sam Altman 宣布,并计划 2026 年中弃用 Assistants API）,2025 年 4 月 Google DeepMind 跟进,2025 年 12 月 9 日捐给 Linux 基金会 AAIF（Anthropic/Block/OpenAI 共同发起,Google/Microsoft/AWS/Cloudflare/Bloomberg 支持）。据官方,已有超 9700 万次月度 SDK 下载、1 万个活跃 server。**安全问题**:据 Wikipedia "Model Context Protocol" 条目,2025 年 4 月安全研究者分析指出 MCP 存在多项未决安全问题,"including prompt injection, tool permissions that allow for combining tools to exfiltrate data, and lookalike tools that can silently replace trusted ones"(prompt injection、可组合工具外泄数据的权限、可静默替换可信工具的仿冒工具)。**并行工具调用**与 **programmatic/code 工具调用**（把中间结果留在执行环境、只返回最终结果）是减少上下文占用的关键。

### 循环终止与控制
业界共识三重护栏:**max_iterations**（LangChain AgentExecutor 需手动设、典型 15–25,超 20 步通常应分解任务）、**预算**（token/成本上限）、**超时**（任务级+API 级）。辅以:显式停止信号（task_complete 工具）、循环检测（按 tool+args_hash 去重）、kill switch。常见失败:目标模糊、无终止条件、工具误用（重复调同一工具）、重试风暴。LangGraph 的 should_continue 条件边若无迭代上限会无限循环（有案例对畸形文档跑了 80 次）。

### 错误处理与恢复
重试（带退避）、回退（ReWOO 失败回退 ReAct）、自我纠错（Reflexion）。LangGraph checkpoint 让 502 等故障后能从断点恢复而非重启（避免烧掉已花的 API 费用）。结构化生成（JSON 格式的 CodeAgent）减少解析错误——HuggingFace 分析 15724 条轨迹发现首步解析错误使成功率降 21.3%（51.3%→42.3%）、平均步数从 3.18 增至 4.63。

### 可观测性与评估
**OpenTelemetry 成为事实标准**（Pydantic AI、Smolagents、Strands 均原生 emit）。主要平台:
- **LangSmith**:与 LangChain/LangGraph 深度集成,LangGraph Studio 可视化调试 + checkpoint time-travel;闭源,自托管仅企业版。据某基准测试开销最低。
- **Langfuse**:开源（MIT）,OpenTelemetry-native,ClickHouse 后端,prompt 管理 + eval + 追踪。据 ClickHouse 官方博客,**2026 年 1 月 16 日被 ClickHouse 收购**(同日 ClickHouse 完成 4 亿美元 D 轮、估值约 150 亿美元);Langfuse 拥有约 20470 GitHub stars、月 SDK 安装 2600 万+、服务 Fortune 50 中 19 家,MIT 许可不变。
- **AgentOps**:专为 agent 监控,session replay、多 agent 追踪;支持 400+ 框架。
- 其他:Arize Phoenix（OTel-native 开源）、Laminar（长程 agent 专用,Apache 2.0）、Braintrust（eval 优先）、MLflow（Apache 2.0,Linux 基金会）。

### 长程任务的 agent loop 挑战
长轨迹超出上下文窗口需 compaction/note-taking/multi-agent;状态管理需 checkpoint/durable execution;多 agent 系统 token 消耗约为单 chat 的 15 倍,且失败时可能再放大 10 倍（subagent 递归 spawn 或工具返回超大结果）,而 Anthropic 公布的架构无 circuit breaker。Anthropic 明确:**需要所有 agent 共享同一上下文或 agent 间多依赖的领域（如编码、调试）不适合当前的多 agent 系统**。

---

## 框架横向对比

| 框架 | 核心抽象 | 循环机制 | 持久化/可恢复 | HITL | 多 agent | 可观测性 | 成熟度 |
|---|---|---|---|---|---|---|---|
| LangGraph | 图/状态机 | Pregel 超步 + 条件边 | checkpointer(三 durability 模式) | interrupt + time travel | supervisor/swarm/hierarchical | LangSmith 深度 | 生产标准 |
| OpenAI Agents SDK | 会话/4 原语 | Runner 自动工具循环 | Sessions | 内置机制 | handoffs | 自动 tracing | 生产(OpenAI 优先) |
| Claude Agent SDK | gather-act-verify 循环 | SDK 托管 agent loop | CLAUDE.md + sessions | 权限模式/hooks | subagents(隔离上下文) | 内置 | 生产(Claude 优先) |
| AutoGen v0.4 / MAF | 会话/actor 事件驱动 | 异步消息 | MAF checkpointing | 支持 | group chat/Magentic-One | OpenTelemetry | MAF 1.0 GA |
| LlamaIndex Workflows | 事件驱动 step | event emit/listen | stateful pause/resume | 支持 | AgentWorkflow | OTel/Langfuse | 1.0 稳定 |
| CrewAI | 角色化 | Crews/Flows | Flows 可审计 | 支持 | hierarchical/sequential | 集成 | 生产 |
| Google ADK | 编排原语 | Sequential/Parallel/Loop | event compaction | ToolConfirmation | A2A + sub-agents | Foundry/OTel | GCP 生态 |
| Pydantic AI | 类型安全 | 类型校验工具循环 | durable execution | 工具批准 | graph/A2A | Logfire/OTel | 快速成熟 |
| Smolagents | 代码即行动 | CodeAgent ReAct 循环 | 自建 | 基础 | 基础 | OTel | 实验/轻量 |
| Strands(AWS) | 模型驱动 | LLM 决定路径 | AgentCore | 支持 | Graph/Swarm/Workflow | OTel 原生 | AWS 生产 |

---

## 设计权衡分析

1. **自治 vs 可控**:Anthropic 建议"先用 LLM API,确需才上框架";workflow（预定义路径）比 agent（动态）更可预测可靠。模型越强,可给更多自治。
2. **抽象层的代价**:框架抽象掩盖底层提示/响应、增加调试难度,是客户错误的常见来源。图式（LangGraph）显式可审计但复杂;模型驱动（Strands）代码少但依赖模型推理。
3. **token/延迟权衡**:ReAct（浪费 token）vs ReWOO（执行期无观测的刚性）vs Reflexion（延迟）vs Plan-and-Execute（replan 复杂度）;多 agent 高 90% 性能但 15 倍 token、token 用量解释 80% 方差。
4. **上下文窗口非银弹**:Anthropic 指出更大窗口仍受 context pollution 困扰,compaction/note-taking/sub-agent 比等待大窗口更实际。
5. **代码即行动 vs JSON**:CodeAct 成功率高约 20% 但需沙箱、安全面更大;仅当工作流真正程序化（迭代/分支/聚合）时才值得。

---

## 建议

**阶段一(原型)**:用最简方案。单 agent ReAct 直接调 LLM API 或用轻量框架（Pydantic AI / Smolagents / OpenAI Agents SDK）。先把一个 agent loop 端到端跑通,设好 max_iterations + 超时。**触发升级的基准**:当提示膨胀到模型忘规则、工具选错、或需要跨会话状态时。

**阶段二(有状态生产)**:需要持久化/审批门/并行 fan-out 时上 LangGraph。state schema 设计是最关键决策——用 Annotated reducer 防止并行分支覆盖;MemorySaver（开发）→ Postgres（多实例）。**基准**:当 coordinator 上下文在工作流完成前填满,或序列化路由延迟超 SLA,从扁平 supervisor 升级到 hierarchical。

**阶段三(规模化)**:接入 OpenTelemetry 可观测性（LangSmith 若在 LangChain 生态,否则 Langfuse/MLflow 避免锁定）。从 day 1 实现 tracing + 预算 + 循环检测。引入 compaction/note-taking/sub-agents 应对长程。MCP 接入外部工具,A2A 跨框架协作。

**何时用多 agent**:仅当任务可自然并行、价值高到能吸收 15 倍 token 成本（法律尽调、竞品情报、文献综述）。**何时不用**:紧耦合任务（编码、调试)。务必加 circuit breaker / per-run 上限防止 subagent 递归爆炸。

**改变建议的阈值**:若 token 成本/性能比恶化、单 agent 上下文在完成前填满、或失败模式集中在多步因果链——升级架构或拆分子任务,而非靠提示工程修结构性协调失败。

---

## 注意事项

- **数字来源差异**:Anthropic 多 agent"高 90.2%"、"15 倍 token"、"token 解释 80% 方差"(BrowseComp 上三因素解释 95% 方差)来自 Anthropic 官方工程博客;CodeAct"高约 20%"出自 arXiv:2402.01030;ReWOO"5× token 效率/+4% 准确率"出自 arXiv:2305.18323;MCP"9700 万月下载/1 万 server"出自 MCP 官方博客;Langfuse 被 ClickHouse 收购出自 ClickHouse 官方博客。但 Plan-and-Execute"准确率高 92%"等来自单一第三方博客,未经一手验证,应谨慎引用;LangGraph"400+ 公司/LinkedIn/Uber"、Strands"千万下载"、LlamaIndex"2500 万月下载"为第三方或厂商陈述。
- **来源分歧**:ReWOO 的工具失败鲁棒性,工程实践(LangGraph 教程)与原论文附录说法相反,本报告已并列呈现。
- **API 快速演进**:Claude Agent SDK 有 V2 session-based 接口与 0.2.x generator 模式并存;OpenAI Agents SDK 的 sandbox 特性、MAF 版本号、各框架 GA 日期都在快速变化,部分搜索结果含 2026 年中后期的推测性或未来时表述（如尚未发布的模型名）,不应当作既成事实。
- **MCP 安全**:生产部署需谨慎实现权限/认证/沙箱,2025 年随 Cloudflare、Auth0、New Relic 等加入,治理与可观测性工具链才逐步成熟。
- **框架边界模糊**:许多框架在快速互相借鉴（都加事件驱动、checkpoint、HITL、MCP、A2A、OTel）,"阵营"划分是当前快照而非永久分类。
- 本报告聚焦推理时（inference-time）架构与横向框架工程实现,与训练/RL 侧的 Agentic RL 报告互补;具体生产经验（如 Anthropic 内部评估）多来自厂商自述,独立复现证据有限。