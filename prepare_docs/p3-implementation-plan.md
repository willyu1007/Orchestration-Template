# Repo 模板落地方案（阶段3执行计划 · 增强版）
日期：2025-11-19
面向：代码大模型（可直接按步骤从零生成模板）

> 核心导向：**底座优先 → 注册与命名 → 模块化基线 → 路由（知识/功能体） → 数据库子系统 → 接口/合约子系统 → 触发/护栏 → 脚手架与工作文档 → CI Gate → 样例与回归**。  
> 要求：每一阶段设置**硬性验收门槛**，未通过不得进入下一阶段；严格遵循“根目录仅一个 `AGENTS.md` 入口”的规则。

---

## 目标状态目录（供执行时逐步生成）

```
/ (repo root)
├─ AGENTS.md
├─ .editorconfig
├─ .gitignore
├─ LICENSE
├─ Makefile
├─ doc_agent/
│  ├─ ROUTING.md
│  ├─ FUNCTIONS.md
│  ├─ AGENTS.md
│  ├─ guide/
│  │  ├─ ai-friendly.md
│  │  ├─ module-rules.md
│  │  ├─ init-rules.md
│  │  ├─ test-rules.md
│  │  ├─ pr-rules.md
│  │  ├─ commit-branch-rules.md
│  │  ├─ db-rules.md                 # 新增
│  │  └─ api-rules.md                # 新增
│  ├─ registry/
│  │  ├─ capability_registry.yaml
│  │  ├─ agent_registry.yaml
│  │  └─ doc-node-map.yaml
│  ├─ triggers/
│  │  └─ agent-triggers.yaml
│  └─ guardrails/
│     └─ guardrails.yaml
├─ modules/                           # 模块化开发根目录
│  └─ (由脚手架创建实例目录)
├─ db/
│  └─ engines/
│     └─ postgres/
│        ├─ DB_SPEC.yaml             # 数据库设计 SSOT（模板）
│        ├─ migrations/              # up/down 成对迁移文件
│        └─ README.md                # 可选，人类说明（AI 可不读）
├─ api/
│  ├─ openapi/                       # OpenAPI 工件与基线
│  │  ├─ openapi.yaml                # 当前版本（模板）
│  │  └─ baseline/                   # 兼容性基线
│  └─ README.md                      # 可选，人类说明
├─ spec/
│  ├─ front-matter.schema.yaml
│  ├─ capability_registry.schema.yaml
│  ├─ agent_registry.schema.yaml
│  ├─ module_manifest.schema.yaml     # 新增（模块元数据校验）
│  ├─ db_spec.schema.yaml             # 新增（DB SSOT 校验）
│  ├─ triggers.schema.yaml
│  └─ guardrails.schema.yaml
├─ scripts/
│  ├─ route_lint.py
│  ├─ registry_lint.py
│  ├─ function_index_sync.py
│  ├─ module_graph_build.py           # 新增（从模块清单与注册推导关系图）
│  ├─ db_lint.py                      # 新增（DB_SPEC 与迁移结构检查）
│  ├─ db_migration_dry_run.py         # 新增（本地/容器化干跑）
│  ├─ contract_check.py               # 新增（接口兼容性与基线对比）
│  ├─ generate_openapi.py             # 新增（生成/验证 openapi.yaml）
│  ├─ generate_client_types.py        # 新增（可选，前端类型）
│  ├─ trigger_dry_run.py
│  ├─ trigger_conflict.py
│  ├─ guardrail_check.py
│  ├─ doc_node_map_sync.py
│  ├─ scaffold_project.py
│  └─ scaffold_module.py
└─ .ci/
   ├─ dev_check.yml
   ├─ pr_gate.yml
   └─ schedule_guardrail.yml
```

---

## Phase 0 — **底座与约束**
**目标**：建立最小可运转骨架、唯一入口与元数据规范。

**步骤**
1. 新建目录结构与占位文件（见上）。
2. 根 `/AGENTS.md`（唯一入口，选读清单放 front‑matter 的 `ai_optional_docs`）：
   ```yaml
   ---
   audience: ai
   doc_kind: policy
   entry: true
   ai_optional_docs:
     - doc_agent/ROUTING.md
     - doc_agent/FUNCTIONS.md
     - doc_agent/guide/ai-friendly.md
     - doc_agent/guide/module-rules.md
     - doc_agent/guide/init-rules.md
     - doc_agent/guide/test-rules.md
     - doc_agent/guide/pr-rules.md
     - doc_agent/guide/commit-branch-rules.md
     - doc_agent/guide/db-rules.md
     - doc_agent/guide/api-rules.md
   updated: 2025-11-19
   ---
   ```
3. `doc_agent/AGENTS.md`（策略库索引，非入口）：只做策略段落锚点目录。
4. `spec/front-matter.schema.yaml`：约束 `audience|doc_kind|route_role|entry|updated|tags`。
5. `Makefile` 占位：`dev_check route_lint registry_lint function_index_sync trigger_dry_run trigger_conflict guardrail_check`。

