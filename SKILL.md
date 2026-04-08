---
name: autoresearch-anything
description: ALWAYS use this when the user wants to optimize anything through systematic experimentation, A/B testing configurations, tuning hyperparameters, or running iterative benchmarks. This includes shell startup time, build configs, ML model parameters, code performance, bundle size, system prompts, SQL queries, or any "tweak and verify" workflow. Use even if they don't explicitly say "experiment" or "benchmark"—if they're trying to make something faster, smaller, or better through trial and error, this skill applies.
---

# Autoresearch-Anything

**The scientific method at silicon speed.** Karpathy's autoresearch loop — propose, verify, keep if better — generalized beyond ML: you write the strategy, the agent runs experiments indefinitely.

The key insight: AI research has matured from discovery into engineering. The bottleneck is no longer human insight per experiment — it's how fast the loop runs. This skill removes the human from the short experimental cycle entirely. You become the lab director writing `program.md`; the agent becomes the tireless grad student running experiments 24/7.

## First Principles

The loop reduces to three primitives and one rule:

| Primitive | What it is | Role |
|-----------|-----------|------|
| **Modify** | File(s) the agent can change | The search space |
| **Verify** | Command that outputs a number | The fitness function — empirical, not subjective |
| **Strategy** | Goal, constraints, memory | `program.md` — the only thing the human writes |

**The rule:** only provably better changes survive. Everything else is discarded and logged.

This is grad student descent as a service — the agent proposes hypotheses, runs experiments, and keeps only verified improvements, compounding gains overnight. `train.py` and `benchmark.py` are ML-specific conveniences; the primitives themselves work in any domain.

## When to Use

- Any goal with a fast, objective, numeric metric (feedback loop < 30 min)
- Shell/system config (startup time, latency, throughput)
- ML hyperparameter search (single-machine)
- Code performance (build size, benchmark time, error count)
- System prompt quality (LLM-as-judge score)
- SQL/query optimization (execution time)
- Any "tweak-and-test" loop that can be automated

## When NOT to Use

- Subjective/non-numeric goals ("make it prettier")
- Feedback loop > 30 minutes per experiment (statistical advantage collapses)
- Production systems without isolation
- Multi-variable changes per experiment
- Irreversible side effects (sending emails, placing orders, real spend, prod writes)
- Feedback latency > 30 min without a usable proxy metric
- Non-discrete or non-scriptable search spaces (physical experiments, human studies)

---

## Core Structure

```
program.md         ← declares Modify + Measure + Strategy (the only required file)
experiments.jsonl  ← auto-generated history
[target files]     ← whatever the agent modifies (any language, any format)
[measure script]   ← optional: only needed when measurement logic is complex
```

---

## program.md Template

```markdown
# Research: [Goal]

## Config
- **Modify**: [file or glob — e.g. config.yaml, src/model.py, prompts/system.txt] ← human input preferred; agent infers if blank
- **Measure**: [shell command that prints a single number] ← human input preferred; agent derives if blank
- **Direction**: lower | higher
- **Iterations**: N  ← optional; omit for open-ended (stop when interrupted or goal reached)
- **Budget**: 300  # seconds per experiment, non-negotiable

## Metric Integrity Check
<!-- Agent MUST answer all three before starting the loop. Any "yes/unsafe" → redesign Verify. -->

1. **Decoupling test**: Can you name a concrete change that would improve Verify's number while visibly NOT improving the true goal? If yes → metric is gameable.
2. **Determinism test**: Does Verify depend on randomness, wall-clock time, network, or external state? If yes → fix seed, pin versions, isolate env, or add median-of-N.
3. **Degenerate-input test**: If the target file is emptied / commented out / replaced with a stub, does Verify still produce a legal number? If yes → is that number *worse* than baseline? If not → Verify rewards destruction. Redesign.

## Current State
- Baseline: [X] (measured, not assumed — run Measure command and record it)
- Target: [Y]

## Constraints
- [e.g., "Must not break existing tests"]
- [e.g., "Output format must remain identical"]

## Dead Ends (DO NOT RETRY)
<!-- Agent MUST append here after every discarded experiment -->
| Exp ID | What was tried | Why it failed |
|--------|---------------|---------------|
| —      | —             | —             |

## Cross-Session Learnings
<!-- Agent MUST write key insights here before ending a session -->
<!-- Next session MUST read this section before proposing any hypothesis -->
- Session [date]: [key insight]

## Past Results
| ID | Hypothesis | Result | Keep? |
|----|-----------|--------|-------|
| 0  | baseline  | —      | ✓     |
```

---

## Hypothesis Heuristics

Turn "what to try next" from art into a finite state machine. Each proposed hypothesis MUST be tagged with one of four buckets:

| Bucket | Meaning | When to use |
|--------|---------|-------------|
| `tune` | Adjust existing param ±20% or toggle single flag | Default — exploit the neighborhood of current best |
| `ablate` | Remove / simplify / disable something | Before adding — smallest change that might reveal dead weight |
| `swap` | Replace one component with a known alternative | When tune + ablate have been exercised |
| `novel` | Qualitatively different approach (new algorithm, new abstraction) | After 5 consecutive discards OR 3 same-bucket retries |

