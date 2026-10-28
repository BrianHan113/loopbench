# Statistical Methodology and Experimental Design for LLM / Agent Evaluations (small-sample focus)

Research date: 2026-10-28. Project design under consideration: ~4-6 workflow conditions x ~10 tasks x 2 repeats (~120 runs), binary resolved/not-resolved, paired by task, task-level bootstrap CIs plus gain/regression counts.

Note on "own computation" items: these are worked numbers I computed for this design with a pure-Python script (Wilson/Clopper-Pearson/Jeffreys intervals, exact McNemar, sign-flip permutation, percentile bootstrap, and a Monte Carlo simulation). Script: `C:/Users/brian/AppData/Local/Temp/claude/C--Users-brian-Documents-code-loopbench/17d69b30-769a-4f6a-b954-a067d3213aa5/scratchpad/calc.py`. They are derived from standard formulas, not taken from a publication. The simulation assumptions are stated where the numbers appear.

## 1. Error bars: Evan Miller, "Adding Error Bars to Evals" (Anthropic, Nov 2024). Key formulas.

### Takeaway
Miller treats an eval as an experiment that samples questions from a super-population. For an independent question set the CLT standard error is enough. Use clustered SEs when questions are grouped, a paired SE when two models answer the same questions, and K resamples per question to remove within-question noise. He recommends power analysis before running anything, and his worked examples need hundreds to ~1,000 questions to detect differences of a few points. Those formulas assume large n, and the next section shows they break down at n of about 10.

