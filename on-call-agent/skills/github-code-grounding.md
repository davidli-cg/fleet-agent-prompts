---
name: github-code-grounding
description: Ground RCA and Q&A in production GitHub code (read-only). Use when
  understanding an incident or answer requires implementation detail — error
  strings, control flow, config, handlers — or when observability findings are
  ambiguous, contradictory, or show a recurring pattern that code can explain.
  Always read the prod branch. Never write to GitHub (no PRs, issues, comments,
  merges, or file edits).
---

# GitHub Code Grounding

## Scope
Use GitHub **only to read** production code and related history so diagnoses and answers are grounded in what actually ships — not inferred from stack traces, error messages, or memory. Recommend code changes in the Slack reply when useful; **do not** implement them in GitHub.

Slack formatting, routing, and reply conventions are handled by the caller (`AGENTS.md`). Observability investigation stays in `langsmith-triage` / `datadog-triage`; this skill is for code context on top of those findings.

## When to load (and when not to)
**Load and use this skill when any of these are true:**
- Full understanding of the failure needs implementation detail (error strings, branches, retries, timeouts, feature flags, config defaults, handler wiring).
- Observability findings do not make sense, conflict, or leave an open "why does this path exist?" question that code can settle.
- The same error/issue looks **recurring** or patterned across traces/logs and code may reveal a systemic cause (bad catch, misconfig, known race, hard-coded message).

**Do not** open GitHub for every alert by default. If logs/traces already name a clear external cause (provider 500, deploy restart, DB timeout with no ambiguous app path), finish without code unless a recommendation needs a concrete file reference.

## Environment / key facts
- GitHub is available via the **built-in Fleet GitHub connection**. Auth is already configured; use the tools directly — do not ask the user to add an integration.
- **Production branch (always):** `prod`. Prefer `prod` for file reads, code search, commits, and blame-style context. Do **not** treat `main` / `master` as production unless the user explicitly asks for a non-prod branch.
- **Org:** `CreditGenie`.

### Service / alert → repo mapping
| Signal | Repository |
| --- | --- |
| `genie-ai-service` | `CreditGenie/genie-ai-service` |
| `genie-ai-langgraph-integration` | `CreditGenie/genie-ai-langgraph-integration` |
| LangSmith **"Genie AI Agent"** / project `genie-ai-agent-prod` | `CreditGenie/genie-ai-agent` |
| LangSmith **"Genie AI LangGraph Deployment"** / project `genie-ai-langgraph-deployment-prod` | `CreditGenie/genie-ai-langgraph-deployment` |
| `risk-ml-service` | `CreditGenie/risk-ml-service` |
| `found-money-service` | `CreditGenie/found-money-service` |

If the failing component is unclear, map from the Datadog `service:` tag or LangSmith project/title first, then open the matching repo on `prod`. If still ambiguous, ask which service/repo — do not guess across unrelated repos.

## Read-only policy (strict)
**Allowed (read / browse only):**
- Repos & tree: `Get Repo`, `List Directory`, `Get File`, `List Branches`
- Code & history: `Search Code`, `List Commits`, `Get Commit`
- PRs (read): `List Pull Requests`, `Search Pull Requests`, `Get Pull Request`
- Issues (read): `List Issues`, `Search Issues`, `Get Issue`
- Projects (read): `List Projects`, `Get Project`, `List Project Items`

**Forbidden — never call these for this agent:**
- Any write to files or branches: `Update File`, `Delete File`, `Create Branch`
- Any PR mutation: `Create Pull Request`, `Merge Pull Request`, `Close Pull Request`, `Request Reviewers`, `Create Review`, `Comment Pull Request`
- Any issue mutation: `Create Issue`, `Update Issue`, `Close Issue`, `Add Labels`, `Remove Labels`, `Comment Issue`
- Any project mutation: `Create Project`, `Update Project`, `Delete Project`, `Add Project Item`, `Add Project Draft Issue`, `Update Project Item Field`, `Clear Project Item Field`, `Delete Project Item`, `Link Project Repository`

If a write tool appears in the tool list, ignore it. Recommend patches in Slack with file paths and snippets; leave implementation to humans.

## How to ground in code
1. **Pick the repo** from the mapping above using the service / LangSmith project already under investigation.
2. **Stay on `prod`.** When searching or fetching files, target `prod` (branch / ref). If a tool defaults to another branch, override to `prod`.
3. **Search narrowly.** Prefer `Search Code` with distinctive strings from the incident (exact `error_message`, exception type, endpoint, tool/node name, metric tag) over broad keywords.
4. **Read the relevant files.** Use `List Directory` / `Get File` to open handlers, config, and call sites. Quote short, specific snippets in the diagnosis — enough to prove the claim.
5. **Optional history.** Use `List Commits` / `Get Commit` (and read-only PR search) on `prod` when a recent change likely explains a new spike — correlate with Datadog `get_change_stories` / deploy timing when available.
6. **Recommend, don't ship.** If code should change, describe the fix (file path on `prod`, what to change, why) in the Slack reply. Do not open PRs or issues.

## Anti-patterns
- Inferring control flow, default config, or "what the catch block does" when `Get File` / `Search Code` on `prod` would answer it.
- Reading `main` / a feature branch and calling it production.
- Opening every repo for one incident — stay on the mapped service repo unless evidence points elsewhere.
- Creating or mutating PRs, issues, labels, comments, projects, or files.

## Output (hand back to the caller)
- What was checked in GitHub: repo, **`prod`**, file paths (and commit SHA if used).
- How the code explains (or fails to explain) the observability findings — short quoted evidence.
- Optional **recommended code change** (path + intent only); never a claim that a PR was opened.
