# Ability Routing

## 0. Purpose and Design Goals

Ability routing is the infrastructure that makes it easy for an AI-first project to reuse existing tools instead of “reinventing the wheel”, while keeping the code base transparent, consistent, and controllable.

In this template:

- An **ability** is a packaged piece of functionality that can be invoked by an AI or a human.
- The project maintains a **single ability pool** and a **standard execution script** so that the AI can discover, validate, and call abilities in a uniform way.
- Abilities are split into **low-level** and **high-level** layers to keep orchestration manageable and to align with how a code model plans and executes work.

The core design goals are:

1. **Maximize AI development efficiency**: the AI should be able to quickly discover and apply the right abilities for a given task, without reading large amounts of implementation code.
2. **Keep orchestration manageable**: the orchestration logic should remain at an abstract level while low-level details are encapsulated behind well-defined abilities.
3. **Support continuous evolution**: as the project grows and the AI generates new scripts and flows, the ability routing system should absorb these outputs and evolve with real usage.
4. **Maintain control and safety**: all calls go through a common execution script with guardrails, logging, and configuration, so that behavior is auditable and adjustable.

---

## 1. Concepts and Rules

### 1.1 Routing Documents (`ABILITY.md`)

A routing document is the human- and AI-readable entry point into the ability system. Every routing document is named `ABILITY.md`.

Routing documents serve two purposes:

1. **Discovery** – help the AI quickly understand which abilities exist in the current working domain.
2. **Navigation** – provide stable paths to the underlying ability registries, without exposing low-level implementation details.

The template follows a modular development model. Code is organized into **module instances**. Each module instance defines its own working boundary. For each module instance, there is a dedicated routing document: `<module-root>/ABILITY.md`.

In addition, the project maintains a **top-level integration routing document** for end-to-end or cross-module scenarios, typically under an integration or system directory, for example: `integration/ABILITY.md`.

In normal development, the AI’s working scope is well-defined, so one orchestration run should only need to read **one** `ABILITY.md`:

- Module-level `ABILITY.md` for feature development and local testing.
- Integration-level `ABILITY.md` for end-to-end or cross-module testing.

Routing documents evolve together with the project. When the AI’s orchestration hits gaps (for example, no ability exists for a common operation), the execution script and CI tools capture that usage and feed it back to improve `ABILITY.md` over time.

#### High-level vs. Low-level Abilities in `ABILITY.md`

Both high-level and low-level abilities live in the **same ability pool** but are presented differently in routing documents:

- **High-level abilities** are preferred and shown first. They represent workflows or agents that cover meaningful chunks of work (e.g. “run end-to-end signup test”).
- **Low-level abilities** represent atomic operations (e.g. “insert row into user table”) and are listed as fallbacks when no suitable high-level ability exists.

The AI should:

1. Scan high-level abilities first, matching **intent and scenario**.
2. Only fall back to low-level abilities when high-level ones cannot cover all orchestration steps.

#### Example: High-level Ability Routing Structure

A typical `ABILITY.md` may describe high-level abilities in a table form:

```markdown
## High-level abilities

| Id                    | Type     | Scope                | When to use                                             | Registry path                                  |
| --------------------- | -------- | -------------------- | ------------------------------------------------------- | ---------------------------------------------- |
| signup_e2e_test       | workflow | integration/test     | Run full user signup E2E test against non-prod env     | .system/registry/high-level/signup_e2e.yaml    |
| billing_regression    | workflow | integration/billing  | Run billing regression suite with seeded test data     | .system/registry/high-level/billing_reg.yaml   |
| code_review_agent     | agent    | dev/code-review      | Review PRs for style, safety, and interface changes    | .system/registry/high-level/code_review.yaml   |
```

Key fields:

- `Id`: unique ability identifier used by the execution script.
- `Type`: `workflow` or `agent`.
- `Scope`: domain or module where this ability is valid.
- `When to use`: natural-language intent description to help semantic matching.
- `Registry path`: path to the YAML registry that defines the ability.

#### Example: Low-level Ability Routing Structure

Low-level abilities are usually grouped by operation type and technology:

```markdown
## Low-level abilities (atomic operations)

| Operation key                 | Kind    | Scope             | Summary                                           | Registry path                                       |
| ---------------------------- | ------- | ----------------- | ------------------------------------------------ | --------------------------------------------------- |
| db.write.user_row            | script  | infra/db          | Write a single user row into the users table     | .system/registry/low-level/db.write.user_row.yaml   |
| http.call.billing_service    | api     | infra/http        | Call billing service HTTP endpoints              | .system/registry/low-level/http.call.billing.yaml   |
| mcp.query.logs               | mcp     | infra/observability | Query logs via MCP log server                  | .system/registry/low-level/mcp.query.logs.yaml      |
```

The AI primarily matches:

- **High-level abilities** by *intent* and *scope*.
- **Low-level abilities** by *operation key* and *summary*.

Routing documents should stay compact and focused on what is actually useful in that working domain. They do not need to list every ability in the global pool, only those that are relevant to that module or integration scenario.

---

### 1.2 Ability Pool

The project maintains a centralized **ability pool** – a set of all registered abilities that are allowed to be called. The ability pool is the single source of truth for:

- What the ability is supposed to do.
- How to call it (through the common execution script).
- Where the implementations live.
- Which configuration and guardrails apply.

We distinguish between **low-level abilities** and **high-level abilities**.

#### 1.2.1 Low-level abilities

Low-level abilities are **atomic operations**, usually tied to concrete implementations:

- Repository scripts
- External services (through MCP)
- External APIs or cloud providers
- Database operations, queue operations, etc.

Low-level abilities are classified by *implementation kind*:

- `script` – project-internal scripts or binaries.
- `mcp` – MCP (Model Context Protocol) tools exposed by external or internal MCP servers.
- `api` – HTTP/RPC API calls to external services or cloud providers.

For each kind, the project maintains **operation-level** and **implementation-level** metadata.

##### **script abilities**

Scripts represent local, project-managed implementations (e.g. Bash, Python, Node).

Maintenance example (conceptual):

- Operation key: `db.write.user_row`
- Implementation: `script`
- Implementation data:
  - Script path: `scripts/db/write_user_row.sh`
  - Invocation style: `./scripts/db/write_user_row.sh <json-payload>`
  - Required environment variables: `DB_HOST`, `DB_USER`, `DB_NAME`
  - Side effects: writes to the `users` table in the non-production database

##### **mcp abilities**

MCP abilities wrap external MCP servers, making them callable within the same execution framework.

Maintenance example (conceptual):

- Operation key: `mcp.query.logs`
- Implementation: `mcp`
- Implementation data:
  - MCP server id: `observability-logs`
  - MCP tool: `search_logs`
  - Typical filters: service name, time window, severity
  - Constraints: available only in certain environments

##### **api abilities**

API abilities wrap external or cloud APIs.

Maintenance example (conceptual):

- Operation key: `http.call.billing_service`
- Implementation: `api`
- Implementation data:
  - Base URL: `https://billing.dev.internal/api`
  - Authentication: `BILLING_API_TOKEN` or service account
  - Endpoints: `/charge`, `/refund`, `/invoice`
  - Rate limits: per minute and per day

In all cases, the AI should not call these implementations directly. Instead, it calls the **standard execution script**, which loads configuration, applies guardrails, and executes the correct implementation for the selected operation key.

#### 1.2.2 High-level abilities

High-level abilities are semantic capabilities composed from low-level abilities. They represent the tasks that the orchestrator or AI planner should prefer.

High-level abilities come in two types:

- `workflow` – relatively deterministic pipelines, usually sequential or with limited branching (e.g. “run build + unit tests + integration tests”).
- `agent` – AI-driven flows that can dynamically choose which low-level abilities to call, often requiring reasoning loops and flexible decision making (e.g. “debug failing test given logs and codebase”).

##### **workflow abilities**

Example (conceptual):

