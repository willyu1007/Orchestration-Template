# Hooks

本文件定义本模板中的 **hook 机制**，用于在以语义为主的基础设施之上，增加一层轻量、可编排、可观测的确定性控制。

在本模板中：

- **Ability Routing** 负责暴露和选择可调用的能力（低层操作 + 高层 workflow / agent），并通过注册表、`ABILITY.md` 等结构化文档，支持 AI 统一地发现和调用能力。
- **Knowledge Routing** 负责暴露和选择可加载的知识文档，通过 `ROUTING.md`、topic route 文件以及带 front matter 的文档，实现多阶段（`understand` / `plan` / `act` / `review`）下的知识检索。
- **Hooks** 则是连接两者的“触发层”：在合适的时间点，基于明确的信号（关键词、工具调用、CI 事件等）激活额外的逻辑，例如：
  - 为 AI 自动推荐能力或知识条目；
  - 在调用前执行 guardrail 检查和补充信息收集；
  - 在调用后记录 usage 数据；
  - 在会话结束或 CI 完成后做聚合处理和质量门禁。

Hook 不承载业务逻辑，核心目标是：

1. 减少 AI 在「找能力 / 找文档 / 记日志」上的负担，最大化开发效率；
2. 提供稳定、可复用的触发点，让 orchestrator 能在不改动业务能力的前提下逐步增强行为；
3. 让 Ability Routing 与 Knowledge Routing 的使用过程 **可观测、可审计、可演进**。

---

## 0. 概念与触发阶段

### 0.1 Hook 与路由系统的边界

- Ability Routing 提供的是「**能做什么、怎么做**」的清单和调用入口；
- Knowledge Routing 提供的是「**要理解什么、怎么查**」的知识图；
- Hooks 只做三件事：
  1. **监听信号**：用户输入、AI 规划步骤、工具调用、CI / 系统事件等；
  2. **产生结构化提示**：把这些信号转成小而稳定的结构化意图或元数据（例如 ability id、scope/topic、stage、risk level 等）；
  3. **调用路由 / 能力 / 脚本**：在不引入新业务能力的前提下，触发建议、检查、记录等辅助行为。

路由本身仍然以语义匹配为主（参见 Knowledge Routing 的 trigger mechanism 章节），Hook 只是为语义匹配补充更确定的“轨道”。

### 0.2 四类触发阶段

本文件在给出了Hook的四种激活方式，并统一其的命名和约定：

1. **命中检验（Prompt Match）**  
   - 对应：用户输入关键词匹配 / `UserPromptSubmit` 类事件  
   - 时机：用户提交 prompt、AI 生成初版任务规划时（主要对应 orchestration 的 `understand` / `plan` 阶段）  
   - 作用：
     - 根据关键词与 matcher 规则，推荐候选 ability；
     - 给 Knowledge Routing 提供结构化触发意图（scope / topic / stage）。
2. **工具调用前检验（Pre-call Check）**  
   - 对应：Pre-call activation（在 ability 被实际执行前）  
   - 时机：orchestrator 决定调用某个 ability，准备创建 `task-key` 和任务缓存时（`act` 前）  
   - 作用：
     - 基于 ability 的 guardrail 规则做环境、风险、配额等检查；
     - 补齐必要字段（例如 target env、变更范围、审批人等）；
     - 明确“可执行 / 不可执行”以及原因。
3. **工具调用后缓存（Post-call Cache）**  
   - 对应：Post-call caching of AI tool usage / `PostToolUse` 类事件  
   - 时机：ability 实际执行结束，准备把结果反馈给 AI 时  
   - 作用：
     - 将本次调用记录到 `/.system/implement/<task-key>/tool_call.md` 或统一日志；
     - 提取「调用意图」「能力 id」「实现形态」「结果摘要」等元数据；
     - 为后续 usage 报表、matchers/guardrails 优化提供原始数据。
4. **会话结束处理（End-session Post-processing）**  
   - 对应：Post-processing based on cached information / `Stop` 类事件  
   - 时机：编辑 / 开发会话结束（IDE 停止、任务标记完成、CI 流水线结束等）  
   - 作用：
     - 聚合一次会话内的所有 tool_call 记录；
     - 生成简要的 session report（可选）；
     - 触发本地格式化 / 构建检查 / TypeScript 编译等质量门禁脚本；
     - 为 CI / 维护 agent 提供增量分析输入。

