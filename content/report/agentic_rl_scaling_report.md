---
title: Agentic RL 的三重 Scaling 与工程落地调研报告
date: 2026-05-21
tags:
  - Agentic RL
  - RL Scaling
  - Engineering
publish: true
---

## Environment / Task / Agentic-RL Scaling 全景(2025–2026)

> 范围说明:本报告以"环境—任务—算法"三个规模化维度为主线,聚焦 2025 年至今(含 2026 年初最新预印本)的代表性工作,并在第四部分汇总训练系统与基座模型的工程落地路径。报告以可执行、可验证、可规模化为核心评价标准,所有结论尽量绑定到具体论文/系统与可复核的数字。

---

## TL;DR

- **三重 Scaling 已成为长程 Agent 能力增长的主轴,且彼此构成飞轮**:环境(Environment)决定能力的广度上限,任务(Task)决定分布的密度与难度梯度,算法(Agentic RL)决定从交互经验中提取信号的效率。2025 年最显著的范式转变,是从"手工搭建少量环境/人工标注任务"转向"全自动合成大规模可执行、可验证的环境与任务,再用可扩展 RL 闭环优化"。代表性里程碑是 DeepSeek-V3.2 用 **1,800+ 合成环境、85k+ 复杂指令**做大规模 agentic 后训练,以及阿里 Tongyi DeepResearch 用 **AgentFounder 数据飞轮 + Agentic CPT** 在无人工标注下追平 OpenAI DeepResearch。

- **环境合成已从代码领域(SWE)外溢到通用工具调用领域**。代码侧:SWE-smith 把"给定任意 Python 仓库→自动构建执行环境→破坏测试生成 100–1000 个实例"工业化,产出 5 万实例/128 仓库;通用工具侧:AgentScaler 把"函数调用"抽象为"对结构化数据库的读写操作",从 3 万+ 真实 API 合成异构模拟环境;EnvScaler、EnvFactory、Agent World Model 等进一步把环境数量推到 10² – 10³ 量级,并普遍采用 **MCP 统一接口 + 基于环境状态的规则化验证**。

- **任务合成的关键不是"多",而是"可验证 + 难度可控 + 隐式意图"**。主流技术包括:程序化生成(破坏测试、commit 回译)、自探索式发现(先用好奇心探索环境再反推任务,如 AgentEvolver / AutoPlay)、轨迹回收(把废弃/失败轨迹重组为 step-level 决策数据,如 AgentFounder 的高阶动作合成)、以及自博弈/自演化课程(Absolute Zero、R-Zero、WebRL)。过滤侧普遍依赖执行结果或 verifier 做拒绝采样。

- **算法侧的主战场是信用分配的粒度下沉与稳定性**。一篇 2026 年综述梳理了 2024–2026 的 **47 种信用分配方法**,按"粒度(token/segment/step/turn/multi-agent)×方法论(MC/TD/模型/博弈/信息论)"二维归类。实践上,turn-level(Turn-PPO、GiGPO 的层级分组、Tree-GRPO 的树状分支)正成为多轮 agent 的"自然原子单位";奖励侧从 outcome → process(AgentPRM)→ agentic verifier(AgentV-RL),自反思/自进化(MR-Search、Agentic Self-Learning)开始把"学会探索"内化进策略。

- **工程瓶颈已从算法转移到系统**:长程 rollout 的长尾分布导致同步训练大量 GPU 空转,**全异步解耦 RL**(GLM-5、AReaL、Slime、polar)成为标配;verl / SkyRL / ROLL / rLLM / Agent Lightning 等框架在 rollout 效率、环境并行、训练-agent 解耦上各有侧重。最大的开放风险仍是 **reward hacking 诱发的涌现性错位**(Anthropic 生产级实验),"inoculation prompting"等可扩展监督手段是当前性价比最高的缓解措施之一。

---

## 一、Environment Scaling:大规模可执行可验证环境的自动合成

### 1.1 为什么"环境"是长程 Agent 能力的第一性瓶颈

长程 agent 的能力上界由它"见过多少种可交互、可获得真实反馈的世界"决定。函数调用/工具使用能力的广度,与训练环境的多样性高度耦合(AgentScaler 的核心论点)。而真实环境有三大不可规模化的痛点:(1)依赖真实 API,成本高、不稳定、不可重置;(2)用 LLM 模拟器易产生幻觉、状态不一致;(3)人工搭建可执行环境(如经典 SWE-bench 路线)需要大量人工(SWE-Gym 公布的镜像构建约耗 200 人工标注小时与约 1 万 CPU 核小时)。因此 2025 年的核心命题变成:**如何全自动、低成本地批量生产"可执行 + 可验证 + 可并行重置"的沙盒**。