- Id: `signup_e2e_test`
- Type: `workflow`
- Description: run the full user-signup flow against the staging environment, including DB seeding and cleanup.
- Steps:
  1. Call `db.write.user_row` to seed a test user.
  2. Call `http.call.signup_service` with test data.
  3. Call `mcp.query.logs` to verify no errors were emitted.
  4. Call `db.read.user_row` to verify persisted state.

##### **agent abilities**

Example (conceptual):

- Id: `code_review_agent`
- Type: `agent`
- Description: review a pull request, focusing on safety, style, and public interfaces.
- Behavior:
  - Reads PR diff and context.
  - Calls `mcp.query.logs` or other diagnostic abilities as needed.
  - Generates structured review comments.
  - May ask for additional input if required (e.g. expected behavior).
- Callable:
  - `mcp.query.logs`
  - `api.query.logs`
  - ...

In both cases, high-level abilities are executed via the same **standard execution script**, not by the AI calling low-level tools directly.

---

### 1.3 Ability Registries

Routing documents (`ABILITY.md`) **do not** contain full technical specifications. Instead, they point to **ability registries**: YAML documents that define:

- Input and output contracts.
- Scope and usage conditions.
- Constraints and guardrails.
- Mapping to implementations and configuration.

Registries are the **single source of truth** (SSOT) for ability semantics. The AI uses routing documents to **find** abilities and registry entries to **understand and safely call** them.

There are separate registry formats for low-level and high-level abilities.

#### 1.3.1 Low-level Ability Registry

A low-level ability corresponds to an **atomic operation type**, not a specific implementation. For example, `db.write.user_row` is an operation; it may have both a `script` implementation and a `mcp` implementation.

The main question for the AI at orchestration time is:

> “Can this operation type be inserted into my execution chain?”

The execution script will then choose a concrete implementation based on configuration and environment.

##### Primary registry contents (operation-level)

Each low-level registry entry describes an **operation key**. Typical fields:

- **Registration info**
  - `operation_key` (primary key, e.g. `db.write.user_row`)
  - `version` (optional semantic version of the contract)
- **Operation description**
  - Natural-language summary
  - Intended use cases
  - Idempotency / side-effect notes
- **Scope**
  - Module(s) or domain(s) where this operation is valid
  - Allowed environments (e.g. `dev`, `staging`, but not `prod`)
- **Input**
  - Structured description or schema (JSON-schema-like; at minimum: required fields, types, and constraints)
- **Output**
  - Structured description or schema (success result, error shape, partial results)
- **Linked implementations** (no reading requried for AI)
  - Implementation ids and documentation paths (for humans and tooling)

Example of operation-level metadata (conceptual):

- Operation key: `db.write.user_row`
- Scope: `auth-service`, `integration/tests`
- Input:
  - `user_id` (string)
  - `email` (string, validated format)
  - `created_at` (ISO timestamp)
- Output:
  - `status` (`success` | `error`)
  - `error_code` (optional)
- Guardrail:
  - Forbid `prod` environment unless explicitly whitelisted.
  - Ensure emails look synthetic in staging/non-prod.

##### Implementation documents

Implementation documents describe **how** an operation is implemented. This is mostly for humans and automation, not required reading for the AI.

Typical fields:

- Registration info
  - Implementation id
  - Implementation kind: `script` | `mcp` | `api`
- Operation linkage
  - Operation key this implementation belongs to
- Invocation & behavior
  - Invocation method (script path, MCP server/tool id, API endpoint, etc.)
  - Constraints and limitations
  - Default parameters and override rules
- Implementation address
  - Script path, MCP server name & tool id, API base URL/endpoint
- Tags
  - E.g. `fast`, `experimental`, `deprecated`, `high-cost`

##### Configuration documents

Configuration documents store environment-specific settings and behavior. They are referenced by both operations and implementations.

For each operation key, the typical fields:
- Implementation key
  - Current default implementation id
  - List of allowed implementation ids
- Current Parameters (per environment or context)
  - `mcp_server`, `mcp_tool`
  - `endpoints` / base URLs
  - `ssh_key` / credentials references
  - Prompt templates or hints for AI
- Alternative Parameters
  - ...
- Routing
  - Referenced `ABILITY.md` paths
