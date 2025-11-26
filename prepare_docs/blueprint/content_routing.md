# Knowledge Routing

This document defines the **knowledge routing** system used to help an AI development assistant reliably find and load the right documentation during task execution.

The goals are:

- **Reduce uncertainty when the AI retrieves documents**, and increase the efficiency and density of useful knowledge.
- **Align the access path with the AI’s orchestration model**, so that routing naturally supports multi‑step task planning and execution.

To achieve this, we introduce a **multi‑level routing structure**:

- **Top‑level routing (scopes)** – high‑level entry points for retrieval.
- **Topic‑level routing (topics)** – problem families within a scope.
- **Concrete documents (knowledge docs)** – actual content that is loaded into context.

We treat **routing artefacts** and **knowledge documents** as two distinct types:

- **Routing documents** – `ROUTING.md` and topic route files under `/routes`, which provide natural‑language navigation and references.
- **Knowledge documents** – the actual content files (Markdown or other formats) that hold the detailed context.

Routing documents are optimized for **AI navigation and selection**, while knowledge documents are optimized for **explanatory content**.

---

## 1. Concepts and Basic Rules

Knowledge routing is designed for an **AI orchestration system** where the AI is the primary actor: it receives a high‑level request (“implement feature X”) and independently decides which routes and documents to follow.

### 1.1 Orchestration stages

For reasoning and retrieval, we conceptually divide the AI’s work into four stages:

- `understand` – understand the problem / background / concepts
- `plan` – decompose the task / design steps / plan an approach
- `act` – perform concrete actions (write code, change config, run scripts, etc.)
- `review` – check / evaluate / summarize / reflect

> Important:  
> Authors **do not** need to bind a document to a stage while writing it.  
> Stage tags are added later in routing files, as **usage hints** for when a document is most helpful.

### 1.2 Key terms

We use the following core terms:

| Term           | Meaning                                                                                             |
|----------------|-----------------------------------------------------------------------------------------------------|
| **scope**      | Lightweight retrieval entry point (a high‑level area); points to one or more topic route files.    |
| **topic**      | Short label representing a **problem family** within a scope.                                       |
| **route**      | A scenario within a topic; describes **when to use** a group of related documents.                  |
| **related_doc**| A reference to an actual knowledge document, with a path and usage hints (stage, doc_usage, etc.). |

Because routing documents and knowledge documents have distinct responsibilities, their formats are defined separately.

---

## 1.3 Routing documents

There are two main types of routing documents:

1. **Top‑level routing document**: `ROUTING.md`
2. **Topic route files**: YAML files under `/routes/…`

#### 1.3.1 Top‑level routing (`ROUTING.md`)

- **File name:** The entry routing document must be named `ROUTING.md`.
- **Purpose:** Acts as the **AI’s entrypoint** to the knowledge routing system. It contains a concise index of `scope` + `topic`, and tells the AI:
  - Which topics exist.
  - Where the corresponding topic route files are.
  - Roughly what each topic is about.

Example:

```yaml
route_indexes:
  - scope: execution.quality_gates
    topic: pre_commit_checks
    summary: >
      All routes related to pre‑commit checks and quality gates.
      Load this topic when you need to understand or handle pre‑commit checks.
    route_file: /routes/execution.quality_gates.pre_commit_checks.yaml

  - scope: frontend
    topic: bug_triage
    summary: >
      Topic routes for frontend bug triage – from understanding a bug report,
      to narrowing down the root cause and verifying the fix.
    route_file: /routes/frontend.bug_triage.yaml
```

`ROUTING.md` may also include prose sections to explain how routing is organized, but the `route_indexes` block is the part AI relies on programmatically.

#### 1.3.2 Topic route files (`/routes/*.yaml`)

Each topic route file focuses on **a single topic within a scope**.

- **Location:** Any path under `/routes`, referenced by `route_indexes.route_file`.
- **Front matter:** We use a small YAML front matter header to declare the scope and topic:

```yaml
---
scope: execution.quality_gates
topic: pre_commit_checks
---
```

- **Body structure:** The body defines a list of `routes`.  
  Each route contains:

  - `when_to_use` – natural‑language description of **when this route is appropriate**.
  - `related_docs` – list of document references; each entry contains:
    - `path` – relative or absolute path to the knowledge document (required).
    - `stage` – optional array of orchestration stages where this doc is especially relevant.
    - `doc_usage` – natural‑language description of how/why to use this doc in the route.

