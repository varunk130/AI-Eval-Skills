---
name: cost-quality-frontier
description: 'Adds cost (input + output tokens × model price) and latency (p50, p95) to eval results, plots model options on a Pareto frontier, and produces a quality-per-dollar composite score so production model selection is grounded in trade-offs, not just quality. Use when: model comparison, cost-aware evals, latency budget, quality-per-dollar, Pareto frontier, model selection, eval economics, picking a model for production.'
---

# Cost-Quality Frontier

Most eval suites today answer the question *"which model is most accurate?"* The production question is harder: *"which model gives me the best quality I can afford, at the latency budget my product allows?"* This skill takes existing eval runs and augments each result with cost and latency, then plots the candidates on a Pareto frontier so the trade-off is visible at a glance.

## Core Principle

**Quality, cost, and latency are a single decision, not three.** A model that is 2 points more accurate but costs 6× and adds 800ms of p95 latency is rarely the right pick. A model that is 1 point worse but cheaper *and* faster usually is — but only if you can see the frontier. This skill makes the frontier explicit.

---

## What You'll Get

| Artifact | Description |
|----------|-------------|
| **Augmented Eval Schema** | Every result row gains `cost_usd`, `input_tokens`, `output_tokens`, `latency_ms`, `model`, `pricing_source` columns |
| **Per-Model Aggregates** | Quality (mean + 95% CI), p50/p95 latency, cost-per-eval-run, cost-per-1k-runs, total tokens consumed |
| **Pareto Frontier Plot** | ASCII / Markdown table of which models are non-dominated on (quality, cost) and (quality, latency) |
| **Quality-per-Dollar Score** | Single composite: `quality_score / cost_per_1k_runs`, with a 95% CI from bootstrap |
| **Decision Matrix** | "If your latency budget is X and your cost ceiling is Y, the best model is Z" — for several common (X, Y) regimes |
| **Sensitivity Analysis** | How the recommendation changes if pricing shifts ±20% or quality measurement noise is ±2 points |

---

## Augmented Eval Result Schema

The skill assumes your existing eval results are tabular (CSV / JSON / DataFrame). It extends the schema with these columns; existing columns are preserved unchanged.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| `model` | string | required | Canonical model id (e.g., `claude-sonnet-4-5`) |
| `input_tokens` | int | from runner | Count after any prompt caching credit |
| `output_tokens` | int | from runner | Generated tokens only |
| `latency_ms` | int | from runner | Wall-clock from request to last token |
| `pricing_source` | string | from `pricing.json` | E.g., `"pricing.json@2026-04-15"` — required for reproducibility |
| `cost_usd` | float | computed | `(input_tokens × in_price + output_tokens × out_price) / 1e6` |
| `quality_score` | float | from eval | Whatever your existing primary quality metric is (0–1 or 0–100) |

A reference `pricing.json` ships with the skill in `pricing.example.json`. Maintainers update this file when prices change; every eval run records *which* version of pricing was used so historical results stay reproducible. **Pricing is read from a local file — the skill never makes external calls to fetch prices.**

```json
{
  "_version": "2026-04-15",
  "_note": "Per-million-token prices in USD. Replace with current vendor prices before relying on results.",
  "models": {
    "claude-sonnet-4-5":  { "in_per_mtok": 3.00,  "out_per_mtok": 15.00 },
    "claude-opus-4-6":    { "in_per_mtok": 15.00, "out_per_mtok": 75.00 },
    "claude-haiku-4-5":   { "in_per_mtok": 0.80,  "out_per_mtok": 4.00  },
    "gpt-5-mini":         { "in_per_mtok": 0.25,  "out_per_mtok": 2.00  },
    "gpt-5":              { "in_per_mtok": 2.50,  "out_per_mtok": 10.00 }
  }
}
```

---

## Process

### Step 1: Intake
I'll ask:
> "Point me at your eval results (file or paste a sample). Tell me: which column is your primary quality score, and what's the desired direction (higher-is-better / lower-is-better)? What's your production latency budget at p95? What's your monthly cost ceiling, and at roughly how many eval-equivalent runs/month? Which `pricing.json` version should I use?"

### Step 2: Augment the Schema
Add `cost_usd` and confirm latency / token columns are populated. Flag any rows missing these — they get excluded from frontier analysis with an explicit count in the report.

### Step 3: Per-Model Aggregates
For each model:
- Mean quality with 95% CI (bootstrap, 1000 resamples)
- p50, p95, p99 latency
- Mean cost-per-run and projected cost at the user's stated runs/month
- Failure rate (eval rows with no completion / timeout / parse error)

### Step 4: Build the Pareto Frontier
A model is on the frontier iff no other model is **strictly better** on all three of (higher quality, lower cost, lower p95 latency). Output two tables:
- **Quality vs Cost** frontier
- **Quality vs Latency** frontier

Dominated models stay in the report but are flagged "dominated by *X*" with the reason.

### Step 5: Quality-per-Dollar Composite
`Q$ = quality_score / cost_per_1k_runs`. Report rank order and a bootstrapped 95% CI on the rank — if two models' CIs overlap, the report says so explicitly instead of pretending the ranking is precise.

### Step 6: Decision Matrix
A small table of "if your constraint is *X*, pick *Y*" recommendations — typically:

| Constraint regime | Recommendation |
|-------------------|----------------|
| No latency cap, no cost cap | Highest quality model |
| p95 < 2s, no cost cap | Best of the latency-feasible set |
| Cost < $X/1k runs, no latency cap | Highest quality within budget |
| p95 < 2s **and** cost < $X/1k runs | Best feasible point on the frontier |

### Step 7: Sensitivity Pass
Re-run the ranking with pricing perturbed ±20% and quality perturbed ±2 points. If the top recommendation flips under any plausible perturbation, the report flags it as "tight" and recommends collecting more eval samples before committing.

---

## Demo Output Snippet

**Frontier — Quality vs Cost (per 1k eval runs):**

| Model | Quality | Cost / 1k | p95 Latency | On Frontier? | Notes |
|-------|--------:|----------:|------------:|--------------|-------|
| claude-haiku-4-5 | 0.78 | $1.20 | 0.6s | ✅ | Cheapest viable point |
| gpt-5-mini       | 0.81 | $1.85 | 0.9s | ✅ | Best Q$ in this run |
| claude-sonnet-4-5| 0.88 | $6.40 | 1.3s | ✅ | Best quality under p95 < 2s |
| gpt-5            | 0.89 | $9.10 | 1.5s | — | Dominated by sonnet-4-5 (similar Q, higher $) |
| claude-opus-4-6  | 0.91 | $32.50| 2.4s | ✅ | Best quality overall, breaks p95 budget |

**Decision (latency budget p95 < 2s, cost ceiling $10/1k runs):**
> Pick **claude-sonnet-4-5**. It's the highest-quality model that satisfies both constraints. opus-4-6 has +0.03 quality but breaks the latency budget; gpt-5 is dominated.

---

## Tips
1. **Record `pricing_source` on every row.** Without it, "this model used to be a great deal" becomes unanswerable when prices move.
2. **Don't average latency — use p95.** A model with great median and a long tail is unsafe in production.
3. **Treat the frontier as a working set, not a leaderboard.** The "right" model depends on the *deployment*, not the eval.
4. **Collect failures separately.** A model that's 5% cheaper but 5% more likely to fail entirely is usually a bad trade — track failure rate as its own dimension.
5. **Re-run on every model release.** Frontiers shift quarterly; the model that won last quarter may be dominated this quarter.