- Hooks
  - `matcher` configuration
  - `guardrail` configuration (e.g. rate limits, environment checks)

The AI interacts with configuration only indirectly via the execution script. The script is responsible for interpreting configuration and selecting the correct implementation.

#### 1.3.2 High-level Ability Registry

A high-level ability corresponds to an **abstract functional capability** rather than a single operation. It combines multiple low-level operations to achieve a larger goal. High-level abilities typically orchestrate operations according to inputs, constraints, and configuration.

Unlike low-level abilities, the primary key is the **ability id**, not an operation key. One registry entry maps to one orchestrated capability.

The AI must answer:

> “Can I apply this ability to cover part of my development plan?”

Again, the AI calls the standard execution script with the ability id; the script then executes the orchestrated flow.

##### Primary registry contents

Typical fields for high-level ability registry entries:

- **Registration info**
  - `id`: unique ability id (e.g. `signup_e2e_test`)
  - `type`: `workflow` | `agent`
  - `version`: optional semantic version
- **Intent and scenarios**
  - Natural-language description of what the ability does
  - Example user stories or scenarios
- **Scope**
  - Modules, domains, or integration boundaries where this ability applies
- **Input**
  - Expected input fields and schemas
  - Optional vs required parameters
  - Example payloads
- **Output**
  - Expected outputs, including structured status and artifacts
  - Typical error states
- **Execution behavior**
  - High-level description of orchestration (sequence or decision logic)
  - Constraints (runtime, cost, environment)
  - Callable low-level operation keys (whitelist of operations the ability may call)

##### Configuration documents

Each high-level abilities also has configuration documents. Example fields:

- Ability id
  - Allowed low-level operation keys (operation-level primary keys)
  - Default and optional configurations for each operation (e.g. which implementation to use)
- Current Parameters
  - `mcp_server`, `mcp_tool`, `endpoints`, `ssh_key`, prompts, timeouts, etc.
- Alternative Parameters
  - `mcp_server`, `mcp_tool`, `endpoints`, `ssh_key`, prompts, timeouts, etc.
- Runtime metadata
  - Execution mode (batch / interactive)
  - Trigger conditions (for automated or scheduled runs)
- Routing metadata
  - Implementation path (code or workflow definition)
  - Referenced `ABILITY.md` paths
  - Tags (e.g. `critical`, `unsafe-in-prod`, `slow`)
- Hooks
  - `matcher`: matching rules to surface this ability to the AI
  - `guardrail`: guardrails that wrap the entire high-level flow

---

## 2. Usage and Typical Flows

High-level and low-level abilities form a single, unified tool pool. Ability routing ensures that once the AI has a task plan, it can map each step to one or more concrete abilities and execute them through the common script.

### 2.1 Task Orchestration Perspective

From the AI orchestrator’s perspective, the typical process is:

1. **Identify the correct `ABILITY.md`**
   - Based on the current working scope (module or integration).
   - Read `ABILITY.md` to inventory available high-level and low-level abilities.
2. **Scan high-level abilities**
   1. Match ability intents and scopes against the orchestration plan.
   2. For each candidate, follow the `Registry path` and read the high-level registry to check:
      - Input/output compatibility with the plan.
      - Environmental constraints.
   3. Call the execution script to create a “use ability” task with the planned input.
   4. If the ability is available, the script returns a task key; record this in the orchestration graph.
   5. Repeat until all high-level candidates are evaluated.
3. **Fill remaining gaps with low-level abilities**
   - For plan steps not covered by high-level abilities, scan low-level abilities in `ABILITY.md`.
   - Match operation keys and summaries to the remaining functional gaps.
   - For each candidate, read the low-level registry entry and create a task via the execution script.
4. **Finalize the execution chain**
   - Construct a full execution graph from the selected abilities and task keys.
   - Ensure dependencies and ordering are consistent with the plan.
5. **Execute according to the plan**
   - Invoke the execution script using task keys at the right time.
   - React to success/failure outputs and adjust the orchestration if needed.

#### Key points and example

Key considerations for the AI planner:

- Prefer **high-level abilities** when they can cover a complete sub-task end-to-end.
- Use **low-level abilities** to fill specific gaps or to handle advanced cases.
- Use **registry input/output schemas** as the reference for payload construction.

Example scenario:

- Task: “Run signup E2E regression for the latest build”.
- Plan:
  - Try to use `signup_e2e_test` high-level workflow.
  - If not available, assemble low-level operations:
    - `db.write.user_row` to seed data.
    - `http.call.signup_service` for API calls.
    - `mcp.query.logs` to validate no errors.
- Execution:
  - For each ability, create a task via the execution script.
  - Use the returned keys to build the execution chain.

### 2.2 Ability Routing Perspective

From the ability routing perspective, once the AI knows what it wants to do, the flow looks like:

1. The AI reads the relevant `ABILITY.md` and selects a set of candidate registry paths.
2. The AI reads the corresponding registry entries to decide which abilities to use.
3. For each selected ability, the AI calls the execution script to **create a task** and receives a task key.
4. The AI builds its orchestration plan around those task keys.
5. When it is time to execute, the AI calls the execution script with the task key.
6. The execution script runs the underlying implementation(s) and returns outputs in the registered format; the AI continues the workflow.

#### Key points and example

- `ABILITY.md` is an **index**, not a detailed spec.
- Registries are **specs**, not execution engines.
- The execution script is the **only** entry point for actually running abilities.

Example:

- Ability chosen: `code_review_agent` (from `ABILITY.md`).
- Registry says:
  - Input: `pr_url`, `focus_areas`.
  - Output: `review_comments` (structured).
- AI calls:
  - `ability task.create` with id `code_review_agent` and input (PR URL and focus areas).
  - Receives task key `task-123`.
  - Later calls `ability task.run` with key `task-123` to execute.

### 2.3 Standard Execution Script

The template provides a **single, global execution script** (or entry point) for abilities. The AI only needs to interact with this script, not with individual scripts, MCP tools, or APIs directly.

The execution script must:

- Provide a **uniform interface** for:
  - Creating tasks.
  - Executing tasks.
  - Querying and updating task payloads.
  - Deleting tasks when finished.
- Enforce **guardrails** and validate **preconditions** before execution.
- Apply **configuration and default parameters** automatically.
- **Record usage** and outcomes to help improve routing documents and registries.
- Promote **simplicity** for the AI: the AI can think in terms of ability ids and operation keys, not runtime details.

#### Interfaces

The execution script exposes at least the following conceptual interfaces:

1. **Create task** *(does not execute)*
   - Input:
     - Ability id or operation key.
     - Structured input payload according to the registry.
   - Behavior:
     - Validates input against registry.
     - Checks guardrails (environment, rate limits, manual approval, etc.).
     - If not executable, returns “unavailable” with reason.
     - If executable, allocates a unique task key and:
       - Creates a folder: `/.system/implement/<key>/`.
       - Creates a record file: `/.system/implement/<key>/tool_call.md`.
       - Caches input and resolved configuration.
   - Output:
     - Task key (e.g. `task-123`).
     - Availability status and any warnings.

2. **Run task** *(executes using cached data)*
   - Input:
     - Task key.
   - Behavior:
     - Loads cached input and configuration.
     - Finds the relevant implementation(s).
     - Executes the underlying tool(s), potentially including nested high-level abilities.
     - Records execution details and results in `tool_call.md`.
   - Output:
     - Standardized result structure from the registry (success/error/payload).

3. **Direct run (no task)** *(execute once without persisting a task)*
   - Input:
     - Ability id or operation key.
     - Input payload.
   - Behavior:
     - Validate and execute directly.
     - Does not create a task folder or usage record (unless configured to do so).
   - Output:
     - Execution result.

4. **Query/modify task payload** *(optional but recommended)*
   - Query by task key to inspect current input.
   - Modify one or more input fields by key.
   - This is useful when the AI or a human wants to adjust parameters before execution.

5. **Delete task**
   - Tasks have a lifecycle:
     - Removed after successful execution.
     - Removed after timeout (e.g. 60 minutes).
     - Removed when cache is cleared.
   - Deleting a task should also clean up related cached data.

