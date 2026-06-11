---
title: SFT 要做到什么程度才能进入 RL?——LLM 后训练 SFT→RL 衔接调研报告
date: 2025-09-04
tags:
  - sft
  - post-training
publish: true
---
## TL;DR
- **没有单一阈值,而是"双轨判断":对传统 RLHF(PPO+奖励模型),SFT 只需训练到能稳定遵循格式、产出连贯回复(InstructGPT 仅用约 13,000 条手写示范),它同时充当 RL 起点策略和 KL 参考点;对 RLVR/GRPO 推理任务,SFT 是"cold start",目标是注入格式与推理范式而非把性能拉满——业界(DeepSeek-R1、Qwen3、Nemotron)的共识是 SFT 应"够用即止",为 RL 保留探索空间。**
- **"SFT 做太多反而损害 RL"已是被反复验证的现象:** 过度 SFT 导致熵坍缩(entropy collapse)、生成多样性/pass@k 下降,从而压缩 RL 探索空间;多篇 2025 论文表明,最高评测分的 SFT checkpoint 往往不是最优 RL 起点,应改用 pass@large-k、held-out 泛化损失、熵/self-BLEU 等多样性指标来选 checkpoint。
- **判断"准备好"的可操作信号:** 格式遵循率高、能产出结构化 CoT、pass@k(大 k)覆盖率仍高、token 熵未坍缩;数据量从极小(LIMO 817 条、DeepSeek-R1 cold start "数千条")到百万级(Tülu 3 939,344 条、Nemotron 33M+ 条)皆可,关键看模型规模、基座强度与目标(对话 vs 推理)。

## Key Findings

1. **两种 RL 范式对 SFT 起点的要求本质不同。** 传统 RLHF(InstructGPT 式 PPO)中,SFT 模型既是 RL 的初始策略,又是 KL 惩罚的参考分布,因此 SFT 必须先建立可用的指令遵循与风格基线;RLVR/GRPO(DeepSeek-R1、Tülu 3、Qwen3)中,SFT 退化为"cold start",主要解决可读性、格式、语言一致性,刻意不追求性能饱和。

2. **业界主流做法是"轻量 cold start":** DeepSeek-R1 用"数千条"长 CoT;Qwen3 明确说 cold start 阶段应"尽量减少样本数和训练步数",目的是"植入基础推理范式而不限制模型潜力";Kimi k1.5 做"lightweight SFT"warmup。这些都印证 SFT 应欠拟合而非收敛。

3. **"跳过 SFT 直接 RL"(zero RL / R1-Zero)在特定条件下可行:** 前提是基座模型已有较强的指令遵循与自我反思能力(Qwen2.5 系列尤其明显)。SimpleRL-Zoo 在 10 个基座上验证,并发现对 Qwen2.5-Math-7B 而言,做 20 步以上 SFT 反而显著降低 RL 潜力,直接 zero RL 效果最好。

4. **SFT 过度 → 熵坍缩 → RL 收益递减,有理论与实证支撑。** Cui et al.(2025)给出 R=−a·exp(H)+b 的熵-性能关系,熵耗尽即性能见顶;多篇论文(Weight Ensembling、Quagmires、Getting LLMs Ready for RL)显示 SFT 越久 pass@1 越高但 pass@k 越低,RL 起点应选"多样性峰值"而非"loss 最低"的 checkpoint。

5. **SFT 不足 → RL 不稳定:** 基座不遵循指令时 RL 常因格式混乱、reward hacking 而失败;"SFT Memorizes, RL Generalizes"明确指出 SFT 的核心作用是稳定输出格式,使后续 RL 能取得增益。

6. **NVIDIA Llama-Nemotron 给出最清晰的"SFT 注入能力、RL 突破天花板"证据:** SFT 蒸馏只能逼近教师 DeepSeek-R1,论文原文称"Using supervised fine-tuning, LN-Ultra can approach the performance of DeepSeek-R1 but not exceed it … large-scale reinforcement learning is an essential approach";LN-Ultra 在 GPQA-Diamond 上 SFT 后为 66.4%,RL 后达 76.0%,超越 DeepSeek-R1 的 71.5%(对比 Llama-3.1-405B-Instruct 仅 43.4%);且他们刻意从较早(欠拟合)的 SFT checkpoint 启动 RL。

## Details

### A. 经验法则与判断标准

