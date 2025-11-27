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
├── outcomes/              # 配置文件    
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

### outcomes
- 执行过程中开发的脚本
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
│   │   ├── task-1-decision.md  # 记录工作中的决策
│   │   ├── task-1-lesson.md    # 错误经验，避免再犯
│   │   └── task-1-task.md      # 任务完成情况和进度   
│   └── task-2/
│       └── ...
└── archive/               # Completed work (optional)
    ├── summary/           # 陈旧内容
    ├── old_tasks/         # 已完成的任务
        └── ...
```

该目录是AI执行过程中的高频交互对象，鼓励AI将思考和决策过程、执行结果等信息持续写入相应的文档


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

### 2.2 接口维护

基于模块化开发思路，业务和数据链路天然的被拆分成了多个模块间的链接关系，也就是此前提到的模板类型图

我们需要维护以下几类接口
- 模块类型定义的对外暴露接口：这类接口是面向业务流程的，相当于模块实例和其他实例的交流通道
- 功能实现接口：功能可以为底层函数，也可以为串联好的工作流
- 服务之间的接口


### 2.3 AI运行上下文

作为项目开发的主力，AI的工作过程必须是可持续和可回溯的。在丢失上下文的情况下，AI需要可以快速掌握所需信息，而不是从头再来一遍。

我们希望维护一套机制，帮助帮助AI完成思考->决策->执行->记录->回顾->思考的工作闭环。这是`workdocs`目录的设计初衷和核心作用：让AI通过阅读自己写入的信息，了解任务进度、避免重复犯错、回溯思考过程、以及查看运行效果。

我们先对`workdocs/`目录下的几个一目了然的文档和目录进行简要说明：
- `/AGENTS.md`：面向AI的使用手册。文档中将包含上下文的更新规范、使用建议、阅读顺序、写入要求等内容，这里我们仅给出要点概述
- `/archive/old_tasks/`：已完成任务归档后将存放在该目录下
- `/archive/summary/`：对于开发流程很长的任务，上下文中的陈旧信息将按照一定规则，总结后放入该目录。根据总行数、更新次数、时间等条件触发清理

接下来，我们将重点介绍  `workdocs/active/[task_name]/`目录。目录中包含五个主要的文档:`plan.md`、`context.md`、`decision.md`、`lesson.md`、以及`task.md`，这些文档使用的基本原则如下
- **when to skip**: 没有难度的执行，如简单bug修复、细节修改、快速更新等
- **when to use**: 当工作可能跨session时，如当完成了一个复杂任务、完整的功能、重大重构时
- **update frequency**： 每次完成一个里程碑节点时，总结工作过程并更新文档
- **keep plan current**：每当任务范围发生变化，需要更新计划、添加新的阶段、记录原因。
- **make tasks actionable**：包含明确的文件名、清晰的验收标准、与其他任务的相关依赖等，eg: "Implement JWT token validation in AuthMiddleware.ts (Acceptance: Tokens validated, errors to Sentry)"

前文提到我们需要帮助AI完成工作闭环，一个典型的工作流程包括：
- 开始任务
  1. 检查是否和当前其他任务重复，确认无重复后在`active/`目录下创建任务目录和5个文档
  2. 根据需求，AI结合代码库深入思考，并将执行计划和阶段性需求写入`plan.md`和`task.md`;
  3. 检查计划是否合理，有没有考虑不周全的地方 
- 实施期间
  1. 参考`plan.md`了解整体策略
  2. 根据要求，定期更新`context.md`、`decision.md`、`lesson.md`文档
  3. 根据检验标准，在`task.md`中勾选已完成的阶段/指标
- 上下文重置后
  1. AI重新读取`plan.md`、`context.md`、`task.md`
  2. 快速了解完整的任务执行状态，根据需求读取`decision.md`、`lesson.md`
  3. 从上次中断的地方继续运行

---

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

#### context.md

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
#### decision.md

本文档用于记录AI执行过程的重要决定（如方案选择）、关键发现（如技术限制）、以及新增/删除文件（如新脚本、函数、接口等）。每条decision需要包含的条目包括

**Example:**
```markdown
# Feature Name - Decision

## DECISION-001 (2025-11-08)
- type: file addition       # file deletion / decision / discovery
- description： ...         # 决策是要做什么
- reason：...               # 决策依据和逻辑
- influence：file paths     # 影响哪些文件
- implemented：completed    # wait approval / processing / completed 
```

---
#### lesson.md

本文档用于记录AI执行过程中犯过的错误、造成的后果、相关教训、以及提醒。每条lesson信息需要包含完整的条目：带有上下文的描述、错误主要原因、造成后果、教训总结、以及提醒（可选）。

**Example:**
```markdown
# Feature Name - Lessons

