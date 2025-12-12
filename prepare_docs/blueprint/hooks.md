# Hooks for AI-Centric Development

> **Status**: Draft, evolving with real usage.  
> **Audience**: People (and AIs) building repos where an AI developer is the primary actor, and hooks are the infra around it.

This document defines a small, event‑driven **hook system** that lives next to abilities, tools, and knowledge routing in an AI‑centric repo.

The core assumptions are:

- The **AI is the primary developer**: it reads code, plans work, edits files, runs tools.
- A long‑running **Agent Runtime (orchestrator)** owns the lifecycle: it receives user input, emits events, runs hooks, and calls abilities.
- **Hooks are repository‑local infrastructure**: they wrap the AI’s work with logging, guardrails, and automation, without being hard‑wired into the AI’s prompt.

The goals of this design:

1. **Help the AI**, don’t fight it  
   - Provide routing hints, guardrails, and summaries in a structured, predictable way.
2. **Stay lightweight and optional**  
   - The repo must work with all hooks disabled; hooks are an “extra layer,” not core logic.
3. **Keep responsibilities sharp**  
   - Only some events may directly influence the AI’s current turn. Others are infra‑only.
4. **Be easy to scaffold and maintain**  
   - Hooks are small YAML files + handler scripts; a dedicated scaffold flow helps create them.

---

## 0. Overview: Roles and Architecture

We assume three main roles in this repo:

- **AI developer**  
  - The large code‑capable model loop that plans, edits, and calls abilities.
- **Agent Runtime / Orchestrator**  
  - A long‑running process or service that:
    - Receives user input.
    - Emits lifecycle events (PromptSubmit, PreAbilityCall, PostAbilityCall, SessionStop).
    - Runs hooks for each event.
    - Calls abilities/tools.
- **Hook layer**  
  - Repository‑local configuration + handler scripts that run on specific events.

High‑level flow for a typical interactive session:

1. User or system sends a new **task / message**.  
2. Runtime emits a `PromptSubmit` event and runs **blocking hooks** for it.  
3. Runtime merges hook signals into a **turn context**, then hands control to the AI developer.  
4. AI plans, chooses abilities to call.  
5. For each ability call:
   - Runtime emits `PreAbilityCall` and runs blocking hooks.
   - If allowed, runtime executes the ability.
   - Runtime emits `PostAbilityCall` and runs infra‑only hooks.
6. When the session ends or a logical phase completes:
   - Runtime emits `SessionStop` and runs infra‑only hooks.
   - Hooks may run build/test, persist summaries, update caches.

Hooks are entirely driven by the runtime; the AI never calls hooks directly. Instead, the AI sees **structured “hook signals” and control messages** that the runtime injects into the turn context.

---

## 1. Event Model and Lifecycle

Hooks attach to **events** emitted by the runtime. Each event type has a clear purpose and a specific **interaction contract** with the AI developer loop.

### 1.1 Event types

This document uses the following events:

- `PromptSubmit`  
  Emitted when a new user task / message is submitted. Used to:
  - Normalize intent.
  - Suggest abilities and documents.
  - Attach routing hints for the first turn.

- `PreAbilityCall`  
  Emitted right before an ability/tool is executed. Used to:
  - Enforce guardrails and safety checks.
  - Validate arguments and environment.
  - Decide whether the ability call is allowed.

- `PostAbilityCall`  
  Emitted right after an ability/tool finishes. Used to:
  - Log usage, latency, and outcomes.
  - Update caches and quality metrics.
  - Attach metadata, but **not** to influence the current turn’s logic.

- `SessionStop`  
  Emitted when a session or logical phase ends. Used to:
  - Run build/test/lint across changed components.
  - Aggregate and persist a session summary.
  - Prepare signals for future sessions (but not retroactively change this one).

### 1.2 Per‑event AI interaction contract

Not all events are allowed to interact with the AI in the same way. To keep the AI’s execution loop predictable, we distinguish:

- **AI‑facing, blocking events** – may directly shape the current turn and must run synchronously:
  - `PromptSubmit`
  - `PreAbilityCall`
