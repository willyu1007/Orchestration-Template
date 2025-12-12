# 阶段七：系统运行时与 Hook 基础设施规划

> 目标（一句话）：在前六个阶段打好的骨架、知识、注册与脚手架基础之上，**完成系统级运行时能力的建设**——包括统一的能力执行脚本层（Ability Runtime）、Hook 运行时（Hook Runtime）、最小可用的 Agent Runtime 主循环，以及面向外部 AI（Cursor / Claude 等）的使用契约。

---

## 1. 阶段定位与范围

### 1.1 在整体路线中的位置

- 阶段一：目录骨架与空文件清单  
- 阶段二：文档规范与 `AGENTS.md` 结构设计  
- 阶段三：核心机制知识与标准流程沉淀  
- 阶段四：各类注册的写入脚本与协同机制（Registry Runtime）  
- 阶段五：脚手架（Scaffolding）与 AI+人交互流程  
- 阶段六：实际注册一批知识 / 能力 / Hook，并用脚手架做小规模“实战”验证  

**阶段七：**在此基础上，补齐“真正能跑”的**系统运行时层**，使得：

- ability_routing 中描述的 “task.create / task.run / direct-run” 有实际脚本可调用；  
- hooks 中定义的事件模型（PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop） 有可执行的 Hook Runtime；  
- 外部 AI（Cursor / Claude 等）可以通过一小撮统一脚本接口，正确地使用 Agent Runtime。

### 1.2 本阶段“做 / 不做”

**本阶段要做：**

1. 完成能力执行层（Ability Runtime）：统一的执行脚本（例如 `scripts/runtime/ability_runner.py`）与工具调用规范；  
2. 补完并落地“执行配置文档”（Config）的 schema 与示例，使 Ability Runtime 有可依赖的配置来源；  
3. 实现 Hook Runtime：读取 `.system/hooks/*.yaml` 配置，按事件模型匹配并调用 Hook handler；  
4. 实现一个最小可用的 Agent Runtime 主循环（可以是参考实现），将 Hook Runtime 与 Ability Runtime 串联起来；  
5. 明确面向外部 AI 的使用契约：哪些命令是阻塞的、如何读取 `hook_signals`、看到哪些信号必须等待/征求用户确认。

**本阶段暂不做：**

- 不引入复杂的 MQ / 分布式任务系统（Hook 可以先用同步模式实现）；  
- 不实现真正的多租户 / 权限体系（只在必要处预留接口与 guardrail）；  
- 不追求高性能优化，主要关注**正确性与契约清晰度**。

---

## 2. 关键概念与分层模型

### 2.1 三层结构：Registry / Config / Implement

为避免能力执行脚本“边写边改结构”，在本阶段先将三层概念固定下来：

1. **Registry（注册信息）——“有什么”**  
   - 描述模块、能力、路由、Hook 的 ID、关联关系、状态等；  
   - 文件示例：  
     - 模块：`modules/overview/*.yaml`  
     - 能力：`.system/registry/low-level/*.yaml`、`.system/registry/high-level/*.yaml`  
     - Hook：`.system/hooks/*.yaml`  

2. **Config（执行配置）——“怎么执行”**  
   - 为能力 / Hook 指定具体的执行实现：  
     - 例如某能力是脚本型（script），其脚本路径、命令行模板、超时、所需环境等；  
     - 后续可扩展 MCP / API 等实现方式；  
   - 统一放在：`.system/registry/config/*.yaml`。  

3. **Implement（实现资产）——“在哪儿执行”**  
   - 真正执行逻辑的脚本 / 服务；  
   - 主要落点：`scripts/tools/`、`scripts/abilities/`、`.system/implement/<task-key>/...` 等。  

Ability Runtime 的职责，即是**消费 Registry + Config 两层，调度 Implement 层**。

### 2.2 事件模型与 HookResult / hook_signals

结合 hooks 设计，运行时需要统一：

