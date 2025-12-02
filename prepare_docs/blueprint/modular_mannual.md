# Modular Development Specification (AI-First Repository Template)

## 0. Goals and Design Principles

This repository template is designed for **AI-first development**: large language models (LLMs) and task orchestrators are expected to do most of the day-to-day implementation work, with humans reviewing, steering, and making critical decisions.

The goals are:

- **Increase AI effectiveness** by constraining context and defining clear boundaries.
- **Make AI work continuous and recoverable** across sessions and agents.
- **Standardize module-level metadata** so infrastructure (routing, guardrails, CI, orchestration) can reason about the system.
- **Avoid unnecessary overhead** for trivial work; most structure is only required for tasks that cross sessions or modules.

Everything is organized around **modules**. A module is the smallest unit that:

- Owns business responsibility.
- Owns its code and configuration.
- Owns its AI work history and decisions.
- Is addressable by infrastructure (knowledge routing, ability routing, orchestrator).

---

## 1. Core Concepts

### 1.1 Module Instance

A **module instance** (or simply “module”) is the concrete unit of work in the repository.

- Path: `modules/<module_id>/`
- Contains all code and AI-facing documentation for a specific business responsibility.
- All AI work is scoped to a single module instance at a time (for both coding and context tracking).
- Every module instance maintains its own:
  - Local policies and usage guide (`AGENTS.md`).
  - Knowledge routing entry point (`ROUTING.md` + `routes/`).
  - Ability routing entry point (`ABILITY.md`).
  - Work context (`workdocs/`).
  - Interaction contracts (`interact/`).
  - Configuration (`config/`).
  - Human-readable docs (`docs/`).

The module instance is the **primary context boundary** for AI.

### 1.2 Module Type

A **module type** describes how modules behave in a business and data-flow sense.

Projects may choose to define `module_type` in two main ways:

1. **Domain-/flow-oriented types (recommended for business-heavy systems)**  
   - Types represent business stages or capabilities (for example, `assign`, `collect`, `overview`, `grading` in a homework system).
   - The type graph then models allowed progressions between stages and the main data/control flows across the business process.

2. **Technical-layer types (alternative pattern)**  
   - Types represent technical layers (for example, `web_frontend`, `http_api`, `domain_service`, `data_pipeline_step`, `llm_orchestrator`).
   - The type graph then models allowed calls between technical layers.

In both cases:

- Types form a **directed graph**:
  - Nodes = types.
  - Edges = allowed data / control flows between types.
- The type graph:
  - Constrains which modules may depend on or call which other modules.
  - Is used by testing and orchestration to construct integration paths and validate scenarios.
- The template maintains a **type registry** and **type graph** at `modules/overview/type_registry.yaml`.

Module instances declare their `module_type` in `MANIFEST.yaml` and are validated against the type graph.

### 1.3 Module Level

**Module level** is independent from type. It captures composition and hierarchy.

- Each instance can declare:
  - `module_level`: a small integer or string (e.g. `domain`, `feature`, `subfeature`).
  - `parent_module`: the `module_id` of its parent, if any.
- Levels form a **tree of modules**:
  - Used primarily for human understanding, documentation grouping, and navigation.
  - The system does not use this as a hard constraint for runtime behavior.

---

## 2. Module Instance Skeleton

Every module instance lives under `modules/<module_id>/` and follows the skeleton below.

```text
modules/
  <module_id>/
    MANIFEST.yaml          # Module metadata and dependencies
    AGENTS.md              # Module-level AI policies & usage guide
    ROUTING.md             # Entry point for knowledge routing
    ABILITY.md             # Entry point for ability/tool routing

    workdocs/              # AI work context and history
      AGENTS.md            # How to use workdocs in this module
      active/              # Ongoing AI tasks
        # <task_id>/ folders
      outcomes/            # Cross-task knowledge and scripts
        decisions.md
        lessons.md
        scripts/
      archive/             # Completed tasks and old summaries
        summary/
        old_tasks/

    routes/                # Concrete knowledge routing rules
    interact/              # Contracts, DB usage, test data, etc.
    config/                # Module configuration
    src/                   # Code (backend/frontend/core/etc.)
      backend/
      frontend/
      core/
      ...                  # As per technology conventions
    docs/                  # Human-facing docs (README, reports, etc.)
    outcomes/              # Optional output artifacts (reports, exports)
```

### 2.1 Core AI-Facing Files

- **`MANIFEST.yaml`**  
  Single source of truth for module metadata. Recommended fields:

  ```yaml
  module_id: payments.api
  name: Payments HTTP API
  description: Handles payment creation, capture, refunds, and webhooks.
  module_type: http_api
  module_level: feature
  parent_module: commerce.payments
  owners:
    - team: payments
      slack: "#payments-dev"
  status: active        # active | deprecated
  dependencies:
    modules:
      - payments.core
      - users.api
    external_services:
      - stripe
  tags:
    - domain:payments
    - category:revenue
  ```

- **`AGENTS.md` (module-level)**  
  The AI usage manual for this module. It should:

  - Explain the module’s responsibility and boundaries.
  - Point to all relevant sub-policies (e.g., `src/backend/AGENTS.md`).
  - Specify coding conventions, review expectations, and “do/don’t” rules.
  - Define where to log decisions and lessons.
  - Provide a recommended reading order for AI agents.

- **`ROUTING.md`**  
  Entry point for **knowledge routing** (see also the separate `content_routing` spec):

  - Lists top-level topics (scopes) relevant to this module.
  - For each topic, points to a concrete routing file under `routes/`.
  - Explains when to open which routing file.

  Example:

  ```markdown
  # Knowledge Routing – payments.api

  ## Topics

  - `business_rules` → `routes/business_rules.yaml`
  - `error_handling` → `routes/error_handling.yaml`
  - `integration_stripe` → `routes/stripe_integration.yaml`
  ```

- **`ABILITY.md`**  
  Entry point for **ability (tool) routing**:

  - Lists recommended tools / scripts / functions to reuse before writing new code.
  - Does **not** hard-block other tools; it is a prioritized hint set.
  - Should be updated as the module gains new reusable capabilities.

  Example:

  ```markdown
  # Ability Routing – payments.api

  ## When implementing payment flows

  1. Prefer `scripts/retry_failed_payments.py` for batch retries.
  2. Prefer `payments.core.PaymentService` for single payment operations.
  3. If no existing tool applies, record the new script in `workdocs/outcomes/scripts/`.
  ```

