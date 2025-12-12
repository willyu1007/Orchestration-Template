# AI-First Repository Template – Overview

This repository template is designed for **AI-first development**, where an AI developer (a code-capable model) is the primary implementer, and humans plus a thin runtime provide guardrails, approvals, and integrations.

This README gives both humans and AIs a **single mental model** for the template, tying together:

- Modular layout
- Strategy docs (`AGENTS.md`)
- Knowledge routing (`ROUTING.md` + `routes/`)
- Ability routing (`ABILITY.md` + registries)
- Workdocs for long-running tasks
- Hooks and runtime
- DevOps extensions (init, database, packaging, deployment)

The goal is to maximize **AI development efficiency** while keeping behavior **observable, controllable, and evolvable**.

---

## 1. Roles and Terminology

To avoid ambiguity, this template uses the following terms consistently:

### 1.1 AI developer and internal orchestration

- **AI developer** – a code-capable AI model that reads this repo, plans work, edits files, and calls tools.
- **Internal orchestration** – the AI’s own planning loop:
  - Understand → Plan → Act → Review
  - Decide which modules, docs, and abilities to use
  - Maintain task state in workdocs

Whenever these docs talk about “orchestration” or “task orchestration”, they refer to this **internal AI process**, not an external orchestrator service.

### 1.2 Runtime and hooks

- **Runtime** – a long-running process (or set of processes) that:
  - Receives user input or system events
  - Runs hooks on key events
  - Executes abilities via a standard execution script
  - Persists logs and task metadata
- **Hooks** – small, event-driven pieces of infrastructure that run around the AI’s work (before/after prompts or ability calls, at session end). They provide:
  - Routing hints
  - Guardrails and checks
  - Telemetry and summaries

The runtime **does not plan development tasks**. It wraps the AI developer with guardrails and automation.

### 1.3 Humans

Humans:

- Provide requirements, domain knowledge, and constraints
- Approve sensitive operations (especially affecting production systems)
- Run scripts that need credentials or local access
- Review AI-generated changes and update the template over time

The repository is structured so that a human who knows the business but not the codebase can still collaborate effectively with the AI.

---

## 2. Modular Architecture – The Core Building Block

Everything in this template is organized around **module instances**. A module is the smallest unit that:

- Owns a piece of business responsibility
- Owns its code and configuration
- Owns its AI work history and decisions
- Is addressable by routing, guardrails, and CI

### 2.1 Module instance layout

Each module lives under `modules/<module_id>/` and follows a standard skeleton:

```text
modules/
  <module_id>/
    MANIFEST.yaml      # Module metadata and dependencies
    AGENTS.md          # Module-level AI strategy & rules
    ROUTING.md         # Knowledge routing entrypoint
    ABILITY.md         # Ability/tool routing entrypoint

    workdocs/          # AI work context and history for this module
      AGENTS.md
      active/
      outcomes/
      archive/

    routes/            # Knowledge routing topic files
    interact/          # External interactions (DB, APIs, events, test data)
    config/            # Module configuration (LLM, parameters, telemetry, flags)
    src/               # Code (backend, frontend, etc.)
    docs/              # Human-facing docs
    outcomes/          # Optional output artifacts
```

**Key idea:** when a task is scoped to one module, the AI developer should be able to stay inside this directory and find everything it needs.

### 2.2 Module metadata and types

- `MANIFEST.yaml` defines:
  - `module_id`, name, description
  - `module_type` – a type from the project’s type graph
  - Optional `module_level` and `parent_module` for hierarchy
  - Dependencies on other modules and external services
- At project level, `modules/overview/type_registry.yaml` defines the **type graph**:
  - Nodes: module types (business stages or technical layers)
  - Edges: allowed data/control flows
- `modules/overview/instance_registry.yaml` lists all module instances and their types.

The type graph is used to:

- Constrain allowed cross-module calls
- Validate integration scenarios
- Help the AI choose legal paths when building end-to-end flows

---

## 3. Strategy Docs (`AGENTS.md`)

Strategy docs tell the AI **how to behave** in different parts of the repo, without repeating detailed specs.

There are several layers:

1. **Project root `AGENTS.md` (optional but recommended)**  
   - Explains project purpose and global constraints  
   - Routes to major areas (modules, infrastructure, documentation)

2. **Modular system root `modules/AGENTS.md`**  
   - Explains the module architecture and type graph  
   - Describes how module instances relate and how cross-module work should be done

3. **Module-level `modules/<module_id>/AGENTS.md`**  
   - Defines the responsibility and boundaries of that module  
   - States which files/dirs are in scope for changes  
   - Adds local coding/testing conventions and safety rules

