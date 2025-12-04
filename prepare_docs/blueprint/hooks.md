# Hooks

## 0. Purpose and Design Goals

Hooks are an event-driven extension layer that sits next to ability routing, knowledge routing, and the execution script.

Most of the repository infrastructure is designed around **semantic matching**: the AI reads routing documents, understands natural-language descriptions, and plans which abilities and knowledge documents to use. Hooks add a thin, deterministic layer on top of that semantic core.

The goals are:

- **Increase AI development efficiency**
  - Provide fast, structured signals (e.g., “this ability is relevant”, “run this check now”) instead of dumping more unstructured text into context.
  - Automate repetitive operations such as logging, validation, and quality checks.
- **Stay lightweight and predictable**
  - Hooks are small, explicit, and easy for a code model to read and modify.
  - The system must keep working even if all hooks are disabled.
- **Decouple optional safety / quality checks**
  - Heavy checks and “soft guardrails” are implemented as hooks, not baked into ability routing.
  - The core routing logic remains focused on finding and executing abilities and loading knowledge.

In short: hooks offer **deterministic, event-driven instrumentation and automation** without complicating the core routing systems.

---

## 1. Concepts and Terminology

### 1.1 Events, hooks, and handlers

- **Event**  
  A discrete signal from the orchestration system or environment. Examples:
  - `UserPromptSubmit` – a user message or prompt is submitted.
  - `PreAbilityCall` – the execution script is about to call an ability.
  - `PostAbilityCall` – the execution script has finished calling an ability.
  - `SessionStop` – a session is being stopped (e.g., user presses “Stop”).
- **Hook**  
  A configuration object that binds one or more events to a handler. It describes:
  - Which `event_type` it listens to.
  - Optional matching conditions (ability id, file patterns, etc.).
  - The handler to execute and what effects are expected.
- **Handler**  
  A script or function that receives **structured event context** and returns a **structured result**. Typical responsibilities:
  - Instrumentation (logging to `tool_call.md`-like locations).
  - Suggesting abilities or documents.
  - Triggering local checks and CI-like operations.
  - Emitting structured intents for knowledge routing.
- **Context**  
  A JSON-like structure describing what happened, such as:
  - Session information, user input, selected abilities.
  - Files touched, commands executed, and their results.
  - Current orchestration stage (`understand`, `plan`, `act`, `review`).

The orchestrator produces events; the hook registry says which handlers to invoke; handlers run and optionally influence orchestration.

### 1.2 Event types

The hook system is intentionally small and focused. A typical baseline includes:

- `UserPromptSubmit`  
  Triggered when a user submits a natural-language request. Used for:
  - Skill / ability auto-activation.
  - Early knowledge routing hints.
  - Lightweight logging of high-level intent.

- `PreAbilityCall`  
  Triggered right before the execution script calls an ability. Used for:
  - Precondition checks (environment, inputs).
  - Optional “are you sure?” confirmations for sensitive actions.
  - Tagging upcoming calls for logging.

- `PostAbilityCall`  
  Triggered after an ability finishes. Used for:
  - Usage tracking (what was called, why, and with which result).
  - Updating metrics and usage-based matchers.
  - Feeding data into future routing improvements.

- `SessionStart` / `SessionEnd` / `SessionStop`  
  Triggered when an interaction session starts, ends, or is explicitly stopped. Used for:
  - Running lightweight health checks.
  - Summarizing what was done in the session.
  - Optionally kicking off heavier CI flows (tests, type checks) without tying them to ability routing.

These event types are enough to support the typical patterns found in the reference hooks: prompt-based skill activation, post-tool tracking, and stop-time validation.

### 1.3 Lifecycle

For a single event:

1. The orchestrator emits an event with a **context object**.
2. The hook registration script resolves which hooks apply.
3. The orchestrator invokes each handler with the context.
4. Handlers may:
   - Write logs or other side effects.
   - Suggest abilities or documents.
   - Emit structured intents.
   - Return messages to be surfaced to the user or AI.
