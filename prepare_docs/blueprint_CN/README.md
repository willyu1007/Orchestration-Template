# 任务编排仓库模板蓝图



我们不是要建设某个具体可运行项目，而是完善提高AI编号效率和准确性的基础设施建设。后续可以作为AI深度参与的项目开发的模板


本蓝图概述了一个为高效且可控的 AI 驱动开发而设计的仓库模板。它将深度 AI 参与整合到软件生命周期中，遵循四个核心原则：智能编排、AI 友好性、模块化开发和自动化维护。每个原则都得到特定机制和约定的支持，确保 AI 代理能够有效地在编码任务中协作，同时开发者保持监督和质量。以下章节详细说明这些原则以及它们如何塑造仓库的设计。

- 建设目标

  - 快速加载所需知识
  - 使用工具处理问题
  - 自动化
    - 上下文记忆
    - 1
  - 一致性
    - 基础单元都需要注册
    - 严格遵循运行规范



文档的front matter
``` yaml
id: fe-layout-skill-001
audience:
doc_type: skill              # skill / workflow / example / alignment
scope: frontend/layout       # 可做成层级：frontend/layout/flex
intents:                     # 支持多个意图
  - fix-flex-overflow
  - fix-horizontal-scroll
stages:                      # 在编排中的适用阶段
  - act
  - review
triggers:                    # 可选：用于运行时匹配条件
  - on-error
  - on-user-complain
attach_to:                   # 可选：绑定到某工作流/节点
  - workflow.frontend-bug-triage.plan
tags:                        # 任意补充标签
  - css
  - flexbox
  - layout
summary: >
  指导如何排查和修复 flex 布局导致的超出和滚动问题。
```





- 文档格式
  - 文档角色和受众：所有文档必须清楚地标明阅读对象（AI或人类）。面向AI的文档应保持简洁（通常<150行）且高度结构化，而面向人类的文档没有严格的长度限制，但需要在开头提醒AI忽略。
  - AI可读文档前置元数据：文件必须以标准化的YAML前置元数据开头。除阅读对象外，元数据还应该包含`purpose`、`doc_role`、`doc_kind`等必要信息以及如`updated_at`、`tags`等可选数据。

  - 格式和样式规则：所有文档使用UTF-8编码和Markdown格式，内容生成和编辑应为英文（用户指定除外）、避免非结构化散文、禁止使用除状态标识外的表情符号。
  - 命名约定：文档文件使用`kebab-case`，环境常量使用`UPPER_SNAKE_CASE`，变量/函数使用`camelCase`，类使用`PascalCase`，数据库实体使用 `snake_case`。标准命名模式使用`<topic>-<type>`以提高清晰度。


  - 组织文档目录：所有面向AI的文档将位于专用路径下，如 doc_agent/（用于 AI 特定知识）或模块特定的 doc/ 子文件夹中，而面向人类的文档（除高级 README 外）将位于 doc_human/ 下或明确分离，以避免混合上下文。进行中的上下文笔记将位于 ai/workdocs/（包含 active 和 archive 子文件夹）中，内部报告位于 ai/reports/，确保 AI 不会意外加载无关的人类导向或归档内容。
- **为所有 AI 可读文档准备前置元数据模式**：每个此类文件必须以标准化的 YAML 前置元数据开头，声明字段如 audience（必须是 "ai" 或 "human"，不能两者兼有）、purpose、doc_role（例如 quickstart、guide、spec、contract 等）、doc_kind（例如 router、agent_policy、capability_index 等），以及元数据如 updated_at 和所有权标签。叶节点文档（路由指向的最终内容文档）应在前置元数据中包含 route_role: leaf，以标记它们为文档路由树中的端点。我们还将引入模式文件（例如 spec/front-matter.schema.yaml 等）来定义这些前置元数据结构，并启用文档格式验证。
- **在文档设计中实现"渐进式披露"原则**：顶级 AI 入口文档（如 ROUTING.md）将保持极其轻量——仅列出基本角色、导航链接或常用命令，没有冗长的背景文本。更深入的详细信息被推迟到较低级别的指南或规范中，通过链接引用，因此 AI 只读取当前任务所需的内容。高级文档回答"做什么，在哪里找到它"；中级文档作为特定任务的手册或策略；低级文档提供完整的技术细节或示例。这种结构帮助 AI 最初加载最少的内容，然后根据需要逐步获取更详细的文档（避免令牌溢出）。我们将把这种渐进式加载方法编入文档编写的最佳实践（例如，在文档中添加关于下一步应阅读哪个文档或部分以获得更深入信息的注释）。


---

React (18+)
MUI v7
TanStack Query
TanStack Router
TypeScript

Node.js/Express
TypeScript
Prisma ORM
Sentry


