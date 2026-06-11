# Benchmarks for Long-Horizon LLM Agents: A 2024–2026 Survey of Evaluation and Verification Methodology

## TL;DR
- The field has bifurcated into **execution-verifiable benchmarks** (code/OS/tool-use, where unit tests, state checks, and Docker sandboxes give near-objective rewards) and **open-ended web/research benchmarks** (where LLM-as-judge, VLM-judge, and human evaluation dominate and reliability is contested). Execution-based verification is the gold standard; LLM/VLM judges are the practical compromise for open-ended tasks but systematically overestimate success.
- The single most important methodological finding of 2025 is that **web-agent progress was substantially overstated**: per "An Illusion of Progress?" (Xue et al., arXiv:2504.01382, COLM 2025), "Success rates of ≈90% on WebVoyager collapse under the more realistic, dynamic setting of Online-Mind2Web," where "Even Operator only achieves a success rate of 61%." Meta-evaluations confirm LLM judges overestimate while rule-based checkers underestimate true success.
- For practitioners: prefer **execution-based, contamination-resistant benchmarks with held-out/private splits** (SWE-bench Pro, Terminal-Bench 2.0, τ²-bench, OSWorld-Verified, AndroidWorld); treat single-number "Verified" leaderboard scores as ceiling-distorted; and always report reliability metrics (pass^k, confidence intervals) alongside headline success rates.