### 2.2 `interact/` – Declaring External Interactions

`interact/` describes how this module interacts with external state and systems. This is essential for safe AI changes and cross-module reasoning.

Recommended structure:

```text
interact/
  db/
    schema.md              # Tables, columns, ownership
    operations.md          # Read/write patterns and constraints
  contracts/
    http.yaml              # HTTP APIs (OpenAPI or similar)
    events.yaml            # Message/event contracts
  testdata/
    mocks/                 # Mock definitions and generators
    fixtures/              # Persistent fixtures and lifecycle rules
  other/
    notes.md               # Any additional interaction contracts
```

- **Database**: document any tables / fields this module creates or modifies, and the nature of operations (read-only vs write, transactional constraints, migration strategy).
- **Contracts**: keep API and event schemas in sync with actual code; changes to code should update these files and trigger contract checks.
- **Test data**: define where mocks and fixtures live, how they are generated, and how long they live.

### 2.3 `config/` – Configuration

`config/` holds module-level configuration. Module config overrides project-level config when keys conflict.

Example structure:

```text
config/
  AGENTS.md           # How to read and update config
  db/                 # DB connection, migration strategy
    connections.yaml
    migrations.yaml
  llm/                # LLM API, prompt buckets, hyperparameters
    providers.yaml
    prompts/
  telemetry/          # Logs, metrics, alerts
    logging.yaml
    metrics.yaml
    alerts.yaml
  parameters/         # Module-level parameters and constants
    defaults.yaml
    overrides.yaml
  prompts/            # Prompt templates and intent rules
    classification.yaml
    coding.yaml
  feature_flags/      # Feature switches and rollout strategies
    flags.yaml
```

Typical configuration may include:

- Connection details and retry/backoff strategies.
- LLM provider selection, temperature, max tokens, and prompt IDs.
- Logging levels, metric names, alert thresholds.
- Business parameters (limits, thresholds, default values).
- Feature flags with owners and rollout plans.

### 2.4 `docs/` – Human-Facing Documentation

`docs/` is for humans. Typical content:

- `README.md`: high-level overview, how to run and test the module.
- Design docs and RFCs.
- Observability dashboards and performance reports.
- Evaluation results generated by AI or scripts.

The main rule: if humans need to read it, it belongs here and should be linked from module-level `AGENTS.md` where relevant.

---

## 3. Workdocs: Maintaining AI Execution Context

The `workdocs/` directory is the core of **AI work continuity**. It enables a loop of:

> think → decide → execute → record → review → think again

Even if the AI loses in-memory context, it can restore state by reading these files.

### 3.1 When to Use Workdocs

Use `workdocs/active/` for tasks that:

- Are complex, high-risk, cross multiple files, or cross multiple sessions.
- Implement a full feature or significant refactor.
- Require coordination with other modules or teams.

You may **skip** `workdocs/active/` for trivial work such as:

- One-line bug fixes.
- Mechanical renames.
- Minor copy changes.

Update frequency:

- Update task files whenever you finish a milestone.
- Update at least once before ending a session if the task is not complete.

### 3.2 Structure of `workdocs/`

```text
workdocs/
  AGENTS.md              # Usage guide for workdocs in this module
  active/
    <task_id>/
      plan.md            # Plan to complete the task
      context.md         # Current context and findings
      tasks.md           # Checklist and progress tracking
  outcomes/
    decisions.md         # Cross-task decisions
    lessons.md           # Cross-task errors and lessons
    scripts/             # Reusable scripts and helper tools
  archive/
    summary/             # Summaries of long-running or closed tasks
    old_tasks/           # Archived task folders
```

**Task IDs**

- Use stable, descriptive IDs: `T-YYYYMMDD-short-slug`, e.g. `T-20251108-payments-refactor`.
- The folder name **must** be the task ID.

### 3.3 `plan.md` – Execution Plan

Purpose:

- Provide a reviewable plan for humans.
- Provide a stable reference for AI to track scope changes.

Recommended structure:

```markdown
# <Feature or Task Name> – Plan (T-20251108-payments-refactor)

## 1. Executive Summary
- What we are building and why (1–3 bullets).

## 2. Current State
- Key files and behaviors today.
- Known issues or limitations.

## 3. Desired Future State
- Target behavior and constraints.
- Non-goals (what will not change).

## 4. Implementation Phases
### Phase 1: <Name>
- Task 1.1: ...
  - Acceptance: ...
- Task 1.2: ...
  - Acceptance: ...

### Phase 2: <Name>
...

## 5. Risks & Mitigations
- Risk: ...
- Mitigation: ...

## 6. Success Metrics
- How we know this task is done and successful.
```

Rules:

- Keep the plan **aligned with reality**. When the scope changes, update `plan.md` and record the reason.
- Each major phase should have acceptance criteria that can be checked from code/tests.

### 3.4 `context.md` – Fast Context Restore

`context.md` answers the question: **“If I lost memory and just arrived, how do I resume this task?”**

Recommended structure:

```markdown
# <Feature or Task Name> – Context (T-20251108-payments-refactor)

## 1. Session Progress (2025-11-08)
### Completed
- ...
### In Progress
- ...
### Blockers
- ...

## 2. Key Files and Responsibilities
- `src/controllers/PaymentController.ts`: ...
- `src/services/PaymentService.ts`: ...

## 3. Important Decisions
- See DECISION-003, DECISION-004 in `workdocs/outcomes/decisions.md`.

## 4. Known Issues and Hypotheses
- ISSUE-001: ...
  - Hypothesis: ...
  - Next steps: ...

## 5. Quick Resume Guide
1. Read this file.
2. Open `plan.md` and check the current phase.
3. Continue with `<specific function/file>`.
4. Update `tasks.md` when a checklist item is completed.
```

Update `context.md` when:

- You complete a sub-phase.
- You discover a significant issue or insight.
- You change assumptions or hypotheses.

### 3.5 `tasks.md` – Checklist and Status

`tasks.md` tracks progress and forces explicit acceptance criteria.

Recommended structure:

```markdown
# <Feature or Task Name> – Tasks (T-20251108-payments-refactor)

## Task Overview
- Description: ...
- Current status: IN_PROGRESS   # PLANNED | IN_PROGRESS | BLOCKED | DONE | CANCELLED
- Notes: ...

## Checklist

### Phase 1: Setup
- [x] Create database schema (Acceptance: migrations run locally and in CI).
- [x] Set up controllers.
- [ ] Configure Sentry error reporting.

### Phase 2: Implementation
- [x] Implement PaymentService with retry logic.
- [ ] Implement idempotency for createPayment.
- [ ] Add input validation and error mapping.

### Phase 3: Tests & Observability
- [ ] Unit tests passing.
- [ ] Contract tests passing.
- [ ] Logs and metrics verified in staging.
```

