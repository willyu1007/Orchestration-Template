
# Architecture Docs Adjustment Plan – Multi-End & Monorepo Support (Options 4.1 and 4.2)

> This plan assumes we extend the existing AI-first repository architecture to handle **multi-end** scenarios  
> (e.g. one backend + multiple clients) and **monorepo** layouts, while **avoiding disruptive refactors**.  
> Options 4.1 and 4.2 are interpreted as *metadata-first* and *mechanism-level* extensions that keep the current
> module-centric design intact.

---

## 1. Intent and Global Principles

### 1.1 Intent

We want to:

1. Model **multiple “surfaces” / ends** explicitly (backend, web frontend, mobile app, desktop client, shared libraries, etc.).
2. Keep the existing **module-instance-centric architecture** as the main building block.
3. Make it easy for **AI developers** to:
   - Locate the correct module(s) for a given surface and app.
   - Understand cross-surface flows (e.g. “web checkout UI → orders API → payments API”).
   - Use abilities, knowledge routing, hooks, and DevOps layouts without guessing which surface they apply to.
4. Preserve **backward compatibility**: projects that do not use multi-end or monorepo patterns should not be forced to change anything.

### 1.2 Principles

- **No mandatory directory refactor**  
  - Keep the `modules/<module_id>/` convention valid.  
  - New multi-end semantics are introduced via metadata and documentation, not forced folder rearrangements.

- **Single source of truth per concern**  
  - Multi-end and monorepo concepts are primarily modeled in:
    - Module `MANIFEST.yaml` + `modules/overview/*` registries
    - DevOps directories (`/ops/packaging`, `/ops/deploy`, `/db`)
    - Ability/knowledge routing metadata
  - The architecture docs are updated so these modeling points are explicit and consistent.

- **Surface-aware, but module-first**  
  - “Runtime surface” (e.g. `web_frontend`, `mobile_app`, `http_api`) is added as a *feature* of a module, not as a replacement for `module_type`.  
  - The existing type graph (business / technical types) still drives allowed flows.

- **AI discoverability over visual neatness**  
  - Instead of relying on humans reading a visually tidy `modules/` tree, we give AI:
    - Registries and metadata that group modules by surface and app.
    - Clear AGENTS and routing docs that describe how to navigate.

---

## 2. Option 4.1 – Multi-End Modeling at the Modular / Registry Layer

### 2.1 Summary

Option 4.1 introduces **explicit multi-end metadata** into the modular specification and registries, with **no change to physical paths**. The `modules/` directory can remain “flat”; grouping and navigation are expressed via:

- New fields in `MANIFEST.yaml` per module instance.
- Extensions to `modules/overview/instance_registry.yaml` and, optionally, `type_registry.yaml`.
- Explanatory text and examples in the architecture docs.

### 2.2 New / Extended Metadata

#### 2.2.1 MANIFEST.yaml (per-module)

Add two *optional* fields:

```yaml
module_id: payments.api
name: Payments HTTP API
description: ...
module_type: http_api
module_level: feature
parent_module: commerce.payments

runtime_surface: http_api        # e.g. http_api | web_frontend | mobile_app | desktop_app | worker | shared_lib
app_id: commerce.checkout        # logical app / host grouping across backends + clients
```

- `runtime_surface`  
  - Describes *where* this module’s code runs conceptually.
  - Default may be inferred from `module_type` for simple projects; explicit field is recommended for multi-end / monorepo setups.

- `app_id`  
  - Groups modules that belong to the same logical app / product slice: one backend service + several clients + shared modules.
  - Examples: `teacher_portal`, `student_app`, `internal_admin`, `commerce.checkout`.

These fields are **advisory**, not hard constraints; they are used by routing, scaffolding, and DevOps flows.

#### 2.2.2 instance_registry.yaml (project-level)

Extend each module entry with the same two fields plus an explicit `repo_path` (without changing the default path convention):

```yaml
instances:
  - module_id: payments.api
    module_type: http_api
    module_level: feature
    parent_module: commerce.payments
    runtime_surface: http_api
    app_id: commerce.checkout
    repo_path: modules/payments.api/     # default, unchanged

  - module_id: checkout.web
    module_type: web_frontend
    runtime_surface: web_frontend
    app_id: commerce.checkout
    repo_path: modules/checkout.web/
```

- `repo_path` keeps the current default (`modules/<module_id>/`) but allows future flexibility if needed.
- Multi-end “views” are obtained by grouping on `runtime_surface` and/or `app_id`.

#### 2.2.3 type_registry.yaml (optional extension)

Optionally annotate types with a *typical* runtime surface to help AI infer defaults:

```yaml
types:
  - id: web_frontend
    description: User-facing web UI modules.
    outbound: [http_api]
    default_runtime_surface: web_frontend

  - id: http_api
    description: HTTP API modules.
    inbound: [web_frontend]
    outbound: [domain_service]
    default_runtime_surface: http_api
```

This stays compatible with both the **domain-oriented** and **technical-layer** type graph variants.

### 2.3 Document-by-Document Adjustments

Below, “add subsection” means append new paragraphs without rewriting existing content.

#### 2.3.1 `modular_mannual.md`

1. **Section 1.1 – Module Instance**  
   - Add a subsection “Runtime surface and app grouping” that:
     - Introduces `runtime_surface` and `app_id` as optional MANIFEST fields.
     - Explains that `runtime_surface` is about *execution context* (client/server/etc.), while `module_type` is about *business or technical role*.
     - States that modules without explicit `runtime_surface` are still valid and are treated according to defaults.

2. **Section 1.2 – Module Type**  
   - Clarify that type graphs can remain domain-oriented, technical-layer-oriented, or hybrid.  
   - Mention that `runtime_surface` is a *helper dimension* for DevOps and routing; it does *not* change type graph semantics.

3. **Section 2 – Module Instance Skeleton**  
   - Keep the skeleton paths; add a note that physical paths still default to `modules/<module_id>/` and that multi-end grouping is handled via `instance_registry.yaml` and `MANIFEST.yaml`.

4. **Section 5.2 – Multi-Module Integration and Testing**  
   - Add short guidance on using `instance_registry.yaml` to:
     - Find all modules for a given `app_id`.
     - Partition involved modules by `runtime_surface` when analyzing a scenario.

#### 2.3.2 `agents_strategy_guide.md`

1. **Section 2.2 – Modular System Root `AGENTS.md`**  
   - Add a paragraph explaining that:
     - Cross-module tasks in multi-end systems should first identify the **app_id** and current **runtime_surface**.
     - The AI should use `modules/overview/instance_registry.yaml` to list modules matching the current app + surface before diving into specific module-level `AGENTS.md`.

2. **Section 4.4 – Directory-Level Strategy**  
   - Provide a concrete example where:
     - `modules/AGENTS.md` routes to “clients vs services” views using `runtime_surface` from the registry, instead of relying on directory names.

#### 2.3.3 `devops_extension_guide.md`

1. **Section 2.5 – Clean separation of DevOps structures**  
   - Add explicit alignment:
     - `/ops/packaging/services` primarily packages modules whose `runtime_surface` is `http_api` or `worker`.
     - `/ops/packaging/apps` primarily packages modules with `runtime_surface` `web_frontend`, `mobile_app`, or `desktop_app`.
   - Clarify that the mapping is driven by `instance_registry.yaml` (module → runtime_surface) rather than ad-hoc naming.

2. **Sections 5 and 6 – Packaging and Deployment**  
   - For each packaging/deployment descriptor example, include `module_id` and (optionally) `runtime_surface` fields.
   - Add guidance that:
     - DevOps scripts may filter modules by `runtime_surface` or `app_id` to compute which artifacts to build or deploy for a logical app.

#### 2.3.4 `ability_routing.md`

1. **Section 1.2 – Ability Pool**  
   - Add `runtime_surface` as an optional semantic tag on abilities:
     - For low-level abilities, it typically matches the module’s surface (e.g. `http_api`).
     - For high-level abilities (e.g. E2E flows), it may be `multi_surface`.

2. **Section 1.3 – Ability Registries**  
   - Low-level registry entries:
     - Add an optional field `runtime_surface` (or reuse the module’s `runtime_surface` if omitted).  
   - High-level registry entries:
     - Add an optional field `app_id` to indicate the logical app / experience the high-level ability belongs to.

3. **Section 2.1 – Task Orchestration Perspective**  
   - Add a recommendation:
     - When planning a multi-end task, group candidate abilities by `app_id` and `runtime_surface` so that orchestration stays aligned with the target app and surface.

#### 2.3.5 `content_routing.md` (Knowledge Routing)

1. **Section 2.2 – Document location and layout**  
   - Add a paragraph suggesting `scope` naming for multi-end projects, e.g.:
     - `frontend.web`, `frontend.mobile`, `backend.http_api`, `backend.worker`.
   - Clarify that:
     - Knowledge documents for a specific surface should declare the appropriate `scope`.
     - Cross-surface docs (e.g. “checkout end-to-end flow across web + mobile”) may use a `scope` like `integration.checkout`.

2. **Section 3 – Implementation Notes**  
   - Suggest that routing maintainers may choose to group topics by `app_id` + `runtime_surface` to keep per-surface knowledge compact.

