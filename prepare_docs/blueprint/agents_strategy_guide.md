# Strategy Guide for `AGENTS.md`

## 1. Purpose & Scope of Strategy Docs

This document defines how strategy documents named `AGENTS.md` are used and organized in this repository.

- **Audience**: any AI system that reads and follows `AGENTS.md` (code models, assistants, task runners, etc.).
- **Goal**: ensure that AI behavior is consistent with the project’s architecture and conventions, while keeping all AI-facing documents **lightweight and structured**.
- **Scope**: only covers `AGENTS.md` files:
  - What they are for.
  - Where they live.
  - How they should be structured.
  - How AI should interpret them.
  - How to maintain them as a network of strategy docs.

Other project documents (such as design docs, tool descriptions, or knowledge indexes) may be referenced by `AGENTS.md`, but their internal structure is not defined here.


## 2. Strategy Doc Layers in This Repo

`AGENTS.md` appears at multiple levels. Together they form a hierarchy of strategies that guide AI behavior.

### 2.1 Project Root `AGENTS.md`

**Location**: repository root.

**Role**: entry point for AI to understand the project as a whole.

This strategy doc:

- Describes the **overall purpose and domain** of the project.
- Explains the **high-level architecture**, including the existence of a modular system and other major areas of the codebase.
- States what AI is **allowed and expected to do** at the project level (examples: add features, refactor, improve tests) and what it **must not do** (examples: modify production configuration directly, change licensing).
- Provides orientation for common task types:
  - Feature development.
  - Bug fixing and debugging.
  - Refactoring and cleanup.
  - Documentation and examples.
- Tells AI **when to move down** into more specific `AGENTS.md`:
  - For work inside the modular system → go to `/modules/AGENTS.md`.
  - For work focused on a specific module or directory → go to the corresponding local `AGENTS.md`.

This document **does not** define detailed rules for a specific module or directory. It sets global expectations and directs AI to more specific strategies.

---

### 2.2 Modular System Root `AGENTS.md` (`/modules/AGENTS.md`)

**Location**: `/modules/AGENTS.md`.

**Role**: entry point for AI to understand and work with the **modular system** used in this project.

This strategy doc:

- Describes the **module architecture**, such as:
  - Module types and their responsibilities.
  - The relationship between module instances.
  - How modules are intended to collaborate.
- Defines **development rules** that apply across all modules:
  - Allowed and forbidden dependencies between modules.
  - Shared coding or UI patterns that modules should follow.
  - Conventions for cross-module integration tests.
- Guides AI on tasks that affect the module system itself:
  - Adding, modifying, or removing module instances.
  - Adjusting module-level configuration.
  - Implementing or updating cross-module integration flows.

It also explains **how to move further down**:

- For concrete work in a single module instance, AI should open that module’s own `AGENTS.md`.
- For tasks related to special sub-areas under `modules/` (for example, shared infrastructure or templates), AI should look for local `AGENTS.md` in those directories.

This document focuses on the **architecture of the modular system**, not on project-wide business rules.

---

### 2.3 Module Instance and Directory `AGENTS.md`

**Location**: any module instance directory or other important directories may provide their own `AGENTS.md`.

**Role**: define local strategies and constraints for a **specific area** of the repository.

Examples include:

- A module instance directory (e.g., a user service module).
- A dedicated configuration directory.
- A `workdocs` directory used for long-running or complex tasks.
- A system implementation directory for framework-level code.

Each local `AGENTS.md`:

- Describes the **responsibility and boundary** of that area.
- States **which files and subdirectories are in scope** for AI changes.
- Clarifies **what must not be touched** from this location (for example, external modules, certain configuration files, or generated artifacts).
- Explains any **local conventions** (style, patterns, testing rules) that apply within this area.
- Indicates when AI should **move up** to a higher-level `AGENTS.md` (for cross-module changes or system-wide decisions).

Local strategy docs are closer to the actual code and should remain concise and specific to their directory.

---

## 3. Required Structure for Every `AGENTS.md`

All `AGENTS.md` files should follow a common, lightweight structure so AI can quickly understand and apply them.

Each `AGENTS.md` should contain the sections below (short paragraphs and bullet lists are preferred over long text).

### 3.1 Scope & Role

Clarify:

- **Scope**:
  - Which part of the repository this document covers (project, module system, specific module, or concrete directory).
