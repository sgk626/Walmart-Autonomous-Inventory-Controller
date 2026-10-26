# Walmart Autonomous Inventory Controller

A two-agent forecasting system where the benchmark is computed outside the agent
loop, so the agents cannot grade their own work.

Built on the [Kaggle Walmart store sales](https://www.kaggle.com/competitions/walmart-recruiting-store-sales-forecasting)
dataset — 421,570 rows across 45 stores and 81 departments.

---

## Result

| Measure | Value |
| --- | --- |
| Baseline — same week last year | 2,184.19 WMAE |
| Residual XGBoost, 10-seed mean | **1,932.11 WMAE** |
| Seed range | 1,915.12 – 1,950.39 |
| Improvement | 252.08, or 11.5% |
| Rows scored | 3,038, identical for model and baseline |
| Seeds beating the baseline | 10 of 10 |

WMAE is mean absolute error in dollars, with holiday weeks weighted 5×. Lower is
better.

---

## The point

The forecast is not the interesting part. The interesting part is that **the
system cannot mark its own homework.**

An agent asked to produce a result and then asked whether that result is good
will say yes. It has no independent reference. So:

- The baseline is computed in plain pandas **before any agent runs**, and pinned.
- The agent's Python shell is given a dictionary of four names: the run function,
  the baseline figure, the model config, and `numpy`. It cannot see the data, the
  scoring function, or the target result.
- Seeding that shell with `globals()` — what most tutorials do — would hand the
  agent `BASELINE_WMAE` as a variable it could overwrite. A system that can
  rewrite its own benchmark has not been measured against one.

## How the audit works

Two independent judgements that never see each other's answer.

**Numeric — decided in code.** A parser extracts every number from the agent's
report and matches it against the run's actual output. Five checks then run:

| Check | What it catches |
| --- | --- |
| Provenance | Figures in the report that trace to no result |
| Coverage parity | Model and baseline scored on different row sets |
| Baseline integrity | The benchmark drifting from its pinned value |
| Time-series integrity | Train/test overlap, leakage across the split |
| Holiday weighting | Recomputes at 1× and 5× to prove the weighting applies |

**Claims — decided by the model, blind to the above.** A separate call reads the
report's written conclusions against a list of what the run actually covered, and
flags statements the evidence does not support.

**Human gate.** An audit pass is permission to *request* sign-off, not to release.
`release()` raises without a named approver, and `approve()` refuses outright on an
escalated run.

## What the tests prove

Three fixtures, differing only in what is wrong with them:

| Fixture | Numbers | Prose | Verdict |
| --- | --- | --- | --- |
| Genuine report | correct | supported | **APPROVED** |
| Row count falsified 3,038 → 3,075 | one wrong | — | **ESCALATE** |
| Numerically perfect, overclaiming conclusions | all correct | unsupported | **ESCALATE** |

A governance layer shown only correct input has not been tested — a working
auditor and a rubber stamp both say yes.

---

## Four bugs, one shape

Every failure found while building this was the same mistake in different clothes.

1. **The agent returned nothing, silently.** Its Python shell started with an empty
   namespace, so the function it was told to call did not exist in its world.

2. **The agent did arithmetic in its head.** The first passing run contained two
   LLM-computed figures presented as results. Both were correct; neither was
   verified. The automated check passed, because it only inspected one field.

3. **Output vanished with no error.** `PythonREPL` swaps `sys.stdout` to capture
   output; LangGraph runs tool calls in parallel threads. Two swaps interleaved and
   left stdout pointing at a dead buffer. No exception — just silence.

4. **The Auditor approved a report it should have rejected.** Told the numeric
   checks had passed and *then* asked to review the written claims, it endorsed
   conclusions this project's own data contradicts. Asked the same question cold,
   with no knowledge of the verdict, it caught both.

**A check that runs after the conclusion is not a check.**

---

## Limitations

Stated plainly, because they are the first questions worth asking.

- **Scope.** The data pipeline and baseline analysis cover all 421,570 rows. The
  governed modelling run is Store 1 only — 3,375 training rows, 3,038 scored.
- **The baseline does not generalise.** Across all 45 stores, last-week naive
  (1,764.19) beats same-week-last-year (1,818.03). The seasonal baseline is a
  Store 1 property. Scaling this up is a benchmark problem, not a compute problem.
- **Not deployed.** This is a prototype of the control pattern. Nothing runs on a
  schedule, reads from a warehouse, or pages anyone.
- **The Architect's mission is narrow.** It executes a locked run and reports it.
  A mission giving the agent real latitude is written but not yet run.

## Running it

```
pip install pandas numpy xgboost scikit-learn langgraph langchain-openai langchain-experimental
setx OPENAI_API_KEY "your-key"     # Windows; use export on macOS/Linux
```

Download the dataset from Kaggle, set `DATA_DIR` in the second cell, and run
`Walmart_Forecasting_Langgraph.ipynb` top to bottom. The agent sections need an
API key; everything up to and including the benchmark runs without one.

A full pass of all three audit fixtures costs a few cents.

---

*The code in this notebook is AI-assisted. The design decisions, the failures
found, and the control structure are documented above.*

