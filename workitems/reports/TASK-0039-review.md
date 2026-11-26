# TASK-0039 代码评审报告（diff 评审）

> 评审对象：提交 `57aeb36`（BASE=`7886325`，单提交、区间干净，21 文件 / 1656 行），即 `TASK-0039-diff.txt` 隔离 diff。
> 口径：L1-execute §13「diff 评审」；独立 fresh 评审，全程只读源码，未改任何代码、未推进状态、未派子代理。
> 本报告为本次任务唯一写入文件。

## 结论

**通过**（无 Critical / Important，标准首评一次通过；1 条 Minor 记 deferred）。

四类 change_type（BIND/FORMULA/ROLLUP/PARAM）齐全、PARAM 未重复接线、before 值均在 update 前读快照取得、写点与业务落库同事务、项删除后按 full_key 仍可查、跨租户隔离、读不审、DDL 只产出未执行——均经源码独立核实成立。3 个声明范围外连带文件改动均经独立判定为必需且最小，无测试弱化。

## 评审范围与依据

- 逐行读完 21 文件 diff；对写点、真实实现、查询端点、3 个连带文件回到源码独立核实，未采信报告自述作为结论依据。
- 独立复跑验收命令 `cd backend && mvn -pl ot-admin test`：`Tests run: 2534, Failures: 0, Errors: 0, Skipped: 0` / `BUILD SUCCESS`（与实施报告一致；基线 2516 + 净增 18）。
- 新增 5 个测试类共 20 个 `@Test`，删除 NoOp 测试 2 个，净增 18 —— 与报告数字吻合，非凑数。
- 范围核对：`git diff --name-only 7886325..HEAD` 恰为报告 §1 的 21 文件；`frontend/`、`ingest/`、`calc/`、`resources/contract/` 及既有 publish/inspect/alarm/calcread/statusread/template 零触碰；全仓 `grep NoOpIotDataItemHistoryService` 零命中。

---

## 评审重点逐条判定

### 1. 四类 change_type + 同事务 + before 时序 + PARAM 唯一接线 —— 通过

| 项 | 判定 | 源码出处（绝对行号，HEAD 工作树） |
| --- | --- | --- |
| PARAM 唯一接线 | 通过 | `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\dataitem\service\impl\IotDataItemServiceImpl.java:466` 唯一调用；对比 `git show 7886325:...` 基线同为 `:462`（仅因新增 import/字段上移 4 行）。全仓主代码 `recordParamChange(` 仅 3 处：接口声明、真实实现、该唯一调用点。 |
| BIND 写点 | 通过 | 同文件 `:671` `@Transactional`；`:690-696` **在 update 前**由 `item.pointId` 反查旧点得到 `beforePointCode`；`:706` 落库、`:711` ref 重建、`:713` 发事件、`:718` `recordBindChange`（after=`point.getPointCode()`）。 |
| FORMULA 写点 | 通过 | 同文件 `:727` `@Transactional`；`item` 由 `:732` `requireItemOfInstance` 取得（`:1072-1079` 直接返回 `selectById` 结果，`this.update(new IotDataItem(), wrapper)` 不修改该对象），故 `:877-879` 的 `item.getFormula()` 确为**变更前值**；after=`expanded` 与 `:860` 落库值一致。 |
| ROLLUP 写点 | 通过 | 同文件 `:885` `@Transactional`；`item` 于 `:890` 取得、`:905` 落库用新空实体，`:915-917` 的 `item.getRollupMethod()` 为旧值、after=`rollupMethod`（SUM/AVG 白名单校验于 `:891`）。 |
| 同事务性 | 通过 | 四个写点均 `@Transactional(rollbackFor=Exception.class)`；`IotDataItemHistoryServiceImpl:51` 注入**无限定符**的 `JdbcTemplate`——`CalcReadDataSourceConfig` 的 `calcReadJdbcTemplate` 是具名 bean 且不参与 `@Primary`，故实际注入的是 `@Primary dynamicDataSource`（`ot-core/.../DynamicDataSourceConfig.java:32-34`）自动装配的 JdbcTemplate，与 MyBatis 共用同一事务绑定连接。`IotDataItemHistoryServiceImplTest.rollsBackWithBusinessSave` 用真实 `TransactionTemplate` 在 H2 上证明回滚后 `COUNT(*)=0`；`IotDataItemHistoryWiringTest` 三失败路径（点位不存在/空公式/非法汇总）均 `verify(..., never())`。 |

