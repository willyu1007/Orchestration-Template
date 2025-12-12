# Miscellaneous Considerations

> This document is **secondary**.  
> It does **not** define the core architecture, routing model, or infrastructure of the template.  
> Instead, it collects **supporting practices** that help AI‑first projects run smoothly over time.

The topics here are intentionally pragmatic and implementation‑oriented:

- Project initialization.
- Local and environment setup.
- Cloud / serverless runtimes and contracts.
- Delivery forms and operational hand‑off.

These guidelines are written with **AI as a primary developer** in mind, but they should also be helpful for humans.

---

## 1. Project Initialization (`/PROJECT_INIT`)

The template can provide a **bootstrap scaffold** that helps a human+AI pair set up a new project from zero.

### 1.1 Directory and lifecycle

We reserve an optional directory:

- `/PROJECT_INIT/`

Characteristics:

- Exists only during **initial bootstrapping** of a new project.
- Contains docs and AI workflows that help generate the first set of modules, configs, and infrastructure.
- **Should be deleted** (or archived outside the main tree) once initialization is complete, so it does not pollute normal routing and context.

Example layout:

```text
/PROJECT_INIT/
  doc/          # Human-facing docs
  AI/           # AI-facing workflows and prompts
```

- `/PROJECT_INIT/doc/`
  - `README.md` – how to use the template to initialize a new project.
  - `quickstart.md` – minimal step‑by‑step instructions for first‑time users.

- `/PROJECT_INIT/AI/`
  - Architecture overview of the template.
  - Constraints and examples for initialization operations.
  - Explanations of initialization scripts and tools.
  - One or more “pseudo‑agents” defined via prompts, describing:
    - How to collect project metadata (name, domain, environments).
    - How to choose module types and initial module instances.
    - How to produce initial `AGENTS.md`, `ROUTING.md`, and `ABILITY.md` for each module.
  - Definition of the **desired end state** for initialization
    (for example: at least one module per core business capability, basic CI, and minimal docs in place).

### 1.2 AI‑assisted initialization flow

A recommended flow for AI‑assisted project initialization:

1. **Collect high‑level input**

   - Project name, domain, and high‑level product goal.
   - Expected environments (`dev`, `staging`, `prod`).
   - Initial list of core business capabilities or flows.

2. **Propose an initial module map**

   - Use the rules from `modular_mannual.md` to propose:
     - Module types (business stages or technical layers).
     - Initial module instances with `module_id`, `module_type`, `module_level`.
   - Present the proposed map for human confirmation.

3. **Generate module skeletons**

   For each approved module:

   - Create `modules/<module_id>/` with the skeleton described in `modular_mannual.md`.
   - Generate minimal:
     - `MANIFEST.yaml`
     - `AGENTS.md`
     - `ROUTING.md`
     - `ABILITY.md`
     - `workdocs/AGENTS.md`
     - `config/AGENTS.md`
     - `docs/README.md`

4. **Set up basic CI and hooks**

   - Generate CI configuration (or placeholders) for:
     - Linting.
     - Tests.
     - Basic strategy/routing checks if available.
   - Optionally add minimal hook configuration under `.system/hooks/` to:
     - Log ability usage.
     - Run basic checks on `SessionStop` or pre‑commit.

5. **Finalize and clean up**

   - Summarize what was created in a final report under `docs/` (e.g., `docs/PROJECT_INIT_SUMMARY.md`).
   - Once the project owner is satisfied:
     - Delete `/PROJECT_INIT/` or move it outside the main repo tree.
     - Verify that the project can be understood using only:
       - Project‑level `AGENTS.md`.
       - Module‑level strategies and routing.

---

## 2. Local Environment and Developer Experience

Although this template is AI‑first, humans will collaborate with the AI and run commands locally. Some non‑core but important considerations:

### 2.1 Local tooling expectations

- Define a minimal, **human‑friendly** toolchain:
  - Language runtime(s) and version managers.
  - Package managers.
  - Recommended editors or IDE plugins (for LSP, formatting, etc.).
- Provide a short `docs/local_dev_setup.md` or similar, and reference it from:
  - Project‑level `AGENTS.md`.
  - Relevant `ROUTING.md` topics (e.g., `scope:dev_env`).

AI should:

- Prefer to call abilities / scripts that wrap local tooling instead of assuming specific commands.
- Use knowledge routing to load the local setup docs before making assumptions about the environment.

### 2.2 Environments (dev, staging, prod)

Even if full infrastructure is defined elsewhere, the template should keep a **consistent story** about environments:

- Use a small, predictable set of environment names.
- Document:
  - Which environments are accessible from CI.
  - Which environments AI is allowed to modify via abilities (for example, `dev` only).
- Ensure ability registries (`ability_routing.md` + `ABILITY.md`) specify:
  - For each ability, which environments are allowed and under what conditions.
  - Any manual‑approval requirements for higher‑risk environments.

---

## 3. Cloud and Serverless Runtimes (Optional but Common)

Many AI‑first projects deploy part of their logic to cloud runtimes or serverless platforms.  
This section lists optional but useful conventions for such setups.

### 3.1 Runtime description

If the repository uses serverless functions, background workers, or similar:

- Add a human+AI readable document such as `docs/runtime/serverless_overview.md` that explains:
  - The main runtime model (functions vs services vs jobs).
  - How they are deployed (CI/CD, infrastructure‑as‑code, manual steps).
  - How logs and metrics are exposed.

