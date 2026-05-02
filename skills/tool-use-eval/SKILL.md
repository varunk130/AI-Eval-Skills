---
name: tool-use-eval
description: 'Evaluation pattern for agents that call tools / functions — measures tool selection accuracy, argument correctness, recovery from tool errors, and avoidance of unnecessary tool calls. Pairs with eval-suite-planner. Use when: tool-use eval, function-calling eval, agent tool selection, tool invocation accuracy, tool argument validation, tool error recovery, agent tools, function-calling agent, MCP eval.'
---

# Tool-Use Eval

Evaluation pattern for agents whose job is to invoke tools / functions / MCP capabilities. Conversational quality and task-completion evals miss the failure modes specific to tool-use; this skill adds the four dimensions that actually predict whether a tool-using agent is production-ready.

## Core Principle

**A tool-using agent has four ways to fail that pure-conversation agents don't.** Generic quality scores can be high while every one of these is broken. This eval pattern separates them so improvements target the right failure mode.

## The Four Tool-Use Dimensions

| Dimension | What It Measures | Failure Example |
|-----------|------------------|-----------------|
| **Tool Selection Accuracy** | Did the agent pick the right tool for the request? | Used `search_web` when `lookup_internal_kb` was the right call |
| **Argument Correctness** | Were the arguments well-formed and schema-valid? | Required field missing, type mismatch, hallucinated parameter |
| **Error Recovery** | When a tool returned an error, did the agent recover sensibly? | Repeats the same failing call, or gives up silently |
| **Restraint** | Did the agent avoid unnecessary tool calls? | Calls 5 tools when 1 would do; calls a tool when no tool was needed |

The four dimensions are **independently** scorable — improvements often help one and hurt another (e.g., tightening argument validation can increase Restraint failures).

## Scoring Anchors (0–3 per dimension)

### Tool Selection Accuracy
| Score | Anchor |
|-------|--------|
| 0 | Wrong tool; result has no chance of satisfying the request |
| 1 | Plausible-looking tool but mismatched to the request's actual intent |
| 2 | Right tool family, but a more specific / cheaper tool existed and was missed |
| 3 | Most appropriate tool selected on the first call |

### Argument Correctness
| Score | Anchor |
|-------|--------|
| 0 | Schema-invalid (missing required field, type mismatch, hallucinated param) |
| 1 | Schema-valid but semantically wrong |
| 2 | Schema-valid and semantically reasonable, with one suboptimal field |
| 3 | Schema-valid, semantically optimal, no extraneous fields |

### Error Recovery
| Score | Anchor |
|-------|--------|
| 0 | Agent ignored the error or repeated the same failing call > 1× |
| 1 | Agent stopped without explaining the failure to the user |
| 2 | Agent tried a different approach but did not surface the failure clearly |
| 3 | Sensible alternative tried *or* failure clearly surfaced with next-step guidance |

### Restraint
| Score | Anchor |
|-------|--------|
| 0 | Tool called when no tool was needed (request answerable from prior context) |
| 1 | Multiple tools called for what should have been a single call |
| 2 | Tool count is right but ordering wasted a call |
| 3 | Minimum-necessary tool calls, ordered well |

## Canonical Test Suite

The skill ships an 18-case seed suite covering common failure shapes:

| Case Type | Count | Tests |
|-----------|------:|-------|
| **No tool needed** | 3 | Restraint baseline — request answerable from context |
| **Single obvious tool** | 3 | Selection + Argument basics |
| **Two plausible tools** | 3 | Selection discrimination |
| **Multi-tool chain** | 3 | Sequencing + Restraint |
| **First call errors** | 3 | Error Recovery |
| **Ambiguous request** | 3 | Pre-call clarification vs. guessing |

Extend with 5–10 cases from your actual production traffic.

## Trace Capture Schema

Every test case captures:
- `request` — the user prompt
- `tool_calls[]` — each call's name, arguments, timestamp, result, error (if any)
- `final_response` — the agent's reply to the user
- `expected_tools[]` — gold-standard tool sequence (for Selection scoring)
- `expected_no_tools` — boolean (for Restraint scoring)

Without the trace, post-hoc scoring is impossible.

## Failure Taxonomy

Standardized labels so triage rolls up cleanly:

| Code | Meaning |
|------|---------|
| `SEL-WRONG` | Wrong tool selected |
| `SEL-SUBOPTIMAL` | Right family, wrong specific tool |
| `ARG-INVALID` | Schema validation failed |
| `ARG-SEMANTIC` | Schema-valid but semantically wrong |
| `REC-SILENT` | Failure not surfaced to user |
| `REC-LOOP` | Same failing call repeated |
| `RES-OVERCALL` | More tool calls than needed |
| `RES-UNNECESSARY` | Tool call when none was needed |

## Process

1. **Inventory the tools** — list with schemas and one-line use cases
2. **Adopt / extend the seed suite** — start with the 18 canonical cases
3. **Run with full trace capture** — request + every call + final response
4. **Score each case on all four dimensions** independently
5. **Aggregate by failure type** — which dimension is dragging?
6. **Track over time** — re-run on every model / prompt / tool-schema change

## Pairs With

- **eval-suite-planner** — designs the broader plan; tool-use-eval is the tool-use chapter
- **eval-result-interpreter** — produces SHIP / ITERATE / BLOCK verdicts
- **eval-triage-and-improvement** — walks through failures, recommends fixes
- **cost-quality-frontier** — Restraint failures show up as cost regressions

## Tips
1. **Score the trace, not the response.** Wasteful tool sequences hide if you only judge output.
2. **Restraint regressions are the most expensive at scale** — a 2× tool-call regression doubles inference + tool-execution costs.
3. **Don't conflate Selection and Argument failures** — different fixes (prompt vs. schema clarity).
4. **Test the no-tool baseline** — over-calling is the hardest failure mode to detect without it.