- **Infra‑facing, post‑processing events** – may log, cache, or summarize, but **do not affect the current turn**:
  - `PostAbilityCall`
  - `SessionStop`

Conceptually:

| Event           | Timing              | Blocking? | May directly shape current turn? | Typical use cases                                           |
|-----------------|---------------------|-----------|-----------------------------------|-------------------------------------------------------------|
| PromptSubmit    | Before planning     | Yes       | **Yes**                           | Routing hints, normalized intent, doc suggestions          |
| PreAbilityCall  | Before ability call | Yes       | **Yes**                           | Guardrails, pre‑flight checks, permission checks           |
| PostAbilityCall | After ability call  | No (for AI)| **No**                           | Telemetry, quality metrics, logging, cache updates         |
| SessionStop     | After session phase | No (for AI)| **No**                           | Build/test/lint aggregation, summaries for future sessions |

Runtime behavior must respect these contracts:

- For **PromptSubmit** and **PreAbilityCall**:
  - All **blocking hooks** must be run **before** the AI planner or ability execution proceeds.
  - Hook outputs are merged into a structured **hook signal list** and sent to the AI with an explicit “ready” control signal.
- For **PostAbilityCall** and **SessionStop**:
  - Hooks may run in the background or synchronously from the runtime’s point of view.
  - Their outputs must **not** be fed back into the AI’s current planning loop.

PostAbilityCall is scoped strictly to ability/tool invocations. It may observe and log the tool's stdout/result as part of the infra context, but it is not used to inspect or modify the primary AI developer's own responses. 
If you need to evaluate the AI's final replies, consider a separate infra-only event such as `ModelOutput`.

---

## 2. Hook Declaration (YAML schema)

Hooks are declared as small YAML files under `.system/hooks/` (one file per hook). The orchestrator loads these files, filters them by event type and match rules, and executes the handlers.

A typical hook file looks like this:

```yaml
id: ability_usage_tracker
event_type: PostAbilityCall
enabled: true
blocking: false

summary: >
  Track each ability call and update ability_usage.json. For slow calls,
  append to slow_abilities.log.

match:
  ability_scope: "*"
  min_duration_ms: 0

handler:
  kind: script
  command: ./scripts/hooks/ability_usage_tracker.py

effects:
  - update_usage_cache: .system/cache/ability_usage.json
  - append_to: .system/logs/slow_abilities.log
```

### 2.1 Required fields

- `id`  
  - Unique identifier within the repo. Use `kebab-case`.
- `event_type`  
  - One of: `PromptSubmit`, `PreAbilityCall`, `PostAbilityCall`, `SessionStop`.
- `enabled`  
  - `true` / `false`. Disabled hooks are ignored by the orchestrator.
- `blocking`  
  - Controls whether hook failure or its decisions may block the main flow:
    - For `PromptSubmit` and `PreAbilityCall`, blocking hooks are run **synchronously** and may affect whether planning or execution proceeds.
    - For `PostAbilityCall` and `SessionStop`, `blocking` is usually `false` (AI‑facing behavior is disallowed; blocking only affects infra flows).

### 2.2 Matching rules

Hooks may limit when they run with a `match` block:

```yaml
match:
  ability_scope: "*"                # glob over ability ids
  min_duration_ms: 100              # only for slow calls
  only_if_changed_paths:
    - "services/**"
    - "apps/**"
```

Common fields:

- `ability_scope`  
  - Glob or list of globs over ability ids (for ability‑related events).
- `min_duration_ms`  
  - For `PostAbilityCall`, only run on calls slower than this threshold.
- `only_if_changed_paths`  
  - For `SessionStop`, only run if any changed files match these patterns.

### 2.3 Handler spec

The `handler` block defines how to run the hook:

```yaml
handler:
  kind: script
  command: ./scripts/hooks/session_stop_build_check.sh
```

- `kind`  
  - Currently `script`. Future kinds might include `builtin` or `http`.
