# 任务：
> 我们要建立一个模板类repo，不涉及任何业务流程。默认编程语言为 python + vue3， 数据库为postgresql + redits。

请你以智能体编排系统的视角，从辅助AI开发、提高代码大模型的任务编排效率、和闭环任务-检索-计划-编码-写入-上下文的链路三个方面，帮助搭建一个项目模板。在模板搭建过程，你需要保证文档和代码的质量。

下述子章节定义了几个主要的考量维度，请你阅读后评估内容和思路并进行完善，制定完成任务的方案，明确执行阶段。对于每个阶段，需要明确执行步骤和验收标准。将方案以文档的形式放在 @ai/maintenance_reports。你也可以在目录下创建其他文档，用于思考和记录（上下文）。每执行完一个阶段，回顾方案文档，然后更新文档（mark 验收标准，标明准备执行下一阶段等）。    

##  1. 文档规范
> 该步骤的主要目标为建立健全的大模型文档操作规范 ，包括格式要求、路径规范、关联操作、特殊定义说明等。
> 重点说明：repo的核心要素是AI友好，所以文档要优先保证

文档路由体系由各级 `ROUTING.md` 构成，负责告诉智能体去哪一层、哪个目录能找到主题相近的资料；真正承载内容的文档是路由的叶子节点。与此同时，`AGENTS.md`（doc_kind: agent_policy）专门向代码大模型说明本目录的安全边界、职责，以及如何与路由/编排体系配合；`CAPABILITIES.md`（doc_kind: capability_index）则列出该目录暴露给编排系统的能力节点、脚本入口或工具封装，便于能力检索。智能体到达叶子文档后，需要对照 front matter（如 `audience`、`purpose`、`doc_role`、`doc_kind` 等）确认自己加载的是正确目标，再继续阅读。  
渐进式原则则解决"怎么读"：抵达正确的位置后，只取当前任务需要的小文档或段落，避免一次性加载整套长文档，并在工作文档中记录已读信息以便后续恢复。

- 基本格式：
  - 需要支持UTF-8编码
  - 生成和修改文档必须使用英文（用户要求除外）
  - 除信号语义外（如✅， ❌， ⚠️，和系统风格的状态指示外），不允许使用颜文字，明确使用范围颜文字的使用情形和范围。

- 阅读对象要明确区分：文档要分为给人读的和给AI使用的。给AI使用的文档会形成文档路由体系（详见下一章节），所以需要区分文档角色和明确文档职责范围。
  - 给人阅读的文档：没有字数限制。需要在文档顶部以统一的格式明任务编排/智能体编排系统不用阅读。
  - 给AI阅读的文档：要保持轻量化（行数 < 150），使用结构化的语言风格，要注意遵循内容渐进式的原则（详见下文）。
  - 文档需要在顶部明确阅读对象（AI / human， 不允许both），职责范围，purpose，以及 `doc_kind`（router / agent_policy / capability_index / guide / spec / quickstart / workdoc 等）。
  - 面向AI的文档应保持轻量化，路由语义准确，内容分节清晰。路由说明详见第二章；`agent_policy` 文档须保持 ≤200 行，突出可/不可执行操作与 handoff 规则；`capability_index` 文档需在 250 行内描述能力拓扑、触发条件与入口。
  - 文档角色（面向AI的）：Router（即 ROUTING.md）、Agent Policy（AGENTS.md）、Capability Index（CAPABILITIES.md）、Guide、Specs、quickstart、规范、合约/契约、规程、脚本说明、Changelog 等。可以根据整体情况补充角色并在 `doc_agent/orchestration/doc-node-map.yaml` 中登记其与能力标签的映射。
  - 文档职责（面向AI的）：以Agent Docs为例，职责可以参考下述信息："ORCHESTRATOR: maintains routing, scope, and priority rules; MODULE AGENTS: enforce local guardrails and context for module tasks. SERVICE AGENTS: provide domain-specific protocols for schema or database work"。和文档角色一样，在阅读整体需求后，你需要自行完善。
  

- 渐进式原则：
	- 入口轻量：面向AI的主入口或具有路由功能的文档（如 `ROUTING.md`）只保留最小必需信息：角色、上下文路由、常用命令。不塞背景故事或长篇说明，避免一次加载过多token。
	- 逐层引用：把细节拆到下游文档，让上层通过路由/链接指向。当智能体有具体任务时，再加载对应的 quickstart、guide、spec 等文件。
	- 文档分层清晰：高层负责"做什么、去哪找"；中层提供操作手册；底层才给完整细节（流程、参数、示例）。结构越清楚，AI 越容易按需阅读。
	- 受众区分：AI 文档偏命令式、数据化；人类文档可以更长，但也尽量保持结构化。不要混杂在同一份文件中，以免上下文膨胀。
	- 保持入口精简：路由只指到下一层节点，不在同一文件重复列出全部内容；删繁就简的同时，用标题、表格、清单提升可搜索度。
	- 持续回收整理：更新流程或目录时同步调整路由，确认"重心文档"仍短小、可读。提醒通过工具定期检查。
	- 错误与经验记录独立化：遇到异常或经验总结，写入专门的 workdoc/ledger，而不是堆在入口文档里，保持基础文档不被杂讯拖长。

	- 写入规范
		- 路径要求：AI相关文档应该遵循路由规范，常见地址为doc_agent/下对应的类别，模块内部的 modules/<*name>/doc/等。面向人类的文档放在 doc_human/ 或其他有人工阅读需求的地方，除README.md外，要与AI文档分目录，避免混放。工作文档和临时上下文目录为ai/workdocs/(active/archive)。报告类文档放入 ai/reports(active/archive)。评估与基线类文档放到 ops/evals/中，并要避免误加载。
	- 报告类文档不要使用主观评估（如"预计效率提升xx%"等），要保持精简，描述事实、指标、后续计划，阐述重点，保持可读性。
	- 有较大变动时，在相关 CHANGELOG / workdoc 中记短摘要，方便溯源，并明确修改时间
	- 引入其他文件要使用相对路径
		- 入口文件分层维护：`ROUTING.md` 只写路由树与常用命令；`AGENTS.md` 只写策略、边界、工具使用守则；`CAPABILITIES.md` 只写能力列表、graph 节点、脚本/服务入口。若目录没有可执行智能体，可省略 `AGENTS.md` 与 `CAPABILITIES.md`；若仅提供策略或能力而无路由，则仍需在上层 `ROUTING.md` 中登记"只读策略/能力索引"路径。
	- 优先使用标题、表格、步骤列表等结构化数据；禁止大段散文
		- 文档正文使用Markdown(.md) ，必要时可附带 JSON/YAML 代码块，但主文件不要改用 HTML/Docx，搭配 YAML front matter（用于声明文档头信息）；配置与路由统一使用YAML，新增字段要遵循现有schema；提示词/模板可以选用Markdown（说明和示例）或YAML（结构化变量定义），但需要在规范中写明哪类文件用哪类后缀；如果专门定义JSON Schema或其他协议文件，在schemas/或对应目录，用.yaml或.json，并在 README / ROUTING.md / AGENTS.md / CAPABILITIES.md 中链接。不要混用富文本格式（word，HTML）或非结构化纯文本，保持UTF-8编码。
	- 文档顶部需要包含格式统一的 front matter， 如果文档内容发生变化，需要检查front matter是否也需要变更
	- 被路由指向的叶子文档必须在 front matter 中声明 `audience`、`purpose`、`doc_role`（例：quickstart/guide/spec/contract），并推荐增加 `route_role: leaf` 供自动校验；命名建议与角色一致（如 `quality-quickstart.md`、`db-spec.md`）；智能体读取时需比对这些字段确认目标正确

### Workdocs 上下文体系 vs 通用知识体系
- **Workdocs（上下文恢复）**：位于 `ai/workdocs/`（项目级）与 `modules/<instance>/workdocs/`（模块级），只承载任务执行过程中的 plan/context/tasks、决策、风险、错误复盘等即时信息，遵循第 10.3 章节的 Hub + 子文档结构。Workdocs 不纳入 `context_routes`，不会参与 doc-node-map/registry 校验，也不对外提供通用知识；其读者限于当前任务的执行者，用于恢复会话历史与审批链路。
- **通用知识体系**：位于 `doc_agent/`、`doc_human/`、`modules/<instance>/doc/`（或其他 `knowledge/` 目录）等，包含 ROUTING/AGENTS/CAPABILITIES/contract/guide/spec/quickstart 等治理文档，是文档路由与能力编排的唯一数据源。所有知识文档都必须有 front matter、被 `context_routes` 索引，并与 `doc_node_map`、`registry.yaml`、`agent-graph.yaml` 保持同步。
- **边界与同步**：Workdocs 可以引用知识文档来说明“已读/待办”，但知识文档不得依赖 workdocs；任何长期策略、标准或可复用经验必须沉淀在通用知识体系中，再由 workdocs 记录其使用结果。流程变更顺序固定为“知识文档更新 → 路由/能力映射同步 → 在 workdocs 记执行日志”。

  - 命名规范（区分大小写）
    - 路由文件统一使用 `ROUTING.md`，智能体策略文件统一使用 `AGENTS.md`，能力索引文件统一使用 `CAPABILITIES.md`，人类阅读的概览统一使用 `README.md`；使用说明类文档命名不允许使用其他名字（如USAGE.md）；
    - 临时文件必须以"_temp"结尾，并放入temp/目录。
    - 文档、文件、目录、脚本使用kebab-case，变量/函数使用camelCase，类/类型/组件使用PascalCase，常量与环境变量使用SCREAMING_SNAKE_CASE，数据库（表/列）使用snake_case
    - 统一「主题-类型」或「主题.类型」后缀表达语义角色，避免同名冲突
  

- 相关脚本或工具
  - 如果生成/删除/位置调整/重命名文档，需要有工具确保 .aicontext/索引随内容变化更新，文档路由体系无错误
  - 需要定期清理过期文档和零时文件
  - 文档规范性检验，如开头必须有purpose和audience等
  - PR相关的例行检验
  - 执行清单需要随开发过程明确
  
- AI使用时：
  - 要提醒AI所有文档都是英文，不要直接使用其他语言进行内容检索
  - 优先读取入口文件，并遵循front matter
  - This repository adopts"三件套"入口：`ROUTING.md` 作为 canonical router entry，`AGENTS.md` 作为 canonical agent-policy entry，`CAPABILITIES.md` 作为 canonical capability entry。它们与 `claude.md`、`gemini.md`、`agnets.md`、`CLAUDE.md`、`GEMINI.md` 等模型特定入口等价，均遵循相同的路由/策略/能力守则。

上述规范要和可以通过融入文档路由体系和功能路由体系（详见下章），需要在doc_agent目录下维护一个整体规范入口，然后根据情形维护topic + when的路由（说明不同 doc class 的目的、写作重点、路由方式，这样模块作者在选目录时不需再翻其它文件）。

### 渐进式原则操作规范
1. **定位入口**：加载根或局部 `ROUTING.md`，根据 `context_routes` 找到匹配的 scope/topic/when；若 topic 标记需要策略，则同步加载相邻 `AGENTS.md`；若需要执行能力或脚本，再查阅 `CAPABILITIES.md` 中的 `graph_node_id`、entrypoint 与依赖。
2. **核对文档**：到达叶子文档后立即检查 front matter（`audience`、`purpose`、`doc_role`、`route_role` 等），若不符合期望则回到上一层路由重新定位。
3. **按需阅读**：优先阅读 quickstart/summary 等轻量文档；若需要更多细节，再逐层深入到 guide/spec/contract，避免一次性加载整套长文档。
4. **记录进展**：在 workdoc 或任务清单中记录已读取文档及关键结论，为后续会话和人工接力提供上下文。
5. **持续收敛**：流程或文档变更时，更新路由、叶子 front matter 以及分层结构，保证入口精简、内容拆分明确。

## 2. 文档路由体系
> 通过不同目录下的 `ROUTING.md` 文档建立路由体系，使用三级引导结构（scope -> topic -> when）帮助智能体编排合理加载上下文。每个目录若存在可执行/可管控的智能体，需要再补充 `AGENTS.md`（doc_kind: agent_policy）说明安全边界，以及 `CAPABILITIES.md`（doc_kind: capability_index）说明能力节点、入口与 graph 对应关系，但不在这些文件里重复 `context_routes`。

`context_routes` 字段由 `ROUTING.md` 维护：按照主题先后指引智能体进入正确的目录或文档类别，相当于"告诉你去哪个楼层、哪排书架"。叶子文档不再维护自己的 `context_routes`，而是通过完备的 front matter 与渐进式操作规范来承接后续的按需阅读。`AGENTS.md` 可以在 `target_docs` 中被引用，作为"进入该能力前必须阅读的策略"；`CAPABILITIES.md` 可以以"capability_index" 角色被引用，提示智能体加载能力索引后再做编排。

在保持轻量的前提下，可用 YAML block 表达 scope→topic→when→target_docs 的层级关系，示例如下：
```yaml
context_routes:
  - scope: Execution – Quality Gates
    topics:
      - name: Pre-commit Checks
        when:
          - description: preparing to run make dev_check
            target_docs:
              - path: /doc_agent/quickstart/quality-quickstart.md
              - path: /scripts/README.md
```

- 功能需求
根据功能需求来设计更加符合实际使用要求的文档路由体系。
  - 开发执行
    - 项目初始化：规划全仓初始流程、依赖安装与目录约定。
  	- 模块初始化：通过脚手架创建模块、同步八大文档与注册信息。
  	- 模块实例的后端开发：实现 API、业务逻辑、契约更新的操作指南。
  	- 模块实例的前端开发：前端框架、组件规范、构建流程的入口。
  	- 模块实例的核心逻辑开发：跨后端/前端的核心算法或服务实现参考。
  	- 工作流模式与提示词模板：标准化任务执行脚本、推荐清单。
  	- 测试与质量（含自动化链路）：测试分类、覆盖要求、CI 校验流程。
  	- AI运行/推理服务（智能体）：部署、监控与维护推理服务的说明。
  - 治理与安全
  	- 智能体编排规则：root 级 orchestrator 原则、上下文调度策略。
  	- Guardrail机制与触发策略：触发规则、禁止/警告/建议操作及解封流程。
  	- 接口/契约维护：签名变更、兼容性校验与 baseline 管理。
  	- 数据库操作与变更管控：Schema 建模、迁移、回滚、审批要求。
  	- 配置管理与环境切换：多环境配置、schema 校验、敏感项处理。
  - 系统资产
  	- 数据流向监控与性能分析：数据 trace、瓶颈分析、可视化命令。
  	- 上下文资产：workdocs、错误记录，任务状态等文档的管理、迭代、与巡检。
  	- 脚本与自动化工具：Makefile/脚本目录与命令入口、维护规范。
  	- 文档规范与路由索引：文档角色、front matter 规范、路由登记与巡检办法。

- 路由体系的建设思路
	- 每个具有子结构的目录应维护一个 `ROUTING.md` 作为路由节点；若目录下仅包含叶子文档，可由上层路由直接指向，无需再增加中间层。
	- 整体呈树状：根目录 `ROUTING.md` 是根节点，子目录的 `ROUTING.md` 充当楼层/房间入口，叶子文档负责提供具体内容且不再维护路由。
	- 路由优先指向下一层 `ROUTING.md`；如需直达叶子文档，可在 `target_docs` 中列出并标注文档角色，保证智能体既能深度导航也能快速命中。若某目录存在 `AGENTS.md` 或 `CAPABILITIES.md`，应在 `target_docs` 中分别以 "Agent Policy" 与 "Capability Index" 身份引用，提示先阅读策略或能力索引再进入工具。
	- 如果路由出现过长链路或重复指向，应回顾目录 `ROUTING.md` 的设计，寻找可优化的结构，同时确认相关 `AGENTS.md` / `CAPABILITIES.md` 是否仍需单独存在。

- 引导结构的设计思路

  - 引导结构可以灵活设计，以适应不同 `ROUTING.md` 的差异化需求。在符合要求的前提下，不同的 `ROUTING.md` 可以有独立的引导结构
  - 一级结构为scope，是生命周期或跨模块的工作场景，描述当前路由节点所覆盖的"责任域"或"工作阶段"。智能体编排系统应该先判断任务类型/阶段，然后找到对应的scope，再在该范围内寻找合适的topic
  - 二级结构为topic，是Scope下的具体职责或能力单元，聚焦"要处理什么类型的问题/任务"。topic将广义责任域进一步拆分成可路由的能力单元，避免在同一 scope 内混杂多个目标
  - 三级结构为when，对应"在什么情境下需要加载这一组文档/资源"；一句话说明场景与目标。让智能体根据具体需求匹配合适的文档集合。
  - 每一层级都可以包含多个下级结构（每个scope可以包含多个topic，每个topic可以包含多个when，每个when可以包含多个 `target_docs` 条目）
  - scope名称保持精炼（常用生命周期、治理域或资产域），topic使用动名词/名词短语，指向明确的职责，when可以使用"在xxx情境下，需要..."的句式，包含触发条件和阅读目的。
  - 上级结构可以共享下级节点，但需要避免循环或意外加载。维护时可以考虑自下而上的方式（path是唯一的，可以从path出发，向上维护）
- `ROUTING.md` 只维护路由关系，不会描述叶子文档内部结构；叶子文档需要通过 front matter 与渐进式规范来承接阅读。路由的详细信息可以由 README、registry 导出的索引或其他辅助视图统一维护（详见"提高准确性"）；`AGENTS.md` 则沿用统一 front matter 和 `agent_policy` 模板，只描述安全/能力规范；`CAPABILITIES.md` 用统一的 `capability_index` 模板描述能力节点、entrypoint、graph 对应关系。

完整的引导结构为 scope→topic→when；topic、when 为必填，scope 通常用于根级或跨域目录的 `ROUTING.md`。根级与跨域目录需要显式声明 scope，单一职责目录可继承上级定义。

