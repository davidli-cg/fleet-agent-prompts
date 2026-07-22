---
name: langsmith-triage
description: Investigate LangSmith traces/runs to diagnose agent/LLM incidents
  and answer trace-level questions. Use when the Slack sender is the LangSmith
  app (alerts titled "Genie AI Agent" or "Genie AI LangGraph Deployment",
  including "Silent Error Count has Spiked"), and for Q&A about runs,
  LLM/tool/retriever spans, prompts, tokens, or threads. When the Slack sender
  is LangSmith, use only this skill — do not load datadog-triage.
---

# LangSmith Triage

## Scope
Answer questions and diagnose incidents that can be answered by investigating LangSmith traces/runs. Refuse anything that cannot (defer to the router's out-of-scope rules). Slack formatting, routing, and reply conventions are handled by the caller (AGENTS.md) — this skill produces the diagnosis and the culprit ids.

**Routing reminder:** if the Slack alert sender is **LangSmith**, load **only** this skill. Do not also run `datadog-triage`.

## Environment / key facts
- LangSmith is available via the **LangSmith MCP server**. Auth is already configured; use these tools directly — do not ask the user to add an integration.

### Workspace and project mapping (always use these for alerts)
- **Workspace (always):** `ai-engineering-prod`
  - workspace ID: `82dc987a-21f6-420e-a81d-b085c57b8d03`
  - Always pass `workspace_id=82dc987a-21f6-420e-a81d-b085c57b8d03` on LangSmith tool calls for alert investigations.
- **Tracing project from LangSmith alert title:**
  - Title contains **"Genie AI Agent"** → project `genie-ai-agent-prod`
    - project ID: `b0993a96-ec79-4b9f-900c-8603ff16043b`
  - Title contains **"Genie AI LangGraph Deployment"** → project `genie-ai-langgraph-deployment-prod`
    - project ID: `8629c985-881d-44ec-874c-4327778d6216`
    - Do **not** use the bare name `genie-ai-langgraph-deployment` (it will fail with "no project found").
- Prefer project **name** or **ID** as `project_name` for `fetch_runs` / related tools once mapped from the title.
- **Q&A without a LangSmith alert message:** if the user does not specify workspace and tracing project, **ask them to specify both before continuing**. Do not guess.

## Alert type: Silent Error Count (critical — do not infer)
Alerts titled **"Silent Error Count has Spiked for Genie AI Agent"** or **"Silent Error Count has Spiked for Genie AI LangGraph Deployment"** have a **fixed, product-specific meaning**. Do **not** reinterpret "silent error" as "swallowed child span", "root succeeded while child failed", or any other LangSmith status pattern.

### What a silent error is
- A **tool** run whose **output** contains this exact user-facing message (also returned as `error_message`):
  - `"I encountered an issue. Please try again later or contact support for assistance."`
- Tools catch failures and return that string instead of raising, so the tool run is usually **`status: success`** / **not** `error=true` in LangSmith. The failure is "silent" to LangSmith's error flag — it only shows up in the tool **output**.
- Typical output shape: `{"error_message": "I encountered an issue. Please try again later or contact support for assistance."}`

### What to query (required for these alerts)
1. Map project from the alert title (Agent → `genie-ai-agent-prod`, LangGraph Deployment → `genie-ai-langgraph-deployment-prod`).
2. Scope the timeframe from the LangSmith alert message timestamp (15–60 min lookback ending at message time).
3. Call `fetch_runs` with:
   - `run_type="tool"`
   - time window FQL **plus** a search for the fixed message, e.g.:
     `and(gte(start_time, "<window_start_iso>"), lte(start_time, "<langsmith_message_time_iso>"), search("I encountered an issue"))`
   - Raise `preview_chars` so `outputs` / `outputs_preview` show the full `error_message`.
4. **Do not** use `error="true"` as the primary filter for Silent Error Count alerts — that finds hard failures (LLM provider 500s, exceptions, etc.) and will surface the **wrong** traces.
5. For each matching tool run, open the parent `trace_id` to see which tool failed, inputs, upstream HTTP/integration failures, and the user `thread_id`. Group by tool name when multiple hits share a pattern.

### Anti-patterns (caused wrong diagnoses before)
- Treating "silent" as "child errored, root succeeded" and querying `error="true"` without searching for the fixed message.
- Reporting an LLM provider HTTP 500 (or any hard LangSmith error) as the Silent Error Count culprit unless that same trace's **tool** output also contains the fixed message above.

## Primary tools for triage
- `list_workspaces` — available if needed; for alerts skip discovery and use `ai-engineering-prod` / the workspace ID above.
- `list_projects` — resolve project only when the alert-title mapping does not apply or the user names an unfamiliar project.
- `fetch_runs` — main investigation tool. Required: `project_name`, `limit` (also pass `page_number`, default 1). Useful args:
  - `workspace_id`: always `82dc987a-21f6-420e-a81d-b085c57b8d03` for prod alert work
  - `error`: `"true"` for errored runs, `"false"` for successes — **not** the right primary filter for Silent Error Count (see above)
  - `is_root`: `"true"` for top-level traces (start here for error-rate / latency alerts; Silent Error Count starts at `run_type="tool"`)
  - `run_type`: `llm` | `chain` | `tool` | `retriever`
  - `filter`: FQL on the run itself (time window, latency, status, name, tags, `search("...")`, etc.)
  - `trace_filter` / `tree_filter`: FQL on the root / any span in the tree
  - `trace_id`: load the full trace (child spans); paginate with `page_number` when needed
  - `order_by`: default `"-start_time"`; use `"-latency"` for slowest-first on latency alerts
  - `preview_chars`: raise if truncated inputs/outputs/errors hide the signal
- Secondary (use when relevant, not every alert):
  - `list_issues` / `get_issue` — LangSmith Engine issues for a project (status/severity filters).
  - `get_thread_history` — conversation/thread message history when the question is about a specific `thread_id` in a project.

## How to investigate with `fetch_runs`
1. **Scope the timeframe from the LangSmith alert message timestamp.** Relevant traces occurred **right before** the alert was posted. Build an FQL window ending at (or just before) the LangSmith message time, looking back a short window (15–60 min; widen only if too few runs):
   `and(gte(start_time, "<window_start_iso>"), lte(start_time, "<langsmith_message_time_iso>"))`
2. **Silent Error Count alerts:** follow the dedicated section above (`run_type="tool"` + `search("I encountered an issue")`). Do **not** use the error-rate path below.
3. **Error-rate alerts:** `is_root="true"`, `error="true"`, plus the time `filter`. Inspect `error`, inputs/outputs, and failing child names.
4. **Latency alerts:** `is_root="true"`, time `filter`, and e.g. `gt(latency, "5s")` (or the alert threshold); `order_by="-latency"`. Then dig into slow spans with `trace_id`.
5. **Full trace:** take `trace_id` from a representative run and call `fetch_runs` again with that `trace_id` (same `project_name` + `workspace_id`) to inspect model/tool/retriever spans, errors, and latency.
6. Identify the pattern (a specific tool returning the silent error message, a tool timing out, a model provider 429s, a bad input shape, a prompt change, a LangGraph `CancelledError` / 422 race, etc.).

## Investigation workflow (alert)
1. From the LangSmith alert, extract: metric (**Silent Error Count** / error rate / latency), threshold, message send time, and project from the title mapping. Alert text may vary (e.g. `Firing - HTTP - Latency has Spiked for Genie AI Agent`); map to the matching investigation path.
2. Query LangSmith in `ai-engineering-prod` for the mapped project, time window ending at the LangSmith message time.
3. Apply the filter for that alert type:
   - **Silent Error Count** → `run_type="tool"` + time FQL + `search("I encountered an issue")` (see dedicated section).
   - **Error rate / latency** → `is_root="true"`, time FQL `filter`, plus `error="true"` or latency FQL.
4. Inspect representative traces: for silent errors, the tool `outputs` / `error_message` and upstream cause; for hard errors, error messages/stack traces; for latency, slow spans and model/tool calls (re-fetch with `trace_id` for full trees).
5. Identify the pattern.

## Code grounding
If the diagnosis needs implementation detail (exact error string origin, tool/node control flow, config), findings are ambiguous, or the failure looks recurring/patterned, load `github-code-grounding` and read the mapped repo on **`prod`** instead of inferring. Do not mutate GitHub — recommend changes in the reply only.

## Output (hand back to the caller)
Follow `AGENTS.md` **Response brevity**. Hand back a short diagnosis the caller can post almost as-is — not a full investigation dump.

- **TL;DR** (1–2 sentences): why the alert fired / the direct answer. Lead with cause.
- **Evidence:** ≤3 short bullets with decisive facts only. Silent Error Count → tool name + the fixed message (and upstream cause if known). Latency → round-count / dominant slow pattern, not a full span table. Errors → dominant error string / failing span. Include `prod` paths only when code was consulted.
- **Culprit ids:** always list `trace_id`(s) and `thread_id`(s) when they exist, plus LangSmith UI links when they can be constructed.
- **Next step:** one short actionable line.
- **Do not** default to latency/tool tables, exhaustive tool inventories, or a multi-paragraph "Root cause" that restates the TL;DR. Go deep only if the user asked for a breakdown or something is actually anomalous (see `AGENTS.md`).