- `command`  
  - Script or executable path. The orchestrator:
    - Sends the event context as JSON on stdin.
    - Expects a JSON `HookResult` on stdout.

### 2.4 Effects (human‑readable)

The `effects` list is **descriptive only**: it documents what the handler intends to do. It is not an execution spec.

Example:

```yaml
effects:
  - run_local_build_checks
  - summarize_errors_to_user
```

This helps both humans and the AI developer understand the hook’s purpose when browsing `.system/hooks/`.

---

## 3. Contexts, Hook Results, and Hook Signals

Handlers receive structured JSON context and return structured results. The orchestrator and AI developer both rely on this structure to reason about hook behavior.

### 3.1 Event contexts (overview)

Each event type has its own context shape; here we only outline the key fields.

#### PromptSubmitContext (AI‑facing)

```ts
interface PromptSubmitContext {
  event_type: "PromptSubmit";
  session_id: string;
  user_raw_input: string;
  current_files?: string[]; // optional snapshot of interesting files
  // Additional session metadata as needed
}
```

Hooks for this event may emit **routing hints** and **normalized intent**.

#### PreAbilityCallContext (AI‑facing)

To make PreAbilityCall useful for both AI and background jobs, we include the caller:

```ts
type InvocationSource =
  | "ai_session"
  | "background_job"
  | "ci_pipeline"
  | "manual_cli";

interface PreAbilityCallContext {
  event_type: "PreAbilityCall";
  session_id?: string;
  ability_id: string;
  args_summary?: string;
  caller: {
    source: InvocationSource;
    session_id?: string; // if ai_session
    job_id?: string;     // if background_job/ci_pipeline
    triggered_by?: string;
  };
  // Optional: target paths, environment, risk level, etc.
}
```

Hooks for this event may **allow, deny, or require human approval** for a specific ability call.

#### PostAbilityCallContext (infra‑facing)

```ts
interface PostAbilityCallContext {
  event_type: "PostAbilityCall";
  session_id?: string;
  ability_id: string;
  status: "success" | "failure" | "partial";
  duration_ms?: number;
  input_summary?: string;
  output_summary?: string;
  tool_call_path?: string[]; // nested calls if any
  error_message?: string;
}
```

In many repos, abilities are implemented as local CLI tools or scripts. The runtime is allowed to capture the tool's stdout/stderr and either:

- embed a truncated excerpt into `output_summary`, or
- store the full stdout in a log file and expose a pointer (e.g. in an
  implementation-specific field such as `stdout_log_path`).

PostAbilityCall hooks may inspect these values to compute telemetry and quality metrics (for example, detecting frequent error-like outputs or empty results), but this remains an infra-only concern. Any conclusions based on stdout should be written to logs or caches, not fed back into the current AI planning loop.

Hooks here are **infra‑only**: they log, track usage, and update caches. They do **not** send AI‑facing signals for the current turn.

#### SessionStopContext (infra‑facing)

```ts
interface SessionStopContext {
  event_type: "SessionStop";
  session_id: string;
  changed_files?: string[];
  abilities_used?: string[];
  // Additional context (e.g., per-service stats)
}
```

Hooks may run builds/tests and persist summaries, again as infra‑only operations.

### 3.2 HookResult: raw handler output

Each handler writes a JSON object to stdout. The orchestrator merges all results for an event.

At the low level, a `HookResult` might look like:

```ts
interface HookResult {
  // For infra: logs, telemetry, cache hints
  logs?: string[];

  // For internal infra use (e.g. marking usage recorded)
  usage_recorded?: boolean;

  // For AI-facing events (PromptSubmit / PreAbilityCall) only:
  hook_signals?: HookSignal[];

  // Optional error reporting
  error?: {
    code: string;
    message: string;
  };
}
```

- For **PromptSubmit** and **PreAbilityCall**:
  - Only `hook_signals` (plus optional logs/error) are relevant.
- For **PostAbilityCall** and **SessionStop**:
  - Only infra fields (logs, usage_recorded, infra‑specific metadata) are relevant.
  - Any AI‑facing fields (e.g. `hook_signals`) must be ignored by the orchestrator.