4. **Directory-level `AGENTS.md`** (optional)  
   - For specialized areas (`workdocs/`, `src/backend/`, `/db`, `/ops`, etc.)

**Priority rule:** when multiple `AGENTS.md` apply, the AI must:

1. Respect global hard constraints from higher levels
2. Apply the most local strategy for detailed decisions
3. Prefer the more restrictive interpretation when rules seem to conflict

This network of strategy docs keeps behavior aligned and predictable across the repo.

---

## 4. Knowledge Routing (`ROUTING.md` + `routes/`)

Knowledge routing helps the AI load **the right documents at the right time** instead of scanning the entire repo.

### 4.1 Concepts

- **Routing documents**
  - `ROUTING.md` – entrypoint
  - Topic route files under `routes/`
- **Knowledge documents**
  - Actual content (Markdown, etc.) with front matter metadata

Routing docs are optimized for **navigation and selection**; knowledge docs are optimized for **explanation and examples**.

### 4.2 Orchestration stages

The AI’s internal orchestration is divided into four conceptual stages:

- `understand` – grasp the problem and context
- `plan` – design steps and architecture
- `act` – implement, run tools, modify files
- `review` – verify, evaluate, and summarize

Routing metadata can tag documents with stage hints so the AI can prioritize different docs at different stages.

### 4.3 `ROUTING.md` and topic routes

Each routing root (project or module) provides a `ROUTING.md` that lists **scopes and topics** and points to topic route files, for example:

```yaml
route_indexes:
  - scope: frontend
    topic: bug_triage
    summary: Bug triage routes for the frontend module.
    route_file: /routes/frontend.bug_triage.yaml
```

Each topic route file (`/routes/<scope>.<topic>.yaml`) defines:

- `when_to_use` – when this route applies
- `related_docs` – knowledge docs plus:
  - `stage` hints (optional)
  - `doc_usage` comments (how to use the doc)

### 4.4 Retrieval flow for the AI

For any task, the AI should:

1. Infer current stage (`understand`, `plan`, `act`, `review`)
2. Open the relevant `ROUTING.md` (module-level, or project-level for cross-module work)
3. Select one or more topics based on the request
4. Load topic route files and match `when_to_use`
5. Pick a **small subset** (1–5) of `related_docs` appropriate for this stage
6. Only then load those docs into context

This keeps context small and targeted, while making document discovery systematic.

---

## 5. Ability Routing (`ABILITY.md` + Registries)

Ability routing defines how the AI discovers and uses **reusable capabilities** (scripts, APIs, MCP tools, workflows, agents) instead of rewriting logic from scratch.

### 5.1 Ability pool and execution script

The project maintains a **global ability pool** with a **single standard execution script**. The AI:

- Does **not** call raw scripts/APIs directly
- Always goes through the execution script to:
  - Create tasks
  - Run tasks
  - Query results

The execution script:

- Applies guardrails and configuration
- Chooses implementations (script / MCP / API)
- Records usage in `.system/implement/<task-key>/tool_call.md`

### 5.2 High-level vs low-level abilities

All abilities live in one pool, but are presented differently:

- **High-level abilities**
  - Type: `workflow` or `agent`
  - Represent meaningful chunks of work (`signup_e2e_test`, `code_review_agent`)
  - Preferable when they can cover a whole subtask

- **Low-level abilities**
  - Atomic operations (`db.write.user_row`, `http.call.billing_service`)
  - Expose scripts, MCP tools, or APIs
  - Used when no suitable high-level ability exists

`ABILITY.md` for a module or integration scope lists:

- Recommended **high-level** abilities first
- Selected **low-level** abilities relevant to that scope
- Registry paths (YAML) that define contracts and configuration

### 5.3 Registries and configuration

Registries provide the **source of truth** for ability semantics:

- **Low-level registry**
  - Operation keys (`db.write.user_row`)
  - Input/output schemas
  - Scope and environment restrictions
  - Links to implementations

- **High-level registry**
  - Ability IDs and types
  - Intents, scenarios, and callable low-level operations
  - Execution behavior and constraints

Configuration files (per environment) define:

- Which implementations are active
- Parameters (endpoints, servers, prompts, timeouts)
- Guardrails (rate limits, environment rules)

The AI’s job is to **respect registry contracts** and call the execution script; it does not need to know the runtime details.

### 5.4 Typical ability usage flow

For a planned step like “run signup E2E tests” the AI should:

1. Open the relevant `ABILITY.md`
2. Look for a matching high-level ability (e.g. `signup_e2e_test`)
3. Check the registry for required input/output
4. Ask the execution script to create a task
5. Run the task when appropriate
6. If no high-level ability fits, assemble a chain of low-level operations instead

