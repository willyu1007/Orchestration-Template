# 模块化开发

我们认为通过限制任何单个操作的范围可以提高AI上下文管理效率。为使人类开发者和AI代理都能在清晰的边界内工作，我们将模块化开发作为基础原则，鼓励将仓库被结构化为定义良好、松散耦合的模块。

我们围绕模块化开发设计了整套的相关规范，用来以模块为单位管理基础设施（如知识/功能路由等）。也就是说，模块是构成项目的基础单元，同时也是其他主要基础体系的唯一载体，需要自行维护独立的配置、策略、工作记录、和代码实现。并维护自身允许使用的功能路由和知识路由。

和模块相关的概念主要有三个： module_level（层级）、module_type（类型）、module_entity（实例）
-  **模块实例**是面向业务的，是实际开发的唯一载体，前端/后端/逻辑的实现都需要对应到模块实例：
   -  `modules/` 目录中的具体模块
   -  AI工作过程的记录按实例维护，即`workdocs\`
   -  所有能力和代码都在模块实例的上下文中运行
   -  实例需要明确自身的策略、知识、路由等，以保证基础体系的可运行
-  **模块类型**是用来描述业务流和数据流，类型间的调用/依赖关系，组成图结构：
   -  模块的抽象分类，是具有执行一致性的运行模式
   -  模版将维护类型的串联关系，是系统运行和数据流转的蓝图
   -  实例可以声明其他模块的依赖，但这种依赖关系受到类型关系的约束
-  **模块级别**和类型是解耦的，层级描述实例的组成/包含关系，构成关系树：
   -  模块实例可以组织在父子树中，每个模块实例包含`module_level`，并可以指向 `parent_module`
   -  层次结构可以反应业务分解，是开发需求的逻辑梳理的产物
   -  关系树用于人类理解系统结构、合理化文档分组，但系统不会专门维护这种结构

---

## 1. 骨架结构及其用途

所有模块实例都有专属的目录，例如"/modules/<*mod_id>/"，本章节中，我们将使用**实例目录**来表示这个实例目录。

实例目录的典型骨架结构包括：
- 模块根目录中的核心AI文档：
  - `AGENTS.md`：实例的本地策略，用于定义AI开发时需要遵循的规范、需要了解的背景等，规则向上覆盖。
  - `ABILITY.md`：实例可以直接或间接调用的封装能力，用于告知AI可用工具，从而提高代码的规范程度和可控程度。
  - `ROUTING.md`：实例相关的顶层知识路由，给出实例开发过程中各情景下任务编排可能需要的路由，最终指向知识文档。
  - `MANIFEST.yaml`：实例的清单文件，捕获模块的关键元数据（其 module_type、对其他模块的任何依赖等）。
- 模块内容的标准子目录，例如：
  - `routes/`：承接顶层知识路由的具体路由，将根据不同情景提供不同的知识文档路径，详见<content_routing.md>。
  - `outcomes/`：开发过程中的产出物
  - `doc/`：用于额外的模块特定文档（指南、合同、模块特定的规范、人类阅读的文档等）。
  - `workdocs/`：用于模块特定的进行中笔记（尽管主要 workdocs 通常在仓库级别，模块特定任务可以在此处跟踪（如果需要））。
  - `config/`：用于模块配置文件。
  - 代码目录，可能包括 `backend/`、`frontend/`、`core/` 等，具体取决于技术（确切结构将遵循约定，例如Web模块可能有 `api/`或`service/` 目录）。

```
module_instance/
├── AGENTS.md              # 策略文档
├── ROUNTING.md            # 顶层路由文档
├── ABILIIY.md             # 能力路由文档
├── workdocs/              # AI工作上下文目录
├── routes/                # 实际路由文档
├── config/                # 配置文件   
├── {code}/                # 可能包括 `backend/`、`frontend/`、`core/
├── interact/              # 实例的锲约、依赖、和影响（例如可能会改变数据库中某张表）     
├── docs/                  # 面向人类的文档（README、评估、observability等）       
└── MANIFEST.yaml          # 模块的关键元数据
    
