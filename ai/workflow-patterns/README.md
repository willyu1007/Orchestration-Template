---
audience: ai
language: en
version: summary
purpose: Lightweight catalog for workflow patterns
---
# AI Workflow Pattern Catalog

> **For AI Agents** - Quick access to curated workflows. Load `PATTERNS_GUIDE.md` only if you need the detailed rationale.

## Quick Commands
```bash
make workflow_list                          # list all patterns
make workflow_suggest PROMPT="build auth"   # recommend by free text
make workflow_show PATTERN=module-creation  # show one pattern
make workflow_apply PATTERN=module-creation # emit checklist
```

## Available Patterns
| ID | When To Use | Complexity | Duration | Category | Priority |
|----|-------------|------------|----------|----------|----------|
| module-creation | Create a brand-new module with docs/tests | medium | 2-4h | development | P0 |
| database-migration | Any schema/data change | medium | 30-60m | maintenance | P0 |
| api-development | Ship a new API endpoint | low | 1-2h | development | P0 |
| bug-fix | Fix a production bug | low | 30-90m | debugging | P0 |
| refactoring | Internal code cleanup | medium | 1-3h | maintenance | P1 |
| feature-development | Multi-task feature delivery | medium | 4-8h | development | P1 |
| performance-optimization | Improve latency or throughput | medium | 2-4h | optimization | P1 |
| security-audit | Review auth, secrets, dependencies | medium | 2-3h | maintenance | P1 |

## Choosing A Pattern
```
Need a new module?          -> module-creation
Shipping an API endpoint?   -> api-development
Touching the database?      -> database-migration
Fixing regressions?         -> bug-fix
Cleaning up legacy code?    -> refactoring
Improving perf budgets?     -> performance-optimization
Full feature delivery?      -> feature-development
Security or compliance?     -> security-audit
```

## Pattern Anatomy
Every pattern YAML includes:
- **Pre-checks** - prerequisites before work starts.
- **Workflow steps** - 6-9 numbered actions with owner/context.
- **Common pitfalls** - warnings and mitigations.
- **Quality gates** - tests, docs, approvals to close the task.
- **Estimates** - coarse duration per step.
- **References** - docs or scripts to load.

## Automation Hooks
- Keywords such as ¡°create¡±, ¡°scaffold¡±, ¡°migrate¡±, ¡°optimize¡±, ¡°audit¡±, or ¡°fix¡± trigger suggestions.
- Guardrail triggers in `doc_agent/orchestration/agent-triggers.yaml` can auto-load patterns when risky areas are detected.

## Maintaining Patterns
1. Add a new YAML file under `ai/workflow-patterns/patterns/` following the existing schema.
2. Run `make workflow_validate` (runs YAML schema + lint).
3. Regenerate `catalog.yaml` via `make workflow_catalog` if you add metadata fields.
4. Update `PATTERNS_GUIDE.md` with rationale for human readers.

Keep this file concise (<150 lines). Anything conceptual or historical belongs in `PATTERNS_GUIDE.md`.

