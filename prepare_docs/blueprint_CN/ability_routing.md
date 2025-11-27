
# 能力路由

本质上，能力路由是AI的工具箱和工具如何组合的映射。我们希望AI的规划尽可能保持在抽象级别上运行，而不是被底层细节淹没。项目通过维护一个双层能力系统，用来保持编排的可管理型。
- 低层能力：细粒度的操作，如运行脚本或调用API。通常代表原子操作，允许由多个实现。
- 高层能力：由低层步骤组成的高级任务或工作流，会优先暴露给编排器进行规划。

通过分离低级和高级，我们实现了模块化——低级方法可以交换或重用，高级工作流封装业务逻辑。由于能力路由体系的入口和作用对象都是模块实例，为更好的理解其作为基础设施的的功能和使用方式，强烈建议阅读<modulate.md>文档中的4.2章节"能力路由的协同"。

低层能力和高层能力都需要进行注册（模板会提供能力相关的注册和修改脚手架，以保证格式的规范）。能力的注册信息会存入统一的文件路径（区分低层和高层），能力实现也会统一存放。
每个模块实例通畅会维护自身的高层能力路由，项目会额外维护一份路由作为入口，这些路由文档命名统一为`ABILITY.md`。所有的`ABILITY.md`只提供文件路径和使用情况说明。注册机制和所有的`ABILITY.md`文档共同组成了能力路由。


在阅读后续内容前，强烈建议阅读并理解下述名词或概念：


- low tier 包含"script"、"mcp"、"api"三种类型。前者为repo内部实现的脚本，后两者为mcp/API等外部服务  
    - high tier 包含"workflow" 和 "agent" 两种类型。前者是更偏确定性的pipeline ，主要拼装repo内脚本；后者更偏动态驱动，会综合内外部能力
    - 只有high tier会进入功能路由体系和接入编排器，trigger和guardrail只对这些高层入口做策略管控。不要让编排器看到low tier节点
    - high tier 和low tier都需要注册，保持两本表，注册中yaml需要明确tier 和 kind
    - low tier 能力保持可替换，high tier住关心可以调用的low tier 的主键id， 具体调用交给配置/著发起



---

## 低层能力

### 类型
低层能力是细粒度的、通常是特定于实现的操作，包括仓库内部脚本、外部服务调用、数据库操作等。我们按类型对低级节点进行分类，script、mcp、api：
- script：项目内部实现的脚本
- mcp：经过封装的外部服务，外部微服务或云提供者调用，例如云数据库的写入
- api：定制需求的外部服务，外部API调用，例如调用大模型来实现某个功能


### 相关文件



低层能力需要维护两类文件：文档和具体实现
- 文档：
- 实现：

### 使用方法

低级能力旨在成为可互换的构建块，

- 手动调用脚本：将直接调用具体的实现，而不是  ！！！！！！！！
- 要求大模型执行：将 ！！！！！！！！！！！！！


---

## 高层能力


### 类型
高层能力通常是编排的工作流或代理，结合了多个步骤或决策。高级节点表示编排器将调度和管理的任务，是语义层面的能力，包含两种类型：workflow和agent。
高层能力包含两种类型：`workflow`和`agent`。
- workflow：更偏确定的pipeline，通常按顺序调用多个脚本
- agent：更偏动态驱动，可以动态调用各种低级方法，通常由AI驱动的决策过程

### 相关文件
- 全局 ABILITY.md，记录高级能力（带有简要描述和对它们的注册表定义或文档的引用）。为了一致性，它将反映 YAML 内容（我们可能从注册表自动生成其中的部分）。此文档将验证行数和完整性。 ！！！！！！！！！！！！！！！！！！

### 使用方法
这些作为编排器的实际可执行单元出现，而低级方法在这些高级过程内部调用。

高层能力将被优先暴露给AI，这些经过封装的业务流将简化AI的策略执行，提升输出的可预见性和规范性。


- 任务编排

- hooks集成
  - 我们在能力条目上添加 safety_profile，以引用与该能力相关的任何触发器或防护栏 ID。例如，像 func.db.apply_all_migrations 这样的能力可以列出它与触发器 db-migration 和防护栏策略 require_approval_for_prod 相关联，总体风险级别为 high。相反，在触发器定义中，我们将包含一个 target 字段，指向相关的能力 ID。这种双向链接确保可追溯性：如果某个能力是高风险，我们可以快速找到保护它的触发器/批准，如果触发器触发，系统知道它正在保护哪个能力（或模块）。尽管触发器和防护栏将在稍后完全实现，我们现在将定义这些链接的数据模式，并在注册表中包含几个初始示例用于测试

---

## 编排过程

