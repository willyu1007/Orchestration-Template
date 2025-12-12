# Documentation Alignment & Improvement Plan

This document proposes **high-level adjustments** to keep all template documents logically consistent, highly collaborative, and optimized for AI developer efficiency. It focuses on **conceptual alignment and cooperation between mechanisms**, rather than detailed wording.

The existing documents are:

- Modular development specification (`modular_mannual.md`)
- Ability routing (`ability_routing.md`)
- Knowledge routing (`content_routing.md`)
- Agent strategy guides (`agents_strategy_guide.md` / `AGENTS.md`)
- Hooks (`hooks.md`)
- DevOps extension guide (`devops_extension_guide.md`)

---

## 1. Shared Glossary and Role Model

### 1.1 Canonical role definitions

To eliminate ambiguity, introduce a **single shared glossary** (either in the root README or a dedicated `GLOSSARY.md`) and align all documents to it:

- **AI developer**
  - The primary actor that reads code, plans tasks, edits files, and calls tools.
  - Owns the internal “orchestration loop” (understand → plan → act → review).

- **Internal orchestration**
  - The AI’s own planning and task decomposition process.
  - All references to “task orchestration” in the template should refer to this.

- **Runtime**
  - The always-on process that receives user input, runs hooks, and executes abilities via the standard script.
  - Does not plan tasks; it only enforces rules, emits events, and persists metadata.

- **Hook layer**
  - Repository-local, event-driven automation around the AI’s work.
  - Provides hints, guardrails, and telemetry.

- **Humans**
  - Provide requirements, knowledge, and approvals.
  - Own sensitive operations (credentials, production changes).

**Action:**

- At the top of each major spec (modular, ability, knowledge, hooks, DevOps), add a short “Roles & Terms” subsection referencing this shared glossary instead of redefining roles independently.

### 1.2 Orchestrator terminology

Current wording sometimes treats an external “Agent Runtime / Orchestrator” as the primary driver of the AI’s actions. This conflicts with the AI-first premise.

**Direction:**

- Reserve **“orchestration”** for the AI developer’s internal planning loop.
- Refer to the long-running process only as **“runtime”** or **“agent runtime”**, not as “orchestrator”.

**Concrete adjustments:**

- In `hooks.md`, rename the role “Agent Runtime / Orchestrator” to **“Agent Runtime”**, and explicitly state that:
  - The runtime controls event emission and hook execution.
  - The AI still owns all task-level orchestration and planning.
- In other docs, where “AI orchestrator” is used, clarify that it means the **AI’s internal orchestration**, not a separate service.

This keeps a clear separation of responsibilities and matches the “AI is the main developer” precondition.

---

## 2. Document-Level Alignment

### 2.1 Modular spec (`modular_mannual.md`) as the central spine

The modular specification already serves as the **structural backbone** for the repo:

- Defines module instances, types, and the type graph
- Defines module skeletons (`MANIFEST.yaml`, `AGENTS.md`, `ROUTING.md`, `ABILITY.md`, `workdocs/`, `interact/`, `config/`, `docs/`)
- Describes single-module and cross-module AI workflows

**Improvements:**

1. **Explicit dependencies**
   - Add a short section “This spec depends on” with links to:
     - Ability routing spec
     - Knowledge routing spec
     - AGENTS strategy guide
     - Hooks spec
     - DevOps extension guide
   - Clarify that those specs refine specific aspects that the modular template exposes.

2. **Reading order for AIs**
   - At the beginning, include a “Recommended reading for AI” subsection listing which docs to open and in which order for:
     - Single-module tasks
     - Cross-module tasks
     - DevOps scenarios
   - This reduces search overhead for the AI and anchors all other docs to the modular spec.

3. **Term consistency**
   - Cross-check usages of “orchestrator”, “runtime”, and “hooks” and align with the glossary from Section 1.

### 2.2 Ability routing (`ability_routing.md`)

The ability routing spec is structurally consistent with the modular spec, but the connections can be made more explicit.

**Improvements:**

1. **Module-level entrypoints**
   - Early in the document, explicitly tie `ABILITY.md` to **module instances** and **integration modules**:
     - Module-level `ABILITY.md` lives under `modules/<module_id>/`.
     - Integration-level `ABILITY.md` (for cross-module scenarios) lives under `modules/integration/`.
   - Confirm that these are the only places the AI should normally read `ABILITY.md`.

2. **Alignment with workdocs**
   - Add a short subsection “From workdocs to abilities” describing how:
     - New helper scripts created under `workdocs/outcomes/scripts/` can graduate into low-level abilities.
     - The AI should prefer using registered abilities over calling scripts in `workdocs` directly once they are promoted.
   - This reinforces the evolutionary path from ad-hoc scripts → standardized tools.