Example:

```yaml
---
scope: execution.quality_gates
topic: pre_commit_checks
---

routes:

  - when_to_use: >
      Use this route when you are dealing with questions about "pre‑commit checks
      / quality gates", including understanding the concept or confirming rules
      before running the checks.
    related_docs:
      - path: /quality/policies/global1.md
        stage: [understand, act]
        doc_usage: >
          Primary alignment document (read first). Establishes the overall
          concept of quality gates and pre‑commit checks and explains their role
          in the delivery process for this project.

      - path: /quality/policies/global2.md
        stage: [understand]
        doc_usage: >
          Optional supplementary alignment document. Provides more detailed
          organization‑wide quality policies, used when stricter or more
          fine‑grained standards are needed.

  - when_to_use: >
      Use this route when you already understand the idea of quality gates and
      are preparing to actually run pre‑commit checks (for example `make dev_check`)
      on your local machine.
    related_docs:
      - path: /doc_agent/quickstart/quality-quickstart.md
        stage: [plan]
        doc_usage: >
          Workflow‑style quickstart. Guides you through running `make dev_check`
          locally and explains the key checks at a high level.

      - path: /scripts/README.md
        stage: [act]
        doc_usage: >
          Skill/operations document. Describes available scripts and their flags,
          and is used during execution and troubleshooting of pre‑commit checks.
```

**Semantics of `stage` on `related_docs`:**

- `stage` is **optional**.
- If `stage` is provided, the document is primarily intended for those stages.
- If `stage` is omitted, the document is considered usable at **any stage**.
- AI should **prefer** documents whose `stage` includes the current stage but may fall back to generic (no `stage`) docs if needed.

**Semantics of `when_to_use` vs. `doc_usage`:**

- `when_to_use` – describes **the scenario for the route as a whole**  
  (“when you are doing X, consider this group of documents”).
- `doc_usage` – describes **the role of an individual document**  
  (“this is the primary alignment doc”, “this is an optional example”, etc.).

---

## 1.4 Knowledge documents

Knowledge documents are the actual content files (usually Markdown) that the AI will eventually load into context.

- **Front matter:** Each knowledge document should use front matter to declare its metadata:

```yaml
---
id: quality-quickstart
doc_type: workflow        # one of: skill | workflow | example | alignment
scope: execution.quality_gates
topics:
  - pre_commit_checks
stages:
  - plan                  # optional; hints where this doc is most useful
summary: >
  Quickstart for running `make dev_check` locally and understanding the main
  checks involved in the quality gate.
---
Body of the document...
```

- **`doc_type` is a soft classification**, not a hard rule. Typical values:

  - `skill` – domain‑specific tips, patterns, and recipes.
  - `workflow` – task orchestration playbooks, step‑by‑step flows.
  - `example` – concrete examples to use as few‑shot prompts.
  - `alignment` – goals, principles, evaluation criteria, and constraints.

  The AI (or tools around it) may use `doc_type` as a **soft filter**, for example:

  - In `plan`: prefer `workflow` and `alignment`.
  - In `act`: prefer `skill` and `example`.
  - In `review`: prefer `alignment` and `skill`.

- **Content structure:**  
  To minimize context size and improve retrieval:

  1. Split large documents into multiple smaller docs when they serve distinct purposes, each with its own front matter.
  2. If splitting is not appropriate, structure content into clear sections and provide a brief section index at the top.

---

## 1.5 Documents that are *not* part of knowledge routing

Knowledge routing only covers routing documents and knowledge documents as defined above. Other documentation types should generally **not** appear in the knowledge routing graph.

Typical non‑routing document types include:

- **Ability routing documents**  
  - Named `ABILITY.md`.  
  - Help the AI discover available tools / capabilities to call.  
  - See `ability_routing.md` (not covered here).

- **Strategy / policy documents for agents**  
  - Often referenced from `AGENTS.md`.  
  - Define how the AI should behave at a higher level (policies, safety rules, etc.).  
  - See `instruction_ai.md`.

- **Contextual / ephemeral documents**  
  - Used to store intermediate reasoning and results of specific task runs.  
  - Scoped to a module instance or a single workflow run.  
  - See `modulate.md` for details.

