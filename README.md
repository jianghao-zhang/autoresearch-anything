# autoresearch-anything

> The scientific method at silicon speed.

Karpathy's autoresearch loop — propose, verify, keep if better — generalized beyond ML to any domain with a measurable optimization target. You write the strategy. The agent runs experiments indefinitely.

The key insight: the bottleneck is no longer human insight per experiment — it's how fast the loop runs. This removes the human from the short experimental cycle entirely. You become the lab director writing `program.md`; the agent becomes the tireless grad student running experiments 24/7.

---

## Three Primitives

| Primitive | What it is | Role |
|-----------|-----------|------|
| **Modify** | File(s) the agent can change | The search space |
| **Verify** | Shell command that outputs a number | The fitness function — empirical, not subjective |
| **Strategy** | Goal, constraints, memory | `program.md` — the only thing the human writes |

**The rule:** only provably better changes survive. Everything else is discarded and logged.

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

## Works Anywhere There's a Metric

| Domain | Modify | Measure |
|--------|--------|---------|
| Zsh startup | `.zshrc` | `for i in {1..5}; do time zsh -i -c exit; done 2>&1 \| awk '/real/{s+=$2;n++} END{print s/n}'` |
| ML hyperparams | `train.py` | `python train.py 2>&1 \| grep val_loss \| tail -1 \| awk '{print $NF}'` |
| Bundle size | `webpack.config.js` | `npm run build --silent && du -sb dist/ \| awk '{print $1}'` |
| System prompt | `prompts/system.txt` | `python eval.py --judge gpt-4o \| grep score \| awk '{print $2}'` |
| SQL query | `queries/report.sql` | `psql $DB -f queries/report.sql \| tail -1` |
| API latency | `src/handler.rs` | `wrk -t4 -c100 -d10s http://localhost:8080/api \| grep Latency \| awk '{print $2}'` |

---

## References

- **autoresearch-anything** (this repo) — Claude Code skill: [`my-autoresearch`](https://github.com/jianghao-zhang/autoresearch-anything)
- **Karpathy's original autoresearch** (upstream) — canonical ML implementation, val_bpb metric, 5-minute budget: [karpathy/autoresearch](https://github.com/karpathy/autoresearch)