在实现上，**命中检验** 和 **会话结束处理** 一般依赖 IDE 或 orchestrator 的事件系统（如 `UserPromptSubmit` / `PostToolUse` / `Stop`），**工具调用前/后** 则更多由能力执行脚本和 registry 的 `matcher` / `guardrail` 字段协同完成。

---

## 1. 通用行为约定

本节提供所有 Hook 共享的通用约束，确保在整个 repo 内的行为一致且可预测。

### 1.1 命中检验

命中检验是最轻量、也是使用频率最高的 Hook 形态，主要职责是：**在用户刚提出需求时，就把合适的能力和知识“摆在台面上”**，减少 AI 自己穷举和猜测的成本。

触发输入：

- 用户原始输入（chat / CLI / 表单）；
- 当前工作目录 / module / integration 信息（决定可见的 `ABILITY.md` 和 `ROUTING.md`）；
- 可选：最近一次 AI 的计划描述。

推荐行为：

1. 使用 ability registry 中的 `matcher` 规则：
   - 关键词 / 同义词匹配；
   - 负向关键词排除（例如 ability 的 matcher 中声明 `backup` 为负向关键词时，不建议该 ability 出现在“备份”场景）。
2. 把匹配结果映射为：
   - 一组候选 ability id（优先高层 ability，再补充底层 operation）；
   - 每个 ability 对应的简要用途 / 典型调用方式；
   - 对应的 Knowledge Routing 入口（scope / topic）。
3. 为 Knowledge Routing 构建结构化触发意图：
   - `scope`: 来自 ability 所在模块或明显相关的 execution 范围，例如 `execution.quality_gates`；
   - `topic`: 常见问题族，例如 `pre_commit_checks`、`bug_triage`；
   - `stage`: 根据当前用户意图推测是 `understand` / `plan` / `act` / `review` 中的哪一种或哪几种。

示例（概念性）：

- 用户输入：“帮我在本地跑一下所有质量检查，看下是否能通过 CI 的 dev_check。”  
- 命中检验 Hook：
  - 从 `ABILITY.md` 中找到 ability `dev_check`（高层能力）以及相关底层 check 脚本；
  - 构建意图 `{ scope: "execution.quality_gates", topic: "pre_commit_checks", stage: ["act","review"] }`；
  - 建议 orchestrator：
    - 优先调用 ability `dev_check`；
    - 同时加载知识文档 `quality-quickstart`（来自对应 topic route）。

在具体实现上，可以参考 `hook_reference.md` 中类似 `skill-activation-prompt` 的模式：在 `UserPromptSubmit` 事件上执行一个命令脚本，脚本只返回结构化建议（例如 JSON 列表），由 orchestrator 决定是否采纳。

### 1.2 工具调用前检验（Pre-call Hook）

工具调用前检验以 **guardrail** 为核心，负责在「即将执行能力」时做最小但关键的防护和数据补全。

输入：

- 待执行的 ability id；
- 根据 registry 解析出的实现信息（脚本路径、MCP server、API endpoint 等）；
- 当前执行环境（local / dev / staging / prod 等）；
- 当前输入 payload（包含 AI 已经填好的字段）。

处理步骤（建议）：

1. 使用 ability registry 中的 `guardrail` 配置：
   - 检查环境限制（例如“禁止在 prod 调用 destructive ability”）；
   - 检查配额 / 频率限制（避免无限循环调用外部 API）；
   - 检查是否需要人工确认（例如“批量删除”“强制迁移”等操作）。
2. 如需要补充字段，则显式返回缺失项：
   - 例如要求 AI 提供 `reason` 字段（调用意图），用于写入 `tool_call.md`；
   - 要求给出 `risk_level` 或 `estimated_impact` 的枚举值；
   - 要求显式选择 `target_env`（local / staging / prod）。
3. 如果依然不可执行：
   - 返回 `unavailable` 状态和明确原因（与 Ability Routing 中的“可用性检查”保持一致）；
   - 由 orchestrator 决定是否换用其他 ability 或回退到纯解释性回答。

