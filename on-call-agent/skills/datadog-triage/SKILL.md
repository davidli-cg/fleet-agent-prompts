---
name: datadog-triage
description: Investigate Datadog logs, APM traces/spans, and metrics to diagnose service/infra incidents and answer questions. Use for Datadog monitor alerts on the AI/ML services, and for Q&A about logs, APM spans, metrics, hosts, deploys, monitors, or RUM/sessions.
---

# Datadog Triage

## Scope
Answer questions and diagnose incidents that can be answered by investigating Datadog logs, APM traces/spans, and metrics. Refuse anything that cannot (defer to the router's out-of-scope rules). Slack formatting, routing, and reply conventions are handled by the caller (AGENTS.md) — this skill produces the diagnosis and the culprit ids.

> **Confirm before first prod use:** the service names below are known; the monitor names, dashboard IDs, and whether Datadog routes through incident.io are environment-specific. Fill in `<...>` placeholders or resolve them dynamically with the search tools. Match the LangSmith skill's pattern of hard-coding stable IDs once confirmed.

## Environment / key facts
- Datadog is available via the **Datadog MCP server**. Auth is already configured; use these tools directly — do not ask the user to add an integration. Datadog MCP is scoped to the org by auth, so there is no workspace ID to pass (unlike LangSmith).
- **Services in scope:** `genie-ai-service`, `genie-ai-langgraph-integration`, `risk-ml-service`, `found-money-service`, `zendesk-cs-service`.
- **Load the Datadog skill guide first.** Before using related Datadog tools, call `list_datadog_skills` and `load_datadog_skill` to pull the relevant guide (logs / spans / metrics). This is Datadog MCP's own pattern and improves query accuracy.

### Alert → service mapping
- Datadog monitor alert naming usually carries the `service:` tag or the service name in the title. Map it to one of the five services above.
- If the service is not obvious from the alert, resolve it with `search_datadog_services` and `search_datadog_monitors` (look up the firing monitor by name/id from the alert), then confirm the `service` tag before investigating. Do not guess.

## Query syntax (do NOT reuse LangSmith FQL here)
- **Log / span search:** Datadog search syntax, e.g. `service:genie-ai-service status:error` or `service:risk-ml-service @http.status_code:5*`. Scope with `env:prod` and a time range.
- **Metric query:** metric-query syntax, e.g. `avg:trace.fastapi.request.duration{service:genie-ai-service}` or `sum:trace.*.errors{service:genie-ai-langgraph-integration}.as_count()`.
- **Log analytics:** `analyze_datadog_logs` takes **SQL** over logs (use for grouping/rates/top-N), not the search or metric syntax above.

## Primary tools for triage
- `search_datadog_monitors` — find the firing monitor from the alert; read its query, threshold, and tags.
- `search_datadog_logs` — raw log entries or log patterns. Main tool for error investigations.
- `analyze_datadog_logs` — SQL over logs for rates, grouping, top offending endpoints/hosts.
- `search_datadog_spans` — raw APM spans matching a query (find slow/errored spans).
- `get_datadog_trace` — full APM trace by `trace_id` (drill into the span tree).
- `aggregate_spans` — counts/percentiles across spans (p95/p99 latency, error counts).
- `get_datadog_metric` / `get_datadog_metric_context` / `search_datadog_metrics` — metric values, and a metric's available tags/type/unit.
- `get_change_stories` — change/deploy events for an APM service over a window. Use to correlate an incident with a recent deploy.
- `search_datadog_incidents` / `get_datadog_incident` — related Datadog incidents.
- `search_datadog_rum_events` / `aggregate_rum_events` — customer session investigation (RUM).
- Secondary: `search_datadog_hosts`, `search_datadog_service_dependencies`, `search_datadog_events`, `search_datadog_dashboards` / `get_datadog_dashboard`, `search_datadog_notebooks` / `get_datadog_notebook`.

## How to investigate
1. **Scope the timeframe from the alert timestamp.** Relevant signals occurred right before the alert fired; look back a short window (15–60 min ending at the alert time; widen only if too sparse). Always constrain to `env:prod` and the mapped `service`.
2. **Read the monitor first.** `search_datadog_monitors` → get the firing monitor's query, threshold, and the exact metric/log condition that tripped. Investigate that signal directly.
3. **Error-rate alerts:** `search_datadog_logs` with `service:<svc> status:error` (+ time). Use `analyze_datadog_logs` (SQL) to group by `@error.message` / endpoint / host and find the dominant error. Pull a representative `trace_id` from an errored log and open it with `get_datadog_trace`.
4. **Latency alerts:** `aggregate_spans` for p95/p99 by resource, or `search_datadog_spans` ordered by duration, to find the slow resource; then `get_datadog_trace` on a slow trace to find the slow child span (DB call, downstream service, model call).
5. **Correlate deploys:** `get_change_stories` for the service over the window — a spike that starts at a deploy points to a release.
6. **Cross-service:** `search_datadog_service_dependencies` when the failure looks downstream (e.g. Aurora / a called service).
7. **Customer-session questions:** `search_datadog_rum_events` scoped to the session/user.

## Known patterns (Credit Genie)
- **`psycopg.errors.ConnectionTimeout` on `genie-ai-service`** — DB connection exhaustion/timeout. Search logs for the exception; check for a coincident Aurora event and connection-pool saturation.
- **Aurora writer restart cascade** — writer restart → connection storms/timeouts across dependents. Correlate the incident window with Aurora events and `get_change_stories`.
- **LangGraph `CancelledError` / 422 race on `genie-ai-langgraph-integration`** — client cancel racing a request; look for the paired 422 and cancelled spans. This one often shows in both Datadog and LangSmith — hand off to `langsmith-triage` for the trace side.
- **`risk-ml-service` DynamoDB calls from a shadow model** — unexpected DynamoDB traffic/latency from a shadow neural model path; check span calls to DynamoDB and recent deploys.

## Code grounding
If the diagnosis needs implementation detail (error handlers, config, call paths), findings are ambiguous, or the failure looks recurring/patterned, load `github-code-grounding` and read the mapped service repo on **`prod`** instead of inferring. Do not mutate GitHub — recommend changes in the reply only.

## Output (hand back to the caller)
Follow `AGENTS.md` **Response brevity**. Hand back a short diagnosis the caller can post almost as-is — not a full investigation dump.

- **TL;DR** (1–2 sentences): why the monitor fired / the direct answer. Lead with cause.
- **Evidence:** ≤3 short bullets — dominant error/message, slow resource, or deploy correlation. Key metric values only when they prove the claim. Include `prod` paths only when code was consulted.
- **Culprit ids:** always list Datadog `trace_id`(s) and `span_id`(s) when they exist, plus links to the log query, trace, or firing monitor.
- **Next step:** one short actionable line.
- **Do not** default to full span/log tables, exhaustive host/endpoint lists, or a multi-paragraph "Root cause" that restates the TL;DR. Go deep only if the user asked for a breakdown or something is actually anomalous (see `AGENTS.md`).