- **Role**:
  - What decisions this document is expected to guide for AI.
  - Which kinds of tasks are most relevant here.

This section answers the question:  
> “Where am I, and what kind of strategy is defined at this level?”

---

### 3.2 Responsibilities & Boundaries

Describe:

- **Responsibilities**:
  - What this part of the codebase is responsible for (functionally or technically).
  - What types of changes are considered appropriate from here.
- **Boundaries**:
  - Which directories or resources are **explicitly in scope**.
  - Which are **out of scope** or **strictly forbidden** for modification.

Where relevant, mention how this area interacts with its neighbors without repeating the full architecture.

---

### 3.3 Input & Output Conventions

Explain what the AI may assume when it reads this `AGENTS.md`, and what it is expected to produce:

- **Typical inputs**:
  - The AI already knows the current file or directory path.
  - The AI knows the high-level task type (e.g., “fix bug”, “add feature”, “clean up tests”).
- **Expected outputs**:
  - Deciding which files or subdirectories can be modified.
  - Deciding whether a higher- or lower-level `AGENTS.md` needs to be consulted.
  - Deciding which kinds of changes or plans are appropriate from this context.

This section answers:  
> “Given that I am here, what decisions should I make, and what information can I rely on?”

---

### 3.4 Typical Decisions

List a few **common questions** that this `AGENTS.md` is meant to answer, and summarize the decisions it should drive.

Examples (to be adapted to each level):

- Project root:
  - “Is this change compatible with the project’s overall goals?”
  - “Should this task be done inside the modular system or elsewhere?”
- Modular system root:
  - “Should this functionality go into an existing module or a new one?”
  - “Is this dependency between modules allowed?”
- Module or directory level:
  - “Where should new code for this feature go in this module?”
  - “Is it allowed to change this configuration file from here?”

The point is not to list all possible decisions, but to show AI the **typical reasoning patterns** expected in this scope.

---

### 3.5 Safety & Restrictions

Clearly state any **safety constraints** relevant to this scope, such as:

- Directly forbidden actions (for example: changing certain directories, touching secrets, modifying generated code).
- Requirements for extra care (for example: tests that must be run after changes, mandatory code review for certain files).
- Known high-risk areas that should be treated with caution.

These constraints must be written in a way that AI can apply automatically:
short, explicit, and easy to check from context.

---

### 3.6 Relations to Other `AGENTS.md`

Describe how this strategy doc relates to other strategy docs:

- **Upwards**:
  - Which higher-level `AGENTS.md` provides broader rules that this document must respect.
- **Downwards**:
  - Whether there are more specific `AGENTS.md` in subdirectories that should be consulted for detailed decisions.
- **Sideways**:
  - Whether changes at this level may require reading a peer `AGENTS.md` (for example, for a sibling module in the modular system).

Use generic labels like “project root strategy doc”, “modular system strategy doc”, “module-level strategy doc”, or “local directory strategy doc” rather than concrete filenames.

This section teaches AI **how to navigate** between strategy docs without listing every path.

---

## 4. Layout & Patterns of `AGENTS.md` in the Repo

This section describes common patterns for where `AGENTS.md` should appear and what each pattern focuses on.

### 4.1 Project-Level Strategy

- **Location**: repository root `AGENTS.md`.
- **Focus**:
  - Overall project purpose and high-level architecture.
  - Global constraints that all other `AGENTS.md` must obey.
  - Routing to major areas of the repository (modular system, core libraries, infrastructure).

This is the **starting point** for AI when entering the repository for the first time or when handling tasks that affect the project as a whole.

---

### 4.2 Modular System Strategy

- **Location**: `/modules/AGENTS.md`.
- **Focus**:
  - Explaining the module architecture and type system used in this project.
  - Describing how module instances are organized and how they interact.
  - Defining shared rules that apply across all modules.

This is the starting point for AI when the task is **about modules**:
introducing a new module, refactoring existing modules, or handling cross-module behavior.

---

### 4.3 Module Instance Strategy

- **Location**: each module instance directory should have its own `AGENTS.md`.
- **Focus**:
  - The function and responsibility of this specific module instance.
  - Which internal directories are used for which concerns (source code, tests, configuration, docs, workspaces, etc.).
  - Constraints on how this module can depend on other modules or core libraries.
  - Local coding or testing conventions that apply to this module.

