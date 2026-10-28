# Benchmarking AI Coding Workflow Frameworks — Lean Learning Project

Oct 28, 2026 · @Brian

## Summary

A small, subscription-only benchmark of AI coding workflow frameworks, built to learn evaluation methods and AI-assisted development rather than to publish results. Four conditions (Claude Code's built-ins used well as the baseline, Superpowers, one minimal framework and your own) run on 12–15 small private tasks with hidden tests: about 120 runs, all on a Claude Code subscription.

**Learning goals**

- **Benchmark design:** tasks with hidden tests, a dev/test split, a committed plan, paired comparisons.
- **Evaluation methods:** exact paired tests and confidence intervals, gain/regression analysis, task validation, reward-hacking audits, LLM-judge calibration, and seeing first-hand how much noise small samples carry.
- **AI-assisted development:** building a real harness with an agent workflow and hooks, and designing your own lightweight framework.

**What this version gives up:** statistical power and generality. With about 120 runs, most differences will sit inside wide confidence intervals. Treat results as directional, and treat the width of those intervals as one of the lessons.

**Evaluation grounding:** [LITERATURE_REVIEW.md](LITERATURE_REVIEW.md) surveys evaluation methods for AI-generated code, agents and RAG systems (metrics, deterministic grading, LLM judges, statistics, reward hacking, harnesses), and the design below follows its recommendations. The headline for this project: at ~10 test tasks × 2 repeats, the design detects a true ~30-point pass-rate difference only about half the time, so rigour has to come from task validity, isolation, hacking audits and honest exact tests rather than sample size. The most original thing it can measure is how often each enforcement mechanism accepts work the hidden tests reject.

## Motivation

Generation is cheap; knowing whether generated code is right is not. Three findings make this project worth doing now.

1. **Verification is the bottleneck.** In GitLab's 2026 AI Accountability Report (Harris Poll, 1,528 respondents), 85% agree AI moved the bottleneck from writing code to reviewing and validating it. Sonar's survey of 1,100+ developers found 96% do not fully trust AI code, yet only 48% always verify it before committing. ([devops.com](https://devops.com/?p=188681))
2. **The harness around a model matters as much as the model.** On Terminal-Bench 2.0, Claude Opus 4.6 scored 58.0% inside Claude Code's harness and 74.7% inside Terminus-KIRA, same weights. ([analysis](https://codex.danielvaughan.com/2026/04/09/harness-performance-terminal-bench/)) Workflow frameworks are a further layer of harness that nobody has measured rigorously.
3. **Workflow packs are hugely adopted but unmeasured.** Superpowers has about 304k GitHub stars and gstack about 137k (checked 28 Oct 2026). The controlled evidence is thin: one 500-task A/B of Superpowers on Codex, one 30-task study by a rival framework's authors, and small trials such as a 12-session Superpowers test. ([MindStudio](https://www.mindstudio.ai/blog/5-claude-code-skills-cut-token-costs-70-percent-benchmarked))
4. **The measuring instruments themselves leak.** OpenAI stopped reporting SWE-bench Verified in Feb 2026 after finding flawed tests in 59.4% of the hard problems it audited. Test-strengthening audits find about 1 in 5 leaderboard "solved" patches wrong ([SWE-ABS](https://arxiv.org/abs/2603.00520)). Cursor found 63% of Opus 4.8 Max's successful SWE-bench Pro resolutions retrieved the fix rather than deriving it; sealing git history and the network cut scores by 9–21 points ([Cursor](https://cursor.com/blog/reward-hacking-coding-benchmarks)). A small benchmark is only useful if every "resolved" verdict can be trusted.

People are choosing workflows that change cost, speed and correctness by large margins, with almost no controlled evidence. A reproducible harness fills that gap and stays useful as the frameworks change.

## Background: the shared pipeline and the frameworks under test

Most workflow frameworks implement variations of one pipeline. They differ in which stages they enforce, how hard (a skill the model may skip vs a hook it cannot), and where humans step in.

1. **Clarify intent** — ask questions and propose options before writing code.
2. **Write a spec** — a durable file with goals, non-goals and checkable acceptance criteria.
3. **Plan small tasks** — 2–5 minute tasks with file paths and a "done when" check.
4. **TDD** — red, green, refactor; the test exists before the code.
5. **Verify before claiming done** — full suite, real behaviour, evidence rather than assertion.
6. **Review** — self, fresh subagent, different model, then human.

| Framework | Core idea | Stages it stresses | Enforcement | Install (Claude Code) |
| --- | --- | --- | --- | --- |
| [Superpowers](https://github.com/obra/superpowers) | Auto-triggering methodology skills | All six; strict TDD that deletes code written before tests; subagent per task with two-stage review | Skills plus a SessionStart bootstrap hook | Official plugin marketplace |
| [gstack](https://github.com/garrytan/gstack) | A "virtual team" of 23 role skills | Clarify (`/office-hours` challenges premises), plan reviews, `/review`, real-browser `/qa`, `/ship` | Slash commands; safety skills enforced by hooks | `git clone` + `./setup` (needs Bun) |
| [pstack](https://cursor.com/marketplace/cursor/pstack) | `/poteto-mode` router over 23 playbooks and 24 principles | Verification with evidence; multi-model review panels (`/interrogate`, `/arena`) | Router skill; the Claude Code port adds a routing hook | Community port [pstack-claude](https://github.com/michael-denyer/pstack-claude) |
| [GSD](https://github.com/gsd-build/get-shit-done) | Context engineering against context rot | Planning in phases, fresh context per task, verification checklists | Slash commands, hooks, subagents | `npx get-shit-done-cc` |
| [Spec Kit](https://github.com/github/spec-kit) | Spec-driven development | Spec: constitution → specify → plan → tasks → implement → converge | Commands; quality rules come from a project constitution | `uv tool install specify-cli` |
| [Ralph loop](https://ghuntley.com/ralph/) | Same prompt in a loop, fresh context each time | Spec files, one task per loop, test "backpressure" | Bash loop, or Anthropic's plugin that re-feeds the prompt from a Stop hook | Plugin, or a one-line `while` loop |
|  |  |  |  |  |

Stars as of 28 Oct 2026: Superpowers about 304k, Spec Kit about 143k, gstack about 137k, pstack-claude under 1k. Popularity is not evidence of effect; that is the point of the project.

This project tests Superpowers plus one minimal framework (Ralph or GSD). The others are context; the harness should make adding any of them later a one-file job.

## Related work and the gap

The gap is real but narrower than it first looks: harnesses and single skills are well studied, and two early studies already test Superpowers. What is missing is an independent, multi-framework comparison with stage ablations, on feature-style work, in the frameworks' native harness. The wider evaluation-methods literature is in [LITERATURE_REVIEW.md](LITERATURE_REVIEW.md); this section covers only harness, skill and framework studies.

**Harness effects (same model, different scaffold).** Strong evidence that scaffolding moves results, usually more in cost and latency than in pass rate.

| Work | Setup | Key finding |
| --- | --- | --- |
| [Harness or Model?](https://arxiv.org/abs/2609.11987) (Sep 2026) | Opus 4.8 and GPT-5.5, native vs neutral harness, 80-task private suite, 792 graded runs | No resolved average pass-rate difference (CIs 11–17 pp wide); neutral harness 1.2–1.6× cost per solve; 22 of 81 runs cancelled at the time limit had a passing patch |
| [Scaffold Effect](https://arxiv.org/abs/2607.22585) (ICML 2026) | 2 models × 3 open harnesses, 50 Terminal-Bench Pro tasks | Pass-rate gaps 0–8 pp, but up to 40× difference in tokens per solved task; failure fingerprints belong to the harness; a later correction raised one harness's token count about 6.6× |
| [Harness-Bench](https://arxiv.org/abs/2605.27922) (May 2026) | 6 harnesses × 8 backends, 106 tasks, 5,194 trajectories | 23.8-point spread between harnesses; report capability per model–harness pair |
| [Same Model, Different Harness](https://arxiv.org/abs/2608.26218) (Aug 2026) | One context-trimming intervention | Large gains under a tight context window, almost none under a wide one |
| [SWE-Serve](https://arxiv.org/pdf/2609.26777) (Sep 2026) | Same model in several scaffolds | Opus 5 scored 69.8% inside Claude Code vs 75.5% in the ~100-line mini-SWE-agent; vendor harnesses don't automatically win |
| [Coding Agents Have Converged](https://arxiv.org/abs/2609.17394) (Sep 2026) | Audit of 254 SWE-bench Verified submissions | Scaffold alone moves one model by up to 29.8 pp; exact McNemar tests separate none of the 29 adjacent pairs in the top 30; only 164 of 500 tasks discriminate |
| [Infrastructure noise](https://www.anthropic.com/engineering/infrastructure-noise) (Anthropic, Feb 2026) | Terminal-Bench 2.0 under different resource limits | Resources alone moved scores 6 pp (p < 0.01); treat gaps under 3 pp sceptically |

**Skills (one skill or library, with vs without).** Mixed to sobering.

| Work | Finding |
| --- | --- |
| [SkillsBench](https://arxiv.org/abs/2602.12670) | Curated skills raise pass rate 33.9% → 50.5% across 18 configurations; software engineering gained least when tasks were already prescriptive |
| [SWE-Skills-Bench](https://arxiv.org/abs/2603.15401) | 39 of 49 skills gave zero pass-rate change; average +1.2 pp; token overhead up to +451% |
| [WebDev-Skills-Bench](https://arxiv.org/abs/2608.23067) | Skills lowered Pass@2 by 1.3–4.2%; uses a length-matched irrelevant control and leave-one-out ablations |
| [Regression Tax](https://arxiv.org/abs/2607.22520) | Across 5,832 paired runs, skills fixed 553 task conditions and broke 324; regressions cancelled 59% of gross gains |
| [TDAD](https://arxiv.org/abs/2603.17973) | TDD instructions alone raised regressions to 9.94%; telling the agent *which* tests to run cut them by 70% (small local models) |

**Workflow frameworks directly.** Closest prior work, with clear limits.

- [RigorBench](https://arxiv.org/abs/2606.22678) (Jun 2026) compares Superpowers, Agent-Skills and Agent-Rigor against a baseline on 30 tasks, scoring process discipline. Limits: written by the team behind Agent-Rigor, one mid-tier model (Gemini 3.5 Flash), a weighted process score as the headline metric, and no gstack, GSD, Spec Kit, pstack or Ralph.
- A [500-task A/B of Superpowers on Codex](https://norbert-laszlo.medium.com/can-a-plugin-improve-codex-benchmarking-the-superpowers-plugin-05d020066565) found 47.8% resolved with the plugin and called the correctness gain inconclusive. Limits: Codex rather than Claude Code, SWE-bench-style bug fixes where clarify and spec stages barely apply.
- [AgentV](https://github.com/EntityProcess/agentv/issues/1100) has a one-task showcase comparing Superpowers, agent-skills and compound against a baseline, with plans to expand.

**Methods worth borrowing for this project.**

1. Pairing by task, exact tests on gains vs regressions with Holm correction over a small pre-declared family, and saying up front what size of effect the design can detect (Harness or Model?, Coding Agents Have Converged, [Bowyer et al.](https://arxiv.org/html/2503.01747)).
2. Counting gains and regressions separately, not just the net change (Regression Tax).
3. A vague request plus a hidden PRD, answered by a simulated user (Asuka-Bench), kept tightly constrained and audited ([τ²-bench](https://arxiv.org/pdf/2506.07982) measured 40–47% simulator error rates in loose domains).
4. Recording raw token usage per turn and treating cost as relative, after Harness or Model?'s own telemetry error; plotting cost against resolve rate ([AI Agents That Matter](https://arxiv.org/abs/2407.01502), [HAL](https://arxiv.org/abs/2510.11977)).
5. Optionally, a length-matched irrelevant-skill control, to separate "more prompt text" from "better process" (WebDev-Skills-Bench).
6. Sealed, single-commit task repos with hidden tests injected only at grading, plus diff and transcript audits for hacking (Cursor, [METR](https://metr.org/blog/2025-06-05-recent-reward-hacking)).
7. Mutation-checking the hidden tests and hand-writing plausible-but-wrong patches that must fail (SWE-ABS, [EvalPlus](https://arxiv.org/abs/2305.01210)).
8. Reporting pass^k (all repeats succeed) beside mean pass@1 ([τ-bench](https://arxiv.org/abs/2406.12045)).

## Research questions

Three questions sized to about 120 runs. Pass-rate answers will be directional (the minimum detectable effect is roughly 30 pp; see Analysis), so write down beforehand what result would surprise you.

| # | Question | Expectation | Grounding |
| --- | --- | --- | --- |
| RQ1 | Do Superpowers, a minimal loop and your own framework solve more tasks than Claude Code's built-ins used well, and at what token and time cost? Put differently: is a framework worth installing at all? | Pass-rate difference inconclusive at this size; clear differences in tokens and time | Harness or Model?, Scaffold Effect, HAL, Superpowers-on-Codex A/B |
| RQ2 | Which tasks does each framework newly solve relative to the baseline, and which does it break? | Every framework breaks at least one task the baseline solves | Regression Tax |
| RQ3 | How faithfully does each enforcement mechanism verify? The same "tests must pass before finishing" rule as a skill, as `/goal` (a small model reads the transcript but runs nothing) and as a command `Stop` hook (a script runs the tests). Measured by false-accept rate (stopped as done while visible tests fail on a clean re-run), pass-but-wrong rate (visible tests pass, hidden fail) and test-tampering rate | Hook has the lowest false-accept rate; `/goal` accepts some unverified work; the persistence pressure of all three raises tampering relative to C3's plain skill | TDAD; [`/goal` docs](https://code.claude.com/docs/en/goal); Cursor and METR reward-hacking audits. No published study of transcript-only completion judges' false-accept rate exists |

## Experimental design

Same harness (Claude Code), same model and same tasks for every condition, hidden tests as the grader, and a budget counted in runs per usage window rather than dollars.

### Conditions

| ID | Condition | Why |
| --- | --- | --- |
| C0 | Baseline: Claude Code's built-ins used well. Plan mode to explore and ask questions, then implement, then `/goal` until the repo's visible tests pass (e.g. "the project's own test suite exits 0, no test files modified"). No plugins. | The realistic alternative to installing a framework |
| C1 | Superpowers | The most-adopted framework |
| C2 | Ralph loop (official plugin) or GSD | A minimal-process contrast |
| C3 | Your own lightweight framework, with its verification rule written as a skill | Your design, frozen before the test run |
| C3-goal | C3 with the verification rule enforced by `/goal` instead | RQ3: model-judged enforcement |
| C3-hook | C3 with the verification rule enforced by a command `Stop` hook instead | RQ3: deterministic enforcement |

The baseline is itself a small recipe of your choosing, so write it down exactly (the plan-mode step, the `/goal` condition template, nothing else) and freeze it before any scored run. C0 and C3-goal both use `/goal`, so the enforcement ablation compares variants of C3, not C3 against the baseline.

`/goal` is a prompt-based Stop hook: a small model (Haiku by default) reads the transcript and decides whether the condition is met, but it never runs commands or reads files, so it checks that verification was *shown*, not that it is *true*. Don't describe C0 or C3-goal as "verified until tests pass". Write the condition so the test command and its exit code must appear after the final edit, add "no test file modified" and a turn bound, and pin and log the evaluator model. Evaluation is skipped while subagents or background commands run.

### Controls

- One model for every run (your plan's default), with the Claude Code version pinned; record both per run.
- Per-run caps on turns and wall-clock time; grade whatever patch exists when a cap hits.
- A fresh copy of the task repo per run, with only that condition's plugin loaded. Check each session's init event to confirm which plugins actually loaded.
- 2 repeats per task and condition; 3 on a handful of tasks to see run-to-run variance.
- Sealed task repos: one commit, re-initialised, with no remotes, branches, tags or reflog, built at the pre-fix state. Restrict network egress to package registries where practical. Capable models mine git history and look fixes up online, and the gap grows with capability.
- Fixed resource limits (CPU, memory, timeouts), and run order interleaved across conditions within each usage window, so API drift or a mid-study model update isn't confounded with condition. Log infrastructure failures separately from agent failures.

### Task set

12–15 small tasks an agent finishes in 5–15 minutes, in small repos you write or adapt. Each has hidden FAIL\_TO\_PASS and PASS\_TO\_PASS tests, validated so the reference solution passes and the untouched repo fails.

| Stratum | Count | Notes |
| --- | --- | --- |
| Bug fix | 6–8 | Clear issue description; frameworks expected to help least |
| Small feature, underspecified | 6–7 | A vague request plus a hidden one-page PRD; where clarify and spec stages should matter |

Optionally add 2–3 easy [SWE-bench Pro](https://huggingface.co/datasets/ScaleAI/SWE-bench_Pro) tasks as an external check. Skip FeatureBench: models solve only about 11% of it, so a small sample would mostly measure zeros. Its lesson still applies: the same model scores ~11% on FeatureBench and 74% on SWE-bench, so feature tasks spread systems apart far more than bug fixes. Lean toward the feature stratum if you can only write a few more tasks.

Split before building C3: about 5 dev tasks for developing your framework, the rest as the test set every condition runs. Keep tasks out of any public repo. Power favours tasks over repeats: at the same run count, 20 tasks × 1 repeat matches 10 × 2 (literature review, own simulation), so every extra test task is worth more than an extra repeat.

**Validate every task before any scored run.** Audits find 8–20% of "solved" SWE-bench patches are actually wrong, mostly because the tests are weak, so check:

- The untouched repo fails every FAIL\_TO\_PASS test and passes every PASS\_TO\_PASS test; a do-nothing patch fails.
- The reference solution passes everything, 5–10 times in the pinned environment; drop any test that flips.
- Mutation check: run `mutmut` over the reference's changed lines and confirm the FAIL\_TO\_PASS tests kill most mutants.
- Write one or two plausible-but-wrong patches per task; they must fail.
- Fairness: every behaviour the tests assert is stated in the prompt or the hidden PRD (and reachable through the simulated user); tests don't pin private names or unspecified error strings.
- A task every condition fails on every run is probably broken, not hard.

### Simulated user (feature tasks only)

A small model plays the user. It sees the hidden PRD but never the tests, answers briefly, approves plans that match the PRD, never volunteers implementation hints, and stops after about 5 replies. It answers both `AskUserQuestion` calls (through the Agent SDK's `canUseTool`) and plain-text questions at the end of a turn. Give C0 the same user, so the only difference is whether the agent chooses to ask. Don't use `--permission-prompts none` or `dontAsk` mode for these runs: both remove or deny `AskUserQuestion`.

Simulated users are a known error source: τ²-bench found user-simulator errors in 40–47% of loosely constrained conversations, falling to 16% once the simulator was constrained. Keep the answer rules strict and identical across conditions, pin the simulator model, read a sample of exchanges per condition, and report simulator errors as a confound rather than noise.

### Metrics

One primary outcome; everything else is diagnostic.

| Metric | How |
| --- | --- |
| Resolved (primary) | All FAIL\_TO\_PASS pass and all PASS\_TO\_PASS still pass |
| pass^2 and flakiness | Both repeats resolved; count of tasks whose outcome differs between repeats |
| Gains and regressions vs C0 | Per task: newly solved vs newly broken |
| Partial credit (diagnostic) | Fraction of FAIL\_TO\_PASS passing; PASS\_TO\_PASS tests broken |
| Tokens and estimated cost | Per-turn usage and `total_cost_usd` from the run log; a relative efficiency measure, not real spend. Cost per resolved task, and a cost-vs-resolve plot |
| Wall-clock time and turns | From the run log; define "turn" explicitly |
| Questions asked | Number of simulated-user exchanges |
| False-accept and pass-but-wrong (RQ3) | For runs that stop as done: re-run visible tests in a clean copy (false accept if they fail); hidden tests fail while visible pass (pass-but-wrong) |
| Tampering and hacking | Diff touches existing tests, `conftest.py`, `pytest.ini` or CI config, or adds `skip`/`xfail`; transcript shows `git log --all`, web fetches or reads outside the workspace. Check with a script plus an LLM monitor plus a manual sample (METR found the two methods catch different cases); report as its own category, never silently dropped |
| Mechanism fired | Whether the framework actually engaged: plugins in the init event, skills invoked, Ralph iterations, hook blocks |
| n\_eff | Number of tasks not all-pass or all-fail across conditions; the real sample size |
| Static analysis (secondary) | `ruff`, `mypy`, `bandit` deltas on the diff; reported, not graded |
| Code quality (optional) | See below |

Don't score similarity to the reference solution: BLEU-family and CodeBLEU scores correlate weakly with correctness.

**If you add the code-quality judge**, let tests carry correctness and ask the judge only about what tests miss. Give it the task spec and the test output as references. Use a handful of binary criteria, each in its own call, with an "Unknown" option, rather than one 1–10 score. Prefer a reasoning-capable judge, ideally from a different model family (or list same-family judging as a threat to validity). Run each item 3–5 times and report the flip rate; LLM judges flipped 13.6% of pairwise verdicts on identical reruns in a 2026 study. Pick the ~15 human-rated runs to include clear passes and fails, blind the framework name, and report Cohen's κ and true-positive/true-negative rates rather than raw agreement. At n = 15, 80% agreement has a 95% interval of roughly 55–93%, so call it a calibration pilot, not a validation.

### Analysis

The task, not the run, is the unit of inference.

- **Declare a small confirmatory family before running.** One or two primary contrasts (e.g. C1 vs C0, and C3-hook vs C3); everything else is exploratory. With 4–6 conditions there are up to 15 pairwise tests, and testing them all at α = 0.05 gives roughly a 50% chance of at least one false positive. Holm-correct the declared family.
- **Primary test:** per-task gains and regressions, plus an exact sign-flip permutation test on the task-level differences (each in {−1, −0.5, 0, 0.5, 1} with 2 repeats). A paired t-interval is the alternative; pick one in the plan, not after seeing data. An exact sign test needs at least 6 gains and 0 regressions out of 10 tasks for p < 0.05.
- **Intervals:** Wilson (or Beta-posterior) intervals for each condition's resolve rate; 7/10 resolved is [0.40, 0.89]. The task-level percentile bootstrap is anti-conservative at n ≈ 10 (87–92% coverage in simulation; [Bowyer et al.](https://arxiv.org/html/2503.01747) agree), so plot it as descriptive only.
- **State the minimum detectable effect in the plan.** In the literature review's simulation (the researchers' own calculation, not a published figure), 10 tasks × 2 repeats detects a true +31 pp effect about 54% of the time and +17 pp about 21% of the time. Non-significance is not equivalence. Expect frameworks to separate on cost, time and turns rather than on pass rate.
- **Fix the edge-case rules in advance:** how infrastructure failures, cap hits and missing verdicts are scored, and that graded results are never re-run.
- **Optional learning exercise:** fit a hierarchical Bayesian model with [`bayes_evals`](https://arxiv.org/html/2503.01747) or [HiBayES](https://arxiv.org/abs/2505.05602) and compare it with the frequentist results.

Read transcripts before finalising any secondary metric: code failures into a small taxonomy adapted from [MAST](https://arxiv.org/abs/2503.13657v3) ("claimed done without evidence", "broke existing tests", "misread spec", "test tampering", "environment error"). A table of where each framework first fails across clarify → spec → plan → implement → verify says more at this sample size than a ranking.

## Harness architecture

Build a thin Python runner around the Claude Agent SDK (or `claude -p`) signed in with your subscription, give every run a fresh copy of the task repo, and keep frameworks behind a small adapter so adding one is a single file.

&#91;embedded content: harness data flow · one run\]

The task prompt goes into the container; the hidden PRD goes only to the simulated user; the hidden tests reach the grader only after the agent has finished.

### Components

| Component | Job |
| --- | --- |
| Task registry | Repo at a commit, prompt, hidden PRD, hidden tests, dev/test tag |
| Framework adapter | Load one framework for a run (plugin directory, skills, hooks) and say how to start it |
| Runner | Drive Claude Code on your subscription, enforce caps, save every event, stop cleanly at a usage limit |
| Simulated user | Answer questions from the hidden PRD on feature tasks |
| Grader | Copy the hidden tests in after the run, as a separate step from the agent's diff, overwriting any same-path files the agent created; run them; record pass/fail per test. Also re-run the visible tests in a clean copy for the false-accept rate |
| Auditor | Diff-check for edits to tests and test config; scan the transcript for git-history mining, web fetches and reads outside the workspace; flag runs for manual review |
| Ledger | One JSON line per run; doubles as the resume checkpoint across usage windows |
| Analysis | Exact paired tests, Wilson intervals, gain/regression table, descriptive bootstrap plot, a short report |

### The adapter contract

```python
class FrameworkAdapter(Protocol):
    name: str
    version: str            # git commit or package version, logged on every run

    def build_image(self, base_image: str) -> str:
        """Return an image tag with the framework installed (plugin, skills, hooks, deps such as Bun)."""

    def session_options(self, task: Task) -> dict:
        """Agent SDK options: plugins, setting sources, system prompt additions, allowed tools."""

    def first_prompt(self, task: Task) -> str:
        """How a user would start this framework, e.g. '/poteto-mode <task>' or the plain task."""

    def is_finished(self, events: list[Event]) -> bool:
        """Framework-specific stop condition (e.g. Ralph's completion promise)."""
```

### Implementation notes

- **Subscription auth:** sign in with your plan and make sure `ANTHROPIC_API_KEY` is unset, or runs bill the API instead. Don't use `--bare`, which ignores your subscription login.
- **Isolation:** a fresh workspace per run (a temp directory, or a Docker container if you want stronger isolation), loading only that condition's plugin with `--plugin-dir`. Keep your personal `~/.claude` free of other plugins and hooks while runs are going, or they leak into every condition.
- **Usage windows:** runs share your plan's 5-hour and weekly limits with your own use. Run one at a time, overnight; detect a limit error, stop, and resume from the ledger in the next window.
- **Two question channels:** `canUseTool` catches `AskUserQuestion`; plain-text questions arrive as an ended turn, so keep the session open in streaming-input mode and send the simulated user's reply as the next message.
- **Protect the hidden material:** add a `PreToolUse` hook to your own development setup that blocks reads of hidden tests and PRDs, so the agent helping you build never sees them.
- **Deterministic run IDs:** task × condition × repeat; retry only infrastructure failures, never graded results.

### Tooling to reuse

- [`claude plugin eval`](https://code.claude.com/docs/en/plugin-evals.md): start here. It runs on your plan, has a built-in with/without-plugin baseline, and produces a report. Use it for a first Superpowers suite before writing your own harness. Its limits: one plugin against none, no custom-code graders (so no hidden test suites), and no simulated user. Its isolation also strips CLAUDE.md, skills, hooks and settings, its network isn't sandboxed outside shell commands, and plugin hooks and MCP servers run outside its sandbox. Its 3-run default, no-plugin baseline arm and cost ceiling are a good template for your own harness.
- [Harbor](https://github.com/harbor-framework/harbor) and [Inspect](https://inspect.aisi.org.uk/): containerised eval frameworks worth reading for design ideas, but they typically run on API keys, so they aren't needed for this version. Harbor's task format (Dockerfile + instruction + oracle solution + test script) is a good model for the task registry.
- [Docent](https://docs.transluce.org/): searches and clusters transcripts; useful for coding failures and spotting hacking.

## Development setup

Build the repo with Superpowers plus your own hooks, but keep all of it out of the benchmark runs, where C0 must stay genuinely vanilla.

### Install Superpowers at project scope

```bash
cd loopbench
claude plugin install superpowers@claude-plugins-official --scope project
git add .claude/settings.json && git commit -m "Enable Superpowers for development"
```

- **Never user scope:** that enables it in every project on your machine, including your C0 runs.
- **Project scope** writes `enabledPlugins` to the committed `.claude/settings.json`, which also shows your workflow on the public repo. Anyone cloning it runs the install command once.
- **Check:** in a fresh session, ask to design a component; Superpowers' brainstorming skill should trigger before any code is written. `claude plugin list` shows the plugin and its scope.

### Keep benchmark runs clean

- Run every benchmark workspace in a temp directory **outside the repo**, so the repo's `.claude/` settings and CLAUDE.md never apply to the agent under test.
- Keep `~/.claude` free of user-scope plugins and hooks while runs are going.
- Store hidden tests and PRDs **outside the repo** (e.g. `~/loopbench-private/`) and deny reads of that folder.

### Repo settings and hooks

Add to `.claude/settings.json`, beside the `enabledPlugins` entry:

```json
{
  "permissions": {
    "deny": ["Read(~/loopbench-private/**)"]
  },
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/run-tests.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/run-tests.sh` (make it executable with `chmod +x`):

```bash
#!/bin/bash
cd "$CLAUDE_PROJECT_DIR" || exit 0
[ -f .claude/allow-red ] && exit 0          # escape hatch during TDD's red phase
if ! uv run pytest -q > /tmp/loopbench-tests.log 2>&1; then
  tail -n 30 /tmp/loopbench-tests.log >&2
  exit 2                                     # blocks finishing; Claude sees the failures
fi
exit 0
```

**Pitfall:** `Stop` fires whenever Claude ends a turn, including when it stops to ask you a question. In Superpowers' TDD red phase the tests are supposed to fail, so `touch .claude/allow-red` while writing failing tests and delete it afterwards. If it still gets in the way, leave the Stop hook out until the codebase is stable.

Also add a short `CLAUDE.md`: Python version, `uv` for dependencies, `pytest` for tests, and "never read or write anything under `~/loopbench-private`".

### Autoresearch, later

Only needed for the optional C3-auto phase, after the harness works and C3 is frozen. Two options:

- **SPML's Auto Research skills** (built on Superpowers): `claude plugin marketplace add qqhard/superpowers-ML`, then install the `spml` plugin at project scope using the exact name the marketplace prints. Map its protocol onto this project: variable files = `frameworks/c3-auto/`; fixed files = harness, grader, tasks; eval command = the harness script that runs the dev suite; metric = dev tasks resolved.
- **DIY:** a `frameworks/c3-auto/program.md` with the goal, the rules (edit only these files; accept a change only if it solves 2+ more dev tasks) and the eval command, plus a `PreToolUse` hook that blocks edits outside `frameworks/c3-auto/`. Smaller, and teaches more.

Sources: [Install and manage plugins](https://code.claude.com/docs/en/plugins/install.md) · [Hooks reference](https://code.claude.com/docs/en/hooks) · [Superpowers](https://github.com/obra/superpowers) · [SPML](https://github.com/qqhard/superpowers-ML)

## Plan and usage budget

About 10 weeks part-time. Measure your throughput before planning the main runs, because every later number depends on it.

| Phase | Weeks | Output | Gate to move on |
| --- | --- | --- | --- |
| 0. Plan | 1 | Committed plan with RQs, primary contrasts, the chosen test, the stated minimum detectable effect, edge-case scoring rules, task list | Plan committed before any scored run |
| 1. Warm-up | 1–2 | A 3–5 case `claude plugin eval` suite for Superpowers | You can read its report and explain the Δ |
| 2. Tasks | 2–4 | 12–15 tasks with hidden tests; dev/test split; a task-validation script | Every task passes the validation checks (reference passes 5–10×, untouched repo fails, mutants killed, wrong patches fail, fairness review) |
| 3. Harness | 3–5 | Runner, ledger, grader, simulated user; C0 and C1 adapters | One task end to end, graded, resumable after a limit stop |
| 4. Throughput pilot | 5 | 3 tasks × C0, checked against `/usage` | Runs per 5-hour window known |
| 5. Your framework | 5–7 | C3 developed on dev tasks only, then frozen | Frozen version tagged |
| 6. Main runs | 7–9 | All conditions on test tasks, 2 repeats, including the enforcement ablation | Every planned run graded or logged |
| 7. Analysis | 9–10 | Short report: exact tests and intervals, gains vs regressions, false-accept and tampering rates, what you learned; structured against the [ABC checklist](https://arxiv.org/html/2507.02825v5)'s reporting items | Numbers regenerate from the ledger |

### Usage budget

| Stage | Runs |
| --- | --- |
| Throughput pilot | \~6 |
| Main comparison: \~10 test tasks × 4 conditions (C0, C1, C2, C3) × 2 repeats | \~80 |
| Enforcement ablation (C3-goal, C3-hook): \~10 test tasks × 2 conditions × 2 repeats | \~40 |
| Toy auto-research (optional) | \~30–60 |

Runs per 5-hour window is unknown until the pilot. As a rough guess of 4–8 small runs per window, the \~120 main runs take 15–30 windows, roughly two to four weeks of overnight batches. If that is too slow, drop to one repeat on secondary conditions, cut C2, or run only C3-hook for the ablation. Keep 2 repeats on the primary contrast so pass^2 and flakiness stay measurable. If you have time to write more tasks, spend it there rather than on a third repeat.

## Optional extension: a toy auto-researched framework

A small-scale autoresearch loop applied to your own framework. It won't produce a trustworthy result; the point is to see first-hand how noise and overfitting behave when an agent optimises against a benchmark.

- Start from your frozen C3. The loop may edit only its few files: CLAUDE.md, skills, hooks.
- Each trial runs the \~5 dev tasks once; 5–10 trials in total.
- Accept a change only if it solves at least 2 more dev tasks than the current best. At this size, smaller gains are indistinguishable from noise. Even this is a noise filter, not a significance test: on 5 dev tasks, a 5–0 sweep gives only p = 0.0625.
- Every accept/reject decision spends some of the dev set's validity ([reusable holdout](https://arxiv.org/abs/1506.02629)); log how many gate queries the loop made. In 2026 self-improving-agent work, strict held-out gating mattered more than the optimiser ([RSEA](https://arxiv.org/abs/2606.28374)).
- Run the final candidate on the test tasks as C3-auto, and compare its dev-to-test drop with C3's.
- Protect the harness, grader and tasks with hooks; the loop sees dev scores and dev transcripts only.

Expected outcome: gains on dev that shrink on test. Documenting that drop is the lesson.

## Risks

| Risk | What goes wrong | Mitigation |
| --- | --- | --- |
| Usage limits | Runs stall mid-batch or block your own Claude use | Overnight batches; clean stop and resume; check `/usage` before each batch |
| Too little data | Real differences lost in noise | State the ~30 pp minimum detectable effect up front; exact tests; report intervals; prefer more tasks over more repeats; note what a larger study would need |
| Too many comparisons | A false positive among 10–15 pairwise tests | Pre-declare 1–2 primary contrasts; Holm correction; label the rest exploratory |
| Weak hidden tests | Wrong patches graded as resolved | Mutation check, wrong-patch check and fairness review for every task |
| Tasks too hard or too easy | Everything fails or everything passes, so nothing is compared | Pilot each task with C0 first; aim for roughly 30–70% solved |
| Bias toward your own framework | C3 tuned to the test tasks | Develop only on dev tasks; freeze before the test run |
| Leaky setup | Personal plugins, hooks or memory load into every condition | Clean config while running; confirm loaded plugins in each run's init event |
| Simulated user hints or errors | The user model gives the solution away, or answers wrongly | Strict answer rules; pinned simulator model; read a sample of transcripts; report simulator errors |
| Reward hacking | The agent edits or deletes tests, mines git history or looks the fix up; "keep going" pressure from Ralph, `/goal` and Stop hooks makes it more likely | Hidden tests stay out of the workspace until grading; sealed single-commit repos; restricted network; auditor plus manual review; report hacked runs as their own category |
| Unreliable judges | `/goal` or the quality judge accepts bad work or flips between runs | Treat `/goal` as transcript-only and measure its false-accept rate; repeat judge calls; report κ on human ratings |
| Infrastructure noise | Resource limits, API drift or a model update move scores | Pin resource limits; interleave conditions in time; log infrastructure failures separately |
| Framework churn | Versions change mid-study | Pin commits and the Claude Code version |

## Deliverables

1. A small open-source harness with adapters for every condition and instructions for adding another.
2. A private set of 12–15 validated tasks with hidden tests.
3. Your own lightweight framework, frozen and documented.
4. A literature review of evaluation methods ([LITERATURE_REVIEW.md](LITERATURE_REVIEW.md), done).
5. A short report or blog post: results with exact tests and intervals, gains vs regressions, the skill vs /goal vs hook comparison as false-accept and tampering rates, and an honest section on what the noise taught you.

Why it's worth doing: it exercises skills the industry says are scarce (evaluation design, verification, honest statistics) and gives you a concrete project to discuss in interviews or with a prospective Masters supervisor.

## Handoff notes for future sessions

Status on 28 Oct 2026: plan and literature review ([LITERATURE_REVIEW.md](LITERATURE_REVIEW.md)) only; no code, tasks or runs yet. Research notes behind the review are in `research_notes/`. Scope is this lean, subscription-only version; the larger API-funded design was dropped and should not be revived without being asked.

### Decisions made so far

- A learning project, not for publication; directional results are fine.
- Claude Code subscription only: no API key, no `--bare`; budget in runs per usage window.
- Four conditions (C0 built-ins used well as the baseline, C1 Superpowers, C2 Ralph or GSD, C3 your own); plain no-features mode was dropped plus an enforcement ablation (the same verification rule as a skill, as /goal, and as a command Stop hook).
- 12–15 small private tasks with hidden tests, split into dev and test; C3 developed on dev only and frozen before the test run.
- Superpowers plus your own hooks is fine for building the harness; mention that in the write-up.
- After the literature review: exact paired tests with Holm correction over 1–2 pre-declared contrasts as the primary analysis, bootstrap as descriptive only; every task mutation-checked and sealed; an auditor for hacking; `/goal` treated as a transcript-only judge; RQ3 measured by false-accept, pass-but-wrong and tampering rates.

### Open questions

- Ralph or GSD as C2?
- Which one or two contrasts are confirmatory, and sign-flip permutation or paired t as the pre-registered test?
- Can you write ~20 tasks instead of 12–15? Power favours tasks over repeats.
- Plain `claude -p` scripting or the Python Agent SDK for the runner? The SDK is needed for `canUseTool`.
- How many runs fit in a 5-hour window? Measure in phase 4.
- Whether to do the toy auto-research extension.

### Before you start any session

1. Re-check that `claude -p` and the Agent SDK still draw from subscription limits (Claude Help Center, "Use the Claude Agent SDK with your Claude plan").
2. Re-check framework versions and install methods.
3. Glance at arXiv for new framework or skill benchmarks and add anything relevant to Related work (or, for evaluation methods generally, to LITERATURE_REVIEW.md).

### First tasks

- [ ] Commit the plan, RQs and primary comparisons to a new repo
- [ ] Build a 3–5 case `claude plugin eval` suite for Superpowers and read the report
- [ ] Write the task-validation script (base fails, reference passes 5–10×, mutation check, wrong patches fail)
- [ ] Write the first 3 tasks with hidden tests and validate them
- [ ] Build the runner and ledger for C0 on one task, including stop-and-resume at a usage limit
- [ ] Run the throughput pilot and record runs per window

## References

**Frameworks**

- [Superpowers](https://github.com/obra/superpowers) · [gstack](https://github.com/garrytan/gstack) · [Spec Kit](https://github.com/github/spec-kit) · [GSD](https://github.com/gsd-build/get-shit-done) ([v2 notes](https://nevercodealone.de/de/vibe-coding/vibe-coding-modelle/gsd-get-shit-done-framework-claude-code-2026))
- [pstack (Cursor)](https://cursor.com/marketplace/cursor/pstack) · [pstack-claude port](https://github.com/michael-denyer/pstack-claude/wiki) · [poteto-mode explainer](https://flaviocopes.com/poteto-mode/)
- [Ralph Wiggum as a "software engineer"](https://ghuntley.com/ralph/) · [Ralph plugin overview](https://www.atcyrus.com/stories/ralph-wiggum-technique-claude-code-autonomous-loops)

**Harness and skill research**

- [Harness or Model? (arXiv 2609.11987)](https://arxiv.org/abs/2609.11987)
- [The Scaffold Effect in Coding Agents (arXiv 2607.22585)](https://arxiv.org/abs/2607.22585)
- [Harness-Bench (arXiv 2605.27922)](https://arxiv.org/abs/2605.27922)
- [Same Model, Different Harness (arXiv 2608.26218)](https://arxiv.org/abs/2608.26218)
- [SkillsBench (arXiv 2602.12670)](https://arxiv.org/abs/2602.12670) · [SWE-Skills-Bench (arXiv 2603.15401)](https://arxiv.org/abs/2603.15401) · [WebDev-Skills-Bench (arXiv 2608.23067)](https://arxiv.org/abs/2608.23067)
- [The Regression Tax (arXiv 2607.22520)](https://arxiv.org/abs/2607.22520) · [TDAD (arXiv 2603.17973)](https://arxiv.org/abs/2603.17973)
- [RigorBench (arXiv 2606.22678)](https://arxiv.org/abs/2606.22678) · [Superpowers on Codex A/B](https://norbert-laszlo.medium.com/can-a-plugin-improve-codex-benchmarking-the-superpowers-plugin-05d020066565) · [AgentV workflow benchmark issue](https://github.com/EntityProcess/agentv/issues/1100)
- [Terminal-Bench harness analysis](https://codex.danielvaughan.com/2026/04/09/harness-performance-terminal-bench/) · [MindStudio 12-session Superpowers test](https://www.mindstudio.ai/blog/5-claude-code-skills-cut-token-costs-70-percent-benchmarked)

**Evaluation methodology** (full list in [LITERATURE_REVIEW.md](LITERATURE_REVIEW.md))

- [Anthropic, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) · [Anthropic, infrastructure noise](https://www.anthropic.com/engineering/infrastructure-noise)
- [Miller, Adding Error Bars to Evals (arXiv 2411.00640)](https://arxiv.org/abs/2411.00640) · [Bowyer et al., don't use the CLT at small n (arXiv 2503.01747)](https://arxiv.org/abs/2503.01747) · [Coding Agents Have Converged (arXiv 2609.17394)](https://arxiv.org/abs/2609.17394) · [Randomness in agentic evals (arXiv 2602.07150)](https://arxiv.org/abs/2602.07150)
- [Cursor, reward hacking in coding benchmarks](https://cursor.com/blog/reward-hacking-coding-benchmarks) · [METR, recent reward hacking](https://metr.org/blog/2025-06-05-recent-reward-hacking) · [OpenAI on retiring SWE-bench Verified](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)
- [Agentic Benchmark Checklist (arXiv 2507.02825)](https://arxiv.org/abs/2507.02825) · [SWE-ABS (arXiv 2603.00520)](https://arxiv.org/abs/2603.00520) · [UTBoost](https://arxiv.org/html/2506.09289v1) · [τ²-bench (arXiv 2506.07982)](https://arxiv.org/abs/2506.07982) · [HAL (arXiv 2510.11977)](https://arxiv.org/abs/2510.11977)
- [CodeJudgeBench (arXiv 2507.10535)](https://arxiv.org/abs/2507.10535) · [Reliability without Validity (arXiv 2606.19544)](https://arxiv.org/abs/2606.19544) · [The Coin Flip Judge? (arXiv 2606.13685)](https://arxiv.org/abs/2606.13685) · [Husain & Shankar, AI Evals FAQ](https://hamel.dev/blog/posts/evals-faq/)
- [SWE-Serve (arXiv 2609.26777)](https://arxiv.org/abs/2609.26777) · [MAST failure taxonomy (arXiv 2503.13657)](https://arxiv.org/abs/2503.13657)

**Tasks and evaluation tooling**

- [SWE-bench Pro (arXiv 2509.16941)](https://arxiv.org/abs/2509.16941) · [dataset](https://huggingface.co/datasets/ScaleAI/SWE-bench_Pro) · [FeatureBench](https://github.com/LiberCoders/FeatureBench) · [Asuka-Bench (arXiv 2606.05920)](https://arxiv.org/abs/2606.05920)
- [Harbor](https://github.com/harbor-framework/harbor) · [Inspect](https://inspect.aisi.org.uk/) · [inspect-harbor](https://pypi.org/project/inspect-harbor/)

**Claude Code documentation**

- [Hooks reference](https://code.claude.com/docs/en/hooks) · [`/goal`](https://code.claude.com/docs/en/goal) · [Run Claude Code programmatically](https://code.claude.com/docs/en/headless) · [Handle approvals and user input](https://code.claude.com/docs/en/agent-sdk/user-input) · [Test plugins with evals](https://code.claude.com/docs/en/plugin-evals.md)
- [Use the Claude Agent SDK with your Claude plan](https://support.claude.com/en/articles/15036540)

**Industry context**

- [GitLab AI Accountability Report and Sonar survey coverage](https://devops.com/?p=188681)