**SFT 该收敛还是该欠拟合?** 主流答案是"为 RL 保留探索空间,刻意停早"。Qwen3 技术报告(arXiv 2505.09388)在 cold start 阶段写明:"目标是在模型中植入基础推理范式,而不过度强调即时推理性能……这确保模型潜力不被限制,为后续 RL 留出灵活性……应尽量减少训练样本数与训练步数。" 这是对"SFT 不应饱和"最权威的官方表述。值得注意的是,即便在传统 RLHF 中,InstructGPT 原文也提到 SFT 模型在 1 个 epoch 后即在验证损失上过拟合,但继续训练(最终用 16 epochs)仍提升 RM 分与人类偏好——说明"过拟合"本身在 SFT 阶段并非禁忌,关键是它如何影响下游。

**两种关于 SFT 角色的观点并存:**
- **SFT 作为 format/cold start(主流推理模型观点):** DeepSeek-R1、Qwen3、Kimi、SFT-Memorizes-RL-Generalizes(arXiv 2501.17161,ICML 2025)都强调 SFT 主要稳定格式与推理范式。该论文结论:"尽管 RL 泛化更强,SFT 对有效 RL 训练仍是必要的:SFT 稳定模型的输出格式,使后续 RL 得以实现增益。"
- **SFT 作为能力注入(distillation 观点):** Llama-Nemotron、LIMO 强调高质量 CoT 蒸馏能直接注入强推理能力。LIMO(arXiv 2502.03387)原文:"With merely 817 curated training samples, LIMO achieves 57.1% accuracy on AIME and 94.8% on MATH, improving from previous SFT-based models' 6.5% and 59.2% respectively, while only using 1% of the training data required by previous approaches"(后续修订版更新为 AIME24 63.3%、MATH500 95.6%)。

**"SFT 做太多损害 RL"的证据链(强):**
- Cui et al.《The Entropy Mechanism of RL for Reasoning》(arXiv 2505.22617)证明熵坍缩使性能见顶(R=−a·exp(H)+b,理论上限为 H=0 时 R=−a+b),提出 Clip-Cov/KL-Cov 两种通过限制高协方差 token 更新来控熵的方法。
- 《Quagmires in SFT-RL Post-Training》(arXiv 2510.01624,>100 万 GPU 小时实验)发现:更高 SFT 分数的模型,RL 后有时反而显著差于直接对基座做 RL;pass@1 对 RL 结果的预测力弱(R²≈0.4),而 held-out 泛化损失和 pass@large-k 预测力强(Llama3-8B 上 Spearman R² 提升约 0.5、接近 0.94)。
- 《Getting Your LLMs Ready for RL with Lightweight SFT》(OpenReview)指出最高评测分 SFT checkpoint 因"分布性遗忘"(在传统过拟合之前就过度偏离基座分布)而非最优 RL 起点,熵与 self-BLEU 是更可靠的早停信号,提出 AESL(Adaptive Early-Stop Loss)。
- 《Weight Ensembling Improves Reasoning》(arXiv 2504.10478):SFT 越久 pass@k 越早衰减(diversity collapse),建议 WiSE-FT 把早期 checkpoint 与晚期 checkpoint 做权重平均,同时保住 pass@1 与 pass@k。

**RL 是否真能超越基座?** Yue et al.《Does RL Really Incentivize Reasoning Beyond the Base Model?》(arXiv 2504.13837,NeurIPS 2025)发现 RLVR 提升 pass@1 但在大 k 时基座 pass@k 反而更高,且 RL 训练越久推理边界越窄;蒸馏才能引入新推理模式。ProRL(arXiv 2505.24864)则反驳,认为是 RL 步数太少、领域太窄所致,长时间 RL 能扩展边界。这是当前最大的开放争论。

### B. 具体工程/量化指标

**数据量级(差异极大,取决于范式与模型):**
- 传统 RLHF SFT:InstructGPT 原文"The SFT dataset contains about 13k training prompts (from the API and labeler-written)",由约 40 名标注员手写;现代做法 10k–100k 条,常混合人写+蒸馏。
- 推理 cold start:DeepSeek-R1 "数千条"长 CoT;Kimi k1.5 "小而高质量"的 long-CoT warmup;LIMO 817 条、s1 约 1000 条。
- 大规模 SFT 注入:Tülu 3 官方原文"In total, we collect 939,344 prompts … of which 57% are sourced from public resources and 43% are synthetically generated in house",OLMo 2 最终 SFT mix 939,104 条;NVIDIA 将 Llama-Nemotron-Post-Training Dataset 描述为"33M+ filtered samples for math, code, reasoning, and general chat"(其中 OpenCodeReasoning 子集含 735K 条 Python 样本,源自 28K 道题)。

