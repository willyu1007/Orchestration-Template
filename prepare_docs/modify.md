# 讨论变更总结
我们对初版需求（request.md）进行了讨论，主要包括命名、规范、共识、结构调整、流程调整、机制增强、分歧解决等几个方面。

## 1. 文档规范
- **命名规范**：原始设计意图是在模块初始化时由脚手架同步创建“八大文档”以确保必要文档齐全。讨论指出“八大文档”提法源自过时版本，已不再适用。因此调整为明确列出模块实例初始化需要创建的**关键文档和目录**，并通过脚手架一次性生成。这样做避免了过时概念造成的歧义，确保模块初始化时**实际所需的文件**都被正确创建和同步，核心目标不变。
    
-   **命名规范**：原始设计曾考虑规范叶子文档类型和命名（如是否统一使用 quickstart、guide、runbook 等名称）以承上启下。讨论后决定
    -  `Quickstart`不是固定的文件/位置，而是角色，起到承接ROUTING.md与衔接下游文档的作用
    -  `Quickstart`类型的文档命名不做限制，但 front-matter 必含 `route_role: quickstart` 与 `handoff:` 指向下一跳。 
    -  文档路由体系的下游文档（如各叶子说明、合同`CONTRACT.md`等），只需格式合规且利于检索，无需严格命名统一。
-   **命名规范**：原设计明确了面向AI的三类核心文档命名规范（路由体系用  `ROUTING.md`，能力索引用  `CAPABILITIES.md`，策略规范用  `AGENTS.md`），并要求文档标明受众和角色，所有架构体系相关的AI文档必须使用约定文件名。讨论后决定：
    -  文档路由体系确认使用 `ROUTING.md`
    -  代码大模型的入口文档确认使用 `AGENTS.md`，用以说明策略、体系入口、边界和总则等
    -  能力索引使用`ABILITY.md`替代 `CAPABILITIES.md`
    -  面向人类的文档除 `README.md` 外无硬性命名要求
-   **格式规范**：经由讨论后，决定增加Schemas格式规范 和 示例文档的维护说明。这两类文档都是给  Examples，AI 阅读）**  
    - `spec/front-matter.schema.yaml` / `spec/workdocs.schema.yaml` / `spec/registry.schema.yaml` / `spec/triggers.schema.yaml`。  
    - `examples/`：按渐进式提供 scope-topic-when、模块示例架构、功能体说明样例。
-   **流程调整**：原设计提供了大量规则和结构，可能导致一定**重复和复杂**。讨论一致认为需精简流程，消除不必要的重叠：
    - 在搭建模板时，先明确核心思路，列出所需基础功能和规范文档，然后对这些要素进行归类整合，采取自下而上的方式逐步搭建。
    - 每步都检查并剔除多余部分，降低系统复杂性。此调整旨在确保流程稳健清晰，避免无效冗余设计，提高模板的可维护性。
-   **结构调整**：原设计认为，每个有子目录的目录都需要单独维护路由体系文档（ROUTING.md）、功能体系文档（ABILITY.md）、和策略文档（AGENTS.md）。讨论后决定：根目录下只保留AGENTS.md，使其成为全局策略和护栏的唯一入口
-   **机制增强**：对所有重要机制（文档路由、功能路由、策略路由、上下文恢复机制、trigger、guardrial、模块实例/类型/层级）的相关结构，使用"Unified Configuration Schema， single source (like a YAML tree)"，并开发相关工具，进一步提升可靠性。
   
## 2. 文档路由体系

-   **机制增强**：原方案主要关注通用文档体系，没有单独强调实际开发中的项目/模块特定需求文档。经讨论决定：对于每个具体项目或模块实例的重要需求说明或背景文档，也必须转化成AI可读取的文档并正确接入既有体系。即新增的特定需求文档应当挂载到相应的路由节点上，成为路由叶子或其下游，使代码大模型能够按照统一思路获取项目特殊要求。此举将“需求->文档->路由”本身视为一条规则，保证AI不会遗漏项目定制信息。
        
-   **共识确认**：讨论澄清了文档路由注册的范围：
    -   路由体系注册的对象仅限各级  `ROUTING.md` 及其直接叶子节点文档，不包含叶子节点引用的更下层内容，也不涉及纯人类阅读文档。
    -   所有路由和叶子文档必须在注册登记，但叶子文档内部再细分的章节或附属示例文件则遵循渐进式原则按需阅读，无需在路由中逐一注册
    -   叶子文档如果包含了多个下级文档，则文档内容需要体现渐进式原则，按照情形引导代码大模型阅读
    