```

### 代码目录
具体包含哪些目录取决于技术，确切结构将遵循约定。例如Web模块可能有 `api/`或`service/` 目录

### interact
- 数据库：需要哪些数据库中的数据（表、主键、字段），是否有写入、修改等操作
- 契约文件： scheme / contract
- mock、 fixture 或其它数据
- 其他

### workdocs
每个模块实例都需要维护工作记录，用以记录AI在执行任务过程中的所思所想和所作所为。骨架结构形如
``` 
workdocs/
├── AGENTS.md              # This file
├── active/                # Current work
│   ├── task-1/
│   │   ├── task-1-plan.md      # 完成任务的计划
│   │   ├── task-1-context.md   # 记录工作上下文
│   │   └── task-1-task.md      # 任务完成情况和进度   
│   └── task-2/
│       └── ...
├── outcome/               # 工作产出
│   ├── lessons.md         # 错误经验，避免再犯
│   ├── decisions.md       # 重要决策
│   └── scripts/           # 开发过程中新增的脚本
└── archive/               # Completed work (optional)
    ├── summary/           # 陈旧内容
    ├── old_tasks/         # 已完成的任务
        └── ...
```

---

## 2. 维护

### 2.1 实例注册

模板将实现一个模块初始化的脚本，通过脚手架和注册表来维护模块生态系统的一致性。典型流程如下：
  1. 创建目录：创建`modules/<mod_id>/init/`，用于存放过程文档（如需求文档、必要信息记录、特殊要求等）以及充当创建期间的暂存区。该目录为临时目录，将在初始化流程结束后删除；
  1. 交互式信息收集：AI（或开发人员）将提供或被要求提供必要的详细信息，如模块领域、名称、类型和任何初始配置（如是否连接到数据库等）。
  2. 生成模块骨架：创建目录 modules/<mod_id>/，使用上面列出的文件填充它（使用提供的信息填充 YAML 前置元数据或清单中的占位符，例如，模块名称、类型、日期、所有者）。
  3. 注册模块实例：向模块注册表添加一个条目，结合前序步骤收集的信息和格式规范填写内容。
  4. 类型关系维护：如果模块对应的类型为新类型，则需要在
  5. 如果模块预期具有公共能力，脚手架可能还会在全局能力注册表中插入一个骨架条目（初始成熟度为 candidate，稍后由开发人员/AI 完善）。
  6. 生成后，脚手架可以运行快速一致性检查：例如，以验证新模块的文档已链接（新模块的 ROUTING 包含在父路由中等）、命名正确，甚至可以执行占位符测试（如空测试通过）的模式运行 make dev_check。只有当这些都通过时，模块脚手架才被视为完成。
  7. 确认后，脚手架可以提示删除 init/ 文件夹并将模块标记为就绪。

通过上述流程，我们可以完成：
  - 新模块实例的注册：
    - 我们不需要额外维护或注册模块层级关系，仅要求模块实例在注册过程中填写`parent_module`和`module_level`字段。
  - 新类型注册：如果模块实例属于新的类型，即项目中还没有注册过相同类型的实例，我们需要进行模块类型的注册


初始化过程提供的是一个完全链接的模块，保证模块实例可以嵌入其他主要的体系，但初始化仅要求骨架结构完整，允许真实逻辑为空，且不会提前编写任何业务逻辑。也就是说，基础设施的实际应用需要随着开发深入逐步完善。

它提供了代码和能力所有权的清晰性，并使用自动化（脚手架、注册表）来维护一致的模块生态系统。这不仅有助于并行开发，还意味着 AI 不会意外地纠缠上下文——它知道哪些文档和能力属于哪个模块，并尊重这些边界。

**MANIFEST.yaml**

### 2.2 AI运行上下文

作为项目开发的主力，AI的工作过程必须是可持续和可回溯的。在丢失上下文的情况下，AI需要可以快速掌握所需信息，而不是从头再来一遍。

我们希望维护一套机制，帮助帮助AI完成思考->决策->执行->记录->回顾->思考的工作闭环。这是`workdocs`目录的设计初衷和核心作用：让AI通过阅读自己写入的信息，了解任务进度、避免重复犯错、回溯思考过程、以及查看运行效果。

我们先对`workdocs/`目录下的几个一目了然的文档和目录进行简要说明：
- `/AGENTS.md`：面向AI的使用手册。文档中将包含上下文的更新规范、使用建议、阅读顺序、写入要求等内容，这里我们仅给出要点概述
- `/archive/old_tasks/`：已完成任务归档后将存放在该目录下
- `/archive/summary/`：对于开发流程很长的任务，上下文中的陈旧信息将按照一定规则，总结后放入该目录。根据总行数、更新次数、时间等条件触发清理。

前文提到我们需要帮助AI完成工作闭环，一个典型的工作流程包括：
- 开始任务
  1. 检查是否和当前其他任务重复，确认无重复后在`active/`目录下创建任务目录和5个文档
  2. 根据需求，AI结合代码库深入思考，并将执行计划和阶段性需求写入`plan.md`和`task.md`;
  3. 检查计划是否合理，有没有考虑不周全的地方 
- 实施期间
  1. 参考`plan.md`了解整体策略
  2. 根据要求，定期更新`context.md`、`decisions.md`、`lessons.md`文档
  3. 根据检验标准，在`task.md`中勾选已完成的阶段/指标
- 上下文重置后
  1. AI重新读取`plan.md`、`context.md`、`task.md`
  2. 快速了解完整的任务执行状态，根据需求读取`decisions.md`、`lessons.md`
  3. 从上次中断的地方继续运行

---

**进行中的任务：workdocs/active**

接下来，我们将重点介绍  `workdocs/active/[task_name]/`目录。目录中包含三个主要的文档:`plan.md`、`context.md`、以及`task.md`，这些文档使用的基本原则如下
- **when to skip**: 没有难度的执行，如简单bug修复、细节修改、快速更新等
- **when to use**: 当工作可能跨session时，如当完成了一个复杂任务、完整的功能、重大重构时
- **update frequency**： 每次完成一个里程碑节点时，总结工作过程并更新文档
- **keep plan current**：每当任务范围发生变化，需要更新计划、添加新的阶段、记录原因。
- **make tasks actionable**：包含明确的文件名、清晰的验收标准、与其他任务的相关依赖等，eg: "Implement JWT token validation in AuthMiddleware.ts (Acceptance: Tokens validated, errors to Sentry)"

#### **plan.md**

AI需要在执行复杂任务前，将完成任务的思路和计划写入本文档，起到两个方面的作用：
  - 可以进行人工审核，决定是否需要调整计划
  - AI可以基于制定的计划，更新实施结果
建议写入文档的内容包括：执行摘要、现状分析、未来状态规划、实施阶段、包含验收标准的详细任务清单、风险评估、成功指标。当执行范围发生变化或进入/发现新的阶段时，AI应该及时更新本文档。

**Example:**
```markdown
# Feature Name - Implementation Plan
## Executive Summary
What we're building and why
## Current State
Where we are now
## Implementation Phases
### Phase 1: Infrastructure (2 hours)
- Task 1.1: Set up database schema
  - Acceptance: Schema compiles, relationships correct
