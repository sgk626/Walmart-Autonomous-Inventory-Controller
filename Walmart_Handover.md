# Walmart Autonomous Inventory Controller — handover
**As of 26 Oct 2026, 11:00 MYT**

Drop this into the "Walmart agentic ai project" so a new chat starts warm.

> **Status: the core is built.** Both agents run, the governed pipeline
> completes end to end, and the Auditor passes a negative test. This was the
> milestone the whole roadmap was gated behind, reached seven days ahead of
> the revised target. What remains is hardening and write-up, not building.

---

## What this project is

A two-agent system (Architect / Auditor) for retail demand forecasting on the
Kaggle Walmart store-sales data. The portfolio point is **AI governance**, not
forecasting accuracy: the agents are measured against a benchmark computed
outside the agent loop, so they cannot grade their own work.

Stack: Python, LangGraph (`create_react_agent` + `PythonREPL`), GPT-4o, XGBoost.
Live notebook: `Walmart_Forecasting_Langgraph.ipynb`.

Data: `D:/sgk62/Documents/Own Projects/Walmart Autonomous Inventory Controller/archive`
(train.csv 421,570 rows; stores.csv 45; features.csv 8,190). Loaded straight
into pandas — **this notebook does not create `inventory.db`.** The SQLite work
lives only in the older 61-cell copy.

---

## The pinned ground truth — do not recompute

| Item | Value |
| --- | --- |
| BASELINE_WMAE | **2,184.19** (seasonal naive, Store 1) |
| TARGET_2012 | **1,932.11** (residual XGBoost, 10-seed mean) |
| Seed range | 1,915.12 to 1,950.39 |
| Improvement | 252.08, or 11.5% |
| Scope | Store 1 only |
| Split | train < 2012-01-01, test >= 2012-01-01 |
| Holiday weight | 5x |
| Store 1 test rows | 3,075 |
| COMPARABLE rows | **3,038** (37 have no 52-week prior) |

Caveat: across all 45 stores the baseline ranking reverses (last-week 1,764 vs
seasonal 1,818), so the seasonal baseline is a Store-1 property, not a global
one. Say this if asked — it shows you checked.

---

## Done

- Deterministic merge, WMAE with 5x holiday weighting, no-leakage time split
- Both naive baselines; BASELINE_WMAE pinned at 2,184.19
- Residual XGBoost design locked; `RESIDUAL_CONFIG` drives the model
- `locked_run()` reproduces TARGET_2012 exactly and asserts coverage parity
  and baseline integrity on every call
- **Empty agent return fixed.** Cause was the tool, not the prompt:
  `PythonREPL()` starts with an empty namespace, so `locked_run` was never in
  scope. Fixed by seeding a curated dict — entry point and reference figures
  only. Data frames, the metric function and TARGET_2012 are deliberately
  withheld. This is the governance control, not a convenience: seeding with
  `globals()` would let the agent rebind its own benchmark.
- **Architect run completed**, reproducing 1,932.11 on 3,038 rows
- **Unverified-arithmetic finding.** First successful run passed every check
  and computed the delta and percentage in-model rather than from the dict.
  Both were correct; neither was verified. Fixed by returning the derived
  figures from `locked_run()`. Keep both trajectories — same agent, one
  sentence of mandate changed, two unverified figures to none.
- **AUDITOR_PROMPT rewritten.** Removed the Monte Carlo mandate (referred to
  nothing) and the `WMAE <= 10%` rule (uncomputable — WMAE is in dollars).
  Added coverage parity as an instruction. Four mandates, all computable.
- **Negative test passing.** Report with `rows_scored` falsified 3,038 -> 3,075
  returns ESCALATE and flags the figure as NO SOURCE; genuine report returns
  APPROVED.

---

## Open — in priority order

1. **Write it up.** The build is ahead of the write-up now. The conclusion
   markdown cell exists; the README and the interview narrative do not.

2. **Prepare the scale answer** — what breaks going from Store 1 to all 45:
   - The seasonal-naive baseline stops being the right benchmark (ranking
     reverses across 45 stores), so the benchmark becomes per-series.
   - Agent cost and latency scale with the number of series; the loop runs
     once per scope, so 45 stores is a batching problem.
   - Coverage parity must be recomputed per store — the 37 missing-prior rows
     in Store 1 are not a fixed fraction elsewhere.

3. **The real Architect mission.** `architect_mission` (the "beat the baseline"
   cell) is still unrun and is a better portfolio piece than the harness test,
   because it gives the agent latitude and the Auditor something to reject.
   Blocked on a design choice: rebuild the SQLite table so the agent queries a
   database, or seed the pandas frames into its namespace. SQLite reads better
   in the write-up.

4. **Harden the negative test.** Current one is easy mode — a single integer
   flatly contradicted by a dict field. Harder cases: a float at the wrong
   precision, or a figure internally consistent but wrongly derived.

5. **Claim enumeration is still a judgement.** The Auditor counted 5 claims in
   one run and 6 in another on the same five numbers (it split the range).
   Both internally consistent, but if that needs to be airtight the claims
   must be extracted by code, not by the model.

6. **Optional: retrieval in the Auditor** — a written forecasting governance
   policy it retrieves from and cites. Turns one artifact into evidence of both
   RAG and agentic design. Only worth doing once 1 and 2 are done.

---

## Housekeeping

- API key is a Windows user env var (`setx OPENAI_API_KEY`), expires ~25 Nov.
  Nothing to scrub before a GitHub push — no key appears in the notebook.
- The notebook carries a markdown cell reading "The code is AI assisted."
  Keep it. Honesty about method is a strength in interview, and the design
  decisions are yours regardless.
- The older 61-cell `Walmart_Forecasting.ipynb` is superseded. It holds the
  SQLite setup if item 3 goes that way; otherwise ignore it.

---

## Working agreement

- Full working code plus plain-language explanation, not guided syntax lessons.
- Honest assessment over encouragement — flag overstatements rather than
  letting them through.
- **Targets:** interview-ready by 16 Nov. Job applications have first call on
  weeknights until an offer is in hand; this project is no longer the
  bottleneck for that.