- **EventContext（事件上下文）**：传递给 Hook 的结构  
  - 含 `event_type`（PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop…）、`session_id / task_id / ability_id / module_id / user_id` 以及与事件相关的 `payload`。  

- **HookResult（Hook 执行结果）**：Hook 返回给 Hook Runtime / Agent Runtime 的结构  
  - 包含：  
    - infra 字段（log / metrics / usage_record 等）；  
    - `hook_signals`：**有限种可影响 AI 决策的信号**；  
    - `status` / `guard_decision`（如 allow/deny/need_user_confirm 等）。  

Agent Runtime 负责：合并多个 HookResult，将其中**属于 AI 的部分**（hook_signals + guard 结果）注入给外部 AI 使用。

### 2.3 Turn Context 与外部 AI 的契约

为方便外部 AI 使用 hook_signals，本阶段定义统一的 turn context 结构，例如：

```json
{
  "session_id": "sess-123",
  "turn_index": 5,
  "hook_signals": {
    "routing_hints": [
      {
        "source": "PromptSubmit.guard_safe_ops",
        "hint": "prefer_safe_ops"
      }
    ],
    "ability_guards": [
      {
        "ability_id": "delete_user_data",
        "decision": "deny",
        "reason": "Prod environment"
      }
    ],
    "risk_alerts": [
      "High-risk write detected in module X"
    ]
  }
}
```

外部 AI：

- 不直接调用 Hook handler；  
- 只需要遵守 AGENTS 中关于 `turn_context` / `hook_signals` 的策略说明：  
  - 看到 `ability_guards.decision == "deny"` 的能力，不得主动调用；  
  - 看到 `risk_alerts` 时需要向用户提示风险并征求确认；  
  - 看到 `routing_hints` 时，优先考虑提示中的能力 / 模块。  

---

## 3. 工作块 A：执行配置文档（Config）补完与 schema 定义

> 本质逻辑属于 Phase 4 的“注册与 schema”层，但实际开发工作放在 Phase 7 开头完成。

### 3.1 任务 A1：Config schema 文件补全

在 `.system/registry/schemas/` 下新增：

- `ability_impl.schema.yaml`  
  - 定义能力执行配置的结构：  
    - `id`（ability_id，引用 registry）  
    - `impl.kind`：`script` / `mcp` / `api`（本阶段优先实现 `script`）  
    - `impl.script.path`：脚本路径  
    - `impl.script.args_template`：命令行参数模板  
    - `impl.script.timeout_sec`：超时时间  
    - `impl.script.env_profile`：环境配置 key（可选）  

- （可选）`hook_impl.schema.yaml`  
  - 若需要为 Hook 声明 handler 实现位置（脚本路径 / ability_id 等），可在此统一定义。  

同时在 `knowledge/architecture/registry_schemas.md` 中增加 Config 一节：

- 明确 Registry / Config / Implement 三层含义；  
- 指出 Config schema 的位置与用途。  

### 3.2 任务 A2：Config 示例文件与最小内容

在 `.system/registry/config/` 下新增：

```text
.system/
  registry/
    config/
      abilities.yaml       # 能力执行配置样例
      hooks_impl.yaml      # （如需要）Hook 实现映射样例
```

示例结构（abilities.yaml）：

```yaml
abilities:
  - id: generate_example_data
    impl:
      kind: script
      script:
        path: scripts/tools/generate_example_data.py
        args_template: "--input {input_file} --output {output_file}"
        timeout_sec: 60
        env_profile: default
```

AI 可以主导生成这些样例并按 schema 填写，人工只需要做结构性检查。

**完成标志（A 块）：**

- Config 层的 schema 与示例文件已建立；  
- registry_schemas 文档中对 Config 的描述清晰；  
- Ability Runtime 实现可以安全依赖这些配置。  

---

## 4. 工作块 B：能力执行层（Ability Runtime）

**目标：**实现统一的能力执行脚本层，使 ability_routing 中描述的 `task.create / task.run / direct-run` 具备实际脚本入口。

### 4.1 目录与文件结构

建议：