AI working inside a module should follow this strategy doc as its **primary guide** for that module.

---

### 4.4 Directory-Level Strategy

Certain directories benefit from a dedicated `AGENTS.md` to clarify their specialized role. Examples include:

- A directory for shared implementation details or low-level frameworks.
- A directory for long-running-task workspaces (plans, contexts, task lists, results).
- A directory for project-wide docs or examples that must follow specific patterns.

Each directory-level `AGENTS.md` should:

- Explain the **purpose** of that directory.
- Clarify which files are safe for AI to edit or create.
- Indicate when AI should escalate to a higher-level strategy doc.

---

## 5. Priority, Boundaries & Conflict Handling Between `AGENTS.md`

Because multiple `AGENTS.md` files can apply at once, AI must follow clear rules for priority, boundaries, and conflict resolution.

### 5.1 Priority Rules

When several strategy docs are relevant to a decision, AI should apply them in this order:

1. **Respect global hard constraints** from higher-level strategy docs (project root, modular system root).
2. **Apply the most local applicable strategy** (module-level, then directory-level) for detailed decisions.
3. Where local and global strategies both speak about the same topic:
   - The local strategy may **refine or narrow** the global strategy.
   - The local strategy must **not contradict** explicit hard rules from higher levels.

In practice:

- Local decisions should be made using the nearest relevant `AGENTS.md`.
- If a local rule appears to violate a clear global rule, the global rule wins.

---

### 5.2 Expressing Boundaries in `AGENTS.md`

To make priority and scope usable, each `AGENTS.md` should:

- Explicitly list the **directories and files in scope**.
- Explicitly list the **directories and files out of scope** for AI modifications.
- Mention any **sensitive areas** that require separate confirmation or review.
- Avoid vague terms; prefer concrete directory names and clear expressions like:
  - “Do not edit files under these paths from this module.”
  - “Only add new files under this directory for this kind of concern.”

Clear boundaries help AI avoid unintended changes and make it easier to reason about which `AGENTS.md` applies.

---

### 5.3 Handling Conflicts and Uncertainty

If AI detects or suspects a conflict between `AGENTS.md` documents, it should:

1. Prefer the **higher-level hard constraints** when they are explicit.
2. Prefer the **more restrictive interpretation** when unsure (avoid unsafe actions).
3. Record the point of uncertainty in the appropriate project workspace or log mechanism, so humans can review and adjust the strategies if needed.

`AGENTS.md` authors should avoid ambiguity where possible, but the AI must still behave conservatively when ambiguity occurs.

---

## 6. Maintaining the Network of Strategy Docs

`AGENTS.md` form a network of strategies rather than isolated files. Keeping this network healthy improves AI effectiveness.

### 6.1 Strategy Doc Index (Optional but Recommended)

The project may maintain a simple index that tracks:

- Locations of all `AGENTS.md` files.
- Their layer (project root, modular system, module instance, or directory-level).
- A short description of each strategy doc’s scope.
- Last updated time and, optionally, the reason for the last change.

This index helps both humans and AI understand the structure of the strategy doc network and find the right entry points.

---

### 6.2 Consistency Checks

Automated checks can help ensure that `AGENTS.md` stay consistent with the repository layout.

Examples of checks:

- Each module instance directory under the modular system has its own `AGENTS.md`.
- Directories that are meant to be AI-facing (for example, for long-running tasks or shared frameworks) have a corresponding `AGENTS.md`.
- Paths mentioned in `AGENTS.md` as within scope or out of scope still exist and match the actual project structure.

These checks reduce the risk that strategy docs become outdated or misleading.

---

### 6.3 Updating `AGENTS.md` With Structural Changes

When the repository structure or architecture changes, AGENTS docs should be updated as part of the change.

Typical triggers for updating `AGENTS.md` include:

- Adding, renaming, or removing modules.
- Reorganizing directories that are referenced by existing `AGENTS.md`.
- Changing global constraints or project-level goals.
- Adding new categories of AI-relevant directories (for example, new workspace areas).

Each update should keep the document:

- **Short**: remove obsolete or redundant text.
- **Structured**: maintain the required section layout.
- **Clear**: describe only the strategies that truly belong at that level.

Keeping `AGENTS.md` minimal and up to date makes them more useful to AI over time.