**结论：四类齐全、缺一不可的要求满足；PARAM 未重复写；before 为 update 前快照而非后验伪造；同事务性有源码结构 + 契约测试双重支撑。**

### 2. 真实实现与 NoOp 移除 —— 通过

- `IotDataItemHistoryServiceImpl`（新文件）用 JdbcTemplate 显式带 `tenant_id` 落库（`:112-122`），查询 `list()`（`:86-116`）拼接 `WHERE tenant_id = ?` + itemId/fullKey 二选一 + 可选时间窗 + `ORDER BY change_time DESC, id DESC` + `LIMIT/OFFSET`（pageSize 封顶 200）。
- NoOp 实现及其测试均为删除（D），符合 brief「真实实现上线后移除 NoOp」；删除后全仓零残留引用，独立 `mvn test` 全绿证明所有调用方/测试编译通过。
- 接口扩展四 record 方法 + `history` + `list`，载荷字段完整（itemId/fullKey/operatorId/before/after，三个新 record 另带 changeSummary）。PARAM 载荷沿用既有 `IotDataItemParamChange`（含 Thresholds）。

### 3. 查询端点（D2）—— 通过

- `IotDataItemController.java:150-167`：`GET /api/v1/iot/dataitem/history`，`@SaCheckPermission("iot:instance:query")`；两者都不给时 `:161` 抛 `BusinessException(ResultCode.PARAM_ERROR...)`，`ResultCode.PARAM_ERROR=400` 已核。
- 按 itemId 或 fullKey 双定位：`IotDataItemHistoryServiceImpl.list():100-108` 的 `if/else if/else throw`；full_key 为 NOT NULL 冗余列，测试 `orderedFilteredAndByFullKey` 插入同 fullKey、不同 itemId（200/999）三行按 fullKey 全查回——**项删除后按 full_key 仍可查**有断言。
- tenant 从会话强制注入：`:109` `TenantContext.getTenantId()` 组装 Query，请求无 tenant 参数；`tenantIsolation` 测 tenant=1/2/999 分别 1/1/0 行。
- 倒序 + 时间窗 + 分页在 `list()` 与 `IotDataItemHistoryServiceImplTest` 均有断言；VO 含 changeType/changeTypeText/beforeJson/afterJson/operatorId/operatorName/changeTime。
- **读不审**：端点无 `@OperateLog`，`IotDataItemHistoryControllerTest.endpointShape` 反射断言 `assertNull(...getAnnotation(OperateLog.class))`。

### 4. 边界（D3）—— 通过

- 不双写 sys_operate_log：四条 record 方法只 INSERT `iot_data_item_history`（`IotDataItemHistoryServiceImpl:112-122`），无 operate_log 写入；写点既有 `@OperateLog` 保持。
- 发布动作不入时间线：时间线仅在 bindPoint/saveFormula/saveRollup/overrideParams 四个保存写点追加，publish/regenerate 等发布路径未接 record。
- 与 `DataItemConfigChangedEvent` 并存：三个写点均 `publishEvent` 后才 `recordXxx`，事件与 history 互不替代。

### 5. 测试有效性与零回归 —— 通过

- 独立复跑 2534/0/0（基线 2516 + 18），既有测试零回归；20 个新 `@Test` 减 2 个删除的 NoOp 测试 = 净增 18。
- TDD 红转绿留痕见报告 §3，根因逐条对应（final 类继承、VO 形态、TenantContext mock、DDL 反引号、权限守卫例外），非改期望凑输出。
- 四类写入（`allFourChangeTypesPersisted` 断言落库集合恰为 `{BIND,FORMULA,ROLLUP,PARAM}`）、同事务回滚、倒序、删除后按 full_key 可查、跨租户隔离均有真实断言。
- 调试清理：新增主代码无 `System.out/printStackTrace/TODO/FIXME`；`IotDataItemHistoryServiceImpl:122` 仅一条 `log.debug("[ITEM-HISTORY] ...")` 属既有 `[ITEM-HISTORY]` 追踪范式留痕，非调试残留。