```text
scripts/
  runtime/
    ability_runner.py   # CLI 入口：task.create / task.run / direct-run
    ability_lib.py      # 内部库：解析 registry + config、调度实现、调用 Hook Runtime
```

以及能力实现脚本目录（可根据前序规划确定）：

```text
scripts/
  tools/
    generate_example_data.py
    # ...其他工具实现
```

任务记录目录：

```text
.system/
  implement/
    <task-key>/
      tool_call.md      # 记录调用参数 / 结果 / 错误等
```

### 4.2 能力执行接口设计

对齐 ability_routing 中的设计，支持三个接口：

1. `task.create`  
   - 输入：ability id / operation key + payload（可通过文件或 stdin 提供）；  
   - 行为：  
     - 根据 registry 查找 ability 定义；  
     - 根据 config 查找执行实现；  
     - 做基础输入校验与 guardrail 检查；  
     - 若可执行，分配 task key，在 `.system/implement/<task-key>/` 下初始化目录与 `tool_call.md`；  
   - 输出：task key + availability 状态。  

2. `task.run`  
   - 输入：task key；  
   - 行为：  
     - 读取 `.system/implement/<task-key>/tool_call.md` 中缓存的调用信息；  
     - 调用 Hook Runtime 触发 `PreAbilityCall` / `PostAbilityCall` 事件；  
     - 按 config 调用具体实现脚本，写回结果。  

3. `direct-run`  
   - 输入：ability id + payload；  
   - 行为：  
     - 一次性进行校验 → 调用 → 返回结果，不建立 task 目录；  
   - 用于低风险 / 调试场景。  

4. 查询和修改参数

### 4.3 工具调用规范

为方便扩展，规定：

- `impl.kind == "script"` 的能力实现：  
  - 通过调用指定路径的 Python/其它脚本；  
  - 输入优先采用 “input.json + output.json” 的文件协议，能力脚本只需要：  
    - 从 input.json 读取入参；  
    - 将结果写入 output.json。  

未来支持 `mcp` / `api` 时，在 ability_lib 中做封装，不改变 ability_runner CLI 接口。

**完成标志（B 块）：**

- `ability_runner.py` 提供可用的 task.create / task.run / direct-run CLI；  
- 能够对至少一两个真实能力执行 end-to-end 调用，并在 `.system/implement` 下留下调用记录；  
- 发生错误时有清晰的错误信息与退出码。  

---

## 5. 工作块 C：Hook Runtime（Hook Runner）

**目标：**让 `.system/hooks/*.yaml` 中的配置真正“活起来”，在指定事件类型上按配置匹配并执行 Hook handler。

### 5.1 Hook Runner 基础结构

建议文件：

```text
scripts/
  runtime/
    hook_runner.py      # 实现 run_hooks(event_type, context, blocking_only=False)
```

以及 Hook 实现脚本目录（如有）：

```text
scripts/
  hooks/
    usage_tracker.py
    prod_config_guard.py
    # ...其他 Hook handler
```

### 5.2 `run_hooks` 设计

核心函数：

```python
def run_hooks(event_type: str, context: dict, blocking_only: bool = False) -> list[HookResult]:
    ...
```

行为：

1. 读取 `.system/hooks/<event_type>.yaml` 中的 Hook 配置；  
2. 过滤 `enabled == true` 的 Hook；  
3. 若 `blocking_only == True`，进一步过滤 `blocking == true`；  
4. 按顺序执行每个 Hook：  
   - 将 `EventContext` 以 JSON 形式通过 stdin 传给 handler 脚本；  
   - 读取 stdout 中的 JSON 作为 `HookResult`；  
   - 如发生错误，记录并转化为 error 型 HookResult。  

5. 返回所有 HookResult 列表，供 Agent Runtime 合并。  

### 5.3 Blocking / 非 Blocking 语义

在代码与文档中明确：

- `PromptSubmit` / `PreAbilityCall`：  
  - Agent Runtime 必须以 `blocking_only=True` 调用 `run_hooks`；  
  - HookResult 中的 `hook_signals` / guard 决策可影响后续 AI / ability 执行。  

