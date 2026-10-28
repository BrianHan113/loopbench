# Evaluating RAG Systems, Tool-Using Assistants, and Production LLM Applications

Notes compiled 2026-10-28. Verification legend: items marked **[V]** were read or confirmed from search/fetch output in this session. Items marked **[K]** are canonical papers/docs cited from prior knowledge at their stable arXiv/doc URLs. They were not re-fetched, so the report writer should treat exact numbers on [K] items as needing a spot check before publication.

---

## 1. Retrieval metrics and IR benchmarks (recall@k, precision@k, MRR, nDCG, hit rate, context precision/recall; BEIR, MTEB, BRIGHT)

### Takeaway
Retrieval is still evaluated with classic IR metrics: rank-unaware recall@k, precision@k and hit rate, plus rank-aware MRR and nDCG@k. These are cheap, deterministic and debuggable, and practitioners say to debug retrieval before generation. The 2024-2025 work has one main lesson: standard query-document relevance labels correlate only weakly with downstream RAG answer quality. That has pushed the field toward *utility-based* retrieval evaluation (eRAG) and claim-level context recall (RAGChecker), and toward reasoning-intensive benchmarks (BRIGHT) where MTEB-leading embedders collapse.

### Cited Findings
- Practitioner guidance (Husain & Shankar FAQ, revised Sept 2026) says to treat retrieval as a search problem, score it with IR metrics such as Recall@k, Precision@k and MRR, and build query-document pairs by generating questions from facts in your own corpus (a "reverse process"). Their order of work is to debug retrieval first, then generation. **[V]** — [Hamel Husain & Shreya Shankar, AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- The same FAQ notes that similarity measures (e.g., cosine) are appropriate for retrieval and search, unlike for open-ended generation. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- eRAG (Salemi & Zamani, SIGIR 2024): evaluating retrievers with query-document relevance labels "shows a small correlation with the RAG system's downstream performance." eRAG instead feeds each retrieved document *individually* to the RAG system's own LLM and uses that document's downstream task score as its relevance label. The labels are then aggregated with set- or rank-based metrics such as precision and nDCG. It improves Kendall's tau correlation with end-to-end performance by 0.168 to 0.494 over baselines and uses up to 50x less GPU memory than end-to-end evaluation. **[V]** — [arXiv 2404.13781](https://arxiv.org/abs/2404.13781v1); [PyPI erag](https://pypi.org/project/erag/)
- RAGChecker defines retriever metrics at the *claim* level. **Claim recall** is the share of ground-truth answer claims entailed by the retrieved chunks. **Context precision** is the share of the k retrieved chunks that contain at least one ground-truth claim. **[V]** — [RAGChecker, arXiv 2408.08067 (HTML)](https://arxiv.org/html/2408.08067v2)
- RAGChecker found that swapping BM25 for E5-Mistral improved overall precision, recall and F1 regardless of the generator. Higher retriever claim recall also *raised* the generators' noise sensitivity, so the retrieval and generation metrics trade off. **[V]** — [arXiv 2408.08067](https://arxiv.org/html/2408.08067v2)
- In 2026 practitioner tooling, the five standard RAG metrics are Answer Relevancy, Faithfulness, Contextual Relevancy, Contextual Precision and Contextual Recall (DeepEval/Confident AI). These are LLM-judged analogues of IR precision and recall. **[V]** — [Confident AI, LLM Agent Evaluation guide (2026)](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide)
- BEIR (Thakur et al., 2021): a heterogeneous zero-shot IR benchmark of 18 datasets, reported mainly as nDCG@10. BM25 proved a robust zero-shot baseline that many dense retrievers failed to beat out of domain. **[K]** — [arXiv 2104.08663](https://arxiv.org/abs/2104.08663)
- MTEB (Muennighoff et al., 2022) covers 8 embedding task types (retrieval, reranking, clustering, STS, etc.) and found no single embedding method dominated all tasks. MMTEB (2025) extends it to a massively multilingual set of 500+ tasks. **[K]** — [arXiv 2210.07316](https://arxiv.org/abs/2210.07316); [MMTEB arXiv 2502.13595](https://arxiv.org/abs/2502.13595)
- BRIGHT (Su et al., 2024) is a reasoning-intensive retrieval benchmark (StackExchange, coding, math). A model scoring about 59 nDCG@10 on MTEB retrieval scored about 18 nDCG@10 on BRIGHT. Adding LLM-generated reasoning to queries improved scores. **[K]** — [arXiv 2407.12883](https://arxiv.org/abs/2407.12883)
- BrowseComp-Plus (Aug 2025, NeurIPS 2025) fixes the corpus (about 100K documents with human-verified evidence and mined hard negatives) so the retriever's contribution to a deep-research agent can be isolated. GPT-5 scored 55.9% with BM25 and 70.1% with Qwen3-Embedding-8B, using fewer search calls. Search-R1 with BM25 scored 3.86%. Retriever quality therefore still drives agentic-search outcomes. **[V]** — [NeurIPS 2025 page](https://neurips.cc/virtual/2025/128011); [arXiv 2508.06600](https://arxiv.org/html/2508.06600v1); [alphaXiv benchmark page](https://www.alphaxiv.org/benchmarks/university-of-waterloo/browsecomp-plus)
- A 2026 production case study ("The Coverage Illusion") traces failures from pre-retrieval routing to post-retrieval cascades in a production RAG system. Retrieval-coverage metrics can look healthy while routing errors upstream cause failures. Only the title and abstract-level snippet were seen. **[V, abstract only]** — [arXiv 2605.27220](https://arxiv.org/pdf/2605.27220)

### Inferences
- The retrieval-metric problem has a direct analogue in coding agents: the "retrieval" step is code search and context gathering (finding the right files). Relevance labels such as "the file the human patch touched" are like the qrels that eRAG showed correlate weakly with downstream success. An eRAG-style *utility* measure (does supplying this file change the pass rate?) is the more valid measurement.
- Fixed-corpus design (BrowseComp-Plus) is the right template for a reproducible coding benchmark. Freeze the repository snapshot and the tool environment so that differences between frameworks are attributable to the framework, not to environment drift.
- BRIGHT's result (MTEB leaders collapse on reasoning-heavy retrieval) warns against assuming that leaderboard rank on a generic benchmark transfers to a specialized workload.

### Gaps
- Per-dataset eRAG Kendall's tau tables and BRIGHT exact numbers were not re-verified this session.
- I found no 2026 consensus paper that replaces nDCG/recall@k for RAG retrieval. Utility-based labels remain research-stage.

---

## 2. Generation metrics: faithfulness/groundedness, answer relevance, context relevance (RAG triad), correctness, citation accuracy, hallucination rate; claim-level decomposition (FActScore, RAGChecker)

### Takeaway
Generation quality is split into *reference-free* metrics and *reference-based* metrics. The reference-free ones are faithfulness (answer is supported by the retrieved context), answer relevance and context relevance, which together form TruLens's "RAG triad". The reference-based ones are correctness against a gold answer and claim recall. The strongest validated approach decomposes answers into atomic claims and checks entailment per claim (FActScore, RAGChecker). This correlates noticeably better with humans than BLEU, ROUGE or BERTScore, but it still falls short of human-human agreement. Faithfulness is not truth: a fully grounded answer can still be wrong if the context is wrong.

### Cited Findings
- RAGAS (Es et al., 2023) defines reference-free faithfulness (claims extracted from the answer and checked against context), answer relevance (similarity between the question and questions regenerated from the answer) and context relevance. It reported agreement with human preferences on WikiEval of about 0.95 for faithfulness, 0.78 for answer relevance and 0.70 for context relevance. Context relevance was the hardest dimension. **[V; the 0.95/0.78/0.70 figures were seen secondhand via RAGXplain]** — [RAGAS arXiv 2309.15217](https://arxiv.org/pdf/2309.15217); [RAGXplain arXiv 2505.13538](https://arxiv.org/pdf/2505.13538)
- The TruLens "RAG triad" covers context relevance (query to context), groundedness (context to answer) and answer relevance (query to answer). Each edge is scored by an LLM feedback function. If all three pass, the answer is claimed to be free of hallucination *relative to the retrieved context*. **[K]** — [TruLens RAG Triad docs](https://www.trulens.org/getting_started/core_concepts/rag_triad/)
- RAGChecker's generator metrics are claim-level:
  - **Context utilization**: share of retrieved ground-truth claims that the response actually uses.
  - **Relevant/irrelevant noise sensitivity**: incorrect claims entailed by relevant or irrelevant chunks.
  - **Hallucination**: incorrect claims not entailed by any chunk.
  - **Self-knowledge**: correct claims not entailed by any chunk.
  - **Faithfulness**: claims entailed by the chunks.
  - **Overall precision/recall/F1**: claims checked against the ground-truth answer.

  **[V]** — [arXiv 2408.08067](https://arxiv.org/html/2408.08067v2)
- RAGChecker meta-evaluation used 280 pairwise instances from 8 RAG systems across 10 domains, with two annotators (90.95% agreement within 1 point). Pearson/Spearman correlation with human "overall assessment":

  | Metric | Pearson / Spearman |
  |---|---|
  | RAGChecker | 61.93 / 60.90 |
  | RAGAS Answer Similarity | 48.31 / 57.23 |
  | ROUGE-L | 43.10 / 43.21 |
  | CRUD-RAG Recall | 41.25 / 39.71 |
  | TruLens Answer Relevance | 35.15 / 33.59 |
  | BLEU | 35.14 / 29.42 |
  | BERTScore | 33.51 / 35.57 |
  | Human upper bound | 70.09 / 68.89 |

  On completeness, RAGAS Answer Similarity slightly beat RAGChecker on Spearman (61.35 vs 58.11). **[V]** — [arXiv 2408.08067](https://arxiv.org/html/2408.08067v2)
- RAGChecker's system-level findings:
  - Relevant noise is harder for generators to handle than irrelevant noise, because generators trust a relevant chunk wholesale.
  - Open-source models trusted context more readily than GPT-4.
  - Context utilization tracked overall F1 most closely.

  **[V]** — [arXiv 2408.08067](https://arxiv.org/html/2408.08067v2)
- FActScore (Min et al., 2023) breaks long-form generations into atomic facts and computes the percentage supported by a knowledge source (Wikipedia, for biographies). ChatGPT scored about 58%. An automated estimator approximated human FActScore with low (about 2%) error. **[K]** — [arXiv 2305.14251](https://arxiv.org/abs/2305.14251)
- Claim-level verification is being specialized by domain: MedRAGChecker (Jan 2026) adapts claim-level verification to biomedical RAG. **[V, title/abstract only]** — [arXiv 2601.06519](https://arxiv.org/pdf/2601.06519)
- On citation/support accuracy, TREC 2024 RAG Track (Thakur, Pradeep, Upadhyay, Campos, Craswell, Lin; SIGIR 2025) compared GPT-4o with human assessors on whether cited documents support answer sentences (45 runs, 36 topics, 3-level scale):
  - Exact label match was about 56% when humans labeled from scratch.
  - Exact match rose to about 72% when humans post-edited GPT-4o labels.
  - In disagreement analysis, an independent human sided with GPT-4o more often than with the original assessor.

  The authors conclude LLM judges can be a reliable alternative for support assessment. **[V]** — [arXiv 2504.15205](https://arxiv.org/abs/2504.15205)
- On hallucination rate as a metric, CRAG scores each answer as accurate (+1), missing/"I don't know" (0) or hallucinated (−1). Abstention is rewarded over confident errors. **[K]** — [CRAG arXiv 2406.04744](https://arxiv.org/abs/2406.04744)
- Faithfulness measures consistency with retrieved context, not factual truth. High faithfulness agreement therefore does not imply correct answers. A 2026 production study found human faithfulness ratings clustered at the top of the scale, which produced near-zero Krippendorff's alpha (a ceiling effect that makes agreement statistics unstable). **[V, from search summary]** — [arXiv 2605.27220](https://arxiv.org/pdf/2605.27220)
- "Beyond Correctness" (Oct 2025) argues that outcome-only correctness rewards in RAG training let models reach right answers with unfaithful reasoning, and proposes rewarding faithful reasoning. **[V, title/snippet only]** — [arXiv 2510.13272](https://arxiv.org/pdf/2510.13272)

### Inferences
- The claim-decomposition pattern transfers to coding agents. Instead of one holistic "is this PR good?" judgment, decompose into checkable units: each acceptance criterion, each test, each changed hunk being attributable to the task. Score precision (no unrequested changes, the analogue of "hallucination") and recall (all requirements met).
- RAGChecker's hallucination/self-knowledge split maps to whether an agent's change is grounded in what it actually read from the repo or invented. Examples are fabricated APIs, or a correct fix made without inspecting the relevant code.
- CRAG's +1/0/−1 scoring is a useful model for agents that can abstain ("I can't do this safely"). It penalizes confidently wrong patches more than declined tasks.

### Gaps
- I did not find a 2026 meta-evaluation that directly compares RAGAS, DeepEval, TruLens and RAGChecker on the same human-labeled set. Head-to-head validation evidence is still mostly the RAGChecker 2024 table above.
- Citation-accuracy metrics beyond TREC support assessment (e.g., ALCE citation recall/precision) were not researched this session.

---

## 3. Frameworks: RAGAS, ARES, RAGChecker, TruLens, DeepEval, Arize Phoenix, LangSmith, Braintrust, promptfoo, OpenAI Evals, Patronus/Lynx, Vectara HHEM

### Takeaway
The frameworks fall into three groups:
- **Metric libraries:** RAGAS, DeepEval, RAGChecker, TruLens feedback functions.
- **Statistically grounded evaluators:** ARES, with fine-tuned judges plus prediction-powered inference.
- **Observability and experiment platforms:** Phoenix, LangSmith, Braintrust, promptfoo, OpenAI Evals, which manage datasets, traces, experiments and CI.

There are also dedicated hallucination-detector models (Patronus Lynx, Vectara HHEM). Only a few have published human-agreement validation: RAGAS (WikiEval), ARES (KILT/SuperGLUE/AIS with PPI confidence intervals), RAGChecker (280-pair meta-eval) and Lynx (HaluBench). Platforms mostly leave validation to the user, and practitioner consensus is that off-the-shelf judge prompts must be validated against your own labels.

### Cited Findings
- **RAGAS**: reference-free LLM-judged faithfulness, answer relevance and context relevance. It reports high WikiEval agreement for faithfulness and weaker agreement for context relevance (see section 2). **[V]** — [arXiv 2309.15217](https://arxiv.org/pdf/2309.15217)
- **ARES** (Saad-Falcon, Khattab, Potts, Zaharia; NAACL 2024):
  - Generates synthetic training data and fine-tunes lightweight LM judges for context relevance, answer faithfulness and answer relevance.
  - Uses prediction-powered inference (PPI) with "only a few hundred human annotations" to correct judge error and give confidence intervals.
  - Evaluated on eight tasks from KILT, SuperGLUE and AIS, and remains effective under domain shift.

  **[V]** — [arXiv 2311.09476](https://arxiv.org/abs/2311.09476)
  - From memory, the paper reports ARES beating RAGAS in ranking accuracy (Kendall's tau) on context and answer relevance by large margins. The exact figures were not re-verified this session. **[K]** — [arXiv 2311.09476 HTML](https://arxiv.org/html/2311.09476v2)
- **RAGChecker** (Amazon Science, Aug 2024): claim-level diagnostic metrics for retriever and generator, open source. It has the best published correlation with humans among the compared metrics but stays below the human upper bound. **[V]** — [arXiv 2408.08067](https://arxiv.org/abs/2408.08067); [GitHub](https://github.com/amazon-science/RAGChecker)
- **TruLens**: RAG triad feedback functions with tracing (now part of Snowflake's ecosystem). Its Answer Relevance metric correlated only 35/34 (Pearson/Spearman) with human overall judgments in RAGChecker's meta-eval. **[V for correlation]** — [arXiv 2408.08067](https://arxiv.org/html/2408.08067v2); [TruLens docs](https://www.trulens.org/getting_started/core_concepts/rag_triad/) **[K]**
- **DeepEval** (Confident AI): pytest-style unit testing for LLM outputs. It has RAG metrics (answer relevancy, faithfulness, contextual relevancy/precision/recall), G-Eval custom criteria, and in 2026 agent metrics for tool calling, task completion and trace-based evaluation. The 2026 guide argues that agent judging requires inspecting "the final task result, the path that produced it, and the components involved." **[V]** — [Confident AI guide](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide); [DeepEval GitHub](https://github.com/confident-ai/deepeval) **[K]**
- **G-Eval** (Liu et al., 2023) underlies many "custom criteria" judges. It uses chain-of-thought plus form-filling, with probability-weighted scores, and reported higher Spearman correlation with humans on summarization than prior metrics. **[K]** — [arXiv 2303.16634](https://arxiv.org/abs/2303.16634)
- **Arize Phoenix**: open-source tracing (OpenTelemetry/OpenInference) plus LLM-eval templates (hallucination, QA correctness, relevance) and datasets/experiments. **[K]** — [Phoenix docs](https://arize.com/docs/phoenix)
- **LangSmith**: LangChain's tracing, datasets, offline experiments, online evaluators on production traces, and annotation queues. **[K]** — [LangSmith docs](https://docs.smith.langchain.com/)
- **Braintrust**: experiment tracking, scoring and comparison, with the open-source `autoevals` scorer library and CI integration. **[K]** — [Braintrust docs](https://www.braintrust.dev/docs); [autoevals](https://github.com/braintrustdata/autoevals)
- **promptfoo**: declarative YAML test matrices (prompts × providers × test cases), deterministic and LLM-rubric assertions, CI use, and red-teaming. **[K]** — [promptfoo docs](https://www.promptfoo.dev/docs/intro/)
- **OpenAI Evals**: an open-source registry and framework for eval templates (match, fuzzy match, model-graded). **[K]** — [GitHub openai/evals](https://github.com/openai/evals)
- **Patronus Lynx** (July 2024): an open hallucination-detection model fine-tuned from Llama-3 (8B/70B), released with HaluBench (about 15k samples, including medical and finance). The paper reported Lynx-70B outperforming GPT-4o and Claude-3-Sonnet as a judge on HaluBench. **[K]** — [arXiv 2407.08488](https://arxiv.org/abs/2407.08488)
- **Vectara HHEM**: a small cross-encoder classifier that scores whether a summary or answer is consistent with its source. It powers Vectara's public LLM hallucination leaderboard, which measures summarization hallucination rates per model. **[K]** — [HF model card](https://huggingface.co/vectara/hallucination_evaluation_model); [hallucination-leaderboard](https://github.com/vectara/hallucination-leaderboard)
- Practitioner position on all of these: "Don't use judge prompts off the shelf." Measure each judge's true positive rate and true negative rate against human labels so you can correct failure-rate estimates. Generic metrics "waste time and create false confidence," and BERTScore/ROUGE are "not useful for evaluating LLM outputs in most AI applications." **[V]** — [AI Evals FAQ (Sept 2026)](https://hamel.dev/blog/posts/evals-faq/)
- 2025 survey (Brehme et al., SDS25) is a systematic review of RAG evaluation that specifically examines whether LLM-based automated evaluation agrees with humans. **[V, abstract]** — [arXiv 2504.20119](https://arxiv.org/abs/2504.20119v1)
- 2025 survey (Gan et al.) splits RAG evaluation into internal (component) and external (system) evaluation. It also meta-analyzes evaluation practices in high-impact RAG papers. **[V, abstract]** — [arXiv 2504.14891](https://arxiv.org/abs/2504.14891v1)

### Inferences
- ARES's PPI approach transfers well to coding-agent benchmarking. Use a cheap LLM judge on all runs (e.g., code-quality grading), hand-label a few hundred, and use PPI to produce bias-corrected success estimates with confidence intervals. This is more defensible than raw judge scores.
- Platform tools (Phoenix, LangSmith, Braintrust, promptfoo) handle the plumbing (traces, datasets, CI matrices) but provide no validity guarantees. A coding-workflow benchmark should report judge validation (TPR/TNR or agreement with humans) explicitly.
- promptfoo's "matrix" model (prompts × providers × tests) maps naturally onto "frameworks × models × tasks" for a loop/workflow benchmark.

### Gaps
- I did not verify 2026 product changes (acquisitions, new agent-eval features) for Phoenix, LangSmith, Braintrust or promptfoo this session.
- Independent (non-vendor) validation of DeepEval, Phoenix and LangSmith default judge templates against human labels was not found.
- HHEM's latest version and accuracy numbers were not re-verified.

---

## 4. Benchmarks: RAG, long-context and agentic search (CRAG, FRAMES, RGB, KILT, BrowseComp, HotpotQA) and tool use (BFCL, tau-bench, ToolBench, MCP-Bench, MCP-Universe)

### Takeaway
RAG benchmarks evolved in stages:
- Wikipedia-grounded QA with provenance (KILT, HotpotQA).
- Robustness-ability probes (RGB).
- Realistic, dynamic, multi-source QA with hallucination-penalizing scoring (CRAG).
- Multi-hop "retrieve-and-reason" tasks (FRAMES).
- Hard-to-find-answer agentic browsing (BrowseComp), now with fixed-corpus variants for reproducibility (BrowseComp-Plus).

Tool-use benchmarks moved from single-call AST matching (BFCL) to multi-turn, stateful, user-simulated tasks with reliability metrics (tau-bench pass^k, tau2-bench). In 2025-2026 they moved again, to live MCP-server ecosystems (MCP-Bench, MCP-Universe), where frontier models still succeed less than half the time.

### Cited Findings
- **KILT** (Petroni et al., 2021) has 11 datasets across 5 knowledge-intensive tasks on one Wikipedia snapshot. "KILT scores" award downstream credit only when the correct provenance is also retrieved. **[K]** — [arXiv 2009.02252](https://arxiv.org/abs/2009.02252)
- **HotpotQA** (Yang et al., 2018) is multi-hop Wikipedia QA, about 113k questions, with sentence-level supporting-fact annotations enabling explainability evaluation. **[K]** — [arXiv 1809.09600](https://arxiv.org/abs/1809.09600)
- **RGB** (Chen et al., AAAI 2024) tests four RAG abilities: noise robustness, negative rejection, information integration and counterfactual robustness. LLMs struggled especially with negative rejection and counterfactual robustness. **[K]** — [arXiv 2309.01431](https://arxiv.org/abs/2309.01431)
- **CRAG** (Meta, NeurIPS 2024 D&B) has 4,409 QA pairs across 5 domains and 8 question types, with mock web-search and knowledge-graph APIs and "dynamism" levels. Straightforward RAG lifted the best LLMs from about 34% to about 44% accuracy. State-of-the-art industry RAG answered only about 63% of questions without hallucination. **[K]** — [arXiv 2406.04744](https://arxiv.org/abs/2406.04744)
- **FRAMES** (Google, 2024) has 824 multi-hop questions needing 2-15 Wikipedia articles, testing factuality, retrieval and reasoning jointly. Accuracy was about 0.40 without retrieval and about 0.66 with a multi-step retrieval pipeline. **[K]** — [arXiv 2409.12941](https://arxiv.org/abs/2409.12941)
- **BrowseComp** (OpenAI, Apr 2025) has 1,266 hard-to-find but easy-to-verify questions for browsing agents. GPT-4o with browsing scored in the low single digits, and OpenAI Deep Research scored about 51.5%. **[K]** — [arXiv 2504.12516](https://arxiv.org/abs/2504.12516)
- **BrowseComp-Plus** (Aug 2025): fixed corpus, human-verified evidence, hard negatives. Live web search APIs are cited as a source of "unfairness and irreproducibility". Expanding the corpus 10x did not change conclusions. **[V]** — [arXiv 2508.06600](https://arxiv.org/html/2508.06600v1); [alphaXiv overview](https://www.alphaxiv.org/overview/2508.06600v1)
- **BFCL** (Berkeley Gorilla) evaluates function calls via AST matching and executable checks. V3 added multi-turn and V4 added agentic components (web search, memory, format sensitivity). Secondary sources disagree on who currently leads, and BFCL updates its leaderboard frequently. **[V, secondary]** — [Spheron blog](https://www.spheron.network/blog/tool-calling-benchmarks-bfcl-tau-bench-latency-optimization/); [BenchLM BFCL v4 page (updated Sept 2026)](https://benchlm.ai/md/benchmarks/bfcl-v4.md); [official leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html) **[K]**
- **tau-bench** (Sierra, 2024) simulates users with an LLM in retail and airline domains, using domain policies and database APIs. Success is judged by comparing the final database state with the goal state. It introduced **pass^k**, the probability that *all* k trials succeed, to measure reliability. GPT-4o-class agents succeeded on under 50% of tasks, and pass^8 in retail fell below about 25%. **[K]** — [arXiv 2406.12045](https://arxiv.org/abs/2406.12045)
- **tau2-bench** (2025) adds a telecom "dual-control" domain where both the user and the agent act on a shared environment. A 2026 roundup cautions that scores are comparable only within a domain column (retail, airline and telecom are not interchangeable). **[K]** for paper — [arXiv 2506.07982](https://arxiv.org/abs/2506.07982); **[V, secondary]** — [alatirok roundup (2026)](https://alatirok.com/agentic-ai-benchmarks-2026/)
- **ToolBench/ToolLLM** (Qin et al., 2023) covers 16k+ real RapidAPI APIs and is evaluated by ToolEval (LLM-judged pass rate and win rate). StableToolBench later showed that live-API instability made results irreproducible and introduced cached/simulated APIs. **[K]** — [arXiv 2307.16789](https://arxiv.org/abs/2307.16789); [StableToolBench arXiv 2403.07714](https://arxiv.org/abs/2403.07714)
- **MCP-Universe** (Salesforce, Aug 2025, NeurIPS 2025) runs tasks against real MCP servers with execution-based evaluators. GPT-5 reached about 44% success and Grok-4 about 33%. Enterprise agents such as Cursor did not beat a standard ReAct framework. Reasoning models had lower format compliance. A third-party listing reports GPT-5 at 43.72% and Claude-4.0-Opus at 28.14%, which conflicts slightly with the paper's 44.16%. **[V]** — [arXiv 2508.14704](https://arxiv.org/pdf/2508.14704); [NeurIPS page](https://neurips.cc/virtual/2025/124527); [benchmarklist](https://benchmarklist.com/benchmarks/mcp_universe/)
- **MCP-Bench** (Accenture, Aug 2025) connects agents to 28 live MCP servers with 250 tools. Top models exceed 98% on schema compliance and valid tool naming, yet differ substantially on planning, dependency awareness and parallelism. Third-party overall scores: GPT-5 0.749, o3 0.715, gpt-oss-120b 0.692, Claude Sonnet 4 0.681. Small models degrade sharply going from single-server to multi-server tasks. **[V]** — [arXiv 2508.20453](https://arxiv.org/pdf/2508.20453); [HF papers](https://huggingface.co/papers/2508.20453); [alphaXiv](https://www.alphaxiv.org/benchmarks/uc-berkeley/mcp-bench)
- 2026 work: an SoK on agentic RAG (Mar 2026) covers taxonomy and evaluation. "Beyond Static Leaderboards" (Jun 2026) argues that accuracy-only leaderboards hide which deployment regime favors which configuration, comparing single-pass RAG with multi-hop setups. It also argues LLM-judge scoring needs an external anchor. **[V, abstract-level]** — [arXiv 2603.07379](https://arxiv.org/pdf/2603.07379); [arXiv 2606.19704](https://arxiv.org/pdf/2606.19704)
- An agentic-RAG routing benchmark (AIMultiple, updated Sept 2026) covers 759 questions requiring selection among 11 SQL databases across 70+ LLMs. It is a single vendor source with unconfirmed model names, so it is low reliability. **[V, low confidence]** — [AIMultiple](https://aimultiple.com/agentic-rag)

### Inferences
- The benchmarks that held up well (CRAG, tau-bench, BrowseComp-Plus, MCP-Universe) share three design features: (1) **outcome checks against verifiable state** (DB state, exact answers, execution) rather than judged prose, (2) **controlled environments** (mock APIs, fixed corpora), and (3) **reliability or abstention-aware scoring** (pass^k, −1 for hallucination). A coding-workflow benchmark should copy all three: test-based success, frozen repos and containers, and pass^k across repeated runs.
- MCP-Universe's finding that an enterprise coding agent (Cursor) did not beat plain ReAct supports benchmarking *workflow frameworks* directly. Scaffolding does not reliably add value, and that has to be measured.
- The ToolBench → StableToolBench and BrowseComp → BrowseComp-Plus pattern is the same lesson twice: live external dependencies destroy reproducibility.

### Gaps
- Exact current (Oct 2026) leaders on BFCL V4, tau2-bench, MCP-Universe and BrowseComp could not be confirmed from primary leaderboards. Secondary sources conflict.
- I did not research MCP-Atlas, LiveMCPBench, MCPMark or other 2026 MCP benchmarks in detail. Coverage of 2026 MCP benchmarks beyond MCP-Bench/MCP-Universe is a gap.
- CRAG, FRAMES and BrowseComp numbers are from prior knowledge [K] and should be spot-checked.

---

## 5. Methodology: golden datasets, synthetic test generation, error-analysis-first, offline regression vs online monitoring, A/B tests, guardrails, eval-driven development, component vs end-to-end

### Takeaway
The dominant 2025-2026 practitioner methodology (Husain & Shankar, with Yan and Liu in a similar vein) works in this order:
- Do error analysis first: read about 100 real traces, open-code them, cluster failures into a taxonomy.
- Write narrow, binary, validated evaluators only for the failure modes you actually observe.
- Keep small deterministic CI regression suites for known failures.
- Run sampled reference-free judges in production with confidence intervals, feeding failures back into CI.

Component evaluation (retrieval first, then generation; step-level tool-call diagnostics) is layered under end-to-end success. Product A/B metrics complement offline evals rather than replace them.

### Cited Findings
- **Error analysis first.** "Start with error analysis, not infrastructure." Review at least 30 traces manually from a pool of about 100 diverse traces. The authors spend 60-80% of development time on error analysis and evaluation. Do not write rubrics before looking at data, because criteria should emerge from review. **[V]** — [AI Evals FAQ (Sept 18, 2026; rev. Sept 21, 2026)](https://hamel.dev/blog/posts/evals-faq/)
- **Process.** Build representative traces, then open coding, then axial coding into a failure taxonomy with counts. Iterate until new traces stop revealing new failure types (a secondary mirror gives about 100 traces and stopping after about 20 yield nothing new). Start with random sampling, then targeted sampling (negative feedback, length/tool-count/latency outliers). **[V, secondary]** — [xskill mirror](https://marketnews.xskill.com/a/7263?m=en); [Lenny's Newsletter](https://www.lennysnewsletter.com/p/building-eval-systems-that-improve); [How I AI](https://www.chatprd.ai/how-i-ai/hamel-husains-guide-to-ai-evals-with-error-analysis)
- **Review cadence.** Review at least 100 fresh traces every 2-4 weeks, plus 10-20 outlier traces weekly. Mix random, clustering, outlier, classifier-flagged and feedback-based sampling. Appoint a domain-expert "benevolent dictator" for labels, and do not outsource error analysis. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **Binary over Likert.** "Binary evaluations force clearer thinking and more consistent labeling." Track per-check pass rates plus a "pass all" rate. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **LLM-as-judge validation.** Build one judge per failure mode and label 100-200 examples per failure mode, passing and failing. Split roughly 10-20% train (few-shot), 40-45% dev, 40-45% held-out test. Report TPR and TNR. Give each judge only the trace parts it needs. Same-model judging is usually acceptable if aligned. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **Code first.** Use assertions and deterministic checks before LLM judges, and build expensive evaluators only for persistent failures. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **On "eval-driven development".** Husain & Shankar caution against writing evaluators for imagined failures up front, with an exception for explicit constraints. This position differs from the "write evals first, like TDD" framing promoted by some vendors. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **Synthetic data.** Define dimensions of variation, enumerate tuples, then convert them to natural-language queries in a separate step. Never use synthetic data to estimate production failure *frequency*. Be careful in specialized, high-stakes or low-resource domains. For RAG retrieval, generate questions from facts extracted from the corpus. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **ARES-style synthetic data.** Synthetic query/passage/answer triples train judges, and a small human-labeled set calibrates them through PPI. **[V]** — [arXiv 2311.09476](https://arxiv.org/abs/2311.09476)
- **Offline CI vs online monitoring.** CI guards against known regressions with small, purpose-built datasets (often 100+ examples), favoring "assertions or other deterministic checks over LLM-as-judge evaluators." Production monitoring samples live traces, runs reference-free evaluators asynchronously and tracks confidence intervals ("If the lower bound crosses your threshold, investigate further"). Representative production failures get added back to CI. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **Guardrails vs evaluators.** Guardrails are inline, fast and usually deterministic, aimed at clear-cut high-impact failures. Evaluators run asynchronously after generation for nuanced qualities. Slow, non-deterministic LLM judges are rarely suitable as synchronous guardrails. "Do not use llm guardrails off the shelf blindly." **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- **Component vs end-to-end (RAG).** Evaluate retrieval with IR metrics, then generation with error analysis and validated judges. The FAQ maps Jason Liu's tiered RAG-eval framework onto the question-context-answer relationships. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
  - Jason Liu's related writing argues that RAG evals reduce to a small set of relationships among question, context and answer. **[K]** — [jxnl.co, "There Are Only 6 RAG Evals" (2025)](https://jxnl.co/writing/2025/05/19/there-are-only-6-rag-evals/)
- **Component vs end-to-end (agents).** Phase 1 treats the agent as a black box: a precise success rule per task, recording the *first upstream failure*. Phase 2 is step-level diagnostics: tool choice, parameter extraction, error handling, context retention, efficiency and checkpoints. Test tool calls in parts (name, arguments, result, resulting state, including authorization and preconditions). Use **transition failure matrices**, with rows as the last successful state and columns as the first failure, to show where failures cluster. Early-stage errors cascade. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- 2026 trend: agent evaluation is moving to trace-level judging of outcome, path and components. **[V]** — [Confident AI (2026)](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide)
- **A/B and online product metrics.** The FAQ's Copilot example tracks whether users accept suggestions *and keep the code afterward*, complementing offline evals. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
  - GitHub's Copilot study found acceptance rate was the usage metric that best predicted developers' *perceived* productivity. **[K]** — [Ziegler et al., arXiv 2205.06537](https://arxiv.org/abs/2205.06537)
- **Coding agents specifically (FAQ).** Use public benchmarks (SWE-bench, Terminal-Bench, Aider Polyglot, HumanEval) for broad agents and product-specific evals for narrow workflows. Build private benchmarks from hard internal tasks, each with a working environment and code-based tests. Start with clearly specified tasks and stable environments verifiable by unit tests, then add code-quality checks (net new lines, cyclomatic complexity) and interaction checks. Copilot-style tactics include deleting function bodies and regenerating them against existing tests. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- Eugene Yan's survey of LLM-evaluators recommends task-specific evaluators and validating evaluators against human labels. He notes that pairwise comparison is often more reliable than direct scoring for subjective criteria, while direct scoring suits objective checks such as faithfulness. **[K]** — [eugeneyan.com, "Evaluating the Effectiveness of LLM-Evaluators"](https://eugeneyan.com/writing/llm-evaluators/)
- "Who Validates the Validators?" (Shankar et al., 2024) documents **criteria drift**: users' grading criteria change as they grade outputs, so evaluator alignment must be iterative. **[K]** — [arXiv 2404.12272](https://arxiv.org/abs/2404.12272)

### Inferences
- For a coding-workflow benchmark, the methodology translates directly:
  1. Run frameworks on tasks and *read the transcripts* before choosing metrics.
  2. Open/axial-code failure types (wrong file, test hacking, premature stop, loop thrash, scope creep).
  3. Make hidden tests the primary binary outcome.
  4. Add narrow, TPR/TNR-validated judges only for observed qualitative failures.
  5. Use transition failure matrices over framework phases (plan → edit → test → review) to locate where each framework breaks.
- "Accepted and kept" (Copilot) is the coding analogue of "user satisfaction". Offline pass rates should eventually be sanity-checked against a retention-style signal (e.g., whether a human would merge it unchanged).
- Since synthetic tasks cannot estimate real-world failure frequency, a benchmark built from synthetic tasks should present results as *capability* comparisons, not production failure rates.

### Gaps
- I found no rigorous published A/B methodology specific to RAG (e.g., variance and sample-size guidance for LLM-feature experiments) this session.
- "Eval-driven development" as a formal methodology has no single canonical source. The FAQ's caution is the clearest primary statement found.

---

## 6. Critiques: reference-free metric unreliability, metric gaming, automatic metrics vs user satisfaction

### Takeaway
Reference-free and LLM-judged metrics are useful but fragile. The main problems:
- They sit well below human-human agreement (RAGChecker's best correlation is about 0.62 against a human ceiling of about 0.70).
- They suffer known biases (position, verbosity, self-preference).
- Raw percent agreement overstates validity, a point the 2026 "reliability without validity" critique makes.
- They depend on criteria and metadata framing.
- Ceiling effects make agreement statistics meaningless.

Separately, optimizing faithfulness or correctness can be gamed, and offline metrics can diverge from user outcomes. The remedy is validation against domain-expert labels, chance-corrected agreement, outcome anchors and periodic re-validation.

### Cited Findings
- **"Reliability without Validity"** (Jun 2026) is a large-scale evaluation of LLM judges across agreement, consistency and bias. Common exact-match agreement checks do not correct for chance and overstate a judge's discriminative ability. The authors propose a "Minimum Viable Validation Protocol." **[V, abstract-level]** — [arXiv 2606.19544](https://arxiv.org/abs/2606.19544)
- **"Do LLM Judges Measure What We Mean?"** (NeurIPS 2026) audits six LLM judges on 386 answers to 65 Polish public-sector RAG questions. It finds judge trustworthiness depends on the full setup (criteria, evidence, output form, score interpretation) and is sensitive to metadata, not just the judge model. **[V, abstract-level]** — [NeurIPS 2026](https://neurips.cc/virtual/2026/178300)
- **Survey of LLM-judge security and reliability** (Mar 2026) reviews 45 studies (2020-2026) covering position bias and adversarial vulnerabilities of judges. **[V, abstract-level]** — [arXiv 2603.29403](https://arxiv.org/pdf/2603.29403)
- **Legal-RAG judge selection** (Sept 2025) argues that no single inter-rater reliability statistic suffices and recommends multiple IRR metrics plus significance testing. **[V, abstract-level]** — [arXiv 2509.12382](https://arxiv.org/html/2509.12382v1)
- **Judge bias, foundational evidence.** MT-Bench/Chatbot Arena work found GPT-4 judges reach over 80% agreement with humans (comparable to human-human) but exhibit position, verbosity and self-enhancement biases. **[K]** — [Zheng et al., arXiv 2306.05685](https://arxiv.org/abs/2306.05685)
- **Gap to humans.** Even the best claim-level RAG metric reached Pearson 61.9 vs a human ceiling of 70.1. TruLens answer relevance was 35, about the same as BLEU. **[V]** — [arXiv 2408.08067](https://arxiv.org/html/2408.08067v2)
- **Ceiling effects.** Human faithfulness ratings concentrated at the top of the scale gave near-zero Krippendorff's alpha in a 2026 production study. **[V, search summary]** — [arXiv 2605.27220](https://arxiv.org/pdf/2605.27220)
- **Faithfulness is not correctness.** Faithfulness can be high with wrong context. It can also undercount semantically correct answers when claim decomposition or entailment judging fails. That point came from a practitioner blog, not peer review. **[V]** — [RAGAS paper](https://arxiv.org/pdf/2309.15217); [practitioner blog (Level Up Coding)](https://levelup.gitconnected.com/rag-evaluation-in-practice-faithfulness-answer-relevance-and-context-precision-in-ragas-ce13b0363cda)
- **Human labels are also noisy.** In TREC 2024 RAG support assessment, an independent human agreed with GPT-4o more often than with the original human assessor. The "gold standard" itself carries error. **[V]** — [arXiv 2504.15205](https://arxiv.org/abs/2504.15205)
- **Metric gaming via outcome-only rewards.** Correctness-only rewards in RAG training let models reach right answers through unfaithful reasoning (Oct 2025). **[V, snippet]** — [arXiv 2510.13272](https://arxiv.org/pdf/2510.13272)
- **Leaderboard gaming.** "The Leaderboard Illusion" documented how private testing and selective disclosure on Chatbot Arena distort rankings. **[K]** — [arXiv 2504.20879](https://arxiv.org/abs/2504.20879)
- **Accuracy-only leaderboards hide deployment trade-offs.** No single RAG configuration dominates across regimes, so an external anchor is needed for judge-scored metrics (Jun 2026). **[V, abstract-level]** — [arXiv 2606.19704](https://arxiv.org/pdf/2606.19704)
- **Automatic metrics vs user outcomes.** METR's 2025 randomized trial found experienced open-source developers were about 19% *slower* with AI tools, despite expecting a speedup and believing afterwards that they had been faster. Perceived and measured outcomes diverge. **[K]** — [arXiv 2507.09089](https://arxiv.org/abs/2507.09089)
- **Practitioner critique.** Generic metrics "create false confidence," and dashboards of generic scores should give way to narratives with root causes. **[V]** — [AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)

### Inferences
- For a coding-workflow benchmark, any LLM-judged dimension (code quality, plan quality) should report chance-corrected agreement (Cohen's kappa or Krippendorff's alpha) or TPR/TNR against human labels, not raw percent agreement.
- Test-based success is the coding analogue of an "external anchor". It is far more robust than judged metrics, but it is gameable too (special-casing tests, weakening assertions). Hidden tests and diff inspection for test tampering mirror the "faithfulness is not correctness" caveat.
- The METR result and the Copilot acceptance study together suggest benchmark wins may not translate to developer productivity. Present offline benchmark results as necessary but not sufficient evidence.

### Gaps
- Full texts of the 2026 judge-critique papers (2606.19544, NeurIPS 2026 audit, 2603.29403) were not read. Their specific numbers and protocol details are unverified.
- I found no 2026 study that directly measures the correlation between RAG offline metrics (RAGAS etc.) and end-user satisfaction or retention in production. This remains a notable evidence gap in the field.