**Rules:**
1. **Exploit first**: default to `tune` against the current best
2. **Ablate before add**: prefer removing over adding when both are plausible
3. **Copy before invent**: before `novel`, search git log, community forks, arxiv for existing solutions
4. **Rotation rule**: 3 consecutive hypotheses in the same bucket → force-switch to a different bucket
5. **Log the bucket**: record `bucket` field in experiments.jsonl alongside `hypothesis`

This replaces "agent randomly tries things" with a bounded exploration policy.

---

## Agent Loop

### Phase 0 — Bootstrap

**Priority rule: human-provided config is always preferred. Bootstrap only fills in what's missing.**

```
IF Modify is provided by human → use it as-is
ELSE → infer: scan codebase/configs/prompts for files that plausibly affect the goal;
       pick smallest scope; write to program.md

IF Measure is provided by human → use it as-is, skip to validation
ELSE → derive:
       FIND:    look for existing benchmarks, test suites, profilers, eval scripts
                (bench.py, pytest --benchmark, cargo bench, wrk, hyperfine, ...)
       BUILD:   if none found, write the minimum command that outputs one number
                (shell one-liner preferred; Python script only if logic is complex)
       COMPOSE: chain steps if measurement requires multiple operations

ALWAYS → validate Measure command:
       - Run it on unmodified target; confirm it prints exactly one number and exits cleanly
       - Run it 3x; if variance > 10% → add median/repeat logic before proceeding

ALWAYS → Metric Integrity Check:
       - Ask: "How could an agent game this metric without actually improving the goal?"
       - If there's a clear exploit → redesign the metric (applies to human-provided metrics too)
       - Write the answer to program.md

ALWAYS → record baseline:
       - Run Measure → record as Experiment 0 in experiments.jsonl
       - git commit -m "Experiment 0: baseline ([value])"
```

### Session Start (mandatory when resuming)
1. Read `program.md` — **Dead Ends** and **Cross-Session Learnings** first
2. Re-run `Measure` on current state to confirm environment still matches baseline
3. If Modify/Measure are missing → run Bootstrap above

### Per-Experiment Loop

```
# Stopping conditions (check before each iteration):
#   - Iterations: N was set → stop after N experiments, print summary
#   - Target was set and reached → stop, announce success
#   - User interrupts (Ctrl-C) → stop, write session learnings, print summary
#   - None of the above → continue indefinitely

LOOP:
  1. Read Dead Ends — never retry anything listed there
  2. Propose one hypothesis — modify only files listed in Modify
  3. Run: timeout $BUDGET sh -c "$MEASURE" → parse the number
  4. Compare to best so far:
       BETTER  → git commit "Exp N: [hypothesis] (old → new)"
                 record learning in experiments.jsonl
                 update Cross-Session Learnings if insight is durable
                 check stopping conditions
       WORSE   → git checkout [target files]  (revert target only, not program.md)
                 append to Dead Ends with reason
  5. Stall check: 5 consecutive discards?
       YES → inject novelty: search arxiv / community forks / inspect git log patterns
             propose a qualitatively different hypothesis class
       NO  → go to step 1
```

### Session End (mandatory)
Write key insights to **Cross-Session Learnings** before closing.
`program.md` is the only persistent memory — the next session agent starts cold.

---

## experiments.jsonl Schema

```json
{
  "id": 1,
  "timestamp": "2026-04-09T10:00:00",
  "hypothesis": "reduce batch size from 64 to 32",
  "bucket": "tune",
  "value": 142.3,
  "baseline": 180.0,
  "kept": true,
  "timed_out": false,
  "duration_seconds": 47.2,
  "learning": "smaller batch = better GPU utilization at this model size"
}
```

---

## Git Workflow (Critical)

Every experiment must be committed — git is the experiment log.

```bash
# Baseline commit
git init
git add .
git commit -m "Experiment 0: baseline"

# Each kept experiment
git add [target files] experiments.jsonl program.md
git commit -m "Experiment 3: lazy load conda (530ms → 180ms)"

# Each discarded experiment
git checkout [target files]   # revert target only
# do NOT revert program.md — Dead Ends entry stays
```

---

## Optional: benchmark.py

Only needed when the Measure command requires complex Python logic (e.g., multi-run median, statistical testing, ML training eval).