## 3. 功能路由体系（编排图）

-   **机制增强**：原设计构想的**功能路由**（能力编排图）主要以“功能体”（相对完整的智能体/能力组合）为节点。讨论中提出改进方案：
    - 功能体系的结构按照两级五类设计，两个级别分别是 "low tier" 和 "high tier"
    - low tier 包含"script"、"mcp"、"api"三种类型。前者为repo内部实现的脚本，后两者为mcp/API等外部服务  
    - high tier 包含"workflow" 和 "agent" 两种类型。前者是更偏确定性的pipeline ，主要拼装repo内脚本；后者更偏动态驱动，会综合内外部能力
    - 只有high tier会进入功能路由体系和接入编排器，trigger和guardrail只对这些高层入口做策略管控。不要让编排器看到low tier节点
    - high tier 和low tier都需要注册，保持两本表，注册中yaml需要明确tier 和 kind
    - low tier 能力保持可替换，high tier住关心可以调用的low tier 的主键id， 具体调用交给配置/著发起
    
-   **命名规范**：原方案使用  **CAPABILITIES**  一词作为能力索引文档名（及doc_kind）。讨论发现该命名可能引起歧义：基础能力称作 capability，而编排层的智能体本质也是组合能力的一种，但通常被称为agent。因此团队意识到需要澄清或调整命名，方案如下：
    - CAPABILITIES.md的命名换成 ABILITY.md
    - high tier 使用 "ability_registry"进行注册，low tier 使用 "method_registry"进行注册，
    - low tier 使用 "method_registry “进行注册，以业务动作为主键，同一业务动作允许多种实现。
    - low tier的命名方式使用 `base.<domain>.<action>`, 参考格式：
    ``` yaml
    - id: base.db.migrate
    tier: low
    kind: script       
    script: scripts/db/migrate.py:main
    inputs: ...
    outputs: ...
    owner: ...
    tags: [db, migration]

    - id: base.db.query
    tier: low
    kind: mcp
    mcp_server: cloud-db
    mcp_tool: query
    inputs: ...
    outputs: ...
    owner: ...
    tags: [db, external]

    - id: base.db.call
    tier: low
    kind: api
    api_server: QWEN
    api_key: ...
    inputs: ...
    outputs: ...
    owner: ...
    tags: [llm, external]

    ```
    - high tier的命名方方式： `able.<domain>.<intent>`, 参考格式：
    ``` yaml
    - id: able.db.apply_all_migrations
    tier: high
    kind: workflow
    domain: db
    intent: apply_all_migrations
    steps:
        - base.db.migrate
        - base.db.check_health
    depends_on:
        - base.db.migrate
        - base.db.check_health
    owner: ...
    maturity: candidate

    - id: able.repo.maintenance_assistant
    tier: high
    kind: agent
    domain: repo
    intent: maintenance_assistant
    can_call:
        - base.db.query
        - base.db.migrate
        - base.fs.read_file
        - base.mcp.*         # 某些 mcp 能力
    owner: ...
    maturity: experimental
    ```    

-   **机制增强**：关于模块实例和编排节点的关系原设计未详述。涉及跨模块功能调用的流程十分模糊，经由讨论，功能理由需要做如下调整：
    -   两套注册体系（ability_registry和method_registry）都需要加上`owner_id`字段
    -   对于业务/编排去，一次调用至多对应一个high tier功能体，这些功能体有且只有一个`owner_id`，即便内部用到了别的模块能力，那也是它的`depends_on`，对外不暴露跨模块细节
    -   对于开发视角则更像选轮子，当在某个模块里写代码时，AI可以从能力路由里看到其他模块公开的可复用功能，然后决定是否使用轮子，这是实现细节，不是编排决策。
    -   可以在`ability_registry`中增加如下内容：
    ``` yaml
    id: func.order.create
    owner_module: mod.order.core
    exposure: public      # public = 面向业务的入口
    kind: workflow
    ...

    id: func.order.sync_user_snapshot
    owner_module: mod.order.core
    exposure: internal    # internal = 轮子 / 给其它功能体复用，不给业务编排直接打
    kind: workflow
    ...

    ```    