**Epoch / 学习率(经验区间):**
- 通用 SFT:1–3 epoch 常见,Tülu 3 用 2 epochs(8B 用 lr 5e-6、70B 用 2e-6,sum loss,batch 128,seq 4096),并发现"训练更久没有进一步收益"。
- 推理 SFT 可更激进:LIMO 用 15 epochs(lr 5e-6 cosine,无 warmup,batch 64)。Llama-Nemotron(arXiv 2505.00949):LN-Nano 第一阶段 4 epochs(lr 1e-4,seq 32k,batch 256),LN-Super 1 epoch(lr 5e-6,小规模实验显示 3–4 epoch 更好但受算力限制),LN-Ultra warmup 到 1e-5 后 cosine 衰减到 1e-6、第一个 epoch 后曾遇梯度爆炸需重启优化器状态。
- 《Data Repetition Beats Data Scaling》(arXiv 2602.11149):固定更新预算下,在小数据集上多 epoch 优于大数据集单 epoch;Olmo3-7B 在 400 样本上训 128 epoch 比 51200 样本训 1 epoch 在 AIME 高 12–26 个百分点。

**判断 SFT 是否"准备好"的指标(综合多篇):**
- 格式遵循率 / 结构化 CoT 能力(R1 的 |special_token|<reasoning_process>|special_token|<summary> 模式)。
- pass@large-k 覆盖率(强 RL 结果预测指标)与生成多样性(熵、self-BLEU)未坍缩。
- held-out 推理样本的泛化损失。
- 不要只看 pass@1 / 评测最高分。

**checkpoint 选择策略:** 不选 loss 最低或评测最高,而选多样性峰值/pass@k 仍高、刚学会格式但未过拟合的点;Llama-Nemotron 明确"虽有更高分的 SFT checkpoint,但从更早的 checkpoint 启动 RL 以改善最终 RL 结果"。部分工作建议把目标采样温度下的熵控制在约 0.3。

### C. 失败模式与权衡

- **SFT 不足:** 格式混乱、不遵循指令、reward hacking、RL 不收敛。SFT-Memorizes-RL-Generalizes 指出基座不遵循指令时 RL 常直接失败。
- **SFT 过度:** 熵坍缩、pass@k/多样性下降、探索空间受限、RL 增益递减甚至负收益。从 SFT 到 RL 的漏洞检测论文(arXiv 2602.14012)显示 epoch=5 的 SFT checkpoint 经 GRPO 后 P-Pass@1 相对 epoch=3 下降 9.1%,P-Pass@8 急剧下降(表明探索被限制)。多模态的《SFT or RL?》(arXiv 2504.11468)甚至发现对已对齐模型 SFT 后再 GRPO 平均掉 12.7%。
- **跳过 SFT(zero RL):** 仅当基座已具指令遵循/反思能力时可行。SimpleRL-Zoo(arXiv 2503.18892)原文:"models with more than 20 steps showed substantially reduced RL potential. Therefore, we conclude that RL training produces the best performance gain when applied directly to the base model without any supervised fine-tuning, i.e., the zero RL training",并指出 Qwen2.5 基座本身"already exhibit strong instruction-following and self-reflection abilities"。Open-Reasoner-Zero 用纯 PPO(GAE λ=1、γ=1、无 KL)在 Qwen2.5-32B 上以 1/10 步数超越 DeepSeek-R1-Zero-Qwen-32B。
- **中间步骤:** rejection sampling 几乎是标配(Llama 2/3 用约 4 轮);DeepSeek-R1 在 RL 收敛后用 rejection sampling 生成约 600k 推理 + 200k 非推理 SFT 数据再训一轮;Tülu 3/OLMo 在 SFT 与 RLVR 间插入 DPO,且官方称"从 DPO 启动 RLVR 比从 SFT 启动得到更高的最终模型"。

### D. 不同模型类型的差异

