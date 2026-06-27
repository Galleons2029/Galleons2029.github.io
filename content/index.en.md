---
title: Jialong Liu
publish: true
lang: en-US
---

> [!info]- 🌐 Language / 语言
> **English (current)** · [中文](index)

Hi, I'm **Jialong Liu (柳佳龙)**, a M.S. student at Wuhan University (jointly trained with the SJTU BCMI Lab), advised by Prof. Zuchao Li. My research focuses on **LLM post-training**, with a particular interest in **reinforcement learning for long-horizon agent tasks**. I'm currently an Agent post-training intern at Xiaohongshu (RedNote).

📫 [Galleons@whu.edu.cn](mailto:Galleons@whu.edu.cn) · [GitHub](https://github.com/Galleons2029) · 📞 (+86) 155-5821-6267

![[profile.png]]

## Research Interests

I work on post-training algorithms for large language models, with a main thread on **optimizing LLMs in multi-turn / interactive settings within long-horizon agent tasks**. Keywords:

`Reinforcement Learning` · `Memory` · `Multi-turn Conversation` · `Long-Horizon Task` · `Continual Learning`

I currently have **4 first-author papers** accepted / under review, and actively contribute to open-source communities (vLLM-Omni, SGLang Diffusion, SGLang RL).

## Selected Research

- **SRPO: Self-Reflective Policy Optimization for Long-Horizon Reasoning** — *ICML 2026 (Accepted)*
  A self-reflective policy optimization framework that lets the model analyze its own reasoning trajectories and synthesize errors into a "Reflection Patch," paired with a Reset-with-Memory mechanism that turns sparse terminal signals into dense token-level learning signals. Built on Qwen3-8B, it reaches SOTA with only 8% of the training FLOPs (AIME'24 73.3%) and leads clearly on agent benchmarks such as WebShop, ALFWorld, and SWE-Bench-Lite.

- **TTPO: Turn-Level Trajectory Optimization for Multi-Turn LLM Reasoning** — *EMNLP 2026 (Under Review)*
  A critic-free RL method that reparameterizes GRPO at the dialogue-turn granularity. Across six multi-turn tasks it reduces run-to-run volatility by ~19–22 points and improves average performance by 9–12 percentage points.

- **HELM: Hierarchical Epistemic Learned Memory for Long-Horizon Agents** — *EMNLP 2026 (Under Review)*
  A hierarchical epistemic learned-memory framework that exposes memory as an event-driven interface, building a three-tier nested memory store coupled with epistemic governance and a learned controller. It significantly improves success rates on GAIA and WebArena and supports auditable recall-provenance diagnostics.

- **CHOP: Segment-Level On-Policy Distillation for Long-Horizon Agents** — *NeurIPS 2026 (Under Review)*
  A hybrid distillation method for long-horizon agents that dynamically assesses the reliability of token-level supervision and switches to segment-level Wasserstein distribution matching when reliability is low, fundamentally mitigating the degradation of conventional on-policy distillation.

## Engineering & Projects

- **Xiaohongshu (RedNote) · Agent Post-Training Intern** (Jun 2026 – present): Working on the general-agent post-training of the next-generation dots model, improving its agentic capabilities across multi-turn tool use, planning, and long-horizon interactive tasks.
- **Xiaomi · Inference-Acceleration Intern** (Mar 2026 – Jun 2026): Contributing to Xiaomi's inference-acceleration framework and the open-source omni-modal framework vLLM-Omni; optimizing in-house model deployment with CUDA Graphs, kernel fusion, and related techniques.
- **Bank-Copilot**: A long-horizon AI banking/finance agent built on a Harness architecture, combining Graphiti + FalkorDB into a dual "vector memory + graph memory" channel, running 24/7.
- **Cascade-RAG**: A multi-agent deep-research assistant built on LangGraph with a "plan–build–verify–repair" loop, reliably producing 10,000+ word reports.

## Tech Stack

- **Language**: Python · C · TypeScript · Java · Rust · Shell
- **AI Framework**: PyTorch · LangGraph · Agno · Scikit-Learn
- **AI Infra**: vLLM · SGLang · VERL · Roll · SLIME
- **Infrastructure**: Slurm · Distributed Training · Docker · Git · CI/CD

## Learning Notes

Notes from my self-study path in data science and machine learning, with a few classic open courses I recommend.

### Math — Linear Algebra

- MIT 18.06 Linear Algebra
- MIT 18.065 Matrix Methods in Data Analysis, Signal Processing, and Machine Learning
- *Introduction to Linear Algebra* — Gilbert Strang
- *Advanced Algebra* — Weisheng Qiu

### AI Infra Roadmap

- CS336: Large Language Models — Algorithms, Systems, and Applications

Further reading: [A Survival Guide to a PhD — Andrej Karpathy](https://cs.stanford.edu/people/karpathy/advice.html)

## Navigation

- [[cv|Full CV]]
- Tech blog: see the **blog** folder in the left sidebar
- Paper notes: see the **paper** folder in the left sidebar