### 3.3 HookSignal: AI‑visible signals

To avoid every hook inventing its own schema, we standardize **hook signals** that the runtime passes to the AI developer loop.

```ts
type HookSignalKind =
  | "routing_hint"      // PromptSubmit: ability/doc suggestions
  | "normalized_intent" // PromptSubmit: structured task description
  | "ability_guard";    // PreAbilityCall: allow/deny/require-human

interface HookSignal {
  source_event: "PromptSubmit" | "PreAbilityCall";
  hook_id: string;
  kind: HookSignalKind;
  code: string; // machine-readable short code
  severity?: "info" | "warning" | "error";
  payload: any; // structured per kind (see below)
}
```

#### PromptSubmit hook signals

PromptSubmit hooks may emit:

- `routing_hint`:

  ```jsonc
  {
    "source_event": "PromptSubmit",
    "hook_id": "skill_activation_prompt",
    "kind": "routing_hint",
    "code": "ROUTE_SUGGESTION",
    "payload": {
      "suggested_abilities": [
        {
          "id": "refactor_code",
          "reason": "User asked to clean up TypeScript code.",
          "confidence": 0.92
        }
      ],
      "suggested_documents": [
        {
          "path": ".system/docs/pre_commit.md",
          "reason": "User mentioned pre-commit checks."
        }
      ]
    }
  }
  ```

- `normalized_intent`:

  ```jsonc
  {
    "source_event": "PromptSubmit",
    "hook_id": "intent_normalizer",
    "kind": "normalized_intent",
    "code": "INTENT_PARSED",
    "payload": {
      "scope": "dev_workflow",
      "topic": "pre_commit_checks",
      "stage": "understand",
      "raw_input_snippet": "Before I commit, what checks should I run?"
    }
  }
  ```

The AI developer is expected to treat these as **high‑confidence hints** for routing and planning on the first turn.

#### PreAbilityCall hook signals

PreAbilityCall hooks may emit `ability_guard` signals:

```jsonc
{
  "source_event": "PreAbilityCall",
  "hook_id": "prod_config_guard",
  "kind": "ability_guard",
  "code": "ABILITY_DENIED",
  "severity": "error",
  "payload": {
    "ability_id": "edit_file",
    "reason": "Editing prod config files is not allowed from ai_session.",
    "require_human": true
  }
}
```

The AI and runtime interpret these as guardrail decisions:

- `ABILITY_ALLOWED` – normal execution may proceed.
- `ABILITY_DENIED` with `require_human = true` – stop auto‑run and show the reason to a human.
- `ABILITY_DENIED` with `require_human = false` – mark the ability as “disabled for this session” and plan around it.

For background and CI callers, the runtime may act on `ability_guard` signals without involving the AI at all.

### 3.4 Event‑level constraints

The orchestrator enforces per‑event constraints on `HookResult` and `HookSignal`:

- For `PromptSubmit` and `PreAbilityCall`:
  - `hook_signals` are allowed and passed to the AI in the turn context.
- For `PostAbilityCall` and `SessionStop`:
  - Any AI‑facing fields (e.g. `hook_signals`) are ignored (and optionally logged as a misconfiguration).

This ensures that **only decision‑before events may influence the current turn**, while post‑processing events remain infra‑only.

---

## 4. Agent Runtime / Orchestrator Behavior

The runtime is the **single source of truth** for when events are emitted and how hooks are run. The AI developer never calls hooks directly; it only sees:

- Inputs that have already been processed by blocking hooks.
- Structured `hook_signals` attached to the turn context.
- A small **control message** indicating that a turn is ready to begin.

### 4.1 Turn pipeline and blocking hooks

High‑level pseudocode for the main loop:

```ts
async function handleUserInput(rawInput: string, session: SessionState) {
  // 1. Emit PromptSubmit and run blocking hooks
  const promptCtx = buildPromptSubmitContext(rawInput, session);
  const promptResults = await runHooks("PromptSubmit", promptCtx, { blockingOnly: true });
  const promptSignals = collectHookSignals(promptResults);

  // 2. Build turn context for the AI developer
  const turnContext = {
    user_raw_input: rawInput,
    hook_signals: promptSignals,
    // ... other session state (files, history, etc.)
  };

  // 3. Emit control message: turn is ready
  const startControl = {
    type: "control",
    turn_ready: true,
    source_event: "PromptSubmit",
    hook_signals: promptSignals,
  };

  // 4. Call the AI planner with the ready signal + context
  const plan = await callAiPlanner(startControl, turnContext);

  // 5. Execute planned ability calls
  for (const step of plan.steps) {
    const preCtx = buildPreAbilityCallContext(step, session);
    const preResults = await runHooks("PreAbilityCall", preCtx, { blockingOnly: true });
    const preSignals = collectHookSignals(preResults);

    const abilityControl = {
      type: "control",
      turn_ready_for_ability: true,
      ability_id: step.ability_id,
      source_event: "PreAbilityCall",
      hook_signals: preSignals,
    };

    const guardDecision = interpretAbilityGuards(preSignals, step);

    if (!guardDecision.allow) {
      await handleGuardrailDecision(guardDecision, step, session);
      continue;
    }

    const result = await executeAbility(step);

    const postCtx = buildPostAbilityCallContext(step, result, session);
    // PostAbilityCall hooks are infra-only
    runHooks("PostAbilityCall", postCtx, { blockingOnly: false }); // fire-and-forget or awaited infra
  }

  // 6. If the session should stop, run SessionStop hooks (infra-only)
  if (session.shouldStop) {
    const stopCtx = buildSessionStopContext(session);
    runHooks("SessionStop", stopCtx, { blockingOnly: false });
  }
}
```

Key points:

- **PromptSubmit** & **PreAbilityCall**:
  - Blocking hooks are run synchronously.
  - Their `hook_signals` are collected and passed to the AI with a **turn‑ready control message**.
- **PostAbilityCall** & **SessionStop**:
  - Hooks run as infra; their outputs never modify the current turn’s plan.

### 4.2 Execution start signals for the AI

To remove ambiguity for the AI developer, the runtime always sends an explicit **start signal** when a turn or ability execution is ready to proceed.

Examples:

- For planning a new turn:

  ```jsonc
  {
    "type": "control",
    "turn_ready": true,
    "source_event": "PromptSubmit",
    "hook_signals": [ /* routing_hint / normalized_intent */ ]
  }
  ```

- For executing a specific ability step:

  ```jsonc
  {
    "type": "control",
    "turn_ready_for_ability": true,
    "ability_id": "refactor_code",
    "source_event": "PreAbilityCall",
    "hook_signals": [ /* ability_guard or empty */ ]
  }
  ```

The AI must treat these control messages as the **only legitimate start signal** for:

- Beginning a planning step (PromptSubmit).
- Executing a planned ability call (PreAbilityCall).

If `turn_ready` / `turn_ready_for_ability` is absent or false, the AI should assume that hooks are still running or that the call is not allowed.

### 4.3 Background and non‑AI callers

Abilities may also be invoked by:

- Cron / background jobs.
- CI pipelines.
- Manual CLI scripts.

To keep PreAbilityCall hooks effective in these scenarios:

- All ability invocations must use a shared **ability runner API or CLI**, e.g.:
  - `execute_ability(ability_id, args, invocation_source)`
  - `ability_runner --ability apply_migration --source background_job`
- The runner:
  - Builds a `PreAbilityCallContext` with `caller.source` set appropriately.
  - Runs `runHooks("PreAbilityCall", ctx)` synchronously.
  - Interprets `ability_guard` signals to allow/deny execution.

This guarantees that guardrails and pre‑flight checks are applied **consistently** across AI and non‑AI callers.

---

## 5. Hook Scaffolding (for humans and AIs)

Writing hook YAML and handler scripts by hand can be tedious. To make hooks easy to adopt, we recommend a dedicated **scaffold workflow**, e.g. an ability called `scaffold_hook`.