5. The orchestrator merges the handler results and continues the normal flow.

If no hooks are registered for an event, nothing special happens and the system continues as usual.

---

## 2. Registration and Execution Model

### 2.1 Hook registry files (`.system/hooks/*.yaml`)

Hooks are declared via small YAML files that are easy for both humans and code models to read and modify.

Recommended layout:

- Directory: `.system/hooks/`
- One file per hook, for example:
  - `.system/hooks/skill_activation_prompt.yaml`
  - `.system/hooks/post_tool_use_tracker.yaml`
  - `.system/hooks/session_stop_build_check.yaml`

Example hook definition:

```yaml
# .system/hooks/post_tool_use_tracker.yaml
id: post_tool_use_tracker
event_type: PostAbilityCall
enabled: true

summary: >
  Track every ability/tool call in a central log so future routing
  and documentation can be improved from real usage.

match:
  # Optional filters; omit if this hook should see all calls
  ability_scope: "*"
  min_duration_ms: 0

handler:
  kind: script
  command: ./scripts/hooks/post_tool_use_tracker.sh

effects:
  # Informal description for the AI and maintainers
  - append_to: .system/logs/tool_call.md
  - update_usage_cache: .system/cache/ability_usage.json
```

Design notes:

- Use **short, stable fields** (`id`, `event_type`, `enabled`, `handler`, `match`, `effects`).
- The `effects` field is descriptive, not a programming language; the handler performs the real work.
- Configuration is kept close to the repository, so a code model can inspect and modify it without external tooling.

### 2.2 Hook registration script

Instead of defining “required”, “recommended”, or “optional” hook combinations, this template uses a **single registration script** that tells the orchestrator which hooks exist and how to run them.

Typical location:

- `./scripts/hooks/register_hooks.py` (or `.sh`, `.ts`, etc.)

Responsibilities:

1. **Discover hook definitions**
   - Scan `.system/hooks/*.yaml`.
   - Parse each file and validate basic fields.
2. **Apply environment filters**
   - Respect `enabled` and any environment-specific flags (e.g., only run certain hooks in `dev`).
3. **Emit machine-readable registration**
   - Output JSON describing hooks grouped by `event_type`.

Example (conceptual) JSON output:

```json
{
  "UserPromptSubmit": [
    {
      "id": "skill_activation_prompt",
      "handler": {
        "kind": "script",
        "command": "./scripts/hooks/skill_activation_prompt.sh"
      }
    }
  ],
  "PostAbilityCall": [
    {
      "id": "post_tool_use_tracker",
      "handler": {
        "kind": "script",
        "command": "./scripts/hooks/post_tool_use_tracker.sh"
      }
    }
  ],
  "SessionStop": [
    {
      "id": "session_stop_build_check",
      "handler": {
        "kind": "script",
        "command": "./scripts/hooks/session_stop_build_check.sh"
      }
    }
  ]
}
```

The orchestrator calls this script at startup or on demand, caches the result, and uses it to decide which handlers to run for each event.

### 2.3 Execution contract

Handlers are invoked with a simple standard contract:

- **Input**: a JSON object on `stdin` (for scripts) or a structured argument (for in-process handlers).
- **Output**: a JSON object on `stdout` describing the hook result.

Example input (for `PostAbilityCall`):

```json
{
  "event_type": "PostAbilityCall",
  "session_id": "sess-123",
  "ability_id": "billing_regression",
  "operation_key": "workflow.billing.regression",
  "status": "success",
  "duration_ms": 187000,
  "input_summary": "Run regression tests against non-prod billing",
  "output_summary": "All 42 test cases passed",
  "tool_call_path": ".system/implement/task-123/tool_call.md",
  "timestamp": "2025-01-15T10:23:45Z"
}
```

Example output:

```json
{
  "logs": [
    {
      "level": "info",
      "message": "Recorded billing_regression run to tool_call.md"
    }
  ],
  "usage_recorded": true
}
```

If the handler fails, the orchestrator should treat it as **non-fatal** by default and continue the main flow, unless the hook is explicitly marked as blocking in its YAML definition.

---

## 3. Built-in Hook Points and Examples

This section shows how the key hook types map to concrete use cases, including triggers, inputs, and outputs.

### 3.1 `UserPromptSubmit`: skill activation prompt

**Use case:** Automatically propose relevant abilities when the user writes a natural-language request.

Example definition:

```yaml
# .system/hooks/skill_activation_prompt.yaml
id: skill_activation_prompt
event_type: UserPromptSubmit
enabled: true

summary: >
  Read the incoming user message, cross-check it against the ability
  pool, and suggest a small set of abilities that are likely useful.

handler:
  kind: script
  command: ./scripts/hooks/skill_activation_prompt.sh

effects:
  - suggest_abilities
  - optionally_emit_structured_intent
```

Example event context:

```json
{
  "event_type": "UserPromptSubmit",
  "session_id": "sess-789",
  "user_message": "I need to run the full billing regression tests",
  "files_touched": [],
  "stage": "understand",
  "timestamp": "2025-01-15T09:00:00Z"
}
```

Example handler output:

```json
{
  "suggested_abilities": [
    {
      "id": "billing_regression",
      "reason": "Matches 'billing regression tests' and regression workflow scope",
      "confidence": 0.9
    }
  ],
  "messages_to_user": [
    "You can run the billing regression workflow ability to execute the full suite."
  ]
}
```

The orchestrator surfaces the suggestions to the AI or user. The AI still uses ability routing to validate and choose which ability to call; the hook only provides **structured hints**.

### 3.2 `PostAbilityCall`: post-tool-use tracker

**Use case:** Track all ability and tool invocations in a central log to improve future routing and documentation.

Example definition (like the earlier snippet):

```yaml
# .system/hooks/post_tool_use_tracker.yaml
id: post_tool_use_tracker
event_type: PostAbilityCall
enabled: true

summary: >
  After each ability call, append a concise record to a log file and
  update a lightweight usage cache for later analysis.

handler:
  kind: script
  command: ./scripts/hooks/post_tool_use_tracker.sh

effects:
  - append_to: .system/logs/tool_call.md
  - update_usage_cache: .system/cache/ability_usage.json
```

Example event context:

```json
{
  "event_type": "PostAbilityCall",
  "session_id": "sess-123",
  "ability_id": "code_review_agent",
  "operation_key": "agent.code_review",
  "status": "success",
  "duration_ms": 124000,
  "input_summary": "Review changes in src/payments/*.ts",
  "output_summary": "Found 3 issues, suggested fixes in-line",
  "tool_call_path": ".system/implement/task-456/tool_call.md",
  "timestamp": "2025-01-15T10:45:12Z"
}
```

Example handler output:

```json
{
  "logs": [
    {
      "level": "info",
      "message": "Logged code_review_agent usage to tool_call.md"
    }
  ],
  "usage_recorded": true,
  "update_matcher_suggestions": [
    {
      "ability_id": "code_review_agent",
      "signal": "frequent_usage",
      "weight": 0.7
    }
  ]
}
```

This pattern borrows from the reference design where a post-tool-use tracker powers usage analytics and better matchers, but here it stays declarative and repository-local.

### 3.3 `SessionStop`: optional build and quality checks

**Use case:** When a session is being stopped, optionally run heavier checks such as type checking or targeted test suites, without entangling them with ability routing.

Example definition:

```yaml
# .system/hooks/session_stop_build_check.yaml
id: session_stop_build_check
event_type: SessionStop
enabled: false  # opt-in, can be enabled per repo or environment

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
  - summarize_errors_to_user
```

Example event context:

```json
{
  "event_type": "SessionStop",
  "session_id": "sess-456",
  "changed_files": [
    "services/billing/src/invoice.ts",
    "services/billing/src/line_item.ts"
  ],
  "timestamp": "2025-01-15T11:30:00Z"
}
```

Example handler output:

```json
{
  "messages_to_user": [
    "I ran `npm run build:billing` for the modified billing service.",
    "The build failed: TypeScript error in services/billing/src/invoice.ts:123.",
    "Recommend fixing the error before merging."
  ],
  "follow_up_actions": [
    {
      "kind": "open_file",
      "path": "services/billing/src/invoice.ts"
    }
  ]
}
```

This mirrors the build-check and error-reminder hooks from the reference material but keeps them **optional** and **decoupled** from ability selection. They behave more like local CI steps triggered by session lifecycle.

---

## 4. Handler Design: Inputs and Outputs

### 4.1 Context model

Handlers receive a structured context object. A minimal TypeScript-style model:

```ts
interface HookContextBase {
  event_type: string;
  session_id: string;
  timestamp: string;
  repository_root?: string;
  stage?: "understand" | "plan" | "act" | "review";
}

interface UserPromptSubmitContext extends HookContextBase {
  event_type: "UserPromptSubmit";
  user_message: string;
  files_touched?: string[];
}

interface PreAbilityCallContext extends HookContextBase {
  event_type: "PreAbilityCall";
  ability_id: string;
  operation_key?: string;
  planned_input_summary?: string;
}

interface PostAbilityCallContext extends HookContextBase {
  event_type: "PostAbilityCall";
  ability_id: string;
  operation_key?: string;
  status: "success" | "failure" | "skipped";
  duration_ms?: number;
  input_summary?: string;
  output_summary?: string;
  tool_call_path?: string;
}

interface SessionStopContext extends HookContextBase {
  event_type: "SessionStop";
  changed_files?: string[];
}
```

Guidelines:

- Keep the base fields stable. Extend with additional fields instead of mutating existing ones.
- Make sure every field is derivable from the repository state or orchestration state (no hidden dependencies).
- Use summaries (`*_summary`) rather than raw, large payloads to avoid blowing up context.

### 4.2 Result model

Handlers return a `HookResult` object. Example model:

```ts
interface HookResult {
  logs?: { level: "debug" | "info" | "warn" | "error"; message: string }[];

  messages_to_user?: string[];

  suggested_abilities?: {
    id: string;
    reason: string;
    confidence?: number;
  }[];

  suggested_documents?: {
    path: string;
    reason: string;
    stage?: "understand" | "plan" | "act" | "review";
  }[];

  // Optional structured intent, reused by knowledge routing
  new_intent?: {
    scope?: string;
    topic?: string;
    stage?: "understand" | "plan" | "act" | "review";
    intent: string;
    signals?: Record<string, unknown>;
  };

  follow_up_actions?: {
    kind: "open_file" | "run_command" | "create_task";
    payload: Record<string, unknown>;
  }[];

  usage_recorded?: boolean;

  error?: string;
}
```

The orchestrator is responsible for:

- Logging `logs`.
- Surfacing `messages_to_user`.
- Passing `suggested_abilities` to the ability routing layer as hints.
- Passing `new_intent` to the knowledge routing layer as a trigger input.
- Executing or queuing `follow_up_actions` where appropriate.

### 4.3 Handler implementation patterns (examples)

#### 4.3.1 Simple logging handler

Example script (conceptual Python):

```python
#!/usr/bin/env python3
import json
import sys
from pathlib import Path

def main():
  ctx = json.load(sys.stdin)
  log_path = Path(".system/logs/hooks.log")
  log_path.parent.mkdir(parents=True, exist_ok=True)
  with log_path.open("a", encoding="utf-8") as f:
    f.write(json.dumps(ctx) + "\n")
  json.dump({"usage_recorded": True}, sys.stdout)

if __name__ == "__main__":
  main()
```

