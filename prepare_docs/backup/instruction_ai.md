# AI Strategy Documents Specification (`AGENTS.md`)

> This document describes how **strategy documents** (usually named `AGENTS.md`) are used in this repository template to guide AI‑first development.  
> It is written for both **code models** and **AI task orchestrators**.

---

## 0. Goals and Design Principles

Strategy documents are the **first stop** for an AI working in this repository. They explain:

- What this part of the codebase is responsible for.
- Which rules and guardrails apply.
- How to find the right **knowledge** (`ROUTING.md` + routes/…).
- How to find the right **abilities** (`ABILITY.md` + ability registries).
- How to use **workdocs**, hooks, and other infrastructure.

The design goals are:

1. **Maximize AI development efficiency**

   - Minimize “where do I start?” and “what should I read?” overhead.
   - Keep context small and relevant by providing clear navigation instead of large dumps of documentation.
   - Allow a code model to align quickly with the project’s architecture and expectations.

2. **Make AI behavior predictable and auditable**

   - Encode expectations in plain text, version‑controlled documents.
   - Keep policy and behavior close to the code that it governs.
   - Use a consistent precedence model so humans know **which rule wins** when policies differ.

3. **Integrate with routing and hooks**

   - Strategy documents are **not** another routing system.
   - Instead, they explain how to use the **knowledge routing**, **ability routing**, and **hook** systems that are defined elsewhere.
   - This document aligns with:
     - `modular_mannual.md` – module structure and workdocs.
     - `content_routing.md` – knowledge routing.
     - `ability_routing.md` – ability routing.
     - `hooks.md` – event‑driven hooks.

---

## 1. Strategy Document Layers

Strategy documents live at **three main levels**, following the modular design described in `modular_mannual.md`:

- **Project level** – global behavior for cross‑module work.
- **Module level** – behavior for a single module instance.
- **Subdirectory level** – behavior for specific areas inside a module (code, config, workdocs, docs, etc.).

The same file name is used at all levels: `AGENTS.md`.  
The **nearest file wins**, with project‑level documents acting as a baseline.

### 1.1 Project‑Level Strategy (`modules/AGENTS.md`)

**Location**

- `modules/AGENTS.md`

**Responsibility**

- Describe how the project uses **modules**, **module types**, and **the type graph** (see `modular_mannual.md`).
- Explain how to work on **cross‑module** or **end‑to‑end** tasks.
- Define **global policies** that apply to all modules unless overridden.

**Minimum contents**

A project‑level `AGENTS.md` should:

1. **Summarize the architecture**

   - High‑level description of module types (e.g., domain stages or technical layers).
   - Pointer to:
     - `modules/overview/AGENTS.md` – project‑wide modeling rules.
     - `modules/integration/AGENTS.md` – integration / end‑to‑end workflows.
     - Registries such as `modules/overview/type_registry.yaml`, `instance_registry.yaml`, `interfaces.yaml`, `api_gateway.yaml`.

2. **Explain cross‑instance work**

   - When a task involves multiple modules, describe:
     - How to decide which modules are in scope.
     - How to locate their module‑level `AGENTS.md`.
     - How to use `modules/integration/scenarios.yaml` and `modules/integration/workdocs/…` to manage integration tasks and scenarios.
   - Encourage using **integration scenarios** and `workdocs` plans instead of ad‑hoc cross‑module changes.

3. **Declare global policies**

   - Global constraints that **always apply**, such as:
     - “Do not bypass the type graph when one exists.”
     - “Never write directly to production data from this repository.”
     - “Treat `workdocs` as the source of truth for long‑running AI tasks.”
   - Any global safety and review rules (who must review, when to ask for a human, etc.).

4. **Route to core specifications**

   - Explicitly link to:
     - `modular_mannual.md`
     - `content_routing.md`
     - `ability_routing.md`
     - `hooks.md`
   - Briefly state what each document is for and when an AI should open it.

### 1.2 Module‑Level Strategy (`modules/<module_id>/AGENTS.md`)

**Location**

- `modules/<module_id>/AGENTS.md` for each module instance.

