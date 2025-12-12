# 阶段七：系统运行时与 Hook Runner 规划

> 目标（一句话）：在前六个阶段打好的骨架、知识、注册与脚手架基础之上，**完成系统级运行时能力的建设**——包括统一的能力执行层（Ability Runtime）、可被 Agent Runtime 调用的 Hook Runner、最小可用的 Agent Runtime 主循环，以及面向外部 AI（Cursor / Claude 等）的使用契约。注意：Hook 不再被建模为单独常驻进程，而是由 Agent Runtime 在需要时同步/异步调用的“事件引擎”。

---

## 1. 阶段定位与范围

### 1.1 在整体路线中的位置

- 阶段一：目录骨架与空文件清单  
- 阶段二：文档规范与 `AGENTS.md` 结构设计  
- 阶段三：核心机制知识与标准流程沉淀  
- 阶段四：各类注册的写入脚本与协同机制（Registry 层）  
- 阶段五：脚手架（Scaffolding）与 AI+人交互流程  
- 阶段六：实际注册一批知识 / 能力 / Hook，并用脚手架做小规模“实战”验证  

**阶段七**：在上述基础上，补齐“真正能跑”的**系统运行时层**，使得：

- ability_routing 中的“统一能力池 + 标准执行脚本”有实际实现；  
- hooks 文档中定义的四类事件（PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop）有统一的 Hook Runner 接口；  
- Agent Runtime 能在与外部 AI 协作的前提下，按事件模型协调 Hook 与 Ability 的调用。

### 1.2 本阶段“做 / 不做”

**做：**

1. 完成 **Tool Runner**：  
   - 提供 `task.create / task.run / task.get / task.update / task.delete / direct-run` 等接口；  
   - 按 Registry + Config 解析能力实现（script/mcp/api），并负责基础 usage 记录。  
2. 完成 **Hook Runner**：  
   - 以库的形式暴露 `run_hooks(event_type, context, blocking_only)`；  
   - 支持四类事件；  
   - 被 Agent Runtime 调用，而不是单独常驻。  
3. 完成 **Agent Runtime 主循环的最小实现**：  
   - 明确 PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop 的调用顺序；  
   - 支持“外部 AI 逐步创建任务、修改参数、选择性执行”的交互模式，而不是要求 AI 一次性产出完整 plan。  
4. 在能力接口（registry/config）中，收敛 guardrail 设计：  
   - 不再使用模糊的 `guardrail` 字段，而是通过 `hooks.pre_call` 显式挂载 PreCall Hook；  
   - 将能力级 Hook（PreAbilityCall / PostAbilityCall）统一纳入 ability 元数据。

**不做：**

- 不把 Hook Runner 实现为必须的独立后台服务，是否异步由具体实现决定；  
- 不在本阶段引入复杂的 MQ / 分布式任务系统；  
- 不实现完整对话系统，只实现足以驱动测试场景的最小 Agent Runtime Loop。

---

## 2. 关键角色与分层结构

### 2.1 角色定义

本阶段涉及三个主要“运行时角色”：

1. **Tool Runner**  
   - 提供任务级接口（`task.*` / `direct_run`）；  
   - 负责从 registry+config 中解析能力实现，并真正调用实现脚本/服务；  
   - 对每次调用记录基础 usage（能力 ID / 状态 / 时延 / 错误等）。  

2. **Hook Runner（Hook Engine）**  
   - 以库形式提供 `run_hooks(event_type, context, blocking_only)`；  
   - 读取 `.system/hooks/*.yaml`，匹配事件与 handler，统一执行并合并结果；  
   - 本身不是独立进程，由 Agent Runtime 在需要的事件点调用。  

3. **Agent Runtime**  
   - 长生命周期的“协调器”：  
     - 接收用户/系统输入；  
     - 构造 PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop 事件；  
     - 调用 Hook Runner；  
     - 调用 Ability Runtime；  
   - 对外暴露一组“操作接口”（create/update/run/delete task 等），供外部 AI 在多轮对话中依次调用。

### 2.2 会话级事件 vs 能力级事件

四类事件自然分为两组：

- **会话级（Session 级）事件**  
  - `PromptSubmit`：每次新任务/消息进入；  
  - `SessionStop`：本次会话或阶段结束。  
  - 特点：不针对某个具体 ability；更多是“这轮会话中发生了什么”的视角。

- **能力级（Ability 级）事件**  
  - `PreAbilityCall`：执行某个 ability 之前；  
  - `PostAbilityCall`：执行某个 ability 之后。  
  - 特点：强绑定到某个 ability 或一类 ability。

设计约定：

- PromptSubmit / SessionStop 的 Hook 只出现在系统级 Hook 配置中（`.system/hooks/*.yaml`），**不挂在 ability 本身**；  
- PreAbilityCall / PostAbilityCall 既有全局 Hook，也允许在 ability 元数据中显式声明能力级 Hook。

