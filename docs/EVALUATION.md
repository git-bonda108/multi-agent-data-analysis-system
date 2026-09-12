# Evaluation

An honest account of what is tested today, what the code visibly guards against, and what an adequate evaluation harness would look like. Nothing below claims coverage that does not exist.

## What automated tests exist

**`test_order_data.py`** is the only executable test in the repository. It is a script (not pytest): a `test(name, func, *args)` helper calls each function and counts pass/fail, where "pass" means *the call did not raise* — return values are never asserted against expected data.

- Run with: `python test_order_data.py`
- **Precondition:** it loads `~/Downloads/mock_order_data_analysis.csv` and exits immediately if that file is absent. The fixture is **not** in the repository, so the script does not run from a fresh clone as-is. Any CSV with `TotalAmount`, `Quantity`, and `OrderStatus` columns placed at that path satisfies it.
- Coverage (17 calls): `data_tools` — quality summary, `execute_complex_query` (ORDER BY asc/desc, LIMIT, WHERE >, WHERE <, GROUP BY, and combinations), `multi_condition_filter` (AND/OR), `advanced_group_by`, `get_data_info`; `statistical_tools` — statistics, correlations, feature relationships, distribution analysis, feature importance.
- **Not covered:** everything in `data-insights.py` (routing, table extraction, session handling, UI), `visualization_tools.py`, `formatting_tools.py`, and all agent/LLM behavior. No API calls are made by the test.

There is no pytest suite, no CI workflow, and no coverage tooling in the repository.

## Manual test documentation

`TEST_CASES.md`, `TEST_CASES_DETAILED.md`, `QUICK_TEST_REFERENCE.md`, and `TESTING_COMPLETE.md` at the repository root are checklists of manual scenarios written during development (queries to type into each tab and expected qualitative behavior). They are documentation, not executable artifacts; no harness runs them, and any pass/fail statements in them reflect one-off manual sessions, not reproducible results. No quantitative quality metrics (accuracy, latency, cost) exist anywhere in the code or docs.

## Edge cases the code visibly handles

Enumerated from the source, with locations:

- **Missing API key** — `main` checks `OPENAI_API_KEY` and renders setup instructions instead of crashing (`data-insights.py`).
- **Empty dataframe** — `query_dataframe` and `filter_dataframe` return early on `dataframe.empty`.
- **Unknown columns** — `filter_dataframe`, `group_by_aggregate`, `detect_outliers`, `hypothesis_test`, `feature_importance`, `advanced_group_by` raise `ValueError`s that name the missing column; the tool wrappers convert these to strings that include the *valid* column list so the model can retry within the run.
- **Non-numeric input to numeric analyses** — `detect_outliers` and `hypothesis_test` reject non-numeric columns; `calculate_statistics`/`analyze_correlations` return empty frames when there are no (or fewer than 2) numeric columns, and the tool wrappers translate that to explanatory text.
- **NaN handling** — statistical functions operate on `.dropna()` data; `mode()` guards the empty-mode case; string `contains` filters pass `na=False`.
- **Degenerate ANOVA groups** — `feature_relationships` only runs `f_oneway` when there are ≥2 groups, all non-empty, and swallows per-pair failures rather than aborting the sweep.
- **Fallback chains** — `query_data` falls back from `execute_complex_query` to `query_dataframe`; table extraction cascades through parser → formatter → direct recomputation; the router defaults to the Analysis agent when no keyword matches.
- **UI resilience** — every tab body is wrapped in `try/except` with a user-facing error and a "refresh/re-upload" hint, so one failing tab does not take down the page.

## What the code does not handle

No timeouts, no retries, and no rate-limit or quota handling around `Runner.run_sync` — an OpenAI API failure surfaces as a caught exception string in the UI. No guard against oversized tool outputs (a broad filter stringifies every matching row into model context). No validation that uploaded files parse or that multiple files share a schema (`pd.concat` silently unions columns). The bare `except:` clauses in the extraction path can mask real defects as "no table found."

## Proposed evaluation harness

None of the following exists yet; it is the design this system should adopt.

1. **Deterministic layer (pytest, no API key needed).** Convert `test_order_data.py` into a pytest suite with a small golden CSV committed under `tests/fixtures/` (e.g. 50 rows, numeric + categorical + nulls + duplicates + one outlier). Assert *values*, not just non-raising: sorted order, filter row counts, known correlation coefficients, quality-summary null counts. Add unit tests for `route_query` (a table of query → expected agent) and for `parse_text_to_dataframe`/`extract_dataframe_from_result` against captured agent-output strings — these parsers are the highest-regression-risk code in the repo and currently have zero tests.
2. **Agent layer (recorded or budget-capped live runs).** A golden set of ~20 natural-language queries with expected outcomes expressed as checkable properties (which agent handled it, which tool(s) were called, does the extracted table have the expected shape/values). Run against the live API in a manually triggered CI job with a per-run cost cap; record transcripts as fixtures for offline regression comparison.
3. **Gates.** CI (currently absent) should require: deterministic suite green on every push; agent-layer suite green before release; no `bare except` additions (lint rule). Track three metrics per agent-suite run: routing accuracy (correct agent), task success rate (property checks pass), and mean tokens per query — all reported from the harness, none hand-claimed.