**Responsibility**

- Define the **responsibility and boundaries** of the module.
- Explain how to work effectively **inside this module only**.
- Describe how this module participates in cross‑module flows (but delegate global rules to project‑level docs).

**Minimum contents**

A module‑level `AGENTS.md` should include:

1. **Module summary and boundaries**

   - Plain‑language description of what the module owns and does **not** own.
   - Key inputs & outputs (APIs, events, data tables, files).
   - Relationship to module type and level (with a pointer to the relevant entries in `modules/overview/instance_registry.yaml`).

2. **Local directory map**

   - Short explanation of notable directories:
     - `src/…` – main code, with pointers to any subdirectory `AGENTS.md` (e.g. `src/backend/AGENTS.md`).
     - `workdocs/` – task‑level planning and context (with `workdocs/AGENTS.md`).
     - `docs/` – human‑oriented documentation.
     - `config/` – configuration (with `config/AGENTS.md`).
     - `interact/` – public contracts (HTTP, events, DB).
   - Encourage the AI to **follow these pointers** instead of scanning the whole tree.

3. **AI workflow for this module**

   Describe how an AI should work through the **Understand → Plan → Act → Review** cycle (same stages as in `content_routing.md`):

   - **Understand**
     - Read this `AGENTS.md`.
     - Load relevant knowledge via this module’s `ROUTING.md` and topic routes.
     - Inspect `interact/` contracts if the task touches APIs, events, or data.
   - **Plan**
     - If the task is non‑trivial or may span sessions, create or update a `workdocs/active/<task_id>/plan.md`.
     - Use `tasks.md` as a checklist.
   - **Act**
     - Edit code and configuration under `src/…` and `config/…`.
     - Use abilities from this module’s `ABILITY.md` for running scripts or tools.
   - **Review**
     - Run tests and diagnostics using abilities or scripts defined for this module.
     - Update `workdocs/active/<task_id>/context.md` and `tasks.md` to reflect what was done.
     - If the task is finished, move any summaries or reusable scripts into `workdocs/outcomes/…`.

4. **Local policies and guardrails**

   - Coding conventions specific to this module (naming, patterns, performance rules).
   - Local safety rules (e.g., no direct DB writes in certain directories, or required tests for risky changes).
   - Expectations for integration with hooks (e.g., “build/test hooks must pass before requesting human review”).

5. **Pointers to routing and abilities**

   - Where to find:
     - `ROUTING.md` and important topic route files.
     - `ABILITY.md` for this module.
   - Any module‑local conventions for knowledge and ability naming.

### 1.3 Subdirectory‑Level Strategy (e.g., `src/backend/AGENTS.md`)

**Location examples**

- `modules/<module_id>/src/backend/AGENTS.md`
- `modules/<module_id>/docs/AGENTS.md`
- `modules/<module_id>/config/AGENTS.md`
- `modules/<module_id>/workdocs/AGENTS.md`

**Responsibility**

- Specialize the module‑level strategy for one **area of work**.
- Reduce cognitive load by documenting local patterns and rules **next to the resources they govern**.

**Patterns**

1. **Source code (`src/…/AGENTS.md`)**

   - Explain:
     - How code in this area is organized (layers, boundaries).
     - Allowed dependencies (e.g., “controllers may call services, but not repositories directly”).
     - Testing expectations (unit vs integration tests).
   - Provide a **small number of canonical examples**: “good diff vs bad diff”.

2. **Configuration (`config/AGENTS.md`)**

   - Describe:
     - Where configuration files are.
     - How configuration is loaded at runtime.
     - Separation between environments (`dev`, `staging`, `prod`).
     - How secrets are handled (what **must not** be committed).
   - Provide instructions for AI on:
     - When to introduce new configuration keys.
     - How to document configuration changes and link them back to `workdocs` or tasks.

3. **Workdocs (`workdocs/AGENTS.md`)**

   - Explain how to:
     - Create and name `task_id` folders under `workdocs/active/`.
     - Maintain `plan.md`, `context.md`, and `tasks.md`.
     - Move finished work into `workdocs/outcomes/…` and archives.
   - Clarify:
     - What counts as a “long‑running” task.
     - When to write to `decisions.md` or `lessons.md` instead of just task‑local files.