---

## 3. Ability 接口与 Config Schema 补完（含 PreCallHook）

### 3.1 Registry / Config / Implement 三层回顾

- **Registry（注册表）**：描述“有什么能力”，包括 ability id、operation_key、意图、输入输出 schema、适用范围等。  
- **Config（执行配置）**：描述“如何执行能力”，包括实现方式（script / mcp / api）、路径、超时、环境配置以及能力级 Hook 绑定等。  
- **Implement（实现）**：真正执行逻辑的脚本/服务（如 `scripts/tools/*.py`）。

本阶段需要在 Config 层中补齐“能力级 Hook”相关字段，同时把 guardrail 概念下沉为 PreCallHook 绑定。

### 3.2 Config Schema 中的 Hook 字段

在 `.system/registry/schemas/ability_impl.schema.yaml` 中，新增或重构相关字段，例如：

```yaml
# ability_impl.schema.yaml（示意）
type: object
required: [id, impl]
properties:
  id:
    type: string          # ability_id
  impl:
    type: object
    required: [kind]
    properties:
      kind:
        type: string      # script | mcp | api
      script:
        type: object
        properties:
          path:
            type: string  # 脚本路径
          args_template:
            type: string  # 命令行模板
          timeout_sec:
            type: integer
          env_profile:
            type: string
      # mcp/api 配置略
  hooks:
    type: object
    properties:
      pre_call:
        type: array
        items:
          type: string    # hook_id 列表，相当于原先 guardrail 的具体化
      post_call:
        type: array
        items:
          type: string    # 能力级 PostAbilityCall Hook
```

设计意图：

- 原先模糊的 `guardrail` 概念被具体化为 `hooks.pre_call`：  
  - “guardrail 就是 PreCallHook 的一种”；  
  - 所有执行前的风险控制和限制，都通过 PreAbilityCall Hook 实现。  
- `hooks.post_call` 用于挂载与该能力强相关的附加逻辑（如特殊审计），而**通用 usage 统计由 Ability Runtime 内部负责**，不要求每个 ability 都挂一个 PostHook。

### 3.3 会话级 Hook 的 Config 仍保留在系统层

PromptSubmit / SessionStop 类型的 Hook，仍通过 `.system/hooks/*.yaml` 声明。例如：

```yaml
# .system/hooks/intent_normalizer.yaml
id: intent_normalizer
event_type: PromptSubmit
enabled: true
blocking: true
handler:
  kind: script
  command: ./scripts/hooks/intent_normalizer.py
```

这些 Hook 不隶属于任何单一 ability，因此不会出现在 ability 接口元数据里，而是由 Agent Runtime 在相应事件点统一调用。

---

## 4. Tool Runner设计 (统一的能力执行脚本)

### 4.1 目标

Tool Runner 为外部世界提供一个“任务中心”的抽象，使 AI 和其他调用方可以：

- 以 id+payload 的形式创建能力调用任务；  
- 在实际执行前多次查看/修改参数；  
- 在合适的时机执行任务；  
- 查询结果或删除任务。

同时，Tool Runner 是 “标准执行脚本” 的实现者，必须符合 ability_routing 文档中对 Create task / Run task / Direct run / Query/modify 的描述。

### 4.2 对外接口

建议能力运行库（`scripts/runtime/tool_runner.py`）提供如下 CLI/API 接口：

- `task.create(ability_id, input_payload) -> {task_key, status}`  
  - 典型行为：  
    - 按 registry 校验 ability 是否存在；  
    - 读取 config 并解析实现信息；  
    - 调用 Hook Runner 的 `run_hooks(PreAbilityCall, …, blocking_only=true)`：  
      - 包含全局 PreAbilityCall Hook；  
      - 包含 ability.hooks.pre_call 中声明的 PreCallHook；  
    - 若通过检查，则分配 task_key，创建 `.system/implement/<task-key>/` 目录和 `tool_call.md`，保存输入与配置；  
    - 返回任务 key 和 availability 状态（如 `available` / `denied_guardrail` / `need_human_confirm` 等）。  

- `task.get(task_key) -> {ability_id, input_payload, status, metadata}`  
  - 用于查询已缓存的参数和状态。  

- `task.update(task_key, patch) -> {updated_payload}`  
  - 用于在执行前修改部分参数；  
  - 不触发 Hook（除非后续特别配置），由 Agent Runtime 再决定何时重新做 PreCall 检查。  