- Task 1.2: Create service structure
  - Acceptance: All directories created
### Phase 2: Core Functionality (3 hours)
...
```
---

#### **context.md**

AI需要将恢复上下文的会用的关键信息写入文档，内容主要包括：任务进度、已完成与进行中的内容、关键文件及其用途、相关文件链接、以及快速恢复说明。在做出重大决策、完成关键节点、或重要发现后，需要更新此文档。鼓励高频率的更新此文档，特别是每次完成重要工作时，一定要更新“任务进度”部分！

**Example:**
```markdown
# Feature Name - Context
## SESSION PROGRESS (2025-10-29)
### COMPLETED
- Database schema created (User, Post, Comment models)
- PostController implemented with BaseController pattern
- Sentry integration working
### IN PROGRESS
- Creating PostService with business logic
- File: src/services/postService.ts
### BLOCKERS
- Need to decide on caching strategy
## Key Files
src/controllers/PostController.ts
- Extends BaseController
- Handles HTTP requests for posts
src/services/postService.ts (IN PROGRESS)
- Business logic for post operations
- Next: Add caching

## Quick Resume
To continue:
1. Read this file
2. Continue implementing PostService.createPost()
3. See tasks file for remaining work
```
---

#### **tasks.md**

本文档用于概述任务要求、以及通过清单管理的任务开发进度。

**Example:**
```markdown
# Feature Name - 
## Task Overview
- description: 
- current status:
- notice:

## Task Checklist
### Phase 1: Setup 
- [x] Create database schema
- [x] Set up controllers
- [x] Configure Sentry

### Phase 2: Implementation 
- [x] Create PostController
- [ ] Create PostService (IN PROGRESS)
- [ ] Create PostRepository
...

```

---

**工作产出：workdocs/outcome**

相对于`active`目录外，`outcome`是上下文系统中另一个重要的目录。两者第一个显著的区别是：前者的作用域是单个任务，而后者是模块实例中的所有任务。另一个深层次的区别在于，`active`是面向AI工作过程的，核心用途为上下文恢复；而`outcome`即是产出说明，也是完善项目基础建设（如知识路由、能力路由等）的源头。

`outcome`目录包含两个文档，`decisions.md`、`lessons.md`，以及一个目录`scripts/`，分别对应知识/经验产物和功能/脚本产物
- 知识/经验：通过总结错误和跟错决策经验，项目可以不断积累知识。结合知识路由体系，AI在后续工作过程中将得到更准确的指引。
- 功能/脚本：实例功能逐渐完善的过程中，最主要的可复用的产出物是函数或脚本。这些脚本是功能路由（项目的重要基础设施，相当于AI的工具箱）的基石。

#### **decisions.md**
本文档用于记录AI执行过程的重要决定（如方案选择）、关键发现（如技术限制）、以及新增/删除文件（如新脚本、函数、接口等）。每条decision需要包含的条目包括：类型、概述、依据、造成的影响等

**Example:**
```markdown
# Decision
使用说明：规范AI的写入和更新时机、以及写入内容

## DECISION-001 (2025-11-08)
- type: file addition       # file deletion / decision / discovery
- description： ...         # 决策是要做什么
- reason：...               # 决策依据和逻辑
- influence：add <file>     # 影响了哪些层面（工作流）、或影响了哪些文档
- implemented：completed    # wait approval / processing / completed 
```

---
#### **lessons.md**

本文档用于记录AI执行过程中犯过的错误、造成的后果、相关教训、以及提醒。每条lesson信息需要包含完整的条目：带有状态（是否解决）、上下文的描述、错误主要原因、造成后果、教训总结、以及提醒/解决方案（可选）。

除列举具体错误外，文档还有以下两方面的内容：
- 使用说明：
- 错误索引：key=错误编号，value=简短描述，给出对应章节的链接或line （主要考虑AI使用的便捷程度）

**Example:**
```markdown
# Lessons
使用说明: 描述什么时候写入信息，需要写入哪些内容，在遇到错误的时候应该如何使用本文档
错误索引：错误编号以及错误的描述