- 提高准确性
  - 跨文档交叉检验，每个文档都应该有front matter, purpose 应该与topic对齐（topic字面应该包含 doc purpose 的（某个或多个）关键词），如果leaf文档更新purpose， 则需要触发相应路由进行调整
  - 鼓励在 when 中引用命令或触发事件，而不仅是抽象描述，如 when: "Before running database migration scripts"。
  - 变更流程管控，路由/Agent 变更必须附带 checklist：更新文档、运行 lint、同步可视化表格、在 workdoc 中标记。
  - 在根级README中维护一个路由文档表，新增的面向AI的文档需要在README中注册登记，避免存在孤岛；同时在 `doc_agent/orchestration/doc-node-map.yaml` 中登记 `ROUTING.md` topic/when 与 `AGENTS.md`、`CAPABILITIES.md` 中 `graph_node_id`、`capability_tags` 的映射，方便 lint。
  - 统一抽象层级，维护一个文档（always read），用于定义标准 scope 范畴（比如生命周期、治理域、资产域），要求所有 `ROUTING.md` 明确继承或声明等价 scope，避免各自随意命名；相邻的 `AGENTS.md` 与 `CAPABILITIES.md` 仅引用这些 scope，不可自建新称谓。
  - 结构化约束 Topic/When，限定字段必填、值域、长度、关键词等，用关键词表（如 schema change、guardrail update、pre-commit）限制取词，减少同义词混乱。编写 when 时，要聚焦"识别条件 + 阅读目的"。
  - 提供检验脚本，加入语义检验（例如 when 是否包含触发词、Topic 是否与下游 doc 的 purpose 匹配等）。当叶子文档的purpose变更时，需要运行脚本并检查对应的when是否仍然准确。

  
- 脚本和触发规则
		- 每次目录结构更新时，需要判断是否有新增/删除的 `ROUTING.md`、`AGENTS.md` 或 `CAPABILITIES.md`；若路由节点变化，需要同步调整上下游 `context_routes`、doc-node-map 与策略/能力文档
	- 每次有文档新增时，需要判断是否需要加入路由体系（作为leaf节点）。每次有文档删除时，需要判断文档是否为路由体系的leaf节点，如是则需要对路由链路进行检查和调整
	- 手动调用路由体系检验
	- 路由同步机制，在工作流/guardrail变更时同步根路由，以避免孤立 agent
		- 触发配合：要与智能体编排的trigger对齐，为guardrail、workflow 补充 when 描述。例如database 触发器命中时，明确指向db_engines，并同步对应 `CAPABILITIES.md` 条目。
		- 整理路由维护 SOP，形成"修改目录→更新 `ROUTING.md` →（如目录存在智能体）同步 `AGENTS.md` / `CAPABILITIES.md` → 跑校验→记录"的闭环。该SOP文档可以按照渐进式原则整理。
		- 任何入口变更（新增/调整等）scope/topic/when/path 应该先更新相关的总览表（README.md），然后再修改对应的 `ROUTING.md`、`AGENTS.md` 与 `CAPABILITIES.md`。topic和scope的调整需要写入 workdoc，保持溯源。
  
- 注意事项：
		- 要考虑文档内容的渐进原则（见第一章）。例如，假设产品开发分为S1-S5五个阶段，开发规范文档内容会将这5个阶段拆开，路由可以根据当前开发阶段（when的描述），找到合适的文档。如果最终的引导无需在本层 `ROUTING.md` 完成（例如 `ROUTING.md` 指向统一规范说明），则叶子节点的文档需要给出进一步的路由提示；若需要安全策略或能力入口，则在 `AGENTS.md` / `CAPABILITIES.md` 中补充引用。
	- 根 README 增表,格式如下：
		``` markdown
		| Scope | Topic | When (触发场景) | 下游入口 |
		| --- | --- | --- | --- |
		| Execution - Quality Gates | Pre-commit Checks | 准备提交前 | file path |
		| Governance & Safety | Guardrail Management | 调整 trigger 配置 | file path |
		| … | … | … | … |
		```
  更新根 README 表时，同步检查子目录缩略表是否需要变更。
	- 子目录也可以按照上述规则维护局部缩略表（大部分不需要scope）。这些表格在更新路由时（维护SOP）需要同步更新，形成流程约束，更新方式需要在维护SOP文档（或下沉文档）中说明。
	- 可以维护一个编写指南（含标准命名、正反例、共享 topic 的处理方式），在新目录或模块接入时引用，保证结构演进的一致性。

## 3. 功能路由体系（编排图）

### 3.1 功能节点规范
- **统一文档格式**
	- 每个智能体在所属目录维护 `AGENTS.md`（doc_kind: agent_policy），front matter 必须符合下列 schema（未列出的键视为禁止）：
	```yaml
	front_matter:
		audience: "ai" | "human"
		role: string               # 角色名称，如 orchestrator、docops
		purpose: string            # 单句描述，控制在 120 个英文字符以内
		ownership: string          # 负责人或团队标识
		updated_at: YYYY-MM-DD     # ISO 日期
		graph_node_id: string      # 对应 agent-graph.yaml 中的节点 id
		doc_kind: "agent_policy"
		tags?:                    # 可选，枚举自预设关键词表
			- string
	```
- `AGENTS.md` 不再嵌入 `context_routes` 或路由说明；这些信息由同级 `ROUTING.md` 维护。`AGENTS.md` 只描述策略、工具使用界限、必要的 inputs/outputs。若策略发生变更，应通过 `doc_agent/orchestration/doc-node-map.yaml` 将新的 `graph_node_id` 与路由 topic 关联。
- 局部 `AGENTS.md` 不得新增 `target_agent` 字段，也不再冗余列出 `tools_allowed`；工具清单统一维护在 `doc_agent/orchestration/registry.yaml`，供编排图和校验脚本使用。需要引用本地脚本时，只能在正文中描述"先跑 X，再参考 ROUTING 中的 Y"，而不在 front matter 中登记。
- **能力索引（CAPABILITIES.md）**
	- 每个暴露能力/脚本/服务的目录维护 `CAPABILITIES.md`（doc_kind: capability_index），front matter 建议包含：
	```yaml
	front_matter:
		audience: "ai"
		role: "capability_index"
		purpose: "Catalog callable abilities under <scope>"
		ownership: string
		updated_at: YYYY-MM-DD
		scope: string              # 继承自 ROUTING.md
		doc_kind: "capability_index"
	```
	- 正文以 scope→capability→trigger 的结构列出 `graph_node_id`、`kind`、`capabilities`、`entrypoints`、`inputs/outputs`、`constraints`、`related_docs`。`target_nodes` 字段可直接引用 `agent-graph.yaml` 的节点 ID，并在 `doc-node-map` 中建立 ROUTING→CAP→Graph 的映射。
	- 若能力仅是封装脚本或 prompt，也应在此声明所依赖的 schema、Makefile 目标、安全边界，并指向对应的 `AGENTS.md`/guide/spec。
- **注册清单**
	- 所有 Agent/工具 的注册接口统一集中在 `doc_agent/orchestration/registry.yaml`；根 `AGENTS.md` 只负责指向该清单而不再罗列明细。
	- 新增、调整或下线 Agent 时，必须先更新 registry，再同步本地 `AGENTS.md` 与能力图（金字塔顺序为：registry → 能力图 → 局部文档）。
- **节点元数据要求**
	- `id`：全局唯一，推荐使用 `domain-capability[-module]`。
	- `kind`：`router`、`planner`、`tool`、`service`、`memory`、`evaluator`、`human_in_loop` 等；其中 `tool` 视为轻量化智能体，具备单一能力，可被 `target_agent` 直接引用。
	- `entrypoint`：声明调用方式（http/grpc/local/fn/queue）与 URI/命令。
	- `capabilities`：枚举智能体或工具可执行的原子能力，并与编排层登记的 `target_agent`、`capability_tags` 保持一致；若没有对应 target_agent，则视为仅提供文档指引，交由下一层或人工执行。
	- `inputs` / `outputs`：端口名称与引用的 schema（位于 `schemas/`），保证图中数据契约统一。
	- `constraints`：含 `max_latency_ms`、`cost_budget` 等非功能指标；高风险节点需补充安全策略。
	- `policy`、`observability`：定义重试、超时、可观测信号（metrics/logs/traces/events），由 Guardrail/Telemetry 智能体监控。
- **集中式 target_agent 配置**
	- 跨智能体 handoff 只在编排层文件维护：`doc_agent/orchestration/agent-triggers.yaml`、`doc_agent/orchestration/routing.md`、`doc_agent/orchestration/registry.yaml`。
	- 本地 `AGENTS.md` 仅负责策略说明与本地工具列表摘要，禁止直接写入 `target_agent` 或映射表；如需指向能力节点，请在编排层登记后以 `graph_node_id` 引用，并通过 `doc-node-map` 将该节点与 `ROUTING.md`、`CAPABILITIES.md` 的条目关联。
	- 变更顺序：先在编排层注册/更新 target_agent → 更新根 README 模块路由表与对应 `ROUTING.md` → 返回相关目录 `AGENTS.md` / `CAPABILITIES.md` 补充策略或能力说明 → 运行 lint/校验脚本。
	- **角色划分**
		- **Orchestrator 节点**：根 `AGENTS.md` 描述 orchestrator 节点的策略与安全边界；真正的 scope→topic→when 匹配由根 `ROUTING.md` 承担，两者需在 `doc-node-map` 中声明映射。
		- **能力域节点**：DocOps、Guardrail、Module Lifecycle、Workflow & Telemetry、Data & Schema 等子智能体映射为独立节点，各自管理工具端口。
		- **模块节点**：`modules/<name>/AGENTS.md` 映射模块专属节点，声明 `owned_paths`、模块工具端口以及与通用节点的依赖；`modules/<name>/CAPABILITIES.md` 则记录模块对外暴露的能力和工具入口。
		- **功能工具节点**：脚本或 API 封装成 `kind: tool/service` 节点，可被多个上游智能体引用，责任人登记在 `registry.yaml`.
		- **能力索引节点**：每个拥有 graph 节点或复合脚本的目录在 `CAPABILITIES.md` 中登记 scope→capability→trigger，对应 `agent-graph.yaml`、`registry.yaml`、`agent-triggers.yaml` 的条目，形成可检索的能力地图。

#### 基础能力 vs Agent 双层体系
- **基础能力（Base Capability）**
  - 载体：`doc_agent/orchestration/capabilities.yaml`（机器可读）与各目录 `CAPABILITIES.md`（可读索引）。
  - 内容：定义脚本、命令、API、提示模版等原子能力，字段包含 `capability_id`、`kind: tool/service/script`、`entrypoint`、`inputs/outputs`、`constraints`、`related_docs`，指向具体实现与 schema。
  - 限制：不写 guardrail/权限，仅描述“能做什么”“怎么调用”；若基础能力需要审批，引用相应工作指引或 agent policy。
- **Agent 能力（Composite Agent）**
  - 载体：`doc_agent/orchestration/registry.yaml`、`modules/registry.yaml#graph_bindings`、`agent-graph.yaml` 以及 `AGENTS.md`。
  - 内容：封装若干基础能力并附带执行守则、审批链和触发条件，字段包含 `id`、`provides`（引用基础能力 ID）、`tools_allowed`、`graph_node_id`、`scope/topic/when` 等。
  - 限制：Agent 只能引用已登记的基础能力；若需要新增工具，必须先完善能力层再更新 agent。
- **登记顺序与校验**
  1. 新增/调整基础能力 → 更新 `capabilities.yaml` + `CAPABILITIES.md` → 运行 `make capability_index_check`。
  2. 需要 agent 调度时，更新 `modules/registry.yaml#graph_bindings` 或 orchestrator registry，声明 `provides` 对应的基础能力 → 执行 `python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`。
  3. 在 `ROUTING.md`、`agent-triggers.yaml`、`agent-graph.yaml` 中引用 agent，并在 `workdocs` 记录命令与验证结果。
  4. CI / 本地必须跑 `make capability_index_check`、`make registry_gen --check`、`python scripts/registry_check.py`、`make route_lint` 以确认双层映射一致；若任一步骤遗漏，视为未完成登记。

### 3.2 编排图建模
- **图模型**
	- 采用"有向多重图"描述智能体关系，鼓励 DAG，如需循环需在 `policy` 中定义幂等与退出条件。
	- 图规格文件建议存放于 `doc_agent/orchestration/agent-graph.yaml`，结构与仓库提供的编排图描述保持一致，引用统一的 JSON Schema（建议放入 `schemas/agent-graph.schema.json`）。
	- `graph` 元数据应声明 `id`、`version`、`mode`（dag/state_machine/petri_net）、`entry_nodes`、`terminal_nodes`、调度策略（如 `parallel_if_independent`）、安全策略、审计开关。
- **边定义**
	- 边由 `from: <node>:<out_port>`、`to: <node>:<in_port>` 组成，可附带 `condition.expr`、`transform.script_ref`、`qos.priority`、`error_route` 等字段。
	- 所有条件表达式应引用 schema 字段，脚本需在 `transforms/` 目录提供实现与测试。
- **图与路由联动**
- `AGENTS.md` 与能力索引（`capabilities.yaml` / `CAPABILITIES.md`）中的 `graph_node_id` 必须与编排层登记的 `target_agent` 一一对应；`ROUTING.md` 的 `context_routes.target_docs` 则通过 `doc-node-map` 指向这些策略/能力文档及叶子内容，可由校验脚本比对。
	- `agent-triggers.yaml` 中的 `target_agent`、`preload_docs`、`required_validations` 与图的边条件保持一致，触发事件指向具体节点端口。
- **示例片段**
```yaml
schemas:
  Plan: {type: object, properties: {steps: {type: array, items: {type: string}}, intent: {type: string}}}

nodes:
  - id: docops
    kind: planner
    entrypoint: {type: local, uri: "python scripts/doc_route_check.py"}
    capabilities: [route_lint, doc_index]
    inputs:  [{port: in,  schema_ref: "#/schemas/Plan"}]
    outputs: [{port: report, schema_ref: "#/schemas/Plan"}]

edges:
  - from: orchestrator:plan
    to: docops:in
    condition: {expr: "plan.intent == 'route_audit'"}
```

### 3.3 功能抽象与图构建流程
- **双索引策略**
	- 文档路由索引：依托各目录 `ROUTING.md` 的 scope→topic→when 结构及根 README/registry 导出的索引视图，解决"去哪读文档"。
	- 能力路由索引：依托 `doc_agent/orchestration/capabilities.yaml`（能力 registry）+ `CAPABILITIES.md`（可读版索引）+ `agent-graph.yaml` 中的节点、端口与 `capability_tags`，解决"调用哪个智能体/工具/脚本"；`CAPABILITIES.md` 负责把能力语义拆成可检索的 scope→capability→when。
	- Orchestrator registry（`doc_agent/orchestration/registry.yaml`）仅登记可对外暴露的智能体/服务；capability registry 记录底层基础能力（数据库 CRUD、API 调用、脚本等），两者通过 `capability_id` 与智能体 `provides` 字段建立关系。
	- 新增能力流程：先在 capability registry 登记或更新基础能力 → 在 `CAPABILITIES.md` 说明组合方式 → 生成或调整智能体后在 orchestrator registry 注册入口 → 更新 `ROUTING.md` 与 `agent-graph.yaml`；两套索引通过 `doc_agent/orchestration/doc-node-map.yaml`、`graph_node_id`、`capability_tags` 及编排层的 `target_agent` 映射互相引用。
- **模板通用能力**
		1. 识别仓库共性工作（初始化、文档治理、触发器维护、健康检查、合规审查）。
		2. 为每类工作创建能力域智能体节点，补充本地 `AGENTS.md`（策略）与 `CAPABILITIES.md`（能力索引）、工具 manifest、schema，并在图中连至 orchestrator；对应的 `ROUTING.md` 需要加入新的 topic/when。
	3. 将常用工具封装为子节点或边上的 transform，保持输入输出契约统一。
	4. 在 capability registry 记录底层能力项，在 orchestrator registry 登记暴露的智能体入口，并通过自动化脚本或 README 视图导出轻量索引，保持入口文档精简。
  
	除上述能力外，模版通用的agent必须覆盖第2章列出的功能需求。请进一步检验
- **模块实例能力**
		1. 在 `modules/<name>/AGENTS.md` 定义模块节点（策略）并在 `modules/<name>/CAPABILITIES.md` 登记模块暴露的能力，继承/扩展 scope 词表，并在图中以子图表示。
	2. 模块专用脚本或 API 封装为工具节点，通过边连接至模块节点；如需复用通用工具，在 capability registry 中登记依赖，并在 orchestrator registry 中标明对应智能体的 `requires` 字段。
		3. orchestrator registry 维护"模块入口 → 模块子图"的映射，确保 orchestrator 可按 scope 将任务引导至正确子图，而无需在模块 `AGENTS.md` / `CAPABILITIES.md` 重复说明。根 README 仅以表格或链接形式引用 registry 生成物。
	4. 若模块与通用能力交互（如 DocOps 校验模块文档），需在图中添加跨子图边，并在安全策略中定义访问限制。

### 3.4 图驱动的路由与调度
- **决策流程**
	1. Orchestrator 根据任务描述匹配 scope→topic→when，加载 `target_docs`，按需读取 `AGENTS.md`（策略）与能力 registry（`capabilities.yaml` + `CAPABILITIES.md` 可读层），再通过编排层的 `target_agent` 映射定位到图中的起始节点与端口。
	2. 根据边条件选择下游节点，继续依赖集中式映射执行对应智能体；遇到多条 `when` 时按 `priority`、`selection_policy` 解析后确定顺序或选择策略。
	3. 调度器按图 `scheduler.strategy` 运行节点，遵循并发/串行约束；执行结果写入 `ai/workdocs/`。
	4. 若节点失败，根据边的 `error_route` 或节点 `fallback` 切换至替代路径，并记录至 `handoff-history.md`。
- **触发与事件**
	- `agent-triggers.yaml` 定义事件到图边的映射，例如：`event: db_migration_pending → from orchestrator:plan to data_schema:review`。
	- 触发器脚本（`scripts/trigger_manager.py`）需校验事件是否存在对应边，避免孤立事件。
- **可视化与索引**
	- 使用 `make trigger_visualizer` 或自定义脚本将图导出为可视化（Mermaid/Graphviz），并在 `doc_agent/orchestration/routing.md` 引用。
	- 根 README 路由表可通过 orchestrator registry 自动生成（推荐使用 `make registry_gen` 或自定义脚本），包含 `scope/topic/when`、`capability_tags`、`Graph Node/Port` 与 `target_docs` 列，保持入口文档只存放摘要。
