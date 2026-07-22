# fleet-agent-prompts

Prompts only — no application code.

This repository versions [LangSmith Fleet](https://docs.langchain.com/langsmith/fleet) agent instructions (`AGENTS.md`) and skills. Fleet is LangChain's solution for building no-code agents: you define behavior with prompts and skills, then run them in LangSmith without writing agent code.

Each agent lives in its own directory (e.g. `on-call-agent/`) with:

- `AGENTS.md` — Fleet system instructions
- `skills/<skill-name>/SKILL.md` — skill files referenced by those instructions