- Trigger: typically attached to `PostAbilityCall`.
- Input: full event context.
- Output: `{"usage_recorded": true}`.
- Effect: standardized logging without affecting routing.

#### 4.3.2 Ability suggestion handler

Handler logic (conceptual):

1. Read `UserPromptSubmitContext`.
2. Load `ABILITY.md` and ability registries for the current scope.
3. Use a code model to semantically match the prompt against ability descriptions.
4. Return `suggested_abilities` with reasons.

Example output:

```json
{
  "suggested_abilities": [
    {
      "id": "signup_e2e",
      "reason": "User asked to test signup flow end-to-end",
      "confidence": 0.84
    }
  ],
  "messages_to_user": [
    "You can use the signup_e2e workflow ability to run the full signup test suite."
  ]
}
```

This mirrors the “skill activation” hook pattern from the reference while keeping ability routing as the final source of truth for what can be executed.

#### 4.3.3 Knowledge-trigger handler

For `UserPromptSubmit` or `SessionStop` events, a handler can emit a `new_intent` that aligns with the structured intent schema used by knowledge routing:

```json
{
  "new_intent": {
    "scope": "execution.quality_gates",
    "topic": "pre_commit_checks",
    "stage": "plan",
    "intent": "ensure_code_quality_before_commit",
    "signals": {
      "user_raw_input": "I want to run a pre-commit check",
      "cli_command": "make dev_check"
    }
  }
}
```

The orchestrator passes `new_intent` to the knowledge routing layer, which then selects appropriate routing documents and knowledge docs. The hook only builds the structured hint; it does not implement routing logic itself.

#### 4.3.4 Soft guardrail handler (decoupled from routing)

If stronger checks are needed, implement them as **soft guardrails** via hooks:

- Trigger: `PreAbilityCall` or `SessionStop`.
- Behavior:
  - Inspect the context (target environment, operation key, changed files).
  - Run domain-specific checks (e.g., disallow destructive operations in production).
  - Either:
    - Return a warning in `messages_to_user`, or
    - Mark the call as blocked by returning an error flag that the orchestrator understands.

Example output:

```json
{
  "messages_to_user": [
    "This ability would write to production. In this environment, it is blocked."
  ],
  "error": "blocked_by_policy"
}
```

These checks are implemented via hooks and **do not require any special fields in the ability registries**. This keeps ability routing clean while still allowing strong policies where needed.

---

## 5. Integration Scenarios and Flows

### 5.1 Collaboration with ability routing (example flow)

Scenario: A developer types, “Run the full billing regression tests for me.”

1. **Event emission**
   - The orchestrator emits a `UserPromptSubmit` event with the user message.

2. **Hook execution**
   - `skill_activation_prompt` receives the context.
   - It loads ability routing information for the current scope.
   - It matches the message against high-level abilities and returns:

     ```json
     {
       "suggested_abilities": [
         {
           "id": "billing_regression",
           "reason": "Regression workflow for billing module",
           "confidence": 0.9
         }
       ]
     }
     ```

3. **Orchestrator action**
   - The orchestrator surfaces the suggestion to the AI.
   - The AI:
     - Verifies the ability via its registry.
     - Creates a task via the execution script.
     - Executes as usual.

4. **Post-call tracking**
   - After the ability completes, a `PostAbilityCall` event is emitted.
   - `post_tool_use_tracker` records the run in `tool_call.md` and updates usage metrics.

Outcome: abilities remain discoverable and safe through their routing definitions; hooks simply add **auto-activation and instrumentation**.

### 5.2 Collaboration with knowledge routing (example flow)

Scenario: A developer keeps asking about pre-commit checks and running `make dev_check`.

1. **Event emission**
   - The developer types: “Before I commit, what checks should I run?”
   - `UserPromptSubmit` event is emitted with `user_message` and possibly the last CLI command.