- **安全政策/操作手册文档**
	- 存放位置：安全政策类文档集中在 `doc_agent/policies/`（AI 版）与 `doc_human/policies/`（人类版），操作手册/运行规程位于 `doc_agent/quickstart/`、`doc_agent/flows/` 或模块目录下 `doc/`。
	- 读取规则：编排系统在命中相关 `when` 时，必须先加载 `target_docs` 列出的安全政策或操作手册；若 `target_agent` 为空，视为仅提供读物，后续步骤需通过下级 `scope/topic/when` 或人工执行。
	- 更新规则：变更安全政策或操作手册时，需同步更新对应 `context_routes`、根 README 路由表、`handoff-history.md` 与相关 workdoc；重大政策变更需 Guardrail 节点复核并记录审批。
	- 必含内容：文档头部 front matter 声明 `audience`、`role`、`purpose`、`ownership`、`updated_at`；正文涵盖适用范围、执行步骤、风险/禁止项、审计/留痕要求，以及引用的下游文档或脚本。安全政策还需维护告警阈值、处罚/解封流程；操作手册需列出所需工具、输入输出、恢复策略。

### 3.5 迭代管理与维护
- **版本生命周期**
	- 每次图或路由变更需同步更新：相关 `AGENTS.md`、能力 registry（`capabilities.yaml` + `CAPABILITIES.md`）、`agent-graph.yaml`、`doc_agent/orchestration/registry.yaml`、`agent-triggers.yaml`、`handoff-history.md`。
	- 在 `ai/workdocs/active/<task>/plan.md` 制定迭代计划，完成后归档至 `ai/workdocs/archive`，并在 `ai/maintenance_reports/` 发布图演进日志。
- **自动化校验**
		- 扩展/新增脚本：
			- `agent_graph_check.py`：校验图结构、端口 schema、边条件、循环与安全策略。
			- `doc_route_check.py`：验证 `context_routes` 与图节点 ID/端口一致。
			- `registry_check.py`：确认智能体在 orchestrator registry 中登记、字段完整。
			- `capability_index_check.py`：比对 `capabilities.yaml` / `CAPABILITIES.md` 与 `agent-graph.yaml`、orchestrator registry 的节点、entrypoint、`capability_tags` 是否一致。
			- `agent_lint.py`：检查 `AGENTS.md` 与能力索引（`capabilities.yaml` / `CAPABILITIES.md`）的 front matter、`graph_node_id`，并验证 orchestrator registry 与 capability registry 的映射关系。
	- 将图校验纳入 `make dev_check`，确保 CI 即时阻断不一致变更。
- **遥测与反馈**
	- Workflow & Telemetry 节点运行 `context_usage_tracker.py`、`ai_chain_optimizer.py`，统计节点命中率、平均延迟、失败原因。
	- 遥测结果写入 `ai/maintenance_reports/route-health.md`，作为下一轮图优化、节点拆分/合并的依据。
- **治理与审计**
	- Guardrail 节点负责审批新增/变更的高风险节点或边（如访问生产数据库），并在 `handoff-history.md` 记录审批日志与回滚策略。
	- 引入 `audit` 配置（开启时必须记录 metrics/logs/traces/events），确保重要调用具备追溯线索。

### 3.6 验收指标
- 文档指标：所有 `AGENTS.md` 含统一 front matter 与 `graph_node_id`；能力索引（`capabilities.yaml` + `CAPABILITIES.md`）与 `agent-graph.yaml`、orchestrator registry 映射一致；所有 `ROUTING.md` 含 scope→topic→when→`target_docs`，并通过 doc-node-map 与图保持一致，无 lint 报错。
- 图指标：`agent-graph.yaml` 通过 JSON Schema 校验；边/节点均有责任人、schema、条件；不存在悬空节点或孤立事件。
- 路由指标：根 README 路由表、编排层 target_agent 映射与图节点一一对应；随机任务可在两跳内定位到执行节点和工具端口。
- 运维指标：遥测报告显示核心任务命中率 ≥80%、失败调用具备 fallback 或人工接管记录；`handoff-history.md` 与 workdoc 完整追踪变更。

## 4. AGENTS.md 策略规范
> 面向代码大模型的策略入口，限定可执行范围、守护机制与提交要求，确保每次能力调用都按规执行。

- **职责边界**
  - `AGENTS.md` 只描述策略、权限与执行守则，不重复 `ROUTING.md` 的路径信息，也不列举完整能力（那是 `CAPABILITIES.md` 的职责）。
  - 文档必须使用 front matter：`audience: ai`、`doc_kind: agent_policy`、`role`、`graph_node_id`、`ownership`、`updated_at`、`related_routes`（可选），方便模型快速识别。
  - 对于通过 `agent-graph.yaml` 注册的每个节点，`AGENTS.md` 都是其安全/治理说明书；智能体加载顺序为：`ROUTING` → `AGENTS` → `CAPABILITIES` → 叶子文档/工具。

- **必含内容**
  - **Access Control**：列出允许/禁止/需审批的操作、访问级别、敏感目录/命令、环境限制（如"禁止直连生产 DB"）。
  - **Execution Guardrails**：描述必须运行的 lint/test、所需的 doc / workdoc 更新、fallback 策略、dry-run/审批要求。
  - **Tools & Dependencies**：标注可调用的脚本/服务、对应 registry 条目、输入输出 schema、必须通过的封装命令。
  - **Escalation & Logging**：指定触发 guardrail 的事件、提交审批的联系人、`handoff-history.md` / workdoc 应填的字段。
  - **Related Routes & Capabilities**：引用本目录 `ROUTING.md` 中的 scope/topic/when，以及 `CAPABILITIES.md` 中能力条目的 ID，帮助模型在守则与能力之间建立映射。

- **维护方式**
  - 变更流程：registry / agent-graph → `ROUTING.md` → `AGENTS.md` → `CAPABILITIES.md` → workdoc/ledger；任何策略改动都需更新 `doc_agent/orchestration/doc-node-map.yaml`。
  - `AGENTS.md` 行数控制在 200 行内，使用表格/要点而非长段落，便于模型快速解析。
  - 建议在文档末尾附 "Agent Policy Checklist"，列出提交前需要确认的步骤（lint、tests、doc update、approvals），供模型或人工复核。
  - 若一个目录没有执行权限，仅提供文档，允许省略 `AGENTS.md`；若共享同一守则，可在上一级 `AGENTS.md` 中定义公共策略，并在子目录通过 `related_routes` 指向它。

## 5. 模块化思想

### 5.1 模块与模块实例核心概念
- `modules/` 目录是整个模块体系的根，承担“导航 + 公共资产 + 能力出口”三类职责：
  - **导航**：根目录下的 `README.md` / `ROUTING.md` 需要列出当前仓库支持的模块清单、scope/topic/when 以及各实例的入口（quickstart、contract、agent policy 等），帮助编排系统在两跳内定位到目标实例。
  - **公共资产**：可在根下维护 `common/`（共享组件、hooks、UI 库）、`api/`（统一 API 契约、客户端类型生成器）、`templates/`（脚手架片段）等子目录，为所有实例复用；这些资产同样需要在根级 `ROUTING.md`/`CAPABILITIES.md` 中注册能力节点。
  - **能力集成**：根目录负责维护模块关系图、`modules/registry.yaml`（SSOT）以及跨模块触发器映射，确保 orchestrator 能从全局视角安排 handoff；新增/删除实例时，先更新 `modules/` 根目录的索引/registry，再通过同步脚本刷新 `doc_agent/orchestration/` 侧的视图文件。

- **模块类型（Module Type）**：在 `modules/` 根目录中为每一种职责建立子目录，并以统一的命名规范描述其阶段，例如 `1_Assign`、`2_Select`、`3_SelectMethod`。每个类型都需要配套一份类型契约，明确该类实例必须履行的职责、接口与数据约束，保证任何实现都可以被等价替换。
- **层级结构（Hierarchy）**：类型之间可以形成树形结构，从业务阶段到能力集合再到实体单元逐层细化。上层节点聚焦目标与上下游，下层节点负责执行细节。通过前缀或编号即可快速推断所在阶段，适合在复杂流程中定位责任区。
- **模块实例（Module Instance）**：同一类型可存在多个实例，放置在 `modules/<instance>/` 目录。实例需要提供代码、脚本与完整的文档集（README、Agent Policy、Capability Index、Contract、Guide、Runbook、ChangeLog、Bugs、Progress 等），并在前言写明受众、目的、文档角色、责任人和版本信息，确保阅读者无需再查阅其他资料。
- **关系与扩展（Relations）**：实例之间既可以互为替换实现，也可以以 `requires` / `provides` / `related_modules` 等元数据描述协作链路。每个实例都要在元数据里声明依赖对象、输出能力和可组合的兄弟模块，方便 orchestrator 和维护者判断影响范围。
- **模块注册（Module Registry）**：初始化完成后必须在 `modules/registry.yaml` 维护模块实例的单一真相源（SSOT），字段包含：`id`（`<type>.<instance>`）、`type_path`、`instance_path`、`route_refs`、`graph_bindings`、`capability_refs`、`requires`/`provides`、`related_modules`、`ownership`、`status`、`doc_set`、`last_verified_at` 等。`doc_agent/orchestration/module-registry.yaml` 仅作为自动生成的视图，供 orchestrator 消费；更新流程为“修改 `modules/registry.yaml` → 运行 `python scripts/module_registry_sync.py --write`（或等效命令）将变更同步到 `doc_agent/orchestration/*.yaml` → 再执行 `python scripts/doc_node_map_sync.py --write`、`make registry_gen --check`”以保证 `registry.yaml`、`capabilities.yaml`、`agent-graph.yaml` 与模块清单保持一致。
- **文档与代码协同（Docs & Code）**：新增或变更实例时，目录、代码、文档和元数据必须同步更新。类型契约、实例文档和图谱/registry 等治理资产需要同一批人维护，任何阶段完成后都要打上“文档已更新”标记，再推进下一阶段，确保知识不缺口。
- **示例（业务系统）**：以审核-分配-策略链路为例，可在 `modules/1_assign/` 下维护分配阶段的类型契约，再在 `modules/1_assign/2_select/` 与 `modules/1_assign/3_select_method/` 中分别建立选择器层级。`modules/1_assign/3_select_method/M_AI_Select/` 可以是 AI 选择实例，而 `M_Manual_Select` 则是人工兜底，二者共享同样的目录结构和文档集，便于横向比较与动态替换。
- **配置体系（Configs）**：
  - **全局层（项目根或 `modules/config/`）**：集中维护跨模块共用的配置，如数据库环境、外部能力 API（LLM、Auth、Webhook、MQ）、消息/任务系统参数、日志与可观测性设置、缓存/搜索集群、Feature Flag、身份/安全策略、成本/资源限制、测试/沙箱环境等。推荐目录结构：
    ```
    modules/config/
      ├── ROUTING.md          # scope/topic/when → config 文档入口
      ├── AGENTS.md           # 配置变更策略/审批/命令
      ├── CAPABILITIES.md     # 可复用配置生成/校验能力
      ├── db/                 # DB 连接、迁移策略
      ├── llm/                # 大模型 API、prompt bucket、温度/超参等接口参数
      ├── telemetry/          # 日志、metrics、alerts
      ├── parameters/         # 全局参数
      ├── prompts/            # 提示词模版、意图识别规则（按场景分类）
      ├── feature_flags/      # 开关、回滚策略
      └── config-index.yaml   # key → 文档/路径映射
    ```
    - 提供脚本（如 `python scripts/config_lookup.py --key llm.default.model`）或 `make config_show KEY=...`，方便智能体快速检索；命令输出需要贴到 `workdocs/active/context.md#Automation`。
  - **模块层（`modules/<instance>/config/`）**：记录实例专属配置（接口限流、模块 Feature Flag、依赖服务 endpoint、局部策略、模型/函数超参、prompt override）。要求 front matter 声明 `doc_role: config_spec`，在 `modules/<instance>/ROUTING.md` 注册 `topic: Configuration`，并在 `modules/registry.yaml#config_refs` 列出配置文件路径。
  - **提示词与超参**：prompt 模版、LLM 温度/TopK 等接口参数视为配置资产，遵循相同目录与审批流程：在 `modules/config/prompts/` 或 `modules/<instance>/config/prompts/` 中维护模版，配合 `prompt-index.yaml`（intent → 路径），同时在 `capabilities.yaml` 登记生成/渲染脚本，确保可检索。
  - **结合方式**：全局目录存“权威配置 + 模板变量”，模块目录存“局部 override / 引用说明”；模块配置引用全局变量时，只写变量名与读取方式，不复制敏感值，并在 `config/index.yaml` / README 中留文档链接。更新配置需：① 修改对应 config 文档 → ② 如影响能力/agent，按基础能力→agent 登记流程同步 → ③ 在 `workdocs`、`handoff-history.md`、`ai/maintenance_reports/route-health.md` 记录命令与审批结果。
  - **治理文档**：`config/README.md`（或 `doc_agent/guides/config-governance.md`）必须描述配置命名规范、审批规则、检索命令、敏感信息处理及提示词管理流程；任何配置体系改动需先更新该治理文档，再执行上文命令，避免隐式变更。

- **信息注记**
  - 在每个 README 中写明路径导航与关联模块，帮助新接手的 AI 快速定位职责、依赖与协作边界，减少沟通成本。
  - 模块注册表是路由、图谱、触发器与自动化脚本引用的唯一真相源（SSOT）：5.2/5.3/5.4 所述的 checklist、关系构建、系统化管理都依赖该登记信息；其中 `route_refs` 只为了让文档路由知道该实例的文档入口，`graph_bindings` 只在实例确实映射到 agent/capability 时使用，二者分别链接到文档体系与能力体系，避免混淆但确保需要自动化时可快速接入。

### 5.2 模块实例生命周期

#### 5.2.1 初始化阶段
- 文档清单和目录参考内容：
  - `modules/<instance>/workdocs/` 分为 `active/` 与 `archived/` 两层，完全复用第 10 章“ai/workdocs” 的目录与角色：活跃资料放入 `workdocs/active/`，历史快照移至 `workdocs/archived/<timestamp>/`，并通过 `handoff-history.md` 记录迁移。项目级 `ai/workdocs/active/<task>/` 只负责跨模块、底层基建或流程串联等全局任务；模块实例级 `modules/<instance>/workdocs/active/` 只关注本实例的设计、实现与验证，默认彼此解耦以降低复杂度、提升编排准确性。若确需复用信息，可在任一侧以“参考”形式手动附链接，但不做强制双向同步。`workdocs/active/` 的基线结构如下：
    1. `workdocs/active/plan.md`（audience: ai，doc_role: plan，route_role: leaf）：对应 `ai/workdocs/.../plan.md`，记录模块实例的长期策略、分阶段目标、风险清单、Guardrail 要求。
    2. `workdocs/active/context.md`（audience: ai，doc_role: context_log，route_role: hub）：保持 ≤120 行的模块上下文枢纽，固定包含 `Session Progress`、`Decisions & Approvals`、`Risks & Blocks`、`Active Files`、`Learning Snapshot` 五个摘要小节，并链接到 `workdocs/active/context/` 子目录。
       - `workdocs/active/context/` 采用与 10.3 相同的拆分：`session-progress.md`（context_log）、`decisions.md`（decision_log）、`risks.md`（risk_log）、`active-files.md`（context_log）、`lessons.md`（lessons_log，对接 10.6）。模块级 lessons 记录该实例的缺陷与补救动作，项目级 `context.md` 仅引用其摘要，避免信息重复。
    3. `workdocs/active/tasks.md`（audience: ai，doc_role: task_log）：同步任务拆解、验收标准、触发器命中情况，与第 10 章的任务管理结构一致。
    4. `workdocs/active/contract.md`（doc_role: contract）：继承 plan/tasks 中定义的接口约束，声明输入输出、兼容性、回滚要求。
    5. `workdocs/active/runbook.md`（doc_role: runbook）：提供执行步骤、常见故障、触发器链接、dry-run 与审批说明，可引用 plan/context/tasks 的片段。
    6. `workdocs/active/changelog.md`（doc_role: changelog）：记录版本演进，与 `ai/maintenance_reports/`、`handoff-history.md` 互相引用。
    7. `workdocs/active/progress.md`（doc_role: progress_log）：跟踪里程碑、上下游协作状态，结构与 tasks 对齐。
    8. `workdocs/active/bugs.md`（doc_role: issue_log）：实现第 10.6 节“错误记录与学习”的存档，链接触发器与防范措施。
    9. `workdocs/active/quickstart.md`（doc_role: quickstart，route_role: hub，可选）：串联上述文档，让路由按 `ROUTING → Quickstart → plan/context/tasks → contract/runbook/bugs` 渐进加载；`AGENTS.md` / `CAPABILITIES.md` 继续承担策略与能力索引。
    9. 若模块需要投产告警或触发模板，可在 `workdocs/quickstart.md`（doc_role: quickstart，route_role: hub）中串联以上文档，对应 `ROUTING → Quickstart → plan/context/tasks/...` 的渐进式入口；同时 `AGENTS.md` / `CAPABILITIES.md` 继续作为策略与能力索引。
  - 根 README 与 `ROUTING.md` 仅注册 Quickstart / Agent / Capability 等入口，路径指向上述文档；这样既复用了第 10 章的上下文恢复布局，又保持“ROUTING → AGENT/Quickstart → plan/context/tasks → contract/runbook/bugs”等渐进链路，避免直接在路由层暴露大量叶子文件。
  - **知识文档目录**：模块的通用知识、路由叶子与契约说明统一存放在 `modules/<instance>/doc/`（或自定义 `knowledge/` 目录），该目录必须遵循文档路由体系：包含 `ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md`、`CONTRACT.md`、quickstart/runbook/spec/guide 等，均带完整 front matter，并通过类型/根级 `context_routes` 注册。知识文档供所有智能体按渐进式原则加载；`workdocs/` 仅用于上下文恢复与执行记录，不能替代知识文档，也不直接被路由引用。

**初始化注册流程（仅允许脚手架渠道）**

1. **准备需求文档**
   - 针对模块功能准备独立的需求/背景文档（若不存在则创建），记录业务目标、边界、输入输出、上下游依赖、风险约束。该文档位于 `modules/<instance>/doc/requirements.md`（示例）或其他约定位置，front matter 标注 `doc_role: requirement`。
   - 在 `ai/workdocs/active/<task>/plan.md` 登记“模块初始化”任务 ID，并链接上述需求文档，说明初始化目的与验收标准。

2. **信息提取与交互**
   - 通过脚手架前置调查表或交互（会话/异步）与需求方确认：模块职责、需要暴露的能力/接口、数据契约、依赖模块、测试与审批要求。
   - 所有可注册信息（`ownership`、`route_refs`、`graph_bindings`、`capability_tags`、必备文档清单等）必须写入 `workdocs/active/plan.md` 或 `requirements.md`；未收集齐全时，不得进入脚手架命令。