3. **Hooks and guardrails**
   - Clarify how the `matcher` and `guardrail` fields in ability configuration connect to the hooks system:
     - `matcher` drives which abilities are suggested by **PromptSubmit hooks**.
     - `guardrail` drives which checks are applied in **PreAbilityCall hooks**.
   - Link to the relevant sections in `hooks.md`.

4. **Runtime naming**
   - Where the document mentions “orchestrator” as a separate entity, align wording with the shared glossary (AI internal orchestration vs runtime).

### 2.3 Knowledge routing (`content_routing.md`)

The knowledge routing spec is conceptually aligned but contains two slightly divergent layout descriptions.

**Improvements:**

1. **Resolve layout duplication**
   - There are two `2.2 Document location and layout` sections with different emphases (module-root vs project-root routing). Clarify the intended pattern:

     - Either: **Module-centric routing (recommended)**  
       - Each module instance has its own `ROUTING.md` and `routes/` under `modules/<module_id>/`.
       - Project-level routing (`ROUTING.md` at repo root) is optional and reserved for global topics.

     - Or: **Hybrid model** with both root-level and module-level entrypoints, clearly explaining when each should be used.

   - Make this choice explicit and remove any contradictory text.

2. **Connect to modular spec and workdocs**
   - Add cross-links back to the modular specification when discussing module instance routing roots.
   - Clarify that **ephemeral workdocs** (plans, context, task logs) are **not** knowledge docs unless explicitly promoted and registered.
   - Adjust references like “see `modulate.md`” so they point to the actual modular/workdocs specification file.

3. **Trigger vs hook terminology**
   - The knowledge routing spec uses “trigger mechanism” and structured intents; hooks use events and hook signals.
   - Add a small mapping:
     - Structured intents from triggers → `routing_hint` and `normalized_intent` HookSignals on `PromptSubmit`.
   - This keeps routing and hooks conceptually synchronized.

### 2.4 Agent strategy guide (`agents_strategy_guide.md`)

The `AGENTS.md` strategy guide aligns well with the modular spec but could be referenced more explicitly.

**Improvements:**

1. **Explicit tie-in to modules**
   - In the project root and modular system sections, mention that module instances conform to the skeleton in the modular spec and that module-level `AGENTS.md` are expected under `modules/<module_id>/`.
2. **Cross-instance guidance**
   - Ensure that root- and integration-level `AGENTS.md` clearly route the AI to:
     - `modules/overview/` registries (types, instances, interfaces)
     - `modules/integration/` scenarios and workdocs
   - This reinforces the intended workflow for cross-module tasks.

3. **Conflict handling examples**
   - The conflict-handling section already defines priority rules. Consider adding one brief example showing:
     - A global rule (e.g., “don’t modify prod configs”)
     - A module-level refinement (e.g., “you may edit staging configs here”)
     - How the AI chooses the more restrictive interpretation when in doubt.

### 2.5 Hooks (`hooks.md`)

Hooks are the most likely source of conceptual confusion due to the “Agent Runtime / Orchestrator” wording.

**Improvements:**

1. **Rename roles and emphasize AI primacy**
   - Rename “Agent Runtime / Orchestrator” to **“Agent Runtime”** throughout.
   - Add a short “Roles” subsection spelling out:

     - Runtime emits events and runs hooks.
     - AI developer owns the plan and calls abilities.
     - Hooks cannot override that plan; they can only block unsafe actions or attach hints.

2. **Clarify “AI-facing” vs “infra-facing”**
   - Keep the existing distinction between `PromptSubmit`/`PreAbilityCall` (AI-facing) and `PostAbilityCall`/`SessionStop` (infra-facing), but explicitly state that:
     - AI-facing events modify the context for the **current internal orchestration step**.
     - Infra-facing events only affect future sessions or infra state.

3. **Tie hook signals to routing and abilities**
   - For each `HookSignal.kind`:
     - `routing_hint` → knowledge routing topics (`ROUTING.md` + `routes/`)
     - `normalized_intent` → structured intents used to choose scopes/topics
     - `ability_guard` → guardrails defined in ability registries
   - Explicitly mention that hooks should **use existing routing and registry metadata** instead of inventing ad-hoc logic.

4. **Describe AI’s perspective succinctly**
   - Add a short “From the AI developer’s perspective” subsection summarizing:

     - What extra data appears in the turn context (hook signals, control messages).
     - Which signals are safe to treat as high-confidence hints.
     - That the AI should wait for a clear `turn_ready` control message before starting each planning step.

### 2.6 DevOps extension guide (`devops_extension_guide.md`)

The DevOps guide is conceptually aligned but can be better anchored to modules and routing.

**Improvements:**

1. **Link DevOps areas to modules**
   - For `/db/`:
     - Explain how module-level `interact/db/` docs relate to the central `/db/schema/` and `/db/migrations/`.
     - Suggest patterns for mapping schema/migration ownership back to modules.

   - For `/ops/packaging/` and `/ops/deploy/`:
     - Indicate how service definitions here relate to `interfaces.yaml` and module instances.
     - Encourage linking from module-level `docs/README.md` to the relevant packaging/deploy definitions.

