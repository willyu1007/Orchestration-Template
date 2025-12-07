# Hooks

## 0. Purpose and Design Goals

In this repository template, "hooks" are an event-driven infrastructure that lives alongside the AI developer. The AI model is the primary author of code and automation; hooks give it an environment that can:

- Observe what is happening (tool calls, file edits, session boundaries).
- Enforce lightweight guardrails at predictable points.
- Automate repetitive checks (build, tests, formatting).
- Accumulate structured telemetry that can be used by routing and documentation.

A good hook system for this project should be:

- **AI-friendly** – small, local, readable definitions that a code model can safely modify.
- **Predictable** – hooks trigger at well-defined lifecycle events, not whenever a prompt happens to remember them.
- **Optional by default** – the repo is usable even with all hooks disabled.
- **Composable** – multiple hooks can react to the same event without knowing about each other.
- **Cheap to maintain** – adding or removing a hook should not require editing the core orchestrator.

## 1. High-level Architecture

At a high level there are three main actors:

1. **AI developer (code model loop)**  
   - Plans work, edits files, calls tools/abilities, and decides when a session/phase is done.
   - Interacts through a small set of actions: `edit_files`, `run_tests`, `use_ability`, `declare_session_stop`, etc.

2. **Agent Runtime / Orchestrator**  
   - A long-running process that wraps the AI developer.  
   - Responsible for:
     - Emitting lifecycle **events** whenever the AI performs certain actions.
     - Running all **hooks** that match those events.
     - Aggregating `HookResult`s and feeding them back to the AI as environment feedback.

3. **Hooks (YAML + handlers)**  
   - Repository-local definitions living in `.system/hooks/*.yaml`.  
   - Each hook describes:
     - Which event type it listens to.
     - Optional match conditions (ability id, file patterns, environment, etc.).
     - The handler to execute and what kind of side effects it may produce.

Visually:

```text
User ⟷ AI Developer (LLM loop)
           │
           ▼
    Agent Runtime (orchestrator)
           │ emits events
           ▼
  +--------+-------------------------+
  | .system/hooks/*.yaml             |
  | + handlers (scripts / builtins)  |
  +----------------------------------+
           │ returns HookResults
           ▼
      Feedback to AI developer
        (messages, actions, hints)
```

The rest of this document spells out:

- Which **events** exist and when they fire.
- How hooks are **declared** and **executed**.
- How to scaffold new hooks with the help of the AI.
- Recommended patterns, debugging tips, and design principles.

## 2. Event Model and Lifecycle

Hooks are triggered by discrete **events** emitted by the runtime. Events are not arbitrary strings; they correspond to concrete points in the AI development lifecycle.

The core event types:

1. **`PromptSubmit`**  
   - When a new user message (or high-level instruction) enters the system.  
   - Typical uses:
     - Early intent extraction and routing hints.
     - Ability activation suggestions from natural language.
     - Logging user-level tasks or tickets.

2. **`PreAbilityCall`**  
   - Just before the runtime calls a concrete ability/tool on behalf of the AI.  
   - Typical uses:
     - Permission and risk checks for destructive actions.
     - Environment validation (e.g. required services running).
     - Parameter sanity checks.

3. **`PostAbilityCall`**  
   - Right after an ability/tool finishes.  
   - Typical uses:
     - Telemetry and usage statistics.
     - Slow-call detection and performance alerts.
     - Lightweight quality checks on outputs.

4. **`SessionStart` / `SessionEnd` / `SessionStop`**  
   - `SessionStart`: beginning of a new focused work session.  
   - `SessionEnd`: natural end of the conversation, no additional checks.  
   - `SessionStop`: explicit "phase boundary" declared by the AI (e.g. "I'm done with this refactor").  
   - Typical uses:
     - Aggregate build/test runs over all changed files.
     - Generating summaries or status updates.
     - Recording session-level metrics.

> **Important:** the runtime, not the AI, is responsible for emitting these events at the correct times.  
> The AI can request "stop" or "run extra checks", but it cannot silently skip a mandatory event like `PostAbilityCall`.

A rough timeline for a typical coding session:

```text
PromptSubmit
  └─ PromptSubmit hooks
      └─ AI plans work, chooses abilities
          └─ For each step:
                PreAbilityCall
                  └─ Pre hooks
                      └─ ability/tool runs
                          └─ PostAbilityCall
                              └─ Post hooks
... repeat ...
AI declares session_stop
  └─ SessionStop hooks (build/test/summary/etc.)
```