这一阶段的 Hook 实现通常直接内嵌在统一的 ability 执行脚本中（参见 Ability Routing 中对 task-based 执行流程的描述），不依赖 IDE 事件。对于复杂场景，也可以在 IDE 侧暴露一个 PreToolUse 事件，由设置中的 Hook 调用统一脚本完成上述检查。

### 1.3 工具调用后缓存（Post-call Hook）

Post-call Hook 用于**把一次具体的能力调用变成有价值的可回溯记录**，为后续分析和改进提供基础。

输入：

- `task-key`（如果是 task-based 执行）；
- ability id 与具体实现（脚本 / MCP / API 等）；
- 执行结果（成功 / 失败 / 失败原因）；
- 可选：AI 对结果是否符合预期的反馈。

行为约定：

1. 在 `/.system/implement/<task-key>/tool_call.md` 中追加或更新记录，包括：
   - 创建时间与结束时间；
   - 调用意图（AI 提供的一两句话）；
   - ability id、实现 id 以及关键参数摘要；
   - 执行结果、错误信息；
   - AI 对结果的满意度 / 是否需要后续人工复查。
2. 对于非 task-based 的快捷调用，如果需要记录，也应尽量复用同一格式，以方便工具（如 `ability usage-report`）统一读取。
3. 可以在 IDE 事件 `PostToolUse` 上配置一个轻量脚本（类似 `post-tool-use-tracker`）：
   - 从 IDE 暴露的工具调用上下文中抽取关键信息；
   - 将其序列化追加到对应的 `tool_call.md` 或集中日志文件。

这些记录为 Ability Routing 中提到的检查工具提供基础：例如 `ability usage-report`、`ability consistency-check` 等。

### 1.4 end-session 处理

End-session Hook 在一次连续开发 / 编辑会话结束时触发，典型事件包括 IDE 的 `Stop`、任务显式完成、CI job 结束等。

推荐职责分为两类：

1. **质量门禁（Quality Gates）**
   - 自动执行本地格式化、静态检查、构建检查等脚本：
     - 例如调用 `prettier` 格式化当前工作集；
     - 触发扩展的构建 / 测试脚本（可对应 Ability Routing 中的高层 ability，如 `dev_check`、`tsc-check` 等）；
   - 将结果以简要形式反馈给 AI：
     - 提醒哪些文件没有通过格式化；
     - 哪些测试 / 类型检查失败；
     - 是否建议在提交前修复。
2. **会话级汇总与交接**
   - 汇总本次会话中的工具调用记录（可从多个 `tool_call.md` 读取）；
   - 生成一个简要的 session 报告，例如：
     - 主要修改模块与 ability；
     - 涉及的知识 scope/topic；
     - 仍然存在的风险或未完成任务；
   - 将该报告写入固定位置（例如 `.system/logs/session-*.md`），供后续 AI 或人工继续工作时快速加载。

在具体落地上，可以参考 `hook_reference.md` 中 Stop 事件下串联多个脚本的做法：格式化 / 构建检查 / 可选的 TypeScript 检查等。

---

## 2. Hook 实现与配置

本节从实现角度说明如何在仓库中配置 Hook，使之可被代码大模型和任务编排器稳定调用。

### 2.1 事件类型与配置入口

模板假定存在如下基础事件类型（IDE / orchestrator 提供）：

- `UserPromptSubmit`：用户提交输入时触发；
- `PostToolUse`：工具或能力执行完成后触发；
- `Stop`：当前编辑 / 开发会话结束时触发。

在具体实现中，这些事件可以通过类似 `settings.json` 的配置文件与命令脚本绑定。一个概念性的结构如下（仅示意）：

```jsonc
{
  "hooks": {
    "UserPromptSubmit": [
      {
        // 命中检验：自动推荐能力 / 文档
        "hooks": [
          { "type": "command", "command": ".claude/hooks/skill-activation-prompt.sh" }
        ]
      }
    ],
    "PostToolUse": [
      {
        // 工具调用后缓存：记录调用信息
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/post-tool-use-tracker.sh" }
        ]
      }
    ],
    "Stop": [
      {
        // end-session：格式化 + 构建检查等
        "hooks": [
          { "type": "command", "command": ".claude/hooks/stop-prettier-formatter.sh" },
          { "type": "command", "command": ".claude/hooks/stop-build-check-enhanced.sh" }
        ]
      }
    ]
  }
}
```