- **通用对话模型 vs 推理模型:** 对话模型(Llama 3)走 SFT→(rejection sampling)→DPO,几乎不用在线 RL;Meta 明确称 PPO/RLHF 比这些方法更不稳定、更难 scale,Llama 3.1 最大模型用 lr 1e-5、8.5K–9K 步训 SFT。推理模型(R1、Qwen3、Nemotron、Kimi)走 cold-start SFT→大规模 RLVR/GRPO,SFT 要求注入长 CoT 范式。
- **模型规模:** 规模越大,zero RL 越可行(大基座本身能力更强);Llama-Nemotron 仅对最大的 LN-Ultra 做推理 RL,因为 SFT 蒸馏对大模型设了天花板,需 RL 才能超越教师(GRPO 用 rollout prompt size 72、每 prompt 16 个回复、global batch 576,约 140k H100 小时,并采用按难度递进的课程化训练)。VLAA-Thinking 发现 7B 与更小模型 SFT 损害 RL 的幅度相近(规模不能免疫)。LIMR(arXiv 2502.11886)发现 LIMO/s1 的小数据 SFT 法在 32B 有效但在 7B 显著欠佳,7B 上 RL 更有效。

## Recommendations

**分阶段决策(从基座质量出发):**

1. **先评估基座。** 若基座(如 Qwen2.5 系列)已能遵循指令、产出格式化 CoT、有自我反思,优先尝试 **zero RL / 极轻 cold start**(<数千条或直接 RL)。基准:在目标域 pass@k(k=8–64)覆盖率是否已较高、是否能自发产生结构化推理。
2. **若基座弱或需要特定输出格式/语言一致性,做轻量 cold start SFT。** 目标是格式与范式,不是分数。规模参考:推理任务数千~数万条高质量长 CoT;通用对话 10k–100k 条。epoch 1–3(小数据可多 epoch),监控验证集而非追求 loss 最低。
3. **用正确指标选 checkpoint。** 选 pass@large-k 高、token 熵/self-BLEU 未坍缩、held-out 泛化损失低的点,而非评测最高分。若条件允许,做 WiSE-FT(早晚 checkpoint 权重平均)同时保住 pass@1 与 pass@k;或像 Nemotron 一样刻意从较早 checkpoint 启动 RL。
4. **在 SFT 与 RL 间考虑中间步骤。** 通用对话:加 DPO/rejection sampling(从 DPO 启动 RLVR 经验上更优)。推理:RL 收敛后回头做 rejection-sampling SFT 再 RL(R1 式多阶段)。
5. **RL 阶段防熵坍缩。** 监控 policy entropy;若早期急剧下降,用 Clip-Cov/KL-Cov、提高 clip 上界(DAPO)、或调采样温度。保持 KL 参考(传统 RLHF)以防 reward hacking 与遗忘。

**触发调整的阈值/基准:**
- 若 RL 后 pass@1 升但 pass@k(大 k)显著降 → SFT 起点过度,回退到更早 checkpoint 或降低 SFT 强度。
- 若 RL 训练早期 reward 不涨、格式频繁出错 → SFT 不足,补 cold start。
- 若 RL 增益在数百步内饱和且熵接近 0 → 熵坍缩,引入熵管理。
- 若 SFT 已能逼近但无法超越目标教师 → 必须上 RL(Nemotron 经验:SFT 66.4 → RL 76.0,超越教师 71.5)。

## Caveats

- **"RL 能否超越基座"仍是开放争论。** Yue et al. 与 ProRL 结论相反,前者认为当前 RLVR 只是把基座已有能力的采样效率提高、并未拓展边界,后者认为长时间 RL 可拓展。读者应将"RL 增益"理解为情境相关,而非普适。
- **大量证据来自数学/代码等可验证域和中小模型(≤12–32B)。** 在创意写作、多轮 agent、超大模型上的 SFT→RL 衔接规律可能不同;pass@k 在长序列上估计成本高。
- **部分二手来源(博客/Medium)对 R1 等的复述可能简化或有误**,本报告关键数字尽量回到 arXiv 原文与官方技术报告/博客;Llama-Nemotron 的精确数字来自其 arXiv 全文。
- **多篇关键论文为 2025–2026 年的预印本或会议投稿(部分 ICLR 2026 在投/撤稿)**,结论可能随同行评议更新;熵控制、最优 SFT-RL transition 等方向仍在快速演进。
- **跨团队工程现实:** SFT 与 RL 常由不同团队负责、各自优化指标,导致"高 SFT 分≠好 RL 结果"的脱节,这是 Quagmires 论文强调的实践陷阱。
- **关于"SFT 必须够深"的反直觉证据:** Llama-Nemotron 团队强调的是 SFT *数据*要大规模高质量(代码数据从 25k 扩到 736k 仍未饱和),但 RL *起点* checkpoint 反而要选更早/更欠拟合的——"数据要多"与"训练步数要克制"并不矛盾,需分开理解。