Rules:

- Each checkbox should be **objectively verifiable** from code or tests.
- If a checklist item becomes irrelevant, mark it with `~` and explain why.

### 3.6 `outcomes/` – Decisions, Lessons, Scripts

`workdocs/outcomes/` aggregates cross-task knowledge and reusable assets for the module.

#### 3.6.1 `decisions.md`

Records important decisions, discoveries, and meaningful file additions/deletions.

Recommended header:

```markdown
# Decisions – Module <module_id>

Usage: Record decisions that affect behavior, contracts, or architecture.
Each decision has a stable ID so other docs can link to it.
```

Decision entry template:

```markdown
## DECISION-001 (2025-11-08)
- type: file_addition       # file_deletion | decision | discovery
- description: Introduce PaymentService as the only entrypoint for payment logic.
- reason: Avoid duplication and inconsistent business rules across controllers.
- impact:
  - Affects `src/controllers/PaymentController.ts`
  - Introduces `src/services/PaymentService.ts`
- related_tasks:
  - T-20251108-payments-refactor
- status: completed          # waiting_approval | in_progress | completed
```

Triggers to update:

- New or changed public interfaces.
- Architectural changes.
- Non-trivial refactors.
- Discoveries about external systems or constraints.

#### 3.6.2 `lessons.md`

Records mistakes, their consequences, and how to avoid them.

Recommended structure:

```markdown
# Lessons – Module <module_id>

## Index
- ERROR-001: Missed contract update for payments API.
- ERROR-002: ...

---

## ERROR-001 (2025-11-08)
- status: solved              # open | solved | obsolete
- summary: Forgot to update `contracts/http.yaml` after changing API response.
- context:
  - Task: T-20251108-payments-refactor
  - Files: `src/controllers/PaymentController.ts`, `interact/contracts/http.yaml`
- root_cause: Changed response shape without updating the HTTP contract.
- consequence: Frontend calls failed; required 3 rounds of debugging.
- lesson: Any API response change must update `interact/contracts/http.yaml` and re-run contract tests.
- reminder_or_solution:
  - Guardrail: Run `make contract_compat_check` when modifying controllers.
  - Consider adding CI check that compares controller response types with contract.
```

The **index** at the top should map error IDs to short summaries to help AI quickly decide whether a lesson is relevant.

#### 3.6.3 `scripts/`

Stores scripts or helpers created during development that are worth reusing.

Guidelines:

- Each script must contain a **machine-readable metadata block** at the top (comment or docstring), for example:

  ```python
  """
  script_id: S-20251108-retry-failed-payments
  module_id: payments.api
  purpose: Retry failed payments older than 24 hours.
  inputs:
    - db: read/write payments.transactions
  outputs:
    - db: updates status and retry_count
  safe_to_repeat: true
  """
  ```

- Scripts that are promoted to stable tools should be:
  - Linked from `ABILITY.md`.
  - Potentially moved into a shared tools library, with `scripts/` keeping only thin wrappers or references.

### 3.7 `archive/` – Completed Work

`archive/` is maintained primarily by automation (e.g., CI jobs). It preserves completed work and makes it easy for AI and humans to reconstruct what happened.

```text
workdocs/
  archive/
    summary/
      T-20251108-payments-refactor.md
    old_tasks/
      T-20251108-payments-refactor/
```

#### 3.7.1 Archiving Rules

- After a task is completed and stable, move its folder from `active/` to `archive/old_tasks/`.
- Each archived task folder should keep its original `plan.md`, `context.md`, and `tasks.md`.
- Each archive entry should be timestamped, so AI can prefer **recent** material when multiple tasks touch similar areas.

#### 3.7.2 `summary/` – Task Summaries

For long-running or high-impact tasks, create a summary file in `archive/summary/` named after the task ID:

- File name: `T-<YYYYMMDD>-<slug>.md`, e.g. `T-20251108-payments-refactor.md`.
- Purpose: provide a **fast, stable overview** of the task outcome without reading the full archived task folder.

Recommended template:

```markdown
# Summary – T-20251108-payments-refactor

## 1. Task Overview
- Module: payments.api
- Time window: 2025-11-08 ~ 2025-11-10
- Outcome: ...

## 2. Key Changes
- Code:
  - src/services/PaymentService.ts
  - src/controllers/PaymentController.ts
- Contracts:
  - interact/contracts/http.yaml#/paths/~1payments/post

## 3. Important Decisions
- DECISION-001, DECISION-002 (see `workdocs/outcomes/decisions.md`)

## 4. Lessons and Risks
- ERROR-001, ERROR-003 (see `workdocs/outcomes/lessons.md`)
- Residual risks: ...
```

Guidelines:

- Not every task needs a summary. Prioritize:
  - Long-running tasks.
  - Tasks with cross-module impact.
  - Tasks that changed public interfaces or data contracts.
- When in doubt, prefer writing a **short summary** rather than none; summaries are cheap context for future AI sessions.

---

## 4. Additional Module Metadata

Besides workdocs, each module must keep metadata in `interact/`, `config/`, and `docs/` to help AI reason safely.

### 4.1 Database Interactions

The goal of `interact/db/` is to make **data shape** and **data usage** explicit at the module level, so AI and automation can safely change code without surprising downstream consumers.

Each module should maintain at least two documents:

```text
interact/
  db/
    schema.md       # Tables, columns, ownership, and semantics
    operations.md   # Business operations → tables/fields/invariants
```

#### 4.1.1 `schema.md` – Structure and Semantics

`schema.md` describes the structural aspects of the data this module owns or actively manipulates:

- Tables and views this module creates or writes to.
- Columns, types, and constraints (PK/FK, uniqueness, check constraints).
- Ownership and responsibility (which module is the primary owner).
- High-level semantics for important fields (e.g., enums, status machines).

This document is the main reference when:

- Designing new features that require schema changes.
- Reviewing the impact of migrations.
- Reasoning about ownership and cross-module coupling.

#### 4.1.2 `operations.md` – Business Operations → Tables / Fields / Invariants