Each event carries a structured **Context** object that describes what is happening; handlers read this context and produce a **HookResult**.

## 3. Hook Declaration (YAML schema)

Hooks are declared via small YAML files that are easy for both humans and code models to read and modify.

### 3.1 Layout

- Directory: `.system/hooks/`
- One file per hook, for example:
  - `.system/hooks/skill_activation_prompt.yaml`
  - `.system/hooks/post_tool_use_tracker.yaml`
  - `.system/hooks/session_stop_build_check.yaml`

Each file defines exactly one hook.

### 3.2 Core fields

The typical schema (conceptual, not strict):

```yaml
id: post_tool_use_tracker          # globally unique within the repo
event_type: PostAbilityCall        # which event this hook listens to
enabled: true                      # can be toggled per environment

summary: >
  Short human/AI readable description of what this hook does and why it exists.

blocking: false                    # if true, failures can block the main flow

match:                              # optional filters to reduce when this hook fires
  ability_scope: "*"                # glob or list of ability ids
  min_duration_ms: 0                # only run for calls slower than this
  only_if_changed_paths:            # for SessionStop-like hooks
    - "services/**"
    - "apps/**"

handler:
  kind: script                      # or builtin
  command: ./scripts/hooks/post_tool_use_tracker.py

effects:                            # declarative hints about side effects
  - append_to: .system/logs/tool_call.log
  - update_usage_cache: .system/cache/ability_usage.json
```

#### `id`

A short, kebab-case name, unique within the repo. Used in logs and debugging.

#### `event_type`

One of the supported event types: `PromptSubmit`, `PreAbilityCall`, `PostAbilityCall`, `SessionStart`, `SessionEnd`, `SessionStop`, etc.

#### `enabled`

Boolean flag. The runtime must treat hooks as **optional by default**:

- If all hooks are disabled, the repo still works.
- Enabling/disabling a hook never requires changing orchestrator code.

#### `blocking`

Controls how the runtime treats handler errors:

- `blocking: false` (default): handler failures are non-fatal; the runtime logs them and continues. Ideal for telemetry, suggestions, and non-critical automation.
- `blocking: true`: handler failures or explicit "block" responses can prevent the underlying action from completing (e.g. stopping a dangerous ability call or failing a SessionStop build check).

#### `match`

A small set of filters that narrow when the hook should run. Common fields:

- `ability_scope`: a glob or list specifying which abilities/tools this hook cares about.
- `min_duration_ms`: minimum observed duration before the hook is triggered (useful for slow-call detection).
- `only_if_changed_paths`: list of path globs; for session-level hooks that only make sense when certain parts of the repo changed.

The runtime resolves these filters against the event context before invoking the handler.

#### `handler`

Describes how the hook logic is executed.

Typical form (script handler):

```yaml
handler:
  kind: script
  command: ./scripts/hooks/session_stop_build_check.sh
```

Conventions:

- Handlers read a single JSON **Context** object from `stdin`.
- Handlers write a single JSON **HookResult** object to `stdout`.
- Exit status `0` → success; non-zero → handler error (treated according to `blocking`).

#### `effects` (declarative hints)

`effects` is not executed by the runtime; it's documentation for humans and AI:

- Helps a code model understand the intent and side effects of the hook.
- Can be used by tooling to generate dashboards or to cross-check expected behavior.

Examples:

```yaml
effects:
  - append_to: .system/logs/hooks.log
  - emit_metrics: "hook.execution.duration"
  - suggest_abilities
  - emit_new_intent_for_knowledge_routing
```

## 4. Context and Result Objects

Each event type has a corresponding **Context** schema. The exact fields may evolve, but the spirit is:

```ts
type PostAbilityCallContext = {
  event_type: "PostAbilityCall";
  session_id: string;
  ability_id: string;
  operation_key?: string;
  status: "success" | "error" | "cancelled";
  duration_ms?: number;
  input_summary?: string;
  output_summary?: string;
  tool_call_path?: string[];
  changed_files?: string[];
  // ... other repo- or ability-specific fields
};
```

Handlers produce a `HookResult`:

```ts
type HookResult = {
  logs?: string[];                 // log lines for hook-specific logs
  messages_to_user?: string[];     // user-facing messages (AI can surface or summarize)
  suggested_abilities?: {
    id: string;
    reason: string;
    confidence?: number;
  }[];
  suggested_documents?: {
    path: string;
    reason: string;
  }[];
  follow_up_actions?: any[];       // e.g. open_file, run_command, etc.
  new_intent?: any;                // structured intent object for knowledge routing
  usage_recorded?: boolean;        // telemetry flags
  block_execution?: boolean;       // only meaningful when blocking: true
};
```

The runtime:

1. Collects all `HookResult`s for a given event.
2. Logs their `logs`.
3. Merges user messages and suggestions.
4. If any blocking hook sets `block_execution`, it halts the underlying action and surfaces a clear explanation back to the AI.

## 5. Agent Runtime / Orchestrator Behavior

Although the AI is the primary developer, the runtime controls **when** hooks are triggered. A simplified TypeScript-like pseudocode:

```ts
async function handleUserInput(rawInput: string, session: SessionState) {
  // 1) PromptSubmit
  const promptCtx = buildPromptSubmitContext(rawInput, session);
  const promptResults = await runHooks("PromptSubmit", promptCtx);
  const routingContext = mergeHookResultsIntoRouting(promptResults, session);

  // 2) Ask the AI to plan next steps, given routing hints
  const plan = await callCodeModel({
    kind: "route_and_plan",
    rawInput,
    routingContext,
    availableAbilities,
  });

  for (const step of plan.steps) {
    // 3) PreAbilityCall
    const preCtx = buildPreAbilityCallContext(step, session);
    const preResults = await runHooks("PreAbilityCall", preCtx);
    if (preResults.some(r => r.block_execution)) {
      continue; // or surface an error to the AI/user
    }

    // 4) Execute the ability/tool
    const abilityResult = await executeAbility(step);

    // 5) PostAbilityCall
    const postCtx = buildPostAbilityCallContext(step, abilityResult, session);
    await runHooks("PostAbilityCall", postCtx);

    // 6) Feed any messages/actions back into the AI loop
    updateSessionFromHookResults(session, postCtx);
  }

  if (session.shouldStop) {
    const stopCtx = buildSessionStopContext(session);
    await runHooks("SessionStop", stopCtx);
  }
}
```

Key points:

- The runtime is a **long-running process** (or equivalent service) that owns the event loop.
- Hooks are never called "directly" by the AI. Instead, they are injected at lifecycle boundaries.
- The AI can still **request** extra checks via dedicated abilities (e.g. `manual_run_session_stop_hooks`), but it cannot bypass mandatory events.

## 6. Hook Scaffolding (for humans and AI)

Writing YAML and handler scripts by hand is tedious. This repo assumes the existence of a **hook scaffolding workflow**, typically exposed as an ability like `scaffold_hook`.

### 6.1 Goals

- Reduce friction for adding or modifying hooks.
- Make it easy for the AI itself to propose and implement new hooks.
- Keep the core runtime and registration logic deterministic and simple.

### 6.2 Scaffold workflow

A typical scaffold flow:

1. **Describe the desired behavior** in natural language:  
   > "Whenever an ability call takes more than 2 seconds, log it to `.system/logs/slow_abilities.log` and update a usage cache."

2. The scaffold workflow asks a few structured questions:
   - Which event_type? (`PostAbilityCall`)
   - Should this hook be blocking? (usually `false`)
   - Scope: all abilities or a subset?
   - Preferred handler language (Python, TS, shell)?
   - Log file and cache file paths?

3. The workflow generates:
   - A new `.system/hooks/<id>.yaml` file matching the schema.
   - A handler script stub in `./scripts/hooks/`.

4. It then runs the hooks **registration/validation** script to ensure:
   - YAML parses correctly.
   - Event type is valid.
   - Handler file exists and is executable.

5. If validation passes, it summarizes the new hook for the user/AI and suggests a small test.

Scaffolding is implemented as a separate ability or tool; the core runtime and registration logic remain non-LLM scripts.

## 7. Patterns and Recipes

This section captures common patterns for hooks.

### 7.1 Telemetry / usage tracking (`PostAbilityCall`)

**Goal:** track every ability/tool call to a central log and usage cache.

- `event_type`: `PostAbilityCall`
- `blocking`: `false`
- `match`: `ability_scope: "*"`; maybe `min_duration_ms: 0`
- Handler:
  - Append an entry to `.system/logs/tool_call.log`.
  - Update `.system/cache/ability_usage.json` with counters (per ability, per status, etc.).

This data can later be used to:

- Improve routing (which abilities are actually used).
- Detect dead or flaky abilities.
- Generate documentation or changelogs.

### 7.2 Ability activation hints (`PromptSubmit`)

**Goal:** when a user (or higher-level agent) describes a task, suggest a small set of relevant abilities and knowledge documents.

- `event_type`: `PromptSubmit`
- `blocking`: `false`
- Handler:
  - Analyze `user_raw_input` and the current ability catalog.
  - Return `suggested_abilities` and an optional structured `new_intent` for knowledge routing.

The runtime merges these suggestions into the routing context passed to the AI developer.

### 7.3 Safety / permission checks (`PreAbilityCall`)

**Goal:** gate high-risk operations on simple policies.

Examples:

- Deleting resources.
- Running commands outside the repo.
- Large-scale modifications across multiple services.

Typical configuration:

- `event_type`: `PreAbilityCall`
- `blocking`: `true`
- `match`: `ability_scope` limited to dangerous abilities.

The handler can inspect the context (arguments, environment, current branch, etc.) and decide to:

- Allow the call (do nothing special).
- Block it and return a clear message.
- Request an explicit confirmation step from the human operator.

### 7.4 Build/test aggregation (`SessionStop`)

**Goal:** before a refactor or feature is considered "done", run a minimal set of builds/tests for the impacted services.

- `event_type`: `SessionStop`
- `blocking`: often `false` initially, may become `true` when stable.
- `match.only_if_changed_paths`: globs for `services/**`, `apps/**`, etc.
- Handler:
  - Derive a set of services from `changed_files`.
  - Run `npm run build:<service>` or equivalent commands.
  - Summarize results in `messages_to_user` and `follow_up_actions`.

The AI can then fix failures or summarize status for a human reviewer.

## 8. Observability, Debugging, and Governance

A hooks system only works if it is observable and manageable.

### 8.1 Listing hooks

Provide a simple command or ability such as:

- `list_hooks` or `./scripts/hooks/list`

Expected output (conceptual):

```text
id                        event_type       enabled  blocking  match_summary
------------------------  ---------------  -------  --------  ------------------------------
skill_activation_prompt   PromptSubmit     true     false     ability_scope=*
post_tool_use_tracker     PostAbilityCall  true     false     ability_scope=*
session_stop_build_check  SessionStop      true     false     changed_paths=services/**,apps/**
...
```

This helps both humans and the AI understand "what is wired where".

### 8.2 Logging

Each hook handler can emit `logs` in its `HookResult`. The runtime should:

- Append them to a central hook log (e.g. `.system/logs/hooks.log`).
- Always include at least: `hook_id`, `event_type`, `status`, `duration_ms`.

In development, consider a `HOOKS_DEBUG=1` switch that makes the runtime:

- Dump full Context and HookResult to a separate trace log.
- Surface handler errors more aggressively.

### 8.3 Handling failure and noise

Guidelines:

- Start non-blocking. Turn on `blocking: true` only after a hook has proved stable.
- For noisy validation hooks:
  - Tighten `match` conditions (ability id, path patterns, environments).
  - Add explicit skip rules inside the handler.
- If a hook becomes obsolete, disable it first; delete later when you are confident it is not needed.

### 8.4 Avoiding hook sprawl

Over time hooks can proliferate. To keep things healthy:

- Adopt clear naming conventions (`<event>_<brief_purpose>`).
- Group related handler scripts under `./scripts/hooks/`.
- Prefer a few well-factored hooks over many overlapping ones.
- Periodically review `list_hooks` output and prune low-value ones.

## 9. Design Principles (Recap)

To close, the hooks system in this repo follows a few non-negotiable principles:

- **AI-centered**: the AI is the primary developer; hooks exist to give it a safer, more automated environment.
- **Optional by default**: the system must function without any hooks; hooks add value, not hard dependencies.
- **Small and explicit**: each hook does one clear job and is described in a small, repository-local YAML file.
- **Repository-local**: definitions and handlers live with the code they affect and are discoverable by a code model.
- **Event-driven and deterministic**: the runtime, not the AI, decides when events fire and which hooks run.
- **Decoupled from routing**: ability and knowledge routing stay simple; hooks add instrumentation, automation, and optional guardrails around them.
- **Incrementally adoptable**: you can start with telemetry-only hooks, then gradually introduce stronger checks and automation.

With this structure, hooks can be gradually introduced and evolved as the AI developer gets more capable, while keeping the overall system fast, predictable, and maintainable.