2. **Hook execution**
   - A `UserPromptSubmit` hook emits a `new_intent`:

     ```json
     {
       "new_intent": {
         "scope": "execution.quality_gates",
         "topic": "pre_commit_checks",
         "stage": "understand",
         "intent": "learn_which_checks_to_run_before_commit",
         "signals": {
           "user_raw_input": "Before I commit, what checks should I run?"
         }
       }
     }
     ```

3. **Knowledge routing**
   - The orchestrator forwards `new_intent` to the knowledge routing system.
   - Knowledge routing uses `scope`, `topic`, and `stage` to select the right routing documents and knowledge docs.

4. **Response**
   - The AI reads the loaded docs (pre-commit policies, example commands) and responds with precise guidance.

Hooks provide the structured **trigger inputs**; knowledge routing performs the actual document selection and loading.

### 5.3 Collaboration with CI / local automation (example flow)

Scenario: The developer ends a session after editing multiple TypeScript services.

1. **Event emission**
   - The developer stops the session.
   - `SessionStop` event is emitted with `changed_files`.

2. **Hook execution**
   - `session_stop_build_check` is enabled in `dev` environments.
   - The handler:
     - Groups changed files by service (e.g., `billing`, `auth`).
     - Builds a per-service build command (e.g., `npm run build:billing`).
     - Runs the commands and collects results.

3. **Handler output**

   ```json
   {
     "messages_to_user": [
       "Ran `npm run build:billing` and `npm run build:auth` for modified services.",
       "Billing build: success.",
       "Auth build: failed with TypeScript error in services/auth/src/login.ts:58."
     ],
     "follow_up_actions": [
       {
         "kind": "open_file",
         "payload": { "path": "services/auth/src/login.ts" }
       }
     ]
   }
   ```

4. **Orchestrator action**
   - The orchestrator surfaces the messages and optionally executes follow-up actions (e.g., open the failing file).

This pattern generalizes the “tsc-check” and build-resolver hooks in the reference: heavy, monorepo-aware checks are applied **only when explicitly configured** and are not part of ability routing.

---

## 6. Maintenance and Troubleshooting

### 6.1 Where to look

- **Definitions:** `.system/hooks/*.yaml`
- **Registration logic:** `scripts/hooks/register_hooks.*`
- **Handler implementations:** `scripts/hooks/*.sh` / `*.py` / `*.ts`
- **Logs:** `.system/logs/hooks.log` and `tool_call.md` files

A code model can efficiently maintain hooks by:

- Reading a small set of YAML definitions.
- Adjusting handler commands or conditions.
- Regenerating or refactoring handler scripts as needed.

### 6.2 Common issues

**Hook not executing**

- Check that `enabled: true` in the YAML.
- Ensure the registration script includes the hook in its JSON output.
- Verify handler executables have the right permissions (`chmod +x` for shell scripts).
- Confirm the orchestrator is calling the registration script at startup.

**Unexpected or heavy behavior**

- For heavy hooks (build/test), start with `enabled: false` and enable only in specific environments.
- Narrow `match` patterns (e.g., only run on certain paths or ability scopes).
- Consider moving very heavy logic to CI pipelines instead of session-time hooks.

**False positives in validation hooks**

- Make matching logic more precise (e.g., filter by `operation_key` or environment).
- Add explicit skip rules inside handlers for low-risk operations.

### 6.3 Design principles recap

- **Optional by default**: the system must function without any hooks.
- **Small and explicit**: each hook does one thing and is described in a small YAML file.
- **Repository-local**: definitions and handlers live in the repository and are discoverable by a code model.
- **Decoupled from routing**: ability and knowledge routing remain focused on matching and retrieval; hooks add instrumentation, automation, and optional guardrails.

With this structure, hooks can be gradually introduced and evolved to support more advanced workflows while keeping AI development fast, predictable, and maintainable.