**验收门槛**
- 根仅一个 `AGENTS.md`，front‑matter 合规；`doc_agent/` 与 `doc_human/` 存在。

---

## Phase 1 — **注册与命名（两分法+成熟度）**
**目标**：落地**基础能力**与**功能体**双注册，统一 ID 与成熟度；为模块关联预留字段。

**步骤**
1. `spec/capability_registry.schema.yaml` 与 `spec/agent_registry.schema.yaml`：
   - 能力：`id (base.<domain>.<action>)`, `kind`, `inputs|outputs`, `owner`, `tags`。
   - 功能体：`id (func.<domain>.<intent>)`, `type=function`, `entrypoint`, `depends_on: [base.*]`, `maturity: experimental|candidate|stable`, `module_id`, `owner`, `tags`。
2. 初始化空表：`doc_agent/registry/capability_registry.yaml`、`doc_agent/registry/agent_registry.yaml`（各给 1–2 条示例）。
3. `scripts/registry_lint.py`：唯一性/前缀/依赖存在性/成熟度取值校验。

**验收门槛**
- `registry_lint` 全绿；功能体的 `depends_on` 与 `module_id` 字段存在且格式正确。

---

## Phase 1.5 — **模块化基线（实例与关系）**
**目标**：建立模块目录约定、模块清单与自动关系推导（不新增第三本全局注册表）。

**步骤**
1. 模块目录规范：`modules/<mod_id>/`；`mod_id` 规则：`mod.<domain>.<name>`（蛇形或短横线均可，但统一在 lint 中约束）。
2. 模块清单文件：`modules/<mod_id>/MANIFEST.yaml`（受 `spec/module_manifest.schema.yaml` 约束）：
   ```yaml
   mod_id: "mod.<domain>.<name>"
   domain: "<domain>"
   provides_functions: [ "func.<domain>.<intent>" ]
   requires_modules:   [ "mod.<domain>.<dep>" ]
   db: { engine: "postgres", migrations_dir: "db/engines/postgres/migrations" }
   api: { has_contract: true, contract_doc: "modules/<mod_id>/doc/CONTRACT.md" }
   owner: "<team-or-person>"
   ```
3. `scripts/module_graph_build.py`：从所有 `MANIFEST.yaml` 与 `agent_registry.yaml` 推导模块依赖图（输出 `.mmd` 或 `.json`），并校验：
   - `provides_functions` 与 `agent_registry` 一致；
   - 无循环依赖（或标注允许的“选择型互斥”）。

**验收门槛**
- 至少 1 个示例模块通过 `module_graph_build`；MANIFEST 与注册表一致。

---

## Phase 2 — **路由体系（知识/功能体）**
**目标**：构建**最小可达**路由树，确保 AI 能从入口找到知识与功能体说明。

**步骤**
1. `doc_agent/ROUTING.md`：≤ 15 入口，采用 `scope/topic/when`；叶子指向**说明/指南**或 **Quickstart**。
2. `doc_agent/FUNCTIONS.md`：从 `agent_registry` 同步功能体索引（`function_index_sync.py`），包含 `id/intent/entrypoint/depends_on/maturity/module_id`。
3. `scripts/route_lint.py`：校验结构与可达率；禁止策略内容出现在 ROUTING。

**验收门槛**
- `route_lint` 可达率 ≥ 99%；`FUNCTIONS.md` 与 `agent_registry.yaml` 同步。

---

## Phase 3 — **数据库子系统（DB）**
**目标**：提供 DB 设计 SSOT、迁移规范与 Guardrail + Trigger 的最小闭环（以 Postgres 为例，保持可移植）。

**步骤**
1. `spec/db_spec.schema.yaml`：表/列/索引/外键/约束/备注等字段规范。
2. 初始化 `db/engines/postgres/DB_SPEC.yaml`（空模板+示例表）。
3. `db/engines/postgres/migrations/`：规范 `YYYYMMDDHHMM_<name>_up.sql` 与 `_down.sql` 成对出现。
4. `scripts/db_lint.py`：校验 DB_SPEC 结构、命名规范与迁移文件对齐（只做静态检查）。
5. `scripts/db_migration_dry_run.py`：本地或容器化“干跑”最新迁移与回滚（可用 SQLite/容器 Postgres；模板先提供占位实现）。
6. 路由与指南：`doc_agent/guide/db-rules.md`（迁移流程、命名规则、危险操作清单）。
7. 触发/护栏：
   - `agent-triggers.yaml`：`db-migration`（P0, block）——命中 `db/engines/**/migrations/**`；`required_checks: [ "db_lint", "db_migration_dry_run" ]`；
   - `guardrails.yaml`：禁止直删表/列；需要审批（策略锚点由 `doc_agent/AGENTS.md` 提供）。

