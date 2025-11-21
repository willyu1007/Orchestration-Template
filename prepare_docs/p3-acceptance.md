# 验收方案（指标/方法/案例）· 增强版
日期：2025-11-19

## 1. 结构/一致性
**指标**
- 根仅一个 `AGENTS.md`；`ai_optional_docs` 路径均存在。  
- ROUTING 可达率 ≥ **99%**；`FUNCTIONS.md` ↔ `agent_registry.yaml` 100% 同步。  
- 每个 `func.*` 记录具有 `module_id` 与 `depends_on`；指向存在的 `mod.*` 与 `base.*`。  
- 每个模块目录包含 `MANIFEST.yaml` 并通过 schema 校验。

**方法**
- `make route_lint`、`make registry_lint`、`make function_index_sync`、`python scripts/module_graph_build.py --check`。

---

## 2. 模块化开发
**指标**
- `module_graph_build` 无未豁免的循环；`provides_functions` 与 `agent_registry` 完全一致。  
- 单 PR 单模块；跨模块改动采用父/子 PR。

**方法**
- 解析 `MANIFEST.yaml` 与改动路径；CI `pr_gate.yml` 检查路径与引用关系。

---

## 3. 数据库（DB）
**指标**
- `DB_SPEC.yaml` 通过 `db_spec.schema.yaml` 校验；迁移文件 **成对存在**，命名规范。  
- `db_lint`、`db_migration_dry_run` 均通过；`db-migration` 触发为 P0 且可阻断。  
- Guardrail：禁止直删表/列；审批记录写入 workdocs/decisions.md。

**方法**
- `make db_lint && make db_migration_dry_run`；查看干跑报告与 Guardrail 报告。

---

## 4. 接口/合约（API）
**指标**
- `CONTRACT.md` 存在（对接口类功能体）；`openapi.yaml` 生成成功。  
- `contract_check` 与 `baseline` 对比通过；Breaking 需审批。  
- `BREAKING CHANGE:` footer 会触发 `contract-change` P0。

**方法**
- `make generate_openapi && make contract_check`；CI 报告校验触发与审批链路。

---

## 5. 触发/护栏
**指标**
- `trigger_dry_run` 通过率 **100%**；`trigger_conflict` 冲突 **=0**。  
- 所有 P0 触发具 `required_checks`；`preload_docs` 的策略锚点可解析。

**方法**
- `make trigger_dry_run && make trigger_conflict && make guardrail_check`。

---

## 6. 测试/覆盖与成熟度
**指标**
- `candidate` ≥ **70%** 覆盖；`stable` ≥ **80%**；`experimental` 无硬性门槛。  
- `dev_check` 包含 lint/单测/脚本干跑/路由一致性。

**方法**
- CI 动态读取 `agent_registry.yaml#maturity` 设置阈值；不达标阻断合并或降级成熟度。

---

## 7. AI 友好与可恢复
**指标**
- 从 `workdocs/context.md` 恢复到“可继续编码” ≤ **2 分钟**（抽样）。  
- 同类错误在 `lessons.md` 记录后，下一周期复发率下降。

**方法**
- 抽样两次中断/恢复；检查触发器是否预载相应经验条目。

---

## 8. 通过标准与出具记录
1) 各阶段门槛全部通过并打 tag。  
2) 汇总 `dev_check`、DB/API、触发/护栏、模块图等报告到 `ai/maintenance_reports/initial-acceptance.md`。  
3) 若执行样例回归，需一次通过全链路。

---

## 9. 常见不通过与定位
- 路由死链/策略误入 ROUTING → 修正 ROUTING 与链接。  
- `func.*` 缺少 `module_id/depends_on` → 补登记或补能力。  
- 迁移缺对/危险 SQL → 修正迁移、走 Guardrail 审批。  
- 合约 Breaking 未审批 → 添加 `BREAKING CHANGE:` 并走审批后再合入。  
- 覆盖率未达 → 降为 `experimental` 或补测。