3. **运行脚手架并生成目录**
   - 仅允许通过脚手架命令（如 `make ai_begin MODULE=<instance>`）初始化模块目录；禁止手动复制粘贴。
   - 脚手架输出须覆盖：通用代码目录、`workdocs/active/` 结构、`doc/ROUTING.md`/`AGENTS.md`/`CAPABILITIES.md`/`CONTRACT.md` 等路由叶子文档，以及与渐进式原则匹配的知识文档（quickstart、runbook、lessons 等）。
   - 命令输出和生成文件列表粘贴到模块级 `workdocs/active/context.md#Automation` 与项目级 `context.md`（若有全局跟踪）。

4. **注册模块信息**
   - 根据脚手架生成的元数据，先更新 `modules/registry.yaml`（SSOT），补齐 `route_refs`、`graph_bindings`、`capability_refs`、`ownership`、`status` 等字段。
   - 运行 `python scripts/module_registry_sync.py --write`（或等效命令）将上述信息同步到 `doc_agent/orchestration/module-registry.yaml`，并自动刷新 `doc_agent/orchestration/registry.yaml`、`doc_agent/orchestration/doc-node-map.yaml`、`doc_agent/orchestration/agent-graph.yaml` 中的关联节点；如仓库尚未提供脚本，则需手动更新同步文件并在 workdoc 记录理由。
   - 在模块 `workdocs/active/plan.md#Registration` 记录注册内容：`id`、`type_path`、`instance_path`、`route_refs`、`graph_bindings`、`capability_refs`、`requires`/`provides`、`ownership`、`status`、`doc_set` 完整度、`last_verified_at`。
   - 执行 `python scripts/doc_node_map_sync.py --write`、`make registry_gen --check`、`make route_lint`，确保脚手架生成的文档与路由一致。命令结果写入 `workdocs/active/context.md#QA`。

5. **更新关系图与相关文档**
   - 根据注册信息更新模块关系图/能力图：`doc_agent/orchestration/agent-graph.yaml`、`doc_agent/orchestration/registry.yaml`、`doc_agent/orchestration/agent-triggers.yaml`（如需触发器）、根 `ROUTING.md` 与类型级 `ROUTING.md` 的 `context_routes`。
   - 添加必要的 `when` 条目，将实例的 README / Agent / Capability / Contract 作为 target docs。更新 `ai/maintenance_reports/route-health.md` 或其他维护日志，标记新模块接入。
   - 确认 `workdocs/active/tasks.md` 中的“初始化注册” checklist 全部完成后，将任务状态标记为 ✅，并在 `handoff-history.md` 留下初始化记录。

> 以上步骤合称“模块初始化注册”，完成后才可进入 5.2.2（功能开发阶段）。若任一步骤缺失或脚手架未运行，则视为初始化失败，禁止继续开发。

#### 5.2.2 功能开发阶段（需求 → 检索 → 调用 → 开发 → 测试 → 上下文）

1. **需求与检索范式**
   - 在 `modules/<instance>/workdocs/active/plan.md` 记录需求、风险、触发器，并在 `tasks.md` 拆分“需求澄清 / 文档检索 / 开发 / 测试 / 登记”子任务。
   - 按顺序检索：`modules/ROUTING.md` → `doc_agent/ROUTING.md` → 相关 `AGENTS.md`/`CAPABILITIES.md` → 叶子指南/契约；必要时运行 `make doc_route_refresh` 或 `python scripts/doc_node_map_sync.py --report`，并把命令输出粘到 `context.md#Automation`。
   - 在 `context.md#Session Progress` 与 `context/decisions.md` 记录已读文档、结论与缺口；未解析的需求以 ⚠️ 形式写入 `context/risks.md`。

2. **调用与开发**
   - 仅使用登记在 `modules/registry.yaml` / `doc_agent/orchestration/registry.yaml` 中的命令（Make 目标、脚本、工具）；每次调用需在 `context.md#Automation` 记“命令 → 目的 → 输出摘要”，并在 `active-files.md` 标记受影响文件。
   - 新代码/脚本引用 `modules/common/`、`modules/api/` 等共享资产时，在 `workdocs/active/plan.md#Dependencies` 登记依赖关系，并在 `modules/registry.yaml` 的 `requires`/`related_modules` 字段同步。
   - 开发结束后更新 `tasks.md` 状态，并在 `context.md#Active Files` 列出需 review 的关键 diff。

3. **测试与上下文写入**
   - 至少运行：`make dev_check`、`make module_health_check`（如有）、模块专用测试、契约/脚本校验；命令及结果必须贴到 `context.md#QA`，失败时记录阻塞原因与追踪人。
   - 将发布前必须完成的审批、验证步骤写入 `tasks.md` 和 `plan.md#Verification`，同一任务完成后在 `context/decisions.md` 记录审批结论。
   - 完成阶段开发后，在 `workdocs/active/changelog.md`、`progress.md`、`bugs.md` 中同步最新状态，准备 handoff。

4. **新知识与能力登记**
   - 新增/更新文档：放入 `modules/<instance>/doc/`，补全 front matter，并在根/类型路由 (`modules/ROUTING.md`) 中登记对应 `target_docs`；将路径追加到 `modules/registry.yaml#route_refs`，执行 `python scripts/module_registry_sync.py --write` + `python scripts/doc_node_map_sync.py --write` + `make route_lint`，确保路由可达。
   - 基础能力（capability）注册：当新增脚本/命令/API 时，先在 `doc_agent/orchestration/capabilities.yaml` 与本目录 `CAPABILITIES.md` 中登记 `capability_id`、`kind`、`entrypoint`、`inputs/outputs`、`related_docs`，再运行 `make capability_index_check`；若该能力属于模块实例，也需在 `modules/registry.yaml#capability_refs` 补充引用。
   - Agent 注册：当需要 orchestrator 调用该能力组合时，在 `modules/registry.yaml#graph_bindings` 或 `doc_agent/orchestration/registry.yaml` 增加 agent 条目，声明 `provides`（引用基础能力 ID）、`tools_allowed`、`graph_node_id`、`scope/topic/when`；随后执行 `python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`、`make registry_gen --check`、`make route_lint`，并在 `agent-graph.yaml`、`agent-triggers.yaml` 中更新节点/边。
   - 在 `workdocs/active/changelog.md` 和 `ai/maintenance_reports/route-health.md` 标记新增能力、生效时间、验证命令。

5. **接口与数据库操作规范**
   - **接口**：任何 API 变更都需同步 `modules/<instance>/doc/CONTRACT.md`、`.contracts_baseline/`、`tools/openapi.json`、前端类型（`python scripts/generate_openapi.py`、`python scripts/generate_frontend_types.py`、`make contract_compat_check`、`python scripts/type_contract_check.py`），并在 `context.md#QA` 粘贴日志；更新 `modules/registry.yaml` 的 `capability_refs`、`provides` 与触发器映射。
   - **数据库**：遵循第 6 章流程：更新 schema YAML、迁移 SQL（`*_up.sql`/`*_down.sql`），运行 `make db_lint`、`make migrate_check`、`make rollback_check`、`python scripts/db_env.py ...`，并把 dry-run/审批输出记录在 `context.md#Automation`；同步 `doc_agent/specs/DB_SPEC.yaml`、`db/engines/*/CHANGELOG.md`、`modules/registry.yaml#related_modules`，必要时更新触发器和图谱。
   - 任意接口 / DB 变更均需在 `handoff-history.md` 记录审批与生效时间，确保全局统一性。

#### 5.2.3 测试与维护阶段（规范流程）

1. **测试数据与脚本治理**
   - 目录规范：所有 mock/fixture/template 放在 `modules/<instance>/tests/fixtures/` 或 `tests/modules/<instance>/`，配套 `fixtures/README.md`（说明用途、生成方式、适用环境）。若复用全局样例，需引用 `modules/config/parameters/` 或 `modules/common/fixtures/` 而非复制文件。
   - 元数据：在 `modules/<instance>/doc/test-data.md` 或 `config/prompts/fixtures-index.yaml` 记录字段说明、来源、刷新频率；在 `modules/registry.yaml#test_assets` 写入脚本/模板路径。
   - 生成/刷新：提供 `make fixture_refresh MODULE=<instance>` 或 `python scripts/fixtures/generate_fixture.py --module <instance>`；执行后将命令输出贴到 `workdocs/active/context.md#Automation`，并在 PR / handoff 中附结果。未运行生成脚本的 fixture 视为不可用。

2. **链路与文档的变更触发机制**
   - 触发条件：接口签名、数据库结构、能力/脚本、配置、文档角色变更，或新增/删除 agent/capability 时，必须触发“链路回溯”流程。
   - 流程：
     1. 更新实际内容（代码/文档/配置）。
     2. 根据变更类型同步以下文件：
        - 文档层：`ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md`、`modules/<instance>/doc/*.md`。
        - 能力层：`doc_agent/orchestration/capabilities.yaml`、`modules/registry.yaml#capability_refs`。
        - Agent 层：`modules/registry.yaml#graph_bindings`、`doc_agent/orchestration/registry.yaml`、`agent-graph.yaml`、`agent-triggers.yaml`。
        - 记录层：`modules/<instance>/workdocs/active/changelog.md`、`handoff-history.md`、`ai/maintenance_reports/route-health.md`。
     3. 统一运行校验命令：`make route_lint`、`make capability_index_check`、`python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`、`make registry_gen --check`、`python scripts/registry_check.py`、`make trigger_check`。命令日志粘贴到 `workdocs/active/context.md#QA`。
   - 未完成上述步骤的变更不得合入；CI 需包含 `make dev_check`（内含上述 lint）确保自动阻断。

3. **模块状态追踪与半自动化更新**
   - 状态文档：`modules/<instance>/workdocs/active/progress.md` 作为权威进度表，按阶段列出关键节点、验收标准与完成度；必要时在仓库级维护 `ai/maintenance_reports/<instance>-status.md`，用于跨团队查看。
   - 半自动生成：提供 `python scripts/module_status_report.py --module <instance>`（或等效 Make 目标）读取 `tasks.md`、`modules/registry.yaml#status`、测试日志，生成“节点完成情况 + 待办 + 验证命令”草稿；输出写回 `workdocs/active/progress.md` 指定区块，由人工审核勾选完成度。
   - 更新节奏：至少每个里程碑/周执行一次状态脚本；脚本发现缺口（测试未跑、文档未同步）时需自动创建 TODO（写入 `tasks.md`）并在 `ai/maintenance_reports/route-health.md` 打标。人工确认后在 `progress.md`、`handoff-history.md` 记录“自动报告版本 + 审核人 + 时间”。
   - 发布/维护：在功能上线前后运行 `make health_check`、`make health_show_quick_wins` 等巡检命令，结果写入 `ai/maintenance_reports/<module>-health.md`；若升级或替换实例，必须更新 `modules/registry.yaml#status`、`workdocs/active/changelog.md`、图谱与触发器，并准备回滚/`@deprecated` 计划。

### 5.3 模块开发事务规范

1. **目录基线**
   - `modules/<instance>/frontend/`：`src/components/`、`src/pages/`、`src/stores/`、`src/assets/`、`tests/`、`types/`，附 `frontend/README.md`（构建/测试命令、入口）并在 `modules/<instance>/ROUTING.md` 注册 `topic: Frontend`。
   - `modules/<instance>/backend/`：`app/controllers`、`app/services`、`app/repositories`、`app/schemas`、`tests/`、`fixtures/`，注册 `topic: Backend`。
   - `modules/<instance>/core/`（或 `services/`）：承载核心算法/跨端逻辑，提供 `CAPABILITIES.md` 说明暴露能力、`capability_id`、`graph_node_id`。
   - `modules/<instance>/config/`：模块级配置、prompt、超参数、feature flag，遵循 5.1 的配置规范，并在 `modules/registry.yaml#config_refs` 记录。

2. **前端开发规范**
   - 包管理与命令：仅使用 `pnpm`（`pnpm install/dev/test/lint/build`）；命令输出贴到 `workdocs/active/context.md#Automation`。
   - 组件命名使用 kebab-case，路由定义集中在 `src/router/`，全局状态放在 `src/stores/`；相关说明写入 README。
   - 新增接口调用必须使用 `tools/openapi.json` 生成的类型（执行 `pnpm generate:types`），并同步 `modules/<instance>/doc/CONTRACT.md`；未同步合同不得合入。
   - UI/交互规范、可复用 Hook、设计系统引用写入 `modules/<instance>/doc/UX.md`，变更后运行 `pnpm test` + `pnpm lint`。
   - Figma 设计对齐：开发 UI 组件时可通过 Figma MCP 工具直接引用设计稿，按需调用即可。关键设计稿链接记录在 `modules/<instance>/doc/UX.md` 中，无需维护设计 Token 或版本同步流程。

3. **后端开发规范**
   - 控制器/服务/仓储分层明确：Controller 只接收/响应、Service 处理业务逻辑、Repository 封装数据访问、Schema 定义输入输出；每层 README 说明职责。
   - 命令：`make dev_check`、`pytest modules/<instance>/backend/tests`、`ruff`、`mypy`（如适用）；结果贴到 `context.md#QA`。
   - 数据库/缓存改动遵循第 6 章流程，更新 `modules/<instance>/doc/storage.md`、`modules/registry.yaml#requires`、`workdocs/active/context.md#Automation`。
   - API 变更需同步 `.contracts_baseline/`、`tools/openapi.json`、前端类型，并运行 `make contract_compat_check`、`python scripts/type_contract_check.py`。

4. **核心能力与跨端逻辑**
   - 公共算法、推理流程、外部模型封装集中在 `core/`，并在 `CAPABILITIES.md` 登记 `capability_id`、`entrypoint`、`inputs/outputs`、约束。
   - 若依赖外部模型或函数超参，将参数写入 `modules/<instance>/config/parameters/`，在 `config-index.yaml`、`modules/registry.yaml#capability_refs` 登记引用，并在 `workdocs/active/plan.md#Dependencies` 说明。

5. **自动化与联动要求**
   - 任何新目录/文档必须在 `modules/<instance>/ROUTING.md` 注册 topic，并运行 `python scripts/doc_node_map_sync.py --write`、`make route_lint` 确认可达。
   - 每个开发阶段结束前运行 `python scripts/module_status_report.py --module <instance>`（或 `make module_status_report`），将输出粘到 `workdocs/active/progress.md` 并人工确认。
   - 所有开发动作需按 5.2.2、5.2.3、5.4 规定执行命令（能力登记、链路校验、配置同步），确保 router/registry/graph、测试、工作文档一致。

### 5.4 系统化治理规范

1. **路由 × Registry × Graph 同步守则**
   - 执行顺序：① 更新 `modules/<instance>/ROUTING.md` 与根 `ROUTING.md`；② 编辑 `modules/registry.yaml`（`route_refs`、`graph_bindings`、`requires`/`provides`、`status` 等）并运行 `python scripts/module_registry_sync.py --write`；③ 更新 `doc_agent/orchestration/registry.yaml`、`agent-graph.yaml`、`agent-triggers.yaml`；④ 运行 `python scripts/doc_node_map_sync.py --write`、`make route_lint`、`make capability_index_check`、`make registry_gen --check`、`python scripts/registry_check.py`、`make trigger_check`。
   - 上述命令输出必须贴到 `modules/<instance>/workdocs/active/context.md#QA`，并在 `ai/maintenance_reports/route-health.md` 标记“结构同步”记录。缺任一步骤的改动一律视为未闭环。

2. **依赖与跨模块协作**
   - `modules/registry.yaml` 是 `requires` / `provides` / `related_modules` 的唯一真相源；新增依赖时同步更新 `modules/<instance>/workdocs/active/plan.md#Dependencies`，附上验证与回滚步骤。
   - 跨模块调用必须通过已登记的 contract/trigger/agent，禁止直接引用其他模块文件或共享未登记脚本。共享脚本需先注册为基础能力，再由相应 agent `provides` 引用。
   - 牵涉多个模块时，在 `workdocs/active/tasks.md` 创建“跨模块通知/验证”子任务，完成前不得关闭主任务，并在 `handoff-history.md` 记录通知结果。

3. **数据流向与模块关系图（自动化维护）**
   - 数据流向图存于 `doc_agent/orchestration/dataflow/graph.yaml`，使用 `python scripts/graph_sync.py --graph dataflow` 维护并生成 `docs/graphs/dataflow.mmd`；任何能力或 agent 变更需更新该文件并在 `ai/maintenance_reports/route-health.md` 粘贴最新版图像链接。
   - 模块关系图存于 `modules/graphs/module-relations.yaml`，由 `python scripts/module_graph_sync.py --graph relations` 输出 `modules/graphs/module-relations.mmd`；展示模块实例的 `requires`/`provides`/状态。根 README 或 `modules/ROUTING.md` 必须链接最新图像。
   - CI / 巡检必须运行上述脚本并比较 diff，未同步图文件的变更禁止合入。

4. **审计与可追溯**
   - 初始化、接口/数据库变更、能力/agent 调整、配置更新、上线/下线都要在 `workdocs/active/context.md`、`changelog.md`、`handoff-history.md`、`ai/maintenance_reports/route-health.md` 留痕。
   - 高风险操作需运行 `make agent_lint`、`make trigger_check`、`make health_analyze_issues`（或仓库规定脚本），并附日志。
   - 每季度执行一次治理巡检：重复第 1 项的命令集合，输出 `ai/maintenance_reports/<module>-governance.md`，列出差异、责任人和整改计划。

5. **交付阶段 Checklist（与 5.2 对应）**
   - 初始化：
     - `make ai_begin` 产出的文档 front matter 完整且已挂入 `ROUTING.md`；
     - `modules/registry.yaml`、`capabilities.yaml`、`agent-graph.yaml`、`agent-triggers.yaml` 经 `module_registry_sync` / `doc_node_map_sync` / `route_lint` / `capability_index_check` / `registry_gen --check` / `registry_check.py` / `trigger_check` 全部 PASS；
     - `workdocs/active/context.md#Automation/#QA` 记录命令与结果。
   - 功能开发：
     - 新增文档/能力/agent 按 5.2.2 登记完毕，`modules/registry.yaml` 的 `route_refs`、`capability_refs`、`graph_bindings`、`requires`/`provides` 已更新；
     - `tasks.md` 中“开发/检索/测试”项均已勾选。
   - 测试与维护：
     - `make dev_check`、`make module_health_check`、契约/脚本校验、fixture 生成脚本完成并粘贴日志；
     - `modules/registry.yaml#status`、`workdocs/active/progress.md`、`ai/maintenance_reports/<module>-health.md` 更新为最新。
   - 上线/下线：
     - `.contracts_baseline/`、OpenAPI、DB spec、配置、图谱/触发器同步；
     - `handoff-history.md` 记录审批、时间窗、回滚方案。

