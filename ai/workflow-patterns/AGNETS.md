---
spec_version: "1.0"
agent_id: "ai_workflow_patterns"
role: "Standard workflow patterns library for common development tasks"

policies:
  goals_ref: /doc_agent/policies/goals.md
  safety_ref: /doc_agent/policies/safety.md

parent_agent: /ai/agent.md
merge_strategy: "child_overrides_parent"

context_routes:
  always_read:
    - /ai/workflow-patterns/README.md
  on_demand:
    - topic: "Pattern Details"
      paths:
        - /ai/workflow-patterns/PATTERNS_GUIDE.md
    - topic: "Pattern Catalog"
      paths:
        - /ai/workflow-patterns/catalog.yaml
---
# Workflow Patterns Agent Guide

> Houses reusable task plans for common repo work. Use it to keep orchestration deterministic.

## Scope
- Eight curated patterns covering module creation, DB changes, APIs, bug fixes, refactors, feature work, performance, and security.
- Auto-triggered by `scripts/workflow_suggest.py` and the agent trigger config.

## Context Shortcuts
| Need | Load | Purpose |
| --- | --- | --- |
| Quick summary | `/ai/workflow-patterns/README.md` | Pattern list and priorities |
| Pattern metadata | `/ai/workflow-patterns/catalog.yaml` | Machine readable catalog |
| Full guide (human) | `/ai/workflow-patterns/PATTERNS_GUIDE.md` | Deep dive when tutoring humans |
| Trigger rules | `/doc_agent/orchestration/agent-triggers.yaml` | How suggestions fire |

## Usage
1. Run `make workflow_list` or `make workflow_suggest PROMPT="..."` before planning.
2. Load the specific `patterns/<name>.yaml` when applying steps.
3. Keep generated checklists inside your workdoc to show progress.
4. When editing patterns, validate and regenerate the catalog.

## Commands
```bash
make workflow_list
make workflow_suggest PROMPT="module refactor"
make workflow_show PATTERN=<id>
make workflow_apply PATTERN=<id>
```

## Maintenance
- Preserve the schema fields (id, description, steps, pitfalls, references).
- Update README tables whenever you add or retire a pattern.
- Version bump is not needed; rely on git history.

## References
- `/ai/workflow-patterns/README.md`
- `/ai/workflow-patterns/catalog.yaml`
- `/ai/workflow-patterns/PATTERNS_GUIDE.md`
- `/doc_agent/orchestration/agent-triggers.yaml`

