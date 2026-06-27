---
title: 工业界 LLM 后训练 SFT 阶段流程详解：数据侧与训练侧实操参考
date: 2026-06-21
tags:
  - Agentic RL
  - RL Scaling
  - Engineering
  - data
publish: true
---

## TL;DR
- **数据为王，配方趋同**：当前主流做法已收敛为"高质量合成数据为主 + 少量人工标注兜底 + 拒绝采样/LLM-as-judge 严格过滤 + 多技能配比迭代"。规模上两条路线并存——大规模派（Qwen2.5/Qwen3 超 100 万、DeepSeek-V2/V3 约 150 万、推理蒸馏可达数百万至千万级）与精炼派（LIMA 1000 条、DEITA 6000 条）。两者不矛盾：通用对齐"少而精"够用，但推理/agent 等硬技能仍需大规模数据持续扩展。
- **训练侧已标准化**：全参数 SFT、只对 response/assistant token 计 loss（prompt 与 tool 输出 mask 掉）、序列打包 + cross-document attention 隔离、2 epoch 左右、LR 5e-6~1e-5（随模型增大而减小）、cosine/constant schedule、effective batch ≈128、按 epoch/验证 loss 选 checkpoint。SFT 越来越被定位为 RL 前的"冷启动/格式与行为注入"阶段。
- **Agent/tool-use 是当前最大增量**：核心是 trajectory（thought-action-observation 三元组）多轮 SFT，工具 JSON schema 放进 system prompt，只对 assistant 的 reasoning + tool_call token 计 loss、mask 掉 user 与 tool 返回；数据几乎全合成（APIGen 6 万验证样本、ToolACE 26,507 个 API、ToolBench 16,464 个 API、Toucan 150 万真实 MCP trajectory），并辅以执行级验证与 error-masking。

---

## Key Findings

1. **合成数据已成主流，人工标注退居"种子+质检"。** Nemotron-4 340B 全程仅依赖约 2 万条人工标注，原文："throughout the entire alignment process, we relied on only approximately 20K human-annotated data (10K for supervised fine-tuning, 10K Helpsteer2 data for reward model training and preference fine-tuning), while our data generation pipeline synthesized over 98% of the data used for supervised fine-tuning and preference fine-tuning."Llama 3 原文："We use synthetic data generation to produce the vast majority of our SFT examples, iterating multiple times to produce higher and higher quality synthetic data across all capabilities."Tülu 3 的 939,344 条 prompt 中 57% 来自公开资源、43% 内部合成，且"All of our new synthetic SFT datasets contain responses that are created by either GPT-4o or Claude 3.5 Sonnet (for coding)"。

2. **拒绝采样（rejection sampling）是连接 SFT 与 RL 的关键引擎。** Llama 3 对每个 prompt 从最新策略采样 K=10~30 个回答，用奖励模型选最优作为 SFT 目标。DeepSeek-R1 在 RL 收敛后用拒绝采样收集约 60 万条推理样本 + 20 万条非推理样本，共约 80 万条，再对 DeepSeek-V3-Base 做 2 epoch SFT 回炉。Qwen3 的"thinking"数据通过对 Stage-1 query 用 Stage-2 模型自身做拒绝采样生成。

3. **数据量级跨越三个数量级。** LIMA 用 1000 条人工精选即对齐（750 条来自 Stack Exchange/wikiHow 社区 + 250 条人工撰写，微调 LLaMA-65B）；DEITA 6000 条达到 SOTA 级数据效率；DeepSeek-V2/V3 约 150 万条；Qwen2.5/Qwen3 超 100 万条；推理蒸馏数据集（AM-DeepSeek-R1 140 万、OpenThoughts 扩到 120 万）走大规模路线。新证据（Nanbeige4）显示在控制分布与质量后，推理 SFT 从数十万扩到千万级仍无饱和。

4. **配比按"技能"组织，通过 ablation 迭代。** Llama 3 SFT 数据：通用英语 52.66%、推理与工具 21.19%、代码 14.89%、考试类 8.14%、多语言 3.01%、长上下文 0.11%。Tülu 3 主混合 939,344 条，先做单技能最优 mixture 再合并迭代。

5. **训练超参高度收敛。** 典型：2 epoch、LR 5e-6（8B）~2e-6（70B）、effective batch 128、max len 4096、按验证 loss 选 checkpoint（常为第 1 epoch）。Llama 3 旗舰用 LR ~1e-5、8.5K–9K steps。

