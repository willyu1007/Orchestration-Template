# 讨论完善

你上述的分析和建议很全面，以下是对你报告的想法，包括分歧点、疑问、和需要进一步明确的地方。请你结合此前的分析和我进行讨论，讨论后我会逐一确认。这个阶段的是为第3阶段（形成从零开始搭建模版的具体方案）做好准备。

## 1. 待明确的地方

### 1.1 八大文档的说法并不准确（来源于不再适用的陈旧版本），不用在意该说法。这里相关的核心目标是两点，第一个是要明确模块实例目录的路径规范，实例初始化需要同步创建对应的目录和文档。包括但不限于模块实例根目录的 ROUTING.md， CAPABILITIES.md, AGENTS.md, config子目录，上下文恢复机制的workdoc子目录（结构和根目录保持一致），前后端核心逻辑等。第二个是要完善模块初始化的脚手架，让模块实例在初始完成后满足可以直接融入项目体系、规范和思想可以落位 （初始化后直接将重要信息写入模块实例的几个关键文档）、将开发需求正确转换成文档并使其满足文档体系等。核心目标不变，如果上述内容不完善，你可以继续补充。 

### 1.2  关于Quickstart文档位置，实际并不是想规定哪个具体文档的位置，而是想明确叶子文档的承上启下（向上承接ROUTING体系，向下将路由转入渐进原则的文档群），文档本意是想讨论叶子文档是否可以有类型规范（例如 quickstart、runbook、spec、guide等），但这种做法收益不大，并且还增加了路由体系的复杂程度，我觉得只需要在文档的角色说明中进行明确即可。这里不用在文档名称（quickstart或其他）和具体位置，只需要让叶子文档可以正确衔接ROUTING和渐进原则即可。

### 1.3 文档角色和命名确实十分重要，特别是在建设repo模版的过程中。例如，三个主要体系的相关文档都要求文档命名规范（文档路由体系对应 ROUTING.md，能力路由体系对应 CAPABILITIES.md，大模型策略和规范对应 AGENTS.md），上下文友好也规定了目录和目录下的文件组成（workdocs/）。核心原则是，和体系相关的面向AI的文档，都应该有规范的命名，但体系之下的，例如渐进原则的文档、contract等文档，核心要求是可以被正确检索，所以命名上只要满足格式要求即可。面向人读的文档，除了README之外，其实并没有严格的要求，毕竟这部分文档除了会被同步写入外，不会进入核心的任务编排流程。

### 1.4 系统的复杂性方面，这确实是我们在从零搭建repo模版时候需要考虑的问题。请将引导文档相关的内容视为参考，实际构建方案中需要更加稳健的流程，避免意义不明的重叠性和没有必要的复杂性。可以考虑先明确核心思路，穷举所需的基础功能和规范文档，再通过聚类或类似的方法整合基础功能和文档，在自下而上的搭建过程中，逐成检查以消除重叠性和降低复杂性。1.5. 路由层级方面，我们分为几种情况来讨论。如果开发过程几种在某一模块功能上，则上下文、知识文档路由、功能路由、策略和规范等都应该限定在模块实例的目录中（让代码大模型直接去找模块目录）。如果是跨模块的功能开发，则需要在任务编排时考虑模块实例的关系，任务编排完成后，每个步骤仍然建议通过具体的模块实例来实现，也就是先做顶层编排（路由体系是跨模块实例的），再做具体实现（具体需求路由至具体模块实例）。另外，项目初始化和模块实例初始化是两种不同的需求，我会在后文中给出相关需求和设想。

## 2. 必要文档

我们将构建repo模版的文档分为以下类别：规范文档、脚手架文档、人类阅读文档。

综述和体系路由入口，模块文档应该直接跳转到对应的承接文档，也就是整体使用指南，所有AGENTS文档都可以参考

### 2.1 规范文档
规范文档用来明确项目全周期都应该遵循的原则，这类文档的核心目标就是帮助代码大模型遵循规则，融入我们设想的体系中，并按照要求进行工作。规范类文档需要满足三种情景的使用需求：

- 说明模版的建设思路和规则规范：例如ROUTING路由体系、能力路由体系、AI友好方面的规范、注册和更新、以及其他所有引导文章中提到的思路和规范。我们需要让代码大模型可以迅速掌握核心架构思路以及使用方法。在repo模版完成后，这类文档也就确定了，通畅只会随着模版的升级而变化。我们可以对建设思路和规则规则进行预分类，每个类型应该在根目录的AGENTS.md文档中维护一个路由节点。承接路由的文档可能是独立的，也可能是ROUTING的叶子文档，但都需要按照渐进原则设计。
- 可以将模版顺利转换成具体项目：除了通用的规则外，我们还需要保证每个项目需求可以被落实成文档，并正确接入我们设计的体系中，以保证代码大模型可以按照统一的思路获取项目相关的要求。
- 可以支撑实际的开发过程：模块化开发的思路，