- `PostAbilityCall` / `SessionStop`：  
  - 即使有 Hook，也只作为 infra 行为处理；  
  - 默认可以同步执行，后续有需要再扩展为真正后台 worker；  
  - 返回的 HookResult 不允许携带 AI-facing 的 hook_signals。  

**完成标志（C 块）：**

- 至少能够：  
  - 通过配置 `.system/hooks/*.yaml` + handler 脚本，  
  - 对 PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop 事件完成 Hook 调用；  
- 发生错误时 Hook 不会破坏核心能力调用流程（尤其是 PostAbilityCall / SessionStop 场景）。  

---

## 6. 工作块 D：Agent Runtime 主循环与外部 AI 使用契约

**目标：**实现一个最小可用的 Agent Runtime 主循环（可以是脚本 + 伪代码组合），并与外部 AI（Cursor / Claude 等）约定清晰的使用协议。

### 6.1 Agent Runtime 最小主循环

可以在 `scripts/runtime/agent_runtime.py` 中实现一个参考 loop：

1. 接收外部 AI 或用户的请求（可以是一个 JSON 请求文件）；  
2. 构造 `PromptSubmitContext`，调用：  
   - `run_hooks("PromptSubmit", ctx, blocking_only=True)`  
   - 合并 HookResult 中的 `hook_signals` 到 `turn_context`；  
3. 将 `turn_context` 与用户输入交给外部 AI（这一部分可以先用伪接口说明）；  
4. 外部 AI 返回“plan”（包括将要调用的 ability 列表及参数）；  
5. 对每个 ability 调用：  
   - 构造 `PreAbilityCallContext`，调用 `run_hooks("PreAbilityCall", ...)`；  
   - 根据 guard 决策决定是否调用 `ability_runner`；  
   - 执行完后构造 `PostAbilityCallContext`，调用对应 Hook；  
6. Session 结束时发出 `SessionStopContext`，调用 SessionStop Hook。  

> 阶段七可以只要求：Agent Runtime 能在一个测试脚本中跑通这个流程，不要求对接真实在线模型。

### 6.2 外部 AI 使用 Agent Runtime 的协议

通过根级 `AGENTS.md` + 专门的 runtime 相关 `AGENTS`，明确外部 AI 需要遵守的规则：

1. **调用原则：**  
   - 不直接调用底层工具脚本 / 修改 registry；  
   - 执行能力必须通过 `scripts/runtime/ability_runner.py`；  
   - 修改注册信息必须通过 Phase 4/5 的脚手架脚本。  

2. **阻塞行为：**  
   - 将所有 runtime / scaffolding 脚本视作“阻塞调用”；  
   - 在命令返回前不能假设操作成功；  
   - 返回后要检查退出码与 result JSON。  

3. **信号解释：**  
   - 在每轮决策前，先读取 Agent Runtime 提供的 `turn_context`：  
     - `ability_guards.decision == "deny"` 的能力不得主动调用；  
     - 遇到 `need_user_confirm` 的能力调用，需先向用户解释风险并征求确认；  
     - 参考 `routing_hints` 优先选择推荐路径。  

4. **异常情况：**  
   - 对无法理解的 `hook_signals`，必须：  
     - 不直接忽略；  
     - 在对应 scope 的 `workdocs` 中记录；  
     - 如有疑问，向用户说明不确定性。  

**完成标志（D 块）：**

- 存在一套清晰的 AGENTS 章节/文件，描述外部 AI 使用 Agent Runtime 的完整规则；  
- 至少在一个模拟场景中，外部 AI 可以通过这一协议驱动 Agent Runtime 完成简单任务。  

---

## 7. 工作块 E：系统级联调与验收场景

**目标：**通过若干代表性的 end-to-end 流程，验证 Ability Runtime + Hook Runtime + Agent Runtime 的协同工作能力。

建议至少包含以下验证场景：

