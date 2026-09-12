# Hardening

Current security and operational posture as found in the code, followed by a staged path to production. This is a single-process Streamlit app that sends user-uploaded data to the OpenAI API; the ladder is sized to that reality.

## Current posture

**Secrets.** The only credential is `OPENAI_API_KEY`, read from the environment or a local `.env` via python-dotenv. `.env` is gitignored, `setup.sh` writes only a placeholder, and a scan of HEAD found no committed credentials. Good baseline.

**Transport security — one deliberate hole.** `data-insights.py` line 20 disables TLS certificate verification process-wide:

```python
ssl._create_default_https_context = ssl._create_unverified_context
```

Every HTTPS connection the process makes — including OpenAI API calls carrying the API key and the user's data — is exposed to man-in-the-middle interception. This is the single most important line to remove before any deployment beyond a trusted laptop.

**Authentication and authorization.** None. Streamlit serves the app to anyone who can reach the port; every visitor shares the operator's OpenAI key and spend. There is no rate limiting and no per-user cost cap.

**Input handling.** Uploaded CSV/Excel files are parsed by pandas in-process with no size limit beyond Streamlit's default uploader cap. Natural-language queries are interpreted via regex and pandas operations — there is no `eval`/`exec` and no SQL engine, so classic injection does not apply; the realistic abuse is cost (queries that stringify large result sets into model context) and resource exhaustion (very large uploads). `unsafe_allow_html=True` is used only for static, hard-coded CSS/header markup, not for user- or model-derived content — keep it that way, since rendering model output with that flag would create an injection path.

**Data privacy.** Uploaded data rows flow to the OpenAI API through tool results embedded in agent context. Nothing is persisted server-side (in-memory session state only), but there is no notice to users that their data leaves the machine.

**Error handling and observability.** Errors are caught broadly and rendered as UI messages or returned as strings into agent context. There is no logging, no tracing, no metrics, and several `bare except:` clauses that discard the failure entirely. Operationally the app is a black box: no health endpoint, no cost accounting, no record of what was asked or answered.

**Dependencies.** `requirements.txt` pins only lower bounds (`openai>=2.8.0`, `openai-agents>=0.6.0`) and leaves the rest floating; no lockfile, no vulnerability scanning. The model is likewise unpinned (SDK default), so behavior can change under a dependency or default-model bump with no code change.

## Ladder to production

### Stage 1 — Identity, keys, transport (before anyone else uses it)

1. Delete the SSL-bypass line; if a corporate proxy motivated it, point `SSL_CERT_FILE`/`REQUESTS_CA_BUNDLE` at the proxy's CA bundle instead.
2. Put authentication in front of the app (reverse proxy with OIDC/SSO, or Streamlit Community Cloud's viewer allowlist). One shared page, one shared key — so gate the page.
3. Move the API key to a managed secret store (deployment platform secrets; not a `.env` on the server). Rotate it if it was ever used while TLS verification was disabled.
4. Pin the model explicitly by passing `model=` when constructing agents (the `model_name` parameter of `create_agents` is currently accepted but unused — wire it through), and pin dependencies with a lockfile.

### Stage 2 — Monitoring and cost control

1. Add structured logging around `route_query`: query, chosen agent, duration, exception (the function already has the single choke point; instrument it).
2. Enable the Agents SDK's tracing so each run's tool calls and token usage are inspectable; export token counts as a per-session cost metric.
3. Replace `bare except:` clauses in the extraction path with logged, typed exception handling so parser regressions become visible.
4. Add per-session query quotas and a hard cap on rows serialized into tool output (e.g. `head(100)` in `filter_data`/`execute_sql_query`, mirroring the cap `order_by` already has).

### Stage 3 — Deployment engineering

1. Containerize (python:3.11-slim, non-root user, `pip install` from the lockfile, `streamlit run` as entrypoint) with health checks on Streamlit's `/_stcore/health` endpoint and memory/CPU limits sized for pandas workloads.
2. Enforce an upload size limit (`server.maxUploadSize` in Streamlit config) and validate that multi-file uploads share a schema before `pd.concat`, failing with a clear message instead of silently unioning columns.
3. Add timeout and bounded-retry handling around `Runner.run_sync` so an OpenAI outage degrades to a clear message rather than a hung script run.
4. Rebuild agents when the dataframe changes (the current session logic reuses agents created for the first upload, leaving tool closures bound to stale data — correctness as well as hygiene).

### Stage 4 — Compliance and data governance

1. State in the UI that uploaded data is sent to OpenAI for processing; link OpenAI's API data-usage terms. If datasets may contain personal data, this disclosure is a prerequisite, not a nicety.
2. Define retention: today nothing persists beyond the session, which is a strong default — document it and keep logs (Stage 2) free of raw data rows, logging shapes and column names rather than values.
3. If regulated data is in scope, route through an enterprise OpenAI endpoint with a data-processing agreement, and add an allowlist of permitted upload sources.

## Secrets found at HEAD

None. The scan of the working tree at HEAD found no API keys, tokens, connection strings, committed `.env` files, or key material; no redactions were required.