- `task.run(task_key) -> {result}`  
  - 默认行为：  
    - 读取缓存参数与配置；  
    - 做 runtime 自身的基础校验（task 是否过期、实现是否可用等）；  
    - 执行实现脚本/服务；  
    - 记录结果与基础 usage（能力 ID、状态、时延、错误等）；  
    - 调用 Hook Runner 的 `run_hooks(PostAbilityCall, …, blocking_only=false)`，包括：  
      - 全局 PostAbilityCall Hook；  
      - ability.hooks.post_call 中的能力级 Hook。  
  - 是否再次执行 PreAbilityCall：  
    - **默认不再执行**；  
    - 若某些高风险能力在 config 中声明 `double_check_on_run=true`，则可在 run 时再次调用 PreAbilityCall Hook。

- `task.delete(task_key)`  
  - 删除缓存任务与相关临时文件。  

- `ability.direct_run(ability_id, input_payload) -> {result}`  
  - 一次性调用：  
    - 在内部等价于 `task.create -> task.run -> task.delete` 的缩略版；  
    - 适用于低风险操作或调试场景。

### 4.3 usage 记录责任划分

- 通用 usage / telemetry（例如所有能力调用的计数、时延统计）统一由 **Tool Runner 内部**完成；  
- PostAbilityCall Hook 不再被默认用于“每个能力都记 usage”，而是只在有额外附加行为需求时被能力显式挂载。

---

## 5. Hook Runner 设计

### 5.1 角色与命名

- 名称：**Hook Runner**  
- 形态：一个由 Agent Runtime / Tool Runner 调用的库（脚本/模块），**不是必须的长驻进程**；  
- 主要职责：  
  - 解析 `.system/hooks/*.yaml`；  
  - 根据 `event_type` 和 `match` 规则选择适用 Hook；  
  - 调用相应 handler（通常是脚本）；  
  - 合并所有 `HookResult` 并返回给调用方。

### 5.2 核心接口

统一接口设计：

```python
def run_hooks(event_type: str, context: dict, blocking_only: bool = False) -> list[HookResult]:
    ...
```

- `event_type`：`PromptSubmit` / `PreAbilityCall` / `PostAbilityCall` / `SessionStop`；  
- `context`：对应事件的上下文（由 Agent Runtime 构建）；  
- `blocking_only`：  
  - `True` 时，仅执行该事件中 `blocking=true` 的 Hook（典型用于 PromptSubmit / PreAbilityCall）；  
  - `False` 时，执行所有匹配 Hook。

### 5.3 事件级行为约定

- **PromptSubmit / PreAbilityCall**  
  - Agent Runtime 必须使用 `blocking_only=True` 调用 Hook Runner；  
  - HookResult 中的 `hook_signals` 会被合并进 turn_context，并作为 control message 传给外部 AI；  
  - 其中 PreAbilityCall 的 `ability_guard` 等信号可直接影响是否允许执行某次能力调用。  

- **PostAbilityCall / SessionStop**  
  - Agent Runtime 使用 `blocking_only=False` 调用 Hook Runner（可同步或异步实现）；  
  - runtime 必须忽略任何 AI-facing 的字段（如 `hook_signals`），只关心日志/缓存等 infra 字段；  
  - 这些 Hook 不得影响当前 turn 的决策，仅影响未来回合或外部观测。

### 5.4 全局 Hook 与能力级 Hook 的协作

对于 PreAbilityCall / PostAbilityCall：

- Hook Runner 统一从两个来源加载 Hook：  
  1. `.system/hooks/*.yaml` 中声明的“全局 Hook”；  
  2. ability config 中 `hooks.pre_call` / `hooks.post_call` 指定的能力级 Hook。  

Agent Runtime 在触发事件时会：

- 先构造上下文（含 ability_id / caller 信息等）；  
- 将这两个来源的 Hook 合并后传给 Hook Runner，由其统一调度执行。

---

## 6. Agent Runtime 与外部 AI 的协作模式

### 6.1 目标

> 让外部 AI 按“逐步操作”的方式驱动运行时，而不是一次性返回完备 plan。AI 可以在多轮对话中：发现能力 → 创建任务 → 修改参数 → 选择性执行。

### 6.2 对外操作原语（Operation Primitives）

在本阶段，Agent Runtime 对外暴露一组“操作原语”，例如通过 CLI 或某种 RPC 层：

- `LIST_ABILITIES / INSPECT_ABILITY`  
  - 帮助外部 AI 依据 `ABILITY.md` 和 registry 信息确认某个 ability 是否存在、输入输出约束是什么。  

- `CREATE_TASK(ability_id, input_payload)`  
  - 对应 Ability Runtime 的 `task.create`；  
  - Agent Runtime 会构造 PreAbilityCallContext，调用 Hook Runner 进行 PreCall 检查，并将结果反馈给外部 AI。  

- `GET_TASK(task_key)` / `UPDATE_TASK(task_key, patch)`  
  - 便于 AI 在执行前查看、调整参数。  