1. **低级能力 + PostAbilityCall Telemetry Hook**  
   - 使用阶段 5/6 的脚手架新增一个简单 low-level ability（例如生成日志）；  
   - 为所有 ability 注册一个 `PostAbilityCall` Hook，用于记录使用情况；  
   - 通过 `ability_runner` 多次调用该能力，检查：  
     - 能力执行成功；  
     - Hook 成功记录 usage 到日志。  

2. **PreAbilityCall Guard + ability_runner 联动**  
   - 新增一个 PreAbilityCall Hook，禁止在某些条件下执行危险能力；  
   - 分别执行“正常”调用与“应被拒绝”的调用；  
   - 验证：  
     - ability_runner 会在 Hook deny 时拒绝执行；  
     - 外部 AI 能看到 guard 决策并采取合理行为。  

3. **脚手架 + Runtime 联动**  
   - 使用脚手架新增一个新能力 + 对应 Hook；  
   - 随后用 Ability Runtime 调用该能力；  
   - 确认从“脚手架注册”到“运行时调用”的全链路无缝衔接。  

所有这些实验过程需要在对应 scope 的 `workdocs` 中留有完整记录。

---

## 8. 人 / AI 分工与工作方式

| 工作块 | 任务内容 | AI 角色 | 人类角色 |
|--------|----------|---------|----------|
| A：Config schema & 示例 | 设计并填充执行配置文档 | 主笔：写 schema 与样例 | 审核字段设计与引用关系 |
| B：Ability Runtime | 实现 `ability_runner` + 工具调用规范 | 主笔：写代码与测试脚本 | 决定接口形态与安全约束 |
| C：Hook Runtime | 实现 `run_hooks` 与 handler 调用 | 主笔：写代码与基础 handler | 选择优先实现的 Hook 场景 |
| D：Agent Runtime & 外部 AI 协议 | 设计最小主循环与 `turn_context` 契约 | 主笔：写参考实现与文档 | 校正行为规范与边界 |
| E：系统联调 | 设计与执行 end-to-end 验收用例 | 主笔：组织测试与记录 | 判断是否达到“可用”的标准 |

工作方式继续沿用前几阶段：**“AI 写 / 人审 / AI 改”**，人类不直接重写大量脚本，而是主要在：

- 明确需求与边界；  
- 调整接口与行为契约；  
- 做关键路径的集成测试。  

---

## 9. 阶段七完成判定标准（Definition of Done）

满足以下条件时，可认为 Phase 7 已完成：

1. Config 层的 schema 与示例文件已落地，Ability Runtime 代码实际依赖这些配置运行；  
2. `scripts/runtime/ability_runner.py` 能够对至少一两个能力完成 end-to-end 调用，并在 `.system/implement` 下留下调用记录；  
3. `scripts/runtime/hook_runner.py` 能按 `.system/hooks/*.yaml` 中的配置执行 Hook handler，至少支持 PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop 四类事件；  
4. 存在一个最小可用的 Agent Runtime 主循环（脚本/库均可），在测试场景中可以：  
   - 接收输入；  
   - 触发 PromptSubmit / PreAbilityCall / PostAbilityCall / SessionStop 事件；  
   - 调用 Ability Runtime 执行能力；  
   - 合并 HookResult 并输出 turn_context；  
5. 根级及相关 `AGENTS.md` 中，存在一套清晰的“外部 AI 使用 Agent Runtime”的行为规范：  
   - 规定哪些命令是阻塞的；  
   - 如何读取并理解 `hook_signals`；  
   - 何时需要等待/征求用户确认。  
6. 至少有 2–3 条系统级 end-to-end 用例（如简单能力 + Hook 场景）跑通，且执行过程均有 `workdocs` 记录与简单日志。  

当以上条件满足时，可以认为：模板的**系统级运行时层**已经打通，后续阶段可以进一步：

- 优化性能与可靠性；  
- 引入异步 Hook worker / MQ 等高级特性；  
- 丰富能力种类与 Hook 场景，在真实项目中大规模使用。  