4. **Docs (`docs/AGENTS.md`)**

   - Explain which documents are meant for **humans**, which for **AI**, and which for both.
   - Define how documentation is **routed**:
     - How this directory’s content is referenced from `ROUTING.md` topics.
     - How new docs should be registered in the routing system.

---

## 2. Precedence and Conflict Resolution

Strategy documents follow a **nearest‑wins with upward override** principle:

1. **Project‑level (`modules/AGENTS.md`)**

   - Defines global rules and guarantees.
   - Acts as the fallback whenever lower levels are silent.

2. **Module‑level (`modules/<module_id>/AGENTS.md`)**

   - Specializes global rules for one module.
   - May **tighten** global policies (e.g., “this module requires additional tests”) but should not silently relax hard global constraints (e.g., “no production writes”).

3. **Subdirectory‑level (`…/AGENTS.md`)**

   - Specializes module rules for a narrow area.
   - Focuses on local conventions and patterns.

**For code models**

When you detect conflicting rules:

1. Prefer the **most local** `AGENTS.md` that explicitly speaks about the situation.
2. If a local rule **appears** to contradict a hard global rule, treat it as **invalid** and:
   - Follow the global rule.
   - Note the conflict in the relevant `workdocs` context for human review.

**For orchestrators**

- When assembling context:
  - Always load the **closest** `AGENTS.md` to the target path.
  - Optionally also load the module‑level and project‑level `AGENTS.md` for clarity on global rules.
- Make it explicit in prompts which rules come from which level (e.g., “global”, “module”, “local”).

---

## 3. How AI Should Use Strategy Documents

### 3.1 Code Model Responsibilities

When a code model is assigned a task:

1. **Locate the relevant strategy document**

   - Determine the module (`module_id`) and the main directory affected.
   - Load the nearest `AGENTS.md` in that directory.
   - If the task is cross‑module, start from `modules/AGENTS.md` and follow its guidance.

2. **Align on boundaries and goals**

   - Use `AGENTS.md` to:
     - Understand the boundaries of your responsibility.
     - Confirm where **not** to make changes.
     - Identify which modules or directories you need to collaborate with.

3. **Plan using the four orchestration stages**

   - Map your workflow to `understand` → `plan` → `act` → `review` as defined in `content_routing.md`.
   - Ensure `workdocs` are created or updated for anything non‑trivial or cross‑session.

4. **Route to knowledge and abilities**

   - Follow links in `AGENTS.md` to:
     - `ROUTING.md` and topic routes.
     - `ABILITY.md` and ability registries.
   - Prefer **high‑level abilities** where possible; fall back to low‑level operations only when necessary (per `ability_routing.md`).

5. **Respect hooks and guardrails**

   - Strategy documents may mention required hooks (e.g., “build checks must pass on SessionStop”).
   - Do not bypass hooks or validation scripts.
   - Interpret hook feedback (logs, warnings) as part of your `review` phase.

6. **Document decisions and lessons**

   - When a decision has cross‑task impact, log it in:
     - `workdocs/outcomes/decisions.md`
     - or another location specified by `AGENTS.md`.
   - For mistakes or non‑obvious pitfalls, update `lessons.md` so future sessions learn from them.

### 3.2 Orchestrator Responsibilities

An AI orchestrator (scheduler, multi‑agent controller, or tool runner) should:

1. **Start from strategy, not raw code**

   - On new tasks, locate and load the nearest applicable `AGENTS.md` before expanding context.
   - Use contents of `AGENTS.md` to:
     - Decide which routing topics to query.
     - Decide which abilities are allowed or preferred.
     - Decide whether `workdocs` need to be created.

2. **Respect module boundaries**

   - Avoid mixing code from multiple modules in a single context unless the project‑level `AGENTS.md` explicitly describes how to do cross‑instance work.
   - For integration tasks:
     - Route to `modules/integration/AGENTS.md`.
     - Use `modules/integration/scenarios.yaml` and `workdocs/…` to coordinate.

