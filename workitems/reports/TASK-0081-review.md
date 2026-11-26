# TASK-0081 diff 评审报告（代码评审）

> 评审者：fresh 代码评审子代理（不共享协调者上下文；只依据 brief「必读」与工作区源码判断）。
> 口径：`.agents/protocol/L1-execute.md` §13「diff 评审」（与 TASK 验收标准/brief 接口一致性 + 明显质量缺陷，分级 Critical / Important / Minor）。
> 输入：diff 包（09357d1..b491e34，9 文件）、TASK-0081-brief/report/TASK 详情/plan-review、ADR-0008、L1-execute §13；工作区源码 `D:\workspace\WisFactory`（HEAD=b491e34，工作区仅余 `?? .agents/`）。
> 独立实测：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` 复跑 → **Tests run: 2384, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**（与执行者报告一致）。

---

## 一、发现清单

### I-1（Important）— 非法 `defaultCalcPeriod` 新默认被静默写为 0（到达驱动），违反 fail-closed 不变量

- 出处：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotUpgradeApplyServiceImpl.java:250-255`、`:525-530`（`safeMapPeriod`）
- 说明：`safeMapPeriod` 捕获 `mapCalcPeriod` 的 `RuntimeException` 后返回 `null`，而 T2 分支对**新默认** `newSeconds == null` 直接写 `patch.setCalcPeriod(0)`——正是 `mapCalcPeriod`（`IotDataItemServiceImpl.java:1156-1157`）注释明令禁止的「配了周期却被当成到达驱动」语义漂移。触发条件可达：模板项 `defaultCalcPeriod` 仅受 `@Size(max=16)` 校验（`IotDataItemTemplate.java:92-94`），无格式校验，值如 `WEEKLY`/`MINUT` 可入库并经 `buildContentJson` 写入版本 content；确认后实例 `calc_period` 由周期驱动静默变 0（到达驱动），计算语义被改。0013 生成链在同样坏码下是整批失败（fail-closed），升级链却静默降级——同源换算范式行为不一致。建议：旧默认坏码保留「不误判为覆盖」的宽容（注释本意），但**新默认坏码应抛异常整批回滚**，不得写 0。

### m-1（Minor）— 可复用接缝 `applyToVersion` 存在隐藏会话/权限上下文耦合

- 出处：`IotUpgradeApplyServiceImpl.java:153`（T1 调 `dataItemService.regenerateForInstance`）；`IotDataItemServiceImpl.java:365-367`（`regenerateForInstance` 内 `dataPermService.resolve(SecurityUtils.getUserId())`，不可见即 403）
- 说明：brief 重点 6 要求核实接缝「无隐藏耦合（如依赖确认接口上下文/会话态）」。`applyToVersion(instance,target)` 签名干净，但 T1 路径经 `regenerateForInstance` 间接依赖当前登录用户（`SecurityUtils.getUserId()`）与租户上下文（`TenantContext.getTenantId()`）。确认链路已同用户鉴权，当前无缺陷；但供 TASK-0083 发布自动跟随复用时，若发布侧无用户会话或权限模型不同，可能触发 403/NPE。**对 0083 是否实际致害：无法判定**（0083 未实现、不在本评审范围）；耦合事实本身成立，建议 0083 设计时显式核实或为接缝提供免会话的补插入口。

### m-2（Minor）— T1 逐台结果无条件报 rowUpdated=true / 「补插」成功，未核对 regenerate 返回值

- 出处：`IotUpgradeApplyServiceImpl.java:151-156`
- 说明：`addedCodes` 非空即调用 `regenerateForInstance` 并写死 `rowUpdated=true` + 「新增项补插」动作，未取返回值。`regenerateForInstance` 按占用幂等跳过（含逻辑删行永久占用键，`IotDataItemServiceImpl.java:215-227`）：若新增 data_code 已被逻辑删行占用，实际返回 0、不插行，结果却报补插成功。数据面「跳过占用」是裁决明示的设计（接口 javadoc 已登记），此处仅为**逐台留痕 VO 的准确性瑕疵**（rowUpdated/actions 与实际行变更不符），不影响数据正确性；建议以 `regenerateForInstance` 返回值决定是否置 rowUpdated 与动作文案。