Typical scaffold flow:

1. A human or the AI describes the desired hook in natural language:
   - Event type (`PromptSubmit`, `PreAbilityCall`, etc.).
   - High‑level intent (logging, guardrail, routing hint).
   - Whether it should be blocking.
   - Matching rules (ability_scope, changed_paths, etc.).
2. The scaffold workflow asks follow‑up questions to fill in required fields.
3. It generates:
   - `.system/hooks/<id>.yaml` (following the schema above).
   - A handler stub under `./scripts/hooks/<id>.{py|sh|ts}`.
4. It calls a **registration/validation script**, e.g.:
   - `./scripts/hooks/register_hooks.py --validate`
   - to ensure YAML parses, event_type is valid, and handler path exists.
5. It returns a short summary of what was created and how to test it.

Important constraints:

- The scaffold can be AI‑driven, but the **registration and runtime logic remain deterministic scripts**.
- Broken hooks should be caught by the registration/validation step, not in the middle of an AI session.

---

## 6. Patterns and Recipes

This section outlines common patterns to implement with hooks.

### 6.1 Telemetry and usage tracking (PostAbilityCall)

- **Event**: `PostAbilityCall`  
- **blocking**: `false`  
- **Goal**: centralize telemetry about ability use.

Typical YAML:

```yaml
id: ability_usage_tracker
event_type: PostAbilityCall
enabled: true
blocking: false

summary: >
  Track each ability call and update ability_usage.json. For slow calls,
  append to slow_abilities.log.

match:
  ability_scope: "*"

handler:
  kind: script
  command: ./scripts/hooks/ability_usage_tracker.py

effects:
  - update_usage_cache: .system/cache/ability_usage.json
  - append_to: .system/logs/slow_abilities.log
```

The handler:

- Reads the context (`ability_id`, `status`, `duration_ms`, etc.).
- Optionally inspects a truncated view or log pointer for the tool's stdout (via `output_summary` or a repo-specific field) to derive basic quality metrics.
- Appends a telemetry row to a log or updates a small JSON cache.
- Returns a `HookResult` with only infra‑relevant fields (e.g. `usage_recorded: true`).

### 6.2 Ability activation hints (PromptSubmit)

- **Event**: `PromptSubmit`  
- **blocking**: `true`  
- **Goal**: suggest relevant abilities/documents and normalize intent.

Typical YAML:

```yaml
id: skill_activation_prompt
event_type: PromptSubmit
enabled: true
blocking: true

summary: >
  Analyze the incoming user message and suggest a small set of relevant
  abilities and documents as routing hints.

handler:
  kind: script
  command: ./scripts/hooks/skill_activation_prompt.sh

effects:
  - suggest_abilities
  - emit_normalized_intent
```

The handler:

- Reads `user_raw_input` and optionally other session metadata.
- Produces `HookSignal` objects of kind `routing_hint` and `normalized_intent`.
- Returns them in `hook_signals` inside the `HookResult`.

The runtime merges these signals and includes them in the `turn_ready` control message for the AI.

### 6.3 Safety and permission checks (PreAbilityCall)

- **Event**: `PreAbilityCall`  
- **blocking**: `true`  
- **Goal**: prevent dangerous or misconfigured ability calls.

Typical YAML:

```yaml
id: prod_config_guard
event_type: PreAbilityCall
enabled: true
blocking: true

summary: >
  Prevent editing of production configuration files from AI sessions.

match:
  ability_scope: "edit_file"

handler:
  kind: script
  command: ./scripts/hooks/prod_config_guard.py

effects:
  - enforce_prod_safety
```

The handler:

- Checks `caller.source` and target paths (from context or args).
- If unsafe, emits an `ability_guard` signal with `ABILITY_DENIED` and a reason.
- If safe, emits `ABILITY_ALLOWED` (or no signal).

The runtime interprets `ability_guard` signals to:

- Allow or deny the ability call.
- Potentially stop auto‑run and surface a message to a human.