-   **机制增强**：原方案定义了触发器(trigger)和护栏(guardrail)机制，但二者和功能路由的关系未详述。经讨论，决定如下将trigger/guardrail纳入能力体系，便于编排器编做风险感知编排。规则如下：
    -   触发器和护栏不作为能力节点（不是ability，不被编排器直接调用），而是作为控制平面，与实际的能力节点双向引用
    -   在ability_registry.yaml中，增加`safety_profile`字段，引用触发器和护栏，示例如下：
    ``` yaml
    id: func.db.apply_all_migrations
    ...
    safety_profile:
        triggers:
            - db-migration
        guardrails:
            - require_approval_for_prod
        risk_level: high

    ```
    - 在 trigger/guardrail 配置里，增加`target`字段来反向引用能力，示例如下：
    ``` yaml
    # doc_agent/triggers/agent-triggers.yaml
    - id: db-migration
    level: P0
    enforcement: block
    match:
        kind: file_glob
        pattern: "db/engines/**/migrations/**"
    target:
        kind: function
        ids:
        - func.db.apply_all_migrations
        - func.db.apply_single_migration
    preload_docs:
        - doc_agent/AGENTS.md#db-migration-policy
    required_checks:
        - db_lint
        - db_migration_dry_run

    ```
    - 双向引用是为了保证“需要守护的能力，始终有完整的策略链路可以查”和“策略变更的影响可见”，不要求必定触发，相关字段可以为空。
 
-   **流程调整**：对于基础功能（现在是low tier）和功能体（high tier）的注册准入规则，原方案中没有明确标准。讨论认为，建立注册规则是必要的，可以防止能力体系无序膨胀：
    - 只要涉及具体流程或功能，不对low tier的注册做任何限制，但要明确阻止临时脚本的登记。
    - high tier增加状态字段`maturity`，明确进入编排器的准入门槛
        - `maturity`有两类状态：`candidate`、 `stable`
        - candidate允许AI和人调用，但不进入能力路由，也不对外公开能力
        - stable进入能力路由体系，可以有编排器调用；需保证registry条目完整、至少通过2个测试用例。
      - CI做硬性gate，只有maturity == stable的高层节点才允许写入ABILITY.md
      - 编排器只看准入后后的高层能力，用于实际任务编排和生产执行
      - 开发和维护中允许具有全量试图，可以看到所有注册的高层级和低层级的能力。
  
    
-   **机制增强**：原设计中注册新的功能体节点，但能力体系本身有多种交叉关系（tier/kind/maturity/module/depends_on/trigger/guardrail……），流程复杂而易错。经讨论，决定做一套注册工具，收窄接口，保证能力体系的稳健。该工具将先满足基本需求，后续再演进。
    -   底层功能注册工具：register_lowtier
        - 交互式/参数化创建 method_registry.yaml 条目，强制填完最低字段要求
        - 根据不同的kind类型(script、mcp、api)，强制给出相应的字段
        - 跑一遍 registry_lint 确认不会撞 ID 或破坏引用。
    -   高层功能注册工具：register_hightier
        - 根据参数生成ability_registry.yaml的新条目
        - 给出需要补的 doc/test 模板路径（但不一定生成内容）
        - 可以可选生成一个说明文档骨架（doc_agent/.../func-xxx.md）
        - 更新 ABILITY.md 的索引， 但maturity强制为`candidate`，禁止进入编排体系
    -   功能路由准入工具：promote_ability
        - 检查是否可以把某个ability从candidate提升至stable 
        - 检查 docs/test/trigger/guardrail 是否齐备；
        - 触发一次 dev_check 子集（比如只跑相关 tests + lints）；
        - 修改 maturity 字段；
        - 可选更新 ABILITY.md 上的状态标签。
    
    严禁使用其他途径进行注册，相关registry/YAML一律不可直接手改。
    
## 4. AGENTS.md 策略规范

