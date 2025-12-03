# 运行策略




面向代码大模型的策略入口，限定可执行范围、守护机制与提交要求，确保每次能力调用都按规执行。

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


## 1. 基本规则


### 命名规范

    - low tier的命名方式使用 `base.<domain>.<action>`, 参考格式：
   
    - high tier的命名方方式： `able.<domain>.<intent>`, 参考格式：

### 文档规范


### PR规范

### PR 检查规范

### 测试规范

## 2. 策略加载流程

## 3. 维护



## 根目录AGENTS.md

   - 根目录下的AGENTS.md的职责包括:
        - 定义全局边界，例如允许/不允许、审批点、危险操作流程、提交前置要求等。
        - 给出AI必读清单，给出项目级操作的AI选读清单（需要明确什么时候阅读）
        - 指明两套路由的入口位置（知识路由 ROUTING.md、功能体路由 ABILITY.md）。
        - 约定工作文档（workdocs）的最小写作与恢复规则。
        - 说明触发器/审批的基本原则与优先级。


标准化文档格式、命名约定、目录结构，并确保文档针对 AI 消费进行了优化

- **定义文档角色和受众**：所有文档必须清楚地标明是面向 AI 还是面向人类读者（在前置元数据中），并遵守特定角色的指导原则。面向 AI 的文档（如路由指南、代理策略、能力索引等）应保持简洁（通常 <150 行）且高度结构化，而面向人类的文档（例如 README）没有严格的长度限制，但应标明 AI 可以忽略它们。

- **执行格式和样式规则**：所有文档使用 UTF-8 编码和 Markdown 格式。所有内容生成和编辑（除非用户特别要求）应为英文。避免非结构化散文或过度使用表情符号/符号（简单的状态图标除外）。
  
- **建立文件、目录和代码的命名约定**：例如，文档文件使用 kebab-case（例如 data-spec.md），环境常量使用 UPPER_SNAKE_CASE，Python 变量/函数使用 camelCase，类使用 PascalCase，数据库实体使用 snake_case。标准命名模式可防止混淆和冲突（例如，使用 <topic>-<type> 或 <topic>.<type> 后缀以提高清晰度）。
  
- **组织文档目录**：所有面向 AI 的文档将位于专用路径下，如 doc_agent/（用于 AI 特定知识）或模块特定的 doc/ 子文件夹中，而面向人类的文档（除高级 README 外）将位于 doc_human/ 下或明确分离，以避免混合上下文。进行中的上下文笔记将位于 ai/workdocs/（包含 active 和 archive 子文件夹）中，内部报告位于 ai/reports/，确保 AI 不会意外加载无关的人类导向或归档内容。
  
- **为所有 AI 可读文档准备前置元数据模式**：每个此类文件必须以标准化的 YAML 前置元数据开头，声明字段如 audience（必须是 "ai" 或 "human"，不能两者兼有）、purpose、doc_role（例如 quickstart、guide、spec、contract 等）、doc_kind（例如 router、agent_policy、capability_index 等），以及元数据如 updated_at 和所有权标签。叶节点文档（路由指向的最终内容文档）应在前置元数据中包含 route_role: leaf，以标记它们为文档路由树中的端点。我们还将引入模式文件（例如 spec/front-matter.schema.yaml 等）来定义这些前置元数据结构，并启用文档格式验证。
  
- **在文档设计中实现"渐进式披露"原则**：顶级 AI 入口文档（如 ROUTING.md）将保持极其轻量——仅列出基本角色、导航链接或常用命令，没有冗长的背景文本。更深入的详细信息被推迟到较低级别的指南或规范中，通过链接引用，因此 AI 只读取当前任务所需的内容。高级文档回答"做什么，在哪里找到它"；中级文档作为特定任务的手册或策略；低级文档提供完整的技术细节或示例。这种结构帮助 AI 最初加载最少的内容，然后根据需要逐步获取更详细的文档（避免令牌溢出）。我们将把这种渐进式加载方法编入文档编写的最佳实践（例如，在文档中添加关于下一步应阅读哪个文档或部分以获得更深入信息的注释）。
  
- **确定要在仓库根目录和模块中存在的核心文档文件**：按照约定，每个作为导航中心的目录将包含一个 ROUTING.md（用于导航），如果该目录包含 AI 可执行组件，则可能包含一个 ABILITY.md（原名 CAPABILITIES.md，用于能力索引）和一个 AGENTS.md（用于本地代理策略）。但是，根据修订后的共识，仓库的根目录将保存主要的 AGENTS.md，作为整个项目的单一全局策略和防护栏入口（整合全局规则）。模块目录如果有自己的执行逻辑，可以有自己的 AGENTS.md，但要求在每个子目录中都有所有三个文件的要求已放宽，以减少冗余。如果目录纯粹用于文档或不托管活动代理，它可以省略本地 AGENTS.md 或 ABILITY.md，更高级别的路由将简单地链接到该模块中的相关文档。



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

- 可以维护一个编写指南（含标准命名、正反例、共享 topic 的处理方式），在新目录或模块接入时引用，保证结构演进的一致性。


---

模块实例 vs 集成