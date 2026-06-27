---
title: CV
publish: true
---

# Jialong Liu

- Mobile: (+86) 155-5821-6267
- Email: Galleons@whu.edu.cn
- Github: github.com/Galleons2029

# Education

- B.S. in Safety Engineering, XiangTan University, 2020 - 2024
- M.S. in Data Science, Wuhan University, 2025 - 2027 (expected)


# Professional Experience

## Research Experience

### SRPO: Self-Reflective Policy Optimization for Long-Horizon Reasoning
- First Author, Accepted
- Wuhan University / Shanghai Jiao Tong University
- ICML 2026

- Proposed the **Self-Reflective Policy Optimization (SRPO)** framework, enabling models to analyze their own reasoning trajectories and summarize mistakes into **Reflection Patches**. Combined with a **Reset-with-Memory** mechanism, SRPO transforms sparse terminal rewards into dense token-level learning signals ($O(1) \to O(T)$), addressing the credit assignment problem in long-horizon tasks without external critics, reward models, or larger teacher models. Based on Qwen3-8B, SRPO achieved state-of-the-art performance using only **8% of the training FLOPs**, reaching 73.3% on AIME'24 and significantly outperforming prior methods on agent benchmarks including WebShop (64.7%), ALFWorld (76.8%), and SWE-Bench-Lite (31.2%).

### TTPO: Turn-Level Trajectory Optimization for Multi-Turn LLM Reasoning
- First Author, Under Review at Meta 3.5
- Wuhan University / Chinese Academy of Sciences / Shanghai Jiao Tong University
- EMNLP 2026

- Proposed **Turn-Level Trajectory Policy Optimization (TTPO)**, a critic-free reinforcement learning method that reparameterizes GRPO at the dialogue-turn granularity and treats each turn as a macro-action. TTPO introduces three key components: **Turn-Level Importance Ratio Surrogates** (PPO-style clipping), **Trajectory Pruning** (removing low-reward branches to suppress exponential growth), and **Cross-Turn Reward Propagation** (propagating terminal rewards back to earlier turns). Across six multi-turn dialogue tasks, TTPO reduced run-to-run variance by approximately 19–22 $U_{90\text{-}10}$ points, improved average performance by 9–12 percentage points, and was consistently preferred in human blind evaluations while maintaining high-percentile quality ($A_{90}$).

### HELM: Hierarchical Epistemic Learned Memory for Long-Horizon Agents
- First Author, Under Review
- Wuhan University
- EMNLP 2026

- Proposed the **Hierarchical Epistemic Learned Memory (HELM)** framework, exposing memory through event-driven interfaces and constructing a **Structured Hierarchical Nested Memory (SHNM)** consisting of episodic traces ($\mathcal{M}^{(1)}$), integrated recollections ($\mathcal{M}^{(2)}$), and thematic indices ($\mathcal{M}^{(3)}$). These memory layers are connected through provenance links and cognitive metadata (timestamps, sources, tool states). HELM further incorporates **Epistemic Governance** and a learned controller that decides READ/WRITE/CONSOLIDATE/PRUNE operations under budget constraints to prevent high-confidence misuse of weak evidence. Under matched budgets and scaffolding conditions, HELM significantly improved success rates on long-horizon benchmarks including GAIA (L2: $8.9 \to 12.8$; L3: $0.8 \to 1.7$) and WebArena ($19.5 \to 22.4$), while providing auditable memory provenance diagnostics.

### CHOP: Segment-Level On-Policy Distillation for Long-Horizon Agents
- First Author, Under Review
- Wuhan University / Shanghai Jiao Tong University / Xiaomi
- NeurIPS 2026

- Proposed **CHOP**, a hybrid distillation method for long-horizon agents. CHOP dynamically estimates the reliability of token-level supervision during training and switches to **segment-level Wasserstein distribution matching** when supervision quality deteriorates. This approach fundamentally mitigates the performance degradation commonly observed in traditional On-Policy Distillation methods on long-horizon reasoning and complex agent tasks. CHOP significantly outperformed existing OPD and Top-K LSM baselines on AIME 2024 long-form mathematical reasoning and SWE-bench Verified software engineering tasks, enabling more stable and efficient long-context capability transfer.

## Project Experience

### Bank-Copilot: AI-Powered Financial Management Platform

- Built long-horizon agents using the Harness architecture and deployed backend agent services based on a **data-driven** design pattern with Flink, enabling stable 24/7 operation.
- Constructed graph-based memory with Graphiti and FalkorDB, structuring key entities, accounting relationships, and business events into a knowledge graph while combining retrieval systems to form a dual-channel architecture of vector memory and graph memory.
- Developed a Unix-like execution environment exposing file system and CLI tool access, expanding agent action spaces while reducing dependence on context window length.
- Unified tool invocation, fault-tolerance mechanisms, and Langfuse observability into a single state graph, leveraging PostgreSQL Checkpointer for session-level rollback and disaster recovery.

### Cascade-RAG: Deep Research Assistant