说明：

- `matcher` 字段用于过滤需要执行 Hook 的工具类型或场景（例如只在编辑写入类操作后记录调用）。
- 每个事件可以配置多个脚本，按顺序依次执行。
- 上述脚本路径和命名仅为示例，可根据项目需要调整；真正的行为以本文件的概念约定为准。

### 2.2 必选 / 推荐 / 可选 Hook 组合

结合效率与安全性，可以将 Hook 大致分为三档：

1. **必选（Essential）**
   - 命中检验脚本（如 `skill-activation-prompt`）：
     - 负责在 `UserPromptSubmit` 时根据 matcher / trigger rules 自动推荐能力和知识；
     - 不依赖项目特定实现，通常可以直接复用。
   - 工具调用后缓存脚本（如 `post-tool-use-tracker`）：
     - 负责在 `PostToolUse` 时记录调用信息；
     - 写入 `tool_call.md` 或集中日志。
2. **推荐（Recommended）**
   - 结束时的格式化脚本（如 `stop-prettier-formatter`）：
     - 在 `Stop` 事件执行，确保代码风格一致；
     - 与能力池中对应的格式化能力保持一致。
   - 结束时的构建检查脚本（如 `stop-build-check-enhanced`）：
     - 在 `Stop` 事件执行，运行项目的基础构建 / 测试；
     - 应优先调用已有的能力（如 `dev_check`）。
3. **按需可选（Optional）**
   - TypeScript 专用类型检查脚本（如 `tsc-check`）：
     - 仅在多服务 TypeScript monorepo 或复杂前端项目中启用；
     - 单服务仓库或非 TS 堆栈可以不配置。
   - 其他项目特定检查脚本：
     - 例如 OpenAPI schema 校验、数据库迁移 dry-run 等。

推荐做法：**先落地必选 Hook，确保有基本的能力建议与 usage 记录，然后再按项目栈逐步补充推荐和可选 Hook**。

---

## 3. 与 Ability Routing 的协同

Hook 机制与 Ability Routing 的关系，可以从“发现 → 选择 → 执行 → 维护”四个阶段来理解。

### 3.1 利用 matcher 加速能力发现

Ability registry 在低层和高层能力配置中都支持 `matcher` 字段，用于：

- 定义能力相关的关键词 / 同义词；
- 定义负向关键词，避免在不合适场景下误触发；
- 根据模块 / scope 对匹配范围做过滤。

命中检验 Hook 应该直接基于这些 matcher 规则：

1. 从当前工作 scope 的 `ABILITY.md` 出发，定位可见的能力集合；
2. 读取各能力的 matcher 配置，将用户输入与这些规则进行比对；
3. 将命中的能力按照优先级排列：
   - 高层 ability（workflow / agent）优先；
   - 低层 operation 作为补充选项；
4. 将结果以简要列表的形式反馈给 orchestrator，让代码大模型可以在规划阶段直接引用这些能力，而无需自己穷举搜索。

这样，**Ability Routing 中定义的 matcher 就变成了命中检验 Hook 的配置源**，避免出现“文档一套、实际行为一套”的情况。

### 3.2 在 guardrail 中实现安全与前置检查

Ability registry 中的 `guardrail` 字段用于描述能力执行的限制和策略，例如：

- 仅允许在 `staging` / `test` 环境调用；
- 超过某个影响范围需要人工确认；
- 某些操作只能在 CI 中自动执行，禁止本地直接调用。

这些规则在 Pre-call Hook 中被具体执行：

1. 当 orchestrator 计划调用某个 ability 时：
   - 统一执行一段 Pre-call 检查逻辑，解析该 ability 的 guardrail；
   - 根据当前环境和输入决定：
     - 是否允许直接执行；
     - 是否需要提问获取更多信息；
     - 是否需要人工确认。
2. 如果 guardrail 检查失败：
   - 返回“不可执行 + 原因”给 AI；
   - 建议替代方案（例如使用只读 variant 或在非 prod 环境执行）。
3. guardrail 也可以附带需要写入 `tool_call.md` 的审计字段：
   - `risk_level`、`change_scope`、`approver` 等。

