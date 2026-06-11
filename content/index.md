---
title: 柳佳龙 | Jialong Liu
publish: true
lang: zh-CN
---

> [!info]- 🌐 Language / 语言
> **中文（当前）** · [English](index.en)

你好，我是**柳佳龙 (Jialong Liu)**，武汉大学硕士研究生（武大–上交 BCMI 实验室联合培养），导师李祖超。研究方向为 **大模型后训练（LLM Post-Training）**，尤其关注 **强化学习在智能体长程任务（Long-Horizon Tasks）中的应用**。目前在小米科技担任推理加速实习生。

📫 [Galleons@whu.edu.cn](mailto:Galleons@whu.edu.cn) · [GitHub](https://github.com/Galleons2029) · 📞 (+86) 155-5821-6267


## 研究兴趣

我专注于大模型后训练阶段的算法研究，研究主线是 **智能体长程任务中 LLM 多轮对话 / 交互场景下的优化与提升**。关键词：

`Reinforcement Learning` · `Memory` · `Multi-turn Conversation` · `Long-Horizon Task` · `Continual Learning`

目前有 **4 篇一作论文** 中稿 / 在投，同时积极参与开源社区建设（vLLM-Omni、SGLang Diffusion、SGLang RL）。

## 代表性研究

- **SRPO: Self-Reflective Policy Optimization for Long-Horizon Reasoning** — *ICML 2026 (录用 / Accepted)*
  自反思策略优化框架，让模型分析自身推理轨迹并将错误综合为「反思补丁」，配合 Reset-with-Memory 将稀疏终端信号转化为密集 token 级信号。基于 Qwen3-8B 仅以 8% 训练 FLOPs 取得 SOTA（AIME'24 73.3%），并在 WebShop / ALFWorld / SWE-Bench-Lite 等智能体基准上显著领先。

- **TTPO: Turn-Level Trajectory Optimization for Multi-Turn LLM Reasoning** — *EMNLP 2026 (在投 / Under Review)*
  回合级轨迹优化，一种无需评论器的强化学习方法，将 GRPO 重参数化至对话回合粒度。在六个多轮任务上将运行间波动性降低约 19–22 点、平均性能提升 9–12 个百分点。

- **HELM: Hierarchical Epistemic Learned Memory for Long-Horizon Agents** — *EMNLP 2026 (在投 / Under Review)*
  层次化认知学习记忆框架，将记忆暴露为事件驱动接口，构建三层嵌套记忆存储并耦合认知治理与学习型控制器。显著提升 GAIA 与 WebArena 成功率，并支持可审计的回忆溯源诊断。

- **CHOP: Segment-Level On-Policy Distillation for Long-Horizon Agents** — *NeurIPS 2026 (在投 / Under Review)*
  面向长链路 Agent 的混合蒸馏方法，动态判断 token 级监督的可靠性，并在低可靠状态下切换为 segment-level Wasserstein 分布匹配，从根源缓解传统 OPD 的性能退化。

## 工程与项目

- **小米 · 推理加速实习生**（2026.3–至今）：参与小米推理加速框架设计开发，贡献全模态推理框架 vLLM-Omni，使用 CUDA Graphs、内核融合等技术优化内部大模型部署。
- **Bank-Copilot**：基于 Harness 架构的 AI 银行财务管理长程 Agent，结合 Graphiti + FalkorDB 构建「向量记忆 + 图谱记忆」双通道，支持 7×24 稳定运行。
- **Cascade-RAG**：Multi-Agent 深度研究助手，基于 LangGraph 搭建「规划–构建–验证–修复」闭环，稳定支持单篇 10,000+ 字长文报告输出。

## 技术栈

- **Language**: Python · C · TypeScript · Java · Rust · Shell
- **AI Framework**: PyTorch · LangGraph · Agno · Scikit-Learn
- **AI Infra**: vLLM · SGLang · VERL · Roll · SLIME
- **Infrastructure**: Slurm · Distributed Training · Docker · Git · CI/CD

## 学习笔记

这里记录我在数据科学与机器学习路上的自学笔记，并推荐一些经典的公开课程。

### Math — Linear Algebra

- MIT 18.06 Linear Algebra
- MIT 18.065 Matrix Methods in Data Analysis, Signal Processing, and Machine Learning
- 《Introduction to Linear Algebra》 — Gilbert Strang
- 《高等代数》 — 丘维声

### AI Infra Roadmap

- CS336: Large Language Models — Algorithms, Systems, and Applications

延伸阅读：[A Survival Guide to a PhD — Andrej Karpathy](https://cs.stanford.edu/people/karpathy/advice.html)

## 站内导航

- [[cv|完整简历 / Full CV]]
- 技术博客见左侧目录的 **blog** 文件夹
- 论文笔记见左侧目录的 **paper** 文件夹