#### 2.3.6 `hooks.md`

1. **Section 1.1 – Event Types and Contexts**  
   - Extend `PreAbilityCallContext` and `SessionStopContext` types with optional fields:

     ```ts
     interface PreAbilityCallContext {
       ...
       ability_id: string;
       runtime_surface?: string; // from ability or owning module, e.g. web_frontend, http_api
       app_id?: string;          // logical app grouping, if known
       ...
     }

     interface SessionStopContext {
       ...
       runtime_surfaces?: string[]; // surfaces touched in this session
       app_ids?: string[];          // apps touched in this session
       ...
     }
     ```

2. **Section 6.3 – Using AI to monitor and improve practices**  
   - Add examples where:
     - A `SessionStop` hook aggregates ability usage by `runtime_surface`.
     - A `PromptSubmit` hook uses `app_id` hints to emit routing suggestions (“you are likely working on the teacher web app”).

#### 2.3.7 `README.md` (AI-First Repository Template – Overview)

1. **Section 2 – Modular Architecture**  
   - Add a short sub-section “Multi-end & monorepo usage” summarizing:
     - `runtime_surface` and `app_id` metadata.
     - The fact that physical paths remain `modules/<module_id>/` by default.
     - That AI should use `instance_registry.yaml` for app/surface-aware navigation.

2. **Section 9 – Cross-Module Integration**  
   - Mention that integration scenarios (`modules/integration/scenarios.yaml`) may refer to `app_id` as an additional dimension when describing flows across surfaces.

---

## 3. Option 4.2 – Surface-Aware Mechanisms (Abilities, Knowledge, Hooks, DevOps)

### 3.1 Summary

Option 4.2 builds on Option 4.1’s metadata to make **core mechanisms explicitly surface-aware**:

- Abilities: know which surface/app they belong to.
- Knowledge routing: scopes/topics reflect surfaces.
- Hooks: can enforce different guardrails per surface.
- DevOps: packaging/deploy descriptors tie cleanly into surfaces and app groups.

This does **not** introduce new metadata beyond `runtime_surface` and `app_id`; instead, it wires them through the existing mechanisms.

### 3.2 Ability Routing Adjustments (Detail)

1. **Ability registries**  
   - Low-level abilities:
     - Use `runtime_surface` tags to distinguish, for example:
       - `http.call.checkout_backend` (`runtime_surface: http_api`)
       - `ui.snapshot.checkout_page` (`runtime_surface: web_frontend`)
   - High-level abilities:
     - Use `app_id` + `runtime_surface` or `multi_surface` to describe coverage:
       - `checkout_e2e_web` (`app_id: commerce.checkout`, `runtime_surface: web_frontend`)
       - `checkout_e2e_mobile` (`app_id: commerce.checkout`, `runtime_surface: mobile_app`)
       - `checkout_e2e_cross_surface` (`app_id: commerce.checkout`, `runtime_surface: multi_surface`).

2. **Routing documents (`ABILITY.md`)**  
   - Module-level `ABILITY.md`:
     - Should mention the module’s `runtime_surface` in the introduction, so an AI quickly sees whether abilities are client- or server-side.
   - Integration-level `ABILITY.md`:
     - Group abilities by `(app_id, runtime_surface)`, for example:

       ```markdown
       ## Checkout – Web

       | Id                    | Type     | Runtime surface | Description                     |
       | --------------------- | -------- | --------------- | ------------------------------- |
       | checkout_e2e_web      | workflow | web_frontend    | E2E checkout via web frontend.  |

       ## Checkout – Mobile
       ...
       ```

3. **Execution guidance**  
   - In the “Usage and Typical Flows” section, add a recommendation:
     - When orchestrating a scenario for a specific app/surface, prefer abilities matching that `app_id` and `runtime_surface`, falling back to `multi_surface` abilities when appropriate.

### 3.3 Knowledge Routing Adjustments (Detail)

1. **Scopes and topics**  
   - Encourage a surface-aware naming convention for `scope` fields in knowledge docs and ROUTING indexes, e.g.:
     - `frontend.web.checkout`, `frontend.mobile.checkout`, `backend.checkout.api`.
   - Multi-surface topics (e.g. “checkout E2E”) can sit under `integration.checkout`.

2. **Topic route files**  
   - For each major app/surface, maintain dedicated routes:
     - `/routes/frontend.web.checkout.yaml`
     - `/routes/frontend.mobile.checkout.yaml`
   - Inside each route, explain clearly whether the content is:
     - Surface-specific (e.g. CSS/layout patterns), or
     - Cross-surface but viewed from this surface’s perspective (e.g. error mapping).