#### Availability checks and guardrails

The execution script enforces a common set of checks before an ability is accepted or executed:

- Validate input structure and constraints using registry schemas.
- Evaluate guardrail rules:
  - Environment restrictions (e.g. “no prod writes”).
  - Manual approval requirements.
  - Rate limits and quotas.
- Determine whether interactive input is required:
  - If required, clearly signal whether it needs human input or AI-generated input.

Feedback to the AI includes:

- For unavailable abilities:
  - Clear reason (e.g. “restricted in prod”, “missing configuration”, “requires manual approval”).
- For available abilities:
  - Confirmation and task key.

#### Logging and feedback

For each **task-based** invocation (where a task key is created), the script should record in `tool_call.md`:

- When the task was created.
- The reason or intent for the call (the AI should provide a short explanation).
- The ability and implementation actually executed.
- Execution results and errors (if any).
- Whether the output matched expectations (the AI should provide feedback).

This information is later used to:

- Improve `ABILITY.md` content.
- Refine matcher and guardrail rules.
- Discover new candidate abilities from AI-generated scripts.

#### Handling high-level abilities

High-level abilities introduce additional complexity:

- They often call multiple low-level abilities.
- They may call abilities outside the current project (e.g. generic agents, MCP servers).
- They may require interactive input (e.g. human approval, clarification questions).

Recommended handling:

1. **Nested execution**
   - Allow the execution script to **invoke itself** recursively for nested abilities.
   - Record parent-child relationships between task keys for traceability.
2. **Shared primitive operations**
   - Ensure the execution script has robust base support for:
     - MCP calls.
     - API calls.
     - Local script execution.
   - High-level abilities should only orchestrate; they should not reimplement these primitives.
3. **Interactive input**
   - Define clear markers in the registry for input fields that require human input.
   - The execution script should:
     - Pause execution and request input via a documented channel.
     - Or allow AI-supplied input when marked as safe and automatic.

#### Invocation conventions and examples

For the AI, the core conventions are:

- Always check `ABILITY.md` to find the relevant id and registry path.
- Always respect the registry input/output definitions.
- Prefer creating tasks (with keys) when:
  - The ability is part of a larger workflow.
  - You need history and traceability.
- Use direct run only when:
  - You need a one-off call.
  - Traceability is not important, and the operation is low-risk.

Example workflow for a high-level ability:

1. Read `ABILITY.md`, find `signup_e2e_test`.
2. Read registry to understand required input.
3. Call `ability task.create` with the ability id and input.
4. If available, receive task key `task-123`.
5. Later, call `ability task.run` with `task-123` to execute.

### 2.4 Directional Invocation

Directional invocation means explicitly selecting and calling an ability, either manually or by giving the AI a precise instruction to do so.

There are three main patterns:

1. **Task-based, via execution script**
   - Follow the standard flow: create task → run task using a key.
   - This is the recommended path for most usage, as it records intent and results.

2. **Direct scripted execution**
   - The user (human or AI) reads the registry and calls the execution script directly with:
     - Ability id or operation key.
     - Input payload according to the registry.
   - This bypasses task creation and logging; use for low-risk utilities or debugging.

3. **Raw tool invocation (not recommended)**
   - Call underlying scripts, MCP tools, or APIs directly, without going through the ability system.
   - This produces no ability-level logging or context and should be reserved for exceptional scenarios only (e.g. emergency debugging).

Example directional usage:

- A human developer wants to quickly run a one-off billing regression:
  - They identify `billing_regression` in `integration/ABILITY.md`.
  - They invoke the execution script directly with the ability id and minimal input.
- An AI agent that has a durable plan and needs traceability:
  - Uses the task-based interface for every ability that affects shared state.

---

## 3. Registration and Maintenance

### 3.1 Ability Registration

Apart from template-provided implementations, abilities enter the system through:

1. **Continuous integration of development outputs** (AI or humans add new scripts or flows).
2. **Manual registration** using template-provided scaffolding tools.

Every ability, regardless of level or type, must be registered through a common registration process. Registration ensures:

- The registry entries follow the schemas.
- References to implementations and configurations are valid.
- `ABILITY.md` documents are updated consistently.

#### Atomic operation registration

1. Define a new operation key (e.g. `queue.publish.email_event`).
2. Create a low-level operation registry entry with:
   - Summary, schema, constraints, and scope.
3. Optionally add one or more implementations (scripts, MCP, or APIs).

#### Low-level routing registration

When a new implementation is added for an existing operation:

1. Create or update the implementation document.
2. Update configuration to:
   - Mark it as the default or as an optional implementation.
   - Restrict environments if necessary.
3. Update relevant `ABILITY.md` documents to include the operation (if applicable for that module or integration).

#### High-level routing registration

When a new workflow or agent is created:

1. Define a unique ability id and type (`workflow` or `agent`).
2. Create a high-level registry entry:
   - Intent, scope, input/output, and orchestration description.
   - Allowed low-level operation keys.
3. Add configuration for environments and prompts.
4. Add the ability to the relevant `ABILITY.md` documents with a concise usage description.

#### Registry and implementation directories

The template recommends a consistent directory structure such as:

```text
.system/
  registry/
    low-level/        # Low-level operation registries
    high-level/       # High-level ability registries
    config/           # Configuration files (per ability/operation)
  implement/
    <task-key>/       # Cached tasks and tool_call.md
```

The actual paths may vary, but they must remain stable and be referenced consistently in `ABILITY.md` and registry entries.

Template-provided CLI/scaffolding commands may include (conceptual names):

- `ability register operation` – create a new low-level operation registry.
- `ability register implementation` – register a concrete script/MCP/API implementation.
- `ability register ability` – register a new high-level workflow or agent.
- `ability lint` – validate registry schemas and references.
- `ability sync-ability-docs` – synchronize `ABILITY.md` from current registries.

### 3.2 Routing Maintenance

Day-to-day development uses:

- Module-level `ABILITY.md` for feature work.
- Integration-level `ABILITY.md` for end-to-end and integration testing.

These two entry points are the primary maintenance targets.

#### 3.2.1 Sustainable Evolution

As described in section 2.1, the project provides a set of pre-packaged abilities for the AI to use. However, in real development, these abilities will **not** cover all needs. The AI (or humans) will often:

- Write new scripts.
- Compose new flows.
- Apply tools in new ways.

The template aims to **harvest** these outputs and incorporate them into the ability system.

For each significant new function implemented during orchestration, we want to capture:

- The **intent**: what the function is supposed to do.
- Whether the AI used existing abilities or created new code:
  - Which abilities were called (and how).
  - Which new scripts were created (paths and names).
- Where new scripts are stored (the template should require new scripts to be saved under a designated directory and documented).

A separate document (e.g. `modular_manual.md`) will define the exact paths and required comments for these new scripts.

Using this information, we can derive:

- **Actual functional outputs** (new scripts and flows)
  - Compare their described purpose with existing abilities.
  - Decide whether to absorb them into the ability pool as new low-level operations.
- **Real development demand**
  - Check whether new abilities should be added to routing documents (`ABILITY.md`).
  - Decide if several low-level abilities can be composed into a new high-level ability (better semantic abstraction).

The long-term goal is that, especially in AI-heavy development, the ability routing system will **adapt to real usage**, reducing friction and improving AI productivity over time.

#### 3.2.2 Trigger Points for Maintenance

There are several natural trigger points for routing maintenance.

**New ability registration**

Whenever a new ability (low-level or high-level) is added to the registered ability pool:

- Scan module-level `ABILITY.md` documents to see where it should appear, based on:
  - Module roles, responsibilities, and dependencies.
- Scan integration-level `ABILITY.md` documents and integration specs to:
  - Attach the new ability where it can contribute to integration or E2E testing.

**New module instance or module changes**

When a new module instance is added or its responsibilities change:

- Scan the ability pool based on the module’s declared responsibilities and components.
- Generate or update the module’s `ABILITY.md` so that it exposes the most relevant abilities for that module.

**Periodic verification (CI)**