-   **机制增强**：我们希望维护一个全局的必读/选读文档清单，帮助代码大模型理解工作流程，加速构建阅读策略。讨论后决定，对根目录下的AGENTS.md文档以及 doc_agent/AGENTS.md文档的职责进行明确
    - 根目录下的AGENTS.md的职责包括:
        - 定义全局边界，例如允许/不允许、审批点、危险操作流程、提交前置要求等。
        - 给出AI必读清单，给出项目级操作的AI选读清单（需要明确什么时候阅读）
        - 指明两套路由的入口位置（知识路由 ROUTING.md、功能体路由 ABILITY.md）。
        - 约定工作文档（workdocs）的最小写作与恢复规则。
        - 说明触发器/审批的基本原则与优先级。
    - doc_agent/AGENTS.md的角色定位为策略库地图（把各域/各模块的策略文档聚合成可检索的目录），职责包括:
        - 列出域策略清单（如 Guardrail、DocOps、DBOps…）与跳转锚点。
        - 给触发器和功能体提供“到哪一段策略文档”的稳定链接，避免把策略当作路由树的一部分。
        - 不要覆盖根级AGENTS.md的策略，不复述全局边界，不给出入口清单。
    - 必读清单：
        - 文档路由原则、渐进式拆分法
        - workdocs、上下文恢复、触发/护栏协作  
        - 模块目录规范、依赖/关系、跨模块提交策略  
    - 选读清单：
        - 测试规则（test-rules）：把“何时测、测到什么程度、谁来挡”说清楚，给触发器/CI 提供依据。
        - PR规则（pr-rule）：统一“提交粒度/跨模块/描述体例/检查项”，让编排器与 CI 有明确门槛。单一模块原则：一个 PR 只改一个模块目录；跨模块用“父 Issue/PR + 子 PR（各一）”；
        - Commit/Branch 规则：
            - Trunk‑based + 简化 Conventional Commits（推荐）
            - 分支：main + feature/* | fix/*；
            - 提交头：feat|fix|refactor|test|docs|chore(scope): summary；
            - REAKING CHANGE: 放到 footer（用于触发合约/版本警戒）。

-   **机制增强**：原方案中，关于模块实例目录下AGENTS.md文档的角色定位很模糊。经讨论，明确模块实例目录下的AGENTS文档（modules/<mod_id>/AGENTS.md）的相关定义：
    -   文档职责：模块实例的本地规则、边界、注意事项等；对AI而言，它就是模块实例对应策略内容的终点
    -   文档角色：属于文档路由体系的叶子节点，指向模块实例策略内容。
    -   其他说明：AGENTS.md本身不需要维护下级文档的路由，只需要做好模块内策略路由/索引的角色
        -   对于简单功能模块，只需说明模块功能/角色、风险点、工具使用等信息
        -   对于多场景的复杂功能模块，模块实例需要通过ROUTING.md/其他叶子文档（例如policy.md）等触及下级包含具体内容的文档。
        -   模块级AGENTS.md的增删改遵循ROUTING体系操作规范
    -   注意，根节点的AGENTS.md 以及 doc_agent/AGENTS.md不是叶子节点！

  
## 5. 模块化思想

-   **命名规范**：原设计在5.2.1“初始化注册流程”中将模块初始化产出概括为同步“八大文档”。讨论认为该表述已陈旧且不精准。现改为直接罗列模块初始化需生成的具体内容，并强调仅允许通过脚手架完成模块初始化。

-   **机制增强**：经由讨论，现对模块实例的初始化
    -   经由脚手架生成后的骨架包括：
        -   模块目录`modules/<mod_id>/`；`mod_id` 规则：`mod_<domain>_<name>`  （格式需要在lint中约束）
        -   模块根级清单文档，初始化完成后脚手架需要立即写入关键信息
            -  `modules/<mod_id>/MANIFEST.yaml` （具体需要在spec约束）
            -  `modules/<mod_id>/AGENTS.md`
            -  `modules/<mod_id>/ABILITY.md`
            -  `modules/<mod_id>/ROUTING.md`（如果模块功能简单，这不需要）
        -   子级目录，初始化仅需要根据规范创建目录和文档
            -   上下文目录：`workdocs/`
            -   知识文档目录：`doc/` 
            -   配置文件：`config/`
            -   代码目录：`frontend/`、`backend/`、`core/`等
    -   初始化规范
        -   规范文档建议路径`doc_agent/modules/init-rules.md`，脚手架建议路径`scripts/scaffold-module/`;
        -   创建``modules/<mod_id>/init/`目录，存放使用说明、快速引导、问卷、生成的中间文件；初始化完成后整体删除目录。
        -   创建过程仍然为通过交流收集必要信息，然后调用脚手架实现
    -   初始化完成后，脚手架会将进行模块实例注册
        -   需要校验注册的一致性、路由可达、命名规范、占位用例可运行
        -   模块实例注册：`doc_agent/modules/instance_registry.yaml`
        -   模块关系图：`doc_agent/modules/type_graph.yaml`
    -   初始化终点定义：
        -   只注册+就绪文档，目录/注册/路由/策略/触发/占位用例齐备
        -   不强制**交付任何业务功能代码。
  
-   **概念共识**：原方案中提到了和模块相关的三个概念：层级、类型、实例，但这些概念的定位和承担的职责并未明确。经讨论达成共识：
    -   doc_agent 和 doc_human 需要维护模块概念的解释
        -   面向AI的模块化思路说明要保持精简，说明三个概念的角色和职责即可；文档路径建议为`doc_agent/guides/module.md`
        -   面向人类的说明可以更加具体，保持内容的结构化以符合阅读习惯；文档路径建议为`doc_human/guide/module.md`
    -   模块有 module_level（层级）、module_type（类型）、module_id（实例）
    -   模块层级是模块实例的组成/包含关系，构成从属关系树，层级和类型是解耦的
        -   每个实例有 module_level 和可选 parent_module
        -   层级树用来表达“谁属于谁”的业务分解，不干预类型/能力注册
    -   模块类型是用于关系维护的，只在脚手架/文档里用，不在能力路由和编排决策等执行路径中当主语
        -   在doc_agent/module-types/TYPE_GRAPH.yaml中维护
        -   模块类型可以组成图结构，该结构用来描述业务流和数据流（类型间的调用/依赖关系），同类型模块的链接接口相同
        -   类型图是设计层 SSOT，但只当蓝图，不直接用来调度
        -   帮AI理解“整个系统大概怎么分块、怎么串联”，在新实例初始化中给MANIFEST做校验
    -   模块实例是面向业务的，是实际开发（运行、编排、CI、Guardrail等）的唯一载体
        -   是注册、路由、编排、测试、Guardrail 的唯一一等公民，
        -   所有注册的功能（高层和底层）最终都是挂在具体的模块实例下
        -   workdocs、AGENTS、ROUTING、MANIFEST 也都按实例维护
        -   每个实例在 MANIFEST.yaml 声明 module_type 与 requires_modules
        -   CI 检查：实例依赖必须是类型图“允许的边”的具体展开

 
-   **机制增强**：讨论强调模块实例与全局项目的衔接是模块化架构中最需要关注的问题。跨模块功能开发的规范在第3章**功能体路由体系**中进行了说明，请参考关于模块实例和编排节点的关系。这里增加硬性规定
    -   每个ability节点有唯一的`owner_id`，业务编排入口只用 `exposure = public`
    -   跨模块的“轮子调用”只体现在 depends_on 里，不改变“业务入口在单模块”的约束
    -   module_type 只在脚手架/文档里用，不在能力路由和编排决策里当主角
    
-   **流程调整**：原设计提到维护模块关系图的双版本（AI读/人读）。经讨论决定简化为单一来源：仅维护供AI使用的模块关系图作为SSOT，不再另外维护人类版本。如有面向人类的可视化需求，可通过脚本从SSOT导出生成，无需实时双份更新。目前由于暂不需要人读关系图，相关需求可以延后开发。
    
-   **流程调整**：针对模块删除等变更风险，原方案未单列说明。结合讨论共识，将在Guardrail策略中增加移除某模块实例约束：
    -  必须先处理完该模块包含的功能和文档，单个功能和文档的删除遵循一般删除原则
    -  必须检验全局对模块实例的相关依赖，对于无法确认删除后果的，需要认为确认后再执行删除
    -  其他类型的大规模改动（如批量重构），必须引导AI一次只聚焦单一模块或功能的改动
    -  触发器中设置  `enforcement: block`  等规则强制实现该约束，避免AI进行超出管控范围的批量修改
        
## 6. 数据库联动

-   **共识确认**：原方案第6章详细制定了数据库模式管理、迁移/回滚流程、CRUD智能体闭环等机制。讨论中并未对数据库部分提出新的修改要求，表明对原有设计基本予以认可，同时也进一步明确：
    -   数据库相关操作仅允许调用注册的能力（ability、method）来完成
    -   通过配置文档和触发机制来控制具体的实现方式和作用对象，例如是对云数据库进行操作、还是使用本地数据库
    -   同步操作记录时，要与操作对象关联（要区分数据库，维护表结构）
    
-   **共识确认**：由于数据库操作具有高风险，原方案在触发器配置中对数据库相关触发设定了严格等级。讨论再次强调：
    -   数据库操作需要明确相关的Guardrial和trigger
    -   确保数据库联动环节与安全机制一致，保证数据SSOT的同步更新
    -   高敏感操作必须经人工确认，必须闭环文档同步。
    
## 7. API 联动管理

-   **共识确认**：原方案第7章建立了从接口合同到网关编排的API生命周期闭环，讨论中未提出结构或机制调整，仅做出强调
    -   关注接口的合同管理、OpenAPI/类型到处、网关模块的agent策略
    -   API接口要正确登记，trigger 和 Guardrail要确保合同校验、类型同步等API相关功能的正确性。
    
-   **共识确认**：原方案考虑了接口变更的安全审批机制。讨论重申了接口变更需要严格受控的原则，支持原有设计中关于OpenAPI更新、接口合同修改必须经过Guardrail智能体检查并人工确认的要求。
    
## 8. 自动化链路

-   **机制增强**：原设计未详述如何处理多触发器同时命中的情况。讨论决定：
    -   引入触发器优先级机制：为每个触发器配置显式优先级，并在运行时按优先级顺序评估触发
    -   出现多个触发器并发命中时，除按优先级执行最高优先的动作外，需要记录所有命中情况
    -   参考规范：`trigger_id` 命名空间（domain.scope.action）、优先级（P0/P1…）、互斥组、`enforcement`（observe/warn/block）、`preload_docs`、`required_checks`。 
    
-   **共识确认**：原方案已经规划了自动化检查命令和CI集成。讨论确认：
    -   CI定时任务和检查在模板中是必要且可行的。实现时会限定CI脚本的检查范围在引导文档规定的重点内容上，以控制复杂度。
    -   在持续集成环境下定期运行关键链路的健康检查，确保自动化链路长期可靠运转。
    
-   **机制增强**：针对代码大模型的`PR Review`功能，讨论提出可将其并入自动化链路体系的想法：
    -   增添模板特定的检验要求，并将其作为AI Review的一部分。
    -   检验要求可以加入ROUTING体系或直接作为AGENTS.md的必读文档
    -   建议尚处构思阶段，需进一步探讨收益情况和讨论如何融入路由/触发器机制，
    
## 9. 脚本规范

-   **共识确认**：原方案第9章规定了`scripts/`目录下脚本的组织、命名、头部注释和能力登记流程。讨论没有提出对此部分的异议，但由于能力路由体系的修改，脚本规范的设计也发生了部分变动：任何非临时的脚本，都需要通过注册脚手架进行注册。
    
## 10. 交互式项目开发

-   **机制增强**：针对长流程的上下文恢复/清理问题，原方案虽有workdocs归档机制但未详细规定清理策略。讨论决定引入动态上下文管理机制：
    -   采用“滑动窗口+摘要”的方式定期清理 workdocs/active/ 中的文档的内容
    -   在写入workdoc时记录时间戳和操作计数
    -   根据预设阈值（如总行数超过120行、更新超过5次、内容超过30天）等触发清理
    -   清理过程先将过长/过旧内容自动生成摘要备份，转入archive/summary/
    -   此机制将通过CI定时任务执行而非由智能体实时触发，增强可控可预测性，要避免AI自身清理影响当前上下文
    
-   **机制增强**：讨论决定在工作文档中记录任务状态（pending/in_progress/completed），并要求代码大模型在流程中实时更新，在相关workdoc条目中维护每个任务的最新状态以提高任务可见性。
  
-   **流程调整**：原方案里第8章和第11章给出了一些检查项和操作清单的示例表格（checklist）。讨论认为：
    -   这些引导文档中的checklist属示意性质，在实际模板中需要重新编写以确保流程鲁棒
    -   要根据模板实施细节制定精确的检查清单，涵盖各阶段必要步骤和验收标准，并放入相应文档
    -   新清单需要聚焦模板本身，不引入项目特定要求，以避免超出模板适用范围的内容
    -   讨论并未给出明确的checklist方案，需要在撰写计划方案前明确   

-   **机制增强**：讨论中提到了"自动整合经验教训"的创意，即Automate Lesson Integration：
    -   在自动化链路中将加入机制，使AI从工作文档中的错误记录或经验总结中自动提取教训，并更新相关策略或提示
    -   该功能将尽量与现有trigger/Guardrail体系解耦实现，尽量不干扰其他模块。
    -   这样做的原因是希望模板具有自我改进能力：随着使用推进，AI能逐步积累避免重复错误的知识，提高闭环开发效率和质量。
    -   想法尚处构思阶段，需要进一步探讨实现方案
-   **机制增强**：讨论决定， trigger/guardrail都是一等公民，需要有自己的配置和lint
    -   trigger和guardrail都需要有自己的registry（triggers.yaml / guardrails.yaml）
    -   trigger和guardrail都被视为“配置即代码”，需要纳入评审和归回套件
    -   配套相关工具
        -   生成器：从模板生成trigger或guardrail，升级或调整trigger/guardrail
        -   冲突检验：检查互斥、循环、优先级等。
    -   规范文档，内容包括：命名空间、优先级、互斥/并行规则、冲突检测、回退策略、审计字段（谁触发/为何阻断）

## 11. 测试与验证

-   **共识确认**：原方案第11章规划了三层测试：文档/路由一致性校验、能力契约测试、自动化链路演练，以及提交PR的流程要求。讨论中并未推翻这些内容，双方同意按照原方案执行测试验收。这意味着实现中将确保：  
    1. 结构一致性校验：定期运行工具检查所有  `ROUTING.md`、`AGENTS.md`、`ABILITY.md` 结构正确、无断链，frontmatter完整。  
    2. 能力契约测试：对每个能力域执行脚本，验证智能体输入输出与注册信息的一致。  
    3. 自动化链路演练：通过 Dry-run 模式下运行触发器检查和上下文追踪，确保没有执行路径偏离预期。  
        
-   **共识确认**：对于超出模板核心范围的功能，如多语言支持、成本监控、用户反馈循环，原方案未作涉及，仅在11.4节提到暂不纳入成本预算和生产部署。讨论结论与此一致：为避免增加系统复杂性，这些扩展特性不纳入当前模板构建范围，。
    

## 12. 模板初始化规范（全仓级）

-   **机制增强**：讨论决定新增项目初始化脚手架，即当需要将模板应用到实际项目时，通过专门流程来收集项目信息并对模板仓库进行定制化。流程包括：
    -   与用户交互填写临时需求文档或上传初始需求
    -   逐步完善并确认需求后，由脚手架脚本将模板转换成项目仓库（修改配置、替换占位内容等）
    -   这相当于模板在实际项目上的第一次落地操作，可视为全仓级的初始化，与模块级初始化相对应
    -   项目初始化的终点定义为“只注册+就绪文档”与校验通过
    -   需要进一步明确，项目初始化过程需要收集哪些信息，完成后需要改动/新增哪些文档，要把初始化后手工调整量降到最低。
        
-   **路径结构**：讨论决定在模板中添加一个专用的初始化目录来承载项目初始化所需资料
    -   创建  `/init/`，包含模板使用说明、快速开始指引、初始化流程的规范说明以及交互脚本/配置文件等。
    -   将模板初始化为项目时，可以按此目录下的指引逐步完成
    -   初始化完成后，会将该目录及其中文件整体从项目仓库中移除
    -   repo模板本身应该包含绝大部分通用文档和通用功能，以确保初始化的起点质量
    

## 13. 后续演进
后续演进可以放到README中，不要出现在任何AI阅读的文档里

### 规范增强
- 代码风格与 Lint 约束：以工具配置为准，规范只声明“不得在同一 PR 大规模格式化”。
- 依赖与供应链：版本锁定策略、许可白名单、依赖审计触发点。
- 安全与密钥：禁止明文、私密配置位、泄露扫描要求。
- 成本/配额护栏（AI 调用）：成本上限、超限降级策略（可晚点启用）。
- 数据/隐私最小集：PII 不落盘、匿名化占位（若项目需要再开）。


### 交互增强
- 提供工具，整理上层功能和下层功能的包含关系
- 提供工具用于度量和测试trigger/guardrail：命中频率、平均阻断时长、全量模拟的干跑测试
- 文件或功能的新增/调整产生的交叉影响：影响哪些注册与文档（精确到文件/字段），需提供机器可校验的完成清单。 
  
### 增强型需求
- 为不同规模项目提供精简版模板以减少小型项目负担，待模板全貌完成并验证有效后，可进一步提炼出单模块轻量版用于小型项目
- 语言支持
- 成本监控
- 用户反馈循环