### 模块实例的功能开发
  1. 阅读模块实例目录下的`ABILITY.md`；
  2. 查看`ABILITY.md`允许使用的高层能力以及**应用情景**说明；
     - 如发现有可以使用的高层能力，依据路由跳转至能力注册文档，检验是否要求、输入、输出等是否满足条件。如满足条件直接调用，如不满足退回`ABILITY.md`继续查询（高层能力有`maturity`字段，优先调用字段为`stable`的能力）
     - 如没有可以使用的高层能力，进入步骤三
  3. 检索`ABILITY.md`包含的低层能力以及**功能描述**，决定是否调用
     - 如由可以调用的
  4. 将选择能力的决策过程记录到工作上下文（`workdocs/active/<task>/details.md`）中
     - 如调用了高层能力，记录调用原因、能力`name`和`maturity`，以及执行结果
     - 如调用了低层能力，记录调用原因、能力`name`、以及执行结果。 

### 多个模块实例的联调

---

## 注册和维护

我们先带入项目开发的视角，梳理不同类型能力的生命周期



注册表工具和模式验证旨在缓解在复杂的引用网络（ID、依赖、触发器、所有者等）中可能出现的错误。在设计讨论期间，团队确定管理不善这些关系可能导致编排错误或安全漏洞，因此强调统一配置模式和这些关系的自动化。严格的注册表还意味着 AI 和开发人员可以查询单一来源（YAML 或 ABILITY.md）以发现可用能力及其用法，提高透明度和重用性。

- 低层能力的典型产生过程：
  1. 1
  2. 2
  3. 3


- 面向AI的抽象高层能力进入能力路由的典型过程：
  1. 1
  2. 2
  3. 3



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

   

不论是是哪个层级哪种类型的能力，进入能力路由时都需要进行注册。注册表工具和模式验证旨在缓解在复杂的引用网络（ID、依赖、触发器、所有者等）中可能出现的错误

我们将维护两个基于 YAML 的注册表文件来列出和定义这些节点：
  - **低级能力的方法注册表**（建议文件：doc_agent/orchestration/method_registry.yaml）。每个条目包含一个唯一 ID，格式为 base.<domain>.<action>（例如，base.db.migrate 用于数据库迁移脚本）。条目将详细说明如何执行它，例如，如果类型是 script，提供脚本路径和函数；如果类型是 mcp，提供目标服务和操作；如果类型是 api，提供外部 API 端点或密钥等。我们还列出输入/输出模式引用、所有者（负责的维护者或模块）以及用于搜索的标签。
  - **高级能力的能力注册表**（建议文件：doc_agent/orchestration/ability_registry.yaml）。每个条目有一个格式为 able.<domain>.<intent> 的 ID（例如，able.db.apply_all_migrations 用于应用迁移的工作流，或 able.repo.maintenance_assistant 用于帮助仓库维护的代理）。字段包括 tier（high）、kind（workflow 或 agent），以及详细信息，如步骤（对于工作流，要调用的低级 ID 序列，包含依赖关系）或它可以调用的能力（对于代理类型，它可能调用的低级 ID 白名单）。条目还将有一个 owner_module（链接到负责的模块）以及可能的 domain 和 intent 字段以澄清目的。


- **设置注册规则和成熟度级别**：为了防止能力集不受控制的增长，我们引入了添加新能力的治理：
  - **低级（方法）注册**：低级方法相对不受约束——任何需要的脚本或调用都可以添加到方法注册表中。但是，不允许注册临时或临时脚本。所有方法必须旨在作为可重用或永久能力（临时的一次性脚本应保留在 temp 目录中，不在注册表中）。
  - **高级（能力）注册**：我们为高级能力实现一个成熟度字段。新能力可以从 candidate 开始（意味着它们是实验性的或正在开发中）。候选能力可供 AI 和开发人员查看，并可能在测试中使用，但它们不会包含在官方能力索引中，也不会在生产中主动编排。一旦能力得到验证（例如，至少有 2 个测试用例、文档、稳定的接口），它可以标记为 stable。只有稳定的能力才能完全集成：它们出现在 ABILITY.md 中，并由编排器考虑用于实时任务执行。我们的 CI 管道将执行此规则：如果能力不稳定，它不应列在正式索引中或在生产工作流中调用。编排器在规划时将过滤掉非稳定能力。这提供了一个安全网：实验性功能在被验证之前不会意外触发关键操作。
- **开发能力注册工具**：为了简化添加新能力，我们计划构建交互式脚本。例如，register_lowtier.py 脚本指导开发人员（或 AI）添加新的低级方法条目。它将提示所有必填字段（强制要求没有缺失），并根据类型进行验证（例如，如果类型是 script，确保提供了脚本路径和入口点函数；如果类型是 api，需要 API 端点和凭据等）。它还将运行 lint（例如检查重复的 ID 或断开的引用）。类似地，我们可能会为高级节点创建 register_ability.py，包括将初始成熟度设置为 candidate 并确保存在所有者和至少一个存根文档/策略的逻辑。这些工具通过将所有更改通过受控路径进行，有助于维护单一事实来源原则。


 doc_kind: ability_index
