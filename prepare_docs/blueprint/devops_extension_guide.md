# AI DevOps Extension Guide

## 1. Purpose & Scope of This Guide

This guide describes how the repository template supports extended usage scenarios beyond core feature development, under the assumption that AI acts as the primary implementer and humans collaborate as information providers and approvers.

It focuses on four scenarios:

- Project initialization based on the template
- Database usage with a repository mirror
- Packaging applications from the template
- Deploying applications to different environments

The guide defines how the repository layout should support these scenarios and how AI and humans are expected to use that layout. It is an architectural and usage guide. It deliberately avoids describing concrete tools, APIs, or platform-specific details.

---

## 2. Common Principles for Extended Scenarios

All extended scenarios in this guide share several common principles.

### 2.1 AI-first, human-supported

- AI is the primary executor for routine work inside the repository boundaries.
- Humans provide project knowledge, constraints, and approvals, and execute sensitive operations when necessary.
- The repository structure is designed so that AI can discover what it needs from files and directories, without hidden, out-of-band knowledge.

### 2.2 Scenario-local workspaces

- There is no single global workspace for long-running tasks.
- Each scenario uses its own local workspace directory under the relevant path. For example, project initialization uses:

  ```text
  /PROJECT_INIT/workdocs/
  ```

- A scenario-local workspace is used to:
  - Capture inputs gathered from humans and other systems.
  - Record AI reasoning, plans, and decisions for that scenario.
  - Track status, open questions, and outcomes.
- A scenario-local workspace is only responsible for its scenario. It does not control or store state for unrelated development tasks.

### 2.3 Scripted bridges to external systems

- Real-world systems such as databases, artifact registries, and cloud platforms are not accessed arbitrarily.
- The repository provides a set of scripts or commands for each scenario that form the **only bridge** between the repository and those external systems.
- AI should treat these scripts as the canonical way to apply changes or to retrieve structured information from the outside world.

### 2.4 Single source of truth with local mirrors

- For each external system, there is a clear single source of truth (for example, the real database or the real cloud environment).
- The repository holds **mirrors** and **descriptions**:
  - These are structured files that describe schema, environments, deployment targets, and so on.
  - AI uses the mirrors and descriptions to reason, plan, and test.
- Scripts are responsible for keeping mirrors and external systems aligned as needed.

### 2.5 Clean separation of DevOps structures

- DevOps-related structures (initialization, database management, packaging, deployment) are placed in clearly defined areas of the repository.
- These structures must not pollute business modules or break the template’s core layout.
- Each scenario’s files and scripts live in its own dedicated area and use a small, predictable set of patterns.

---

## 3. Project Initialization Based on the Template

### 3.1 Goals of initialization

Project initialization is the process of taking the template and turning it into a concrete project description. It is **not** about implementing features.

The goals of initialization are:

- Collect project-specific information:
  - Product and domain description
  - Expected environments and runtime context
  - High-level requirements and constraints
- Integrate that information into the template’s existing documents and configuration locations.
- Provide suggestions for module usage and structure, without directly modifying module registration mechanisms.

Initialization is an early, one-time phase that prepares the repository for future development work.

### 3.2 Repository layout for initialization

Initialization uses a dedicated directory at the project root, for example:

```text
/PROJECT_INIT/
  doc/       # Human-facing instructions for running the initialization flow
  AI/        # AI-facing guidance and prompts used only during initialization
  workdocs/  # Scenario-local workspace for initialization
```

Key points:

- `/PROJECT_INIT/workdocs/` is a workspace **only for initialization**. It does not participate in normal development.
- Typical contents of `/PROJECT_INIT/workdocs/` include:
  - Project profile: goals, stakeholders, domain context.
  - Environment overview: planned environments and their roles.
  - Requirements and constraints discovered during conversations.
  - Suggested module structure and priority, written as proposals rather than final registration data.

After initialization is complete, this directory may remain as historical documentation or may be archived/removed, but it does not hold the ongoing development state.

### 3.3 How AI uses initialization structures

During initialization, AI should:

1. Read guidance under `/PROJECT_INIT/AI/` to understand the expected flow and question types.
2. Interactively gather information from humans or other sources.
3. Store the collected information, interpretations, and proposals under `/PROJECT_INIT/workdocs/`.
4. Once the information stabilizes, **merge the results into existing template documents and configuration locations**:
   - Fill in product, environment, and requirement sections that the template already provides.
   - Respect the existing layout instead of inventing new structure.
5. For module-related information:
   - Record suggested module boundaries, responsibilities, and priorities under `/PROJECT_INIT/workdocs/`.
   - Do not directly change module registration mechanisms. Those are handled by dedicated tools or later steps that consume the suggestions.

AI must treat the template’s layout as fixed during initialization. It may fill in designated placeholders, but should not restructure directories or repurpose reserved areas.

### 3.4 How humans collaborate in initialization

Humans collaborate by:

- Following the instructions under `/PROJECT_INIT/doc/` to understand what information is needed.
- Answering AI’s questions about product, environment, and constraints.
- Reviewing summaries and proposals under `/PROJECT_INIT/workdocs/` to ensure AI’s understanding is correct.
- Approving the integration of finalized information into the template’s standard documents and configurations.
- Executing any required commands to mark initialization as complete or to archive `/PROJECT_INIT`.

The goal is that even a developer unfamiliar with the business can complete initialization by relying on the template structure and AI guidance.

---

## 4. Database Usage with a Repository Mirror

### 4.1 Goals and single-source-of-truth model

In this template, **real databases are always the single source of truth** for data and schema. This holds regardless of whether databases are hosted in the cloud, on local machines, or on servers.

However, because AI is the primary developer, the repository must provide a **mirror** of the database structure that AI can read, edit, and test against without directly manipulating live databases.

The goals for database support are:

- Allow AI to design and evolve schemas based on a clear, versioned description in the repository.
- Avoid direct, ad-hoc database changes by AI or humans that would drift from the repository’s understanding.
- Support multiple database environments with well-defined roles and permissions.

### 4.2 Repository layout for database artifacts

Database-related artifacts should live in a dedicated area of the repository, for example:

```text
/db/
  schema/        # Structured descriptions of tables, relationships, and constraints
  migrations/    # Migration definitions tracked in version control
  config/        # Environment-level database configuration
  samples/       # Optional sample data or fixtures for development and testing
  workdocs/      # Scenario-local workspace for database design and change planning
```

Key ideas:

- The **schema** directory is the primary place where AI learns about tables, columns, indices, and relationships.
- The **migrations** directory holds change definitions that can be applied to real databases.
- The **config** directory describes database environments and how they should be accessed.
- The **samples** directory is optional and can provide representative datasets for testing.
- The **workdocs** directory is used by AI to record design decisions, migration plans, and environment-related notes.

### 4.3 Scripted access and change

All important database operations should be performed via scripts or commands defined under the database area, for example:

- Creating or updating schemas
- Applying migrations
- Seeding sample data
- Performing controlled introspection of live databases

Guidelines:

- AI should **not** attempt to directly connect to databases and issue arbitrary statements outside of these scripts.
- Instead, AI should:
  - Propose changes by editing schema descriptions and migration definitions.
  - Use workdocs to explain the intention and impact of changes.
  - Produce or adjust scripts that implement those changes.
- Humans execute the scripts and review their output.
- Scripts are responsible for:
  - Applying changes to the targeted environment.
  - Updating the repository’s schema descriptions and related artifacts as needed.

This pattern ensures that the repository’s mirror stays aligned with the real databases and that AI’s understanding remains accurate.

### 4.4 Multi-environment configuration

Projects often use several database environments, for example:

- Development and local testing
- Shared testing or staging
- Production

The database configuration in the repository should:

- Enumerate the environments by name.
- Specify connection information or references for each environment.
- Define allowed operation types per environment, such as:
  - Development: migrations and data changes allowed.
  - Testing: migrations controlled, data changes allowed via scripts.
  - Staging: limited changes, usually after review.
  - Production: strong restrictions, possibly read-only from AI’s perspective.

AI should read these configurations when planning work, so that it:

- Chooses the correct environment for each task.
- Respects environment-specific constraints and does not propose forbidden operations.

### 4.5 Human collaboration in database work

Humans are responsible for:

- Reviewing schema changes and migration plans written by AI in workdocs and migration definitions.
- Deciding when to apply changes to shared or production environments.
- Executing database scripts and handling credentials.
- Investigating unexpected issues in coordination with AI, with results written back into the database workdocs.

This division of responsibilities allows AI to lead database design and test workflows while keeping critical operations under controlled human oversight.

---

## 5. Packaging Applications from the Template

### 5.1 Goals of packaging

Packaging is the step of turning code into artifacts that can be distributed and run in specific environments. In this template, packaging aims to:

- Provide a consistent way to build artifacts from modules and services.
- Favor a single, clear packaging model for back-end and service components.
- Support developers who are not familiar with packaging tools, relying on AI to interpret and operate the configuration.

### 5.2 Repository layout for packaging

Packaging-related configuration and scripts should be separated from business code. A typical layout might include:

```text
/ops/packaging/
  services/      # Packaging definitions for back-end or long-running services
  jobs/          # Packaging definitions for scheduled or batch jobs
  apps/          # Packaging definitions for client applications
  scripts/       # Shared scripts or commands to build artifacts
  workdocs/      # Scenario-local workspace for packaging plans and records
```

Principles:

- Each packaging definition should be small and structured, describing:
  - Which code or module it packages.
  - The entry point or command to run.
  - Dependencies or runtime requirements.
- Scripts should be the standard way to trigger builds and produce artifacts.
- Packaging outputs (artifacts) should go to designated output locations, not into business code directories.

### 5.3 Container-based packaging as the default for services

For back-end modules and long-running services, the default packaging target is a **container image**.

Guidelines:

- Each service intended to run as a process in an environment should have a container-oriented packaging definition, for example:
  - A configuration file that describes the base runtime, entry point, ports, and environment variables.
- AI should treat container images as the main way to package and run such services in any environment.
- When working on a service, AI should:
  - Locate the relevant packaging definition under the packaging area.
  - Adjust configuration fields rather than inventing new, ad-hoc packaging methods.
  - Use shared scripts to build container images.

This gives AI and humans a clear mental model: **“to run a service, build and deploy its container image.”**

### 5.4 Other packaging forms

Some components are not back-end services, for example:

- Libraries or shared modules distributed through language-specific package managers.
- Front-end applications packaged as static files or bundles.
- Tools and scripts packaged as command-line utilities.

The repository can support these forms by:

- Providing per-component packaging definitions under dedicated subdirectories.
- Using the same principles: structured configuration, a small number of scripts, and clear output locations.

AI should select the appropriate packaging form based on the component type and the presence of a matching packaging definition.

### 5.5 How AI and humans use packaging structures

**AI responsibilities:**

- Discover packaging definitions for the components involved in a task.
- Update configuration fields consistently with the repository layout and scenario constraints.
- Propose and refine packaging strategies in packaging workdocs.
- Trigger packaging scripts (where allowed) and interpret build logs for humans.
- Record packaging attempts, successes, and failures in the packaging workdocs.

**Human responsibilities:**

- Review packaging configurations and changes proposed by AI, especially when they affect shared or production builds.
- Execute build commands when direct execution is restricted.
- Consult packaging workdocs to understand the current packaging strategy and history.

This structure allows developers who are not packaging experts to rely on the template and AI to produce consistent artifacts.

---

## 6. Deploying Applications to Different Environments

### 6.1 Goals and deployment models

Deployment is the step that takes packaged artifacts and makes them run in target environments. In this template, deployment is described in terms of **deployment models**, not specific platforms.

Three common models are:

1. **HTTP services**: long-running services receiving requests over HTTP/HTTPS.
2. **Serverless, jobs, and event-driven workloads**: code that runs in response to triggers or on schedules.
3. **Client applications**: applications delivered to end users, such as web front-ends, mobile apps, or desktop apps.

The goal is to:

- Allow AI to understand where and how each artifact is expected to run.
- Give developers a stable structure they can follow even if they are unfamiliar with the underlying platforms.
- Keep deployment-related files separate from business logic while still close enough to be maintained together.

### 6.2 Repository layout for deployment descriptions

Deployment-related configuration and scripts should live in a dedicated area, for example:

```text
/ops/deploy/
  http_services/   # Deployment descriptions for HTTP services
  workloads/       # Deployment descriptions for serverless, jobs, and event-driven workloads
  clients/         # Deployment descriptions for client applications
  scripts/         # Shared deployment and rollback scripts
  workdocs/        # Scenario-local workspace for deployment planning and history
```

Deployment descriptions are structured documents that capture:

- Which artifact is being deployed.
- Target environment(s) and their roles.
- Resource expectations (such as capacity, scaling, and resilience).
- Connectivity needs (such as endpoints and dependencies on other services).

Deployment scripts provide the standard way to apply these descriptions to concrete platforms.

### 6.3 HTTP services

For HTTP services:

- The repository should provide service descriptors that define, for example:
  - The container image or artifact to deploy.
  - The port and protocol expected.
  - Health check behavior.
  - Configuration inputs such as environment variables.
- Deployment scripts use these descriptors to create or update service instances in the chosen runtime environment.

AI should:

- Locate the HTTP service descriptor for the relevant module.
- Propose configuration changes and deployment plans in deployment workdocs.
- Use logs and script output to help humans debug deployment issues.

### 6.4 Serverless, jobs, and event-driven workloads

For serverless functions, scheduled jobs, and event-driven workloads:

- The repository should describe:
  - Triggers (HTTP events, queues, schedules, or other signals).
  - Runtime characteristics and resource requirements.
  - Any required permissions or access patterns.
- Infrastructure descriptions can use a declarative style so that the runtime resources are derived from files in version control.

AI should:

- Work with these descriptions rather than directly editing remote configuration.
- Propose changes and new workloads by editing the declarative definitions and documenting intentions in workdocs.
- Treat deployment scripts as the mechanism to apply changes to the real infrastructure.

### 6.5 Client applications

For client applications such as web front-ends, mobile apps, and desktop apps:

- Deployment may involve:
  - Uploading static assets to hosting platforms.
  - Publishing builds through distribution channels such as app stores or package repositories.
  - Making installers available to users.
- The repository should define:
  - Where build artifacts are placed.
  - Where deployment instructions and configuration for each target platform live.
  - What manual steps are needed when full automation is not possible.

AI should:

- Use the repository descriptions to guide humans through manual steps when required.
- Record deployment attempts and results in deployment workdocs so that the process can be repeated and improved.

### 6.6 Human collaboration in deployment

Humans are essential in deployment, especially for production environments:

- They review deployment plans and configuration changes proposed by AI.
- They execute deployment and rollback scripts when required.
- They control credentials and access to sensitive systems.
- They consult deployment workdocs to understand history, current state, and known issues.

AI leads the orchestration, diffing, and log analysis, while humans provide final approval and oversight.

---

## 7. Maintaining and Evolving Extended Scenario Practices

### 7.1 Keeping structures discoverable and consistent

As extended scenarios grow, it is important to keep their structures easy to discover and consistent with each other.

Recommended practices:

- Maintain a lightweight overview of the extended-scenario directories, explaining:
  - Which scenarios exist.
  - Where their configuration, scripts, and workdocs live.
- When introducing new scenarios or sub-scenarios, reuse the existing patterns:
  - A dedicated directory.
  - Scenario-local workdocs.
  - Structured configuration.
  - A small set of scripts as the bridge to external systems.

This makes it easier for AI to infer how new scenarios behave by analogy with existing ones.

### 7.2 Updating conventions safely

Over time, database, packaging, and deployment practices will evolve. To update them safely:

- Avoid scattering scenario logic across arbitrary locations. Keep entry points and configuration centralized in their scenario directories.
- When changing a convention:
  - Update the scenario directory layout and configuration structures.
  - Provide transitional guidance in workdocs or comments for affected components.
  - Deprecate old patterns gradually instead of removing them abruptly.
- Ensure that changes are reflected in both AI-facing guidance and human-facing instructions so that both can adapt.

### 7.3 Using AI to monitor and improve practices

AI can help maintain and improve extended scenario practices by:

- Analyzing scenario-local workdocs to detect recurring problems or friction points.
- Suggesting improvements to scripts, configuration structures, and documentation.
- Proposing new checks or small automation steps that reduce manual effort.

By treating these extended scenarios as first-class architectural elements in the repository, the template allows AI to remain an effective primary developer not only for core code, but also for initialization, database work, packaging, and deployment.
