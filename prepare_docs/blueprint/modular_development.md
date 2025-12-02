# Modular Development Specification

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

- Examples: `web_frontend`, `http_api`, `background_worker`, `data_pipeline_step`, `llm_orchestrator`.
- Types form a **directed graph**:
  - Nodes = types.
  - Edges = allowed data / control flows between types.
- The type graph:
  - Constrains which modules may depend on which other modules.
  - Is used by testing and orchestration to construct integration paths.
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

`archive/` is maintained primarily by automation (e.g., CI jobs). Rules:

- After a task is completed and stable, move its folder from `active/` to `archive/old_tasks/`.
- Optionally create a short summary in `archive/summary/` for very long-running tasks.
- Each archive entry should be timestamped, so AI can prefer **recent** material when multiple tasks touch similar areas.

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
#### 4.1.1 schema.md – Structure and Semantics

schema.md describes the structural aspects of the data this module owns or actively manipulates:
- Tables and views this module creates or writes to.
- Columns, types, and constraints (PK/FK, uniqueness, check constraints).
- Ownership and responsibility (which module is the primary owner).
- High-level semantics for important fields (e.g., enums, status machines).

This document is the main reference when:
- Designing new features that require schema changes.
- Reviewing the impact of migrations.
- Reasoning about ownership and cross-module coupling.

#### 4.1.2 operations.md – Business Operations → Tables / Fields / Invariants

operations.md maps business-level operations (not individual functions) to the tables and fields they rely on, and the invariants they must preserve.

Recommended structure:
``` markdown
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

- Focus on conceptual operations that are meaningful for humans and AI.
- Do not attempt to track every individual function or method: This mapping is not a call graph; it is an impact map.
- Whenever a new operation is introduced or an existing operation changes semantics:
  - Update operations.md.
  - Consider adding or updating relevant entries in workdocs/outcomes/decisions.md and lessons.md.

#### 4.1.3 When to Update interact/db/

Update schema.md and/or operations.md when:
- A table or column is added, removed, or repurposed.
- A business operation starts touching new tables/fields, or changes its invariants.
- Functions or scripts perform bulk writes or destructive operations.
- Transactions or consistency guarantees are introduced or tightened.

Keeping interact/db/ current enables:
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

Example:

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

Rules:

- Each `module_type` declared in `MANIFEST.yaml` **must** exist here.
- Dependencies between modules must follow the type graph’s allowed edges.
- Updates are rare and must be made when:
  - A new type is introduced.
  - Data flows between types change.
  - Obsolete flows are removed.

Validation and coverage:

- **Scenario → type graph (legality check)**
  For any integration or end-to-end scenario defined in modules/integration/scenarios.yaml:
  - For every interaction between modules (derived from required_interfaces and interfaces.yaml), the caller’s module_type and callee’s module_type must form a valid edge in type_registry.yaml.
  - CI and/or the orchestrator should reject scenarios that introduce interactions not allowed by the type graph.

- **Type graph → scenarios (coverage insight)**
  The project may maintain additional metadata (e.g., in instance_maturity.yaml or a dedicated coverage file) to indicate which type edges are exercised by at least one scenario. This can be used to:
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

Validation and coverage:

- When a scenario is **created or modified**:
  - The orchestrator or CI pipeline must validate that all interactions implied by required_interfaces and involved_modules respect the type graph in modules/overview/type_registry.yaml.
    - For each required interface, look up its module_id and module_type in interfaces.yaml and instance_registry.yaml.
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