## ERROR-001 (2025-11-08)
- status: solved
- summary: When updating modules/payments/api/contract, forgetting to update CONTRACT.md caused interface inconsistencies
- reason: Modified API but not updated CONTRACT.md
- consequence: The front-end call failed, and the troubleshooting was repeated 3 times.
- lesson: Any changes to the interface must be synchronized with doc/CONTRACT.md and contracts_baseline must be refreshed.
- remind / solution: Triggering the `contract-change` guardrail, executes `make contract_compat_check`.
```
---

#### **scripts目录**
该目录用于存放AI执行过程中生成的脚本或函数。每个脚本需要明确描述结构的注解或装饰器，便于导出规范。


临时开发的功能或脚本


这里需要补充示例

---
### 2.3 其他

**数据库**：需要在`interact/`目录下相应文档中声明的数据库操作包括：
- 新增或修改了表/表字段
- 函数、流程、脚本等会新增/删除/修改数值的情况

**测试**：需要在`interact/`目录下声明的测试信息包括：
- 测试数据的生成规则，包括mock和fixtrue
- 测试数据的格式、储存、生命周期的管理，包括mock和fixtrue
  
**接口协议**： 需要在`interact/`目录下声明的接口协议包括：
- 模块类型定义的对外暴露接口：这类接口是面向业务流程的，相当于模块实例和其他实例的交流通道
- 功能实现接口：功能可以为底层函数，也可以为串联好的工作流
- 服务之间的接口

**配置信息**
- 配置信息可能包含：{需要补充}
- 鼓励优先定义全局参数，而不是使用临时变量
- 模块实例需要维护自身的配置信息，项目也会维护一份高层的配置，出现同名时使用实例本地的配置 

**人类阅读文档**：`docs/`目录的维护原则：
- 该目录一般由人类开发者自行维护
- 对模块的性能检查，维护报告等数据，不论是AI还是脚本的生成的，都应该指定路径，并放在目录下

---


## 3. 结合AI的开发流程


除了功能的代码实现和内部信息外，模块实例并不维护系统资源的SSOT（如知识文档、功能体等）。但模块实例的重要职责之一是框定任务编排的边界，为保障模块实例可以更好的履行该职责，我们需要规范模块实例和各类基础设施的联动。

从AI实际参与开发的角度出发，我们首先要确保AI阅读入口的明确性。


### 3.1 单一模块的功能开发


### 3.2 多个模块的联调测试


基于模块化开发思路，业务和数据链路天然的被拆分成了多个模块间的链接关系，也就是此前提到的模板类型图

涉及多个模块实例时，我们将视角切换至应用场景：这些可以提供各式功能的模块，要如何与实际应用需求相结合。

我们在模块总目录（"/modules/"）下

SSOT


**module根目录**
- 已认证的contract





### 3.3 通用规则


我们要求编排系统尊重模块的边界
- **任务编排节点**
  - 单个编排节点的范围限定在单个模块实例中
  - 模块实例需要维护自身的知识文档和可用能力，这个过程是动态的，AI在工作过程中可以请求添加将能力加入路由
  - 整个编排过程允许跨模块的；通常而言，AI从入口模块开始编排，结合整体需求和模块关系图，完成 ？？？？
  - 我们设置一个规则，即每个能力注册表条目必须包含 owner_module，任何具有 exposure: public 的高级能力被视为**业务入口**（如 API 或可由外部或编排器触发的工作流），而 exposure: internal 意味着**它是一个助手**，不直接调度。**需要其他模块的能力**仅作为依赖项； 区分**开发**，**测试**，**联调**，**业务**？？？？？？

- **模块移除和大型重构**，删除模块应该是受控的操作：
  1. 通过类型图检查模块间的依赖关系，如果依赖关系复杂，需要手动批准才能继续
  2. 删除或重新分配模块所有的知识文档
  3. 检查模块包含的能力是否为专属能力，如是则删除能力
  4. 不允许一次操作中间同时重构或删除多个模块




- 内容修改
- 层级维护

我们将更新全局 ROUTING.md 以包括一个枚举可用模块的部分（至少将它们作为导航的一部分列出）。例如，根 README/ROUTING 可以有一个模块摘要表和指向它们的快速入门或主文档的链接，以帮助 AI 轻松找到模块入口点。

---


## 4. 应用基础设施（infrastructure collaboration）

模块实例确定了AI任务编排节点的边界，模板中的基础设施（如知识路由、能力路由等）则用于提高AI的效能和一致性。我们可以从AI编排的视角出发，看一看模板中包含的主要基础设施是否可以融入基于模块化思路的具体开发流程中。

在获取任务后，AI编排周期的工作方式大致如下：
- 读取 `AGENTS.md` 中的任何必需策略
- 加载 `workdocs/` 目录下相关任务的上下文信息，加载必要的触发器
- 查阅 `ROUTING.md` 以定位相关知识文档
- 查阅 `ABILITY.md` 以识别合适的工具
- 更新 `workdocs/` 目录下的相关文档，记录决策过程和执行结果 

### 4.1 知识路由的协同
我们将从应用和增强两个方面，分别讨论知识路由和模块实例开发的协同。

**应用**：
模块实例可以直接应用**知识路由**体系（详见<content_routing.md>）。实例目录维护的 `ROUTING.md`文档以及 `routes`目录，两者的作用分别为：
- `ROUTING.md`: 作为AI查询知识文档的入口，包含精简的scope/topic顶层索引，并给出每个topic对应的路由文件在哪里和大概是干什么的；
- `routes/` 目录：承接ROUTING.md的具体路由，使用`when_to_use` + `doc_usage`两层结构，每层级使用自然语言进行描述，最终指向知识文档地址

**增强**：
我们可以通过以下几个渠道，总结开发过程中好的技巧和工作流程，充实项目的知识文档池
- 归纳AI的工作过程：我们要求AI将做出的重要决策和从错误中学到的经验，分别记录到`workdocs/`目录下的文档`decisions.md`和`lessons.md`中。我们定期回顾和总结这些内容，检查是否可以形成的知识文档。
- 优化开发体验：虽然AI是主力开发，但人类开发则仍然需要投入到项目开发的全周期中，从需求的发起、AI的交流、得到的反馈、到功能的验收、整体进度的推进等，开发人员需要思考如何利用知识文档，和AI保持思想层面的一致性、帮助AI规避常见问题、提高代码一致性等

**总体而言**：
我们希望随着项目的深入，知识文档的深度和准度都可以得到积累，让AI能够在后续工作中输出质量更高且更符合设计需求的成果物。人类开发者需要在过程中，感知知识路由是否有效，并有意识的进行调整，使AI用起来更加顺手。
  
### 4.2 能力路由的协同
与知识路由类似，我们也将从应用和增强两个方面来讨论能力路由的协同。

**应用**：
我们在<ability_routing.md>文档中，详细解释了如何让AI复用脚本从而提高代码的一致性。例目录下的`ABILITY.md`给出了不同情景下的AI可以调用的工具。但和知识路由的 `ROUTING.md`文档不同，`ABILITY.md`不是强制限定能力的边界，实质上更像优先选择的建议。AI使用能力路由的流程示意如下：
1. AI依据任务编排，总结代码实现需求，加载相关知识和规范
2. 在`ABILITY.md`中查询是否有匹配需求的封装工具使用
3. 决定选择工具或是在工程包含的全量工具集中进一步检索
4. 选择调用工具或自行开发

模块实例本身的完成度和开发重心会随着项目深入而发生变化，我们可能需要动态维护`ABILITY.md`来调整AI的工具调用建议，进而使模块实例和能力路由的协同更加高效。
- 新模块实例：实例初始化完成后，会根据功能需求和项目的工具池来构建初始的`ABILITY.md`，形成基本的能力路由
- 开发阶段：每当模块实例需要新增功能、改变需求、或是工具池发生变化、发现更适合AI的工作模式时，都建议更新实例的`ABILITY.md`，提供更为清晰的指引

**增强**：
我们希望AI可以反馈其查找、使用、开发工具的过程，我们有两个主要的记录载体：
- `scripts/` 目录：如果不能在能力路由中找到合适的工具或方法，AI需要把符合规范的脚本文件放到目录下
- `decisions.md`：和知识文档的利用方式不同，我们会使用触发器的方式来记录AI对能力路由的使用情况。在AI阅读`ABILITY.md`时缓存AI工具使用需求，并监管后续的调用或生成。

通过归纳上述两个载体中和能力相关的内容，我们可以：
- 从可复用的角度出发，整合新开发的脚本或功能，然后将其加入能力路由
- 发现AI使用路由的习惯、调用工具的偏好、追踪效果，然后完善`ABILITY.md`

**总体而言**：
开发是动态变化的过程，我们需要将动态变化同步到能力路由体系中，所以需要定期
- 更新路由文档`ABILITY.md`，引导AI根据需求调用能力，起到提高AI决策效率和透明度的作用
- 检查`scripts/`目录下的新增功能，从项目层面考虑是否有需要整合到能录路由中
 
### 4.3 其他设施的协同

**策略协同**： AGENTS.md文档中定义的规范和执行策略遵循向上覆盖的原则
- 就地维护模块实例自身的需求或仅和实例有关联的说明/规则，可以直接在对应层级的`AGENTS.md`中写明，例如：整个实例都需要遵循的，写到实例根目录下的AGENTS.md，前端开发的策略和规范，写入实例前端目录下的策略文档。
**触发器协同**：由于触发器的作用对象是知识文档、封装后流程、路由信息等，其作用机制和模块化的开发方式没有显性冲突
- 触发器会将AI使用能力路由的情况写入文档，用作后续分析