3. **Integrate with hooks**

   - Use registered hooks (described in `hooks.md`) to:
     - Enrich prompts with lightweight signals (e.g., relevant abilities, recent logs).
     - Enforce pre‑ and post‑conditions (e.g., run tests after certain changes).
   - Treat hook outputs as part of the task context and route them back into `workdocs` where appropriate.

4. **Maintain context continuity**

   - When a task spans multiple sessions:
     - Re‑open relevant `workdocs`.
     - Reload the same `AGENTS.md` documents used previously.
   - Record which strategy documents were applied to each task, so future sessions can reconstruct behavior.

---

## 4. Recommended Structure for `AGENTS.md`

To keep strategy documents consistent and easy to parse, the following structure is recommended at **all levels** (project, module, subdirectory):

1. **Header**

   - Title (including scope, e.g., “Module Strategy – `payments.api`”).
   - Scope (project / module / path).
   - Intended audience (code model, orchestrator, humans).

2. **Responsibilities and Boundaries**

   - What this scope owns.
   - What it explicitly does **not** own.

3. **Quickstart for AI**

   - A short step‑by‑step recipe for the **first run** in this scope:
     - which docs to open,
     - which routes to use,
     - which abilities are most relevant.

4. **Workflow and Stages**

   - Mapping of `understand` / `plan` / `act` / `review` to this scope.
   - Any scope‑specific checklists.

5. **Routing and Abilities**

   - Links to:
     - `ROUTING.md` topics and key knowledge docs.
     - `ABILITY.md` and frequently used abilities.
   - Any local conventions for names or tags.

6. **Policies and Guardrails**

   - Safety and security rules.
   - Code quality expectations (tests, review, style).
   - Restrictions on abilities (e.g., “no production environment for this module”).

7. **Workdocs and Logging**

   - How to use `workdocs` in this scope.
   - When and where to log decisions and lessons.

8. **Change and Maintenance Notes**

   - Who conceptually “owns” this `AGENTS.md` (team, role).
   - When it was last substantively updated.
   - Any TODOs for future improvement.

---

## 5. Strategy Index and Consistency Checks

Strategy documents rarely contain all details themselves. Most concretions live in **separate knowledge docs**, scripts, and registries. To keep everything in sync:

1. **Strategy index**

   - Maintain a small, machine‑readable index file (for example, `.system/strategy_index.yaml`) that lists:
     - Each `AGENTS.md` path.
     - Its scope (`project`, `module`, or `path`).
     - Key linked routing files (`ROUTING.md`, topic routes).
     - Key linked ability registries (`ABILITY.md`, registry paths).
     - Last reviewed timestamp.

2. **Automated checks**

   - Provide a script (e.g., `scripts/strategy/check_consistency.py`) that can:
     - Verify that every module has a corresponding `AGENTS.md`.
     - Check that all paths referenced in `AGENTS.md` exist (knowledge docs, ability registries).
     - Optionally cross‑check with knowledge routing and ability routing:
       - Ensure important docs appear in both `AGENTS.md` and routing files.
       - Detect dead links and unused routes.

3. **CI and hooks**

   - Integrate these checks into CI and hooks:
     - Run a lightweight strategy check on pull requests that touch `AGENTS.md`, routing, or registries.
     - Optionally run a broader consistency check as a scheduled job.
   - Use hooks (see `hooks.md`) to trigger:
     - A warning when strategy documents are changed without related routing/registry updates.
     - A reminder to update `workdocs/outcomes/decisions.md` for major behavior shifts.

4. **Human review**

   - Treat modifications to project‑level and module‑level `AGENTS.md` as **high‑impact** changes.
   - Prefer explicit human review for:
     - New cross‑module behaviors.
     - Relaxation of safety constraints.
     - Large changes in recommended workflows.

With this structure, strategy documents stay:

- **Readable for humans**
- **Navigable for AI**
- **Aligned with routing, abilities, and hooks**
- And most importantly: focused on **making AI development efficient, predictable, and recoverable** across sessions and agents.
