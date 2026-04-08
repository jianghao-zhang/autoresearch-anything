# autoresearch-anything

> The scientific method at silicon speed.
>
> *For domains with fast, objective, scriptable feedback.*

Karpathy's autoresearch loop — propose, verify, keep if better — generalized beyond ML. You write the strategy. The agent runs experiments.

The key insight: the bottleneck is no longer human insight per experiment — it's how fast the loop runs. You become the lab director writing `program.md`; the agent becomes the tireless grad student running experiments 24/7.

The catch is that improvement is only as good as the metric. This works when the loop is measurable, repeatable, and hard to game — not for every workflow.

---

## Three Primitives

| Primitive | What it is | Role |
|-----------|-----------|------|
| **Modify** | File(s) the agent can change | The search space |
| **Measure** | Shell command that outputs a number | The fitness function |
| **Strategy** | Goal, constraints, memory | `program.md` — the only thing the human writes |

**The rule:** only provably better changes survive. Everything else is discarded and logged.

---

## When to Use

- Fast, objective, numeric metrics
- Shell/system config (startup time, latency, throughput)
- ML hyperparameter search (single-machine)
- Code performance (build size, benchmark time, error count)
- Prompt/eval tuning with a stable judge or benchmark
- SQL/query optimization
- Any tweak-and-test loop that can be automated safely

## When NOT to Use

- Subjective goals ("make it prettier")
- Feedback loops that are too slow for iteration
- Irreversible side effects (sending emails, placing orders, real spend, prod writes)
- Production systems without isolation
- Multi-variable changes per experiment
- Non-scriptable or hard-to-revert search spaces

---

## Metric Integrity Check

Before running the loop, pressure-test the metric:

1. **Decoupling test** — can you improve the number without improving the true goal?
2. **Determinism test** — does the number depend on randomness, time, network, or external state?
3. **Degenerate-input test** — if you delete or stub the target, does the metric still look good?

If the answer is "yes" in the wrong way, redesign the metric first. Results are only verified **against the chosen metric**.

---

## Hypothesis Heuristics

The search policy should not be a random walk. Use bounded exploration:

- `tune` — adjust an existing parameter or toggle one flag
- `ablate` — remove, simplify, or disable something
- `swap` — replace a component with a known alternative
- `novel` — try a qualitatively different approach after repeated failures

Default to `tune`, prefer ablation before adding complexity, and rotate buckets if you keep retrying the same kind of idea.

---

## program.md (minimal)

```markdown
# Research: [Goal]

## Config
- **Modify**: [file or glob]
- **Measure**: [shell command → single number]
- **Direction**: lower | higher
- **Iterations**: N  ← omit for open-ended

## Current State
- Baseline: [X]  ← measured, not assumed
- Target: [Y]

## Dead Ends (DO NOT RETRY)
| Exp ID | What was tried | Why it failed |
|--------|---------------|---------------|

## Cross-Session Learnings
- Session [date]: [key insight]

## Past Results
| ID | Hypothesis | Result | Keep? |
|----|-----------|--------|-------|
| 0  | baseline  | —      | ✓     |
```

Human-provided `Modify` and `Measure` are always preferred. If missing, the agent scans the codebase to infer `Modify` and finds or builds the smallest command that outputs one number for `Measure`.

---

## Examples by Domain

| Domain | Modify | Measure |
|--------|--------|---------|
| Zsh startup | `.zshrc` | `for i in {1..5}; do time zsh -i -c exit; done 2>&1 \| awk '/real/{s+=$2;n++} END{print s/n}'` |
| ML hyperparams | `train.py` | `python train.py 2>&1 \| grep val_loss \| tail -1 \| awk '{print $NF}'` |
| Bundle size | `webpack.config.js` | `npm run build --silent && du -sb dist/ \| awk '{print $1}'` |
| SQL query | `queries/report.sql` | `psql $DB -c "EXPLAIN (ANALYZE,FORMAT JSON) $(cat queries/report.sql)" \| python parse_ms.py` |
| API latency | `src/handler.rs` | `wrk -t4 -c100 -d10s http://localhost:8080/api \| grep Latency \| awk '{print $2}'` |
| LLM agent system prompt | `prompts/agent_system.txt` | `python run_eval.py --suite swe_bench_lite \| grep pass_rate \| awk '{print $NF}'` |
| Agentic RL reward shaping | `envs/reward.py` | `python rollout.py --n 64 --seed 0 \| grep mean_return \| awk '{print $NF}'` |
| Agent memory retrieval | `memory/retriever.py` | `python eval_recall.py --k 10 \| grep recall@10 \| awk '{print $NF}'` |
| Trading strategy params | `strategies/momentum.yaml` | `python backtest.py --years 5 \| grep sharpe \| awk '{print $NF}'` |
| Blog post hook | `posts/draft.md` | `python llm_judge.py --rubric engagement --model gpt-5 \| grep score \| awk '{print $NF}'` |

---

## References

- **autoresearch-anything** (this repo) — Claude Code skill: [`my-autoresearch`](https://github.com/jianghao-zhang/autoresearch-anything)
- **Karpathy's original autoresearch** (upstream) — canonical ML implementation, val_bpb metric, 5-minute budget: [karpathy/autoresearch](https://github.com/karpathy/autoresearch)
