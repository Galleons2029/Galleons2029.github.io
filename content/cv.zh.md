---
title: CV_zh
publish: true
---

# 柳佳龙

- 手机：(+86) 155-5821-6267
- 邮箱：Galleons@whu.edu.cn
- Github：github.com/Galleons2029

# 教育经历

### 武大-上交BCMI实验室联合培养，研究方向为 LLM Post-Training on Long-Horizon Tasks
- 导师: 李祖超
- 学校：武汉大学-硕士
- 预计 2027 毕业

### 安全工程
- 学校：湘潭大学-学士
- 时间：2020-2024

# 专业经历

## 研究经历

### SRPO: Self-Reflective Policy Optimization for Long-Horizon Reasoning
- 一作录用
- 武汉大学/上海交通大学
- ICML 2026

- 提出**自反思策略优化 (SRPO)** 框架，让模型分析自身推理轨迹并将错误综合为"反思补丁 (Reflection Patch)"，配合 **Reset-with-Memory** 机制将稀疏终端信号转化为密集 token 级学习信号 ($O(1) \to O(T)$)，无需外部评论器、奖励模型或更大教师模型即可破解长程任务的信用分配难题；基于 Qwen3-8B 仅以 **8\% 训练 FLOPs** 取得 SOTA：AIME'24 73.3\%，并在 WebShop (64.7\%)、ALFWorld (76.8\%)、SWE-Bench-Lite (31.2\%) 等智能体基准上显著领先。

### TTPO: Turn-Level Trajectory Optimization for Multi-Turn LLM Reasoning
- 一作在投 Meta 3.5
- 武汉大学/中科院/上海交通大学
- EMNLP 2026

- 提出**回合级轨迹优化 (TTPO)**，一种无需评论器的强化学习方法，将 GRPO 重参数化至对话回合粒度并以每回合作为宏动作 (macro-action)，配套**回合级重要性比率代理** (PPO 式裁剪)、**轨迹裁剪** (剪除低奖励分支以抑制指数级增长) 与**跨回合奖励传播** (将终端奖励回传至早期回合) 三大核心组件；在六个多轮对话任务上将运行间波动性降低约 19--22 个 $U_{90\text{-}10}$ 点、平均性能提升 9--12 个百分点，并在保持高分位质量 ($A_{90}$) 的同时于人工盲评中被一致优先选择。

### HELM: Hierarchical Epistemic Learned Memory for Long-Horizon Agents
- 一作在投
- 武汉大学
- EMNLP 2026

- 提出**层次化认知学习记忆 (HELM)** 框架，将记忆暴露为事件驱动接口，构建**三层嵌套记忆存储 (SHNM)**——情景痕迹 ($\mathcal{M}^{(1)}$)、整合回忆 ($\mathcal{M}^{(2)}$) 与主题索引 ($\mathcal{M}^{(3)}$)，通过溯源边与认知元数据 (时间戳、来源、工具状态) 互联，并耦合**认知治理 (Epistemic Governance)** 与学习型控制器，在预算约束下决策 READ/WRITE/CONSOLIDATE/PRUNE 操作以遏制无效证据的高置信误用；在五个长程基准的匹配预算与脚手架条件下显著提升 GAIA (L2: $8.9 \to 12.8$; L3: $0.8 \to 1.7$) 与 WebArena ($19.5 \to 22.4$) 成功率，并支持可审计的回忆溯源诊断。

### CHOP: Segment-Level On-Policy Distillation for Long-Horizon Agents
- 一作在投
- 武汉大学/上海交通大学/小米
- NeurIPS 2026

- 提出面向长链路 Agent 的混合蒸馏方法 **CHOP**，在训练中动态判断 token 级监督的可靠性，并在低可靠状态下切换为 **segment-level Wasserstein 分布匹配**，从根源缓解传统 On-Policy Distillation 在长程推理与复杂 Agent 任务中的性能退化；在 AIME 2024 长程数学推理与 SWE-bench Verified 软件工程任务上均显著优于现有 OPD 与 Top-K LSM 基线，实现更稳定、更高效的长上下文能力迁移。

## 项目经历

### AI银行财务管理平台 Bank-Copilot

- 采用 Harness 架构构建长程 Agent，基于 **data-driven** 设计模式结合 Flink 搭建后台智能体服务，支持 7x24 小时稳定运行。
- 基于 Graphiti + FalkorDB 构建图谱记忆，将关键实体、分录关系、业务事件 3 类信息结构化入图，并结合知识库检索形成“向量记忆 + 图谱记忆”双通道语义支撑。
- 搭建类 Unix 执行环境，开放文件系统与 CLI 两类工具访问能力，扩展 Agent 行动空间并降低对模型上下文窗口的依赖。
- 将工具调用、容错机制与 Langfuse 观测统一到同一状态图，结合 PostgreSQL Checkpointer 实现会话级回溯与容灾恢复。

### 深度研究助手 Cascade-RAG