6. **多阶段/课程式 SFT 成为推理模型标配。** Qwen3 四阶段（长 CoT 冷启动 → 推理 RL → thinking/non-thinking 融合 → 通用 RL）；DeepSeek-R1 冷启动 SFT → 推理 RL → 拒绝采样 SFT → 全场景 RL；长上下文普遍两阶段（先短后长）。

7. **Agent SFT 的特殊性在 trajectory 结构与 loss masking。** 训练样本为 thought-action-observation 序列，只对 assistant token 计 loss、mask user 与 tool 返回；不在训练时真实执行工具（用缓存 observation）；失败轮次可做 error-masking 提升信噪比。

---

## Details

### 一、数据侧（Data Side）

#### 1.1 SFT 数据构造方法

**(a) 人工标注。** 仍是"金标准"但成本高，如今主要用于：种子数据（Self-Instruct 175 条种子、LIMA 1000 条人工精选）、质量校验、非推理数据人工验证（DeepSeek-V3 的创意写作/角色扮演由 V2.5 生成后人工校验）。Nemotron-4 340B 全程仅约 2 万条人工标注（1 万 SFT + 1 万 HelpSteer2 用于 RM/偏好）。

**(b) Self-Instruct（自举式合成）。** 从约 175 条人工种子任务出发，prompt LLM 生成新指令 + 实例，经过滤（去除低质量/相似）再回填任务池。原始论文产出约 5.2 万条指令、约 8.2 万条实例，原文："GPT3 SELF-INST outperforms GPT3 (the original model) by a large margin (+33.1%) and nearly matches the performance of InstructGPT001."Alpaca 即用此法从 175 种子生成 5.2 万条。

**(c) Evol-Instruct（WizardLM）。** 用 LLM 把指令"进化"成更复杂版本，分两类：In-Depth Evolving（加约束、深化、具体化、增加推理步骤、复杂化输入）和 In-Breadth Evolving（变异生成新指令）；并用 Elimination Evolving 过滤失败进化。WizardMath 对每条指令用 GPT-4 进化 5 轮（2 轮 downward 降难度 + 3 轮 upward 升难度）。Auto Evol-Instruct（WizardLM-2）进一步让 optimizer LLM 自动优化进化方法。

**(d) 蒸馏（distillation）。** 用更强模型生成回答。DeepSeek-R1 蒸馏 6 个开源模型（Llama 3.1/3.3、Qwen2.5）、生成 80 万条推理样本、仅 SFT 无 RL，小模型即获强推理能力。Qwen3 小模型用 strong-to-weak distillation（off-policy + on-policy），论文称蒸馏在性能与训练效率上显著优于 RL。On-policy distillation（Thinking Machines Lab）用 Qwen3-32B 作 teacher 训 Qwen3-8B，以 RL 几分之一成本达到等效推理性能。

**(e) Persona-driven（角色驱动）。** Tencent Persona Hub 原文："PERSONA HUB – a collection of 1 billion diverse personas automatically curated from web data. These 1 billion personas (~13% of the world's total population), acting as distributed carriers of world knowledge"，把 persona 加进合成 prompt 即可引导 LLM 从对应视角生成多样化数据，用于数学/逻辑题、指令、知识文本、游戏 NPC、工具（函数）。Tülu 3 也使用 persona-driven 合成数据。

**(f) Rejection sampling（拒绝采样）。** 见 Key Findings 第 2 点，是 SFT 数据自我改进的核心。

#### 1.2 数据清洗与质量过滤

- **去重 + 去污染（decontamination）。** Tülu 3 对评测集做 decontamination、下采样过大数据集。Llama 3 做去重、清洗、移除高 PII/成人内容域。
- **质量打分 / LLM-as-judge。** DEITA 用 EVOL-COMPLEXITY 打复杂度分 c、EVOL-QUALITY 打质量分 q，合成 evol score s = c×q；diversity 用句向量 cosine 阈值 τ=0.9 控制冗余；最终从约 30 万池中选 6000 条即达 SOTA。Qwen3 non-thinking 数据用自动生成的 checklist 评估 response 质量。
- **奖励模型过滤。** Nemotron 用奖励模型（SteerLM/HelpSteer2，输出 5 个属性标量）做质量过滤与偏好排序；Llama 3 用 RM 在拒绝采样中选最优。
- **难度分级。** Evol-Instruct/WizardMath 显式产生不同复杂度；DEITA 用复杂度维度。
- **执行级验证（agent/code）。** APIGen 三阶段验证、ToolACE 双层验证（见 1.6）。