3. **ROUTING.md**  
   - At module and integration root levels, document:
     - How to choose between surface-specific vs cross-surface topics.
     - That the AI may need to open multiple topics when working on cross-surface scenarios (e.g. both web and mobile “checkout” topics plus an `integration.checkout` topic).

### 3.4 Hooks and Guardrails

1. **PreAbilityCall guardrails**  
   - Use `runtime_surface` and `app_id` in hook matching rules, e.g.:
     - A hook that forbids destructive operations on `web_frontend` modules in production environments.
     - A hook that enforces additional logging for `mobile_app` surfaces (where debugging is harder).

2. **PromptSubmit routing hints**  
   - Hooks can analyze incoming user prompts plus `app_id` hints to emit:
     - `routing_hint` signals that suggest:
       - Relevant surface-specific knowledge topics.
       - Surface-specific abilities (web vs mobile vs API).

3. **SessionStop summarization**  
   - Summaries can be grouped by surface/app:
     - “This session modified the checkout web frontend and the checkout API modules.”

### 3.5 DevOps and CI Alignment

1. **Packaging (`/ops/packaging/`)**  
   - Ensure each packaging definition contains:
     - `module_id`
     - `runtime_surface`
     - Optionally `app_id`
   - CI can use these to:
     - Build only the relevant artifacts for a given app (all modules with `app_id=commerce.checkout`).
     - Distinguish client vs server build pipelines.

2. **Deployment (`/ops/deploy/`)**  
   - For `http_services`, expect to deploy modules with `runtime_surface=http_api` or `worker`.  
   - For `clients`, expect `web_frontend`, `mobile_app`, etc.  
   - Deployment descriptors should reference the same `module_id`, `runtime_surface`, and `app_id` values as `instance_registry.yaml`.

3. **Database (`/db/`)**  
   - For data shared across surfaces, link schema/migration entries back to all participating modules via `app_id`.  
   - This makes it easy to see which surfaces are affected by a schema change.

---

## 4. Execution Details and Order

### 4.1 Recommended Order of Edits

1. **Modular spec and registries**
   - Update `modular_mannual.md` to introduce `runtime_surface` and `app_id`.
   - Update `instance_registry.yaml` and `MANIFEST.yaml` schema descriptions accordingly.
2. **Overview and strategy docs**
   - Adjust `README.md`, `modules/AGENTS.md` (via `agents_strategy_guide.md`), and `modules/overview/AGENTS.md` behavior to refer to the new metadata.
3. **Mechanism docs**
   - Update `ability_routing.md`, `content_routing.md`, and `hooks.md` to accept and propagate `runtime_surface` and `app_id`.
4. **DevOps docs**
   - Align `devops_extension_guide.md` with the new runtime surface semantics for packaging and deployment.
5. **Optional knowledge doc**
   - Add `knowledge/architecture/multi_end_monorepo.md` as a human-oriented explanation of these concepts, and reference it from root-level `AGENTS.md` and relevant ROUTING entries.

### 4.2 Impact on Existing Templates

- Projects that ignore multi-end / monorepo concerns:
  - Can leave `runtime_surface` and `app_id` empty.
  - Do not need to change directory layout or registries beyond schema defaults.
- Projects that use multi-end / monorepo:
  - Gain a structured way to model surfaces and apps for AI, without reworking existing modules.

---

## 5. Out-of-Scope and Non-Goals

- **No forced re-layout of `modules/`**  
  - This plan deliberately avoids introducing new mandatory subdirectories like `modules/clients/` or `modules/services/`.  
  - If such a layout is desired later, it can be introduced as a separate “Option 4.3” with clear migration steps.

- **No change to core type graph semantics**  
  - `module_type` and `type_registry.yaml` continue to describe data/control flows.  
  - `runtime_surface` and `app_id` are overlays for observability, DevOps, and AI navigation.

- **No changes to existing build/runtime code in this plan**  
  - All changes are to architecture docs and metadata specifications.  
  - Runtime and scaffolding scripts are expected to consume these fields in later iterations (via updates to Phase 4–7 plans).

---

## 6. Summary

Options 4.1 and 4.2 extend the current architecture so that:

- Multi-end and monorepo scenarios are **first-class concepts** in metadata and documentation.
- AI developers have **predictable, registry-backed ways** to:
  - Find “the right module for this app and surface”.
  - Choose abilities and knowledge tailored to a given surface.
  - Understand and respect DevOps boundaries per surface.
- Existing projects remain valid, and no disruptive refactor of `modules/` is required.

Subsequent adjustments to the 8-phase construction guides should ensure that `runtime_surface` and `app_id` are populated and used consistently as the template is built out.