### 2.2 脚手架文档

有以下4类需求：
- 初始化构造（骨架 + 基本内容）
  - Orchestration Config Files （YAML/JSON）
  - policy and guide
- 添加规范 （是否有固定的格式要求，文档间的联动等）
  
- 修改/删除 规范（什么算修改）
- 使用指引（如何使用这些文档）



### 2.3 人类阅读文档

- README
- 信息记录：例如注册细节，运维记录，数据库表
- 数据流转：模块概览，数据
- 快速上手：维护一些列说明文档，可以理解文2.1章规范文档的给人阅读版本。

### 清单
- root AGENTS.md
- root CAPABILITIES.md
- root ROUNTIN.md
- README.md
- doc_agent/ROUTING.md（
- doc_agent/AGENTS.md
- doc_agent/CAPABILITIES.md
- 编排配置文件（YAML/JSON）
  - doc_agent/orchestration/registry.yaml – 编排器注册表
  - doc_agent/orchestration/capabilities.yaml
  - doc_agent/orchestration/agent-graph.yaml
  - doc_agent/orchestration/doc-node-map.yaml
  - doc_agent/orchestration/agent-triggers.yaml
  - doc_agent/orchestration/trigger-map.yaml
- Policy and Guide Documents (AI-facing, lightweight docs on specific rules or procedures):
  - Security/Compliance Policies (doc_agent/policies/*.md):
  - Operational Guides and “Quickstarts” (doc_agent/quickstart/ and doc_agent/flows/)
  - Documentation Standards Spec (doc_agent/guide/documentation-spec.md or similar)
  - Script Usage Guide (scripts/operations-guide.md and sub-guides)
- Module System Documents:
  - modules/ROUTING.md
  - Module Type Contracts (modules/<type>/TYPE_CONTRACT.md)
  - Module Instance Documentation (within each modules/<instance>/doc/ directory)
    - modules/<instance>/doc/ROUTING.md 
    - modules/<instance>/doc/AGENTS.md – Module Agent Policy
    - modules/<instance>/doc/CAPABILITIES.md 
    - modules/<instance>/doc/CONTRACT.md
    - Guides/Runbooks/Specs (module-specific)
      - Module Quickstart Guide
      - Runbook/Operations Guide:
      - Module Design Spec / Guide
      - Changelog (module-specific)
      - Known Issues / Lessons Learned
    - All module doc files will have full front matter specifying audience, purpose, doc_role (e.g. guide, spec, runbook, etc.)
  - modules/registry.yaml 
    - id: a unique identifier (e.g. <type>.<instance> name),
    -	type_path and instance_path: directory paths,
    -	route_refs: pointers to the module’s key docs (quickstart or README links, etc.),
    -	graph_bindings: how this module’s agent is connected in the global graph (e.g. which parent node it attaches to),
    -	capability_refs: links to any capabilities provided,
    -	requires / provides: dependency metadata (which other modules or global services it relies on or offers to),
    -	related_modules: siblings or alternatives,
    -	ownership, status (active/deprecated), doc_set completeness, last_verified_at, etc.
    This registry is updated whenever modules are added/removed or changed, and serves as the basis for syncing the orchestrator’s view of modules
  - doc_agent/orchestration/module-registry.yaml
- AI Workdocs and Logs
  - Global Workdocs (ai/workdocs/active/…) 
    - plan.md
    - context.md 
    - tasks.md 
    - context/ subdocs
    - These workdocs are maintained by the AI (with human oversight when needed) to recover context and ensure continuity between sessions.
  - Module Workdocs (modules/<instance>/workdocs/active/…)
  - handoff-history.md – A global handoff log that records important transfer points, such as approvals, phase completions, or when development is handed over between AI and humans.
  - Maintenance Reports (ai/maintenance_reports/*.md)
    - route-health.md 
    - retrospective.md
    - Evolution or progress logs 
    - These reports are primarily for maintainers to gauge system health and for the AI to plan optimizations.
- Ops/Evaluation Documents 

### 2.3 面向人类




规范文档，路由等基础设施的脚手架搭建


## 初始化相关
项目初   
模块实例初始



### 初始化功能

- 脚手架命令 (make ai_begin MODULE=<name>)
- 文档/注册表同步脚本（python scripts/module_registry_sync.py）
- 文档-图映射同步（python scripts/doc_node_map_sync.py）
- 路由检查（运行 route_lint 目标脚本 / doc_route_check.py 脚本）
- 能力索引一致性检查（make capability_index_check / 脚本）
- Orchestrator 注册表检查 (python scripts/registry_check.py)
- Agent Lint (python scripts/agent_lint.py)
- 代理图结构检查（python 脚本/agent_graph_check.py）
- 触发器映射同步（python 脚本/trigger_map_sync.py）
- 触发器一致性检查（将 trigger_check 设置为目标）
  - Runs a dry-run simulation of all triggers using trigger_runner.py
  - Possibly scans tasks.
  - Checks for “孤立 trigger” (orphan triggers with no policy or no matching events) and would alert if found 
  - The results (trigger simulation outcomes, orphan check) are logged to route-health.md and any issues need to be fixed or documented.
- 触发执行脚本（python scripts/trigger_runner.py）
  -	It listens for or is called with specific events (like a file commit or a user command).
	-	On an event, it finds the corresponding trigger in agent-triggers.yaml, loads the associated policy section from an AGENTS.md (Trigger Handling section), and preloads any specified docs (e.g. relevant SOPs)
	-	If required_commands are specified (like running tests or lint first), it executes those (possibly via PreToolUse hook)
	-	It then invokes the target agent or tool (e.g. calls the entrypoint of the capability specified).
	-	After execution, in the PostToolUse phase, it logs the output and any files changed to the context logs (context.md#Automation and context/active-files.md)
	-	In dry-run mode, it would simulate these steps without making changes, to validate that the flow is set up correctly
	-	This tool is essential for automating responses to triggers (like auto-running DB migrations check when a new migration file is added, etc.)
- Guardrail Execution Script (python scripts/guardrail_runner.py):
  - On a critical event (e.g. code about to connect to prod DB, or a PR hits a blocked trigger), this is called to handle the approval sequence ￼ ￼.
	-	It loads the relevant guardrail policy from AGENTS.md (or Trigger Handling for that trigger) ￼, ensures required_commands (like extra lint or safety checks) are executed and passed ￼.
	-	It then helps prepare the approval bundle: gathering diff, test results, risk assessment into the context.md#Approvals section ￼ ￼ for the human approver to review.
	-	It waits or signals for approval input (this may be out-of-band from AI, but the process is documented).
	-	If approved, it logs the decision (with rationale, rollback plan) in context and plan docs ￼ ￼, updates tasks (marks Guardrail: task as completed) ￼, and allows the originally blocked action to proceed.
	-	If rejected or requiring rollback, it ensures the rollback script (defined either in policy or as per approver notes) is executed and logs the outcome ￼.
	-	Dry-run (--dry-run all) mode will simulate the above for all defined guardrail triggers to ensure the process can complete automatically (i.e., all required commands exist and pass, all references are correct) ￼.
	-	This function is crucial to maintain a closed-loop approval mechanism and is tightly integrated with the AI’s workflow (the AI must pause and wait for approval where required, then read the approver’s decision from the context to continue).
- Context Usage Tracker (python scripts/context_usage_tracker.py):
  - Trigger hit rates (how often each trigger fired over time, how often guardrails were invoked, etc.) ￼.
	-	Average execution times per agent or tool, failure rates, etc.
	-	These stats are then written to ai/maintenance_reports/route-health.md for analysis ￼.
	-	This helps identify hotspots or inefficiencies in the orchestration, e.g. if a particular path is very slow or a trigger is firing too frequently.
- AI Chain Optimizer (python scripts/ai_chain_optimizer.py)
  - E.g., if it finds an 80% hit rate on one path, it might suggest making that path more direct (less hops).
	- Or if certain nodes often fail and fallback, propose improvements or splitting/merging nodes.
	- It would write recommendations to route-health.md or a similar report.
- Schema/Contract Validators：
  - Database Schema Linter (python scripts/db_lint.py and make db_lint): 
	-	Migration Check (make migrate_check / part of db agent tools):
	-	Rollback Check (make rollback_check):
	-	These DB-related commands are likely integrated with the Data & Schema agent and listed in its tools_allowed in registry (e.g., make db_lint, make migrate_check, etc.) .
	-	API Contract Checker (python scripts/type_contract_check.py and make contract_compat_check): 
- Continuous Integration (CI) Composite Commands:
  - make dev_check.
	-	make registry_gen.
	-	make guardrail_check
	-	make module_health_check
	-	Periodic Cron/CI tasks: Weekly or periodic tasks like:
    - Running make trigger_check and make guardrail_check to produce updated telemetry.
  	- Exporting snapshots of trigger-map.yaml to check for consistency manually ￼.
	  - Checking for any “孤立” (orphaned) docs or nodes not covered by any route (perhaps part of route lint already).
- Config Lookup Utility (python scripts/config_lookup.py and make config_show)
- Visualization and Reporting Tools
  - The guide mentions a make trigger_visualizer to output a diagram of the agent graph or trigger map. 
	-	Similarly, after all setup, generating an Agent Index or summary (maybe a Markdown table of agents and their roles) could be useful, though not explicitly mentioned, it could be derived from registry.yaml for documentation.
	-	If needed, a Telemetry Dashboard script could compile the route-health metrics into charts or more digestible reports, but that may be beyond initial scope and left for future.