`operations.md` maps **business-level operations** (not individual functions) to the tables and fields they rely on, and the invariants they must preserve.

Recommended structure:

```markdown
# Operations – <module_id>

## Operation: Create Payment
- Description: Create a new payment for an order.
- Tables:
  - payments.transactions
    - fields:
      - id (PK)
      - order_id (FK → orders.id)
      - status (enum: PENDING, AUTHORIZED, CAPTURED, FAILED)
      - amount
- Invariants:
  - For a given `order_id`, at most one ACTIVE transaction (status != FAILED).
- Typical call sites:
  - src/controllers/PaymentController.ts#createPayment

## Operation: Cancel Payment
- Description: Cancel an existing payment and release authorization.
- Tables:
  - payments.transactions
    - fields:
      - status (enum: PENDING, AUTHORIZED, CAPTURED, FAILED, CANCELED)
- Invariants:
  - Cannot cancel a CAPTURED payment.
- Typical call sites:
  - src/services/PaymentService.ts#cancelPayment
```

Guidelines:

- Focus on **conceptual operations** that are meaningful for humans and AI:
  - “Create Payment”, “Cancel Order”, “Grant Refund”, “Rotate API Keys”.
- Do **not** attempt to track every individual function or method:
  - This mapping is not a call graph; it is an **impact map**.
- Whenever a new operation is introduced or an existing operation changes semantics:
  - Update `operations.md`.
  - Consider adding or updating relevant entries in `workdocs/outcomes/decisions.md` and `lessons.md`.

#### 4.1.3 When to Update `interact/db/`

Update `schema.md` and/or `operations.md` when:

- A table or column is added, removed, or repurposed.
- A business operation starts touching new tables/fields, or changes its invariants.
- Functions or scripts perform bulk writes or destructive operations.
- Transactions or consistency guarantees are introduced or tightened.

Keeping `interact/db/` current enables:

- Safer refactors and migrations.
- Better integration and regression test planning.
- Transparent impact analysis for AI-driven changes.

### 4.2 Tests

In `interact/testdata/` (and/or a dedicated `tests/` directory) document:

- How test data is generated (mocks vs fixtures).
- Data formats, storage locations, and lifecycle rules.
- Which scenarios rely on which fixtures.

This helps AI:

- Choose the right fixtures when writing tests.
- Understand which tests are safe to run in which environments.

### 4.3 Interface Contracts

In `interact/contracts/` document:

- Public APIs defined by the module type (business-facing interfaces).
- Internal service interfaces (HTTP, gRPC, events, message queues).
- For each, keep the contract in sync with the code and the gateway configuration.

These contracts are cross-referenced by the **interface registry** at the project level (see Section 5.2.3).

### 4.4 Configuration and Feature Flags

Use `config/` to:

- Centralize non-secret configuration.
- Prefer parameters and feature flags over hard-coded magic numbers.
- Express rollout strategy and ownership.

Module-level configuration overrides project-level defaults when keys conflict.

---

## 5. AI Workflow

This section describes how an AI orchestrator should use the template for single-module and cross-module work.

### 5.1 Single-Module Development

For a task scoped to a single module instance:

1. **Identify the module**
   - Determine `module_id` from the task description.
   - Navigate to `modules/<module_id>/`.

2. **Load policies and context**
   - Read `AGENTS.md` (module-level).
   - If the task is complex or cross-session, create or open a task folder under `workdocs/active/` and read:
     - `plan.md`
     - `context.md`
     - `tasks.md`

3. **Load relevant knowledge and tools**
   - From `ROUTING.md`, locate relevant routing files in `routes/` and load only what is needed.
   - From `ABILITY.md`, identify recommended tools and scripts.

4. **Plan**
   - If no `plan.md` exists for this task, create one following Section 3.3.
   - If a plan exists, validate that it still matches the desired outcome; update if the scope changed.

5. **Execute**
   - Implement changes in `src/` following the module’s AGENTS policies.
   - Keep changes aligned with declared DB interactions, contracts, and config.

6. **Maintain workdocs**
   - Update `context.md` as you complete phases or discover issues.
   - Update `tasks.md` checklist.
   - Log any significant decisions in `workdocs/outcomes/decisions.md`.
   - Log any mistakes and lessons in `workdocs/outcomes/lessons.md`.
   - Place reusable scripts in `workdocs/outcomes/scripts/` and reference them in `ABILITY.md` if appropriate.

7. **Finish**
   - Mark the task as `DONE` in `tasks.md`.
   - Optionally summarize the task in `archive/summary/` once archived.

### 5.2 Multi-Module Integration and Testing

Cross-module work (integration tests, performance investigations, end-to-end flows) must **not** overload a single module instance. Instead, it uses project-level structures under `modules/`.

#### 5.2.1 Project-Level Layout

```text
modules/
  AGENTS.md                   # Root-level AI policies
  ROUTING.md                  # Root-level knowledge routing
  routes/                     # Root-level knowledge routing rules

  overview/                   # Project-wide registries
    AGENTS.md
    README.md
    type_registry.yaml        # Module types and type graph
    instance_registry.yaml    # All module instances
    instance_maturity.yaml    # Module instance completion summary
    interfaces.yaml           # Public interface registry
    api_gateway.yaml          # API gateway exposure and rules

  integration/                # Cross-module testing & integration
    AGENTS.md                 # Integration-specific policies
    ROUTING.md                # Integration knowledge routing
    ABILITY.md                # Integration-related tools
    scenarios.yaml            # End-to-end and integration scenarios
    workdocs/
      <scenario_id>/          # One per scenario or integration task
        plan.md
        context.md
        tasks.md
    outcomes/
      reports/                # Test and performance reports
    tools/                    # Human-facing tools (dashboards, scripts)

  config/                     # Project-level config
    api_gateway/              # Gateway level configuration
    parameters/
    prompts/
```

#### 5.2.2 Type Graph – `type_registry.yaml`

Defines all module types and allowed data flows between them.

A common and recommended pattern is to let module types represent **business-flow stages or capabilities**. For example, in a homework system:

```yaml
types:
  - id: assign
    description: Create and publish assignments.
    outbound:
      - collect
      - overview

  - id: collect
    description: Collect student submissions for an assignment.
    inbound:
      - assign
    outbound:
      - overview
      - grading

  - id: overview
    description: Provide dashboards and summary views over assignments and submissions.
    inbound:
      - assign
      - collect
      - grading
    outbound: []

  - id: grading
    description: Grade submissions and produce feedback.
    inbound:
      - collect
    outbound:
      - overview
```