6. **运维阶段 Checklist**
   - 迭代后刷新 `workdocs/active/changelog.md`、`progress.md`、`bugs.md`；运行 `make health_check`、`make health_show_quick_wins` 并在 `route-health.md` 记录差异。
   - 配置/能力/触发器改动后立刻执行 5.3 的同步命令，并在 `workdocs/active/context.md` 粘贴日志。
   - Handoff：AI Owner 与人类 Owner 在 `context.md#Session Progress` 和 `handoff-history.md` 写明交接内容、下一步计划、责任人。

7. **治理节奏**
   - 每个里程碑前后执行 lifecycle review：按本章 checklist 勾选并输出 `ai/maintenance_reports/<module>-governance.md`；若发现缺口，在 `tasks.md` 创建 TODO 并标注责任人/截止时间。
   - 可在 `modules/<instance>/CHECKLIST.md` 保存模块专属清单（引用本章节）；新增条目同时回写到该章节或配置治理文档，保持规范唯一。

## 6. 数据库联动
> **领域提示**：数据库场景侧重结构化数据治理——Schema 变更、迁移与回滚、凭据隔离、Redis TTL 生命周期，以及变更后的文档/CHANGELOG 闭环。下文沿用统一框架，但示例与工具均围绕上述要点展开。

### 6.1 建设目标
- 建立从空项目到可运行模板的数据库协作流程，覆盖 PostgreSQL（主库）与 Redis（缓存）常见场景。
- 让智能体与人类开发者都能通过统一路由快速定位 Schema、迁移脚本、审计记录，降低误操作风险。
- 将数据库变更纳入自动化链路（lint、迁移校验、回滚演练、文档同步），保证交付质量。

### 6.2 目录与初始资产
- `db/engines/postgres/`
	- `schemas/`：以 YAML（如 `tables/<name>.yaml`）描述表结构、索引、约束。
	- `migrations/`：成对的 `*_up.sql` / `*_down.sql`，按时间戳命名。
	- `docs/DB_SPEC.yaml`：数据库总体约束与术语表。
- `db/engines/redis/`
	- `schemas/`：声明 key 结构、TTL、序列化格式。
	- `docs/`：维护 Redis 端的使用规范、容量策略。
- `doc_agent/specs/DB_SPEC.yaml`：AI 版总览，含 scope、路由、常用命令；行数控制在 150 行内。
- `doc_human/guides/DB_CHANGE_GUIDE.md`：人类版操作指南，记录审批与回滚流程。
	- 初次生成模板时：
		- 复制上述目录结构，初始化空 schema + 示例迁移。
		- 在 capability registry（`doc_agent/orchestration/capabilities.yaml`）登记底层数据库/缓存能力条目；随后在 orchestrator registry（`doc_agent/orchestration/registry.yaml`）注册 `Data CRUD agent`、`Redis Lifecycle agent` 等暴露给 orchestrator 的封装智能体，并在 `agent-triggers.yaml` 配置 "Execution – Database / Migration Review / Before running migration scripts" 等场景的 `target_agent`。
		- 本地 `AGENTS.md` 补充策略与 `related_docs`，`CAPABILITIES.md` 描述能力语义与组合关系，`ROUTING.md` 仅保留路由节点。

### 6.3 Schema 建模规范
- 统一使用 YAML Schema 定义表/索引字段，字段说明包含：`name`、`type`、`nullable`、`default`、`description`、`tags`。
- 所有 schema 文件必须带 front matter：`audience: ai`、`purpose`、`ownership`、`updated_at`。
- 新增或调整 Schema 时：
	1. 更新 YAML 描述并运行 `make db_lint`（调用 `scripts/db_lint.py`）检查字段类型、命名、索引规则。
	2. 如碰到引用 `modules/<name>/` 数据，请同步在模块文档中登记依赖。
- Redis 结构需指定 key pattern、数据结构（list/hash/stream）、最大长度和清理策略，并在 `scripts/context_usage_tracker.py` 的配置中登记。

### 6.4 智能体驱动的 CRUD 与生命周期闭环
- 模板必须自带结构化数据的 `Data CRUD agent` 与缓存数据的 `Redis Lifecycle agent`。生成模板时，先在 capability registry 记录相关底层能力，再在 orchestrator registry 注册对应智能体；同时补充本地 `AGENTS.md`（策略+工具说明）与 `CAPABILITIES.md`（能力索引），`ROUTING.md` 仅保留链接。
- 标准闭环：
	1. Orchestrator 根据 `context_routes`、orchestrator registry 映射与触发器配置将 CRUD 任务分发至相应智能体。
	2. 智能体根据操作类型调用封装脚本，例如：
		- `python scripts/db_env.py --run-sql plan.sql`（结构化读写）
		- `python scripts/context_usage_tracker.py --redis-key <pattern>`（Redis 状态巡检）
	3. 所有执行先做 dry-run，生成 SQL 预览或 Redis 操作计划；最终执行由人工确认并在指定环境运行。智能体需输出风险评估与回滚建议，供人工审阅。
	4. 人工确认执行后，智能体自动触发文档更新流程：调用 `scripts/docgen.py --section=db` 或自定义脚本更新
		- `doc_agent/specs/DB_SPEC.yaml` 中的表/键摘要
		- `db/engines/postgres/CHANGELOG.md` / `db/engines/redis/CHANGELOG.md`
		- 模块目录下的 `storage.md`、`CONTRACT.md` 等
	5. Workflow & Telemetry agent 收集 CRUD 调用日志，更新 `ai/maintenance_reports/route-health.md`。
- Redis 数据生命周期：
	- `Redis Lifecycle agent` 负责 key 创建、续期、淘汰。每次调整 TTL 或结构时，自动更新 `db/engines/redis/schemas/` 与 `doc_agent/specs/DB_SPEC.yaml`。
	- 建议使用 `scripts/db_env.py --redis-plan` 生成操作计划，输出 JSON（命令、风险、回滚方式）。
- 人工兜底：
	- 智能体在 dry-run 后输出执行清单，由人工选择目标环境并完成最终执行；操作过程需在 `ai/workdocs/active/<task>/context.md` 留痕。
	- 如文档同步脚本失败或人工选择不执行自动同步步骤，需在 `handoff-history.md` 登记原因并补写手动更新指引。
- 除上述生命周期智能体外，其他数据库操作能力（DDL 脚本生成、索引优化、数据迁移、清理任务等）需先在 capability registry 注册基础能力，再在 orchestrator registry 登记封装智能体，并在 `doc_agent/orchestration/agent-triggers.yaml` 配置触发条件；漏洞修补、批处理、报表同步等后续开发能力同样遵循"能力登记 → 智能体封装 → 图更新 → 文档同步"的流程。

### 6.5 迁移与回滚流程
- 迁移脚本约束：
	- 每个 `*_up.sql` 必须有同名的 `_down.sql`。
	- 禁止直接 `DROP TABLE`，需通过 `deprecated` 标记 + 后续清理。
	- 变更前先运行 `make migrate_check`（调用 `scripts/migrate_check.py`）验证依赖、锁等待、兼容性。
- 标准步骤（写入 Data & Schema agent 的 playbook）：
	1. 编写/更新 schema YAML。
	2. 创建迁移 SQL（使用 `scripts/db_env.py --template` 可生成脚手架）。
	3. 运行 `make db_lint` → `make migrate_check` → `make rollback_check`（确保 `_down.sql` 可执行）。
	4. 更新 `doc_agent/specs/DB_SPEC.yaml` 的变更摘要，记录主键/索引变化。
	5. 在 `ai/workdocs/active/<task>/context.md` 记日志，完成后归档。
- 如需本地演练，可使用 `docker-compose up db` 启动测试数据库，或调用 `scripts/db_env.py --load-sample` 注入示例数据。

### 6.6 文档与路由维护
- 路由要求：
	- 根 `ROUTING.md` 维护数据库相关 `context_routes`，`target_docs` 指向 AI 版 spec、政策文档及 Data & Schema agent 的策略说明。
	- `db/engines/postgres/ROUTING.md` 作为子路由，细化 "Schema Reference"、"Migration SOP"、"Rollback Procedures" 等 `topic`，连接实际文件或下级 agent。
		- 模块目录若定义私有数据表，需在 `modules/<name>/ROUTING.md` 注册 "Module – <name> / Storage / …" 条目，`target_docs` 指向模块内的存储说明，并引用同级 `AGENTS.md`（策略）与 `CAPABILITIES.md`（能力入口）。
- 文档内容：
	- 安全政策：权限矩阵、审批流程、连接方式，存于 `doc_agent/policies/database-security.md`（若无需拆分，可在 DB_SPEC.yaml 中声明）。
	- 操作手册：常用命令、故障排查、监控指标；需在 orchestrator registry 中登记对应 `tools_allowed`（如 `make db_lint`、`make migrate_check`），并在手册中引用这些条目。
	- 变更登记：维护 `db/engines/postgres/CHANGELOG.md`（按日期、变更类型、责任人）及 `ai/maintenance_reports/route-health.md`（自动统计命中率）。

### 6.7 智能体与工具联动
- Data & Schema agent（能力域智能体）：
	- 注册工具：在 orchestrator registry 中为该 agent 标注 `tools_allowed`（如 `make db_lint`、`make migrate_check`、`make rollback_check`、`python scripts/db_lint.py` 等），便于编排层调度。
		- `ROUTING.md` 中相关 `context_routes.when` 的 `target_docs` 指向 DB spec / 政策文档 / `AGENTS.md` / `CAPABILITIES.md`。
	- 在 `agent-graph.yaml` 中声明与 Guardrail agent、Workflow & Telemetry agent 的边，确保变更前后都有审计。
- Guardrail agent：
	- 针对 `event: db_migration_pending` 的触发器，要求查阅政策、生成审批工单，必要时阻断高风险操作。
- Workflow & Telemetry agent：
	- 读取 `scripts/context_usage_tracker.py` 输出的迁移执行日志，更新 `ai/maintenance_reports/route-health.md`。

### 6.8 安全审批与自动化
- 编排钩子：
	- `agent-triggers.yaml` 中为 "检测到 migrations/ 新文件" 定义触发条目，指定 `target_agent: data_schema`、`preload_docs`、`required_validations`。
	- 引入 `selection_policy: highest_priority`，保证审批流程先于执行脚本。
- 审批记录：
	- 在 `ai/workdocs/active/<task>/tasks.md` 添加 "审批" 子任务，标记审批人、时间、结果。
	- 完成后更新 `handoff-history.md`，写明变更摘要、审核步骤、回滚实验结果。
- 安全红线：
	- 禁止智能体直接执行生产数据库命令；只能生成脚本或计划，由人工确认。
	- 所有 SQL 模板需通过 `scripts/secret_scan.py` 检测，避免硬编码凭据。

### 6.9 模板流程示例
1. 需求：新增 `orders` 表（PostgreSQL）。
2. 步骤：
	- 在 `db/engines/postgres/schemas/tables/orders.yaml` 定义字段与索引。
	- 使用 `scripts/db_env.py --template orders` 生成迁移脚本骨架。
	- 填写 `*_up.sql`、`*_down.sql`，运行 `make db_lint`、`make migrate_check`、`make rollback_check`。
	- 更新 `doc_agent/specs/DB_SPEC.yaml` 的 "Tables → orders" 段落。
		- 在根 `ROUTING.md` 添加 `context_routes` 条目：scope "Execution – Database"、topic "New Table Onboarding"、when 描述对应场景，并在 `target_docs` 中指向相关策略 (`AGENTS.md`)、能力索引 (`CAPABILITIES.md`) 与 DB spec。
	- 通过 Data & Schema agent 的 playbook 输出审批表，让 Guardrail agent 审核。
	- 审批通过后，将迁移脚本纳入 CI（`make dev_check`），并在 `ai/workdocs/active/orders-table/context.md` 记录结果。

### 6.10 验收指标
- 文档：`db/engines/postgres/docs/DB_SPEC.yaml` 与 `doc_agent/specs/DB_SPEC.yaml` 内容同步、路由可用；安全政策、操作手册 front matter 完整。
- 自动化：`make db_lint`、`make migrate_check`、`make rollback_check` 在 CI 中无报错；触发器能检测到迁移新增。
- 智能体：Data & Schema agent 在 registry 中登记完整的 `tools_allowed`、`graph_node_id` 并与 `ROUTING.md` 的 `context_routes` 映射一致；Guardrail agent 有审批记录。
- 数据质量：迁移执行日志收敛在 `ai/maintenance_reports/route-health.md`；回滚演练至少每季度一次并留有记录。

## 7. API 联动管理

> **领域提示**：API 场景聚焦合同兼容、OpenAPI/客户端类型同步、网关策略与中间件、外部暴露安全审批。框架与数据库章节一致，但下文所有示例与工具都围绕接口生命周期展开。

### 7.1 建设目标
- 构建从接口合同、模块实现到网关编排的完整闭环，支持空仓快速生成可运行的 API 模板。
- 让智能体与人工开发者可通过统一路由定位合同、脚手架、测试与客户端类型，降低接口漂移风险。
- 将 API 变更纳入自动化链路：合同校验、OpenAPI 导出、类型同步、测试与文档更新一体化执行。

### 7.2 目录与初始资产
- `modules/api_gateway/`：默认网关模块，包含 `AGENTS.md`、`doc/ROUTING.md`、`doc/CONTRACT.md`、`doc/MIDDLEWARE.md`，定义路由策略与中间件。
- `modules/<name>/doc/CONTRACT.md`：模块级接口合同；与 `.contracts_baseline/` 快照配套，用于回归比较与审批。
- `tools/codegen/contract.json`：仓库默认合同 Schema；新增端点时可复用字段定义。
- `tools/openapi.json`：整合后的 OpenAPI 3.0 输出，供客户端或文档站点引用。
- `scripts/generate_openapi.py`、`scripts/contract_compat_check.py`、`scripts/type_contract_check.py`、`scripts/generate_frontend_types.py`：合同→OpenAPI→类型→校验的核心脚本。
- `ai/task-templates/api-endpoint.yaml`：新增接口时的标准工作流模板，指引智能体填写 plan/context/tasks。
- 初始化模板时：
  1. 复制 `modules/api_gateway/`，在 capability registry（`doc_agent/orchestration/capabilities.yaml`）登记 API 网关与合同校验等基础能力，再在 orchestrator registry（`doc_agent/orchestration/registry.yaml`）注册 `modules.api_gateway.v1` 智能体及 `tools_allowed`。
  2. 为各模块准备 `doc/CONTRACT.md`、`doc/ROUTING.md`，并在根 README 路由表登记接口类文档。
  3. 将 `tools/openapi.json` 与 `.contracts_baseline/` 纳入版本控制，提供后续校验基线。

### 7.3 合约与接口规范
- 合同文件统一使用 Markdown + front matter：`audience: ai`、`doc_role: contract`、`route_role: leaf`、`ownership`、`updated_at`、`graph_node_id`。
- `modules/<name>/doc/CONTRACT.md` 需列出端点、方法、路径、鉴权、请求/响应 JSON Schema，并可引用 `tools/codegen/contract.json` 的共享字段。
- 变更合同后更新 `.contracts_baseline/` 快照，运行 `make contract_compat_check` 确认兼容性；若存在破坏性调整，在 workdoc 与 `handoff-history.md` 记录审批信息。
- OpenAPI 导出流程：执行 `python scripts/generate_openapi.py` 生成最新 `tools/openapi.json`，随后运行 `python scripts/generate_frontend_types.py` 将类型同步至 `frontend/` 或 `frontends/` 下的 `types/api.d.ts`。
- 若合同依赖共享 Schema，需在 `schemas/` 目录补充 JSON Schema，并在合同中以 `$ref` 引用，同时在 `doc_agent/orchestration/doc-node-map.yaml` 登记映射。

### 7.4 智能体驱动的 API 生命周期
- `modules/api_gateway/AGENTS.md` 中的模块智能体负责路由、鉴权、速率限制：先在 capability registry 登记所依赖的基础能力（合同校验、OpenAPI 导出、类型同步），再在 orchestrator registry 中注册 `modules.api_gateway.v1` 入口并关联 `tools_allowed`、`graph_node_id`。
- Orchestrator 通过根 `ROUTING.md` 的 API topic 引导加载 `modules/api_gateway/doc/ROUTING.md`、目标模块 `CONTRACT.md` 与 `CAPABILITIES.md`。
- 标准闭环：
  1. 在 `ai/workdocs/active/<task>/plan.md` 登记 API 需求，引用 `ai/task-templates/api-endpoint.yaml` 拆分任务。
  2. 编写或更新模块合同与实现后，运行 `make contract_compat_check` 与 `python scripts/type_contract_check.py` 校验实现与合同一致。
  3. 执行 `python scripts/generate_openapi.py` 刷新 `tools/openapi.json`，必要时运行 `python scripts/generate_frontend_types.py` 更新客户端类型。
  4. 运行模块与网关测试（如 `pytest tests/example/test_smoke.py`）并记录结果，在 `ai/workdocs/.../context.md#API` 粘贴输出。
  5. 更新根 README 路由表、模块 `doc/ROUTING.md` 与 `doc_agent/orchestration/doc-node-map.yaml` 的能力映射后提交变更。
- Workflow & Telemetry agent 可结合 `scripts/context_usage_tracker.py` 记录 API 命中率，并在 `ai/maintenance_reports/route-health.md` 标注。

### 7.5 自动化校验与生成流程
- 合同校验：`make contract_compat_check`、`python scripts/contract_compat_check.py`、`python scripts/type_contract_check.py`。
- OpenAPI 与类型导出：`python scripts/generate_openapi.py`、`python scripts/generate_frontend_types.py`。
- 文档同步：`python scripts/module_doc_gen.py --module <name> --section api` 更新模块说明；必要时执行 `make docgen` 刷新 front matter。
- CI 集成：确保 `make dev_check` 包含合同与类型校验；新增脚本需在 `Makefile` 注册相应目标。

### 7.6 文档与路由维护
- 根 `ROUTING.md` 新增 scope "Execution – API"，topic "Endpoint Lifecycle"，when "设计或变更接口合同" 指向 `modules/api_gateway/doc/ROUTING.md`、`modules/api_gateway/AGENTS.md`、`tools/openapi.json`（通过说明文档包装）。
- 模块级 `ROUTING.md` 应细分 "API Contracts"、"Integration Tests"、"Client Types"，分别指向合同、测试目录与类型生成脚本。
- 在 README 路由表登记 API 文档，并在 `doc_agent/orchestration/doc-node-map.yaml` 维护合同文档、能力节点与触发器的映射关系。