This keeps orchestration logic at the semantic level while keeping runtime behavior consistent and auditable.

---

## 6. Workdocs – Continuous AI Work Context

Workdocs allow the AI to **persist plans, context, and progress** for non-trivial tasks so future sessions can resume efficiently.

### 6.1 When to use workdocs

Use `workdocs/active/<task_id>/` for tasks that are:

- Multi-step or long-running
- High-risk or cross multiple files/modules
- Likely to require human review or cross-session continuity

You can skip workdocs for trivial changes (one-line fixes, tiny refactors).

### 6.2 Task folders

Each active task has a folder like:

```text
workdocs/active/T-YYYYMMDD-slug/
  plan.md
  context.md
  tasks.md
```

Recommended usage:

- `plan.md` – the execution plan and acceptance criteria
- `context.md` – current state, findings, and how to resume
- `tasks.md` – checklist with objectively verifiable items

When a task is done and stable, archive it under `workdocs/archive/` and optionally add a summary under `workdocs/archive/summary/`.

### 6.3 Cross-task outcomes

`workdocs/outcomes/` collects module-wide knowledge:

- `decisions.md` – important decisions and discoveries
- `lessons.md` – mistakes, root causes, and guardrail ideas
- `scripts/` – helper scripts created during tasks that may become reusable abilities

Over time, stable content from `decisions.md` / `lessons.md` / `scripts/` can be promoted into formal **knowledge docs** and **abilities**.

---

## 7. Hooks and Event-Driven Infra

Hooks are **repository-local automation** that run on specific runtime events and produce structured signals for the AI or infra.

### 7.1 Events

Hooks attach to four main events emitted by the runtime:

- `PromptSubmit` – new user task/message arrives  
- `PreAbilityCall` – right before an ability is executed  
- `PostAbilityCall` – right after an ability finishes  
- `SessionStop` – session or logical phase ends

### 7.2 Blocking vs non-blocking

- **Blocking events** (AI-facing)
  - `PromptSubmit`, `PreAbilityCall`
  - Hooks can:
    - Suggest routing choices (abilities, docs)
    - Normalize intent
    - Enforce guardrails (allow/deny ability calls)
  - Their signals are provided to the AI as part of the turn context

- **Non-blocking events** (infra-only)
  - `PostAbilityCall`, `SessionStop`
  - Hooks can:
    - Log usage and latency
    - Update caches and reports
    - Run builds/tests and write summaries
  - They **do not** affect the current turn’s logic

### 7.3 Hook declarations and handlers

Hooks are declared as YAML files under `.system/hooks/`, for example:

```yaml
id: prod_config_guard
event_type: PreAbilityCall
enabled: true
blocking: true

match:
  ability_scope: "edit_file"

handler:
  kind: script
  command: ./scripts/hooks/prod_config_guard.py

effects:
  - enforce_prod_safety
```

Handlers receive a JSON context and return a structured result. For AI-facing events they may emit **hook signals**, such as:

- `routing_hint` – suggested abilities/documents
- `normalized_intent` – parsed task description
- `ability_guard` – allow/deny ability calls

The runtime merges these signals and exposes them to the AI as structured context. The AI still owns the plan; hooks only provide hints and guardrails.

### 7.4 Relationship to routing

Hooks and routing are intentionally aligned:

- Knowledge routing can use trigger-like hooks to surface relevant topics/docs
- Ability routing uses `matcher`/`guardrail` config in registries to drive hooks
- Workdocs can be updated by hooks after ability calls or at session end

Together they create a light-weight, event-driven layer around the AI’s internal orchestration.

---

## 8. DevOps Extensions (Init, DB, Packaging, Deployment)

Beyond core feature work, the template supports **extended DevOps scenarios** in a way that is still AI-first.

Each scenario has its own **workspace** with `workdocs/` and a small set of scripts that act as the **only bridge** to external systems.

### 8.1 Project initialization – `/PROJECT_INIT/`

Used to turn the template into a concrete project:

```text
/PROJECT_INIT/
  doc/       # Human instructions
  AI/        # AI-only guidance
  workdocs/  # Scenario-local workspace
```

The AI:

- Gathers product/domain information from humans
- Writes proposals and summaries into `/PROJECT_INIT/workdocs/`
- Integrates finalized information into existing template docs and config
- Leaves module registration to dedicated tools

### 8.2 Database mirror – `/db/`

Makes database structure explicit and AI-friendly without giving direct access to live DBs:

```text
/db/
  schema/     # Table/relationship descriptions
  migrations/ # Versioned migrations
  config/     # Environment configs and allowed operations
  samples/    # Optional sample data
  workdocs/   # Design & change planning
```

