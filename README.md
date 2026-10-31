# Multi-Agent Retail Forecasting with AI Governance

Two agents produce and independently verify a retail demand forecast. The benchmark is
computed outside both agent loops and withheld from their namespaces, so neither agent
can reach the data, the scoring function, or the number it is measured against.

**Stack:** Python, LangGraph, GPT-4o, XGBoost, pandas.
**Data:** [Kaggle Walmart store sales](https://www.kaggle.com/competitions/walmart-recruiting-store-sales-forecasting) — 421,570 rows across 45 stores and 81 departments.

---

## Result

| Measure | Value |
| --- | --- |
| Baseline — same week last year | 2,184.19 WMAE |
| Residual XGBoost, 10-seed mean | **1,932.11 WMAE** |
| Seed range | 1,915.12 – 1,950.39 |
| Improvement | 252.08, or 11.54% |
| Rows scored | 3,038, identical for model and baseline |
| Seeds beating the baseline | 10 of 10 |

WMAE is mean absolute error in dollars, with holiday weeks weighted 5×. Lower is better.

The model predicts the *residual* — the gap between actual sales and the same week last
year — and adds the seasonal baseline back. Plain XGBoost on raw sales was tried first and
failed a 2011 backtest on 0 of 10 seeds.

---

## The point

The forecast is not the interesting part. The interesting part is that **the system cannot
mark its own homework.**

An agent asked to produce a result and then asked whether that result is good will say yes.
It has no independent reference. So:

- The baseline is computed in plain pandas **before any agent runs**, and pinned.
- The Architect's Python shell is a dictionary of four names: the run function, the baseline
  figure, the model config, and `numpy`. It cannot see the data, the metric, or the target.
- Seeding that shell with `globals()` — what most tutorials do — would hand the agent
  `BASELINE_WMAE` as a variable it could overwrite. A system that can rewrite its own
  benchmark has not been measured against one.

## The two agents

| | Architect | Auditor |
| --- | --- | --- |
| Namespace | `AGENT_NS` — `locked_run`, `BASELINE_WMAE`, `RESIDUAL_CONFIG`, `np` | `AUDITOR_NS` — `locked_run`, `np` |
| Tool | `python_tool` → `repl` | `auditor_tool` → `auditor_repl` |
| Mission | Execute the locked run and report the figures | Re-perform the run independently and report what *it* got |

The namespaces are separate objects, so the Auditor cannot see anything the Architect left
behind, and neither can reach the dataframes. Both shells share a lock, because
`PythonREPL` swaps `sys.stdout` to capture output and LangGraph runs tool calls in parallel
threads — see bug 3.

## How the audit works

Three independent judgements, combined by code at the end. None sees another's answer.

**1. Numeric — decided in code.** A parser extracts every number from the Architect's report
and matches it against the run's actual output. Five checks then run:

| Check | What it catches |
| --- | --- |
| Provenance | Figures in the report that trace to no result |
| Coverage parity | Model and baseline scored on different row sets |
| Baseline integrity | The benchmark drifting from its pinned value |
| Time-series integrity | Train/test overlap, leakage across the split |
| Holiday weighting | Recomputes at 1× and 5× to prove the weighting applies |

No model is consulted and no model is told the outcome.

**2. Re-performance — the second agent redoes the work.** Reading a report and re-performing
the work are different assurance activities. The Auditor calls `locked_run()` in its own
namespace and states the mean, the seed range, the rows scored and the baseline. It is never
asked whether the figures match — it reports, and code compares. An auditor that grades its
own agreement is the failure this project is about. Every figure it reports goes through the
same provenance parser used on the Architect, so the Auditor is held to its own standard.

**3. Claims — decided by the model, blind to both.** A separate stateless call reads the
report's written conclusions against a computed list of what the run actually covered, and
flags statements the evidence does not support.

**Human gate.** An audit pass is permission to *request* sign-off, not to release.
`release()` raises without a named approver, `approve()` refuses outright on an escalated
run, and every submission is logged with its approver and timestamps.

## What the tests prove

Three fixtures, differing only in what is wrong with them:

| Fixture | Numbers | Prose | Verdict |
| --- | --- | --- | --- |
| Genuine report | correct | supported | **APPROVED** |
| Row count falsified 3,038 → 3,075 | one unsourced | — | **ESCALATE** |
| Numerically perfect, overclaiming conclusions | all correct | two unsupported | **ESCALATE** |

A governance layer shown only correct input has not been tested — a working auditor and a
rubber stamp both say yes.

The third fixture is the one that earns the project. All five numeric checks passed. The
blind claims review still rejected it, catching *"the residual design generalises well
across the chain and is ready for production deployment"* and *"the seasonal naive baseline
is the correct benchmark at any scope"* — the second using this project's own 45-store
figures against it. No numeric check could have found either.

**The Auditor passes the falsified fixture, and that is correct.** It re-performed the
*run*, which was genuine; what was falsified was the Architect's *report*. Provenance is the
control for that risk, re-performance is not. Each control is scoped to a different failure,
and the combined verdict is computed from all three.

---

## Four bugs, one shape

Every failure found while building this was the same mistake in different clothes.

1. **The agent returned nothing, silently.** Its Python shell started with an empty
   namespace, so the function it was told to call did not exist in its world.

2. **The agent did arithmetic in its head.** The first passing run contained two
   LLM-computed figures presented as results. Both were correct; neither was verified. The
   automated check passed, because it only inspected one field.

3. **Output vanished with no error.** `PythonREPL` swaps `sys.stdout` to capture output;
   LangGraph runs tool calls in parallel threads. Two swaps interleaved and left stdout
   pointing at a dead buffer. No exception — just silence. Restoring `sys.__stdout__` does
   not fix this in Jupyter, because that is the terminal, not the kernel's stream: the real
   stream has to be captured from a healthy kernel before any agent runs.

4. **The Auditor approved a report it should have rejected.** Told the numeric checks had
   passed and *then* asked to review the written claims, it endorsed conclusions this
   project's own data contradicts. Asked the same question cold, with no knowledge of the
   verdict, it caught both.

**A check that runs after the conclusion is not a check.**

---

## Limitations

Stated plainly, because they are the first questions worth asking.

- **Scope.** The data pipeline and baseline analysis cover all 421,570 rows. The governed
  modelling run is Store 1 only — 3,375 training rows, 3,038 scored. 37 of the store's 3,075
  test rows lack a 52-week prior and are excluded from both model and baseline.
- **The baseline does not generalise.** Across all 45 stores, last-week naive (1,764.19)
  beats same-week-last-year (1,818.03). The seasonal baseline is a Store 1 property. Scaling
  this up is a benchmark problem, not a compute problem.
- **This is not an inventory system.** It forecasts weekly sales. There is no reorder point,
  no safety stock, no replenishment policy and no service-level target.
- **Not deployed.** This is a prototype of the control pattern. Nothing runs on a schedule,
  reads from a warehouse, or pages anyone.
- **The Architect's mission is narrow.** It executes a locked run and reports it. A mission
  giving the agent genuine latitude is written but not yet run, so the Auditor has not yet
  had to reject work it did not expect.
- **The claims reviewer is a single model call.** It caught both overclaims on the fixture
  above, but it is a judgement, not a proof, and it has not been tested against adversarial
  phrasing.

---

## Running it

```
pip install pandas numpy xgboost scikit-learn langgraph langchain-openai langchain-experimental
setx OPENAI_API_KEY "your-key"     # Windows; use export on macOS/Linux
```

Download the dataset from Kaggle, set `DATA_DIR` in the second cell, and run
`Walmart_Forecasting_Langgraph.ipynb` top to bottom. The agent sections need an API key;
everything up to and including the benchmark runs without one. No key appears anywhere in
the notebook.

The committed notebook runs 49 code cells in sequence with no gaps, no re-runs and no error
outputs, so `Restart & Run All` reproduces the results above. A full pass of all three audit
fixtures costs a few cents.

---

*The code in this notebook is AI-assisted. The control structure, the design decisions and
the failures documented above are mine.*