On a scheduled basis (e.g. daily or on every main-branch build), CI can:

- Compare the registered ability pool against all module instances and their declared requirements.
- Suggest updates to module-level and integration-level `ABILITY.md` documents.
- Detect unused or broken entries.

**Inspection tools**

The template provides inspection knowledge as pseudo-agents (workflow prompts) that the AI can load to drive verification. These inspection agents may have access to scripts such as:

- `ability list` – list all registered abilities and where they are referenced.
- `ability usage-report` – summarize recent ability usage based on `tool_call.md` records.
- `ability consistency-check` – find mismatches between registries and `ABILITY.md` documents.
- `ability dead-link-check` – detect registry paths in `ABILITY.md` that no longer exist.

AI can use these tools to run maintenance tasks, propose changes, and update routing documents (subject to review, if desired).

### 3.3 Other Considerations

#### 3.3.1 Hooks: Matchers and Guardrails

Ability routing mainly relies on **semantic matching** between the orchestration plan and the ability descriptions. Hook mechanisms help with:

- Suggesting relevant abilities.
- Enforcing safety and policy rules.
- Maintaining routing information.

Hook mechanisms can be activated in four ways:

1. User input keyword matching.
2. Pre-call activation (before an ability is executed).
3. Post-call caching of AI tool usage.
4. Post-processing based on cached information.

The `matcher` and `guardrail` fields in both low-level and high-level configuration documents support these mechanisms.

- **Matcher**
  - A set of rules that map input text (user prompts, plan text) to candidate abilities.
  - Example: if the plan mentions “signup”, “create user”, or “registration”, suggest abilities related to user signup flows.
- **Guardrail**
  - A set of rules applied at execution time to enforce constraints.
  - Example: block destructive operations in production or require manual approval.

Example matcher configuration (conceptual):

- Ability: `db.write.user_row`
- Matcher rules:
  - Keywords: `insert`, `create user`, `write user row`.
  - Negative keywords: `backup`, `archive`.
  - Scope filters: only modules that own user data.

Example guardrail configuration (conceptual):

- Ability: `billing_regression`
- Guardrails:
  - Only callable in `staging` and `test` environments.
  - Requires manual confirmation if the run triggers more than N API calls.
  - Logs every run with a high-visibility tag.

Beyond direct matching and guardrails, we can use **usage-based hooks**:

- Aggregate data from `tool_call.md`:
  - Common AI intents.
  - Frequent ability combinations.
  - Failures due to missing abilities.
- Use this data to:
  - Update matcher rules for more accurate suggestions.
  - Identify areas where new high-level abilities should be created.
  - Mark rarely used abilities as deprecated or lower priority.

#### 3.3.2 Configuration Maintenance

Configuration is critical: incorrect configuration can cause abilities to behave unexpectedly, even if their registry definitions are correct. To keep configuration reliable:

- **Separate configuration from semantics**
  - Registries define what an ability does and its input/output contracts.
  - Configuration defines how and where it runs in a specific environment.
- **Version and scope configuration**
  - Maintain configuration per environment (`dev`, `staging`, `prod`) and, if needed, per module.
  - Track configuration versions and changes so that regressions can be rolled back.
- **Validate configuration**
  - Provide CI checks and `ability lint` commands to ensure:
    - Referenced implementations exist.
    - Environment variables and credentials are present.
    - Timeouts, limits, and endpoints conform to policy.
- **Expose configuration to the AI in a controlled way**
  - The AI should not read raw credentials or secrets.
  - Instead, the AI should rely on the execution script’s feedback to know:
    - Whether an ability is available.
    - Which modes (e.g. environments, optional behaviors) are currently configured.

By keeping configuration healthy and validated, we ensure that:

- The AI can trust ability availability signals.
- Execution results are predictable and consistent.
- Updating infrastructure or external services can be done with minimal impact on orchestration logic.

---

This document defines the conceptual model and operational workflow for ability routing within the template repository. It is designed from the perspective of a code-oriented AI model and an AI task orchestrator, with the single metric of success being **higher AI development efficiency under controlled and observable execution**.