- 采用 Multi-Agent 架构进行上下文管理与复杂任务拆解，保障长程记忆下的任务一致性并降低状态漂移风险。
- 构建递归式多智能体调度机制，实现动态任务分配与质量控制，稳定支持单篇 10,000+ 字长文本报告输出。
- 基于 LangGraph Deep Agent 框架搭建“规划-构建-验证-修复”4 阶段闭环，支持多轮迭代与反馈驱动的自我改进。
- 通过 progressive disclosure 优化工具调用路径，并结合缓存机制降低重复查询成本；使用子代理隔离提升复杂任务执行稳定性。

### SGLang 开源社区维护

- 参与并修复 SGLang Diffusion 社区 bugs，提升相关模块稳定性。
- 测试 Agent 场景下的兼容性与可用性，排查训推一致性相关问题。
- 参与 SGLang RL 在多模态多轮对话场景下的开发与测试，覆盖训练与推理两条链路。

## 工作经历

### Agent 后训练实习生
- 地点：上海
- 公司：小红书 (RedNote)
- 时间：2026.6-至今

- 参与新一代 **dots** 模型的**通用 Agent 后训练**，提升其在多轮工具调用、规划与长程交互任务中的智能体能力。

### 推理加速实习生
- 地点：北京, 海淀
- 公司：小米科技有限公司.
- 时间：2026.3-2026.6

- 参与小米推理加速框架的设计与开发，负责优化模型推理效率和资源利用率，支持公司内部多个 AI 产品的部署和运行。
- 参与全模态推理框架 vLLM-Omni 开源贡献，并适配公司内部大模型部署，排查推理全链路并修复堵塞瓶颈。
- 对内部本地部署模型进行性能分析，针对性能瓶颈部分采用 CUDA Graphs、内核融合等技术进行优化，提升推理效率。

### 算法工程师
- 地点：湖南, 长沙
- 公司：北京大学长沙计算与数字经济研究院.
- 时间：2025.6-2025.12

- 针对北大鹤思算力调度平台搭建智能体助手“小蒜”，集成知识库问答与操作引导能力，基于多智体路由架构深度集成鹤思平台工具调用，问答准确率大于 90\%， 长程任务完成率超过 80\%。
- 采用 Agentic Document Extraction 方案实现企业生产级复杂文档解析，解决当前传统 OCR 模型在极端格式、低质量扫描件上的性能瓶颈，提升文档解析准确率约 15\%（80\%->95\%）。
- 基于在线策略蒸馏技术对本地部署小模型在实际 Agent 任务场景下的性能进行增强。

### 算法研究员
- 地点：湖南, 长沙
- 公司：昆仑元人工智能有限公司.
- 时间：2025.3-2025.6

- 跟进 Reasoning-RAG 相关前沿研究，复现并评测 Research-o1、Research-R1、R1-researcher 等原生检索模型在大规模企业文档检索场景下的效果。
- 设计并实现基于 RAG 的智能体系统，支持文本、图像、表格 3 模态输入与多轮对话，实现企业知识场景下的检索与推理闭环。

### 大模型应用工程师
- 地点：湖南, 长沙
- 公司：云研科技有限公司.
- 时间：2024.5-2025.3

- 作为核心开发成员，基于 FastAPI、RabbitMQ、Qdrant 和 MongoDB 构建毕业生就业推荐系统，服务注册用户近 1000 万、日活 10 万+、在线岗位 30 万+。
- 构建 LLM Agent 解析学生专业、成绩、实习经历等在校信息并生成特征标签，结合 FFM 自动特征交叉技术实现个性化职位推荐。
- 使用 Apache Kafka 与 RabbitMQ 实现异步特征生成和通知推送，提升高并发场景下的吞吐能力与系统稳定性。

# 技术技能

| 类别 | 技能 |
|--------|--------|
| Language | Python, C, TypeScript, Java, Rust, \LaTeX, Shell |
| AI Framework | PyTorch, LangGraph, Agno, Scikit-Learn, Keras |
| AI Infra | vLLM, SGLang, VERL, Roll, SLIME |
| Infrastructure | Slurm, Distributed Training, Docker, Optimization, Git, CI/CD pipelines |

# 荣誉

## 证书

- 2024 MIT 数据科学微硕士毕业证书.
- 2023 春季Rustling开源操作系统训练营结业证书.

## 奖项

- 2023 年美国全国大学生建模大赛特等奖提名.
- 2023 年全国大学生建模大赛省一等奖.
- 2025 年全国研究生金融科技大赛二等奖.

# 研究兴趣

本人专注于大模型**后训练**阶段算法研究，尤其关注**强化学习**在智能体场景下的应用,

目前 4 篇中稿/在投 RL 相关一作论文，研究方向主要体现在智能体长程任务中 LLM 多轮对话/交互场景下的优化与提升 (关键词： Memory, Multi-turn conversation, Long-horizon task, Continual learning).

同时，我也在积极参与开源社区的建设与维护，参与包括 vllm-omni、SGLang Diffusion、SGLang RL 等项目，熟练使用 vibe coding 管理维护大型生产项目。