Rules:

- Each `module_type` declared in `MANIFEST.yaml` **must** exist here.
- Dependencies between modules must follow the type graph’s allowed edges.
- Updates are rare and must be made when:
  - A new type is introduced.
  - Data flows between types change.
  - Obsolete flows are removed.
- Projects that prefer a technical-layer type graph (e.g. `web_frontend` → `http_api` → `domain_service`) can adopt the variant described in Appendix A.

**Validation and coverage:**

- **Scenario → type graph (legality check)**  
  For any integration or end-to-end scenario defined in `modules/integration/scenarios.yaml`:
  - For every interaction between modules (derived from `required_interfaces` and `interfaces.yaml`), the caller’s `module_type` and callee’s `module_type` **must** form a valid edge in `type_registry.yaml`.
  - CI and/or the orchestrator should reject scenarios that introduce interactions not allowed by the type graph.

- **Type graph → scenarios (coverage insight)**  
  The project may maintain additional metadata (e.g., in `instance_maturity.yaml` or a dedicated coverage file) to indicate which type edges are exercised by at least one scenario. This can be used to:
  - Identify edges that are allowed by architecture but never exercised in tests.
  - Prioritize new scenarios or test expansions for under-covered edges.

#### 5.2.3 Instance Maturity – `instance_maturity.yaml`

Summarizes per-module progress to avoid scanning every module.

Example:

```yaml
instances:
  - module_id: payments.api
    module_type: http_api
    maturity: READY_FOR_INTEGRATION   # SCAFFOLDED | IN_PROGRESS | FUNCTIONAL | READY_FOR_INTEGRATION | READY_FOR_RELEASE | DEPRECATED
    capabilities:
      interfaces_implemented: 0.8     # ratio 0..1
      docs_quality: GOOD              # POOR | FAIR | GOOD | EXCELLENT
    tests:
      unit: PASSED
      contract: PASSED
      integration: NOT_RUN
      e2e: NOT_REQUIRED

  - module_id: payments.core
    module_type: domain_service
    maturity: FUNCTIONAL
    tests:
      unit: PASSED
      contract: NOT_APPLICABLE
      integration: NOT_RUN
      e2e: NOT_REQUIRED
```

Update process (preferably automated via CI):

- Track changes in `workdocs/active/` and test results.
- When milestones are reached (e.g., all planned interfaces implemented, tests passing), bump `maturity`.
- Archive timestamps so AI can know how fresh information is.

#### 5.2.4 Interfaces and Gateway – `interfaces.yaml` & `api_gateway.yaml`

Interfaces are the key objects for integration and end-to-end tests.

Interface entry example (`interfaces.yaml`):

```yaml
interfaces:
  - id: payments.create_payment
    module_id: payments.api
    path: /payments
    method: POST
    description: Create a new payment.
    lifecycle: IMPLEMENTED        # DESIGNED | MOCKED | IMPLEMENTED | QA_VERIFIED | READY_FOR_RELEASE | RELEASED | SUNSET | REMOVED
    quality:
      unit_test_status: PASSED    # NOT_RUN | FAILED | PASSED
      contract_test_status: PASSED
      integration_test_status: NOT_RUN
      e2e_test_status: NOT_REQUIRED
      perf_test_status: NOT_REQUIRED
      security_scan_status: PASSED
    contracts:
      http: interact/contracts/http.yaml#/paths/~1payments/post
```

Gateway entry example (`api_gateway.yaml`):

```yaml
routes:
  - interface_id: payments.create_payment
    exposure: INTERNAL      # HIDDEN | INTERNAL | PARTNER | PUBLIC
    traffic_policy: CANARY  # DISABLED | MOCK | SHADOW | CANARY | LIMITED | NORMAL
    health: HEALTHY         # HEALTHY | DEGRADED | UNAVAILABLE
```

Lifecycle rules:

- On module creation or new interface design:
  - Add the interface with `lifecycle: DESIGNED`.
- During module development:
  - Promote to `MOCKED` or `IMPLEMENTED` as coding and contract tests are done.
- During integration and scenario testing:
  - Promote to `QA_VERIFIED` or `READY_FOR_RELEASE`.
- `RELEASED`, `SUNSET`, `REMOVED`:
  - Only human maintainers can set these.

Before running integration scenarios:

- Ensure all referenced interfaces have appropriate lifecycle and quality status.
- If not, list blocking interfaces and suggest module-level work first.

#### 5.2.5 Scenarios – `scenarios.yaml`

Scenarios describe integration or end-to-end flows from the user’s or system’s perspective.

Example:

```yaml
scenarios:
  - id: S-checkout-happy-path
    name: Checkout happy path via web
    description: User completes a purchase using credit card through web frontend.
    status: IN_PROGRESS        # PLANNED | IN_PROGRESS | BLOCKED | VERIFIED | DEPRECATED
    last_run_at: 2025-11-08T13:45:00Z
    involved_modules:
      - web.checkout.ui
      - orders.api
      - payments.api
    required_interfaces:
      - payments.create_payment
      - orders.create_order
    preconditions:
      - test_user_exists
      - test_catalog_populated
    steps:
      - given: A logged-in user with items in cart
        when: User clicks "Pay"
        then:
          - An order is created
          - A payment authorization is created
    expected_outcomes:
      - Order status is `PAID`
      - Payment status is `AUTHORIZED`
    workdocs_task_id: T-20251108-checkout-e2e
```

Rules:

- AI should always check `scenarios.yaml` first when doing cross-module testing.
- If no scenario matches the request:
  - Create a new scenario entry following the template.
  - Create a corresponding task folder under `modules/integration/workdocs/`.

**Validation and coverage:**

- When a scenario is **created or modified**:
  - The orchestrator or CI pipeline **must** validate that all interactions implied by `required_interfaces` and `involved_modules` respect the type graph in `modules/overview/type_registry.yaml`.
    - For each required interface, look up its `module_id` and `module_type` in `interfaces.yaml` and `instance_registry.yaml`.
    - Ensure the caller and callee module types form a valid directed edge in the type graph.
  - If any interaction violates the type graph, the scenario definition should be rejected or flagged for explicit human approval.

- The project may additionally:
  - Track which type graph edges are covered by which scenarios (e.g., in a separate coverage report).
  - Use this information to:
    - Suggest new scenarios for uncovered but important edges.
    - Prioritize regression testing on heavily used edges.