#### 1.3 数据配比与混合策略

- **Llama 3 SFT 配比（Table 7）**：通用英语 52.66%、推理与工具 21.19%、代码 14.89%、考试类 8.14%、多语言 3.01%、长上下文 0.11%，平均 4.7 轮、846 token。
- **大规模推理 SFT 配比示例（Looped LM，8.3M 例）**：数学 3.5M、代码 3.2M、科学 808K、对话 767K。
- **冷启动配比示例（Nanbeige4）**：约 50% 数学推理、30% 科学推理、20% 代码，32K 上下文。
- **方法论：skill-specific 先行再合并。** Tülu 3 先为每个技能做最优 mixture/model（逼近单技能性能上限），再合并成 preview mix，然后加减数据集迭代以补齐落后技能、做去污染、下采样。

#### 1.4 数据多样性与覆盖度

Self-Instruct 强调种子任务多样性决定产出多样性；Persona Hub 用 10 亿 persona 保证亿级多样性；DEITA 显式用 diversity 维度（cosine 距离阈值选样）；Qwen3 增加低资源语言的翻译任务占比以提升多语言覆盖。

#### 1.5 数据质量 vs 数量的权衡

- **少而精派：** LIMA（1000 条，LLaMA-65B，15 epoch，batch 32~64，trim 2048 token，AdamW）提出 superficial alignment hypothesis——几乎所有知识来自预训练，对齐只教格式/风格；仅加 30 条多轮对话即显著改善多轮能力。DEITA 6000 条达 SOTA 数据效率。
- **大规模派：** Qwen2.5/Qwen3 >100 万、DeepSeek 150 万、推理蒸馏百万至千万级。Nanbeige4 实验显示推理 SFT 在控制分布与质量后扩到千万级仍持续提升、无早期饱和。
- **结论与取舍：** 通用对齐"少而精"成立；但推理、代码、agent 等硬技能的能力上限随高质量数据规模持续上移。实操上建议"通用层精炼 + 硬技能层规模化"。

#### 1.6 Agent / Tool-use / 多轮交互数据（重点）

**数据结构。** 一条 agent 数据是 task-trajectory 对，trajectory 为 thought-action-observation 三元组序列（规划、tool call、tool response、最终回答）。工具的 JSON schema 放进 system prompt（Qwen3/Nemotron 风格）。

**主要数据集与规模：**
- **APIGen / xLAM（Salesforce, arXiv:2406.18518）**：6 万条验证 function-calling 样本，3,673 个可执行 API、21 个类别；三阶段层级验证——格式检查 → 实际函数执行 → 语义验证；人工评测正确率 >95%；7B 模型在 BFCL 超多个 GPT-4，1B 模型超 GPT-3.5-Turbo 与 Claude-3 Haiku。数据由 DeepSeek-V2-Chat 与 Mixtral-8x22B 生成。
- **ToolACE（华为, arXiv:2409.00920）**：自演化合成 26,507 个多样 API；多 agent 交互对话生成；规则+模型"双层验证"（DLV）；ToolACE-8B（基于 LLaMA-3.1-8B-Instruct）在 BFCL-v1 达 91.41%、v2 达 85.77%，媲美最新 GPT-4。
- **ToolLLM / ToolBench（arXiv:2307.16789）**：16,464 个真实 RESTful API、49 类；三阶段（API 收集 → ChatGPT 生成指令 → ChatGPT 标注解路径）；DFSDT（深度优先搜索决策树）优于 ReAct，可评估多条推理路径并回溯（用 DFS 而非 BFS 以节省 OpenAI 调用）；最终 126,486 条（指令,解路径）对训练 ToolLLaMA。
- **Toucan（arXiv:2510.01179）**：150 万条 trajectory，来自 495 个真实 MCP server、2,000+ 工具；五阶段流水线（MCP 上线 → 任务合成 → 任务过滤 → trajectory 生成 → trajectory 过滤）+ 三种扩展机制（含多轮对话模拟）；用 5 个模型生成 query、3 个 teacher 模型 + 2 个 agent 框架生成 trajectory，规则+模型双重验证；微调后在 BFCL-V3 超更大的闭源模型。
- **Llama 3 工具（arXiv:2407.21783, §4.3.5）**：训练用 Brave Search、Python interpreter、Wolfram Alpha；支持多轮对话中多步工具调用（写分步计划、依次调用、每次调用后推理）；zero-shot 工具使用（给 in-context 未见过的工具定义即生成正确调用）；核心工具实现为带 docstring 的 Python 对象。token 格式：`<|python_tag|>` 前缀 Python 调用、`<|eom_id|>`（end-of-message）用于工具续接、`<|eot_id|>`（end-of-turn）结束轮次、`ipython` 角色承载工具输出。注：有二手解读称 Llama 3 对 tool-use 主要用人工标注 + 旧模型 SFT 样本而非拒绝采样，需以 §4.3.5 原文为准。
- **Nemotron-Cascade（arXiv:2512.13607, 复用 Llama-Nemotron 工具数据）**：覆盖单轮/多轮/多步，含澄清提问、多工具、多轮工具调用、以及"工具列表里找不到合适工具"的负例；每个对话平均 4.4 个可用工具，全部放 system prompt；response 由 Qwen3-235B-A22B 生成；用于 Stage-2 SFT。