### Cited Findings
- Paper metadata: Evan Miller, arXiv:2411.00640, submitted 1 Nov 2024, 14 pp. It frames evaluations as experiments on questions drawn from an unseen super-population and gives formulas for analysing evals, comparing two models, and planning an eval. — [arXiv abstract](https://arxiv.org/abs/2411.00640)
- CLT standard error: SE_CLT = sqrt(Var(s)/n). For binary scores SE_Bernoulli = sqrt(s̄(1−s̄)/n). 95% CI = s̄ ± 1.96·SE. Miller says bootstrapping is unnecessary unless the sampling scheme or estimator is complicated. — [arXiv HTML](https://arxiv.org/html/2411.00640v1)
- Clustered SE: SE_clustered = ( SE_CLT² + (1/n²) Σ_c Σ_i Σ_{j≠i} (s_{i,c} − s̄)(s_{j,c} − s̄) )^{1/2}. It can be more than 3x the naive SE. On DROP the clustered SE was 1.34 vs a naive 0.44. — [arXiv HTML](https://arxiv.org/html/2411.00640v1)
- Variance decomposition: Var(μ̂) = (Var(x) + E[σ_i²])/n, where x is the per-question true score and σ_i² is the within-question (sampling) variance. With K samples per question the within-question term becomes σ_i²/K. Extra K stops helping once E[σ_i²]/K ≪ Var(x). Worked example with uniform difficulty: K=2 cuts variance by 1/3, K=4 by 1/2, K=6 by 5/9, and the limit is 2/3. Do not compute an SE as if all K·N answers were independent. — [arXiv HTML](https://arxiv.org/html/2411.00640v1)
- Paired comparison: Var(μ̂_paired) = (Var(s_A) + Var(s_B) − 2·Cov(s_A, s_B))/n. From summary statistics, SE_{A−B,paired} = sqrt(SE_A² + SE_B² − 2·SE_A·SE_B·Corr(s_A, s_B)). There is also a clustered paired version. The test statistic is z = μ̂_{A−B}/SE_{A−B}. — [arXiv HTML](https://arxiv.org/html/2411.00640v1)
- Power and minimum detectable effect: n = (z_{α/2} + z_β)² (ω² + σ_A²/K_A + σ_B²/K_B)/δ², and MDE δ = (z_{α/2} + z_β)·sqrt((ω² + σ_A²/K_A + σ_B²/K_B)/n), with ω² = Var(x_A) + Var(x_B) − 2Cov(x_A, x_B). Worked numbers: with ω² = 1/9, δ = 3pp, α = 0.05 and power 0.8, you need about 969 questions. At n = 198, raising K from 1 to 10 cuts the MDE from 13.2% to 7.5%. — [arXiv HTML](https://arxiv.org/html/2411.00640v1)
- Recommendations: report the SE beneath each mean, report n and the number of clusters, use paired differences and report the pairwise difference, its SE and CI and the score correlation, do not lower temperature just to cut variance (at temperature 0 on a uniform true/false eval the minimum variance rises from 1/12 to 1/4), run a power analysis first, and build new evals with roughly 1,000 or more questions. — [arXiv HTML](https://arxiv.org/html/2411.00640v1)

### Inferences
- For this project the cluster is the task. With 2 repeats per task, the per-task resolved fraction (0, 0.5 or 1) is the unit of analysis, and n = 10, not 20. Pooling 20 runs as if they were independent would understate the SE, which is exactly the error Miller warns about.
- Miller's paired formula is the right structure, but its z-based CI relies on the CLT. With n = 10 tasks, use a t critical value (2.262 for 9 df) or an exact or permutation method instead (see section 2).
- Own computation using Miller's decomposition, with task difficulty logit ~ N(0, 1.5) giving Var(x) ≈ 0.073 and E[σ²] ≈ 0.177. The SE of one condition's resolve rate is 0.158 at 10 tasks x 1 run, 0.127 at 10x2, 0.108 at 10x4 and 0.095 at 10x10, while 20x1 gives 0.112 and 40x1 gives 0.079. The 95% half-width at 10x2 is about ±25pp. Doubling tasks beats doubling repeats at the same run budget (20x1 = 0.112 vs 10x2 = 0.127).

### Gaps
- Miller gives no guidance specific to n ≈ 10 clusters. His examples are all in the hundreds.

## 2. Small-sample critiques and alternatives: Bowyer et al. 2025, Bayesian / Beta-Binomial, Wilson / Clopper-Pearson, exact paired tests, bootstrap at n ≈ 10, hierarchical models

### Takeaway
Below a few hundred items, CLT intervals, and also bootstrap intervals, are too narrow. Use Wilson or Beta-posterior intervals for single rates. Use exact McNemar / sign tests, sign-flip permutation tests, or paired Bayesian models for comparisons. Use hierarchical Bayesian models when items are clustered, for example tasks containing repeats. At n = 10 tasks, a percentile bootstrap of the task-level paired difference under-covers: about 87-92% actual coverage for a nominal 95% in my simulation, and about 8% false positives at nominal 5%.

### Cited Findings
- Bowyer, Aitchison and Ivanova, "Position: Don't use the Central Limit Theorem in LLM evals with fewer than a few hundred datapoints", arXiv:2503.01747, ICML 2025 Spotlight Position Paper (v1 3 Mar 2025, v3 28 May 2025). They argue that CLT methods suit benchmarks with thousands of examples but usually underestimate uncertainty on small benchmarks. They provide the Python library bayes_evals at github.com/sambowyer/bayes_evals. — [arXiv abstract](https://arxiv.org/abs/2503.01747)
- Bowyer et al. recommend the Wilson score interval or a Beta-Bernoulli Bayesian credible interval for a single model ("we would recommend using WS or Bayesian intervals in practice"). Clopper-Pearson is exact but "overly conservative (too wide) in practice." CLT intervals can collapse to zero width or extend outside [0,1]. At N = 100, 95% CLT intervals cover only 92.5% of the time. — [arXiv HTML](https://arxiv.org/html/2503.01747)
- Bootstrap intervals (10,000 resamples) "also perform poorly" at small N and under-cover. — [arXiv HTML](https://arxiv.org/html/2503.01747)
- Paired comparisons (same questions): "We would recommend using the paired Bayes method as it can account for correlations"; it usually gives narrower intervals. Unpaired Bayes is "a reasonable and easier to implement alternative." Bayesian output allows direct statements like P(θ_A > θ_B | data). For independent models Fisher's exact test is valid but overly conservative. — [arXiv HTML](https://arxiv.org/html/2503.01747)
- Clustered questions: Miller's clustered SE, a CLT adjustment, still fails at small N. Only a Bayesian clustered (hierarchical Beta-Binomial) model achieved correct coverage. — [arXiv HTML](https://arxiv.org/html/2503.01747)
- HiBayES (Luettgau, Coppock, Dubois, Summerfield, Ududec; arXiv:2505.05602, v3 Jul 2025) is a hierarchical Bayesian GLM framework for AI eval statistics. It claims robust inference "particularly in low-data scenarios (e.g., < 20 data points per evaluation)" for both QA and agentic evals, and ships a beta software package. — [arXiv abstract](https://arxiv.org/abs/2505.05602)
- A Bayesian hierarchical model with embedding-space clustering targets settings with limited evaluation data and non-independent prompts. It argues that common metrics misstate performance and uncertainty because they assume classical-inference sample sizes and independent prompts (arXiv:2510.05709, v2 June 2026). This is from a search snippet; I did not read the full paper. — [arXiv](https://arxiv.org/html/2510.05709)
- The exact McNemar test is used in practice for paired agent comparisons on shared instances. "Coding Agents Have Converged" (Sep 2026 preprint) applies exact McNemar to adjacent SWE-bench Verified pairs with Holm correction, and uses a paired bootstrap over instances for interaction tests. Its significance condition on discordant counts b, c is |b − c| ≳ 1.96·sqrt(b + c). — [arXiv HTML 2609.17394](https://arxiv.org/html/2609.17394v1)
- Own computation, 95% intervals for a single condition:
  - 7/10: Wald [0.42, 0.98], Wilson [0.40, 0.89], Clopper-Pearson [0.35, 0.93], Jeffreys [0.39, 0.91], Beta(1,1) posterior [0.39, 0.89].
  - 5/10: Wilson [0.24, 0.76] (width 0.53).
  - 10/10: Wald [1.00, 1.00] (degenerate), Wilson [0.72, 1.00], Clopper-Pearson [0.69, 1.00].
  - 14/20 (which would only be valid if the 20 runs were independent tasks): Wilson [0.48, 0.85].
- Own computation, exact McNemar / sign test (two-sided) on discordant task counts, gains b vs regressions c. 3-0 gives p = 0.25, 4-0 p = 0.125, 5-0 p = 0.0625, 6-0 p = 0.031, 8-1 p = 0.039, 9-1 p = 0.022, 10-0 p = 0.002. With zero regressions you need at least 6 discordant tasks for p < 0.05, and at least 9 for p < 0.005 (Bonferroni over 10 comparisons).
- Own computation, three illustrative 10-task x 2-repeat datasets (task-level difference in {−1, −0.5, 0, 0.5, 1}):

  | Dataset | Mean difference | Percentile bootstrap | t-interval | Sign-flip permutation p |
  |---|---|---|---|---|
  | 4 tasks improved, 0 regressed | +30pp | [+0.05, +0.55] | [0.00, +0.60] | 0.125 |
  | Mixed gains and one regression | +20pp | [−0.05, +0.45] | [−0.10, +0.50] | 0.31 |
  | Mixed | +10pp | [−0.10, +0.30] | — | 0.63 |

  At n = 10, the bootstrap can call a result significant when the exact test does not.
- Own computation, Monte Carlo simulation. Setup: 600 replicate experiments per setting, 10 tasks x 2 repeats, task logit difficulty ~ N(0, 1.5), and a constant logit shift for the better condition. Percentile bootstrap coverage of the true average effect was 0.87-0.92 against nominal 0.95. Its false-positive rate under the null was 0.080, against 0.047 for the t-interval and 0.012 for the sign-flip permutation test (conservative because of discreteness and ties).

### Inferences
- The planned "task-level bootstrap CI" is a reasonable descriptive tool, but at 10 clusters it is anti-conservative. Report it next to an exact sign-flip permutation p-value (or exact McNemar on the per-task majority outcome) and/or a paired Bayesian posterior. A t-interval on the 10 task-level differences was close to nominal in my simulation. Alternatively, report BCa or studentized bootstrap intervals and say plainly that they are approximate.
- Gain/regression counts map directly onto exact McNemar / sign tests. That is the most transparent inferential statement this design supports, for example "5 tasks gained, 0 regressed, exact p = 0.0625."
- A hierarchical Beta-Binomial or logistic mixed model (task random effect, condition fixed effect, repeats nested within task) is the statistically principled model for 10 tasks x 2 repeats x conditions. HiBayES or bayes_evals are the off-the-shelf options. With 10 tasks the posterior will be prior-sensitive, so the prior should be reported.

### Gaps
- I did not read Bowyer et al.'s coverage plots at N = 10 or N = 50; those numbers appear only in figures. I also did not confirm exactly how the paired Bayes model is parameterised.
- No source located that evaluates BCa or studentized bootstrap specifically for ~10-cluster agent evals.

## 3. Variance sources: run-to-run nondeterminism, seeds, prompt sensitivity, infrastructure noise, Madaan et al. 2024, agentic repeat variance

### Takeaway
In agentic coding evals, run-to-run variance is large even at temperature 0, with single-run SWE-bench Verified pass@1 swinging 2-6pp on hundreds of tasks. Infrastructure configuration alone can shift scores by about 6pp. Differences under about 3pp are untrustworthy on 500-task benchmarks without matched configurations and repeats. On 10 tasks that noise floor is far higher.

### Cited Findings
- "On Randomness in Agentic Evals" (Bjarnason, Silva, Monperrus; arXiv:2602.07150, Feb 2026, v3 Mar 2026). About 60,000 SWE-bench Verified trajectories across 3 models and 2 scaffolds. Single-run pass@1 varied by 2.2-6.0pp depending on which run was chosen. SDs exceeded 1.5pp even at temperature 0. Trajectories diverge early, within the first few percent of tokens. Reported gains of 2-3pp may be noise. Recommendations: estimate pass@1 from multiple runs per task, use power analysis to choose the number of runs, and report pass@k (optimistic) and pass^k (pessimistic) with k > 1. — [arXiv abstract](https://arxiv.org/abs/2602.07150)
- Anthropic, "Quantifying infrastructure noise in agentic coding evals" (Gian Segato, 5 Feb 2026):
  - Terminal-Bench 2.0 under six resource configurations: the most- vs least-resourced gap was 6pp (p < 0.01).
  - Infra error rates were 5.8% at strict 1x limits, 2.1% at 3x and 0.5% uncapped.
  - Between 1x and 3x, success stayed within noise (p = 0.40).
  - SWE-bench (227 problems x 10 samples) rose 1.54pp from 1x to 5x RAM.
  - Recommendations: treat resource configuration as an experimental variable, report it, run public evals multiple times across multiple days (time-of-day fluctuation observed anecdotally), and "treat leaderboard gaps below 3 pp with skepticism" because naive binomial CIs already span 1-2pp.
  - — [Anthropic Engineering](https://www.anthropic.com/engineering/infrastructure-noise)
- Madaan et al., "Quantifying Variance in Evaluation Benchmarks" (Meta and others; arXiv:2406.10229, June 2024). Defines seed variance and monotonicity metrics and estimates them across many models. Framing MMLU-style choice tasks as completion tasks can reduce variance for ~7B models. Item analysis and IRT "struggle to meaningfully reduce variance." — [arXiv abstract](https://arxiv.org/abs/2406.10229)
- Repeated-run practices in 2026 coding-agent papers: SWE-Doctor used 5 trials (SD 2.0 points) and reported Pass@5 / All@5. SWE-Serve used 3 runs with a normal-approximation 95% CI. Terminal-Bench 2.1 reportedly requires at least 5 public trials per task. These come from a search summary and are secondary. — [SWE-Doctor](https://arxiv.org/pdf/2607.00990); [SWE-Serve](https://arxiv.org/pdf/2609.26777); [search-summary source list incl. DEV post](https://dev.to/devinios/same-model-same-scaffold-10-runs-agent-benchmark-scores-still-move-by-up-to-6-points-gmb)
- AgentLens reports that repeated-run variance concentrates in a small set of unstable tasks rather than spreading uniformly (search snippet, not read in full). — [arXiv 2607.06624](https://arxiv.org/pdf/2607.06624)
- Within-model scaffold ranges on SWE-bench Verified reach 29.8pp (claude-3-5-sonnet), against an 8.8pp spread across the top thirty systems. Scaffold or harness choice is a first-order variance source. — [arXiv HTML 2609.17394](https://arxiv.org/html/2609.17394v1)
- Prompt dependence: arXiv:2510.05709 argues that test prompts are non-independent and builds a hierarchical model to correct for it (snippet-level). — [arXiv](https://arxiv.org/html/2510.05709)
- METR builds time-horizon CIs with a hierarchical bootstrap over task families, then tasks, then attempts (about 8 attempts per task per model per Epoch). Its intervals are often a factor of about 2 in each direction, and task-distribution uncertainty dominates. — [METR note, 20 Mar 2026](https://metr.org/notes/2026-03-20-impact-of-modelling-assumptions-on-time-horizon-results/); [Epoch AI](https://epoch.ai/data/metr-time-horizons)

### Inferences
- With 2 repeats, the project can estimate per-task flakiness: a task that splits 1/2 under the same condition is unstable. A gain on a task where the baseline was 0/2 and the treatment 2/2 is far stronger evidence than 1/2 to 2/2. Per AgentLens, unstable tasks dominate the noise.
- Infrastructure (container resources, timeouts, API rate limits, time of day) must be held constant across conditions. Runs should be interleaved or randomised in order, not run condition-by-condition, so that drift such as API changes or time-of-day load is not confounded with condition. Infra failures should be logged separately from model failures, with a pre-declared scoring rule for them.
- If conditions vary the scaffold or harness, the 29.8pp within-model scaffold range shows that harness effects can be larger than model effects. That is plausible for workflow comparisons too, but no source tested it at this scale.

### Gaps
- No primary source I found quantifies run-to-run variance specifically for ~10-task custom suites or for Claude Code-style workflows.
- I could not verify Terminal-Bench 2.1's 5-trial requirement against a primary source.

## 4. Minimum detectable effect and power for paired binary outcomes with few tasks; repeats vs tasks

### Takeaway
With 10 tasks x 2 repeats, the project can only reliably detect very large effects, roughly 30pp or more in resolve rate and concentrated on several tasks. A true ~17pp effect would be detected only about 10-27% of the time. Adding tasks buys much more precision than adding repeats once K ≥ 2.

### Cited Findings
- MDE scaling from the 2026 coding-agent literature, using an independent-sample calculation at 80% power and a 65% baseline: about 23pp at n = 50, 19pp at n = 80, 17pp at n = 100 and 8.2pp at n = 500. Pairing shrinks requirements a lot when discordance is low. The paired requirement is |b − c| ≳ 1.96·sqrt(b + c), and the scale factor on n at fixed discordance rates is k ≥ 1.96²(b + c)/(b − c)². — [arXiv HTML 2609.17394](https://arxiv.org/html/2609.17394v1)
- Miller's MDE formula δ = (z_{α/2} + z_β)·sqrt((ω² + σ_A²/K_A + σ_B²/K_B)/n). Increasing K only shrinks the within-task terms σ²/K, so returns diminish once those are small relative to ω². — [arXiv HTML](https://arxiv.org/html/2411.00640v1)
- Bjarnason et al. recommend using power analysis to set the number of runs needed to detect expected effect sizes. — [arXiv abstract](https://arxiv.org/abs/2602.07150)
- Own computation, simulated power for a paired comparison at α = 0.05. Setup: task logit difficulty ~ N(0, 1.5), a constant logit shift, 600 replicates.

  | Design | True average effect | t-test power | Sign-flip permutation power | Percentile bootstrap "power" (inflated by under-coverage) |
  |---|---|---|---|---|
  | 10 tasks x 2 | +9pp | 0.07 | 0.02 | 0.09 |
  | 10 tasks x 2 | +17pp | 0.21 | 0.10 | 0.27 |
  | 10 tasks x 2 | +25pp | 0.35 | 0.20 | 0.47 |
  | 10 tasks x 2 | +31pp | 0.54 | 0.31 | 0.67 |
  | 10 tasks x 4 | +17pp | 0.32 | 0.21 | — |
  | 20 tasks x 2 | +17pp | 0.43 | — | — |
  | 20 tasks x 1 | +17pp | 0.21 | — | — |
  | 40 tasks x 1 | +17pp | 0.49 | — | — |

  Median percentile-bootstrap CI width for the paired difference was about 0.45-0.50 at 10x2, 0.33 at 10x4 and 0.35 at 20x2 or 40x1.
- Own computation: with the exact sign/McNemar test, p < 0.05 needs at least 6 net-discordant tasks out of 10 with no regressions, which is a ≥60% task-level flip rate.

### Inferences
- What the project can conclude:
  - It can describe per-condition resolve rates with honest, wide intervals (about ±25pp).
  - It can name which specific tasks flipped and whether flips are consistent across both repeats.
  - It can detect very large effects, for example a condition that solves 6+ tasks the baseline never solves.
  - It can generate hypotheses for a later, larger confirmatory test.
- What it cannot conclude:
  - That conditions with observed differences of ≤ 20pp differ.
  - That conditions are "equivalent." Non-significance is not equivalence (noted in [arXiv 2609.17394](https://arxiv.org/html/2609.17394v1)).
  - Any fine-grained ranking of 4-6 conditions.
  - That results generalise beyond the 10 tasks. The task sample is the dominant uncertainty, as with METR, so 10 tasks may not represent the task distribution.
- Allocation advice at a fixed budget of about 120 runs: more tasks with fewer repeats beats fewer tasks with more repeats (20x1 matched 10x2 in power at the same run count, and 40x1 roughly matched 20x2). Keeping K = 2 is still useful for estimating flakiness and pass^2. A reasonable alternative is fewer conditions (2-3 confirmatory) x 20 tasks x 2 repeats.
- An equivalence or "no meaningful regression" claim would need a pre-declared margin and a TOST-style test. At n = 10 that is essentially unachievable for any margin under about 30pp.

### Gaps
- The simulation assumes a constant logit-scale effect and normal task heterogeneity. Real coding tasks are often bimodal (always-pass or always-fail), which lowers the effective n further. The n_eff concept in [arXiv 2609.17394](https://arxiv.org/html/2609.17394v1) captures this.
- No published power table specifically for n ≈ 10 paired agent tasks was found. The numbers here are my own.

## 5. Multiple comparisons, pre-registration, exploratory vs confirmatory analysis, garden of forking paths

### Takeaway
With 5-6 conditions there are 10-15 pairwise comparisons. Testing all of them at α = 0.05 gives a 40-54% chance of at least one false "significant" result under the global null, if the tests were independent. Declare the comparison family in advance (for example, each condition vs baseline only), correct with Holm, and label everything else exploratory.

### Cited Findings
- 2026 practice: report raw and Holm-adjusted p-values over a declared comparison family, report sensitivity to grouping rules, and pre-register how missing verdicts or infrastructure failures are scored, reporting their count. — [arXiv HTML 2609.17394](https://arxiv.org/html/2609.17394v1)
- ABC (Agentic Benchmark Checklist) reporting item R.10 calls for measures of statistical significance, and R.8-R.9 call for quantitative evidence of how flaws affect results, using sampling and uncertainty quantification. — [arXiv HTML 2507.02825](https://arxiv.org/html/2507.02825v5)
- Own computation: the family-wise false-positive probability at α = 0.05 is 0.19 for m = 4 tests, 0.23 for m = 5, 0.40 for m = 10 and 0.54 for m = 15 (independence approximation). The Bonferroni per-test α for 10 comparisons is 0.005. With 10 tasks the smallest attainable two-sided exact sign-flip p is 2/1024 ≈ 0.002, so a Bonferroni-corrected significant result needs ≥9 of 10 tasks moving in the same direction (9-0 split, p = 0.0039).

### Inferences
- A concrete plan for this project:
  1. Before running, write down the primary contrast or contrasts, for example the best-guess treatment vs baseline, plus the outcome definition, the infra-failure rule, the test (exact sign-flip or McNemar) and α.
  2. Treat the full condition x condition matrix as exploratory and descriptive.
  3. Use Holm across the declared family.
- Forking paths in eval design include choosing which tasks to keep after seeing results, redefining "resolved," re-running only failed runs, dropping flaky tasks after the fact, and choosing between bootstrap and exact tests after seeing which is significant. Each needs to be fixed in advance or reported.
- Because the project's ~10 tasks will likely also serve as the dev set for tuning, any comparison on them is exploratory by construction (see section 7).

### Gaps
- I did not fetch a primary source on Gelman and Loken's "garden of forking paths" or on pre-registration specifically for LLM evals this session. The concept is standard statistics, but it is not cited here with a verified URL.

## 6. Benchmark efficiency: IRT, tinyBenchmarks, anchor points, metabench, and what they say about small task subsets

### Takeaway
Small curated subsets (about 100 items) can reproduce full-benchmark scores within a few points. That works only because the selection is calibrated on response data from hundreds to thousands of models over a large item pool. These methods do not justify inference from 10 uncalibrated tasks, and IRT does not reduce seed or run variance.

### Cited Findings
- tinyBenchmarks (Maia Polo, Weber, Choshen, Sun, Xu, Yurochkin; ICML 2024, arXiv:2402.14992): 100 curated examples suffice to estimate a model's MMLU (14K items) performance accurately. They release tiny versions of the Open LLM Leaderboard, MMLU, HELM and AlpacaEval 2.0. — [arXiv abstract](https://arxiv.org/abs/2402.14992)
- metabench (Kipnis, Voudouris, Schulze Buschoff, Schulz; ICLR 2025, arXiv:2407.12844): distils six benchmarks with 28,632 items to under 3% of their size using data from more than 5,000 LLMs. It reconstructs individual benchmark scores with about 1.24% RMSE and the total with about 0.58% RMSE, and a single common factor correlates with the total at Spearman r = 0.94. — [arXiv abstract](https://arxiv.org/abs/2407.12844)
- IRT and item analysis struggle to meaningfully reduce variance (Madaan et al. 2024). — [arXiv abstract](https://arxiv.org/abs/2406.10229)
- DeltaSelect (Conn, arXiv:2609.19607, Sep 2026) is designed for affordable A/B testing of coding agents during development. It selects tasks whose single-run results track full-benchmark performance (Pearson correlation), calibrates fractional verifier scores by regression, and fits a dollar budget. Only 22 of 113 DeepSWE tasks (19.5%) had a 5th-percentile correlation ≥ 0.50 with full-benchmark performance. Its case study is small: 13 evaluations at USD 27.86 total, with an adopted version 58.1% cheaper (p = 0.008). It is intended for dev signals, not rankings. — [arXiv abstract](https://arxiv.org/abs/2609.19607)
- Effective size n_eff counts the instances that are neither all-pass nor all-fail across the systems compared. Only the 164 of 500 SWE-bench Verified instances that separate the top ten entries carry ranking information. — [arXiv HTML 2609.17394](https://arxiv.org/html/2609.17394v1)

### Inferences
- The DeltaSelect finding that only about 20% of tasks are reliably predictive in single runs suggests that an arbitrary 10-task subset will be dominated by noisy or uninformative tasks. Tasks that every condition always passes or always fails contribute nothing to paired comparisons. The project should pilot-screen tasks to remove these before the main run, and report n_eff.
- Choosing tasks of intermediate difficulty for the conditions studied (near a 50% baseline) maximises information per task, which is the IRT intuition. The selection must be done before comparing conditions, or it becomes a forking path.

### Gaps
- I did not fetch the anchor-points paper (Vivek et al., 2023) or tinyBenchmarks' full error figures and distribution-shift caveats, so those details are unverified here.

## 7. Overfitting and adaptive data analysis: test-set reuse, holdouts, Goodhart effects in an "auto-research" loop, reusable holdout and Ladder

### Takeaway
Every time an optimisation loop looks at a score and keeps or rejects a change, it leaks information about the evaluation set. With 10 tasks, an auto-research loop will overfit the dev tasks quickly. A disjoint held-out split must gate changes and must be queried rarely, ideally through a threshold or noise mechanism like Thresholdout or the Ladder. 2026 agent self-improvement work finds that strict held-out gating matters more than optimiser sophistication.

### Cited Findings
- Dwork, Feldman, Hardt, Pitassi, Reingold and Roth, "Generalization in Adaptive Data Analysis and Holdout Reuse" (NeurIPS 2015, arXiv:1506.02629). Adaptive reuse of a holdout "can easily lead to overfitting to the holdout set itself." Thresholdout accesses the holdout only through a differentially private mechanism, which allows many adaptively chosen validations while provably avoiding overfitting. — [arXiv](https://arxiv.org/abs/1506.02629); [NeurIPS paper](https://papers.neurips.cc/paper_files/paper/2015/file/bad5f33780c42f2588878a9d07405083-Paper.pdf)
- Blum and Hardt, "The Ladder: A Reliable Leaderboard for Machine Learning Competitions" (arXiv:1502.04585, Feb 2015). It defines leaderboard accuracy under fully adaptive submissions, gives strong theoretical guarantees, resists practical adversarial attacks, and comes with a parameter-free variant. — [arXiv abstract](https://arxiv.org/abs/1502.04585)
- RSEA, "Recursive Self-Evolving Agents via Held-Out Selection" (Nguyen, Nguyen, Vuong; arXiv:2606.28374, Jun 2026). Changes to an agent's strategy, skills and playbook are committed only if they do not regress on a disjoint held-out split ("monotone-safe"). RSEA reached 69.3% single-pass on ALFWorld vs ReAct's 64.6% (McNemar p = 0.015). Unguarded Dynamic Cheatsheet scored well on ALFWorld (70.7%) but collapsed on WebShop (0.14 vs ReAct's 0.43). The search snippet summarises the claim as "the strictness of held-out selection – not the sophistication of the optimizer – governs whether evolution helps or hurts." — [arXiv abstract](https://arxiv.org/abs/2606.28374)
- Practitioner guidance from 2026 vendor blogs (secondary, lower quality):
  - The train vs validation gap measures optimiser overfitting, and a gap of more than a couple of points is a warning sign.
  - Change one thing per round and keep it only if the working score improves and the held-out score holds.
  - A held-out set made only of easy cases will pass every change.
  - Scenarios you tune against are effectively training data.
  - GEPA-style optimisers risk holdout overfitting and evaluator contamination.
  - — [FutureAGI](https://futureagi.com/blog/automated-optimization-for-agent-2026/); [LangWatch](https://langwatch.ai/scenario/best-practices/improving-your-agent/); [Tangle](https://tangle.tools/blog/self-improving-stack-prompt-optimization/)

### Inferences
- For a loop that tunes a framework against dev tasks:
  1. Split tasks into dev (seen by the loop), a gate holdout (queried only for accept/reject, with coarse or thresholded feedback such as "improved by ≥ 2 tasks: yes/no", as in the Ladder or Thresholdout), and a final test set touched once at the end.
  2. Budget the number of gate queries.
  3. Log every query, because each one consumes holdout validity.
- With 10 tasks there is barely enough for one split, let alone three. Either generate or collect more tasks (even 20-30 cheap ones for dev) or accept that any reported gain on reused tasks is optimistically biased and exploratory.
- Goodhart risk in agentic coding: the loop can learn harness-specific exploits, such as gaming weak tests. This matches ABC's finding that weak test suites (SWE-bench Verified's "insufficient test cases") let agents score without solving ([arXiv 2507.02825](https://arxiv.org/abs/2507.02825)). Final-test verification should use stronger or hidden tests than the dev loop sees.
- A no-regression gate on a 10-task holdout is weak: an exact test cannot detect regressions smaller than several tasks. Combine it with a minimum-improvement margin (an inference from the sections 2 and 4 numbers).

### Gaps
- I did not verify the Ladder's specific update rule (release a new score only if it beats the previous best by a threshold) or its error-rate bound against the full paper this session.
- I did not fetch empirical studies of adaptive overfitting in practice (for example Kaggle meta-analyses or ImageNet replication studies), which suggest the effect is often smaller than worst-case theory.

## 8. Contamination detection and mitigation: private and held-out tasks, temporal cutoffs, LiveBench-style rotating sets

### Takeaway
Public coding benchmarks are now demonstrably contaminated: OpenAI stopped reporting SWE-bench Verified in Feb 2026, and Epoch AI rates it "Flawed". Mitigations are private or held-out tasks, tasks created after the model's training cutoff, and rotating refreshed sets like LiveBench.

### Cited Findings
- OpenAI stopped evaluating on SWE-bench Verified (post dated 23 Feb 2026), citing increasing contamination: its repositories are widely used in training, so scores increasingly reflect training exposure. OpenAI recommended SWE-bench Pro instead. This comes from search summaries because the primary page returned HTTP 403 to my fetch. — [OpenAI](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/); [Gigazine coverage](https://gigazine.net/gsc_news/en/20260429-swe-bench-verified)
- Epoch AI rates SWE-bench Verified "Flawed," saying it no longer measures the stated capability because of contamination (reviewed Sep 2026, per search snippet). — [Epoch AI benchmark review](https://epoch.ai/benchmarks/swe-bench-verified/review)
- A May 2026 Datacurve audit reportedly found that SWE-bench Pro graders mis-graded about a third of trials. This is unconfirmed and secondary. — [AI Weekly](https://aiweekly.co/alerts/openai-drops-swe-bench-verified-backs-pro-after-flaw-audit)
- LiveBench (White et al., ICLR 2025 Spotlight, arXiv:2406.19314) resists contamination by drawing questions from recent sources (math competitions, arXiv, news, datasets), adding and updating questions monthly, and scoring against objective ground truth without LLM judges. It covers 6 categories. — [arXiv abstract](https://arxiv.org/abs/2406.19314)
- ABC reporting items R.3-R.4 require measures to prevent data contamination. — [arXiv HTML 2507.02825](https://arxiv.org/html/2507.02825v5)

### Inferences
- For this project, contamination matters mostly as a confound shared by all conditions (same model). It is second-order for a paired workflow comparison, but it does inflate absolute resolve rates. Tasks drawn from public repos or issues before the model's cutoff may be memorised, which would compress differences between workflows toward a ceiling.
- Prefer tasks created after the model's training cutoff, or private or synthetic tasks, and record each task's creation date relative to the cutoff.

### Gaps
- I did not obtain specific contamination-detection methods (for example memorisation probes or gold-patch reproduction rates) from a primary source this session.

## 9. Reporting standards: what good eval reports include (HELM, benchmark reporting checklists, model cards, ABC reporting items)

### Takeaway
A good small-sample eval report states everything needed to judge noise and validity. That means n tasks and n repeats, clustering, the interval method, paired differences with exact p-values, multiplicity handling, infrastructure and harness configuration, failure-scoring rules, contamination controls, baselines, known limitations and released artifacts.

### Cited Findings
- ABC (Zhu, Jin et al., incl. Liang, Stoica, Zaharia, Steinhardt and Kang; arXiv:2507.02825, Jul 2025) has 42 items: 10 on task validity, 19 on outcome validity and 13 on reporting (R.1-R.13).
  - R.1-R.2: open-source the data and harness.
  - R.3-R.4: contamination prevention.
  - R.5-R.6: state the capabilities measured and construct validity.
  - R.7: document mitigation of validity limitations.
  - R.8-R.9: qualitative and quantitative impact of known flaws.
  - R.10: report statistical significance.
  - R.11: interpretation guidance.
  - R.12-R.13: baselines, both non-AI or human and trivial agents.
  - Benchmark flaws can mis-estimate performance by up to 100% in relative terms (TAU-bench counted empty responses as successes; SWE-bench Verified has insufficient tests). Applying ABC to CVE-Bench cut overestimation by 33%.
  - — [arXiv abstract](https://arxiv.org/abs/2507.02825); [arXiv HTML](https://arxiv.org/html/2507.02825v5)
- Miller: report the SE beneath means, n and the number of clusters, paired differences with SE, CI and correlation. — [arXiv HTML](https://arxiv.org/html/2411.00640v1)
- Anthropic infra-noise post: report resource configuration (guaranteed allocation and kill threshold) and enforcement method as experimental variables. — [Anthropic Engineering](https://www.anthropic.com/engineering/infrastructure-noise)
- "Coding Agents Have Converged":
  - Report tiers or rank intervals instead of strict ranks, n_eff and the nesting coefficient.
  - Report machine-readable model-scaffold pairs, attempts and harness modifications.
  - Report per-subset results and grader-structured outputs rather than agent self-reports.
  - Report the count of missing verdicts.
  - — [arXiv HTML 2609.17394](https://arxiv.org/html/2609.17394v1)
- Bjarnason et al.: report multi-run pass@1 together with pass@k and pass^k. — [arXiv abstract](https://arxiv.org/abs/2602.07150)

### Inferences
- A reporting checklist for this project, synthesised from the above:
  1. Task list with source, creation date and difficulty screening.
  2. Number of tasks and repeats, with the task stated as the unit of inference.
  3. Per-condition resolve rate with a Wilson or Beta interval on task-level rates.
  4. Paired differences vs the pre-declared baseline: gain and regression counts, exact McNemar or sign-flip p-value, and the bootstrap or t CI flagged as approximate.
  5. Holm-adjusted p-values for the declared family, with the rest labelled exploratory.
  6. Per-task outcome matrix (condition x task x repeat) and a flakiness count.
  7. pass@2 and pass^2.
  8. Infrastructure, harness and model version, dates, and the infra-failure rule and count.
  9. Which tasks were used for any tuning loop (dev vs held-out).
  10. A limitations section stating the MDE (about 30pp) and that non-significance is not equivalence.

### Gaps
- I did not fetch HELM (Liang et al., 2022) or model-card or "benchmark reporting checklist" primary sources this session, so their specific reporting items are not cited here.