#### 5.2.6 Integration Workdocs

For each scenario-related task, use the same `plan/context/tasks` pattern as module workdocs, but **from the testing perspective**:

- `plan.md`:
  - Define test objectives, coverage boundaries, and metrics (e.g., latency thresholds).
- `context.md`:
  - Record which scenarios were run, data used, and what failed.
  - Record discovered issues and, if possible, which module they likely belong to.
- `tasks.md`:
  - Track progress of test implementation and execution.
  - Track whether regressions have been fixed.

Key difference vs feature development:

- Tests may continue even if some issues are present; the goal is often to **discover** issues, not immediately fix them.
- Issues discovered should be logged with:
  - Scenario ID.
  - Likely owning module and interface.
  - Links to module-level tasks when created.

#### 5.2.7 Cross-Module AI Workflow

A typical AI-driven integration workflow:

1. **Clarify need**
   - Interpret user input as a test scenario.
   - Search `scenarios.yaml`:
     - If a matching scenario exists and is `IN_PROGRESS`, load its workdocs and continue.
     - If it exists and is `VERIFIED`, summarize past results and confirm whether a new run or variation is needed.
     - If it does not exist, create a new scenario and workdocs task.

2. **Decompose work**
   - Build a test plan: which interfaces, which data paths, which metrics.
   - Define milestones in `tasks.md` (e.g., “happy path implemented”, “negative cases implemented”, “performance baseline collected”).

3. **Execute tests and maintain context**
   - Run tests according to the scenario.
   - Update `context.md` with each major finding.
   - For each discovered issue:
     - Record it in `context.md`.
     - Identify the most likely owning module and reference its `module_id`.
     - Optionally open or link to a module-level task.

4. **Fix or delegate**
   - For critical or obvious issues, optionally perform fixes in the owning module:
     - Follow module-level workflow (Section 5.1).
   - Otherwise, clearly document issues and their suspected owners so humans or subsequent AI sessions can handle them.

5. **Output**
   - When a milestone or full scenario is complete:
     - Update the scenario status in `scenarios.yaml`.
     - Generate a summary report under `modules/integration/outcomes/reports/`.
   - When required, produce a human-facing summary and link it from the relevant `docs/` directories.

#### 5.2.8 Strategy Documents for Cross-Instance Work

Strategy documents (`AGENTS.md`) are the main entry point for AI to understand how to work **across** module instances, not just inside a single module. At the project root, `modules/AGENTS.md` should:

1. Define reading order and entry points for cross-instance tasks  
   When a task involves multiple modules, the root `AGENTS.md` should instruct AI to first read:

   - **`modules/overview/AGENTS.md`**: Explain how module types, module instances, interfaces, and the gateway are modeled at the project level.
   - **`modules/integration/AGENTS.md`**: Explain the workflow, responsibilities, and guardrails for integration and cross-module testing.

1. Route to core cross-instance specifications  
   The root `AGENTS.md` should enumerate, with short descriptions, the key documents that define cross-instance behavior, for example:

   - **Module and type modeling rules**  
     - How to define `module_type`.  
     - How to use `module_level` and `parent_module` to represent hierarchy.  
     - Where to find these rules.

   - **Type graph and dependency constraints**  
     - How `modules/overview/type_registry.yaml` is structured.  
     - How the type graph constrains allowed calls and data flows between module types.  
     - How legality checks for scenarios depend on this file.

   - **Instance and interface registration rules**  
     - The meaning of fields in `instance_registry.yaml`, `interfaces.yaml`, and `api_gateway.yaml`.  
     - Required steps when adding or changing module instances and interfaces.

   - **Cross-instance context and scenario management**  
     - How `modules/integration/scenarios.yaml` defines integration/end-to-end scenarios.  
     - How `modules/integration/workdocs/` is used to maintain `plan.md`, `context.md`, and `tasks.md` for each integration task.

2. Define AI behavior for cross-instance situations  
   The root `AGENTS.md` should clearly state how AI should behave when it detects a cross-instance task, for example:

   - On receiving a **cross-module testing / integration** request, AI should:
     1. Start at `modules/AGENTS.md`.  
     2. Follow its routing to:
        - `modules/overview/AGENTS.md` (project overview and type graph).  
        - `modules/integration/AGENTS.md` (integration workflow and policies).  
        - The `AGENTS.md` files of the specific modules involved.
   - If existing scenarios are missing or clearly insufficient, AI should prefer to:
     - Use the scaffolding tools described in Section 5.2.9 to create or extend scenarios and integration tasks,
     - Rather than improvising ad-hoc test logic directly in code.

In summary, strategy documents should **route and constrain** cross-instance work, not attempt to embed all details. Detailed rules live in the overview, integration, and module-level specs; `AGENTS.md` makes sure AI finds and respects them.

#### 5.2.9 Scaffolding Tools for Integration Scenarios

Scaffolding tools exist to turn fuzzy human requests into **structured, documented integration scenarios**. Their primary goals are:

- Collect cross-instance requirements completely and consistently.  
- Project them into the agreed files (`scenarios.yaml`, `workdocs/`, registries) so that AI, CI, and humans share a single source of truth.

At minimum, scaffolding tools should support the following capabilities:

1. Scenario creation and update

   The tool should guide users (human or AI) through an interactive flow to capture the essential information for a cross-instance scenario, such as:

   - Business objective (for example, “from assignment creation to grading and overview”).  
   - Involved business-flow types and stages.  
   - Known module instances and variants.  
   - Preconditions (data setup, environment assumptions, feature flags).  
   - Acceptance criteria (success conditions, key metrics, tolerances).

   Based on this input, the tool should:

   - Create or update the corresponding entry in `modules/integration/scenarios.yaml`.  
   - Generate or update a matching folder in `modules/integration/workdocs/` with initial `plan.md`, `context.md`, and `tasks.md` skeletons for the scenario.

2. Validation against type graph and registries

   When creating or modifying a scenario, the scaffolding tool should automatically read:

   - `modules/overview/type_registry.yaml` (type graph).  
   - `modules/overview/instance_registry.yaml` (module instances).  
   - `modules/overview/interfaces.yaml` (interfaces and owners).

   Using this data, it should:

   - Infer the expected call chain and type-level edges implied by the scenario.  
   - Validate that:
     - Caller and callee `module_type` pairs match allowed edges in the type graph.  
     - Referenced interfaces exist in `interfaces.yaml` and are in a lifecycle state that permits testing.

   If it detects violations (for example, flows not allowed by the type graph, or interfaces not yet implemented), the tool should:

   - Provide explicit feedback on what is invalid and why.  
   - Suggest whether the user should adjust the scenario or first implement/upgrade the relevant modules or interfaces.