- **Format / schema specification documents**  
  - Describe how to format outputs or write to external systems.  
  - Ideally, the operations implied by these docs are wrapped as tools so that the AI can call a capability instead of loading the spec into context.

- **Human‑only documentation**  
  - Purely for human consumption (e.g., HR manuals).  
  - Unless explicitly whitelisted, they should not be pulled into AI context via knowledge routing.

---

## 2. Retrieval Process

### 2.1 Logical flow

All documents that participate in knowledge routing must provide **front matter metadata** that conforms to this specification.

During knowledge retrieval, the AI follows a typical strategy like this:

1. **Interpret the request and stage**  
   - From the user’s request and recent conversation, infer:
     - The current **stage** (`understand`, `plan`, `act`, or `review`).
     - The approximate **scope** (e.g., `execution.quality_gates`, `frontend`).

2. **Select relevant topics from `ROUTING.md`**  
   - Load `ROUTING.md` and read `route_indexes`.  
   - Filter candidates by `scope`.  
   - Use semantic similarity between the request and each `route_indexes.summary` to pick one or a few relevant topics.

3. **Load the corresponding topic route files**  
   - For each selected entry, load the YAML file pointed to by `route_file`.  
   - These files contain `routes` for the chosen `topic`.

4. **Match routes via `when_to_use`**  
   - Iterate over `routes` in the topic file.  
   - Use the current task description and stage to semantically match against `when_to_use`.  
   - Select the most relevant route(s).

5. **Filter `related_docs` by stage**  
   - Within each selected route:
     - Prefer documents whose `stage` includes the current stage.
     - Optionally include documents without `stage` (considered valid for all stages).

6. **Use `doc_usage` (and optionally `doc_type`) to prioritize documents**  
   - From each route’s `related_docs`, use `doc_usage` to decide:
     - Which documents are **primary** and should be read first.
     - Which documents are **supplementary** or **optional**.
     - Which documents are **examples** (for few‑shot prompting).  
   - `doc_type` can be used as an extra hint (e.g., prefer `alignment` during `review`).

7. **Load a small, targeted subset of knowledge docs into context**  
   - Do **not** load every document referenced in routing.  
   - Instead, select a small set (e.g., 1–5 documents) that are most relevant to the current step and stage.

This process balances:

- The flexibility of natural‑language matching (`when_to_use` and `doc_usage`).
- The precision of structured hints (`scope`, `topic`, `stage`, `doc_type`).

### 2.2 Document location and layout

To make routing predictable and discoverable, we recommend:

- Place the **entry routing document** in a well‑known location, e.g.:

  - `ROUTING.md` at the project root.

- Store **topic route files** in a dedicated directory:

  - `/routes/<scope>.<topic>.yaml`, for example:  
    - `/routes/execution.quality_gates.pre_commit_checks.yaml`  
    - `/routes/frontend.bug_triage.yaml`

- Knowledge documents can live in any directory, as long as:

  - Their `path` is stable and referenced correctly from `related_docs.path`.
  - They provide front matter with at least `id`, `scope`, and a `summary`.

Only documents that are part of knowledge routing should be referenced via `ROUTING.md` and `/routes/*.yaml`. Other documents remain outside the routing graph.

---

## 3. Implementation Notes

### 3.1 Stage usage and responsibility

The four orchestration stages (`understand`, `plan`, `act`, `review`) are conceptual tools for both the AI and routing authors.

- **Document authors** focus on writing clear, durable content.  
  They do **not** need to decide “this document is only for `plan`”.
- **Routing authors** (often the same team, but a different mindset) decide:
  - In which stages a document is **most useful**, and
  - How that document should be grouped into routes (`when_to_use`).

Guidelines:

- Only add `stage` to `related_docs` when it actually helps disambiguate usage.
- Leave `stage` empty if the document is broadly useful across stages.
- Keep `when_to_use` phrasing broad enough that it still makes sense from multiple stages, unless the route is truly stage‑specific.

---

### 3.2 Maintenance

To keep knowledge routing reliable and evolvable over time, combine **automated checks** with **periodic human review**.