通过这种方式，**Hook 把 guardrail 从“纯文档约定”变成了“实际生效的执行规则”**。

### 3.3 使用工具调用日志驱动能力维护

Ability Routing 推荐为每次 task-based 调用创建 `tool_call.md` 记录，并提供如 `ability usage-report` 等检查工具。Post-call Hook 正是生成这些记录的关键。

协同方式如下：

1. Post-call Hook 在每次能力执行后更新 `tool_call.md`；
2. 维护 agent 或 CI 中的检查任务可以定期调用：
   - `ability usage-report`：分析高频能力组合，识别哪些场景可以抽象为新的高层 ability；
   - `ability consistency-check`：发现 registry 中存在但长期未被调用的能力；
   - `ability dead-link-check`：检测 `ABILITY.md` 中指向不存在 registry 条目的条目。
3. Pre-call / 命中检验 Hook 也可以根据 usage 统计调整行为：
   - 在相似意图下优先推荐最近成功率更高的能力；
   - 对失败率高的能力增加额外提示或 guardrail。

这使得 Hook 机制不仅参与单次调用，还参与 Ability Routing 的长期演化。

---

## 4. 与 Knowledge Routing 的协同

Knowledge Routing 把知识文档组织为 scope / topic / route / related_docs，并引入 `stage` 以及可选的触发机制。Hook 机制可以被视为该触发机制的具体实现之一。

### 4.1 从命中检验到知识触发

在命中检验阶段，Hook 除了为 Ability Routing 提供候选能力外，还应构造用于 Knowledge Routing 的触发意图（intent），典型字段包括：

- `scope`：根据当前模块和任务性质推断（例如质量相关任务映射到 `execution.quality_gates`）；
- `topic`：根据用户用词中的问题族选择相应 topic（例如 `pre_commit_checks`、`bug_triage`）；
- `stage`：结合用户当前阶段推断是 `understand` / `plan` / `act` / `review` 中的哪一种；
- `signals.user_raw_input`：原始用户输入，供 Knowledge Routing 的语义匹配参考。

然后，orchestrator 可以按 Knowledge Routing 触发机制的建议流程处理：

1. 根据 `scope` / `topic` 过滤 `route_indexes`，选择一到多个 topic route 文件；
2. 在这些 topic route 中，结合 `stage` 优先选择对应阶段的 `related_docs`；
3. 再用 `intent` 和 `signals.user_raw_input` 与 `when_to_use` / `doc_usage` 字段进行语义匹配；
4. 最终选择最合适的知识文档加载到上下文中。

Hook 在此扮演的是“**把非结构化输入转成小巧的结构化 intent**”的角色，而路由决策仍由 Knowledge Routing 的规则和 AI 的语义能力完成。

### 4.2 执行阶段的知识协同

在 Pre-call / Post-call 阶段，Hook 也可以与 Knowledge Routing 协同：

- **Pre-call**：
  - 当准备执行高风险或复杂的 ability 时，Hook 可以建议加载相关的 workflow / alignment 文档；
  - 例如在执行 `billing_regression` 等能力前，提醒 AI 先阅读对应的演练流程和回滚策略。
- **Post-call**：
  - 当能力执行失败或结果不符合预期时，Hook 可以触发“故障排查”类 topic route；
  - 例如加载 `frontend.bug_triage` 下的排查步骤文档，帮助 AI 快速定位问题。

这些协同行为应尽量通过既有的 scope / topic / stage / doc_type 元数据实现，而不是在 Hook 中硬编码文档路径。

### 4.3 知识使用的记录与演进

类似 Ability Routing 的 usage 记录，Knowledge Routing 也可以受益于 Hook 带来的可观测性：

1. Post-call Hook 在写入 `tool_call.md` 时，可附带当前加载的主要知识文档 id：
   - 例如 `docs_used: ["quality-quickstart", "bug-triage-playbook"]`；
2. End-session Hook 汇总本次会话中使用过的知识文档及其 stage：
   - 帮助维护者识别哪些文档被频繁使用、在哪些阶段最有价值；
   - 发现长期未被使用或匹配效果差的文档；
3. 维护 agent 可以基于这些信息调整 Knowledge Routing：
   - 为高价值文档增加更明确的 `when_to_use` / `doc_usage` 描述；
   - 合并冗余文档或拆分过长文档；
   - 调整 route 中的优先级或 stage 标注。