### 6.4 Build/test aggregation at SessionStop

- **Event**: `SessionStop`  
- **blocking**: usually `false` (for AI)  
- **Goal**: run per‑service build/test for changed components and summarize results.

Typical YAML:

```yaml
id: session_stop_build_check
event_type: SessionStop
enabled: true
blocking: false

summary: >
  On session stop, run lightweight build or test commands for changed
  services and summarize any blocking issues.

match:
  only_if_changed_paths:
    - "services/**"
    - "apps/**"

handler:
  kind: script
  command: ./scripts/hooks/session_stop_build_check.sh

effects:
  - run_local_build_checks
  - summarize_errors_for_humans
```

The handler:

- Groups `changed_files` into services/apps.
- Runs corresponding build/test commands.
- Writes a summary report to a log or file for humans / future sessions.

This hook does **not** alter decisions already made in the session; it only prepares data for follow‑up work.

---

## 7. Observability, Debugging, and Governance

Hooks can improve observability but can also become a source of confusion if not governed carefully.

### 7.1 Listing hooks

Provide a simple command or ability to list all registered hooks, e.g.:

```bash
./scripts/hooks/list
```

Example output:

```text
id                      event_type      enabled blocking match_summary
----------------------  -------------   ------- -------- --------------------------
skill_activation_prompt PromptSubmit    true    true     all sessions
prod_config_guard       PreAbilityCall  true    true     ability=edit_file
ability_usage_tracker   PostAbilityCall true    false    ability=* 
session_stop_build_check SessionStop    true    false    paths=services/**,apps/**
```

This helps both humans and the AI quickly understand what infra is in play.

### 7.2 Logging

Hooks should log enough to debug issues without overwhelming logs:

- At least log:
  - `hook_id`
  - `event_type`
  - `status` (success/failure)
  - `duration_ms`
- For blocking hooks, log when they **deny** or **require human** decisions.

### 7.3 Handling failures and noise

- **Blocking hooks**:
  - On failure, the runtime may:
    - Treat it as a soft failure and skip the hook (logging an error), or
    - Fail the event entirely, depending on configuration.
  - For guardrails, prefer explicit `ability_guard` signals over throwing exceptions.
- **Non‑blocking hooks**:
  - Failures should not block AI progress.
  - Log errors and continue.

If a hook becomes too noisy (e.g., constantly denying calls the team considers safe), treat it as a design issue and revisit its logic or matching rules.

### 7.4 Avoiding hook sprawl

To keep `.system/hooks/` manageable:

- Use clear, descriptive ids (`prod_config_guard`, `ability_usage_tracker`).
- Group hooks logically by event_type or purpose when browsing.
- Periodically review and prune unused or obsolete hooks.
- Prefer shared patterns (logging, simple guardrails) over deeply nested hook logic.

---

## 8. Design Principles (Recap)

- **AI‑centric but infra‑driven**  
  - The AI is the main developer; the runtime and hooks provide guardrails and automation around it.

- **Event‑driven and explicit**  
  - Hooks attach to well‑defined events. Only `PromptSubmit` and `PreAbilityCall` may directly influence the AI’s current turn.

- **Blocking vs non‑blocking is a first‑class concept**  
  - Blocking hooks run synchronously and may change whether a turn or ability proceeds. Non‑blocking hooks never affect the current decision loop.

- **Repository‑local and incremental**  
  - Hooks live in `.system/hooks/` alongside docs, abilities, and scripts. They can be adopted one by one.

- **Scaffoldable and maintainable**  
  - A dedicated `scaffold_hook` flow makes it easy for humans and AIs to create hooks that are valid by construction.

- **Predictable AI collaboration**  
  - The AI only acts after receiving explicit `turn_ready` control signals, with `hook_signals` attached. It never has to guess whether hooks have finished, nor does it see mixed messages from post‑processing events.

This design aims to make hooks a **small, predictable extension point** that helps the AI developer work safely and effectively, without entangling core workflows in ad‑hoc prompt glue.
