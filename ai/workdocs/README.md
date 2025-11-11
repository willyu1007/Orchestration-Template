# AI Workdocs

> Lightweight task documentation for AI agents. Use this folder to preserve context between sessions.

## Purpose
`ai/workdocs/` stores structured task packages. Each active task lives in `active/<task-name>/` and contains:
- `plan.md` - strategy and phases.
- `context.md` - progress ledger + quick resume (the most important file).
- `tasks.md` - actionable checklist with acceptance criteria.

Completed tasks move to `archive/`.

## Workflow
1. `make workdoc_create TASK=my-task` to scaffold plan/context/tasks.
2. Update `context.md` every time you finish a milestone, discover a blocker, or change scope.
3. Keep `tasks.md` synchronized with reality (status + owner + exit criteria).
4. Archive with `make workdoc_archive TASK=my-task` once the definition of done is met.

## Best Practices
- Always list blockers and mitigations in `context.md` so future agents can resume quickly.
- Reference related modules, docs, and scripts inline; copy links, not entire content.
- Mention the repository language (from `config/language.yaml`) in every plan so humans know which language to use in reports and comments.
- Keep `plan.md` lean¡ªdescribe phases, owners, and success metrics only.

## Templates
Templates live in `doc_human/templates/`:
- `workdoc-plan.md`
- `workdoc-context.md`
- `workdoc-tasks.md`

Copy them directly or let the helper scripts do it for you.

## Related Docs
- `doc_human/guides/WORKDOCS_GUIDE.md` - full explanation for humans.
- `scripts/workdoc_create.sh` - scaffolding automation.
- `scripts/workdoc_archive.sh` - archiving helper.