```python
#!/usr/bin/env python3
"""Optional measurement helper. Use only when a shell one-liner is insufficient."""

import json, time, signal, statistics
from datetime import datetime

EXPERIMENT_LOG = "experiments.jsonl"
TIME_BUDGET_SECONDS = 300  # must match program.md Budget

def evaluate(config: dict) -> dict:
    """
    Implement your measurement here. Must return {"value": float, ...}.
    Rules:
    - Must complete within TIME_BUDGET_SECONDS (enforced below)
    - Same conditions every run (fixed seed, fixed data, fixed hardware)
    """
    raise NotImplementedError

def run(exp_config: dict) -> dict:
    def _timeout(signum, frame):
        raise TimeoutError

    signal.signal(signal.SIGALRM, _timeout)
    signal.alarm(TIME_BUDGET_SECONDS)
    start = time.time()
    timed_out = False
    try:
        result = evaluate(exp_config.get("config", {}))
    except TimeoutError:
        result = {"value": float("inf"), "success": False}
        timed_out = True
    finally:
        signal.alarm(0)

    record = {
        "id": exp_config["id"],
        "timestamp": datetime.now().isoformat(),
        "hypothesis": exp_config["hypothesis"],
        "value": result["value"],
        "timed_out": timed_out,
        "duration_seconds": round(time.time() - start, 1),
        "learning": exp_config.get("learning", ""),
        "config": exp_config.get("config", {}),
    }
    with open(EXPERIMENT_LOG, "a") as f:
        f.write(json.dumps(record) + "\n")
    status = "TIMEOUT" if timed_out else result["value"]
    print(f"Exp #{exp_config['id']}: {status} ({record['duration_seconds']}s)")
    return record
```

---

## Examples by Domain

| Domain | Modify | Measure |
|--------|--------|---------|
| Zsh startup | `.zshrc` | `for i in {1..5}; do time zsh -i -c exit; done 2>&1 \| awk '/real/{sum+=$2;n++} END{print sum/n}'` |
| ML hyperparams | `train.py` | `python train.py 2>&1 \| grep val_loss \| tail -1 \| awk '{print $NF}'` |
| Bundle size | `webpack.config.js` | `npm run build --silent && du -sb dist/ \| awk '{print $1}'` |
| System prompt | `prompts/system.txt` | `python eval.py --judge gpt-4o \| grep score \| awk '{print $2}'` |
| SQL query | `queries/report.sql` | `psql $DB -c "EXPLAIN (ANALYZE,FORMAT JSON) $(cat queries/report.sql)" \| python parse_ms.py` |
| Rust benchmark | `src/solver.rs` | `cargo bench 2>&1 \| grep test_solve \| awk '{print $5}'` |
| API latency | `src/handler.rs` | `wrk -t4 -c100 -d10s http://localhost:8080/api \| grep Latency \| awk '{print $2}'` |
| LLM agent system prompt | `prompts/agent_system.txt` | `python run_eval.py --suite swe_bench_lite \| grep pass_rate \| awk '{print $NF}'` |
| Agentic RL reward shaping | `envs/reward.py` | `python rollout.py --n 64 --seed 0 \| grep mean_return \| awk '{print $NF}'` |
| Agent memory retrieval | `memory/retriever.py` | `python eval_recall.py --k 10 \| grep recall@10 \| awk '{print $NF}'` |
| Trading strategy params | `strategies/momentum.yaml` | `python backtest.py --years 5 \| grep sharpe \| awk '{print $NF}'` |
| Cold outbound email | `emails/outreach.md` | `python crm_stats.py --campaign q2 \| grep reply_rate \| awk '{print $NF}'` |
| Blog post hook | `posts/draft.md` | `python llm_judge.py --rubric engagement --model gpt-5 \| grep score \| awk '{print $NF}'` |

---

## Best Practices

**DO:**
- Measure actual baseline before touching anything
- One hypothesis per experiment — one variable at a time
- Write Dead Ends entry immediately after every discard
- Use `timeout $BUDGET` in Measure command if not using benchmark.py

**DON'T:**
- Change multiple variables at once
- Skip version control
- Accept a noisy metric — run 3-5x and take median if needed
- Retry anything already in Dead Ends

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| "It should be around X" | Run Measure and record the actual number |
| No Dead Ends entries | Agent will repeat failures across sessions |
| Measure outputs multiple lines | Pipe through `tail -1` or `grep` to get one number |
| Metric can be gamed | Complete Metric Integrity Check before loop starts |
| No baseline commit | `git commit -m "Experiment 0: baseline"` before any modification |

---

## Troubleshooting

**"Measure command output is unparseable"**
→ Test it manually: `sh -c "$MEASURE"` — must print exactly one number.

**"Results are inconsistent"**
→ Add `statistics.median` over 3-5 runs; fix seeds; isolate from background processes.

**"Agent keeps retrying failed ideas"**
→ Dead Ends section wasn't updated. Non-negotiable after every discard.

**"New session repeats work from last session"**
→ Cross-Session Learnings wasn't written. The file IS the memory — treat it as a deliverable, not a footnote.

**"Loop is stuck, no improvement for 5+ experiments"**
→ Trigger novelty injection: search arxiv for domain techniques, inspect community forks, try the opposite of recent attempts. Stall = signal to explore, not to grind harder.

---

## Reference

- **autoresearch-anything** (this skill): https://github.com/jianghao-zhang/autoresearch-anything
  First-principles generalization of Karpathy's loop to any domain with a measurable fitness function.

- **Karpathy's original autoresearch** (upstream): https://github.com/karpathy/autoresearch
  Canonical ML implementation — single-GPU nanochat training, 5-minute budget, val_bpb as the verification metric. The source of the core philosophy.