### 7.7 安全与审批
- `doc_agent/orchestration/agent-triggers.yaml` 的 `contract-change`（critical/block）触发器监控 `modules/*/doc/CONTRACT.md` 与 `tools/**/*.json`，命中时预加载 Guardrail 指南。
- Guardrail agent 审批需确认：是否刷新 `.contracts_baseline/`、是否执行合同与类型校验、是否在 workdoc 标记审批、是否同步路由与触发器。
- 外部暴露接口必须遵守 `doc_agent/policies/security.md`、`doc_agent/policies/quality.md`，声明鉴权、速率限制与审计字段，禁止输出敏感数据。
- 发布前运行 `make trigger_check` 与 `python scripts/registry_check.py` 确认触发器和 registry 同步，并在 `handoff-history.md` 记录审批结论与回滚方案。

### 7.8 API 变更执行规范
| 阶段 | 必做动作 | 产出/校验 |
| --- | --- | --- |
| 需求与检索 | 在 `modules/<instance>/workdocs/active/plan.md` 登记变更目的、影响范围、Guardrail/触发器；检索 `ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md` 确认是否已有能力 | `plan.md#Scope`、`tasks.md` 子任务 |
| 合同设计 | 在 `modules/<instance>/doc/CONTRACT.md` 定义/更新端点、Schema、示例；front matter 写明 audience/purpose/doc_role | 更新后的 `CONTRACT.md`，`ROUTING.md` `target_docs` 同步 |
| 校验命令 | 必须运行 `python scripts/type_contract_check.py`、`make contract_compat_check`、`python scripts/generate_openapi.py`、`python scripts/generate_frontend_types.py`；若前端依赖，执行 `pnpm test` 或等效命令 | 所有命令输出粘贴到 `workdocs/active/context.md#QA` |
| 路由与注册 | 更新根/模块 `ROUTING.md`、`modules/registry.yaml#capability_refs`、`doc_agent/orchestration/registry.yaml`、`agent-graph.yaml`（必要时新增节点）；执行 `python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`、`make route_lint`、`make capability_index_check` | lint 通过，`ai/maintenance_reports/route-health.md` 打标 |
| 测试与验收 | 运行相关 `pytest`、`make dev_check`、契约/类型测试，若触发 Guardrail 按 10.4 审批；`tasks.md` 勾选“API Tests/Docs Sync” | 测试通过截图/日志附在 workdoc；`handoff-history.md` 记录审批结果 |
| 上线与回溯 | 在 `ai/workdocs/active/<task>/context.md#Changelog`、`handoff-history.md` 写入上线时间、命令、回滚方案；`ai/maintenance_reports/route-health.md` 记录接口健康检查（可引用监控链接） | 完整的交接记录与回滚计划 |

> **验收标准**：API 变更仅在上述阶段全部完成且日志留存后可合入；任何缺失步骤视为流程不合规。

### 7.9 验收指标
- 文档：所有 `CONTRACT.md` front matter 完整，`tools/openapi.json` 保持最新，README 路由表登记完备。
- 自动化：`make contract_compat_check`、`python scripts/generate_openapi.py`、`python scripts/type_contract_check.py` 在 CI 无报错；类型生成脚本可重复执行。
- 智能体：`modules.api_gateway.v1` 在 orchestrator registry 中登记合同校验、OpenAPI 导出、类型同步等 `tools_allowed`，并与 capability registry 及 `doc_agent/orchestration/doc-node-map.yaml` 的映射一致。
- 审计：API 变更在 workdoc 与 `handoff-history.md` 留痕，`ai/maintenance_reports/route-health.md` 有接口调用统计与异常记录。

## 8. 自动化链路
> 按情境给出"检索→计划→编码→写入→上下文"的自动化方案，覆盖第 2 章（文档路由）与第 3 章（能力/图）的常见操作。

### 8.1 自动化链路管理与维护
- **目标**：维持路由、能力图、触发器与 workdoc 记录一致，做到"有新增必入档、修改必审计、查看必留痕"。
- **职责分层**：编排层处理审批与触发条件，目录所有者执行文档/脚本改动，Guardrail agent 负责安全边界。所有动作需在 `ai/workdocs/active/<task>/plan.md` 中登记计划，在 `handoff-history.md` 记录完成情况。
- **操作准则**：
  - **新增链路**：① 在 plan.md 写明新链路的 scope/topic/触发器；② 使用 `make doc_route_refresh`、`make registry_show` 确认没有现成能力；③ 依次更新 `ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md`，并在 capability registry、orchestrator registry 补登条目；④ 运行 `make route_lint`、`make capability_index_check`、`python scripts/doc_node_map_sync.py --write`；⑤ 将命令结果贴入 `ai/workdocs/.../context.md#Automation`，同步在 `ai/maintenance_reports/route-health.md` 标记"新增"。
  - **修改链路**：① 先执行 `python scripts/doc_node_map_sync.py --report`、`make registry_gen --check` 找出上下游影响；② 在 `tasks.md` 拆分文档、graph、trigger 更新项；③ 提交前运行 `make dev_check` 或最小化验证组合，并在 `handoff-history.md` 写明"旧→新"差异。
  - **查看/巡检链路**：① 周期性执行 `make trigger_check` + `python scripts/context_usage_tracker.py report --limit 10` 评估触发覆盖；② 若仅做审阅，需在 workdoc context 的 "Audit" 小节注明"view only"及引用的文档；③ 如发现缺口，立刻创建 follow-up task，并把审阅结论同步到 `ai/maintenance_reports/route-health.md`。

#### 示例：数据库新增表的自动化链路
1. **需求登记**：在 `ai/workdocs/active/<task>/plan.md` 写明"新增 PostgreSQL 表"的目标与风险清单，并在 `context.md`、`tasks.md` 拆出 schema、迁移、文档、审批等子任务。
2. **路由与策略挂接**：按 S1/S2 流程为根 `ROUTING.md` 补充 `Execution – Database / New Table Onboarding / when: 在创建新表前` 条目，`target_docs` 连接 `db/engines/postgres/ROUTING.md`、`db/engines/postgres/AGENTS.md`、`db/engines/postgres/CAPABILITIES.md` 与 `doc_agent/specs/DB_SPEC.yaml`，确保编排能定位 Data & Schema agent。
3. **能力校准**：如需扩展能力，更新 capability registry（`doc_agent/orchestration/capabilities.yaml`）、orchestrator registry（`doc_agent/orchestration/registry.yaml`）、`agent-graph.yaml`、本地 `AGENTS.md`/`CAPABILITIES.md` 并运行 `make capability_index_check`、`python scripts/doc_node_map_sync.py --write`；若已有节点，仅需核对 `graph_node_id`、工具清单与路由引用是否一致。
4. **自动化执行**：模型加载策略后，调用 `scripts/db_env.py --template <table>` 生成迁移骨架，填写 `*_up.sql` 与 `_down.sql`，依次运行 `make db_lint`、`make migrate_check`、`make rollback_check`；Dry-run、风险提示和审批清单写入 `ai/workdocs/.../context.md#Automation` 并提交 Guardrail agent 审核。
5. **文档同步与记录**：审批通过后在目标环境执行迁移，同时更新 `doc_agent/specs/DB_SPEC.yaml`、`db/engines/postgres/CHANGELOG.md`、根 README 路由表等；将命令结果贴入 workdoc，并在 `handoff-history.md` 记录"旧→新"差异及生效时间。

### 8.2 触发器机制
- **作用定位**：触发器是自动化链路的前置哨兵，基于文件改动或自然语言描述主动拦截高风险操作，加载必读文档并指向正确的能力入口，确保"触发→路由→执行"闭环不偏移。
- **生效流程**：
  1. **事件捕获**：`agent-triggers.yaml` 定义的 `file_triggers.path_patterns`、`prompt_triggers.keywords` 命中后生成事件（如 `database-operations`）。
  2. **守卫判定**：根据 `priority` 与 `enforcement`（block/warn/suggest）决定是否允许继续执行，以及是否需要人工审批。
  3. **文档预加载**：`load_documents` 列表会在模型后续响应前自动入场，例如数据库操作会强制加载 `/db/engines/README.md`、`/doc_human/guides/DB_CHANGE_GUIDE.md`、`/db/engines/postgres/docs/SCHEMA_GUIDE.md`。
  4. **能力衔接**：触发器与 `context_routes`、`registry.yaml`、`agent-graph.yaml` 配合，将任务导向匹配的 agent 节点或工具端口（如 Data & Schema agent），推进后续自动化步骤。
- **配置步骤**：
  - 编辑 `doc_agent/orchestration/agent-triggers.yaml`，保持 front matter 与版本号一致。
  - 在 `triggers` 下新增/调整条目：命名遵循业务语义（如 `database-operations`），设置 `priority`、`enforcement`、`description`。
  - 通过 `file_triggers.path_patterns` 监控目录/文件类型，`prompt_triggers.keywords` 捕捉命令意图；必要时按子目录细化。
  - `load_documents` 应指向轻量、结构化的守则或操作指南，避免一次性加载大型文档；若需额外文档，可改为在路由或能力索引中追加。
  - 完成配置后运行 `make trigger_check`（或仓库定义的等价命令）校验语法与路径有效性，同时执行 `make registry_gen --check` / `make route_lint` 确保上下游映射未断链。
- **与自动化链路的协同**：
  - 新增链路时，先确认是否复用现有触发器；若需要扩展，先更新 `agent-triggers.yaml`，再同步 `registry.yaml`、对应目录的 `ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md`，并在 `ai/workdocs/.../context.md` 记录命中结果。
  - 触发器优先级应与 Guardrail 策略一致（例如数据库操作保持 `critical/block`），避免出现链路已执行但审批节点缺失的情况。
  - 调整或退役触发器时，同步更新 `doc_agent/orchestration/doc-node-map.yaml`、触发器可视化（若有）、`ai/maintenance_reports/route-health.md` 中的变更记录。

### 8.3 模板初始化情形清单（后续补充）
> **功能基线**：本仓模板必须同时具备用于"检索 → 规划 → 开发 → 测试 → 交付"的下列能力，初始化期间必须逐项勾选：
> 1. **文档与路由体系**：`ROUTING.md` / `AGENTS.md` / `CAPABILITIES.md` / front matter / doc-node-map 的生成、lint 与同步脚本。
> 2. **模块脚手架**：`make ai_begin` 或等效命令，自动创建 `modules/<instance>/` 目录、workdocs、配置、脚本与注册登记（`modules/registry.yaml`）。
> 3. **配置与提示词管理**：`modules/config/` + `modules/<instance>/config/` 的参数、prompt、feature flag，含 routing/policy/validation 脚本。
> 4. **能力/Agent/Graph 资产**：`doc_agent/orchestration/registry.yaml`、`agent-graph.yaml`、`capabilities.yaml`、`trigger-map.yaml`、`agent-triggers.yaml` 及其同步脚本。
> 5. **自动化链路与 Guardrail**：`make route_lint`、`make capability_index_check`、`make trigger_check`、`python scripts/doc_node_map_sync.py`、`guardrail_runner.py` 等命令链。
> 6. **上下文恢复体系**：`ai/workdocs/`（plan/context/tasks/lessons）、`handoff-history.md`、`ai/maintenance_reports/*`、post-tool tracker。
> 7. **数据库 / API / 脚本基线**：`db/engines/*`、`tools/openapi.json`、`.contracts_baseline/`、`scripts/operations-guide.md` 及对应 lint/migrate/contract/type 命令。
> 8. **测试与 CI**：`make dev_check`（含 lint/test/route/capability/registry/trigger）、`make module_health_check`、`python scripts/context_usage_tracker.py`、`make guardrail_check` 等。
> 9. **可观测与巡检**：`ai/maintenance_reports/route-health.md`、`module_status_report.py`、`context_usage_tracker.py`、`python scripts/module_graph_sync.py`。

| 情形 | 适用场景 | 影响面 |
| --- | --- | --- |
| 新增目录/创建 `ROUTING.md` | 新模块或共享目录首次落地 | doc 路由、doc-node-map |
| 新增/删除/重命名叶子文档 | Guide/spec/workdoc 变化 | ROUTING target_docs、front matter |
| 新增智能体或能力（`AGENTS.md`/`CAPABILITIES.md`） | 新工具、脚本、服务 | registry、agent-graph、doc-node-map |
| 调整 graph/registry 节点（含重命名/退役） | 能力迁移、模块改版 | CAP/AGENTS/ROUTING、trigger |
| 定期巡检/CI 自动化 | 版本发布前或周常健康检查 | lint、报告、workdoc |

以下对每个情形给出闭环方案。

#### 8.3.1：创建/更新 `ROUTING.md`
- **输入**：父级 scope/topic/when、计划挂载的 `target_docs`、目录 Owner。
- **操作规范**：
  1. `make doc_route_refresh` → 在 `plan.md` 写“新增 scope/topic”条目并指派责任人；
  2. 复制 `templates/ROUTING.template.md`，按 `scope → topic → when → target_docs` 填写，禁止在 `target_docs` 中写正文；
  3. 运行 `python scripts/doc_node_map_sync.py --write`，确保 doc-node-map 记录新叶子；
  4. 执行 `make route_lint`、`make registry_gen --check`，输出粘贴到 `workdocs/.../context.md#Automation`。
- **交付**：`handoff-history.md` 标记“新 ROUTING 节点 + 责任人 + 命令”；如需补档，在 `tasks.md` 创建 TODO。

#### 8.3.2 ：叶子文档新增/删除/重命名
- **输入**：`target_docs` 现状、受影响的 scope/topic/when。
- **操作规范**：
  1. `rg --files <dir>` + `ROUTING.md` 确认已登记文档，在 `plan.md` 说明操作类型（新增/删除/rename）及影响范围；
  2. 新文档必须先写 front matter（audience/purpose/doc_role/doc_kind），删除/rename 前需在 workdoc 标记“将移除”，并通知依赖方；
  3. 更新 `ROUTING.md` 的 `target_docs`，保持渐进原则；
  4. 运行 `make route_lint`、`make doc_node_map_lint`，rename 时追加 `python scripts/doc_node_map_sync.py --write`。
- **交付**：`context.md#Automation` 粘贴命令日志；PR 描述引用 lint 结果；`handoff-history.md` 记录叶子状态变化与负责人。

#### 8.3.3 ：新增智能体/能力
- **输入**：节点 role/kind/entrypoint、所属模块、能力 owner。
- **操作规范**：
  1. 检索复用：`python scripts/doc_node_map_sync.py --summarize <dir>`、`make registry_show`；若已有能力需记录差异；
  2. `plan.md` 明确需更新文件（`AGENTS.md`、`CAPABILITIES.md`、`registry.yaml`、`agent-graph.yaml`、`ROUTING.md`、`agent-triggers.yaml`/`trigger-map.yaml` 如适用）；
  3. `registry.yaml` 新增节点及 `tools_allowed`、owner，`agent-graph.yaml` 加 node/edge，`AGENTS.md` 编写策略，`CAPABILITIES.md` 写条目并引用 `graph_node_id`，`ROUTING.md` 增加 `target_docs`；
  4. 如需触发器，更新 `doc_agent/orchestration/agent-triggers.yaml`、`trigger-map.yaml` 并声明 `preload_docs`。
- **校验命令**：`python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`、`make capability_index_check`、`make route_lint`、`make registry_gen --check`、`make trigger_check`（若涉及触发器）。
- **交付**：`workdocs/.../context.md#Automation` 贴命令及结果，`handoff-history.md` 记录能力 ID、生效时间、责任人。

#### 8.3.4 ：调整或退役能力/节点
- **输入**：异常报告、受影响的 graph/trigger/doc 列表、回滚计划。
- **操作规范**：
  1. 通过 `make capability_index_check`、`python scripts/registry_check.py` 确定异常节点，在 `plan.md` 拆分任务（rename、退役、触发器清理等）；
  2. 更新 `agent-graph.yaml`（rename/delete + edge 调整）、`registry.yaml`（`agent_capabilities` / `tools_allowed` / 状态）、`CAPABILITIES.md`、`AGENTS.md`、`ROUTING.md`、doc-node-map；
  3. 如涉及触发器/Guardrail，同步 `agent-triggers.yaml`、`trigger-map.yaml` 并运行 `make trigger_check`。
- **校验命令**：`make capability_index_check`、`make route_lint`、`make registry_gen --check`、`python scripts/doc_node_map_sync.py --prune`（退役时）、`python scripts/module_registry_sync.py --write`。
- **交付**：`handoff-history.md` 记录“旧节点 → 新节点/退役”及日期；`ai/maintenance_reports/route-health.md` 标注“节点停用/调整”；`context.md#Automation` 保留命令日志。

#### 8.3.5：定期巡检 / CI 自动化
- **输入**：巡检频次、责任人、必跑命令清单。
- **操作规范**：
  1. 在 `ai/workdocs/active/maintenance/plan.md` 建立巡检模板（路由 lint、能力索引、registry、trigger、guardrail、context 使用等），并在 `tasks.md` 标注周期；
  2. 依次运行 `make dev_check`（需包含 format/lint/tests + `route_lint` + `capability_index_check` + `registry_gen --check`）、`python scripts/doc_node_map_sync.py --report`、`make trigger_check`、`make guardrail_check`、`python scripts/context_usage_tracker.py report --limit 10`；
  3. 将结果写入 `ai/workdocs/active/maintenance/context.md#QA`，若发现异常立即创建 follow-up task，指派负责人和截止时间。
- **交付**：`ai/maintenance_reports/route-health.md` 记录本次巡检摘要；`handoff-history.md` 留存巡检批次、执行人、主要结论。

### 8.4 自动化链路说明文档
- **文档定位**：集中记录链路注册、查询与审计信息，便于智能体快速检索。建议在 `doc_agent/flows/automation-links-guide.md`（`audience: ai`，`doc_role: guide`，`route_role: leaf`）维护面向模型的版本；若需要人类审阅，可在 `doc_human/guides/automation-links.md` 另设对应说明。
- **Front matter 要求**：包含 `audience`、`purpose`、`doc_role`、`route_role`、`updated_at` 等字段，正文保持 ≤150 行、结构化分节。
- **核心内容**：
  - 链路登记表：列出 Link ID、触发器（名称/priority/enforcement）、路由入口（scope/topic/when）、关联 `graph_node_id`、必跑命令、workdoc/报告位置。
  - 变更流程：明确新增/修改/退役时对 `ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md`、`registry.yaml`、`agent-graph.yaml`、`agent-triggers.yaml` 的同步顺序与校验命令。
  - 审计要求：说明 `ai/workdocs/.../context.md`、`handoff-history.md`、`ai/maintenance_reports/route-health.md` 的记录规范，以及例行巡检频率。