**Loss masking（关键）。** 训练目标只对 assistant 的 thought + tool_call token 计 loss，mask 掉 user 输入和 tool 返回（observation）；训练时不真实执行工具，用数据构造阶段缓存的 observation（避免运行时方差）。进阶做法：**error-masking**——对触发执行错误的轮次 zero out token-level loss，避免强化失败行为；**task-aware context masking**——对冗余/高度相似/被裁剪的历史轮 mask loss 梯度。

**生成方式。** 几乎全合成或自举（Toolformer 式自标注、ToolBench 式大规模生成）；SFT 教会基本格式与工具选择，DPO/RL 再优化"何时调用工具 vs 直接回答"。

### 二、训练侧（Training Side）

#### 2.1 训练配方与超参典型取值

| 模型 | LR | Epoch | Batch | Max len | 备注 |
|---|---|---|---|---|---|
| Tülu 3 8B | 5e-6 | 2 | 128（effective） | 4096 | 超参搜索得出；32 GPU/6h |
| Tülu 3 70B | 2e-6 | 2 | 128 | 4096 | 64 GPU/50h |
| DeepSeek-V2/V3 | 5e-6 | 2 | — | — | 1.5M 数据 |
| Llama 3 旗舰 | ~1e-5 | — | — | — | 8.5K–9K steps |
| Looped LM（8.3M） | 2e-5 | 2 | — | 32K | Adam β=(0.9,0.95), cosine |
| LIMA | — | 15 | 32~64 | 2048 | AdamW |

通用规律：模型越大 LR 越小；2 epoch 最常见；warmup 常用 3% 或 0.1 ratio；schedule 用 cosine decay 或 constant；按 epoch/验证 loss 选 checkpoint（Llama 3 衍生实验发现常是第 1 epoch 最优）；AdamW + weight decay 0.0~0.1 + grad clip 1.0、bfloat16。

#### 2.2 Loss 设计与 masking 策略

- **只对 response/target 计 loss**：Llama 3 用标准交叉熵只对 target token、mask prompt token（实现上将 prompt token 的 label 置为 -100 / ignore_index）。TRL 的 SFTTrainer 用 `completion_only_loss`（prompt-completion 数据集默认只算 completion）和 `assistant_only_loss=True`（对话数据集只算 assistant，需 chat template 支持 `{% generation %}` 标记）。
- **多轮对话**：把整段 trajectory 喂入、只对所有 assistant 轮计 loss；或拆成单轮 prompt-response 对（某些自训练场景实验显示更优）。注意 Qwen3 多轮 thinking 数据中历史 think block 是否进入 loss 上下文需按实现权衡（长 think block 会显著拉长上下文）。
- **weighted / 动态 masking**：error-masking、task-aware context masking（见 1.6）。

#### 2.3 序列打包（sequence packing）与 attention 隔离

