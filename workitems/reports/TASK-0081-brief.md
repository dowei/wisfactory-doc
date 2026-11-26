# 实施 brief：TASK-0081（确认升级数据面生效编排——用户已确认开工）

> 用户 2026-11-23 放行。计划评审已过（reports/TASK-0081-plan-review.md），三项机制裁决已写进 TASK 正文——执行者开工只做复核性实证，**不再开放裁决**。

## 批次

- TASK-0081（本批共 1 个，标准任务 3 人天；backend 串行道——FEAT-0005 三档语义最后一环）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/**`（确认编排接线：IotInstanceUpgradeServiceImpl 或新编排服务+逐台结果 VO）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/**`（**仅复用** 0013 生成/展开/边重建组件与逻辑删——不改组件语义）
  - `backend/ot-admin/src/test/java/**`（编排用例）
  - **publish/snapshot 组装器零改动**（裁决①行态承载）；无 SQL/yml/ot-core；不碰前端/ingest/calc
  - D-1（发布自动跟随）已由用户裁决拆出 TASK-0083（前置=0081）——**本任务不实现发布侧接线**，仅按正文把 applyToVersion 留为可复用边界

## 必读（只给路径，不粘正文）

- TASK 详情（**正文含三项裁决全文，务必通读**）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0081.md`
- 计划评审报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-plan-review.md`
- 决策权威：`decisions\ADR-0008.md`（三档政策原文：T1 自动/T2 确认+锁定/T3 逐台迁移+历史保留）
- 上游交付：`reports\TASK-0016-report.md`（确认接口现状：仅簿记 applied_template_version）+ `reports\TASK-0015-report.md`（版本历史/content 基线/buildContentJson）+ `reports\TASK-0036-report.md`（0013 生成范式/引用边重建）
- 上游代码（开工先读）：iot/template/**（IotInstanceUpgradeServiceImpl.confirm 路径、IotTemplateVersionService.buildContentJson/版本历史行）+ iot/dataitem/**（IotDataItem 行字段、regenerate 范式、FormulaCycleDetector/引用边）+ iot/publish/snapshot/ConfigSnapshotAssembler.java（读实例行——裁决①实证点）

## 接口与决策（逐条——TASK 正文裁决，不再讨论）

1. **开工复核性实证**（三项裁决对照源码，偏差即停写报告）：①基线 content 可解析且字段齐备（defaultFormula/defaultSourceType/defaultCalcPeriod/defaultValidMin/Max/defaultAlarmLow/High/aggMethod/valueKind）②`mapCalcPeriod` 换算/公式实例展开/兜底 null→1 复现正确 ③逻辑删双 uk 不含 deleted（键永久占用防复用污染）④组装器读实例行不读 applied 列（裁决①成立）
2. **T1 补插**：确认升级时对 applied→target 版本新增的数据项按 0013 生成范式补插缺失行（相对引用展开+引用边落库）
3. **T2 三方合并应用**：行值为空或等于旧默认（公式按本实例展开比对、calc_period 经 mapCalcPeriod 换算比对、source_type 复现 null→1 兜底）→ 视为未覆盖，写新默认+公式展开+边重建；否则保留实例已覆盖值。状态规则类 T2 无实例行载体——不经本任务（0015 遗留 #3）
4. **T3 逐台迁移**：删项=逻辑删（@TableLogic 退出快照、保留历史行自身）；改类型/改取值方式=结构迁移+边重建；入边与指标引用不清理（ADR 明示后果，逐台 diff 明示）；每台出 diff 与迁移结果
5. **同步并入确认事务**：簿记+行应用+留痕单原子单位；任一实例失败整批回滚+4002 口径；不建异步状态机
6. **留痕过渡口径**：不建表/不写 iot_data_item_history（归 0039）/不改 0014 NoOp——逐台结果=响应逐台 VO+逐台 INFO 日志+@OperateLog 请求级
7. **applyToVersion(instance, target) 可复用边界**：按评审设计留接缝，供 TASK-0083（发布自动跟随）复用——本任务不实现发布侧接线
8. **边界四类**：空=零操作；越权=确认防线 403 不变；并发=整批原子+4002；超限=一次升版实例数分批口径

### 测试口径（TDD）

T2 三方合并（未覆盖继承新默认/已覆盖保留/公式展开比对/calc_period 换算比对/source_type 兜底）；T1 补插；T3 逻辑删+结构迁移+边重建；快照组装体确认前/后字段值+哈希变化断言；四类边界（空/越权/并发/超限）；既有确认/预览/发布测试零回归。红转绿留痕。

## 验收命令

环境事实（2026-11-23 实测）：WSL docker 容器已停——MySQL/Redis 在**宿主 phpStudy**：MySQL 8.0.12@3307（root/root）/Redis 3.0.504@6379（实测读写正常）；sql-exec.sh 每条语句单独一次调用；8098 prod 启动回归用（探测/用毕即停）；TDengine 本批不涉。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. dev 冒烟（每步原始输出贴回）：模板 T2 变更升版 → 预览 → 确认升级 → **确认前发布取 snapshotHash → 确认后发布取 snapshotHash（变化即数据面生效实证）** → 锁定实例保持旧口径且仍在清单 → T3 逐台迁移 diff 与结果 → 四类边界断言
3. prod 档真实启动回归 → 0 ERROR
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内（组装器/SQL/yml/ot-core 零触碰实证）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-report.md`。五项齐全 + 开工复核性实证结论 + 一句话结论。

## 契约

- no-subagents；不改范围外文件（组装器/SQL/yml/ot-core/前端/calc 零触碰）；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」；允许 git commit（main，注明 TASK-0081）。