### 6. DDL 与范围 —— 通过

- `resources/sql/upgrade/20261124_iot_data_item_history.sql` 单条可执行语句（`CREATE TABLE IF NOT EXISTS`），逐行独占、无同行多分号；`IotDataItemHistoryDdlSplitTest` 复刻 `sql-exec.sh` 的 `;\n` 切分验证（1 段、0 同行多语句）并校验表/四类枚举/full_key/倒序索引齐备。
- 脚本头部明文「不在用户库执行」；实施报告 §2.2 声明未在 phpStudy MySQL(8.0.12@3307) 执行，H2 内存库自建同构表验证读写——符合「只产出不执行」。
- 索引 `idx_tenant_fullkey_time (tenant_id, full_key, change_time DESC)` 与 `idx_tenant_item_time` 支撑双定位倒序查询。

---

## 越界反查：3 个声明范围外连带文件独立判定

不因「连带」默认放行，逐个核实编译必需性、最小性与测试弱化：

**① `D:\workspace\WisFactory\backend\ot-admin\src\test\java\net\otlab\admin\module\iot\redline\RedlineFixture.java`（M）—— 判定：必需、最小、无弱化。**
NoOp 类被删除后，该夹具原 `new NoOpIotDataItemHistoryService()` 必然编译失败，属删除 NoOp 的直接编译连带。改动为：import 由 `NoOpIotDataItemHistoryService` 换成 `IotDataItemHistoryService`（`mock(...)` 静态导入文件内已存在，`:91`），`:301` 改为 `mock(IotDataItemHistoryService.class)`。红线用例本不触达时间线，mock 与 NoOp 语义等价（均不产生副作用），无断言放宽、无用例删除。

**② `D:\workspace\WisFactory\backend\ot-admin\src\test\java\net\otlab\admin\module\iot\dataitem\config\IotDataItemMenuPermissionGuardTest.java`（M）—— 判定：必需、最小、无实质弱化。**
brief D2 明文要求历史端点用 `iot:instance:query`，与旧域内守卫「权限码 ⊆ 5 个 `iot:item:*`」冲突，不改守卫则新端点必红。改动为：新增 `DOCUMENTED_EXTRA_PERMS = Set.of("iot:instance:query")`（唯一例外，`:61`），两处 subset 断言在五值集合上并该单码（`:88-92`、`:119-121`），并**新增**历史端点精确锚点断言（`:113-117`）作为收窄补偿——不是放宽后放任，而是「单一码 + 精确锚点」替代原防线。`iot:instance:query` 已在 `20261122_iot_instance_columns.sql:92` 与 alarm/inspect 脚本登记，全局 `IotPermissionsCompleteGuardTest` 的「注解 ⊆ 菜单全集」仍绿。无其他权限码被放开、无既有断言删除。

**③ `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\dataitem\service\IotDataItemService.java`（M）—— 判定：必需、最小。**
控制器 `history()` 经 `IotDataItemServiceImpl.history()`（`:1005-1011`）委托，接口必须声明 `IotItemHistoryVO.Page history(...)` 才能通过 `IotDataItemService` 类型调用；仅新增该一个方法声明 + import，无既有签名变更、无逻辑改动。

**总判定：3 个连带文件均由「删除 NoOp / D2 权限决策 / 查询端点分层委托」直接触发，改动仅限 import + 最小代码行，无测试弱化。予以放行。**

---

## 发现清单

| 编号 | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| F-1 | Minor（deferred） | `IotDataItemHistoryServiceImpl.java:513-535` `count()` | `count()` 在 itemId/fullKey 均空时不抛错、返回该 tenant 全量计数，与 `list():493-495` 的「二选一否则抛」不一致；但当前仅由 `history():546,553` 调用且 `list()` 必先执行并抛错，控制器另有一层校验，该分支**实际不可达**，不影响本次验收；建议后续对 `count()` 补同一校验以保持防御一致性。 |

无 Critical / Important 发现。F-1 为不可达的健壮性建议，记 deferred，不构成本次退回理由。

## 评审动作

首评通过，不退回、无需复评。按契约：本评审只产出报告，不改代码、不推进 TASK 状态、不改 workitems 索引、不派子代理。
