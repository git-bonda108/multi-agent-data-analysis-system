# Architecture

This document describes the system as it exists in the code. Line references are to the current HEAD.

## Component map

| Component | File | Responsibility |
|---|---|---|
| Streamlit app / entry point | `data-insights.py` (`main`) | Page layout, five tabs, file upload, API-key gate, quick-action buttons |
| Data loading | `data-insights.py` (`load_combined_data`) | Reads each uploaded CSV/Excel file and concatenates them into one dataframe; cached with `@st.cache_data` |
| Agent factory | `data-insights.py` (`create_agents`) | Builds the five `Agent` objects and their `@function_tool` wrappers as closures over the loaded dataframe |
| Router | `data-insights.py` (`route_query`) | Keyword-matches the query text and dispatches to one agent via `Runner.run_sync`; also owns the result-table fallback logic |
| Result extraction | `data-insights.py` (`parse_text_to_dataframe`, `extract_dataframe_from_result`) | Recovers tabular structure from the agent's text output |
| Data tools | `tools/data_tools.py` | `query_dataframe`, `filter_dataframe`, `group_by_aggregate`, `detect_outliers`, `get_data_info`, `get_data_quality_summary`, `execute_complex_query`, `multi_condition_filter`, `advanced_group_by` |
| Statistical tools | `tools/statistical_tools.py` | `calculate_statistics`, `analyze_correlations`, `feature_relationships` (Pearson + one-way ANOVA), `hypothesis_test` (one-sample t-test, D'Agostino normality), `distribution_analysis`, `feature_importance` |
| Visualization tools | `tools/visualization_tools.py` | Histogram, scatter, correlation heatmap, bar, box, line, pair plots; each renders via `st.pyplot` as a side effect and returns a confirmation string |
| Formatting tools | `tools/formatting_tools.py` | Coercion of dicts/lists/strings to `DataFrame`, correlation long-format table, statistics rounding |
| Unused module | `tools/advanced_data_tools.py` | SQL-like query helpers (`execute_sql_like_query`, `parse_natural_language_query`, `complex_query`). Not imported by `tools/__init__.py` or the app; `data_tools.py` contains its own overlapping implementations. Kept in tree but dead at runtime. |

## The five agents

`create_agents` constructs five `Agent` instances from the OpenAI Agents SDK:

1. **Analysis Agent** — 10 tools for querying, filtering (single and multi-condition with AND/OR), grouping/aggregation, sorting, dataset info, data quality, outlier detection (IQR or z-score), and SQL-like query execution.
2. **Statistical Agent** — 6 tools for descriptive statistics, correlation matrices (Pearson/Spearman/Kendall), pairwise feature relationships, hypothesis tests, distribution analysis, and correlation-based feature importance.
3. **Visualization Agent** — 7 chart tools.
4. **Formatting Agent** — 4 table-formatting tools.
5. **Orchestrator Agent** — instructions describe routing between the other four, but it is constructed with `tools=[]`, no handoffs are configured, and `route_query` never selects it. It is inert in the current code path; routing is deterministic Python, not LLM-driven.

Every tool is a closure over the dataframe captured when `create_agents` ran, so tools take only query-specific parameters and the LLM never has to (and cannot) address a different dataset.

## Data flow, end to end

1. **Upload** — sidebar `st.file_uploader` accepts one or more `.csv`/`.xlsx` files; `load_combined_data` concatenates them row-wise (`pd.concat`, `ignore_index=True`). No schema alignment is attempted beyond pandas' default column union.
2. **Session state** — the combined dataframe is stored in `st.session_state.dataframe`; agents are created once and stored in `st.session_state.agents`.
3. **Query** — each tab's text input (or quick-action button) calls `route_query(query, agents, dataframe)`.
4. **Routing** — a first-match keyword cascade: SQL-ish terms (`order by`, `where`, `filter`, `top`, ...) → Analysis; chart terms (`plot`, `histogram`, `heatmap`, ...) → Visualization; `statistic`/`correlation`/`relationship`/... → Statistical; `format`/`table`/... → Formatting; anything else defaults to Analysis. Because SQL terms are checked first, "plot the top 10" routes to Analysis, not Visualization — an ordering trade-off visible in the code.
5. **Agent run** — `Runner.run_sync(agent, query)` executes one agent loop: the model reads its instructions (which embed the column list), chooses among its function tools, the tools run pandas/scipy locally, and the model composes `final_output` text.
6. **Table recovery** — `route_query` and the tab code try, in order: `extract_dataframe_from_result` (pipe-table parsing, correlation-pattern regex, whitespace/CSV parsing, and for sort/limit queries direct re-execution against the dataframe), then `formatting_tools.format_as_table`, then — for statistical queries — recomputing the correlation/statistics/relationship table directly from the dataframe, bypassing the LLM text entirely.
7. **Render** — tables via `st.dataframe` (numeric columns rounded to 4 decimals), remaining text in a code block or markdown. Charts short-circuit this: the visualization tools have already drawn to the page by the time the run returns.

## Orchestration analysis: what runs parallel, sequential, async

Nothing runs in parallel and nothing is async. Each user interaction triggers exactly one synchronous agent run (`Runner.run_sync`), which blocks the Streamlit script until the OpenAI API round-trips (including any tool-call iterations inside the run) complete. There is no fan-out, no multi-agent conversation, and no queue. The rationale visible in the design: one specialist per query keeps cost at a single agent run, keeps behavior debuggable (the router is a readable Python function), and fits Streamlit's rerun-per-interaction execution model, which has no natural place for background work.

## State and context engineering

- **Session store:** Streamlit `st.session_state` (per browser session, in-process). Keys: `dataframe`, `agents`, `data_loaded`. There is no database and nothing persists across a server restart.
- **Caching:** `@st.cache_data` on `load_combined_data` keyed on the uploaded file objects.
- **Per-run context:** each agent run receives only the user's query string; there is no chat history. Grounding comes from two channels: the column list interpolated into the agent's instructions at creation time, and tool return values (stringified dataframes via `.to_string()`).
- **Context bounding:** partial. `order_by` truncates to `head(50)`; box and pair plots cap at 5 columns; `query_dataframe` returns head/tail samples. But `filter_data`, `execute_sql_query`, and `detect_outliers` return the full matching rows as text, so a broad filter on a large file pushes an unbounded string into the model's context — a real cost/limit exposure on large datasets.
- **Stale-schema limitation:** agents are only created when `st.session_state.agents` is empty. Uploading a *different* file in the same session replaces the dataframe but not the agents, whose tool closures and instruction-embedded column lists still reference the first dataframe. The deterministic fallback paths use the new dataframe, so the two channels can disagree until the session is restarted.

## Design decisions and trade-offs visible in the code

- **Deterministic router over LLM orchestration.** The Orchestrator agent exists as a design sketch, but the shipped router is keyword matching: zero latency and zero cost for routing, at the price of first-match misroutes and no multi-agent coordination for compound queries.
- **Belt-and-braces result pipeline.** The code does not trust the LLM to return clean tables: three extraction fallbacks, ending in direct recomputation for statistical queries. This guarantees tabular output for the common cases even when the model's prose is unparseable — at the cost of duplicated logic (the same correlation table can be produced by the agent tool, the parser, or the direct call).
- **Tools as pure pandas with string returns.** Tool bodies never call the LLM and raise/return errors as strings (often including the valid column list), which the model can use to self-correct within the run.
- **Broad exception swallowing.** Nearly every tab, tool wrapper, and parser is wrapped in `try/except` that degrades to a warning or a raw-text display. The UI stays up, but failures are indistinguishable from empty results in several paths (see HARDENING).
- **Duplication left in tree.** `tools/advanced_data_tools.py` duplicates and predates functions now in `data_tools.py` but is never imported; `format_statistics_table` is defined twice (identically) in `formatting_tools.py`, the second definition shadowing the first. Harmless at runtime; noted here so readers don't treat the unused module as live code.
- **`model_name` accepted but unused.** `create_agents(dataframe, model_name=None)` documents model selection, but no `Agent` receives a `model` argument; the SDK default applies.
- **Global SSL verification disable.** `data-insights.py` line 20 sets `ssl._create_default_https_context = ssl._create_unverified_context` for the whole process — evidently a workaround for a local certificate problem. Security implications and remediation are covered in [HARDENING.md](HARDENING.md).