**验收门槛**
- `db_lint`、`db_migration_dry_run` 通过；迁移成对；P0 触发可阻断未校验合并。

---

## Phase 4 — **接口/合约子系统（API）**
**目标**：提供合约文档、OpenAPI 工件、兼容性基线与 Trigger/Guardrail。

**步骤**
1. 在功能体层面：若 `func.<domain>.<intent>` 属于接口域（如 `domain=api`），在对应模块生成 `doc/CONTRACT.md` 占位。  
2. `api/openapi/openapi.yaml`（空模板）；`api/openapi/baseline/` 存放上一次稳定版本。
3. `scripts/generate_openapi.py`：从 `CONTRACT.md` 或占位注解生成/验证 `openapi.yaml`（模板实现可先校验格式）。
4. `scripts/contract_check.py`：与 `baseline/` 对比，识别 **Breaking**（方法/路径/必填字段删除或改变）。
5. 路由与指南：`doc_agent/guide/api-rules.md`（合约变更流程、版本与兼容策略）。
6. 触发/护栏：
   - `agent-triggers.yaml`：`contract-change`（P0, block）——命中 `**/CONTRACT.md` 或 commit footer `BREAKING CHANGE:`；`required_checks: [ "generate_openapi", "contract_check" ]`。
   - `guardrails.yaml`：Breaking 需审批；非兼容改动必须同步 baseline 更新。

**验收门槛**
- `generate_openapi`、`contract_check` 通过；P0 触发可阻断未校验合并。

---

## Phase 5 — **触发与护栏（总装）**
**目标**：将 DB/API 与通用触发器统一到配置即代码体系，提供干跑、冲突检测与护栏检查。

**步骤**
1. `spec/triggers.schema.yaml`：含 `id/level/enforcement/match/preload_docs/required_checks/target/mutex_group`。
2. `doc_agent/triggers/agent-triggers.yaml`：至少包含 `db-migration`、`contract-change`、`script-added` 三类示例。
3. `trigger_dry_run.py`、`trigger_conflict.py`、`guardrail_check.py` 接入 Make 与 CI。

**验收门槛**
- 干跑通过率 100%；冲突=0；策略锚点可解析。

---

## Phase 6 — **脚手架与工作文档（AI 友好）**
**目标**：以“只注册+就绪文档”为终点的初始化体验；落地 workdocs 模板与 quickstart 叶子。

**步骤**
1. `scaffold_project.py`：问答收集元信息 → 生成入口/路由/注册占位/CI 占位/可选 DB&API 模板。
2. `scaffold_module.py`：生成 `modules/<mod_id>/` 与 `MANIFEST.yaml`、占位 `doc/CONTRACT.md`（可选）、`migrations/`（可选），并自动更新 `agent_registry` 与 `FUNCTIONS.md`。
3. workdocs 模板：`ai/workdocs/active/<task>/*.md`（context/tasks/decisions/lessons）。
4. Quickstart 约定：`route_role: quickstart` 的叶子用于承接新任务/模块。

**验收门槛**
- 运行两脚手架，`dev_check` 全绿；生成的功能体登记完整（含 `module_id` 与 `depends_on`）。

---

## Phase 7 — **CI Gate 与提交流程（A 档）**
**目标**：把规则固化到 CI Gate：单 PR 单模块、跨模块父/子 PR、覆盖率门槛与 Breaking 警戒。

**步骤**
1. PR 模板与 Commit 规则（A 档）落地。  
2. `.ci/dev_check.yml`：聚合路由/注册/触发/护栏/DB/API 的最小检查。  
3. `.ci/pr_gate.yml`：
   - 阻断跨模块路径混改（单 PR 单模块）；
   - `BREAKING CHANGE:` 触发 `contract_check`；
   - `db-migration` 触发 `db_*` 检查；
   - 根据 `agent_registry.yaml` 中 `maturity` 动态应用覆盖率门槛（70%/80%）。
4. `.ci/schedule_guardrail.yml`：每周干跑+冲突检测报告。

**验收门槛**
- 不合规 PR 被阻断；Breaking/迁移必须通过对应检查；覆盖率门槛生效。

---

## Phase 8 — **样例与回归（推荐）**
**目标**：通过一个示例模块与两条功能体（experimental/candidate），验证从脚手架到 PR 的闭环。

**验收门槛**
- 新增→注册→路由→DB/API 触发→PR→合并，全链路一次通过；生成回归报告。

---

## 附：关键 ID 与命名约束（供 lint 使用）
- 基础能力：`base.<domain>.<action>`  
- 功能体：`func.<domain>.<intent>`，`type=function`，必须有 `module_id` 与 `depends_on`。  
- 模块：`mod.<domain>.<name>`；目录：`modules/<mod_id>/`；必须有 `MANIFEST.yaml`。  
- 触发器 ID：`<domain>.<scope>.<action>`；互斥组与优先级必须显式声明。