2. **AGENTS docs for DevOps directories**
   - Recommend adding local `AGENTS.md` under `/PROJECT_INIT/`, `/db/`, `/ops/packaging/`, and `/ops/deploy/` so the AI has explicit behavior rules for each scenario.
   - Keep them short and focused: scope boundaries, safe operations, human approval requirements.

3. **Cross-reference hooks**
   - For operations that are naturally event-driven (e.g., running tests at `SessionStop`, logging migration outcomes), mention that they are good candidates for hooks rather than manual workflows.

---

## 3. Cross-Cutting Themes for AI Efficiency

### 3.1 Single entrypoint story

To reduce AI cognitive load, keep the “first steps” consistent:

- **For single-module work:**
  1. Root README (this overview)
  2. Root or modular `AGENTS.md` to select a module
  3. Module-level `AGENTS.md`
  4. Module-level `ROUTING.md` and `ABILITY.md`
  5. Module `workdocs/`

- **For cross-module work:**
  1. Root README
  2. `modules/AGENTS.md`
  3. `modules/overview/AGENTS.md` (types and registries)
  4. `modules/integration/AGENTS.md`, `scenarios.yaml`, and integration workdocs

**Action:** make sure this sequence is explicitly described in both the modular spec and AGENTS strategy guide.

### 3.2 Clear boundaries between “context” and “knowledge”

- Workdocs (`workdocs/`) and scenario workspaces (`/PROJECT_INIT/workdocs/`, `/db/workdocs/`, etc.) are **ephemeral context**, not knowledge routing targets by default.
- Knowledge docs must:
  - Carry front matter (id, scope, doc_type, summary, etc.)
  - Be referenced from routing (`ROUTING.md` + `routes/`)

**Action:** ensure each spec uses the same language for this distinction and does not implicitly treat workdocs as knowledge docs.

### 3.3 Evolution paths, not one-off structures

Each mechanism should describe a **feedback loop** that improves AI efficiency over time:

- **Workdocs → knowledge docs** (stable patterns promoted to routing)
- **Scripts → low-level abilities** (frequent scripts registered in the ability pool)
- **Lessons → guardrails** (mistakes turned into hook rules or CI checks)
- **Usage logs → better routing** (hooks/CLI surfacing which docs and abilities are actually used)

**Action:** add short “evolution” subsections in the modular, ability, knowledge, hooks, and DevOps specs summarizing how each piece feeds into the others.

---

## 4. Governance and Maintenance

### 4.1 CI and validation

Align validation checks so they reinforce the same model:

- **Modular / registries**
  - Validate module skeleton completeness
  - Ensure types and instances are consistent with the type graph

- **Knowledge routing**
  - Validate front matter and `ROUTING.md` / `routes` structure
  - Ensure all `related_docs.path` targets exist

- **Ability routing**
  - Validate registry schemas and references from `ABILITY.md`
  - Ensure configuration does not reference missing implementations

- **Hooks**
  - Validate hook YAML structure and handler paths
  - Optionally lint for disallowed fields (e.g., AI-facing signals in infra-only events)

- **DevOps**
  - Validate presence of required directories/files (`schema/`, `migrations/`, packaging/deploy descriptors)

**Action:** keep CI checks described in each spec compatible and avoid overlapping or conflicting rules.

### 4.2 Documentation ownership

For each major spec, identify maintainers (teams or roles) responsible for:

- Approving structural changes
- Keeping cross-references accurate
- Coordinating CI rule updates when conventions evolve

Explicit owners make it easier for both humans and AIs to trust the documentation.

### 4.3 Change discipline

When evolving the template:

1. Update the **central modular spec** first if the overall structure changes.
2. Then update dependent specs (ability, knowledge, hooks, DevOps) to match.
3. Only then modify CI and scaffolding tools.

This keeps the modular spec as the source of truth and avoids drift between documentation and automation.

---

## 5. Summary of Priorities

When refining the documentation set, prioritize improvements that:

1. **Clarify roles and orchestration**  
   - AI developer is the orchestrator of development tasks; runtime + hooks are supporting infra.

2. **Reduce navigation cost for the AI**  
   - Make entrypoints and reading order obvious; ensure each spec knows where it sits in the overall map.

3. **Strengthen cooperation between mechanisms**  
   - Ability routing, knowledge routing, hooks, and workdocs should reinforce each other, not compete.

4. **Support long-term evolution**  
   - Define clear paths for turning repeated behaviors and insights into registered abilities, knowledge docs, and guardrails.

By focusing on these themes, the documentation set will remain coherent and efficient as the AI-first project grows in scope and complexity.