1. **CI validation (recommended)**  
   A CI pipeline should periodically verify:

   - **Metadata completeness**
     - All knowledge documents include front matter with required fields (e.g., `id`, `scope`, `doc_type`, `summary`).
     - All routing documents (`ROUTING.md` and `/routes/*.yaml`) conform to the structure defined in this README.

   - **Routing consistency**
     - Each `route_indexes.route_file` exists and is readable.
     - Each `related_docs.path` points to an existing knowledge document.
     - Optionally, detect:
       - “Orphan documents” (knowledge docs that are never referenced by any route).
       - “Broken routes” (routes that only reference missing or deprecated docs).

   - **Stage integrity (optional)**
     - All `stage` values are within the allowed set (`understand`, `plan`, `act`, `review`).
     - For critical topics, ensure that key stages (e.g., `act`) have at least one usable route.

   CI should **block merges** when routing is inconsistent, and provide clear error messages to help developers quickly fix issues.

2. **Local validation (recommended)**  
   To avoid CI back‑and‑forth, provide local tooling such as:

   - `content-routing lint` – validate routing structure and document metadata on the current branch.
   - `content-routing graph` – generate a graph view showing which routes reference which documents.

   Developers are encouraged to run these commands locally after editing routing or knowledge docs.

3. **Periodic review and evolution (as needed)**  
   Over time, some routes and docs may become stale, overlap, or diverge from reality. Recommended practices:

   - Assign owners to scopes/topics who periodically (e.g., once per iteration or month) review:
     - Whether `when_to_use` descriptions still match how the system is used.
     - Whether some documents are overloaded and should be split.
     - Whether new, frequently occurring scenarios should gain their own routes or topics.

   - For larger changes (topic splits, scope redesign), use a design/RFC process so the team understands and agrees with routing changes.

The goal is to keep the routing graph **accurate, discoverable, and trustworthy** as the codebase and processes evolve.

---

### 3.3 Trigger mechanism

The knowledge routing system – especially routing documents – primarily relies on the AI’s ability to **perform semantic matching** at runtime:

- It reads `ROUTING.md`, topic route files, and front matter.
- It compares the current task description with `when_to_use` and `doc_usage`.
- It chooses routes and documents accordingly.

On top of this, we can add a **trigger mechanism** as an *optional enhancement* to provide more structured hints into the routing process.

#### 3.3.1 Trigger inputs

Triggers may listen to various signals, such as:

- **User inputs**  
  - CLI commands, chat messages, form submissions.
- **System events**  
  - CI status changes, monitoring alerts, log patterns.
- **AI internal steps**  
  - Intermediate intentions generated by other agents or planning components.

#### 3.3.2 Structured intent format

When a trigger recognizes a potential need to consult knowledge routing, it can emit a **structured intent** for the AI or orchestrator to consume, for example:

```json
{
  "scope": "execution.quality_gates",
  "topic": "pre_commit_checks",
  "stage": "plan",
  "intent": "ensure_code_quality_before_commit",
  "signals": {
    "cli_command": "make dev_check",
    "user_raw_input": "I want to run a pre-commit check"
  }
}
```

- `scope` / `topic`  
  - May be set by the trigger if it can confidently infer them.  
  - Otherwise they can be left empty and inferred via semantic matching in routing.

- `stage`  
  - Optional hint about the current orchestration stage.

- `intent`  
  - Optional high‑level summary of what the user / system is trying to achieve.

- `signals`  
  - Raw evidence that led to this intent (e.g., the exact CLI command or user phrase).

#### 3.3.3 Interaction with knowledge routing

The structured intent **does not replace** natural‑language matching; it acts as a **constraint and hint** for routing:

1. Use `scope` / `topic` (if present) to filter `route_indexes` and select one or a few topic route files.
2. Within those topic files:
   - Optionally use `stage` to prioritize `related_docs` with matching `stage`.
   - Use `intent` and `signals.user_raw_input` to semantically match against `when_to_use`.
3. Finally, still rely on natural‑language understanding of `when_to_use` and `doc_usage` to pick the most appropriate documents.

#### 3.3.4 Design principles

- Triggers are **optional**. Routing must still work with only natural‑language matching.
- The structured intent schema should stay **small and stable**, so multiple entrypoints (CLI, web UI, agents) can reuse it.
- Triggers should **not** implement business logic. Their job is to transform messy inputs into structured hints; the actual decision of which routes/docs to load remains with the AI and routing rules.

By combining semantic matching with lightweight triggers, the routing system can offer both **flexibility** and **robustness** in complex projects.

---