- Adopted a Multi-Agent architecture for context management and complex task decomposition, ensuring consistency in long-horizon memory settings while reducing state drift.
- Designed a recursive multi-agent scheduling mechanism for dynamic task allocation and quality control, supporting generation of reports exceeding 10,000 words.
- Built a four-stage closed-loop workflow ("Plan-Build-Verify-Repair") using the LangGraph Deep Agent framework, enabling iterative refinement and feedback-driven self-improvement.
- Optimized tool-calling paths through progressive disclosure and caching mechanisms to reduce redundant queries, while improving execution stability via sub-agent isolation.

### SGLang Open Source Community Contributions

- Participated in fixing bugs within the SGLang Diffusion ecosystem, improving module stability.
- Tested compatibility and usability in agent scenarios, identifying issues related to training-inference consistency.
- Contributed to the development and testing of SGLang RL for multimodal multi-turn dialogue tasks, covering both training and inference pipelines.

## Industry Experience

### Agent Post-Training Intern
- Location: Shanghai
- Company: Xiaohongshu (RedNote)
- Duration: Jun 2026 – Present

- Participating in the **general-agent post-training** of Xiaohongshu's next-generation **dots** model, enhancing its agentic capabilities across multi-turn tool use, planning, and long-horizon interactive tasks.

### Inference Acceleration Intern
- Location: Haidian, Beijing
- Company: Xiaomi Corporation
- Duration: Mar 2026 – Jun 2026

- Contributed to the design and development of Xiaomi's inference acceleration framework, optimizing model efficiency and resource utilization for multiple internal AI products.
- Contributed to the open-source multimodal inference framework vLLM-Omni and adapted it for internal deployment, identifying and resolving bottlenecks across the inference pipeline.
- Conducted performance analysis of locally deployed foundation models and optimized bottlenecks using techniques such as CUDA Graphs and kernel fusion.

### Algorithm Engineer
- Location: Changsha, Hunan
- Company: Peking University Changsha Institute for Computing and Digital Economy
- Duration: Jun 2025 – Dec 2025

- Developed the intelligent assistant "Xiaosuan" for the HESI computing resource scheduling platform, integrating knowledge-base QA and operational guidance capabilities. Built upon a multi-agent routing architecture deeply integrated with platform tools, achieving over 90% QA accuracy and more than 80% long-horizon task completion rates.
- Implemented an Agentic Document Extraction solution for production-grade enterprise document parsing, overcoming limitations of conventional OCR systems on extreme formats and low-quality scans, improving parsing accuracy from approximately 80% to 95%.
- Enhanced the performance of locally deployed small models in real-world agent tasks using online policy distillation techniques.

### Algorithm Researcher
- Location: Changsha, Hunan
- Company: Kunlunyuan Artificial Intelligence Co., Ltd.
- Duration: Mar 2025 – Jun 2025

- Followed cutting-edge research in Reasoning-RAG and reproduced/evaluated retrieval-native models such as Research-o1, Research-R1, and R1-Researcher in large-scale enterprise document retrieval scenarios.
- Designed and implemented a RAG-based agent system supporting text, image, and table inputs with multi-turn interactions, enabling closed-loop retrieval and reasoning in enterprise knowledge environments.

### LLM Application Engineer
- Location: Changsha, Hunan
- Company: Yunyan Technology Co., Ltd.
- Duration: May 2024 – Mar 2025

- Served as a core developer of a graduate employment recommendation platform built with FastAPI, RabbitMQ, Qdrant, and MongoDB, supporting nearly 10 million registered users, over 100,000 daily active users, and more than 300,000 online job postings.
- Developed LLM agents to analyze students' academic backgrounds, grades, and internship experiences, generating structured feature labels and leveraging FFM-based feature crossing for personalized job recommendations.
- Utilized Apache Kafka and RabbitMQ for asynchronous feature generation and notification delivery, improving throughput and system robustness under high-concurrency workloads.

# Technical Skills

| Category | Skills |
|----------|----------|
| Programming Languages | Python, C, TypeScript, Java, Rust, LaTeX, Shell |
| AI Frameworks | PyTorch, LangGraph, Agno, Scikit-Learn, Keras |
| AI Infrastructure | vLLM, SGLang, VERL, Roll, SLIME |
| Infrastructure & Systems | Slurm, Distributed Training, Docker, Optimization, Git, CI/CD Pipelines |

# Honors & Awards

## Certifications

- MIT MicroMasters Program in Data Science, 2024.
- Rustling Open Source Operating System Training Camp Certificate, Spring 2023.

## Awards

- Outstanding Winner Nominee, Mathematical Contest in Modeling (MCM/ICM), 2023.
- First Prize, Provincial Division, China Undergraduate Mathematical Contest in Modeling, 2023.
- Second Prize, National Graduate FinTech Competition, 2025.

# Research Interests

My research focuses on algorithmic advances in the **post-training** stage of large language models, with a particular emphasis on **reinforcement learning** for intelligent agent systems.

I currently have four first-author RL papers accepted or under review. My work primarily investigates optimization techniques for multi-turn dialogue and interactive scenarios in long-horizon agent tasks, including topics such as **Memory, Multi-turn Conversation, Long-Horizon Tasks, and Continual Learning**.

In addition, I actively contribute to open-source communities, including projects such as vLLM-Omni, SGLang Diffusion, and SGLang RL. I am experienced in using vibe coding methodologies to manage and maintain large-scale production systems.