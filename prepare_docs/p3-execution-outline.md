# 执行纲要（面向执行引擎/编排器）· 增强版
日期：2025-11-19

## 一、顺序与里程碑
1) **Phase 0 底座** → 入口/元数据/Make/CI 占位。  
2) **Phase 1 注册** → capability/agent 双注册 + 成熟度。  
3) **Phase 1.5 模块化** → 模块目录/清单/关系推导（不增第三本全局表）。  
4) **Phase 2 路由** → 知识 ROUTING + 功能体 FUNCTIONS（自动同步与 lint）。  
5) **Phase 3 数据库** → DB SSOT + 迁移规范 + `db_*` 检查 + P0 触发/护栏。  
6) **Phase 4 接口** → CONTRACT + OpenAPI + `contract_check` + P0 触发/护栏。  
7) **Phase 5 触发/护栏总装** → 配置即代码 + 干跑/冲突/护栏检查集成。  
8) **Phase 6 脚手架/工作文档** → `scaffold_*` + workdocs + quickstart。  
9) **Phase 7 CI Gate** → 单 PR 单模块、覆盖率门槛、Breaking 警戒。  
10) **Phase 8 样例回归** → 全链路一次通过。

## 二、关键断言（失败即停）
- `AGENTS.md` 唯一入口且 `ai_optional_docs` 可解析。  
- `agent_registry.yaml` 每条功能体具 `module_id` 与 `depends_on`。  
- 每个模块具 `MANIFEST.yaml`；`module_graph_build` 无循环或有明确豁免。  
- DB：迁移成对、`db_lint/db_migration_dry_run` 通过。  
- API：`generate_openapi/contract_check` 通过；Breaking 需审批。  
- ROUTING 可达率 ≥ 99%；FUNCTIONS ↔ registry 同步。

## 三、输入/输出（供脚手架/校验用）
- **输入**：`domain/intent/module_id/owner/tags/depends_on/maturity`；（DB/API 可选项）。  
- **输出**：注册记录、`FUNCTIONS` 索引项、模块 `MANIFEST.yaml`、可选 `CONTRACT.md` 与迁移目录。

## 四、回滚与幂等
- 每阶段通过后打 tag；变更命名/路径在 Phase 7 前统一合入；脚本设计为幂等（重复执行不产生重复项）。

## 五、与决策一致性
- 入口唯一与两套路由 ✅；功能体只在 `FUNCTIONS` 暴露 ✅；两分法注册 + 成熟度 ✅；PR/Commit A 档与硬规则 ✅；模块化、数据库、接口已纳入各自阶段 ✅。