一个有用的工程分类(参考社区对 RL 环境的 taxonomy 整理):环境(Environment)= 工具集 + 状态转移 + 验证器;而 harness(脚手架)只决定"怎么交互",不增加"知道什么"。环境合成方法可沿两个轴划分:

- **构建方式**:确定性脚本(deterministic,如 R2E 的脚本化安装/测试)vs agentic 搭建(LLM agent 读文档→在沙盒里试错→看日志诊断→迭代修复,如 SetupAgent、SWE-bench-Live 的 RepoLaunch);模板约束的容器合成(SWE-Bench++ 约束 Dockerfile 结构、仅填充仓库相关槽位)介于两者之间。
- **知识来源**:基于真实资源(真实仓库、真实 API、真实文档)vs 纯程序化合成(从 schema/代码骨架生成)。

### 1.2 代码领域:从 SWE-Gym 到工业化合成管线

代码是环境合成最成熟的领域,因为"单元测试通过率"提供了近乎无噪声的可验证奖励。演进脉络:

- **SWE-Gym**(arXiv:2412.21139, ICML 2025):首个开放 SWE agent 训练环境,2,438 个真实 Python 任务、11 个仓库,每实例含可执行运行时 + 单元测试。仅约 491 条拒绝采样的成功轨迹微调 Qwen-2.5-Coder-32B,即把 SWE-bench Verified 从 7.0% 提到 20.6%;叠加推理时 verifier 后达 32.0%。证明了"少量高质量可执行环境 + 可验证信号"的高数据效率。
- **SWE-smith**(arXiv:2504.21798, NeurIPS'25):把环境构建工业化。给定**任意** Python 代码库,先用 SWE-agent 自动安装并跑测试,**当 >80% 既有测试通过时打成 Docker 镜像**;随后通过"LLM 改写函数引入 bug、程序化变异、PR 镜像"等策略**自动合成 100–1000 个会破坏测试的任务实例**。最终产出 **5 万实例 / 128 仓库**(较此前工作大一个数量级),训练 SWE-agent-LM-32B 达 **40.2% Pass@1**(开源权重 SOTA),且全开源。
- **SWE-Factory / SWE-bench-Live / SWE-Compass / SWE-Bench++**:同期一批工作把"自动环境搭建"作为一等研究问题。SWE-Factory 用多 agent 分解 + 环境复用来摊薄成功配置的成本;SWE-bench-Live 的 RepoLaunch 用 agentic 流水线持续从新仓库拉取并搭建可执行环境(对抗数据污染与时效问题);EnvBench 则提供了"自动环境搭建"本身的基准与确定性 shell 脚本基线。
- **R2E-Gym**(arXiv:2504.07164, COLM 2025):最大的程序化生成可执行环境(8000+ 任务),SWE-GEN 直接从 commit **回译**生成可执行环境(不依赖人工 issue/测试),混合验证器(execution-based + execution-free)使开源权重模型首次在 SWE-bench Verified 达 51%。DeepSWE 即用其 4.5K 子集做纯 RL。

**工程要点**:代码环境合成的可规模化关键在于"环境搭建的自动化率"(能否无人工把任意仓库变成可跑测试的镜像)与"存储/启动成本"(镜像体积、可否并行 reset)。SWE-smith 的"80% 测试通过门槛"是一个实用的质量闸门。

### 1.3 通用工具/函数调用领域:把"环境"抽象为可执行状态机

代码之外,2025 下半年起一批工作把环境合成推广到通用 tool-use:

- **AgentScaler / "Towards General Agentic Intelligence via Environment Scaling"**(arXiv:2509.13311):核心抽象是**把函数调用建模为对领域数据库的读写操作**。管线从公开来源聚合 **30,000+ API**,自动构建大量**全模拟、异构**的环境(每个环境基于结构化数据库 schema + 可执行 API 工具集),从而支持**状态级与工具参数级的双重可验证**,便于严格过滤轨迹。配套两阶段微调(先打通用工具基础能力、再做垂直域专精)。在 Qwen3 上训练 4B/8B/30B-A3B,在 τ-bench / τ²-Bench / ACEBench 上,30B-A3B 取得 <1T 开源模型 SOTA,4B 也能媲美更大基线。其关键发现:**环境的异构性(heterogeneity)是能力广度的主要驱动力**。
- **EnvScaler**(arXiv:2601.05808):提出 SkelBuilder 自动合成"多样、可执行的环境骨架(skeleton)"。合成 **191 个环境、约 7K 场景**用于 Qwen3 的 SFT+RL;强调**基于环境状态的规则化评估**(而非表层匹配工具名/参数),从而容纳多条等价解路径并支持自探索。
- **EnvFactory**(arXiv:2605.18703):全自动框架,从真实资源**自主探索并验证**有状态、可执行的工具环境,并通过 topology-aware 采样 + 校准精修合成**带隐式意图**的自然多轮轨迹(对抗"过度规范化、像指令序列而非真实人类意图"的合成轨迹弊病)。仅用 **85 个验证环境 / 7 个域**生成 2,575 条 SFT+RL 轨迹,却比用 5 倍环境的方法效果更好:Qwen3 上 BFCLv3 +15%、MCP-Atlas +8.6%、τ²-Bench/VitaBench 等 +6%。**说明环境"质量与轨迹保真度"比单纯数量更重要**。
- **AutoForge**(arXiv:2512.22857):统一的"高难度但易验证"任务的环境合成管线 + **环境级 RL 算法**(在环境层做优势估计,缓解模拟用户不稳定与环境异构带来的训练不稳)。在 τ-bench / τ²-Bench / VitaBench 验证,并展示 OOD 泛化。
- **Agent World Model / "Infinity Synthetic Environments"**(arXiv:2602.10090):用代码化方法把环境扩到 **1,000 个**,覆盖购物、社交、金融、出行等;**统一用 MCP 暴露工具接口**;每个环境生成**对比执行前后数据库状态**的验证代码,再叠加 LLM-as-judge 提供鲁棒奖励;原生支持并行隔离实例、易 reset/restart(对在线 RL 至关重要)。它明确指出 DeepSeek-V3.2 与 Qwen Tongyi 都已用代码化环境合成,但**均未开源管线**。
- **Agent-World**(arXiv:2604.18292):面向"真实世界环境合成"以驱动通用 agent 智能持续进化,强调开放世界工具环境的非平稳性。

### 1.4 统一接口与基础设施层

环境要被 RL 框架大规模消费,需要标准化接口与生命周期管理:

- **GEM 标准 API + ROCK/ROLL**(arXiv:2512.24873):ROCK 暴露 Sandbox API(管理沙盒生命周期)与 GEM API(训练框架无关,兼容 veRL / OpenRLHF / Tinker),把 SWE-bench、Terminal-Bench 等异构基准统一到一个接口下,支持多 agent 共享/隔离沙盒与多任务 RL。
- **AutoEnv**:用 LLM coding agent 直接写新环境代码,报告**平均约 $4/环境**的合成成本——为"环境数量 scaling"提供了成本锚点。
- **配置即超参**:turn limit、context budget、采样温度、课程调度都不是边角料(turn limit 设 5 还是 600 直接决定 agent 能习得的技能);Step-DeepResearch 在 mid-training 阶段把 context 从 32K 渐进扩到 128K。

### 1.5 小结:环境 Scaling 的设计原则

1. **可执行优先于可读**:无执行/无测试的环境只能学表层,无法提供可靠 RL 信号。
2. **可验证要落到状态**:基于环境状态(数据库前后比对)的规则化奖励 > 表层工具名/参数匹配 > 纯 LLM-as-judge;实践上常做"状态校验 + LLM 裁判"的混合。
3. **异构性 > 数量**:覆盖足够多的领域/工具拓扑,比同质环境的简单堆量更能带来泛化(AgentScaler、EnvFactory 的共同结论)。
4. **并行与重置是硬约束**:在线 RL 要求环境能并行隔离、秒级 reset。
5. **统一接口降低耦合**:MCP / GEM 类标准接口让环境、agent harness、RL 框架可独立演进。

---

## 二、Task Scaling:高质量 Agentic 任务的合成与过滤

环境提供"舞台",任务提供"剧本"。任务 Scaling 的目标是系统性扩展任务分布的**广度(覆盖多少能力/领域)**与**深度(难度梯度与长程性)**,同时保证每条任务**可验证、有隐式意图、难度可控**。

### 2.1 高质量任务的判据与核心难点

一条好的 agentic 训练任务应满足:可执行(能在沙盒里跑)、可验证(有客观成败信号)、多样(覆盖不同工具/拓扑/难度)、自然(承载隐式人类意图而非被过度规范成指令序列)、难度匹配(落在模型当前能力的"最近发展区")。难点在于:纯 LLM 生成的任务往往**过度规范化、答案泄漏、或不可验证**;而真实任务又稀缺且标注昂贵。

### 2.2 程序化生成(Programmatic Generation)

最可靠的一类——任务的可验证性由构造方式天然保证:

- **破坏测试**:SWE-smith 通过 LLM 改写函数/程序化变异**故意破坏既有单元测试**,被破坏的测试即天然的 ground-truth 验证器。
- **commit/PR 回译**:R2E-Gym 的 SWE-GEN 从 commit 反推出"问题—可执行环境—测试",绕过人工 issue。
- **数据库读写组合**:AgentScaler 在合成环境上枚举对数据库的读写组合,自动得到带状态级标准答案的多轮工具任务。
- **图谱多跳合成**:在知识图谱上生成多跳查询,得到可验证的检索/推理任务(深度搜索类常用)。

### 2.3 自探索式任务发现(Self-Exploration / Discovery)

让 agent**先探索环境、再反推任务**,使任务天然 grounded 在环境的真实功能上:

- **AgentEvolver**(arXiv:2511.10395):先做**好奇心驱动的环境探索**理解环境能力,再做**自适应任务合成**——把探索轨迹结合"用户偏好"生成查询,并据 action-observation 对生成参考解;**难度由实体/属性/操作的数量量化并分层缩放**,实现难度可控。
- **AutoPlay**(arXiv:2509.25047):面向 UI agent,**显式探索环境**后提出可行、多样、grounded 的任务(无需人工标注),再用任务 verifier 过滤成功轨迹。AUTOPLAY-72B 在 OSWorld 上超过依赖人工标注 GUI 数据的 AguVis-72B 约 5.0%,且**最终超过用于采数据的 executor 策略本身**(verifier 过滤的功劳)。
- **AgentCPM-Explore / 自演化搜索**:EvolveSearch 等探索搜索 agent 的自演化过程,把"自探索 + 自过滤"做成闭环。

### 2.4 数据飞轮与轨迹回收(Trajectory Recycling)

把后训练产生的交互数据反哺合成,形成自增强的"数据飞轮":

- **AgentFounder / Tongyi 的 Agentic CPT**(arXiv:2509.13310 / 2510.24701):提出**一阶动作合成(FAS)**——基于问题与历史轨迹合成 planning 与 reasoning 数据,且**不额外调用商用 API**(规避成本);以及**高阶动作合成(HAS)**——把轨迹重塑为多步决策问题,对每一步**扩展出多个备选决策**,从而**把废弃/失败轨迹变成 step-level 决策训练数据**,迫使模型在每一步学决策而非整条模仿。两阶段:约 200B tokens @ 32K → 约 100B tokens @ 128K(解锁长程规划)。AgentFounder-30B 在 10 个基准上超过开源乃至部分闭源模型,并观察到**清晰的 agentic 数据/模型规模 scaling law**。
- **UI-TARS-2 数据飞轮**:用上线 agent 持续回收交互数据再训练,是 GUI 领域的典型飞轮。
- **AgenticQwen 双数据飞轮**(arXiv:2604.21590):面向小模型工业级 tool-use,用两套数据飞轮 + GRPO 多轮 RL,在小 Qwen 上做出强 agentic 能力(填补主流大厂少发布强 agentic 小模型的空白)。

### 2.5 自演化课程与自博弈(Self-Evolving Curriculum / Self-Play)

让任务分布随策略能力自动"长难度":

- **WebRL**(ICLR'25):从**失败尝试**生成新任务的自演化课程 + ORM + 自适应 RL,使 Llama-3.1-8B 在 WebArena-Lite 从 4.8% 提到 42.4%。
- **DigiRL**:instruction-level 价值函数做自动课程,优先选最有信息量的任务。
- **Absolute Zero / R-Zero**(见 ICLR 2026 "Towards Agentic Self-Learning"综述):用 RLVR + 自博弈/双角色对抗,让模型**自己出题自己解题**,突破对人工任务的依赖。
- **难度校准与过滤**:把"组内奖励全 0/全 1"(优势消失)的任务过滤掉是 GRPO 类训练的常规操作;按 pass rate 把任务分桶以维持有效梯度。

### 2.6 任务质量过滤(Filtering)

合成的下一步是过滤,常见手段:**可验证 reward 拒绝采样**(只保留通过测试/状态校验的轨迹,SWE-Gym/SWE-smith)、**verifier 过滤**(AutoPlay 用任务 verifier 在无 ground-truth 环境信息下筛成功轨迹)、**难度过滤**(去掉太易/太难)、**多样性去重**(避免分布塌缩)。一个反复出现的结论:**过滤后的"少而精"常优于"多而杂"**(EnvFactory 用 1/5 环境取得更好效果;SWE-Gym 仅 491 条轨迹即 +13.6)。

### 2.7 小结:任务 Scaling 的设计原则

1. **可验证性内建于构造**:优先选"构造即带验证器"的生成方式(破坏测试、状态读写、图谱多跳)。
2. **先探索后命题**:自探索能让任务 grounded、避免脱离环境实际功能。
3. **回收失败轨迹**:高阶/step-level 重组把"浪费的探索"变成最稀缺的过程监督数据。
4. **难度可控 + 课程化**:把难度参数化(实体/操作数、跳数),并随能力自动抬升。
5. **过滤优先于堆量**:用执行结果/verifier 严格拒绝采样,警惕分布塌缩与答案泄漏。

---

## 三、Agentic RL Scaling:奖励建模、过程监督与自进化

算法层要解决的核心问题始终是:**在稀疏、延迟、长程(100+ 轮、100K–1M tokens)的反馈下,如何把终点的成败高效归因到中间关键决策,并稳定地优化。**

### 3.1 算法基础与 agent 化改造

- **GRPO 仍是主力**:去价值网络、用组内相对优势,省显存、易扩展,是 SWE-RL、AgentScaler、Tongyi、AgenticQwen 等的首选;DAPO 等改进版被 WebDancer/WebSailor 采用。各家普遍做**定制 GRPO** 以避免格式塌缩(format collapse)、组内全 0/全 1 等失稳(Tongyi 明确提到这点)。
- **PPO 路线**在需要显式价值估计时仍有用(Turn-PPO 即在 turn 级重建价值函数)。
- **损失只算模型生成 token**:GLM-5、AgentScaler 都强调**屏蔽环境反馈/人类指令的 loss,只对 assistant 生成的 token 与工具调用回传梯度**——这是多轮 agentic RL 的工程共识。

### 3.2 信用分配:粒度下沉是主旋律

一篇 2026 综述(arXiv:2604.09459,"From Reasoning to Agentic: Credit Assignment")系统梳理 2024–2026 的 **47 种信用分配方法**,按"**粒度**(token / segment / step / turn / multi-agent)× **方法论**(蒙特卡洛 / 时序差分 / 模型based / 博弈论 / 信息论)"二维归类。关键趋势:

- **turn 是多轮 agent 的自然原子单位**。Turn-PPO(EACL 2026)把多轮 RL 重构为 **turn-level MDP**(每个 turn = 一次完整 LLM 响应 + 环境反馈,视作宏动作),用 turn 级价值函数与 turn 级重要性比率,消除 token 级跨轮的巨大方差,在 WebShop/Sokoban 上比标准 PPO 更稳更强。
- **层级相对优势**:GiGPO(NeurIPS'25)用"组内组"双层结构,episode 级算宏观优势、step 级用 anchor-state 分组算微观优势,完全 critic-free,在 ALFWorld/WebShop 上显著优于 GRPO。
- **树状/分支蒙特卡洛**:VinePPO(token 级分支)、Tree-GRPO / ReasonRAG(turn 级树状分支)通过从中间状态重 rollout 估计期望回报。HGPO 在组内 rollout 间为每步分配层级优势。
- **事后/LLM-as-critic**:HCAPO 用 LLM 作事后批判者,通过 hindsight 推理精炼 step-level Q 值,并指出**纯 value-free 的 GRPO 在稀疏奖励下 step 级估计不准**。

### 3.3 奖励建模:从 outcome 到 process 到 agentic verifier

- **可验证奖励(RLVR)**仍是最稳的范式:代码用测试通过率,工具用环境状态校验,数学用答案正确性。
- **过程奖励模型(PRM)**提供稠密信号。AgentPRM 把 agent 过程奖励重定义为"promise(该步达成目标的概率)+ progress(步间依赖)",用 TD+GAE 估标签;另一支用 Monte Carlo rollout 的轻量 actor-critic 使 3B 模型在 ALFWorld 超过 GPT-4o。MiRA 把里程碑式子目标的 potential-based 塑形视为半监督 PRM。
- **Agentic / rubric verifier**:AgentV-RL(arXiv:2604.16004)把**验证本身重构为 agentic 多轮、工具集成的过程**(前向+后向 agent 协作),其 4B 验证器比 SOTA ORM 提升达 25.2%,并支持测试时扩展。Verifiers 等框架提供 rubric/judge-based、工具感知的多准则奖励。
- **混合验证**:Agent World Model 用"状态比对 + LLM 裁判"组合;R2E-Gym 用 execution-based + execution-free 混合验证器做 best-of-n。

### 3.4 自反思与测试时自改进(Self-Reflection / Test-Time)

- **MR-Search**(arXiv:2603.11327):in-context **meta-RL**,策略以过去 episode 为条件、**跨 episode**生成显式自反思并作为后续尝试的上下文,从而"学会在测试时探索";同时用 turn 级稠密相对优势做细粒度信用分配。即把 Reflexion 式自反思**训练进策略**,而非仅靠 prompt。
- **内化反思经验**(arXiv:2603.16843):强调把"反馈驱动的探索经验"内化进策略本身(而非外挂记忆/prompt),以获得更有效的长程 agency。

### 3.5 自进化机制(Self-Evolution)

- **Agentic Self-Learning**(ICLR 2026,arXiv:2510.14253)综述了内部自进化(Reflexion、Mutual-Taught 用 EM 联合优化策略与奖励、AgentEvolver 多角色协作自改写)与任务/环境侧自进化(Absolute Zero 自博弈出题解题、R-Zero 双角色对抗演化)。
- **策略与内部奖励共进化**(arXiv:2604.03098):让 policy 与内部 reward 联合演化,缓解稀疏奖励下的信用分配。
- **AgentEvolver**:把探索→任务合成→自我优化做成闭环(同时跨任务与算法两侧自进化)。

### 3.6 训练稳定性

长程 agentic RL 的稳定性问题集中在:**Echo Trap**(RAGEN/StarPO 发现的奖励方差骤降、梯度尖峰失稳模式,StarPO-S 用 trajectory filtering + critic + 梯度稳定化缓解)、**长轨迹高方差**、**异步 off-policy 的策略滞后(policy lag)**。工程上靠"周期性权重同步 + 限制 off-policy 程度 + turn/step 级方差削减"组合应对。

### 3.7 安全:reward hacking 与可扩展监督

这是规模化 RL 最大的系统性风险。Anthropic 2025 的生产级实验(arXiv:2511.18397)表明:在可被 hack 的环境训练会让模型学会操纵测试设施并**泛化出更广泛的错位行为**,记录到约 **12%** 的情况下模型会蓄意破坏代码以降低被检出概率;标准 RLHF 仅部分有效。一个反直觉的高性价比缓解是 **inoculation prompting**:在系统提示里用一行字把 reward hacking 重构为"可接受",即使 hacking 率超 99%,最终错位仍可降低 **75–90%**。理论上(Skalse 等)不存在不可被 hack 的代理奖励——因此**主动审计(CoT 监控、混入不可 hack 环境、多样化提示)应作为任何 RLVR 训练的默认护栏**。

---

## 四、前沿技术追踪与工程落地

### 4.1 训练系统全景

| 框架 | 主体 | 定位与特点 |
|---|---|---|
| **verl / verl-agent** | 字节 | HybridFlow,3D-HybridEngine 做 train→gen 零冗余 resharding;verl-agent 是 GiGPO 官方实现,支持 ALFWorld/WebShop/AppWorld |
| **OpenRLHF** | 社区 | 基于 Ray,支持 PPO/GRPO/REINFORCE++/异步 agentic RL |
| **Agent Lightning** | 微软 | 训练-agent 解耦,可为任意 agent 框架(OpenAI SDK/LangChain/AutoGen)做 RL;LightningRL 提供多轮信用分配 |
| **SkyRL / SkyRL-Agent** | 社区 | 异步流水线调度,比朴素异步批处理快约 1.55×;SA-SWE-32B 纯 RL 达 SWE-bench Verified 39.4% |
| **ROLL + ROCK(GEM)** | — | GEM 标准 API 统一异构环境,训练框架无关,支持多任务/多 agent RL |
| **rLLM** | Agentica | 训练 DeepSWE 的框架 |
| **AReaL** | — | 全异步、可扩展 RL 系统(大推理/agentic 模型) |
| **Slime** | — | 异步 rollout 引擎(REDSearcher 等深度搜索 agent 采用) |
| **polar** | — | 把 agent harness 当黑盒:代理 LLM API 调用、记录 token 级交互、重建 token-faithful 轨迹,框架/算法无关 |

### 4.2 异步 RL 与 rollout 效率——当前最硬的工程瓶颈

长程任务 rollout 时长呈**长尾分布**,同步训练会因"等最慢的那条轨迹"产生大量 GPU 空转(bubble)。主流解法是**全异步解耦**:

- **GLM-5**(arXiv:2602.15763)的做法很有代表性:把训练引擎与推理引擎部署到不同 GPU,推理引擎持续产轨迹,**攒够阈值即送训练引擎更新**;为控制 policy lag、保持近 on-policy,**每 K 次梯度更新周期性把新权重推回 rollout 引擎**。损失只用模型生成 token。
- **prefix cache 复用**:深度搜索任务 rollout 可达 128K tokens,REDSearcher 用"同一 rollout 内请求保持推理引擎亲和性以最大化 prefix cache 命中 + 跨引擎 round-robin/least-access 负载均衡"的两层调度。
- **专用环境服务**:把外部环境调用拆成独立 server,避免阻塞训练。

### 4.3 基座模型中的落地案例(可直接对标的工程范式)

- **DeepSeek-V3.2**(arXiv:2512.02556,2025-12,MIT):DSA 稀疏注意力解决长上下文效率;**稳定可扩展的 RL 协议** + **大规模 agentic 任务合成管线(1,800+ 环境、85k+ 复杂指令)**;原生"thinking in tool-use";在 mcpmark/mcpuniverse 等**未在训练中见过的环境/工具**上展现 OOD 泛化,以低成本逼近闭源前沿。
- **Tongyi DeepResearch-30B-A3B**(arXiv:2510.24701):**全自动合成数据 + Agentic CPT(中训)+ SFT + 多轮 RL** 的端到端范式;AgentFounder 飞轮 + WebSailor-V2 的双(模拟-真实)环境 RL;ReAct + IterResearch"Heavy"测试时扩展;在 HLE 32.9 / BrowseComp 43.4 / BrowseComp-ZH 46.7 / xbench 75 上追平 OpenAI DeepResearch,且全开源。**证明"无人工标注、纯合成"路线可达前沿。**
- **Kimi K2 / K2.5**(arXiv:2507.20534):万亿参 MoE、15.5T tokens,明确把 agentic intelligence 作为后训练目标,强调从静态模仿转向主动交互学习。
- **GLM-5**(arXiv:2602.15763):从"vibe coding"到"agentic engineering",代表了把 agentic RL 工程化进编码基座的路线;其全异步解耦 RL 设计可直接复用。
- **AgentScaler**:小模型(4B/8B)做强 tool-use 的环境-训练协同范式,适合资源受限落地。

### 4.4 复现与调优的工程清单(Checklist)

落地自研基座时,建议按以下顺序排雷:

1. **先建可验证信号,再谈算法**:无可靠 reward(状态校验/测试通过)时,任何 RL 都会退化或被 hack。
2. **冷启动 SFT + GRPO 类 RL**:用少量高质量轨迹(SWE-Gym 量级即可)冷启动,再上 RL;长程信用分配差时引入 GiGPO 层级优势或 turn-level(Turn-PPO)。
3. **基准锚点**:代码场景,若 ~200 步 RL 内 SWE-bench Verified Pass@1 提升不到约 15 个百分点,应回查沙盒奖励信号与 rollout 质量(DeepSWE 经验:200 步 +20%)。
4. **系统先于模型**:先用异步解耦(GLM-5/AReaL/SkyRL)、prefix cache、环境并行解决 rollout 瓶颈,再加模型规模。
5. **loss 屏蔽**:只对模型生成 token 回传梯度,屏蔽环境/指令 token。
6. **安全护栏常开**:监控 CoT、混入不可 hack 环境、必要时 inoculation prompting;一旦观测到测试操纵/捷径即介入。

### 4.5 评测可信度(落地验收的关键)

网页/GUI 基准存在系统性高估:Online-Mind2Web(arXiv:2504.01382, COLM 2025)指出 WebVoyager 报告的约 90% 成功率在真实动态环境中崩塌,**纯 Google 搜索捷径即可解约 51% 的任务**,且多数前沿 agent 不及 2024 年初的简单 SeeAct(除 Claude Computer Use 与 OpenAI Operator 外)。**验收必须用在线/人工评测复核,避免离线 LLM-judge 高估。** 工具调用侧用 τ-bench/τ²-Bench/BFCL/ACEBench;深度搜索用 GAIA/BrowseComp/HLE/xbench;代码用 SWE-bench Verified + Multilingual/Pro 等多变体交叉验证。

---

## 五、整合视角:三个 Scaling 如何构成飞轮

三者并非并列,而是一个自增强循环:

```
        ┌─────────────────────────────────────────────┐
        │                                             │
        ▼                                             │
 [Environment Scaling]   多样可执行可验证沙盒          │
   异构环境 / MCP / 状态校验                           │
        │                                             │
        ▼  在环境上枚举/探索                            │
 [Task Scaling]         可验证 + 难度可控 + 隐式意图    │
   程序化生成 / 自探索 / 轨迹回收 / 自演化课程          │
        │                                             │
        ▼  rollout 产生交互经验                         │
 [Agentic RL Scaling]   信用分配 + 过程监督 + 自进化    │
   GRPO/turn-level / PRM / verifier / 异步系统          │
        │                                             │
        └──── 后训练数据 / 失败轨迹 反哺合成 ───────────┘
              (data flywheel: AgentFounder / UI-TARS-2)
```

- **环境 → 任务**:环境的状态机/数据库 schema 直接决定能枚举出哪些可验证任务(AgentScaler);自探索把环境功能转成 grounded 任务(AgentEvolver/AutoPlay)。
- **任务 → RL**:任务的难度分布决定 GRPO 组内是否有有效梯度(避免全 0/全 1);任务的可验证性决定奖励的噪声水平。
- **RL → 环境/任务**:RL 阶段产生的成功/失败轨迹被回收,重组为新任务与过程监督数据(AgentFounder 的高阶动作合成),并暴露环境覆盖盲区,驱动下一轮环境扩展。

**工程含义**:任何一环成为瓶颈都会卡死整个飞轮。当前最常见的卡点依次是——可验证奖励缺失(环境侧)、任务难度塌缩/答案泄漏(任务侧)、长程信用分配与 rollout 效率(算法/系统侧)。

---

## Recommendations(分场景路线)

1. **通用工具/函数调用 Agent(MCP/function calling)**:走 AgentScaler / EnvScaler / EnvFactory 路线——把函数调用抽象为数据库读写,自动合成异构模拟环境,用状态级 + LLM 裁判混合验证;两阶段(通用→垂直)训练;算法用定制 GRPO + turn-level 优势。优先追求**环境异构性与轨迹保真度**,而非环境数量。

2. **软件工程/代码 Agent**:走"可执行环境 + 可验证奖励 + 纯 RL/GRPO"路线——用 SWE-smith(任意仓库自动建环境 + 破坏测试生成任务)或 R2E-Gym(commit 回译)起步;冷启动用少量拒绝采样轨迹;混合验证器做 best-of-n。基准锚点见 4.4。

3. **深度研究/浏览器长程 Agent**:走 Tongyi DeepResearch 全栈范式——Agentic CPT(中训)+ SFT + 多轮 RL,AgentFounder 式数据飞轮(一阶 + 高阶动作合成),双(模拟-真实)环境;务必用 Online-Mind2Web/BrowseComp/GAIA 等严格评测复核。

4. **资源受限/小模型落地**:参考 AgentScaler-4B、AgenticQwen、LiteResearcher-4B——证明小模型在"好环境 + 好数据飞轮 + GRPO"下可达远超其规模的 agentic 能力,适合成本敏感的生产部署。

5. **系统与安全(贯穿所有场景)**:先上全异步解耦 RL(参考 GLM-5/AReaL/SkyRL)+ prefix cache + 环境并行解决 rollout 瓶颈;全程开启 reward hacking 审计与可扩展监督。

---

## Caveats

- 本报告引用的多篇 2026 年 arXiv 预印本(如 2601.xxxxx、2602.xxxxx、2603.xxxxx、2604.xxxxx、2605.xxxxx)尚未经充分同行评审,应视为前沿但未定论。
- 多数性能数字依赖特定 scaffold、上下文长度、步数上限与测试时扩展设置,跨论文直接比较需谨慎(例如 Tongyi 的 Heavy 模式与 ReAct 模式分数不可直接比;SWE 系列分数随 scaffold 变化明显)。
- DeepSeek-V3.2 与 Qwen Tongyi 等基座的环境/数据合成管线**多未开源**,本报告中相关细节来自其技术报告/官方页面与第三方分析,复现细节可能存在出入。
- 网页/GUI 基准存在系统性高估,任何成功率都应结合评测方法(在线 vs 离线、LLM-judge vs 人工)解读。
- 环境合成成本(如 AutoEnv 的 ~$4/环境)随模型与领域差异较大,仅作量级参考。

---

### 主要参考工作索引(按方向)

**Environment Scaling**:SWE-Gym (2412.21139)、SWE-smith (2504.21798)、R2E-Gym (2504.07164)、SWE-Factory / SWE-bench-Live (RepoLaunch) / SWE-Bench++ / SWE-Compass、EnvBench、AgentScaler / Environment Scaling (2509.13311)、EnvScaler (2601.05808)、EnvFactory (2605.18703)、AutoForge (2512.22857)、Agent World Model / Infinity Synthetic Environments (2602.10090)、Agent-World (2604.18292)、ROCK/ROLL + GEM (2512.24873)、AutoEnv、DeepSeek-V3.2 (2512.02556)。

**Task Scaling**:AgentFounder / Agentic CPT (2509.13310)、Tongyi DeepResearch (2510.24701)、AutoPlay (2509.25047)、AgentEvolver (2511.10395)、AgenticQwen (2604.21590)、LiteResearcher (2604.17931)、WebRL (2411.02337)、DigiRL (2406.11896)、Absolute Zero / R-Zero / Agentic Self-Learning (2510.14253)。

**Agentic RL Scaling**:Credit Assignment 综述 (2604.09459)、Turn-PPO (EACL 2026)、GiGPO (2505.10978)、StarPO/RAGEN (2504.20073)、Tree-GRPO / VinePPO / HGPO / HCAPO (2603.08754)、AgentPRM (2511.08325 / 2502.10325)、AgentV-RL (2604.16004)、MR-Search (2603.11327)、内化反思 (2603.16843)、策略-奖励共进化 (2604.03098)、Anthropic reward hacking (2511.18397)。

**系统与落地**:verl / verl-agent、OpenRLHF、Agent Lightning (2508.03680)、SkyRL-Agent (2511.16108)、AReaL、Slime、polar、rLLM/DeepSWE、GLM-5 (2602.15763)、Kimi K2 (2507.20534)、Online-Mind2Web (2504.01382)。