- **路由与索引接入**：在根 `ROUTING.md` 或 `doc_agent/orchestration/routing.md` 的相关 scope/topic 中将该指南作为 leaf 引用，确保智能体按渐进式原则快速命中；必要时在 README 中的路由表注册该文档。
- **与触发器/registry 协同**：每条链路的说明应标注对应触发器条目、`graph_node_id` 和 `capability_tags`，并提示在 `doc_agent/orchestration/doc-node-map.yaml` 登记映射；若触发器或能力图调整，应同步更新说明文档并记录时间戳。
- **维护 SOP**：
  1. 新增链路 → 更新说明文档登记表 → 调整路由/能力/触发器配置 → 运行 `make route_lint`、`make trigger_check`、`make registry_gen --check`、`python scripts/doc_node_map_sync.py --write`。
  2. 修改链路 → 在文档的变更日志区记录"旧值→新值"、责任人与日期，并同步 workdoc/Handoff 记录。
  3. 定期巡检 → 根据说明文档列出的表格逐项核对链路状态，若发现缺口，创建追踪任务并在文档中标注"需关注"。

## 9. 脚本规范

> 目标：规范脚本目录、命名、文档、触发机制与质量要求，确保在模板搭建及后续项目迭代中能够可发现、可复用、可编排、易审计。

### 9.1 目录分层与命名
- 根目录 `scripts/operations-guide.md`（AI 主入口，`ROUTING`/触发器必须指向此文档）与 `scripts/README.md`（人类概览，链向 operations-guide 并提供目录速览）；操作指南集中描述目录结构、命名规则、能力复用流程与审批要求，各子目录 README 指向对应章节。
- 根目录 `scripts/` 统一使用蛇形命名 `<功能>_<动作>.<语言后缀>`（如 `build_index.py`、`lint_code.sh`），便于检索与自动生成能力索引。
- 参考目录结构（模板可按需裁剪）：
  ```
  scripts/
    ├── data/               # 数据处理与特征准备层
    ├── inference/          # 模型 / 智能体推理层
    ├── workflow/           # 任务编排与流水线层
    ├── agents/             # 智能体治理与元数据层
    ├── ci/                 # 持续集成与质量保障层
    ├── guardrail/          # 安全与合规策略层
    ├── intermediate/       # 临时/一次性执行脚本层
    ├── AGENTS.md           # 面向 AI 的策略入口
    └── README.md           # 面向人类的总览说明
  ```
  - 每个子目录附带 `README.md`（人读）与可选 `AGENTS.md`（AI 策略），说明职责、入口脚本、依赖环境。
  - 子目录内部的脚本命名同样遵循 `<domain>_<action>.<ext>`，如 `data/clean.py`、`inference/run_model.py`、`workflow/code_review.py` 等。

### 9.2 脚本头信息与接口约定
- 统一头注释、参数声明、dry-run/风险说明需同步记录在脚本操作指南 `scripts/operations-guide.md` 中，便于自动生成能力索引及审核清单；每次更新头信息规范时，应先修改该指南再批量应用。
- 每个脚本使用统一头注释或 YAML 声明，实现"脚本自描述"：
  ```python
  """
  # Script: data_clean.py
  # Purpose: Clean and normalize raw dataset
  # Entry: python scripts/data/clean.py --input raw.csv --output clean.csv
  # Inputs:
  #   - input (path, required)
  #   - config (path, optional)
  # Outputs: clean CSV with normalized schema
  # Risks: modifies files in-place; use --dry-run before production
  """
  ```
- 输入/输出契约：
  - 采用 JSON Schema / dataclass / argparse 定义参数，文件放在 `schemas/scripts/<name>.schema.json` 或脚本内 `SCHEMA = {...}`。
  - 若输出为文件/JSON/日志，需在头信息中说明文件格式、字段含义、错误码。
- 日志与审计：默认输出结构化日志（JSON 或带前缀的 key=value），标注执行时间、输入参数、结果状态、耗时；重要脚本需记录到 `logs/` 或工作文档。

### 9.3 能力登记与编排衔接
- 基础能力登记：
  - 在 `doc_agent/orchestration/capabilities.yaml` 添加条目，并在 `scripts/operations-guide.md` 的"能力登记"章节记录操作步骤、命名约定、审批要求。
- 智能体映射：
  - 对外暴露的脚本需由封装智能体调用，在 `doc_agent/orchestration/registry.yaml` 登记智能体 `id`、`tools_allowed`、`graph_node_id`、`scope`、`triggers`。
  - 在对应目录的 `AGENTS.md` 定义策略（执行守则、安全边界、所需文档/配置），指向 `CAPABILITIES.md` 条目。
- 路由与触发：
  - 在 `ROUTING.md` 中为脚本/能力添加 `scope → topic → when`，`target_docs` 指向脚本 README/指南、schema、策略文档。
  - 在 `doc_agent/orchestration/agent-triggers.yaml` 配置触发条件（文件模式、命令关键词），指定 `target_agent` 与需预加载的文档。
- doc-node-map：执行 `python scripts/doc_node_map_sync.py --write` 更新脚本与路由的映射，确保智能体和文档索引保持一致。

### 9.4 触发机制与使用流程
- 触发登记表可维护于 `scripts/operations-guide.md`（附录）或 `scripts/trigger_map.yaml`，确保开发者/智能体获取统一触发信息。
- 流程指引：
  1. 触发器命中（文件改动或命令意图），按照 `agent-triggers.yaml` 加载守则文档与 `ROUTING.md`。
  2. Orchestrator 根据 registry/capabilities 找到封装智能体，执行脚本的 dry-run（如支持）或生成计划。
  3. 人工确认（高风险脚本必须）后执行实际命令，并将结果写入 workdoc、handoff。
  4. 自动触发文档更新（CHANGELOG、summary README 等）与遥测统计。

### 9.5 复用策略与版本管理
- 新增脚本前需检索：在 `scripts/operations-guide.md` 中提供"复用流程"Checklist，并指向 `CAPABILITIES.md`、capability registry、doc-node-map 的检索命令。
- 版本策略：
  - 使用 `__version__` 或 changelog 记录脚本变化，说明兼容性；破坏性改动需保留旧命令或提供迁移指南。
  - 退役脚本时更新 capability registry、registry.yaml、doc-node-map，并在 README/CHANGELOG 标注。

### 9.6 文档体系联动
- 顶层 `scripts/operations-guide.md`（或操作指南）包含目录说明、命令速查、SOP；子目录 README 引导至相应章节。
- `AGENTS.md`（若存在执行权限）：声明安全策略、可执行命令、禁止操作、依赖 schema。
- `CAPABILITIES.md`：按能力 ID 收敛信息，便于智能体检索；包含 `graph_node_id`、触发 `when`、依赖脚本、输入输出说明。
- `ROUTING.md`：只保留路由条目，指向子目录 README / 说明文档，不直接堆脚本细节。

### 9.7 测试与质量要求
- 质量门槛（复杂度、dry-run、测试、日志）需在 `scripts/operations-guide.md` 中设定统一指标，便于 lint/CI 校验。
- 代码质量：
  - 遵循语言风格指南（PEP8、shellcheck 等），复杂度（McCabe）控制在合理范围（建议 < 10；超出需说明理由）。
  - 准确性指标：功能脚本需提供单元/集成测试（`tests/scripts/`），验证输入输出与失效场景；对接外部系统时提供 mock 或 dry-run。
  - 必须支持 `--dry-run` 或等效模式（高风险脚本），默认不对生产资源造成副作用。
- CI 集成：
  - `make dev_check` 中包含 `python_scripts_lint.py`、`shell_scripts_lint.sh`、`pytest`（如适用）。
  - 针对关键脚本配置 `make script_smoke_test` 或 `make script_integration_test`。
- 审计要求：
  - 每次脚本变更需在 workdoc / handoff 记录，说明测试结果、dry-run 输出、影响范围。
  - 关键脚本执行需要日志落地，供 `scripts/context_usage_tracker.py` 采集统计。

### 9.8 新增脚本规范
| 阶段 | 必做事项 | 产出/校验 |
| --- | --- | --- |
| 检索与立项 | 根据 `scripts/operations-guide.md#新增脚本流程` 检索现有脚本/能力，记录在 `plan.md#Alternatives`；如需新脚本，创建“脚本名称、用途、Owner”条目 | `plan.md` + `tasks.md`（含输入/输出/触发器/风险） |
| 设计 | 定义头信息（Purpose/Entry/Inputs/Outputs/Risks）、参数 Schema、dry-run 策略、日志格式，写入 `plan.md#Design`；必要时在 `config/` 或 `schemas/scripts/` 创建配置/Schema 模板 | 设计文档 + schema/prompt/示例配置 |
| 实现 | 按 9.2/9.3 规范创建脚本，确保 `--dry-run`、日志、错误处理；同步更新示例配置、README 片段；若脚本需要凭证/配置，引用 `modules/config/` 既有条目 | 新脚本 + 相关配置 |
| 能力登记 | 在 `doc_agent/orchestration/capabilities.yaml`、`scripts/operations-guide.md`、`CAPABILITIES.md` 登记能力；如需 Agent/Graph 支持，更新 `doc_agent/orchestration/registry.yaml`、`agent-graph.yaml`、`AGENTS.md`、`ROUTING.md`；必要时更新 `agent-triggers.yaml`、`trigger-map.yaml` | 运行 `python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`、`make route_lint`、`make capability_index_check`、`make trigger_check` |
| 测试与文档 | 编写/运行单元或集成测试、`--dry-run`、`make script_smoke_test`（如定义）、`make dev_check`；在 README、CHANGELOG、workdoc 记录使用方式与测试结果 | 测试日志粘贴于 `workdocs/.../context.md#QA`; `tasks.md` 勾选 "Tests" |
| 发布与审计 | `make route_lint`、`make capability_index_check`、`make registry_gen --check`、`make dev_check` 全部 PASS 后提交 PR；在 `handoff-history.md` 记录脚本 ID、触发器、上线时间、回滚方案；在 `ai/maintenance_reports/route-health.md` 标记“脚本新增” | 审批/回溯记录齐全，命令输出附在 workdoc |

> **验收标准**：脚本未完成上述任一阶段或缺少命令日志一律不得合并；复用/退役按 9.5、9.6 的策略执行。

### 9.9 触发器矩阵与维护（可选附录）
- 建议在 `scripts/operations-guide.md` 或 README 中维护脚本与触发器矩阵，字段包含脚本 ID、触发器名称、触发条件、对应 scope/topic/when、智能体、预载文档、dry-run 方式、审批要求；该矩阵与 `trigger_map.yaml` 保持同步。
- 例行巡检：
  - 每周执行 `make trigger_check`、`make capability_index_check`，确保触发器和能力索引同步。
  - 将最新矩阵导出至 `ai/maintenance_reports/route-health.md`，作为脚本体系健康检查的一部分。

> 通过上述规范，脚本系统在模板阶段即可形成"目录清晰、能力登记、触发闭环、质量可控、审计留痕"的骨架；各步骤的详细操作清单、审批流程、故障排查可在 `scripts/operations-guide.md` 持续更新。

## 10. 交互式项目开发

> 目标：让代码大模型在“检索 → 计划 → 执行 → 记录 → 回收”链路中保持高节奏、低风险。第 10 章聚焦互动式开发态的编排骨架，确保前序文档规范、路由、能力索引、自动化链路都能在一线任务中被正确触发与复用。

### 10.1 编排分层与职责（操作规范）
- **交互控制层（Orchestrator / Guardrail）**
  - 资产：根 `ROUTING.md`、`doc_agent/AGENTS.md`、`doc_agent/orchestration/agent-triggers.yaml`、`registry.yaml`、`trigger-map.yaml`。
  - 职责：
    1. 接到任务后先加载对应 scope/topic/when 的 `ROUTING.md` 和 `AGENTS.md`，确认可/不可执行操作；
    2. 根据 `agent-triggers.yaml` / `trigger-map.yaml` 匹配触发器，执行 10.2 规定的加载策略、能力与命令；
    3. 每次控制层执行命令前必须跑 `make route_lint` 或局部 `python scripts/doc_node_map_sync.py --report`，并把结果粘到 `workdocs/active/context.md#QA`；
    4. Guardrail 审批要求记录在 `workdocs/active/plan.md#Approvals`，审批日志写入 `handoff-history.md`。
- **执行协作层（Module / Service Agents）**
  - 资产：`modules/<instance>/AGENTS.md`、`CAPABILITIES.md`、`doc/` 下的 quickstart/runbook/contract、脚本 SOP、`modules/registry.yaml`。
  - 职责：
    1. 在加载控制层策略后，根据 `AGENTS.md` 的 Allowed/Required 列表选择工具，调用时引用 `CAPABILITIES.md` + `capabilities.yaml`；
    2. 每次执行脚本/命令需将命令、目的、输出写入 `workdocs/active/context.md#Automation`，并在 `tasks.md` 更新状态；
    3. 任何新能力/脚本都按 5.2.2 流程登记基础能力与 agent，再更新 `modules/registry.yaml`、`doc-node-map`；
    4. 若协同多个模块，必须在 `workdocs/active/plan.md#Dependencies` 和 `modules/registry.yaml#requires` 标注依赖。
- **可观测与学习层（Telemetry / Workdoc / Error Log）**
  - 资产：`ai/workdocs/`、`handoff-history.md`、`ai/maintenance_reports/*`、`context_usage_tracker.py` 输出、`ai/workdocs/active/<task>/context/*.md`。
  - 职责：
    1. `plan/context/tasks` 结构按 10.3 维护，任何命令/审批/触发器结果都要写入对应区域；
    2. `context_usage_tracker.py`、hook 日志定期导出到 `ai/maintenance_reports/route-health.md`，用于 11.章测试；
    3. 错误与经验统一写入 `context/lessons.md`（本地）与 `ai/maintenance_reports/retrospective.md`（全局），并在 10.6 流程中引用；
    4. 交接时更新 `handoff-history.md`（参与人、时间、下一步）并在 `tasks.md` 打勾。
- **触发与回放层（Hooks / Trigger / Slash 命令）**
  - 资产：`agent-triggers.yaml`、`trigger-map.yaml`、hook 实现脚本（UserPromptSubmit/PreToolUse/PostToolUse）、`python scripts/trigger_runner.py`、`scripts/operations-guide.md#trigger-matrix`。
  - 职责：
    1. 根据 10.2 流程执行触发器：捕获事件→加载策略/能力→生成 workdoc 任务→执行命令→日志回放；
    2. Hook 必须在 `PostToolUse` 阶段把命令输出、活跃文件、触发器 ID 写入 `context/active-files.md` 和 `context.md#Triggers`；
    3. 每周运行 `make trigger_check`、`python scripts/trigger_runner.py --dry-run all`，并将结果同步到 `ai/maintenance_reports/route-health.md`；
    4. 新增/修改触发器需按 10.2 中的同步命令更新 trigger-map/registry/graph/ROUTING，并将命令输出贴入 workdoc。

### 10.2 触发器与 Hook 联动（Trigger → Agent Policy → 能力执行）

1. **真相源与结构**
   - `doc_agent/orchestration/agent-triggers.yaml`：定义 trigger ID、匹配条件（path/prompt）、`target_agent`、`enforcement`、`preload_docs`、`required_commands`、`auto_tasks`。
   - `doc_agent/orchestration/trigger-map.yaml`：维持 trigger → agent policy → capability →命令/脚本的映射，由 `python scripts/trigger_map_sync.py` 生成；README 仅引用该文件，不作为 SSOT。
   - 对应 `AGENTS.md` 必须包含 `Trigger Handling` 小节，说明命中该 trigger 时的允许操作、必跑命令、审批要求；`CAPABILITIES.md` 描述可调用脚本/工具。

2. **事件捕获（Hooks）**
   - UserPromptSubmit：识别自然语言意图，按 `agent-triggers.yaml` 中的 prompt 匹配加载 trigger 及其策略。
   - PreToolUse：在执行任何命令前检查触发器是否要求先跑特定命令或审批（`required_commands`），未满足则阻断。
   - PostToolUse：记录本次命令的输出、修改的文件、影响的 service domain，并按 `trigger-map.yaml` 更新状态。

3. **触发器执行流程**
   1. **加载策略**：根据 trigger ID 加载 `AGENTS.md#Trigger Handling`，明确 Allowed/Blocked 操作、guardrail 审批、手动确认。
   2. **加载能力**：读取 `CAPABILITIES.md` / `capabilities.yaml`，注入 `preload_docs` 和可执行脚本（Make 目标、Python 脚本等）；必要时在会话中附上参数示例。
   3. **生成工作事项**：在 `modules/<instance>/workdocs/active/tasks.md` 自动创建 “Trigger:<ID>” 子任务，并在 `plan.md#Approvals` 或 `context.md#Triggers` 记录命中时间、必读文档、待跑命令。记录内容只包含事实（命中 + 命令结果），不重复策略。
   4. **执行命令 / 工具**：Hook 或 agent 调用注册的能力；`required_commands` 通过 PreToolUse enforce；可使用 `python scripts/trigger_runner.py --event <id>` 进行 dry-run。
   5. **审计记录**: PostToolUse hook 将命令输出、活跃文件写入 `context.md#Automation` 与 `context/active-files.md`；若 trigger 为 block 级别，还需在 `handoff-history.md` 记录审批结果。

4. **新增 / 修改触发器流程**
   1. 更新 `AGENTS.md`（增加 Trigger Handling 小节）和 `modules/registry.yaml#graph_bindings`（如触发器涉及新 agent）。
   2. 编辑 `agent-triggers.yaml`、`trigger-map.yaml`，运行 `python scripts/trigger_map_sync.py --write`、`python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`，并执行 `make trigger_check`、`make route_lint`、`make capability_index_check`、`make registry_gen --check`、`python scripts/registry_check.py`。
   3. 若触发器调用新能力，按 5.2.2 的“新知识与能力登记”流程先注册基础能力，再注册 agent。
   4. 在 `ai/workdocs/active/<task>/context.md#QA` 粘贴上述命令输出，并在 `ai/maintenance_reports/route-health.md` 标记新增/调整。