## ERROR-001 (2025-11-08)
- summary: When updating modules/payments/api/contract, forgetting to update CONTRACT.md caused interface inconsistencies
- reason: Modified API but not updated CONTRACT.md
- consequence: The front-end call failed, and the troubleshooting was repeated 3 times.
- lesson: Any changes to the interface must be synchronized with doc/CONTRACT.md and contracts_baseline must be refreshed.
- AI remind: Triggering the `contract-change` guardrail, executes `make contract_compat_check`.
```
---
#### tasks.md

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

### 2.4 产出和交互

**交互**：


**产出**：
实例功能逐渐完善的过程中，最主要的可复用的产出物是函数或脚本。这些脚本是功能路由（项目的重要基础设施，相当于AI的工具箱）的基石。
我们在<ability_routing.md>文档中，详细解释了如何让AI复用脚本从而提高代码的一致性。这里，我们将专注叙述过程中产出脚本的维护规范。



### 2.5 其他


---


## 3. 结合AI的开发流程


除了功能的代码实现和内部信息外，模块实例并不维护系统资源的SSOT（如知识文档、功能体等）。但模块实例的重要职责之一是框定任务编排的边界，为保障模块实例可以更好的履行该职责，我们需要规范模块实例和各类基础设施的联动。

从AI实际参与开发的角度出发，我们首先要确保AI阅读入口的明确性。


### 3.1 单一模块的功能开发


### 3.2 多个模块的联调测试


涉及多个模块实例时，我们将视角切换至应用场景：这些可以提供各式功能的模块，要如何与实际应用需求相结合。

我们在模块总目录（"/modules/"）下

SSOT





### 3.3 通用规则

正如开头所说，我们要求编排系统尊重模块的边界
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

模块实例确定了AI任务编排节点的边界，模板中的基础设施（如知识路由、能力路由等）则用于提高AI的效能和一致性。不论如何，我们可以从AI编排的视角触发

有了上述结构，AI 编排周期的工作方式如下：AI 获取任务，查阅顶级 `ROUTING.md` 以定位相关上下文（并读取 `AGENTS.md` 中的任何必需策略），从能力注册表中识别合适的高层能力，然后执行它。编排器使用一个将上下文 → 策略 → 能力映射的决策流程：

- **上下文查找**：通过路由文件中的 context_routes（范围/主题/时机）找到正确的文档。
- **策略检查**：在继续之前，为该上下文加载任何必需的护栏或策略（来自 `AGENTS.md`）。
- **选择能力**：选择与任务意图匹配的高层能力（工作流/代理）（能力注册表按域和意图组织），并检查是否允许（只有稳定的能力对编排器可用——参见维护部分）。
- **执行计划**：运行高层能力，它按照其步骤中定义的方式编排低层方法调用（脚本、API 调用）。编排器不会微观管理低层调用；它委托给工作流/代理定义，这确保了多个步骤的一致执行。
- **迭代或移交**：如果任务需要多个步骤或子任务，编排器可能会循环返回以获取更多上下文或能力用于下一步（始终遵循路由和策略指导）。在某些情况下，它可能会移交给专门的子代理（例如，用于一系列数据库操作的"DatabaseOps"代理），如能力图中定义的那样。


所有的功能/知识/触发器，都需要挂在到具体的模块实例下
### 4.1 知识路由的协同

### 4.2 能力路由的协同

- 相关文档/文件：实际开发是动态变化的过程，为了将动态变化同步到能力路由体系中，模块实例维护了以下文档（通常由AI执行）
  - 路由文档**ABILITY.md**，引导AI根据需求调用能力，起到提高AI决策效率和透明度的作用。模块初始化时会生成初版的路由文档，也会根据项目开发中涉及的需求动态更新，以持续起到提效的作用。
  - 决策说明：用来记录AI大模型在任务编排过程使用能力路由的情况，记录信息：`希望调用的功能`、`实际调用的能力`、`结果是否可用`等。
  - 新增功能：如果AI找不到任何可以使用的方法
- 实例初始化：
  - 初始化不要求做任何具体的功能实现，但需要根据模块的功能需求，检索当前项目已有的低/高层功能，根据匹配关系生成第一版路由文档
  - 
- 典型流程：
  1. 检查**ABILITY.md**
  2. 根据检验结果，选择调用工具或自行开发
  3. 记录决策过程和结果
  4. **人为**更新路由、注册新功能等动态操作

- 文档维护：

 

- 运行策略
- 触发器体系


- 功能开发
  - 先找工具 -> 没有则维护在内部 -> 记录 -> 视情况整合 

临时开发的功能或脚本

### 4.3 其他设施的协同

- 策略协同： AGENTS.md文档中定义的规范和执行策略遵循向上覆盖的原则
- 触发器协同：

