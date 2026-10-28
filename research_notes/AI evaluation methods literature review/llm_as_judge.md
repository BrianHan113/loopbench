# Non-deterministic Evaluation Methods: LLM-as-a-Judge, Agent-as-a-Judge, Rubric Grading and Human Evaluation (emphasis: code and agent outputs)

Notes compiled 2026-10-28. Verification status: items marked (verified) were fetched or retrieved this session (Oct 2026). Unmarked foundational items (2023-2025) were cited from well-known arXiv papers. I did not re-fetch them this session, so their exact numbers should be spot-checked before final publication. Items marked (snippet) come only from search-result snippets.

---

## 1. Foundations: how LLM judges work (MT-Bench, G-Eval, Prometheus, JudgeLM, PandaLM; pointwise / pairwise / reference-guided; rubric and checklist grading)

### Takeaway
LLM-as-a-judge started with Zheng et al. 2023. That paper found strong-model judges agree with humans about as often as humans agree with each other on open-ended chat. It also catalogued three biases: position, verbosity and self-enhancement. The field has since moved from free-form 1-10 scoring toward decomposed rubrics and checklists, as in CheckEval, HealthBench and PaperBench. Decomposed criteria are more reliable, auditable and calibratable than a single holistic score. This is the design pattern most relevant to a small code-quality judge.