Reference this document from:

- Project‑level `AGENTS.md`.
- Relevant module `AGENTS.md` for modules that own runtime components.
- Knowledge routing topics (for example, `scope:runtime`, `topic:serverless`).

### 3.2 Resource model

For each function or service, it is helpful to describe at least:

- **Resource type and shape**
  - Functions vs long‑running services vs scheduled jobs.
  - Language/runtime and packaging method.
- **Integration points**
  - Triggers (HTTP/API Gateway, events, queues, scheduled tasks).
  - Common libraries or middlewares that must be used.
- **Observability**
  - Where logs go.
  - How tracing/metrics are configured.
  - Any default dashboards that exist (referenced as URLs or names, not raw IDs).

The exact location of these docs can vary (e.g., under `docs/runtime/` or module‑local `docs/`), but they should be discoverable via `ROUTING.md`.

### 3.3 Contracts and guardrails

Serverless or cloud runtimes should have clear contracts. For each function or service, document:

- **Handler / entrypoint**

  - Function signature, expected request/response shapes.
  - Routing rules (paths, methods, event types).

- **Operational limits**

  - Baseline values for timeout, memory, and concurrency.
  - Which values are safe defaults for development vs production.
  - Any limits that must **not** be changed automatically by AI.

- **Permissions (least privilege)**

  - Which cloud resources are required (e.g., object storage, databases, queues).
  - Recommended IAM boundaries (roles, policies), at a conceptual level.
  - Which changes must always be reviewed by a human (for example, granting new write permissions on production data).

- **Cold‑start and performance sensitivity**

  - Dependencies that are heavy or slow to import.
  - Initialization work that must be minimized (e.g., only open DB connections lazily).
  - Notes about latency requirements or SLAs if they exist.

Where possible, keep these contracts close to the owning module:

- Under `modules/<module_id>/docs/runtime/…` or `interact/…` (if expressed as OpenAPI/event schemas).
- Linked from the module’s `AGENTS.md` and `ROUTING.md`.

### 3.4 AI behavior around infrastructure

For **code models**:

- Treat runtime contracts as **source of truth**.  
  Do not invent new endpoints, event types, or IAM permissions without:
  - Reading the existing docs.
  - Updating docs and configuration when you intentionally change behavior.

- When unsure about infrastructure behavior:
  - Prefer to propose changes in `workdocs/active/<task_id>/plan.md`.
  - Ask for human confirmation (as specified in `AGENTS.md`) before performing high‑impact changes.

For **orchestrators**:

- Expose infrastructure‑related docs as part of the `understand` / `plan` context for relevant tasks.
- Use ability routing to route calls to:
  - Deployment scripts.
  - Validation checks (linting IaC, security scanners).
- Optionally use hooks to:
  - Run lightweight validation after infrastructure‑affecting diffs.
  - Emit warnings if abilities try to touch disallowed environments.

---

## 4. Delivery Forms and Operational Handoff

While details differ per project, two delivery forms are particularly common:

- **Code packages** (libraries, services pushed as source + build scripts).
- **Container images** (for services, jobs, or serverless container platforms).

### 4.1 Code packages

When the project produces deployable code packages:

- Document:
  - Build commands (and where they live as abilities).
  - Package naming and versioning conventions.
  - How packages are published (internal registries, public registries).
- Ensure:
  - Ability routing includes high‑level abilities such as “build and publish package X”.
  - Knowledge routing points to package‑related docs when tasks mention publishing or releasing.

AI should:

- Prefer using these high‑level abilities instead of manually assembling build commands.
- Update `workdocs` with:
  - Which version was built.
  - Where it was published.
  - Any follow‑up tasks (e.g., bumping dependencies in other modules).

### 4.2 Container images

When the project uses containers:

- Describe, ideally in a central doc (e.g., `docs/runtime/containers.md`):
  - Base images.
  - Standard entrypoints.
  - Environment variable conventions.
  - Health checks and readiness probes.

- For module owners:
  - Each module that builds containers should say in its `AGENTS.md`:
    - Which Dockerfiles or build scripts it owns.
    - Which abilities to use for building and pushing images.
    - Any special constraints (size limits, startup time goals, security scans that must pass).

AI should:

- Avoid inventing new images or base images without referencing existing patterns.
- Use provided build/push abilities instead of crafting ad‑hoc commands.
- Consult container docs when changing:
  - Exposed ports.
  - Environment variables.
  - Health checks.

---

## 5. How This Document Interacts with Core Specs

This miscellaneous document is deliberately **non‑authoritative**:

- If anything here conflicts with:
  - `modular_mannual.md`
  - `ability_routing.md`
  - `content_routing.md`
  - `hooks.md`
  - or project/module `AGENTS.md`
- Then the **core specs and local strategy docs win**.

Recommended usage:

- Link this document from project‑level `AGENTS.md` as **optional reading** for:
  - People setting up a new project from the template.
  - AI agents or humans working on infrastructure and operations tasks.
- Use knowledge routing (`ROUTING.md`) to expose relevant sections:
  - Initialization topics (`scope:project_setup`).
  - Runtime topics (`scope:runtime`).
  - Delivery topics (`scope:delivery`).

The intent is to:

- Capture operational wisdom that does not fit cleanly into the core specs.
- Give AI concrete hints about project setup, runtimes, and hand‑off.
- Keep core documents focused, while still making real‑world development smoother for both AI and humans.