5. **回放与巡检**
   - `python scripts/trigger_runner.py --event <id> --dry-run`：验证 trigger 是否能正确加载策略/能力/命令；CI 中配合 `make trigger_check` 执行。
   - `context_usage_tracker.py` 和 hook 日志会将触发器命中次数、失败原因写入 `ai/maintenance_reports/route-health.md`，供第 11 章测试阶段审阅。
   - 每周巡检时导出 `trigger-map.yaml` 的快照，确保 trigger 与 agent/capability/registry 映射一致；发现孤立 trigger 必须立即补齐策略或删除条目。

### 10.3 上下文恢复机制（ai/workdocs）
```
ai/workdocs/
├── active/
│   └── <task-name>/
│       ├── plan.md      # 战略计划（~300 行）
│       ├── context.md   # Hub：5 个速览分区 + 指向子文档（≤120 行）
│       ├── context/
│       │   ├── session-progress.md   # SESSION PROGRESS 详单（doc_role: context_log）
│       │   ├── decisions.md          # 审批 / 决策日志（doc_role: decision_log）
│       │   ├── risks.md              # 风险与阻塞清单（doc_role: risk_log）
│       │   ├── active-files.md       # 活跃文件、diff 摘要、post-tool tracker 输出
│       │   └── lessons.md            # 错误与学习详单（doc_role: lessons_log，对应 10.6）
│       └── tasks.md     # 任务拆解、验收标准、进度（~200 行）
└── archive/
    └── <completed>/
```
- **plan.md**：承接第 5–9 章的实施策略与验收指标，按阶段列出 Guardrail、触发器、脚本、审批与测试要求。
- **context.md（Hub）**：保持 ≤120 行，固定包含 `Session Progress`、`Decisions & Approvals`、`Risks & Blocks`、`Active Files`、`Learning Snapshot` 五个小节，每个小节只写 1–2 行摘要并链接到对应子文档：
  - `Session Progress`：显示 ✅ / 🟡 / ⚠️ 状态表，并指向 `context/session-progress.md` 获取完整历史。
  - `Decisions & Approvals`：同步 Guardrail/人工审批的结果，落盘在 `context/decisions.md`。
  - `Risks & Blocks`：列出当前前三个风险摘要，并引用 `context/risks.md` 查看对策。
  - `Active Files`：引用 post-tool tracker，链接 `context/active-files.md` 以追踪 diff、命令输出。
  - `Learning Snapshot`：提供 1–2 句教训摘要并链接 `context/lessons.md`（详见 10.6）。
  - 任一小节超过 40 行或需要表格时，立即把详情写入同名子文档，仅在 Hub 中保留摘要 + 链接，保证渐进式阅读。
- **context/session-progress.md**：`audience: ai`、`doc_role: context_log`、`route_role: leaf`，记录阶段目标、触发器命中、下一步计划，可按日期分节。
- **context/decisions.md**：`doc_role: decision_log`，收敛审批材料、决策人、输入/输出与引用文档，供 10.5 审批闭环引用。
- **context/risks.md**：`doc_role: risk_log`，跟踪风险等级、缓解动作、倒计时，必要时引用 tasks 中的阻塞项。
- **context/active-files.md**：`doc_role: context_log`，同步 post-tool tracker 的活跃文件、命令与 diff 摘要，便于跨智能体 handoff。
- **context/lessons.md**：`doc_role: lessons_log`，是 10.6 要求的权威错误/学习详单；Hub 只保留引用，repo 级复盘另存 `ai/maintenance_reports/retrospective.md`。
- **tasks.md**：以 scope → capability → trigger 的顺序列出待办，关联触发器、命令、文档，所有变更立即打勾或追加备注。
- **操作节奏**：启动任务时初始化目录，执行每个阶段前复核 plan；每次执行完命令或触发器后同步更新 Hub + 对应子文档；完成里程碑后更新 context / tasks 并归档快照；结项时迁移至 `archive/` 并在 `handoff-history.md` 登记收尾动作。

#### 10.3.1 任务可见性与协同规范
- `context.md#Session Progress` 必须包含“可见性”小节：列出触发器命中情况、待审批项、下一次触发检查时间，仅保留摘要 + 链接。
- `tasks.md` 要与 `agent-triggers.yaml` 联动：记录每个任务的触发器 ID、必跑命令、责任人，并用 `status`（`pending/in_progress/completed`）同步进度；第 8 章触发器巡检脚本会比对这些字段。
- `context/active-files.md` 由 PostToolUse hook 自动写入活跃文件、修改类型、关联触发器；人工或跨团队协作时需补充说明，确保 handoff 能快速定位影响范围。
- 需要人工或跨团队协作时，在 `plan.md#Collaboration` 列出 handoff 点、所需文档、review agent（如 code-architecture-reviewer）及触发命令；未登记视为禁止。

#### 10.3.2 错误记录与学习
```
# 错误记录
## ERROR-001: 忘记更新 CONTRACT.md 导致接口不一致
- **日期**: 2025-11-08
- **上下文**: modules/payments/api/contract 更新
- **错误**: 修改 API 但未更新 CONTRACT.md
- **后果**: 前端调用失败，排查耗时 2 小时
- **教训**: 接口改动须同步 doc/CONTRACT.md 并刷新 contracts_baseline
- **AI 提醒**: 触发 `contract-change` guardrail，执行 `make contract_compat_check`
```
- 记录位置：
  - `ai/workdocs/active/<task>/context/lessons.md`：存放完整条目（ID、日期、影响面、复盘、后续动作），使用 `context/lessons.md#ERROR-001` 固定锚点；
  - `context.md#Learning Snapshot`：仅保留 1–2 句摘要 + 链接，避免膨胀上下文；
  - 影响全仓的经验同步至 `ai/maintenance_reports/retrospective.md` 与 `handoff-history.md`，保持统一追溯。
- 联动动作：为每条错误创建任务（如“更新 trigger contract-change 的 preload 文档”），在 `tasks.md` 标记完成，并把修复命令输出链接到 `context/lessons.md#ERROR-001`；必要时同步更新 `ROUTING.md`、能力文档或触发器条目。
- 学习闭环：再次触发同类 guardrail 时，通过 `preload_docs` 或 workdoc 链接自动加载对应 lesson 与复盘，确保经验进入编排体系而非散落在历史记录中。

### 10.4 Guardrail 执行与审批闭环（规范流程）

1. **触发入口与策略加载**
   - 仅当 `agent-triggers.yaml` 中的条目标记 `enforcement: block` 或 `priority: critical` 时，才进入 Guardrail 流程。
   - 触发后必须加载目标 agent 的 `AGENTS.md#Guardrail`（或 `Trigger Handling`）段落，确认 Allowed/Blocked 操作、`required_commands`、审批人、需阅读的政策文档。
   - Hook 自动注入 `preload_docs`（安全政策、SOP、recent incidents），并在 `modules/<instance>/workdocs/active/plan.md#Approvals` 生成审批条目：记录 Trigger ID、变更描述、风险等级、目标环境、预计上线窗口。

2. **审批操作步骤**
   1. 执行 `required_commands`（如 `make db_lint`、`make trigger_check`、`python scripts/guardrail_runner.py --event <id> --dry-run`），命令输出粘贴到 `workdocs/active/context.md#QA`。任何一条命令失败即终止流程。
   2. 在 `context.md#Approvals` 汇总审批材料（diff、测试日志、脚本输出、风控评估），并在 `plan.md#Approvals` 更新状态（Pending → In Review → Approved/Rejected）。
   3. 审批人依据 `AGENTS.md` 守则在 `context.md#Approvals` 写入决议、依据、回滚策略、生效范围；若拒绝，注明 Block 原因与解封条件。
   4. 在 `modules/<instance>/workdocs/active/tasks.md` 自动创建 “Guardrail:<ID>” 子任务；审批完成后勾选，附上 `handoff-history.md` 或 `ai/maintenance_reports/` 的审批链接。
   5. 审批通过后才能继续执行触发器指向的能力；需要人工执行命令时，必须在 `context.md#Automation` 记录执行者、命令、输出。

3. **执行监控与回滚**
   - Guardrail agent 确认 hooks 是否正确预载文档、`required_commands` 是否全部成功，并将检查结果写入 `workdocs/active/context.md#Triggers` 与 `ai/maintenance_reports/route-health.md`。
   - 若需要回滚或审批拒绝：执行回滚脚本（在 `AGENTS.md` 或审批记录中列出），记录在 `context.md#Automation`，并在 `handoff-history.md` 更新“Guardrail 回滚完成/拒绝原因”。
   - 所有 block 触发器的审批/执行日志必须可通过 `python scripts/guardrail_runner.py --event <id>` 回放，以便第 11 章测试阶段审计。

4. **自动化与巡检**
   - 新增或调整 Guardrail 流程需同步：`agent-triggers.yaml`、`trigger-map.yaml`、相关 `AGENTS.md`、`modules/registry.yaml#graph_bindings`；执行 `python scripts/module_registry_sync.py --write`、`python scripts/doc_node_map_sync.py --write`、`make trigger_check`、`make route_lint`、`make capability_index_check`、`make registry_gen --check`、`python scripts/registry_check.py`，并在 `workdocs/active/context.md#QA` 粘贴结果。
   - 每周运行 `make guardrail_check`（封装 `guardrail_runner.py --dry-run all`）验证所有 block 触发器可自动通过审批流程；结果写入 `ai/maintenance_reports/route-health.md`。
   - Guardrail 条目在 workdocs 中不得删除或改写；交接时需引用原条目并在 `handoff-history.md` 记录“审批完成/回滚完成/否决原因”。

## 11 测试及维护
面向智能体的测试与 CI 需要在模板阶段就建立三层防线，分别覆盖结构一致性、能力契约和自动化链路执行，确保与前序章节定义的规范闭环对齐。

### 11.1 结构与路由一致性校验
- **目标**：延续第 1 章文档规范与第 2 章路由体系，确保所有面向 AI 的入口（`ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md` 等）结构正确、路径无断链。
- **执行步骤**：
  - 运行 `make route_lint` 与 `python scripts/doc_node_map_sync.py --report` 校验 `context_routes`、front matter、`doc-node-map` 的一致性。
  - 调用 `make registry_gen --check` 比对根 README 路由表与 registry 输出，确认新增/调整文档已登记。
  - 将 lint 结果、必要的修复操作记录在 `ai/workdocs/active/<task>/context.md#QA`，并在 `handoff-history.md` 标注结构性调整。
- **验收标准**：无语法错误、无孤立文档，`route_role`、`doc_role`、`doc_kind` 等元数据与路由映射一致，检查命令全部通过。

### 11.2 能力契约测试
- **目标**：复用第 3 章的能力索引与编排图，以及第 6、7 章数据库与 API 契约规范，验证智能体输入输出契约与 graph/registry 同步。
- **执行步骤**：
  - 针对能力域运行 `make capability_index_check`、`python scripts/registry_check.py`，确保 `graph_node_id`、`capability_tags`、`tools_allowed` 一致。
  - 数据库与 API 模块执行 `make contract_compat_check`、`python scripts/type_contract_check.py` 等专用测试，确认 `.contracts_baseline/`、OpenAPI 与实现同步。
  - 若出现契约变更，在 `ai/workdocs/active/<task>/plan.md` 记录审批链路，并按照第 6/7 章流程更新 `doc_agent/orchestration/doc-node-map.yaml` 与相关策略文档。
- **验收标准**：所有契约测试通过，graph/registry/能力索引无漂移，工作文档含最新的审批与测试记录。

### 11.3 自动化链路与触发器演练
- **目标**：承接第 8 章自动化链路与第 9 章脚本规范，通过 Dry-run 与触发器校验确保执行路径可靠（暂不引入成本监控）。
- **执行步骤**：
  - 执行 `make trigger_check`、`python scripts/context_usage_tracker.py report --limit 10`，验证触发器命中率与链路健康状况。
  - 对新增/修改的链路，按照第 8 章 SOP 运行对应脚本的 `--dry-run` 模式，检查日志、风险提示与回滚指引是否完整。
  - 在 `ai/workdocs/active/<task>/context.md#Automation` 粘贴 Dry-run 输出，并同步更新 `ai/maintenance_reports/route-health.md` 的链路健康记录。
- **验收标准**：触发器配置无报错，自动化脚本 Dry-run 成功且日志齐全，相关文档与工作记录已更新；成本监控留作后续扩展项。

### 11.4 交付与部署（择期启用）
- 在完成前三层校验后，可按需扩展构建与部署环节（如生成 Agent 索引、刷新路由服务），对应流程应遵循第 8 章的“新增链路”SOP，并在 CI 中串联执行。
- 本阶段暂不强制纳入成本预算或生产级部署动作，可在后续演进时根据项目需求补充。

### 11.5 PR 提交流程要求
- **提交前检查**：必须运行 `make dev_check`（含格式化、测试、lint、路由与能力校验）、`make trigger_check` 以及本章列出的契约测试，确保输出为 PASS，并将关键命令结果粘贴到 `ai/workdocs/active/<task>/context.md#QA`。
- **文档同步**：确认被修改的 `ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md`、registry、workdoc 已按第 1–10 章要求更新；若涉及高风险触发器，附上 Guardrail 审批记录与 handoff 链接。
- **小步提交与自动验收**：PR 需拆成可回放的小步提交，每次 AI 输出生成的 commit 或变更集都必须触发 `make dev_check` 或等效的自动回归测试作为硬验收，通过后才能继续下一步操作。
- **需求可验证化**：在 PR 描述与 workdoc 中把需求写成可验证语句，明确修改范围、禁止改动的文件或接口、必须通过的测试与检查项，便于评审按条目逐一验收。
- **PR 描述规范**：遵循仓库约定的模板，至少包含摘要、受影响范围、验证步骤日志、workdoc 链接以及是否需要后续跟进的 TODO；对 AI 面向文档的改动需在描述中列出新增/删除的 route 节点。
- **审批要求**：如命中 `agent-triggers.yaml` 中的 `enforcement: block` 条目，须在 PR 中 @ 对应责任人确认，待审批通过后方可合入。

## 12. 模板初始化规范（全仓级）
> 目标：在空仓或新建分支时一次性搭建好文档路由、策略、能力索引与自动化检查，使智能体能立即按渐进式原则执行任务。本章聚焦仓库级初始化，不涵盖具体模块的脚手架流程。

### 12.1 前置准备与目录基线
- **必备路径**：`doc_agent/`（含 `ROUTING.md`、`orchestration/`、`policies/` 等）、`doc_human/`、`ai/workdocs/active|archive`、`ai/maintenance_reports/`、`scripts/`、`db/engines/`、`temp/`。初始化时确认目录存在且权限正确。
- **核心模板文件**：复制或生成根 `ROUTING.md`、`AGENTS.md`、`CAPABILITIES.md`、`README.md` 的空白模板，预填 front matter 与占位段落；确保文档遵循第 1 章规范（audience/purpose/doc_role/doc_kind/route_role）。
- **工作文档**：在 `ai/workdocs/active/template-bootstrap/` 下建立 `plan.md`、`context.md`、`tasks.md`，将初始化步骤、验证命令和责任分工写入，作为后续审计依据。

### 12.2 路由与策略骨架
- **根路由**：依据第 2 章的 scope→topic→when 结构，在根 `ROUTING.md` 中注册模板必备主题，例如“Execution – Project Initialization”、“Governance – Guardrail Setup”、“Systems – Automation Scaffolding”，并指向对应叶子文档或下游路由。
- **策略文档**：编写 `doc_agent/AGENTS.md`，声明 orchestrator 的角色、可/不可执行操作、必跑命令、handoff 规则；与 `doc_agent/policies/` 中的安全/质量文档保持一致。
- **能力索引**：在 `doc_agent/CAPABILITIES.md` 或 `doc_agent/orchestration/capabilities.yaml` 中列出初始能力域（例如 DocOps、Guardrail、Workflow & Telemetry、Data & Schema、API Gateway、Scripts Ops），并为每个域占位 `graph_node_id` 与 `capability_tags`。

### 12.3 编排资产登记
- **Registry**：在 `doc_agent/orchestration/registry.yaml` 注册根级智能体、能力域节点及其 `tools_allowed`、`scope`、`ownership`；若暂未实现具体脚本，可设置占位命令并标注 TODO。
- **编排图**：初始化 `doc_agent/orchestration/agent-graph.yaml`，定义 orchestrator→能力域节点的最小 DAG（至少包含入度、出度、entry_nodes、terminal_nodes）；引用统一 schema（位于 `schemas/`）并预留 transform/condition 空位。
- **doc-node-map**：运行 `python scripts/doc_node_map_sync.py --init` 或等效脚本，同步文档与 graph/registry 的映射，确保 `ROUTING.md` 的 `target_docs` 与 `graph_node_id` 对应。

### 12.4 自动化命令与脚本清单
- **Make 目标**：在 `Makefile` 中注册初始化阶段需要的命令，如 `make dev_check`、`make route_lint`、`make capability_index_check`、`make trigger_check`、`make registry_gen --check`；没有实现的命令需在 workdoc 中列出计划完成时间。
- **脚本目录**：按照第 9 章规范准备 `scripts/operations-guide.md`、子目录 `scripts/data|workflow|guardrail|ci|intermediate` 等 README，占位说明脚本命名、输入输出、dry-run 规则。
- **触发器模板**：创建 `doc_agent/orchestration/agent-triggers.yaml` 的基础结构，定义 `version`、`metadata`、空的 `triggers` 数组及示例条目（如 `template-bootstrap`）以检查配置是否生效。

### 12.5 初始化校验流程
- **命令执行**：按顺序运行 `make route_lint` → `make capability_index_check` → `make registry_gen --check` → `make trigger_check` → `make dev_check`，将输出粘贴到 `ai/workdocs/active/template-bootstrap/context.md#QA`。
- **文档巡检**：确认所有入口文档 ≤ 要求行数，front matter 完整；检查 README 路由表是否同步并记录在 `context.md#Docs`。
- **资产登记**：在 `ai/maintenance_reports/route-health.md` 写入“Template Initialization”条目，包含上述命令结果、未完成的 TODO、下一次复盘时间。

### 12.6 交接物与后续动作
- **交接包**：在 `ai/workdocs/active/template-bootstrap/` 完成 checklist 后，归档到 `ai/workdocs/archive/template-bootstrap/` 并更新 `handoff-history.md`（记录范围、命令结果、待办）。
- **PR 要求**：首次初始化的 PR 需引用第 11.5 节要求，附带命令日志、路由/能力对照表、未完成项清单。
- **后续拓展**：模块或特定能力的脚手架可在完成本章节的基线后另行编写模块化流程文档，并在根路由中新增对应 topic/when。