Important rules:

- Real databases remain the **single source of truth**
- All changes go through scripts; AI edits schema/migrations/workdocs, humans run scripts
- Environments (dev, staging, prod) have clear constraints

### 8.3 Packaging – `/ops/packaging/`

Standardizes how code becomes runnable artifacts:

```text
/ops/packaging/
  services/   # Long-running services (containers by default)
  jobs/       # Scheduled/batch jobs
  apps/       # Client applications
  scripts/    # Common build scripts
  workdocs/   # Packaging strategies & history
```

The AI:

- Uses these definitions to build or adjust packaging
- Records decisions in packaging workdocs
- Avoids inventing ad-hoc packaging schemes

### 8.4 Deployment – `/ops/deploy/`

Describes how packaged artifacts run in environments:

```text
/ops/deploy/
  http_services/
  workloads/     # Serverless, jobs, event-driven workloads
  clients/
  scripts/       # Deploy/rollback scripts
  workdocs/      # Deployment plans & history
```

The AI:

- Plans deployments using these descriptors
- Helps humans execute scripts and debug issues
- Records attempts and outcomes in deployment workdocs

Humans retain control over sensitive credentials and production deployments.

---

## 9. Cross-Module Integration

Cross-module work (integration tests, end-to-end flows) is modeled under `modules/` rather than any single module.

Key project-level structures:

- `modules/overview/`
  - `type_registry.yaml` – module types and type graph
  - `instance_registry.yaml` – all module instances
  - `interfaces.yaml` – public interfaces and quality status
  - `api_gateway.yaml` – exposure and traffic policy
  - `instance_maturity.yaml` – readiness levels
- `modules/integration/`
  - `AGENTS.md` – integration strategies and guardrails
  - `ROUTING.md` / `ABILITY.md` – integration-specific knowledge/tools
  - `scenarios.yaml` – integration and E2E scenarios
  - `workdocs/` – per-scenario plans/context/tasks
  - `outcomes/reports/` – test and performance reports

The AI should:

1. Use `modules/AGENTS.md` and `modules/overview/AGENTS.md` to understand cross-module rules
2. Work with `modules/integration/scenarios.yaml` for cross-module testing
3. Maintain scenario workdocs similarly to module workdocs

All interactions implied by scenarios should respect the **type graph** and **interface registries**.

---

## 10. Typical AI Workflows

### 10.1 Single-module feature or bugfix

1. Determine the responsible module
2. Go to `modules/<module_id>/`
3. Read `AGENTS.md` for local rules
4. If non-trivial, create/open a task under `workdocs/active/`
5. Use `ROUTING.md` → `routes/` to load relevant knowledge docs
6. Use `ABILITY.md` to select tools/abilities (prefer high-level)
7. Plan and execute, updating `plan.md`, `context.md`, `tasks.md`
8. Let hooks and the runtime help with guardrails and telemetry
9. Update `decisions.md` / `lessons.md` and scripts as needed
10. Archive workdocs when done

### 10.2 Cross-module integration or E2E scenario

1. Start from `modules/AGENTS.md` to confirm cross-module rules
2. Check or create a scenario in `modules/integration/scenarios.yaml`
3. Create/open the matching scenario folder in `modules/integration/workdocs/`
4. Use integration `ROUTING.md` and `ABILITY.md` for docs/tools
5. Run tests and analyze results, updating scenario workdocs
6. Open module-level tasks for fixes where needed
7. Produce or update reports in `modules/integration/outcomes/reports/`

### 10.3 DevOps scenario (init, DB, packaging, deploy)

1. Enter the corresponding directory (`/PROJECT_INIT`, `/db`, `/ops/packaging`, `/ops/deploy`)
2. Read local `AGENTS.md` (if present) and docs
3. Work within that scenario’s `workdocs/`
4. Use the provided scripts as the **only bridge** to external systems
5. Record decisions and outcomes so future AI sessions can reuse them

---

## 11. How to Evolve This Template

This template is meant to grow with real usage:

- New **scripts** can be promoted into low-level abilities
- Frequently used flows can become **high-level abilities**
- Stable task knowledge can become **knowledge docs** registered in routing
- Repeated mistakes can become **guardrails** (via hooks, tests, or CI)
- New module instances and integration scenarios can be scaffolded using consistent tools

When evolving the template:

- Keep responsibilities clear (modules, routing, abilities, hooks, DevOps)
- Prefer small, composable changes over big restructures
- Always update the relevant `AGENTS.md`, routing, and registries so the AI can discover new capabilities without guesswork