3. Stable entry points for AI

   After creating or updating a scenario, the scaffolding tool should also:

   - Write a concise **scenario summary** into:
     - The `description` field in `scenarios.yaml`.  
     - A short summary section in the scenario’s `plan.md` or `context.md`.

   This ensures that future AI sessions can:

   - Find the scenario via `modules/integration/scenarios.yaml`.  
   - Open its workdocs folder.  
   - Quickly restore intent and context from the summary, without re-asking humans for the same information.

Scaffolding tools may be implemented as CLI commands, a web console, or IDE plugins. Regardless of form, they must strictly adhere to the repository conventions in this specification so that:

- All cross-instance work is discoverable and auditable in plain-text files.  
- AI orchestration and CI pipelines can rely on the same structured metadata.  
- Cross-module development remains efficient, repeatable, and safe.

### 5.3 Instance Registration and Removal

#### 5.3.1 Registration via Scaffolding

A scaffolding script initializes a new module instance to guarantee consistency.

Typical flow:

1. **Collect information (interactive)**
   - `module_id`, `name`, `module_type`, `module_level`, `parent_module`.
   - Domain description and intended responsibilities.
   - Initial dependencies and external services.

2. **Create temporary init folder**
   - `modules/<module_id>/init/` for notes and temporary files during creation.
   - This folder is removed when registration completes.

3. **Generate module skeleton**
   - Create directory structure described in Section 2.
   - Generate minimal content for:
     - `MANIFEST.yaml`
     - `AGENTS.md`
     - `ROUTING.md`
     - `ABILITY.md`
     - `workdocs/AGENTS.md`
     - `interact/`, `config/`, `docs/README.md`

4. **Register in project-level registries**
   - Add entry to `modules/overview/instance_registry.yaml`.
   - If `module_type` is new, add it to `type_registry.yaml`.
   - Initialize entries in `instance_maturity.yaml` and `interfaces.yaml` if applicable.

5. **Finalize**
   - After human confirmation, remove `init/`.
   - Mark the module as `SCAFFOLDED` in `instance_maturity.yaml`.

#### 5.3.2 Definition of “Registered”

A module instance is considered **registered** when:

- `modules/<module_id>/` exists with the full skeleton.
- `MANIFEST.yaml` is complete and valid.
- The module appears in `instance_registry.yaml`.
- Its `module_type` is valid in `type_registry.yaml`.
- It has at least:
  - A minimal `AGENTS.md`.
  - Empty but present `ROUTING.md` and `ABILITY.md`.
  - An initialized `workdocs/` structure.

Business logic may still be empty; registration only guarantees the infrastructure is in place.

#### 5.3.3 Removal and Refactor

Module removal is a controlled operation:

1. **Check dependencies**
   - Use the type graph and instance registry to list all modules that depend on this module.
   - If dependencies are complex, require explicit human approval.

2. **Handle knowledge and abilities**
   - Remove or reassign knowledge documents owned by the module.
   - Remove abilities that are exclusive to this module or move them to a shared location.

3. **Update registries**
   - Remove the module from `instance_registry.yaml` and `instance_maturity.yaml`.
   - Update `interfaces.yaml` and `api_gateway.yaml` to remove or mark interfaces as `SUNSET` / `REMOVED`.

4. **Limit scope**
   - Do not refactor or delete multiple modules in a single operation unless explicitly authorized.

### 5.4 Domain-Oriented Type Graph Example – Homework System

This section illustrates a domain-oriented type graph for a homework system with four major business capabilities:

- Assign: publish homework.
- Collect: collect submissions.
- Overview: provide aggregate views and dashboards.
- Grading: grade submissions and provide feedback.

#### 5.4.1 Stage-Level Types

At the stage level, the type graph describes the high-level flow of work and data across the system:

```yaml
types:
  - id: assign
    description: Create and publish assignments.
    outbound:
      - collect
      - overview

  - id: collect
    description: Collect student submissions for an assignment.
    inbound:
      - assign
    outbound:
      - overview
      - grading

  - id: overview
    description: Provide dashboards and summary views over assignments and submissions.
    inbound:
      - assign
      - collect
      - grading
    outbound: []

  - id: grading
    description: Grade submissions and produce feedback.
    inbound:
      - collect
    outbound:
      - overview
```

This graph expresses that:

- Work typically starts in `assign`, then progresses to `collect`, then to `grading` and `overview`.
- `overview` is a sink that can read from multiple upstream stages to build dashboards.
- The orchestrator should not introduce flows that contradict this graph (for example, `grading` directly calling `assign`), unless the type graph is updated.

#### 5.4.2 Capability-Level Types

Within the `assign` stage, we can introduce more fine-grained capability types:

- `assign.select` – select questions.
- `assign.assemble` – assemble selected questions into an assignment.
- `assign.setting` – configure publishing parameters (deadline, audience, visibility).
- `assign.select_method` – implement concrete selection methods (AI/manual).

Example type entries:

```yaml
types:
  - id: assign.select
    description: Select questions for an assignment.
    inbound:
      - assign
    outbound:
      - assign.assemble

  - id: assign.assemble
    description: Assemble selected questions into an assignment structure.
    inbound:
      - assign.select
    outbound:
      - assign.setting

  - id: assign.setting
    description: Configure assignment metadata (deadline, class, visibility).
    inbound:
      - assign.assemble
    outbound:
      - assign              # Return to the stage-level type once ready to publish.

  - id: assign.select_method
    description: Implement specific selection methods (AI/manual).
    inbound:
      - assign.select
    outbound:
      - assign.select       # Return question candidates back to the selection capability.
```

Here:

- The `assign` stage remains the high-level type for “publish assignments”.
- `assign.select` / `assign.assemble` / `assign.setting` / `assign.select_method` capture internal capabilities and flows inside that stage.
- The orchestrator can reason both at the stage level (e.g., “now we are in `assign`”) and at the capability level (e.g., “we are currently in `assign.select_method`”).

#### 5.4.3 Module Instances and Versions

Concrete module instances implement a specific capability type. For example:

```yaml
# modules/assign.select.ai/ MANIFEST.yaml

module_id: assign.select.ai
name: AI-based question selection
module_type: assign.select_method
module_level: variant          # e.g. 3rd-level capability
parent_module: assign.select   # Optional: parent instance for grouping
```

```yaml
# modules/assign.select.manual/ MANIFEST.yaml

module_id: assign.select.manual
name: Manual question selection
module_type: assign.select_method
module_level: variant
parent_module: assign.select
```

If the AI-based selection module is upgraded, there are two options:

- **New instance version** (coexistence):
  - Create `assign.select.ai_v2` with the same `module_type: assign.select_method`.
  - Route traffic via configuration or `interfaces.yaml` / `api_gateway.yaml`.
- **In-place upgrade** (no coexistence):
  - Keep `module_id: assign.select.ai` and update implementation and metadata.
  - Optionally track a `version` field in `MANIFEST.yaml` or `config/`.

In both cases, the type graph does not change; it still reasons in terms of `assign.select_method`. The module instances are interchangeable workers behind that type.

#### 5.4.4 Scenarios and Legality Checks

A cross-module scenario for “AI-assisted assignment publishing” might involve:

- `assign.select.ai_v2`
- `assign.assemble.default`
- `assign.setting.default`
- `collect.standard`
- `grading.auto`
- `overview.assignments_dashboard`

When the scenario is defined in `modules/integration/scenarios.yaml`:

- The orchestrator uses `interfaces.yaml` and `instance_registry.yaml` to map each interface call to a caller and callee `module_type`.
- The legality checks from Section 5.2.2 ensure that:
  - All flows between types (for example, `assign.select_method` → `assign.select`, `assign.setting` → `assign`, `assign` → `collect`) are valid according to the type graph.
- Coverage reporting (from Sections 5.2.2 and 5.2.5) can highlight which stage-to-stage or capability-to-capability edges are exercised by this and other scenarios.

This domain-oriented type graph makes it clear **how business work is supposed to flow**, while still allowing multiple technical implementations (module instances) behind each capability type.

---

## 6. Infrastructure Collaboration

Module instances provide the **boundaries**; infrastructure (knowledge routing, ability routing, policies, triggers) provides **amplification** for AI capabilities.

### 6.1 Knowledge Routing Collaboration

**Application**

- Each module uses `ROUTING.md` + `routes/` to access the project’s knowledge routing system.
- AI should:
  - Start from `ROUTING.md` to discover relevant topics.
  - Load only the routing files and documents necessary for the current task.

**Enhancement**

- `workdocs/outcomes/decisions.md` and `lessons.md` are the main sources of new knowledge:
  - Regularly review them and promote stable content into dedicated knowledge documents.
  - Register those documents in the routing system so future AI runs discover them.

Overall goal:

- Over time, knowledge becomes deeper and more precise.
- AI spends less time rediscovering decisions or repeating mistakes.

### 6.2 Ability Routing Collaboration

**Application**

- `ABILITY.md` describes the preferred tools and scripts per context.
- AI should:
  1. Infer its capability needs from the task and plan.
  2. Check `ABILITY.md` for matching tools first.
  3. Only search the wider tool pool or introduce new scripts if nothing suitable exists.

**Enhancement**

- When AI writes new scripts to `workdocs/outcomes/scripts/`:
  - Record the intent and usage in script metadata.
  - Log tool-related decisions in `decisions.md`.
- Periodically:
  - Consolidate frequently used scripts into stable, versioned tools.
  - Update `ABILITY.md` to refer to these tools.
  - Deprecate or remove obsolete scripts.

The aim is to keep AI tool usage **consistent, observable, and efficient**.

### 6.3 Strategy Documents and Triggers

**Strategy (`AGENTS.md`)**

- Strategy documents exist at multiple levels:
  - Project (`modules/AGENTS.md`)
  - Module (`modules/<module_id>/AGENTS.md`)
  - Subdirectories (e.g., `src/backend/AGENTS.md`)
- Policies follow a **nearest-wins with upward override** principle:
  - Project-level policies define global rules.
  - Module-level policies specialize for the module.
  - Subdirectory policies specialize further for that area.

Use these documents to:

- Define expectations for workdocs usage.
- Set coding standards and guardrails.
- Describe review and safety rules.

**Triggers and Hooks**

Triggers integrate modules with orchestration and guardrail systems. Examples:

- **Knowledge hooks**:
  - Triggered by user input or task classification.
  - Suggest which routing topics to load from `ROUTING.md`.

- **Ability hooks**:
  - Triggered when AI needs to call tools.
  - Guardrail systems can:
    - Suggest tools from `ABILITY.md`.
    - Enforce pre- and post-conditions.

- **Work process hooks**:
  - Triggered around tool calls (pre_tool_use / post_tool_use).
  - Record decisions and results into `decisions.md`, `lessons.md`, or task-level `context.md`.

The module and project structure in this document is intentionally designed so that:

- Triggers can reason over a predictable directory layout and metadata.
- AI can work efficiently with limited context while still recovering long-running tasks.
- Humans can audit and adjust AI behavior using plain-text, version-controlled documents.

---

## Appendix A. Technical-Layer Type Graph Variant

Some projects may prefer to define `module_type` primarily in terms of **technical layers** rather than business stages. In that case, the type graph describes allowed calls between layers, rather than business-flow stages.

An example type graph for a typical web application:

```yaml
types:
  - id: web_frontend
    description: User-facing web UI.
    outbound:
      - http_api

  - id: http_api
    description: HTTP APIs serving external clients.
    inbound:
      - web_frontend
    outbound:
      - domain_service

  - id: domain_service
    description: Core business logic.
    inbound:
      - http_api
    outbound:
      - data_pipeline_step

  - id: data_pipeline_step
    description: Batch or streaming data processing steps.
    inbound:
      - domain_service
```

In this variant:

- `web_frontend` may call `http_api`, but not the other way around.
- `http_api` may call `domain_service`, but should not directly call `data_pipeline_step` unless explicitly allowed.
- `domain_service` may call `data_pipeline_step` for asynchronous or batch processing.

The same validation and coverage principles apply as in Section 5.2.2:

- Scenarios must only use interactions that respect the type graph.
- The project may track which edges are exercised by which scenarios to understand test coverage at the technical layer level.

This variant is useful when:

- The business domain is simple but the technical architecture is complex.
- The main risks come from violating layering rules (for example, frontend bypassing APIs and directly talking to storage).
- You want to enforce clean architecture boundaries via the type graph.