---

## 二、brief 评审重点逐条核对

1. **开工复核性实证（四项）— 一致，无偏差**。①`buildContentJson`（`IotTemplateVersionServiceImpl.java:432-446`）九字段齐备、`:488-494` `readTree` 可解析；②`mapCalcPeriod`（`IotDataItemServiceImpl.java:1159-1184` null/空白→0、SECOND/MINUTE/HOUR/DAY→1/60/3600/86400、坏码抛错）、`RelativeRefExpander.expand`（:86-130 裸键展开+引用收集）、`buildItem`（:1099-1100 null→1 兜底）三者与裁决逐字相符；③双 UK `uk_tenant_instance_data`/`uk_tenant_full_key` 均不含 deleted（`resources/sql/upgrade/20261122_iot_data_item_domain.sql:94-95`），`@TableLogic`（`BaseEntity.java:38-39`）+ 全局配置（`application.yml:181-183`），逻辑删行永久占键机制成立；④`ConfigSnapshotAssembler.assembleComputeScope`（:191-201）`selectList` 读 iot_data_item 行 + 引用边，查询未涉及 applied 列（全文 applied 仅 javadoc 版本语义说明）。

2. **T2 三方合并正确性（最关键）— 三形态实现齐全，未发现误覆盖用户已改值**。公式按本实例展开比对（`applyFieldChanges` defaultFormula 分支：`expandFormula(oldTemplate)` 与行值比对，空或相等才写新默认展开值）、calc_period 经 `mapCalcPeriod` 换算比对（:251-253）、source_type 复现 null→1 兜底（:259-263，`oldFallback`/`newFallback` 的 `==null?1`）；三形态命中即写、否则保留，白名单更新集仅写本轮判定列（:372-373 调 `lambdaUpdate`，方法体 :388），未覆盖→写常量/覆盖→保留方向正确。已覆盖保留路径有单测 `t2KeepOverriddenFormula` 断言零写。唯一缺口是 I-1（新默认坏码写 0，属「写入错误值」而非「误覆盖」，单列发现）。

3. **T1 补插 — 正确**。ADDED 数据项委托 `regenerateForInstance`（`IotUpgradeApplyServiceImpl.java:152-153`），走 0013 生成范式（相对引用展开 `RelativeRefExpander` + 引用边落库 `insertRefs`），按占用幂等跳过已存在 (instance_id,data_code)（含逻辑删行，`generateForInstance` :215-227）。补插不新建入边、不改写他行公式 → 新项默认不被上级汇总引用，ADR-0008 T1「默认不参与汇总」保险成立（平台无显式「参与汇总开关」字段，本任务未新增，符合不越界）。留痕准确性瑕疵见 m-2。

4. **T3 逻辑删迁移 — 正确**。REMOVED 项 `SET deleted=1`（:168-170）而非物理删；被删项**自身出边**随之清理（:172-173），入边与指标引用不清理（无对应清理代码，actions 文案明示，ADR 明示后果）；改类型/改取值方式走结构迁移（defaultSourceType 分支 :259-283 重写 source_type/formula/rollup + 边先删后插/清公式只删不插，`formulaCleared` 标志见 :270-280、边重建 :376-382）。双 UK 不含 deleted 使逻辑删行永久占键——**机制在 8.0.12 下单据成立**：DDL 已核实 UK 定义不含 deleted；live `deleted=1`（报告 STEP8 行 38）与快照仅含活跃行（v9 grep）一致。MySQL 8.0.12 实机索引行为的现场状态属执行者 live 证据，源码/DDL 层机制无反证。

