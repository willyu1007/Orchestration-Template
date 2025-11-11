---
spec_version: "1.0"
agent_id: "ai_workdocs"
role: "Task context management and recovery for AI development sessions"

policies:
  goals_ref: /doc_agent/policies/goals.md
  safety_ref: /doc_agent/policies/safety.md

parent_agent: /ai/agent.md
merge_strategy: "child_overrides_parent"

context_routes:
  always_read:
    - /ai/workdocs/README.md
  on_demand:
    - topic: "Workdocs Usage"
      paths:
        - /doc_agent/quickstart/workdocs-quickstart.md
---
# Workdocs Agent Guide

> Workdocs cut context reload time to minutes. Use them for any task that survives more than one session.

## Scope
- Each active task lives under `ai/workdocs/active/<task>/`.
- Three files only: `plan.md`, `context.md`, `tasks.md`.
- Archive completed folders under `archive/`.

## Context Shortcuts
| Need | Load | Purpose |
| --- | --- | --- |
| How-to | `/ai/workdocs/README.md` | Directory policy and examples |
| Quickstart (AI) | `/doc_agent/quickstart/workdocs-quickstart.md` | Command focused checklist |
| Human guide | `/doc_human/guides/WORKDOCS_GUIDE.md` | Full methodology |

## File Expectations
- `plan.md` - goal, scope, risks, success signals.
- `context.md` - session progress, key files, decisions, blockers, quick resume paragraph.
- `tasks.md` - checklist with owners and acceptance criteria.

## Workflow
1. Run `make workdoc_create TASK=<name>` before the second session starts.
2. Update `context.md` whenever you wrap a block of work.
3. Link the workdoc from LEDGER entries and PR descriptions.
4. Once delivered, run `make workdoc_archive TASK=<name>`.

## Commands
```bash
make workdoc_create TASK=<name>
make workdoc_list
make workdoc_archive TASK=<name>
```

## Safety
- Do not store secrets or credentials.
- Keep context concise (<=150 lines) to stay routing friendly.
- Move stale folders out of `active/` weekly.

## References
- `/ai/workdocs/README.md`
- `/doc_agent/quickstart/workdocs-quickstart.md`
- `/doc_human/guides/WORKDOCS_GUIDE.md`

Workdocs inherit repo versioning; this agent stays versionless.