- 打包把多条短样本拼成长序列填满 padding，减少 batch 数、提升 GPU 利用率（典型可省 60–85% wall-clock）。
- **cross-document attention masking 必须开启**以保证文档逻辑独立，否则跨样本污染、损害收敛与质量。HuggingFace Transformers 4.44+ 提供 `DataCollatorWithFlattening`，TRL 用 `padding_free=True` 配合 `flash_attn_varlen_func`（计算 cu_seqlens）实现边界感知打包。注意：旧版 TRL `packing=True` 仅处理 input_ids，不处理 attention mask 与 position ids，需谨慎。
- 策略：多轮/对话用 greedy packing，单轮 random packing 即可；多文档任务每序列打包 4–6 篇并开 cross-doc attention（超过该"甜区"反而增加幻觉）。

#### 2.4 多阶段 SFT 训练策略

- **Qwen2.5 长上下文两阶段**：Stage1 仅短指令（≤32,768 token，与其他 Qwen2.5 同数据同步数）；Stage2 混合短（≤32,768）+ 长（≤262,144）指令，兼顾长短任务。
- **Qwen3 四阶段**：长 CoT 冷启动 SFT → 推理 RL（GRPO，3,995 query-verifier 对） → thinking/non-thinking 融合（/think、/no_think 模板，thinking 数据用 Stage-2 模型自身拒绝采样生成） → 通用 RL。
- **DeepSeek-R1**：数千条冷启动 SFT → 推理 RL → 拒绝采样 SFT（约 60 万推理 + 20 万非推理 = 80 万，2 epoch） → 全场景 RL。
- **冷启动 SFT**：用高质量推理数据建立 CoT 基础（Nanbeige4 清洗后收集约 3000 万 QA 样本）。

#### 2.5 长上下文 SFT

- 数据稀缺是核心矛盾。合成方法：WildLong（从真实 query 提取 meta-info、图建模共现、150K 对、支持多文档推理）、Quest（query-centric）、分层合成（扩到百万 token）、LongSkywork（合成表格+检索任务）。
- 配比经验：长上下文 SFT 中保持约 20% 普通（短）SFT 数据效果最佳（LongSkywork）；也有工作（Ultra-Long，100K 例 SFT）发现仅用短上下文 SFT（<8K）即足够，长能力主要靠 continued pretraining 阶段的 RoPE/YaRN scaling 获得。
- 两阶段（先短后长）+ YaRN/DCA/RoPE scaling 是标准组合。

#### 2.6 SFT 与 RLHF/DPO/RLVR 的衔接

- SFT 越来越被定位为 RL 前的"行为/格式/推理模式注入"冷启动阶段。Llama 3 用 SFT → RS → DPO 迭代六轮（每轮收集新偏好与 SFT 数据、从最新模型采样合成数据）。Tülu 3 用 SFT → DPO（length-normalized，实验优于 PPO/SimPO） → RLVR（可验证奖励，适合数学/指令遵循）。
- 经验：DPO 阶段用新 prompt（而非复用 SFT prompt）、scaling 唯一 prompt 数、加 on-policy 数据均提升下游；SFT 数据规模需为偏好优化阶段预留 prompt（Tülu 3 即因把部分 prompt 分配给偏好优化而未进一步扩大 SFT）。

#### 2.7 防过拟合与灾难性遗忘

- **数据混合/replay**：specialization 时混入通用数据可显著抑制遗忘——研究显示即使 15:1 高度倾斜（每 batch 仅 6.2% 通用数据）也能有效正则化（NLI 准确率 83.8% vs 纯数学的 16.5%），且不损失数学性能（11.7% vs 12.0%）。
- **保留 chat template/system prompt 结构**（把 SFT 当 domain-adaptive continued pretraining）以维持指令遵循格式与对话连贯。
- **模型合并（model merging/soup）**：Llama 3 与 Tülu 3 均在各阶段（RM/SFT/DPO）对不同数据/超参的模型做平均（linear weighted averaging / mergekit）。
- **低 LR、少 epoch、early checkpoint** 本身即抗过拟合；用重构基模型指令分布做 rehearsal（无需原始 SFT 数据，Llama-3-70B 医疗实验中优于直接混公开数据集）。

#### 2.8 评估方法