最终目标是：**Hook 把“知识被如何使用”的事实反馈回 Knowledge Routing 的结构设计中，形成闭环**。

---

## 5. 与持续集成（CI）和质量门禁的协同

Hook 机制非常适合作为 CI 和本地开发之间的桥梁：同一套能力和检查脚本既可以被本地 Hook 调用，也可以在 CI 中作为质量门禁执行。

### 5.1 本地质量门禁（通过 Hook 实现）

典型做法：

- 在 `Stop` 事件配置：
  - 代码格式化脚本（如 `stop-prettier-formatter`），调用对应 ability 完成格式化；
  - 构建 / 测试脚本（如 `stop-build-check-enhanced`），调用 ability `dev_check` 或等价能力；
  - 可选的 TS 类型检查脚本（如 `tsc-check`），仅在相关项目开启。
- End-session Hook 将这些脚本的结果整理为摘要：
  - 标记哪些检查失败；
  - 给出建议的修复步骤；
  - 必要时将失败信息写入 `tool_call.md` 或 session 报告。

这样，代码大模型在结束一次工作前就能清楚当前变更是否通过本地的“软门禁”。

### 5.2 CI 中的路由与 Hook 配合

Ability Routing 已经建议在 CI 中定期运行维护性任务（例如同步 `ABILITY.md`、检查 registry 一致性等）。Hook 可以为这些任务提供输入数据和触发点：

- CI job 可以读取由 Post-call / End-session Hook 生成的 `tool_call.md` 和 session 报告：
  - 基于最近一段时间的实际使用调整能力优先级；
  - 检查是否存在大量失败的能力调用，提示需要修复或降级；
  - 检查是否有 registry 中存在但长期没有 usage 记录的能力。
- CI 也可以反向触发 Hook：
  - 当发现路由或能力配置不一致时，触发“维护建议”类 Hook，生成一份供 AI 修复的任务描述；
  - 将这些任务分发给维护 agent 或在下一次开发会话开始时作为建议展示。

### 5.3 防护栏（Guardrails）的分层

为了避免 Hook 本身成为新的风险点，建议将防护逻辑分层：

1. **Ability 层 guardrail**：由 ability registry 定义，Pre-call Hook 强制执行；
2. **环境层防护**：CI / 生产环境通过配置禁止某些 Hook 或能力执行；
3. **Hook 自身的限制**：
   - Hook 脚本应尽量只读或只写受控的系统目录（如 `.system/implement`、`.system/logs`）；
   - 不应直接修改业务数据或部署状态；
   - 对外部命令的调用需要在配置中显式声明。

通过这三层防护，可以在不牺牲效率的前提下保证安全性和可控性。

---

## 6. 维护与演进

Hook 机制本身也需要维护，但维护成本应该尽量低，并与 Ability / Knowledge Routing 的维护流程共享。

建议实践：

1. **少而精**：保持 Hook 数量可控，优先建设少数通用、稳定的 Hook，而不是为每个小场景增加专用 Hook。
2. **配置优先**：把行为差异尽量放在 matcher / guardrail / intent schema 等配置中，而不是散落在脚本逻辑里。
3. **与路由文档同步更新**：
   - 新增高层 ability 时，考虑是否需要更新命中检验 Hook 的 matcher 规则；
   - 调整 Knowledge Routing 的 scope/topic/stage 时，检查是否需要更新相关 Hook 的触发意图构造逻辑。
4. **使用 usage 数据驱动演进**：
   - 定期运行基于 `tool_call.md` 的 usage 报表；
   - 识别“高频但调用路径复杂”的场景，考虑新增高层 ability 或知识文档；
   - 对长期不用的 Hook 或能力进行降级或移除。

最终，Hooks、Ability Routing 和 Knowledge Routing 应构成一个闭环系统：

- 路由为 AI 提供结构清晰的“地图”；
- Hook 在关键节点提供确定性的触发和记录；
- usage 与 CI 反馈驱动路由与 Hook 的长期演进。

只要遵守本文件中的约定，代码大模型和任务编排器就可以在这个模板中 **高效、可控、可持续地完成开发工作**。