- `RUN_TASK(task_key)`  
  - 对应 `task.run`；  
  - 执行成功后，Agent Runtime 可选择将关键结果写入对应 scope 的 workdocs。  

- （可选）`DELETE_TASK(task_key)`  

这些原语可以逐步执行，不要求 AI 在单次调用中完成全部流程。

### 6.3 最小主循环（参考）

一个典型的单轮交互可以被描述为：

1. **收到用户输入**  
   - Agent Runtime 构造 PromptSubmitContext；  
   - 调用 `run_hooks("PromptSubmit", ctx, blocking_only=True)`；  
   - 收集 `hook_signals`（如 routing_hint / normalized_intent 等），生成 `turn_context`；  
   - 将 `turn_context` 及当前任务列表摘要传给外部 AI。  

2. **外部 AI 在当前轮可能发出的动作**（而不是单一 plan 对象）：  
   - 读取 `ABILITY.md` / registry，决定是否需要新建任务；  
   - 调用 `CREATE_TASK` 若想尝试某个能力；  
   - 根据返回的 guard 结果，可能继续 `UPDATE_TASK` 或暂缓执行；  
   - 对已经准备好的任务调用 `RUN_TASK`；  
   - 更新 workdocs（plan/context/tasks）以记录当前状态与后续计划。  

3. **执行能力调用**  
   - Agent Runtime 调用 Ability Runtime 的 `task.run`；  
   - 结果写回 workdocs / stdout，并触发 `PostAbilityCall` Hook。  

4. **会话结束或阶段结束**  
   - Agent Runtime 构造 SessionStopContext；  
   - 调用 `run_hooks("SessionStop", ctx, blocking_only=False)`；  
   - 这一步可用于跑 build/test、写 session summary 等，不影响当前回合。

这样一来：

- 外部 AI 不必一次性产出完整计划，而是逐步驱动运行时；  
- Agent Runtime 承担“事件调度 + Hook/Ability 协作”的职责；  
- workdocs 保证任务跨回合的连续性。

---

## 7. 系统级联调与完成标准（DoD）

### 7.1 最少验收场景

本阶段至少应完成以下 end-to-end 流程验证：

1. **简单能力 + PreCall guard 场景**  
   - 通过脚手架新增一个 low-level ability（例如写日志）；  
   - 在 ability config 中挂载一个 PreCallHook，禁止在特定环境下调用；  
   - 外部 AI 调用 `CREATE_TASK` 时收到 guard 信息，调整行为并最终完成一次合法调用。

2. **简单能力 + PostCall 附加 Hook 场景**  
   - 为某个能力挂载一个 `hooks.post_call`（如写审计日志）；  
   - 执行能力后验证：除 ability_runner 的通用 usage 外，附加 Hook 也按预期执行（写特定日志或更新特定文件）。

3. **脚手架 + Runtime 联动场景**  
   - 用阶段五/六的脚手架新增一个新能力 + Hook；  
   - 外部 AI 通过 Agent Runtime 的操作原语驱动一次完整调用（从发现能力 → create → update → run）；  
   - 验证各层 registry/config/Hook/Ability Runtime 行为一致，且 workdocs 有完整记录。

### 7.2 完成判定

当满足以下条件时，可认为 Phase 7 已完成：

1. Ability Runtime：  
   - 提供 `task.create / get / update / run / delete / direct_run` 接口；  
   - 能对至少一两个能力完成 end-to-end 调用，并在 `.system/implement` 下记录任务与结果。  

2. Hook Runner：  
   - `run_hooks(event_type, context, blocking_only)` 对四类事件均可用；  
   - PromptSubmit / PreAbilityCall 的 HookResult 能被正确转换为 turn_context 中的 hook_signals；  
   - PostAbilityCall / SessionStop Hook 不会影响当前 turn 的 AI 行为。  

3. Agent Runtime：  
   - 主循环支持“外部 AI 逐步操作”的模式，而不是强制要求一次性 plan；  
   - 能正确协调 PromptSubmit → PreAbilityCall → Ability 调用 → PostAbilityCall → SessionStop 的事件顺序；  
   - 支持外部 AI 在多轮对话中创建任务但暂不执行，或只执行部分任务。  

4. 配置与接口：  
   - ability config 中不再使用模糊 guardrail 字段，而是通过 `hooks.pre_call / hooks.post_call` 挂载能力级 Hook；  
   - 会话级 Hook 仍通过 `.system/hooks/*.yaml` 管理。  

达到以上标准后，系统级运行时层（Ability Runtime + Hook Runner + Agent Runtime）即可视为“可用”，后续阶段可以在此基础上做性能/可靠性增强以及更多高级特性（异步 Hook worker、复杂调度策略等）。