- **开发期 dev set + 留出 unseen 评测**：OLMo 2 倡导保留一部分评测任务不在开发期使用并明确声明，以检验泛化（其用 200 例子集 "GSM*" 指导 annealing mixture 选择）。
- **基准组合**：指令遵循（IFEval）、数学（GSM8K、MATH、AIME）、代码（LiveCodeBench、HumanEval）、知识（MMLU、GPQA）、agent（BFCL、MCP-Universe）；agent 训练发现 **training loss 不是下游性能的好预测指标**。
- **人评 / LLM-as-judge**：MT-bench、AlpacaEval、LMArena；LIMA 用人评（标注者一致性 crowd-crowd 82%、crowd-author 81%、author-author 78%）。

---

## Recommendations

**阶段一：搭建数据流水线（0–4 周）**
1. 先用现成高质量开源 SFT 混合（Tülu 3 SFT Mixture 939K）跑通基线，验证训练/评估闭环。
2. 建立 decontamination（对所有目标评测集）与去重流程——这是不可跳过的卫生步骤。
3. 搭建 LLM-as-judge 质量打分（DEITA 式 complexity×quality + 向量去冗余 τ=0.9），对数据分级。

**阶段二：技能化数据构造与配比（4–10 周）**
4. 按技能（通用对话/代码/数学/推理/agent）分别构建最优 mixture，再合并迭代（Tülu 3 方法论）。初始配比可参考 Llama 3：通用 ~50%、推理与工具 ~21%、代码 ~15%、其余补充。
5. 合成数据用更强模型（GPT-4o/Claude 3.5/DeepSeek-R1/Qwen3-235B）生成 + 拒绝采样（K=10~30，RM 选优）。agent 数据走 APIGen 式三阶段执行验证。
6. 通用层走"少而精"（数万条精选即可），硬技能层（推理/agent）走规模化（数十万至百万级），持续监控是否饱和。

**阶段三：训练与多阶段（10–16 周）**
7. 起手配方：2 epoch、LR 5e-6（≤8B 用 5e-6~1e-5，70B 用 2e-6）、effective batch 128、max len 4096~32K、cosine decay + warmup 3%、AdamW + grad clip 1.0、bfloat16、按验证 loss 选 checkpoint。
8. 开启 sequence packing + cross-document attention masking（TRL `padding_free=True` 或 `DataCollatorWithFlattening`）。
9. 只对 response/assistant 计 loss；多轮与 agent 数据 mask user + tool 输出，失败轮次做 error-masking。
10. 推理/agent 模型采用多阶段：冷启动 SFT →（RL）→ 拒绝采样 SFT → RLVR。长上下文用两阶段（先短后长）+ RoPE/YaRN scaling，保留约 20% 短数据。

**阶段四：agent 专项（并行）**
11. trajectory 数据为 thought-action-observation，工具 schema 入 system prompt（平均每对话约 4–5 工具，务必含负例与澄清提问场景）。
12. 训练时不真实执行工具，用缓存 observation；SFT 打基础格式，DPO/RL 优化"何时调用"。

**触发调整的阈值/信号：**
- 若专项技能提升但通用基准（MMLU/IFEval）下降 >2–3 个点 → 增加通用 replay 比例（从 6% 起逐步加）。
- 若验证 loss 第 1 epoch 后即上升 → 减少 epoch 或降 LR。
- 若 agent 下游（BFCL）不涨但 training loss 很低 → 警惕 loss 不代表性能，需扩充 trajectory 多样性/真实性。
- 若推理基准仍随数据量线性上升 → 继续扩 SFT 规模（尚未饱和）。

---

## Caveats

- **闭源厂商（OpenAI/Anthropic/Google DeepMind）极少公开 SFT 具体配方**，本报告相关推断主要来自开源技术报告与第三方解读，代表行业共识而非这些厂商的确证做法。
- **部分数字来自二手解读或博客**（如 Llama 3 是否对 tool-use 专门使用拒绝采样存在二手解读分歧），已在正文标注；以一手技术报告为准。
- **BFCL 等榜单是移动目标**：ToolACE 91.41% 为 BFCL-v1（2024）数据，跨版本不可直接比较。
- **"少而精 vs 大规模"无统一答案**：取决于基模型强度、目标技能、是否后接 RL。LIMA 结论在纯对齐成立，但在推理/agent 等需要注入新能力的场景，大规模高质量数据仍持续收益。
- **超参取值随基模型、数据规模、并行框架而变**，表中数值为典型区间，务必在自有 dev set 上做小规模搜索。
- **部分前沿数据集与论文（Toucan、Nemotron-Cascade、Nanbeige4 等）较新**，结论可能随社区复现而修正。