5. **事务与失败语义 — 正确**。`confirmUpgrade` 单 `@Transactional(rollbackFor=Exception.class)`（`IotInstanceUpgradeServiceImpl.java:236`），`applyToVersion` 无自身事务注解（加入调用方事务，已 grep 确认）；调用序为「先 apply 后簿记 UPDATE」（:304 → :307-311），乐观锁 rows==0 映射 4002（:312-316），任一失败整批回滚。单测 `confirmWiresDataPlaneApply`（InOrder）与 `confirmDataPlaneAppliedButOptimisticConflictRollsBack` 实证无半态。越权/无效 id/版本校验均在 apply 前 fail-closed（:238-280）。

6. **可复用接缝 — 方法面可复用，但见 m-1**。`applyToVersion(IotModelInstance,Integer)` 按实例维度封装完整三档语义，不依赖确认接口的 form/返回上下文；`applyToVersionBatch` 逐台调用、结果顺序与入参一致（单测实证）。确认链路只调 applyToVersion；**发布侧接线未实现**：`git diff 09357d1..b491e34 | grep -i publish` = NONE，`applyToVersion` 全仓仅确认链路调用。隐藏会话耦合见 m-1。

7. **留痕过渡口径 — 正确**。逐台结果 VO（`TemplateUpgradeResultVO`：diff/actions/rowUpdated/tierSummary）、逐台 `[UPGRADE-APPLY]` INFO（`IotUpgradeApplyServiceImpl.java:200-201`）、控制器请求级 `@OperateLog`（`IotTemplateUpgradeController.java:66`）；diff 9 文件无建表/无 iot_data_item_history 写路径/未改 0014 NoOp（越界反查确认）。

8. **live 哈希变化证据 — 一致**。快照文件实存：`backend/ot-admin/data/snapshots/1/2-0-8.json`、`2-0-9.json`；grep 实证 v8 两台均 `/100`、v9 同现 `F01.WS13.E8A.ELEC_CONS/1000`（1×）与 `F01.WS13.E8B.ELEC_CONS/100`（1×）——确认前 ad7e6471…(v8)→确认后 64fdb216…(v9) 的变化与「E8A 新值/E8B 旧值并存」证据成立；单测 `t2InheritNewFormulaDefault` 断言更新集含 `/1000`，与 live 证据对应。

9. **2384 测试零回归 — 复跑确认**。独立执行验收 1 命令得 2384/0/0/0 BUILD SUCCESS；既有确认/预览/发布测试未被接线破坏（报告 TDD 红 9→绿、既有 17+5 例适配 List 返回值零回归）。

## 三、越界反查复核（协调者已比对，本评审独立核实）

- `git diff --name-only 09357d1..b491e34` = 9 文件，全落 brief 声明的 template/** + dataitem/** + src/test/** 范围；`grep -i publish|resources/sql|.yml|ot-core|frontend|ingest|calc` 输出 NONE——组装器/SQL/yml/ot-core/前端/ingest/calc 零触碰。
- `mapCalcPeriod` 仅 `static`→`public static` 可见性放宽：BASE 签名 `static int mapCalcPeriod`，HEAD `public static int mapCalcPeriod`；方法体（不含签名行）逐行 diff 为空（md5 一致），**无其他语义改动**，执行者声明属实。

---

## 四、结论

**不通过（退回一轮）**：存在 1 项 Important（I-1：非法新默认 calcPeriod 静默写 0，违反 fail-closed，可致实例计算语义漂移），按 §13 进 §12 退回路径由执行者修订；另有 2 项 Minor（m-1 复用接缝会话耦合、m-2 T1 留痕准确性）登记，修订轮可一并处理。其余 9 项评审重点与越界反查均核实一致，2384 测试独立复跑全绿，改动范围无越界。复评为 scoped：仅看 I-1 修复 diff 与 m-1/m-2 是否采纳，不重开全场。