### Cited Findings
**Zheng et al. 2023 (MT-Bench / Chatbot Arena)**
- Introduced MT-Bench (80 multi-turn questions) and Chatbot Arena. The paper defines three judge modes: pairwise comparison, single-answer (pointwise) grading, and reference-guided grading. It found GPT-4 as judge reaches over 80% agreement with human preferences, "the same level of agreement between humans." — [Zheng et al. 2023](https://arxiv.org/abs/2306.05685)
- The same paper names position bias, verbosity bias, self-enhancement bias, and limited math/reasoning grading ability as key limitations. It proposes mitigations: swapping positions, few-shot examples, chain-of-thought, and reference-guided grading, where the judge first solves the problem or receives a reference answer. Reference guidance cut math-grading failures substantially. — [Zheng et al. 2023](https://arxiv.org/abs/2306.05685)

**G-Eval (Liu et al. 2023)**
- G-Eval uses auto-generated chain-of-thought evaluation steps plus a form-filling paradigm. It weights each score by the token probabilities of score tokens to get finer-grained continuous scores. With GPT-4 it reached Spearman 0.514 with human judgments on summarization (SummEval), beating prior metrics. The authors also flag a bias of LLM evaluators toward LLM-generated text. — [Liu et al. 2023, G-Eval](https://arxiv.org/abs/2303.16634)

**Prometheus 1 and 2 (Kim et al.)**
- Prometheus (13B) is an open evaluator trained on the "Feedback Collection" dataset, built from about 1K fine-grained score rubrics. It takes a user-defined rubric and a reference answer and outputs feedback plus a 1-5 score. It reported Pearson correlation with human evaluators on par with GPT-4 when given customized rubrics. — [Kim et al. 2023, Prometheus](https://arxiv.org/abs/2310.08491)
- Prometheus 2 (7B and 8x7B) supports both direct assessment (pointwise) and pairwise ranking by merging weights of separately trained models. It reported the highest correlation with humans and proprietary judges among open evaluators at release. — [Kim et al. 2024, Prometheus 2](https://arxiv.org/abs/2405.01535)

**JudgeLM and PandaLM**
- JudgeLM fine-tunes 7B/13B/33B judges on GPT-4 judgments and reports over 90% agreement with the GPT-4 teacher, exceeding human-human agreement. It names position bias, knowledge bias and format bias. It mitigates them with swap augmentation, reference support and reference drop. — [Zhu et al. 2023, JudgeLM](https://arxiv.org/abs/2310.17631)
- PandaLM is a 7B pairwise judge for comparing instruction-tuned models. It reported reaching about 93.75% of GPT-3.5's and 88.28% of GPT-4's evaluation ability (F1) on its human-annotated test set. — [Wang et al. 2023, PandaLM](https://arxiv.org/abs/2306.05087)

**Rubric and checklist approaches**
- CheckEval breaks evaluation criteria into a checklist of fine-grained Boolean (yes/no) questions. It reports better inter-evaluator agreement and interpretability than Likert-style scoring. — [Lee et al. 2024, CheckEval](https://arxiv.org/abs/2403.18771)
- HealthBench (OpenAI, May 2025) has 5,000 multi-turn health conversations graded against 48,562 unique conversation-specific rubric criteria written by 262 physicians. Each criterion carries a positive or negative point weight, and a model grader (GPT-4.1) checks each criterion. The grader was meta-evaluated against physician agreement on criteria. — [Arora et al. 2025, HealthBench](https://arxiv.org/abs/2505.08775); [OpenAI HealthBench page](https://openai.com/index/healthbench/)
- PaperBench (OpenAI, Apr 2025) grades agent replications of 20 ICML 2024 papers with hierarchical rubric trees, co-developed with the papers' authors and totalling 8,316 individually gradable leaf tasks. Leaf scores are binary and aggregate up the tree with weights. The authors built a separate judge benchmark (JudgeEval) to validate the LLM judge. Their best judge configuration (o3-mini-based SimpleJudge) reached F1 of about 0.83 against human grading. — [Starace et al. 2025, PaperBench](https://arxiv.org/abs/2504.01848)
- Anthropic's Jan 2026 agent-evals guidance recommends "clear, structured rubrics to grade each dimension of a task" and grading each dimension with an isolated LLM judge rather than one judge for everything. It warns that "vague rubrics produce inconsistent judgments" and recommends giving the judge an escape hatch, such as returning "Unknown" when information is insufficient. (verified) — [Anthropic, Demystifying evals for AI agents (2026-01-09)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- "Rubrics as Rewards" (2025) uses checklist-style rubrics as RL reward signals with an LLM judge. This is evidence that rubric decomposition is now used in training as well as evaluation. — [Gunjal et al. 2025](https://arxiv.org/abs/2507.17746)

### Inferences
- For a small code-quality judge, the strongest pattern is HealthBench/PaperBench-style decomposition: a handful of binary, observable criteria, each graded separately, with weights. It beats a single 1-10 "quality" score. Binary criteria are easier to calibrate against ~15 human ratings because per-criterion agreement can be measured.
- Pointwise scoring suits a benchmark, since each run gets an absolute score. Pairwise comparison is more reliable per judgment but scales quadratically and invites position bias. For comparing a few frameworks on the same task, pairwise with order swapping is a reasonable secondary check.

### Gaps
- I did not re-fetch the exact HealthBench grader meta-evaluation numbers (macro-F1 versus physicians) this session.

---

## 2. Known biases and failure modes (position, verbosity/length, self-preference, authority, sycophancy, adversarial susceptibility, low agreement on hard items)

### Takeaway
Judge biases are well documented and measurable, but their size varies a lot by judge model and task. The 2026 literature adds two points. First, single-trial judgments are surprisingly noisy: pairwise verdicts flip about 14% of the time on identical reruns. Second, high test-retest consistency does not imply low bias. Judges are also easy to game: null or constant outputs and "master key" tokens can inflate scores. Judges also degrade sharply on items they could not solve themselves.

### Cited Findings
**Position bias**
- With ChatGPT as evaluator, simply swapping answer order let Vicuna-13B "beat" ChatGPT on 66 of 80 queries. The authors propose Balanced Position Calibration (score both orders) and Multiple Evidence Calibration. — [Wang et al. 2023, "LLMs are not Fair Evaluators"](https://arxiv.org/abs/2305.17926)
- A systematic study of position bias across judges and tasks measured repetition stability, position consistency and preference fairness. Position bias is strongest when the two candidates are close in quality. — [Shi et al. 2024, Judging the Judges: position bias](https://arxiv.org/abs/2406.07791)
- 2026: GPT-4o-mini showed significant first-position bias, with 72% "A" majority (p = 0.024). (verified) — [Yagubyan 2026, "The Coin Flip Judge?"](https://arxiv.org/abs/2606.13685)
- 2026: In two production-deployed judges, test-retest reliability above 0.95 coexisted with position bias above 0.10, a "consistency-bias paradox." (verified) — [Norman, Rivera & Hughes 2026, "Reliability without Validity"](https://arxiv.org/abs/2606.19544)
- CodeJudgeBench (code-specific) found that "simply changing the order in which responses are presented can substantially impact accuracy" in pairwise code judging. (verified) — [Jiang et al. 2025/ACL 2026, CodeJudgeBench](https://arxiv.org/abs/2507.10535)

**Verbosity and length bias**
- Zheng et al. showed a "repetitive list" attack exploiting verbosity bias. GPT-4 resisted it much better than GPT-3.5 or Claude-v1. — [Zheng et al. 2023](https://arxiv.org/abs/2306.05685)
- Length-controlled AlpacaEval regresses out length effects and raised Spearman correlation with Chatbot Arena from 0.94 to 0.98. This is evidence that raw judge win-rates are length-confounded. — [Dubois et al. 2024, Length-Controlled AlpacaEval](https://arxiv.org/abs/2404.04475)
- Counterpoint, 2026: across 21 judges, verbosity bias was small (below 0.011) under a single pairwise rubric. This suggests modern judges with explicit rubrics have reduced, though not eliminated, length bias. (verified) — [Norman et al. 2026](https://arxiv.org/abs/2606.19544)

**Self-preference**
- LLM evaluators recognize and favor their own generations, and the strength of self-preference correlates linearly with self-recognition ability (GPT-4, Llama 2). — [Panickssery et al. 2024](https://arxiv.org/abs/2404.13076)
- Self-preference has been linked to lower perplexity: judges favor text that is more familiar to them. — [Wataoka et al. 2024, Self-Preference Bias in LLM-as-a-Judge](https://arxiv.org/abs/2410.21819)

**Authority and other cognitive biases**
- Both human and LLM judges are swayed by fake citations (authority bias), misinformation and superficial "beauty" of presentation. — [Chen et al. 2024, "Humans or LLMs as the Judge?"](https://arxiv.org/abs/2402.10669)
- The CALM framework quantifies 12 judge biases, including position, verbosity, self-enhancement, authority, bandwagon, sentiment, diversity, distraction, CoT and refinement-aware biases. Even strong models show significant bias on specific tasks. — [Ye et al. 2024, "Justice or Prejudice?"](https://arxiv.org/abs/2410.02736)

**Run-to-run non-determinism (2026)**
- On 29 tasks with GPT-4o-mini and GPT-4.1-mini, pairwise preferences flipped 13.6% of the time on average across identical reruns. 28% of questions exceeded a 20% flip rate, and the worst reached 56%. Cross-judge agreement was 76% (κ = 0.51). Semantically equivalent prompt templates changed the majority outcome in 25% of tested cases. A majority vote needed about 11 repeated trials on average, and 15 for high-variance items, to recover the 50-trial verdict with 95% probability. Deterministic decoding reduces inconsistency but does not eliminate it. (verified) — [Yagubyan 2026](https://arxiv.org/abs/2606.13685)
- The same paper reports a "pairwise-pointwise gap." Judges confidently pick pairwise winners even when their own pointwise scores show no significant quality difference (mean gaps of 0.19-0.36 on a 10-point scale). (verified) — [Yagubyan 2026](https://arxiv.org/abs/2606.13685)

**Adversarial susceptibility / reward hacking**
- A "null model" that always returns the same constant (non-answer) response achieved an 86.5% length-controlled win rate on AlpacaEval 2.0, plus high scores on Arena-Hard-Auto and MT-Bench. — [Zheng et al. 2024, "Cheating Automatic LLM Benchmarks"](https://arxiv.org/abs/2410.07137)
- Universal adversarial phrases appended to responses inflate judge scores. Absolute (pointwise) scoring is more vulnerable than comparative scoring. — [Raina et al. 2024, "Is LLM-as-a-Judge Robust?"](https://arxiv.org/abs/2402.14016)
- "Master key" inputs such as a lone ":" or "Thought process:" elicit false-positive "correct" verdicts from generative reward models and LLM judges. — [Zhao et al. 2025, "One Token to Fool LLM-as-a-Judge"](https://arxiv.org/abs/2507.08794)
- CodeJudgeBench released an adversarial companion set (13 Feb 2026) for code judges. (snippet) — [CodeJudgeBench HF dataset README](https://huggingface.co/datasets/mattymchen/codejudgebench/blob/main/README.md)

**Low agreement on hard and technical items**
- JudgeBench pairs responses on challenging knowledge, reasoning, math and coding questions with objective correctness labels. Many strong models, including GPT-4o, performed only slightly better than random guessing. — [Tan et al. 2024, JudgeBench](https://arxiv.org/abs/2410.12784)
- Judges are much less accurate at grading correctness on questions they cannot answer themselves. Supplying a correct human-written reference largely restores performance. — [Krumdick et al. 2025, "No Free Labels"](https://arxiv.org/abs/2503.05061)
- Judge rankings shifted by up to 14 positions across MT-Bench, JudgeBench and RewardBench. A judge that is good on one benchmark is not reliably good on another. (verified) — [Norman et al. 2026](https://arxiv.org/abs/2606.19544)

**Sycophancy**
- Authority and bandwagon biases are the closest well-studied analogues: judges defer to stated sources or claimed majority opinions. — [Ye et al. 2024 (CALM)](https://arxiv.org/abs/2410.02736); [Chen et al. 2024](https://arxiv.org/abs/2402.10669)

### Inferences
- The most dangerous failure for /goal in this project is "claimed but not demonstrated" completion: an agent asserts tests pass and the transcript-only judge accepts it. That is an authority-bias analogue, made worse by the judge having no tools (see section 4). "Master key" results suggest confident completion language could plausibly bias a small judge model.
- Mitigation checklist for a small code judge: randomize or swap order for pairwise checks; use binary per-criterion questions; provide reference material (the task spec, test output); run 3-5 repeats and take the majority, or report the variance; use a judge from a different model family than the generator.

### Gaps
- I found no dedicated, quantified study of judge sycophancy, meaning judges flipping verdicts when the evaluated text pushes back or asserts its own correctness, separate from authority bias.
- I did not obtain the 2026 "Reliability without Validity" paper's full Minimum Viable Validation Protocol; only the abstract was accessible.

---

## 3. Code-specific judges (ICE-Score, CodeJudge, CodeJudgeBench, CodeUltraFeedback, judges vs execution/tests)

### Takeaway
LLM judges of code correctness work reasonably on easy problems, but they are clearly worse than execution on hard ones. They are sensitive to response order and to which model wrote the code. Reasoning ("thinking") models are now the best code judges. Consensus practice, including Anthropic's 2026 guidance, is to use tests and execution for correctness and reserve the LLM judge for qualities tests cannot measure, such as readability, maintainability and adherence to conventions.

### Cited Findings
- ICE-Score (2023) prompts an LLM to rate code on usefulness and functional correctness. It correlated better with human and execution judgments than BLEU/CodeBLEU/CodeBERTScore-style metrics. — [Zhuo 2023, ICE-Score](https://arxiv.org/abs/2304.14317)
- CodeJudge (EMNLP 2024) adds "analyze then summarize" and a taxonomy-guided fault-localization prompt for semantic correctness without tests. It beat prior methods across multiple datasets and languages, even with smaller open models. — [Tong & Zhang 2024, CodeJudge](https://arxiv.org/abs/2410.02184)
- CodeUltraFeedback has 10K coding instructions answered by 14 LLMs, with responses rated by GPT-3.5 as judge on five coding preferences (e.g., instruction-following, explanation, complexity/efficiency, style, readability). It ships CODAL-Bench for evaluating alignment to coding preferences. — [Weyssow et al. 2024, CodeUltraFeedback](https://arxiv.org/abs/2403.09032)
- CodeJudge-Eval asks models to judge the correctness of provided solutions, using verdicts such as accepted, wrong answer and time limit exceeded. Even top proprietary models struggled, which shows judging code differs from generating it. — [Zhao et al. 2024, CodeJudge-Eval](https://arxiv.org/abs/2408.10718)
- **CodeJudgeBench** (v1 Jul 2025; ACL 2026) benchmarked 26 judges on code generation, repair and unit-test generation. (verified) Findings:
  - Thinking models significantly outperform non-thinking ones, and small thinking models (Qwen3-8B) can beat specially trained judges up to 70B.
  - All models show significant randomness.
  - Response order substantially affects pairwise accuracy.
  - Performance varies with which LLM wrote the code.
  - Pairwise beats scalar pointwise judging.
  - Keeping the full unprocessed response, including comments and reasoning, improves judging.
  — [Jiang et al., CodeJudgeBench](https://arxiv.org/abs/2507.10535); [ACL 2026 entry](https://aclanthology.org/2026.acl-long.888/)
- An ICLR 2026 paper benchmarking code verification strategies with LLM judges found that common approaches, such as judge-generated unit tests or direct correctness prediction, struggle on harder coding problems. (snippet) — [ICLR 2026, Benchmarking Code Verification Strategies with LLMs-as-a-judge](https://iclr.cc/virtual/2026/10012641)
- MCTS-Judge applies test-time search to code-correctness judging and reports large accuracy gains for a small code model. This is evidence that more judge compute helps on code. — [MCTS-Judge 2025](https://arxiv.org/abs/2502.12468)
- An empirical SE study (ISSTA 2025) compared LLM-as-judge methods with human ratings on code generation, translation and summarization. Output-based judging with large models aligned best with humans. Alignment varied substantially by task. — [Wang et al. 2025, "Can LLMs Replace Human Evaluators? LLM-as-a-Judge in SE"](https://arxiv.org/abs/2502.06193)
- A 2025 survey covers LLMs as software-engineering judges and their open challenges. — [He et al. 2025, "From Code to Courtroom"](https://arxiv.org/abs/2503.02246)
- CriticGPT (OpenAI): for naturally occurring bugs, trainers preferred model-written code critiques over human critiques 63% of the time. Human+model teams produced more comprehensive critiques with fewer hallucinated bugs than the model alone. — [McAleese et al. 2024, "LLM Critics Help Catch LLM Bugs"](https://arxiv.org/abs/2407.00215)
- Anthropic (Jan 2026): "Coding evals typically rely on unit tests for correctness, with an LLM rubric for code quality." It recommends deterministic graders wherever possible, a reference solution to prove each task solvable, isolated clean environments, and hack-resistant graders. Its example: Claude gained an unfair advantage by reading git history from previous trials. (verified) — [Anthropic, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### Inferences
- The project's design, with tests for correctness and an optional LLM judge for code quality, matches the 2026 consensus. The judge should not be asked "is this correct?" when tests can answer that. It should get test results as context, which is reference-guided grading, so it doesn't re-derive correctness.
- Because judge accuracy varies with which LLM wrote the code, and self-preference exists, the judge should ideally be a different model family from the coding agent. Otherwise, report this as a threat to validity.
- CodeJudgeBench's finding that pairwise beats pointwise suggests an optional secondary analysis: pairwise comparison of two frameworks' outputs on the same task, run in both orders.

### Gaps
- I couldn't extract the specific numbers for judge accuracy versus unit-test ground truth (for example, best-model accuracy on CodeJudgeBench) in this session.
- I found no study of LLM judges rating code *quality* (maintainability, idiom) against expert human ratings at small sample sizes. Most code-judge work targets functional correctness.

---

## 4. Agent-as-a-Judge, trajectory judges, process reward models, judging whether an agent truly verified its work

### Takeaway
Agent-as-a-Judge, where the judge has tools to inspect files, run code and read trajectories, aligns substantially better with human experts on agentic coding tasks than single-pass LLM judges do. Process reward models score intermediate steps rather than outcomes. The /goal evaluator in Claude Code is explicitly the *weaker* kind: a tool-less, transcript-only small fast model that can only judge what the agent surfaced. So it checks whether the agent *showed* evidence of verification, not whether verification is true.

### Cited Findings
- Agent-as-a-Judge (Zhuge et al., Oct 2024) uses agentic systems to evaluate agentic systems, with intermediate feedback over the whole task-solving process. It introduced DevAI, 55 realistic AI-development tasks with 365 hierarchical user requirements, and evaluated three popular agentic coding systems. It "dramatically outperforms" LLM-as-a-Judge and is "as reliable as" the human evaluation baseline. (verified, abstract) — [Zhuge et al. 2024](https://arxiv.org/abs/2410.10934)
- Secondary sources attribute to that paper roughly 90% alignment with human experts for Agent-as-a-Judge versus about 70% for LLM-as-a-Judge, and about 97% reductions in evaluation cost and time versus human evaluation. (snippet; I could not confirm these figures from the abstract) — [arXiv 2508.02994 survey](https://arxiv.org/abs/2508.02994); [Zhuge et al. 2024](https://arxiv.org/abs/2410.10934)
- A Jan 2026 survey on Agent-as-a-Judge sorts judge systems into procedural (fixed multi-agent workflows), reactive (route and invoke tools based on findings) and self-evolving (write their own rules) stages. It covers code judges that execute or inspect artifacts. (snippet) — [You et al. 2026, A Survey on Agent-as-a-Judge](https://arxiv.org/abs/2601.05111)
- AgentRewardBench evaluates LLM judges on web-agent trajectories, with expert-annotated trajectories across several web benchmarks. No single LLM judge excelled across all benchmarks, and rule-based evaluation tended to under-report agent success. — [Lù et al. 2025, AgentRewardBench](https://arxiv.org/abs/2504.08942)
- Process supervision (step-level labels) beat outcome supervision for training reward models on MATH. The PRM800K dataset holds about 800K step-level human labels. — [Lightman et al. 2023, "Let's Verify Step by Step"](https://arxiv.org/abs/2305.20050)
- Generative verifiers frame verification as next-token prediction with chain-of-thought. They outperform discriminative reward models and benefit from majority voting over sampled verifications. — [Zhang et al. 2024, Generative Verifiers](https://arxiv.org/abs/2408.15240)
- **Claude Code /goal (official docs, accessed 2026-10-28):** (verified)
  - "/goal is a wrapper around a session-scoped prompt-based Stop hook." After each turn, Claude Code sends the condition and the conversation so far to the configured small fast model (Haiku by default; override via `ANTHROPIC_DEFAULT_HAIKU_MODEL`). The model returns *not yet met* (with a reason used as guidance), *met*, or *impossible*.
  - The evaluator "doesn't run commands or read files independently" and "does not call tools, so it can only judge what Claude has already surfaced in the conversation."
  - Docs advise conditions with one measurable end state, a stated check (e.g., "`npm test` exits 0"), and constraints (e.g., "no other test file is modified"). Conditions can be up to 4,000 characters. A turn or time bound can be included ("or stop after 20 turns").
  - "completion is decided by a fresh model rather than the one doing the work."
  - The loop stops if Claude keeps answering the evaluator without tool use for several turns.
  - Evaluation is skipped while subagents or background commands are running.
  — [Claude Code docs: /goal](https://code.claude.com/docs/en/goal)
- Anthropic recommends grading *outcomes* (final environment state, e.g., whether the reservation exists in the database) rather than the agent's claims. It notes that rigid step-sequence checks are "overly brittle," and that "You won't know if your graders are working well unless you read the transcripts." (verified) — [Anthropic 2026](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- A 2026 paper argues that benchmarks mis-score computer-use agents. (snippet only; claims not verified) — [arXiv 2607.28367](https://arxiv.org/abs/2607.28367)

### Inferences
- /goal is a transcript-only LLM-as-a-judge, not an Agent-as-a-Judge. Its verdict is only as trustworthy as the evidence the working agent chose to print. An agent can satisfy it by printing a plausible "all tests pass" without actually running the full suite, or by running a narrowed test command. The benchmark should independently re-run the tests in a clean environment as ground truth, and measure **/goal false-accept rate**: how often /goal says "met" while the independent check fails. That is a cheap, high-value judge-validation metric for this project.
- Conditions that require the transcript to show a specific command and its exit code ("`pytest -q` output shows 0 failed, run after the final edit") narrow the judge's latitude. This converts the judge from a semantic assessor into a near-deterministic pattern check.
- Because the default evaluator is a small model (Haiku), JudgeBench-style findings about weak judges on hard items and "master key" susceptibility are relevant. Consider testing /goal with adversarial transcripts, such as an agent claiming success without evidence, to estimate its robustness.

### Gaps
- I found no published study measuring the false-accept rate of transcript-only completion judges like /goal, or of Codex's equivalent. This is a genuine gap the project could fill.
- I could not verify the exact Agent-as-a-Judge agreement percentages from the primary source this session.

---

## 5. Validation and calibration (agreement metrics, label counts, juries/PoLL, Judging the Judges, PPI, EvalGen, critique shadowing)

### Takeaway
Validate a judge as a classifier against human labels using chance-corrected agreement (Cohen's κ for binary or categorical, Krippendorff's α for multiple raters or ordinal scales) and report TPR and TNR separately. Raw percent agreement overstates quality, by 33-41 points versus κ on MT-Bench in a 2026 study. Practitioner guidance (Husain/Shankar) suggests about 100 labels per failure mode for validation. That makes ~15 human ratings a *pilot calibration*, not a validation: confidence intervals will be very wide. Juries of diverse small judges and PPI-style statistical correction are the main ways to stretch scarce human labels.

### Cited Findings
**Agreement metrics**
- "Judging the Judges" evaluated 13 judge models against human annotations on a clean TriviaQA setup. Only the largest models (e.g., GPT-4 Turbo, Llama-3 70B) aligned well with humans, and even they fell short of human-human agreement. Cohen's κ distinguishes judges far better than percent agreement: judges with high percent agreement could still assign scores differing by up to 5 points from human scores. Some smaller models ranked systems well despite poor absolute alignment. — [Thakur et al. 2024, "Judging the Judges"](https://arxiv.org/abs/2406.12624)
- 2026: Across 21 judges, 9 providers and about 541,000 judgments, the gap between exact-match agreement and Cohen's κ was 33-41 percentage points on MT-Bench. The authors argue exact-match agreement "does not correct for chance and overstates discriminative ability." (verified) — [Norman et al. 2026](https://arxiv.org/abs/2606.19544)
- Conventional κ interpretation bands (Landis & Koch): 0.41-0.60 moderate, 0.61-0.80 substantial, 0.81+ almost perfect. — [Cohen's kappa (Wikipedia)](https://en.wikipedia.org/wiki/Cohen%27s_kappa)
- Krippendorff suggests α ≥ 0.800 for reliable data and 0.667 as the lowest bound for tentative conclusions. α handles any number of raters, missing data and ordinal or interval scales. — [Krippendorff's alpha (Wikipedia)](https://en.wikipedia.org/wiki/Krippendorff%27s_alpha)
- Eugene Yan's review of LLM-evaluator research recommends classification metrics (precision/recall on defects) for binary judgments and correlation metrics (Spearman, Kendall τ) for scores. Cohen's κ is preferable to raw agreement, pairwise comparisons are more reliable than direct scoring for subjective criteria, and fine-tuned small judges often generalize poorly beyond their training distribution. — [Yan 2024, "Evaluating the Effectiveness of LLM-Evaluators"](https://eugeneyan.com/writing/llm-evaluators/)

**How many human labels**
- Hamel Husain's "critique shadowing" (verified):
  - Use one "Principal Domain Expert."
  - Collect **binary pass/fail plus a written critique**, not 1-5 scales ("you're doing it wrong").
  - For discovery, label about 30 examples, continuing until no new failure modes appear.
  - For judge validation, aim for **about 100 examples per failure mode**, noting "Below 60, confidence intervals are often too wide."
  - Split labels into train (10-20%, used as few-shot), dev (40-45%) and test (40-45%), and run the frozen judge on test only once.
  - Report **TPR and TNR separately**, aiming for 30-50 examples of each label per split.
  - Iterate the prompt by hand (little success with prompt optimizers).
  - Re-validate after model changes.
  - In one Honeycomb case, the judge reached over 90% agreement in three iterations.
  — [Husain, "Creating a LLM-as-a-Judge That Drives Business Results"](https://hamel.dev/blog/posts/llm-judge/)

**EvalGen and "Who Validates the Validators?"**
- Shankar et al. found "criteria drift": users need criteria to grade outputs, but grading outputs changes and refines their criteria. So evaluation criteria cannot be fully specified before looking at outputs. EvalGen uses mixed-initiative alignment, with LLM-suggested criteria and assertions selected by agreement with a small number of user grades. — [Shankar et al. 2024, "Who Validates the Validators?"](https://arxiv.org/abs/2404.12272)

**Juries and ensembles**
- A Panel of LLM evaluators (PoLL) made of three smaller models from disjoint families outperformed a single GPT-4 judge in correlation with humans across six datasets. It showed less intra-model bias and cost over 7x less. — [Verga et al. 2024, "Replacing Judges with Juries"](https://arxiv.org/abs/2404.18796)
- 2026: Multi-trial aggregation is needed because single trials are noisy: about 11 repeats on average for majority vote to recover the stable verdict. The paper recommends position randomization and explicit uncertainty reporting as standard practice. (verified) — [Yagubyan 2026](https://arxiv.org/abs/2606.13685)

**Prediction-powered inference (PPI) and related statistics**
- PPI combines a small gold-labeled set with a large set of model predictions. It yields valid confidence intervals that are tighter than using gold labels alone, by using the labeled set to estimate and correct the predictor's bias. — [Angelopoulos et al. 2023, Prediction-Powered Inference (Science)](https://arxiv.org/abs/2301.09633)
- PPI++ adds a tuning parameter so the estimator is never worse than the classical gold-only estimate, even with a poor predictor. — [Angelopoulos et al. 2023, PPI++](https://arxiv.org/abs/2311.01453)
- "AutoEval Done Right" applies PPI to model evaluation with LLM or automatic raters. — [Boyeau et al. 2024](https://arxiv.org/abs/2403.07008)
- 2026: "Efficient Inference for Noisy LLM-as-a-Judge Evaluation" (ICML 2026) treats judge bias as measurement error. It compares Rogan-Gladen-style corrections (estimate judge sensitivity and specificity on the human calibration set, then correct the observed rate) with PPI and efficient-influence-function estimators, and characterizes when PPI attains strictly smaller variance. (snippet) — [arXiv 2601.05420](https://arxiv.org/abs/2601.05420); [ICML 2026 poster](https://icml.cc/virtual/2026/poster/61131)
- 2026: Filtered PPI found an LLM judge agreeing with human chatbot preferences on 68.34% of labels. It trains a filter predicting when the judge agrees with humans and asks whether more predictions always help. (snippet) — [arXiv 2602.10464](https://arxiv.org/abs/2602.10464)
- 2026: PRECISE extends PPI to ranking metrics. Augmenting 30 human annotations with LLM judgments cut the standard error of Precision@4 from 4.45 to 3.50. In a production case, 100 human labels identified the best of three variants, later confirmed by A/B test. (snippet) — [arXiv 2606.05308](https://papers.cool/arxiv/2606.05308)
- Bayesian PPI frames the trade-off as an unbiased but high-variance human sample against a lower-variance but biased judge, and offers Bayesian variants for small samples. (snippet) — [arXiv 2405.06034](https://arxiv.org/abs/2405.06034)

**Vendor guidance**
- Anthropic: "LLM-as-judge graders should be closely calibrated with human experts". Prefer deterministic graders where possible, and read transcripts to confirm graders work. (verified) — [Anthropic 2026](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Anthropic's developer docs on LLM-based grading recommend detailed, clear rubrics, empirical or specific outputs (e.g., correct/incorrect or a 1-5 scale), and having the judge reason before answering, then discarding the reasoning. — [Anthropic docs: Create strong empirical evaluations](https://docs.anthropic.com/en/docs/test-and-evaluate/develop-tests)

### Inferences
These are my own calculations and judgments, not from sources.
- **n ≈ 15 is a pilot.** With 15 items, an observed 80% agreement has a 95% Wilson CI of roughly 55%-93%. A Spearman ρ needs to exceed about 0.52 just to be significant at α = 0.05 (two-tailed). The write-up should present ~15 ratings as "calibration / sanity check with wide uncertainty," report CIs, and avoid claiming the judge is "validated."
- **Practical protocol for this project:**
  1. Collect binary per-criterion human labels plus short critiques on ~15 outputs, chosen to include clear passes and fails rather than at random, so both TPR and TNR are estimable.
  2. Hold out most of them. Use 2-3 as few-shot examples only.
  3. Report per-criterion agreement, Cohen's κ (or Krippendorff's α if scales are ordinal or there are multiple raters) and TPR/TNR, each with CIs.
  4. Run the judge 3-5 times per item and report flip rate.
  5. If the benchmark reports aggregate judge scores across many runs, a Rogan-Gladen or PPI correction using the 15 labels gives honest CIs for framework-level means.
- A cheap PoLL-style jury, for example Haiku plus a small model from another family, could reduce self-preference when Claude both writes and judges the code.
- Criteria drift implies the rubric should be finalized only *after* the human rater has looked at some outputs.

### Gaps
- I found no published guidance specifically on minimum human-label counts for judge calibration at n < 30. Husain's ~100-per-failure-mode figure is practitioner advice, not a derived power analysis.
- I did not verify the PRECISE and Filtered-PPI numbers beyond search snippets.

---

## 6. Human evaluation (expert review, pairwise preference, Elo/Bradley-Terry arenas, critiques)

### Takeaway
Crowdsourced pairwise arenas (Chatbot Arena, Copilot Arena, WebDev Arena) rank models with Bradley-Terry models fit to human votes. They are influential but have documented structural biases: private testing, unequal data access and silent deprecations. Expert, rubric-based human review (HealthBench physicians, PaperBench authors) is the gold standard for technical domains but is expensive. For a small project, one expert rater with binary criteria and critiques is the practitioner-recommended form.

### Cited Findings
- Chatbot Arena collects crowdsourced pairwise votes on anonymous model pairs and fits a Bradley-Terry model, rather than online Elo, to get stable rankings with confidence intervals. The authors report that crowd votes agree reasonably with expert raters. — [Chiang et al. 2024, Chatbot Arena](https://arxiv.org/abs/2403.04132)
- Copilot Arena gathers in-IDE pairwise preferences between code completions from different models inside VS Code during real developer work. Its rankings differed from static benchmarks, and it identified biases such as position effects in how suggestions are presented. — [Chi et al. 2025, Copilot Arena](https://arxiv.org/abs/2502.09328)
- "The Leaderboard Illusion" (Apr 2025) documents systematic issues in Chatbot Arena. Undisclosed private testing let some providers test many variants and publish only the best; Meta tested 27 private variants before the Llama 4 release. Data access was unequal: Google and OpenAI received an estimated 19.2% and 20.4% of all Arena data, while 83 open-weight models together received 29.7%. Many models were silently deprecated. Training on Arena data gave large relative gains on Arena-style evaluation, showing overfitting risk. — [Singh et al. 2025, "The Leaderboard Illusion"](https://arxiv.org/abs/2504.20879)
- PaperBench human baselines: ML PhDs attempting replications provide a human reference point for agent scores under the same rubric. — [Starace et al. 2025](https://arxiv.org/abs/2504.01848)
- Husain advises against relying on convenient proxies (e.g., developers) instead of the actual domain expert, and recommends binary verdicts with critiques over Likert scales for human raters. (verified) — [Husain](https://hamel.dev/blog/posts/llm-judge/)
- Human raters share many LLM-judge biases, such as susceptibility to authority cues and presentation. — [Chen et al. 2024](https://arxiv.org/abs/2402.10669)

### Inferences
- Arena-style Bradley-Terry ranking needs hundreds to thousands of votes, so it is not feasible for ~15 ratings. For this project, the arena literature is mainly useful as a cautionary tale about selective reporting. If frameworks are run multiple times, report all runs, not the best one.
- If the single human rater is also the project author, their ratings carry knowledge and preference bias. Blinding framework identity during rating is cheap and worth doing.

### Gaps
- I did not fetch WebDev Arena methodology or any 2026 arena critiques this session.
- I did not verify exact Copilot Arena vote counts.

---

## 7. 2025-2026 SOTA: reasoning-model judges, generative reward models, newer judge-quality benchmarks

### Takeaway
By 2026, the strongest judges are reasoning ("thinking") models or RL-trained generative judges that write a critique before the verdict. They lead on JudgeBench, RewardBench 2 and CodeJudgeBench. Meta-evaluation has shifted toward harder, correctness-grounded benchmarks and toward reliability audits: chance-corrected agreement, test-retest stability and stress-testing harnesses. Agent-as-a-Judge is consolidating as its own subfield.

### Cited Findings
**Reasoning and generative judges**
- J1 (Meta, 2025) trains "thinking" LLM judges with RL on verifiable judgment tasks and outperforms other judges of comparable size. — [Whitehouse et al. 2025, J1](https://arxiv.org/abs/2505.10320)
- RM-R1 frames reward modeling as reasoning ("Reasoning Reward Models"), with chain-of-rubrics generation before judging. — [Chen et al. 2025, RM-R1](https://arxiv.org/abs/2505.02387)
- DeepSeek-GRM introduces Self-Principled Critique Tuning: the judge generates its own principles and critiques. It shows inference-time scaling for reward models (more samples plus a meta-RM vote improves accuracy). — [Liu et al. 2025, Inference-Time Scaling for Generalist Reward Modeling](https://arxiv.org/abs/2504.02495)
- TIR-Judge (revised Feb 2026) trains judges with RL to use tool-integrated reasoning, such as code execution, during judging. It reports up to 6.4% pointwise and 7.7% pairwise gains over strong reasoning judges on RewardBench, RM-Bench, JudgeBench and RewardBench 2. (snippet) — [arXiv 2510.23038](https://arxiv.org/abs/2510.23038)
- CodeJudgeBench: thinking models beat non-thinking models on code judging, and a small thinking model (Qwen3-8B) beat trained judges up to 70B. (verified) — [Jiang et al.](https://arxiv.org/abs/2507.10535)

**Judge-quality benchmarks**
- RewardBench 2 (Jun 2025) uses new, unseen prompts and a best-of-4 format. Models scored about 20 points lower than on RewardBench 1, and scores correlate with downstream best-of-N and RL usefulness. — [Malik et al. 2025, RewardBench 2](https://arxiv.org/abs/2506.01937)
- JudgeBench (2024) is correctness-grounded and hard. GPT-4o-class judges perform near random. — [Tan et al. 2024](https://arxiv.org/abs/2410.12784)
- JETTS benchmarks judges as test-time-scaling evaluators, for reranking and beam search. — [JETTS 2025](https://arxiv.org/abs/2504.15253)
- 2026 reliability audits:
  - "Reliability without Validity" (Jun 2026; 21 judges, κ deflation of 33-41 points, ranking shifts of up to 14 places, consistency-bias paradox). (verified) — [arXiv 2606.19544](https://arxiv.org/abs/2606.19544)
  - "The Coin Flip Judge?" (Apr 2026; 13.6% pairwise flip rate). (verified) — [arXiv 2606.13685](https://arxiv.org/abs/2606.13685)
  - Judge Reliability Harness, a stress-testing framework (Mar 2026). (snippet) — [arXiv 2603.05399](https://arxiv.org/abs/2603.05399)
  - LongJudgeBench for long-form output judging (Jun 2026). (snippet) — [arXiv 2606.01629](https://arxiv.org/abs/2606.01629)
  - "LLMs Cannot Reliably Judge (Yet?)", a robustness assessment (2025). (snippet) — [arXiv 2506.09443](https://arxiv.org/abs/2506.09443)
- A 2026 survey warns that if an LLM judge supplies the training reward, evaluating with the same judge family is circular. It recommends a disjoint judge, a held-out human subset, or a verifiable proxy. (snippet; source ID unverified) — [arXiv 2608.31075](https://arxiv.org/abs/2608.31075)
- Survey on Agent-as-a-Judge (Jan 2026). (snippet) — [arXiv 2601.05111](https://arxiv.org/abs/2601.05111)

**Vendor practice (2026)**
- Anthropic's Jan 2026 guidance favors a layered stack: deterministic graders first, LLM rubric graders for nuance (one isolated judge per dimension), and periodic human calibration. It reports pass@k for "one success matters" and pass^k for consistency; for example, 75% per-trial success gives about 42% pass^3. (verified) — [Anthropic 2026](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Claude Code's /goal (2026) productizes transcript-only LLM judging as a loop-termination mechanism using a small fast model. (verified) — [Claude Code docs](https://code.claude.com/docs/en/goal)

### Inferences
- For a small project in late 2026, a reasonable judge setup is: a reasoning-capable model for the code-quality rubric; extended thinking or a "reason then verdict" prompt; binary criteria; and 3+ samples with majority vote. It need not be the largest model, since small thinking models are competitive on code judging per CodeJudgeBench.
- Judge-quality benchmark results do not transfer reliably across tasks (ranking shifts of up to 14 places). So a leaderboard cannot replace the project's own ~15-label calibration. That is the main methodological justification for doing it.

### Gaps
- I did not obtain current (Oct 2026) leaderboard numbers for RewardBench 2 or JudgeBench.
- Several 2026 papers above were seen only via search snippets, and their claims should be treated as provisional.
- I did not confirm whether OpenAI or Anthropic published new judge-specific guidance between Jul and Oct 2026 beyond the docs cited.