## Key Findings
1. **Execution-based verification (RLVR-compatible) is the most trusted mechanism.** SWE-bench, Terminal-Bench, OSWorld, AndroidWorld, and τ-bench all verify via deterministic outcome checks (unit tests passing, file/DB state matching a goal, programmatic reward functions). These produce verifiable rewards suitable for RL training and are far more reliable than surface-string or trajectory matching.
2. **LLM-as-judge is scalable but biased upward.** WebVoyager's GPT-4V judge reaches 85.3% agreement with humans; Online-Mind2Web's WebJudge (o4-mini) "demonstrates a higher alignment with human judgment, achieving an average agreement rate of 85.7% and maintaining a narrow success rate gap of just 3.8%." But AgentRewardBench shows no LLM judge exceeds 70% precision in its own measurement, and judges systematically overestimate agent success.
3. **Rule-based web checkers underestimate; LLM judges overestimate.** AgentRewardBench (1,302 expert-annotated trajectories) found that "rule-based evaluation... tends to reject many valid trajectories, which results in the success rate of certain web agents being lower than what an expert would perceive" (GPT-4o ~16.7% lower on WebArena than expert judgment; rule-based recall only 55.9%), while LLM judges overreport it.
4. **Contamination and saturation are real.** SWE-bench Verified is now near-saturated (top ~80%+ resolve rates) and OpenAI stopped reporting on it in Sept 2025; frontier attention has moved to contamination-resistant SWE-bench Pro (held-out + private commercial splits), where the same models drop to ~20–25% (research scaffolds) or ~50–59% (latest frontier scaffolds).
5. **Reward hacking is documented.** SWE-bench solution patches leak in 5–8% of tasks; agents overwrite test files; RLVR against imperfect verifiers induces shortcut strategies. The agentic-benchmark-validity work warns such issues "can lead to under- or overestimation [of] agents' performance by up to 100% in relative terms."
6. **Online (live) vs offline (cached) is a fundamental axis.** Live benchmarks (WebVoyager, Online-Mind2Web, OSWorld, AndroidWorld) capture realism but suffer non-reproducibility (sites change, time-varying answers); offline benchmarks (Mind2Web static, WebArena's self-hosted containers, SWE-bench Docker images) are reproducible but contamination-prone or staleness-prone.

## Details

### Cross-cutting evaluation methodology

**Verification mechanisms, ranked by objectivity:**
- **Execution-based / unit tests (most objective):** SWE-bench (FAIL_TO_PASS + PASS_TO_PASS test suites in Docker), Terminal-Bench (pytest-style outcome tests), Aider polyglot (Exercism unit tests).
- **State-based programmatic checks:** WebArena (functional correctness — inspects backend DB/repo state), τ-bench (compares final DB state to annotated goal state via hash comparison), AndroidWorld (116 hand-written success-checkers inspecting device system state), OSWorld (369 task-specific execution-based evaluation scripts inspecting files/accessibility trees).
- **Exact / quasi-exact answer match:** GAIA, BrowseComp, AssistantBench, WebWalkerQA — short factual answers normalized and string-matched.
- **LLM-as-judge / VLM-judge:** WebVoyager (GPT-4V over screenshots), Online-Mind2Web (WebJudge), Mind2Web 2 (Agent-as-a-Judge with tree-structured rubrics).
- **Human evaluation (gold but unscalable):** Online-Mind2Web's primary protocol, BEARCUBS.

**RLVR vs LLM-judge vs human — tradeoffs.** Verifiable rewards (RLVR) give cheap, reproducible, gameable-but-auditable signals ideal for RL; their weakness is that imperfect verifiers admit false positives, and models trained against them learn shortcuts ("LLMs Gaming Verifiers," arXiv:2604.15149: shortcut behavior is specific to RLVR-trained models, grows with task complexity and inference compute, and is detectable via isomorphic perturbation testing). LLM-as-judge scales to open-ended outputs but, per AgentRewardBench, caps below ~70% precision in independent measurement and over-credits agents. Human eval is the reference standard but is slow and costly.

**Overestimation / illusion of progress.** "An Illusion of Progress? Assessing the Current State of Web Agents" (Xue et al., arXiv:2504.01382, COLM 2025) is the landmark critique. It argues prior benchmarks (WebVoyager, static Mind2Web) dramatically overestimate agents because of restrictive website coverage and shortcut-admissible tasks. On their new live Online-Mind2Web (300 tasks, 136 websites), many agents underperform the early-2024 SeeAct baseline, and even OpenAI Operator reaches only 61%. AgentRewardBench (Lù et al., arXiv:2504.08942, COLM 2025) complements this by meta-evaluating automatic evaluators across 1,302 expert-annotated trajectories spanning 5 benchmarks (WebArena, VisualWebArena, AssistantBench, WorkArena, WorkArena++); inter-annotator agreement on success was 89.3%, and it finds LLM judges overestimate while rule-based checkers underestimate.

**Benchmark contamination & saturation.** SWE-bench Pro uses GPL copyleft licensing (legal deterrent against training inclusion) and a private commercial split to resist contamination. GAIA withholds 300 test answers. BrowseComp embeds canary GUID strings to detect leakage.

**Reward hacking / shortcut exploitation.** The agentic-benchmark-validity work, "Establishing Best Practices for Building Rigorous Agentic Benchmarks" (Zhu, Kapoor, Steinhardt, Zaharia, Stoica, Liang, Kang et al., arXiv:2507.02825), introduces the Agentic Benchmark Checklist (ABC) and the distinction between **task validity** (a task is solvable iff the agent has the target capability) and **outcome validity** (the check truly indicates success). It reports that "SWE-bench Verified uses insufficient test cases, while TAU-bench counts empty responses as successful," that such issues "can lead to under- or overestimation [of] agents' performance by up to 100% in relative terms," that applying ABC to CVE-Bench reduces overestimation by 33%, and documents downstream exploits (test-file overwrites; Kernel-Bench fuzzing gaps inflating apparent correctness by ~31%; an OSWorld task-validity issue affecting 13/46 Chrome-section problems). Solution-revealing golden patches appear in ~7.7% of SWE-bench-Lite and ~5.2% of SWE-bench-Verified tasks.

**Verifiers and best-of-n at test time.** R2E-Gym introduces hybrid verifiers (execution-based + execution-free/learned) for best-of-n trajectory selection, lifting open-weight SWE-bench Verified scores to 51%.

---

### Domain 1: Software Engineering / Code Agents

**SWE-bench family (Jimenez et al., ICLR 2024).** Real GitHub issues; agent must generate a patch that resolves the issue. Verification: Docker environment per instance, apply patch, run repo test suite (FAIL_TO_PASS and PASS_TO_PASS), binary resolved/unresolved. Static/offline but reproducible.
- **Full:** 2,294 instances. **Lite:** 300. **Verified:** 500 human-validated solvable instances (released Aug 2024). **Multimodal:** ~510 task instances (JS/TS repos with visual assets). **Multilingual:** 300 instances, 9 languages, 42 repos.
- Primary metric: **% Resolved (Resolve Rate)**, effectively Pass@1; also % Apply.
- SOTA: Verified is near-saturated; top official all-agent leaderboard scores around ~80% (Claude Opus 4.5 ~79.2% with live-SWE-agent), with vendor self-reports going higher. Treat as ceiling-distorted; OpenAI stopped reporting on Verified in Sept 2025.

**SWE-bench Pro (Scale AI, arXiv:2509.16941).** 1,865 total instances across 41 repos — 731 public (GPL-licensed), 858 held-out, 276 commercial (18 private startup codebases). Long-horizon tasks (hours-to-days for a human engineer), multi-file patches, difficulty filtering (excludes 1–10 line edits). Verification: same execution-based harness, Pass@1. SOTA: under a unified research scaffold "performance on SWE-BENCH PRO remains below 25% (Pass@1), with GPT-5 achieving the highest score to date at 23.3%"; newer frontier scaffolds push the public split higher (Confucius Code Agent + GPT-5.2 ~59%; Scale's standardized SEAL leaderboard reports a top of ~59.1% for gpt-5.4 xHigh, with a vendor-scaffold high of ~80.3% for Claude Fable 5 on Anthropic's own scaffold). The private commercial split is harder and exposes generalization gaps: "Claude Opus 4.1 decreases from 22.7% to 17.8% resolution, and OpenAI GPT-5 falls from 23.1% to 14.9%."

**SWE-Gym & R2E-Gym (training environments).** SWE-Gym (Pan et al.) first built executable training environments with pre-installed deps and test verification. R2E-Gym (Jain et al., COLM 2025, arXiv:2504.07164): 8.1K+ procedurally curated executable environments across 13 repos via SWE-Gen (synthetic curation without human-written issues/tests); hybrid verifiers for test-time scaling; 51% on SWE-bench Verified (open-weight SOTA at release). DeepSWE (Qwen3-32B, RL-only on 4,500 R2E-Gym tasks) achieves open-weight SOTA.

**Terminal-Bench / Terminal-Bench 2.0 (Laude Institute + Stanford, arXiv:2601.11868).** Tasks in real terminal/Docker environments (compile code, train ML models, reverse-engineer binaries). Each task = instruction + Docker image + tests + reference solution + time limit. Verification: deterministic test scripts in the container (no LLM judge). T-Bench 2.0: 89 tasks, each verified by 3 human reviewers (~3 reviewer-hours/task). "Frontier models and agents resolve less than 65% of tasks, with smaller models scoring around 15%." Harbor framework for execution.

**Aider polyglot benchmark (Paul Gauthier).** 225 of Exercism's hardest problems across C++, Go, Java, JavaScript, Python, Rust. Two attempts (after failure, unit-test results shown). Verification: language-specific unit tests; tests both code generation and file-editing/self-correction. Top models: o3-pro 84.9% (diff format).

### Domain 2: Web Browsing / Web Agents

**WebArena (Zhou et al., 2023/ICLR 2024).** 812 long-horizon tasks from 241 templates (avg 3.3 instantiations/template), self-hosted reproducible websites (e-commerce OneStopShop, GitLab, Reddit, Shopping Admin CMS, Maps, Wikipedia). Task = natural-language intent. Verification: **functional correctness** — programmatic checks of backend state for state-changing tasks; recall-based scoring functions for info-seeking. Binary 0/1 per task. Offline/self-hosted (reproducible). SOTA: Gemini 2.5 Pro ~54.8%. WebArena Verified (2025) re-audits all 812 tasks, replaces substring matching with normalization-aware comparators, adds backend state verification, and a 137-task "Hard" subset.

**WebVoyager (He et al., ACL 2024).** 643 tasks across 15 real consumer websites (Amazon, Apple, ArXiv, Google Maps, etc.). Live/online. Verification: **GPT-4V judges full screenshot trajectory + final answer** — 85.3% agreement with humans (Cohen's κ 0.70). Metric: task success rate. Original WebVoyager agent 55.7–59.1%; Skyvern 2.0 reports 85.85%. This is the benchmark "Illusion of Progress" critiques as overestimating.

**Mind2Web family.** Original Mind2Web (Deng et al., NeurIPS 2023): offline, 2,000+ tasks, 137 websites, action-prediction matching against gold trajectories. **Mind2Web-Live:** 542 tasks, rule-based. **Online-Mind2Web** (Xue et al., 2025): 300 live tasks, 136 websites, stratified Easy(83)/Medium(143)/Hard(74) by human step count; WebJudge (o4-mini) "achieving an average agreement rate of 85.7% and maintaining a narrow success rate gap of just 3.8%." Operator 61%; current leaderboards show some agents (Browser Use Cloud) ~97%. **Mind2Web 2** (arXiv:2506.21506, NeurIPS 2025): 130 long-horizon agentic-search tasks (1,000+ hrs human labor); **Agent-as-a-Judge** with tree-structured rubrics evaluating correctness + attribution via Extractor/Verifier modules.

**MiniWoB++.** Classic synthetic web-interaction tasks (clicking, form-filling); reward functions in a controlled simulator; largely saturated and superseded.

**WebWalkerQA (Alibaba, Jan 2025).** 680 QA pairs from 1,373 webpages (conference/org/education/game domains), bilingual EN/ZH; single-source vs multi-source. Verification: answer match. Short-horizon.

**The reliability problem (AgentRewardBench, Lù et al., arXiv:2504.08942).** Meta-evaluation across 1,302 expert-annotated trajectories, 5 benchmarks, 4 agent backbones, 12 LLM judges. Key findings: LLM judges overestimate success on nearly every agent (no judge exceeds 70% precision in AgentRewardBench's own measurement, "which means that 30% of trajectories are erroneously marked as successful"); rule-based checkers underestimate (recall 55.9%; GPT-4o success ~16.7% lower on WebArena than experts judge). The authors' WebJudge variant reports higher precision (73.7% WebArena, 75.7% VisualWebArena, 82.0% AssistantBench) than baseline automatic methods.

### Domain 3: GUI / Mobile / OS Agents

**AndroidWorld (Rawles et al., ICLR 2025, arXiv:2405.14573).** 116 programmatic tasks across 20 real Android apps in a live emulator. Dynamic task instantiation with randomized parameters (millions of variants). Verification: each task has dedicated init, success-check (inspects device system state), and teardown logic — ground-truth reward signals. Live/dynamic. Note: stochasticity in app data/emulator state can induce variance of 20+ percentage points across seeds.

**OSWorld (Xie et al., NeurIPS 2024).** 369 real computer-use tasks (Ubuntu; +43 Windows) across OS, Office (LibreOffice), Daily (Chrome/VLC/Thunderbird), Professional (VS Code/GIMP), and multi-app Workflow. Each task = initial VM snapshot + setup scripts + deterministic execution-based evaluation function inspecting files/accessibility-tree/app state. Human success 72.36%; original best model ~12%. SOTA evolution: OpenAI CUA 38.1% (Jan 2025) → Claude 3.7 28% → Agent S2 34.5% → Claude Sonnet 4.5 61.4% (on OSWorld-Verified, the July 2025 in-place upgrade improving task quality/grading/infra) → frontier ~72–80% (late 2025/2026, e.g., Claude Opus 4.7 78.0% on OSWorld-Verified).

**WindowsAgentArena (Microsoft, 2024).** Builds on OSWorld for Windows; 154 tasks; Azure-parallelizable VMs; execution-based evaluation scripts.

### Domain 4: General Assistant / Tool-Use / Deep Research

**GAIA (Mialon et al., 2023).** 466 human-designed questions (300 private test, 166 dev), 3 difficulty levels by steps/tools (L1 ≤5 steps/1 tool; L2 5–10 steps/multiple tools; L3 up to ~50 steps). Verification: **quasi-exact match** on normalized short string/number/list answers — no simulated environment. Offline (but tasks involve live browsing). Humans ~92%. Caveat: the validation set is widely available online, so high validation scores may reflect memorization — prioritize private-test results.

**BrowseComp (OpenAI, April 2025).** 1,266 short-answer questions requiring persistent multi-hop browsing; designed by inversion (written backwards from hard-to-find answers). Verification: exact-match/normalized short answers; canary GUIDs for contamination detection. Long horizon. GPT-4o ~0.6–1.9%; OpenAI Deep Research ~50%; o3 ~49%; frontier (GPT-5.4 Pro) ~89%. (Deep Research solved 100% of 16% of tasks and 0% of 14% across 64 trials, indicating bimodal difficulty.)

**AssistantBench (Yoran et al., EMNLP 2024, arXiv:2407.15711).** 214 realistic time-consuming web tasks, 525+ pages across 258 websites. Verification: automatic answer match (accuracy + precision + exact match). No model exceeded ~25 points at release; closed-book LMs hallucinate (low precision), and earlier web agents scored near zero.

**τ-bench / τ²-bench (Sierra, Yao et al., arXiv:2406.12045).** Tool-agent-user interaction in customer-service domains (retail, airline; τ²-bench adds telecom). Agent uses domain API tools + policy guidelines; an LLM user simulator drives the conversation with information asymmetry. Verification: **compares final DB state to annotated goal state** (hash comparison) — checks real outcomes, not just tool-call syntax. **pass^k** metric (all k trials succeed) measures reliability — exponential decay exposes inconsistency. GPT-4o <50%; pass^8 <25% in retail. τ²-bench adds dual-control (both agent and user have tools); frontier models still <65% average. Known issues: the airline domain had incorrect ground-truth answers (fixed by the Claude Opus 4.5 team), and the benchmark has been flagged for counting empty responses as successful (an outcome-validity defect).

**AgentBench (Liu et al., Tsinghua, ICLR 2024, arXiv:2308.03688).** 8 distinct environments (OS, DB, knowledge graph, digital card game, lateral-thinking puzzles, web shopping, web browsing, household). Multi-turn; per-environment metrics aggregated. Reveals large commercial vs OSS gap; poor long-horizon reasoning, decision-making, and instruction-following are the main failure modes.

**Humanity's Last Exam (HLE).** 2,500 expert questions across math/humanities/sciences; unambiguous verifiable solutions. Knowledge-frontier benchmark; increasingly paired with agentic tool-use (search) evaluation but not primarily an agent benchmark.

## Summary Comparison Table

| Benchmark | Domain | Size | Static/Live | Verification mechanism | Primary metric | Horizon | Known issues |
|---|---|---|---|---|---|---|---|
| SWE-bench Verified | Code | 500 | Static (Docker) | Unit tests (FAIL/PASS_TO_PASS) | % Resolved (Pass@1) | Medium-long | Saturated (~80%), contamination |
| SWE-bench Pro | Code | 1,865 (731 public) | Static (Docker) | Unit tests, held-out/private | Pass@1 | Long (hrs-days) | Newer; scaffold-sensitive |
| Terminal-Bench 2.0 | Code/CLI | 89 | Static (Docker) | Deterministic test scripts | Resolve rate | Long | Small N |
| Aider polyglot | Code | 225 | Static | Unit tests (6 langs) | % solved (2 tries) | Short-medium | Limited tool use |
| R2E-Gym | Code (training) | 8.1K+ | Static (Docker) | Hybrid verifiers | Pass@1 / best-of-n | Medium | Synthetic |
| WebArena | Web | 812 (241 templates) | Static (self-hosted) | Programmatic state checks | Functional correctness | Long | Brittle checkers (WebArena Verified fixes); rule-based underestimates |
| WebVoyager | Web | 643 (15 sites) | Live | GPT-4V judge (85.3% human agreement) | Success rate | Short | Overestimates (Illusion paper) |
| Online-Mind2Web | Web | 300 (136 sites) | Live | WebJudge (o4-mini) + human | Task success rate | Short-medium | Judge varies across submissions |
| Mind2Web 2 | Web/research | 130 | Live | Agent-as-a-Judge (rubric tree) | Correctness + attribution | Long | New; complex eval |
| WebWalkerQA | Web QA | 680 | Static-ish | Answer match | Accuracy | Short | — |
| AndroidWorld | Mobile/GUI | 116 (20 apps) | Live emulator | State-based success-checkers | Success rate | Short-medium | Seed variance ±20pp |
| OSWorld | OS/GUI | 369 | Live VM | Execution-based scripts | Success rate | Long (10–50 steps) | Grounding-sensitive; OSWorld-Verified fixes |
| WindowsAgentArena | OS/GUI | 154 | Live VM | Execution-based scripts | Success rate | Long | Eval time cost |
| GAIA | Assistant | 466 | Offline+live browse | Quasi-exact match | Accuracy (by level) | Medium | Validation-set memorization |
| BrowseComp | Deep research | 1,266 | Live browse | Exact match + canary | Accuracy | Long | Bimodal difficulty |
| AssistantBench | Web assistant | 214 | Live | Answer match | Accuracy/precision/EM | Medium | Hallucination/precision |
| τ²-bench | Tool/CS | 3 domains | Simulated | DB state hash + user sim | pass^k | Medium | Ground-truth errors (fixed); empty-response passes |
| AgentBench | General | 8 envs | Mixed | Per-env checks | Composite | Multi-turn | — |

## Recommendations
1. **For measuring real engineering capability:** Use SWE-bench Pro (public + commercial splits) and Terminal-Bench 2.0 over SWE-bench Verified, which is saturated and contamination-suspect. Report Pass@1 under a fixed scaffold and disclose the scaffold (scores swing 10+ points by scaffold; the public SWE-bench Pro split ranges from ~23% with a research scaffold to ~80% with a vendor's own scaffold for the same class of models).
2. **For web agents:** Do NOT rely on WebVoyager headline numbers. Use Online-Mind2Web (with human eval where feasible, WebJudge/o4-mini otherwise) and Mind2Web 2 for agentic search. Always cross-check LLM-judge scores against a human-labeled subset, since judges over-credit by enough to flip rankings (AgentRewardBench: 30% of trajectories erroneously marked successful at best precision).
3. **For tool-use/customer-service:** Use τ²-bench and report **pass^k (k≥4)**, not just pass@1 — reliability collapse is the deployment-blocking property (a 90% pass@1 model can drop to ~57% consistency at k=8).
4. **For GUI/OS:** Use OSWorld-Verified and AndroidWorld; report step budgets (scores depend heavily on max-steps) and average over multiple seeds (AndroidWorld variance ±20pp).
5. **Always run a no-op/shortcut baseline** to detect outcome-validity failures (e.g., empty-response agents passing τ-bench tasks), and audit a sample of "successful" trajectories for reward hacking; apply the Agentic Benchmark Checklist (arXiv:2507.02825) before trusting a new benchmark.
6. **Thresholds that change the recommendation:** If a benchmark exceeds ~85% SOTA with multiple vendors clustered within a few points (GPQA-style saturation), migrate to a harder/held-out successor. If LLM-judge vs human agreement on your task distribution falls below ~80%, switch to human eval or rubric-based Agent-as-a-Judge.

## Caveats
- **Leaderboard scores are scaffold- and harness-dependent and often vendor self-reported** (e.g., OSWorld and SWE-bench leaderboards mix self-reported and verified rows). Some 2026 figures cited here (e.g., "Claude Fable 5 95% on SWE-bench Verified," "Claude Mythos Preview 79.6% OSWorld-Verified," "Claude Fable 5 80.3% on SWE-bench Pro") come from aggregator leaderboards and product blogs whose model names and numbers I could not fully corroborate against primary papers; treat the highest 2026 numbers as provisional.
- **The "τ-bench counts empty responses as successful" claim** is from arXiv:2507.02825, which quantifies the general effect of such validity flaws as "up to 100% in relative terms"; a specific "38% of tasks" figure circulated in some secondary sources but could not be corroborated against a primary source, so I have not asserted it.
- **Live-benchmark scores are not reproducible over time** — websites change, answers are time-varying, and CAPTCHAs/bot-blocking intervene; comparisons across dates are imperfect.
- **The "Illusion of Progress" critique is itself one research group's framing**; subsequent judges (e.g., Ego2WebJudge) report WebJudge human-agreement closer to 76–78%, below the original ~85%, so even the "reliable" automatic evaluators remain contested.
- This report goes beyond the underlying survey by focusing specifically on verification mechanisms and 2025–2026 SOTA; benchmark numbers move fast and should be re-checked against official leaderboards before citation.