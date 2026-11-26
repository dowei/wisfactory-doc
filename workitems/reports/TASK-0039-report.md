# TASK-0039 实施报告：iot_data_item_history 数据项变更时间线

- BASE：`7886325`
- 提交：`57aeb366f3683add1425bff618502f6dc3f49ece`（`57aeb36`）
- 结论（一句话）：四类 change_type（BIND/FORMULA/ROLLUP/PARAM）已在各自写点同事务落时间线，倒序查询端点按 itemId / fullKey 均可查（项删除后按 full_key 仍可查），tenant 会话强制注入跨租户空集，DDL 只产出未执行，全量 `mvn -pl ot-admin test` 2534/0/0 通过。

## 1. 改动文件清单（21 个，`git diff --name-only 7886325..HEAD`）

后端主代码（`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/`）：
1. `controller/IotDataItemController.java`（M）— 新增 `GET /api/v1/iot/dataitem/history`，权限 `iot:instance:query`，读不审
2. `dto/IotItemHistoryVO.java`（A）— 行 VO + 分页 `Page`
3. `entity/IotDataItemHistory.java`（A）— 时间线行实体（无审计四件套）
4. `service/IotDataItemHistoryQuery.java`（A）— 查询条件（tenantId 强制注入）
5. `service/IotDataItemHistoryService.java`（M）— 扩展 BIND/FORMULA/ROLLUP 三条 record + list/history
6. `service/IotDataItemService.java`（M）— 新增 `history(...)` 委托接口
7. `service/NoOpIotDataItemHistoryService.java`（D）— 占位实现删除
8. `service/impl/IotDataItemHistoryServiceImpl.java`（A）— 真实实现（JdbcTemplate 显式 tenant_id，写入与查询）
9. `service/impl/IotDataItemServiceImpl.java`（M）— 三个写点接线 + history 委托 + 过时注释更新
10. `service/record/IotDataItemBindChange.java`（A）
11. `service/record/IotDataItemFormulaChange.java`（A）
12. `service/record/IotDataItemRollupChange.java`（A）

后端测试（`backend/ot-admin/src/test/java/`）：
13. `net/otlab/admin/module/iot/dataitem/config/IotDataItemMenuPermissionGuardTest.java`（M）— 权限守卫加入 TASK-0039 D2 明文例外
14. `net/otlab/admin/module/iot/dataitem/controller/IotDataItemHistoryControllerTest.java`（A）
15. `net/otlab/admin/module/iot/dataitem/service/NoOpIotDataItemHistoryServiceTest.java`（D）
16. `net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemHistoryDdlSplitTest.java`（A）
17. `net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemHistoryQueryTest.java`（A）
18. `net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemHistoryServiceImplTest.java`（A）
19. `net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemHistoryWiringTest.java`（A）
20. `net/otlab/admin/module/iot/redline/RedlineFixture.java`（M）— **范围外必要连带修复**（见 §4）

资源：
21. `resources/sql/upgrade/20261124_iot_data_item_history.sql`（A）— DDL，只产出不执行

## 2. 验收命令原始输出

### 2.1 `cd backend && mvn -pl ot-admin test`（原样命令）

```
[INFO] Results:
[INFO] 
[INFO] Tests run: 2534, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:05 min
[INFO] Finished at: 2026-11-24T10:25:25+08:00
[INFO] ------------------------------------------------------------------------
```

BASE 基线对照：改动前同命令 2516/0/0；改动后 2534/0/0（新增 18 条契约用例，净增 18）。

### 2.2 DDL 静态检查（未连用户库）

独立复刻 `sql-exec.sh` 的 `;\n` 切分：
```
executable segments: 1
segments with inline semicolon: 0
```
- 唯一可执行语句=建表 `CREATE TABLE IF NOT EXISTS iot_data_item_history`；无同行多语句。
- 契约测试 `IotDataItemHistoryDdlSplitTest` 2 项（逐行切分 + 关键字段/索引/四类枚举齐备）均绿。
- **声明：本 DDL 未在用户 phpStudy MySQL（8.0.12@3307）执行**，仅产出脚本供部署方按环境执行。测试用 H2 内存库自建同构表验证读写契约，不触用户库。

### 2.3 范围核对

`git diff --name-only 7886325..HEAD` 输出见 §1；`frontend/`、`ingest/`、`calc/`、`resources/contract/` **零改动**；既有 publish/inspect/alarm/calcread/statusread/template 模块零改动；`NoOpIotDataItemHistoryService` 已删除且全仓无残留引用。

## 3. TDD 红转绿（每修复附「为什么现在对了」）

| 步骤 | 现象 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- |
| RED 1 | 5 个新测试类编译失败：找不到 `IotItemHistoryVO` / `IotDataItemHistory` / 三个 record / `IotDataItemHistoryQuery` / 真实实现 | 按契约新增全部类型 | 先写测试锁接口，缺失类型即红，绿时签名与调用方一致 |
| GREEN 1 编译级联 | 首个硬错 `IotItemHistoryVO` 为 `final` 但 `Page` 继承它，触发 Lombok 注解处理中断，全仓大量「找不到符号」 | 去掉 `final` | 单一根因解除后级联误报全消——不是几十个 bug，是一个继承约束 |
| GREEN 1 VO 形态 | 测试按「行 VO」直接 `new IotItemHistoryVO()` + `setXxx`，实现初版写成 `rows + total` 容器 + 嵌套 `Row` | 统一 VO 为「行字段 + 静态 Page 封装」 | 测试与实现同一数据模型，序列化路径 `$.data.rows[0].changeType` 自然成立 |
| GREEN 1 缺导入 | 测试类缺 `TenantContext`/`SecurityUtils` import | 补导入 | 静态 mock 是写点 operatorId / 显式 tenant 的前提 |
| GREEN 2 tenant 空 | H2 写入报 `NULL not allowed for column TENANT_ID` | 测试 setUp 增加 `MockedStatic<TenantContext>` 返回 TENANT | 实现从会话显式取 tenant（跨租户隔离的正确行为）；测试补会话即还原真实 Web 场景 |
| GREEN 2 DDL 断言 | `CHANGE_TIME DESC` / `change_time DESC` 均与脚本实际 `` `change_time` DESC ``（反引号）不符；切分段阈值 ≥2 过严（脚本仅 1 条可执行语句） | 断言改为反引号形态、阈值 ≥1 | 测试跟真实 DDL 语法（MySQL 惯例反引号）而非臆造字面量；单语句脚本是正确形态，不是缺陷 |
| GREEN 3 权限守卫 | `IotDataItemMenuPermissionGuardTest` 2 项失败：域内断言「所有控制器权限 ⊆ 5 个 `iot:item:*`」，而 brief D2 明文要求历史端点用 `iot:instance:query` | 守卫加入唯一明文例外 `iot:instance:query`（已在 sys_menu 登记，仿 0061/0062），并新增该端点的精确锚点断言 | 两条约束本冲突：brief 决策优先于旧域内守卫；例外收窄为单一码 + 精确锚点，既放行决策又不放宽其他漂移防线；全局 `IotPermissionsCompleteGuardTest`（注解 ⊆ sys_menu 登记全集）仍绿，`iot:instance:query` 本就登记 |

## 4. 未做项与原因 / 偏离计划

- **未新增菜单种子**：brief D2 允许「时间线作为数据项详情子功能则不新增菜单，报告说明」。本项目无单独控制台入口需求，权限复用已登记的 `iot:instance:query`，故 DDL 只建表、不写 sys_menu（避免造出无人持有的新权限位或占用 171/172 之外的 id）。
- **范围外连带 1 个文件**：`backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/RedlineFixture.java`（在 `iot/redline/` 而非 brief 声明的 `iot/dataitem/**`）——删除 NoOp 后该夹具编译失败（它 new 了 NoOp 实例），必须改为 `mock(IotDataItemHistoryService.class)` 才能通过验收命令。仅改 import + 2 行，语义不变（红线用例本就不触达时间线）。
- **PARAM 未重复接线**：全仓主代码 `recordParamChange(` 仅 3 处：接口声明、真实实现、唯一调用点 `IotDataItemServiceImpl:466`（原 `:462`，因新增 import/字段上移 4 行，逻辑未动）。
- **未置 TASK 已完成 / 未改索引 / 未 push**：按契约由协调者收尾。

## 5. 关键契约证据

- **四类 change_type 全覆盖**：`IotDataItemHistoryServiceImplTest.allFourChangeTypesPersisted` 断言落库类型集合恰为 `{BIND, FORMULA, ROLLUP, PARAM}`，每条含 operatorId/afterJson/changeTime。
- **同事务性（保存失败无记录）**：`IotDataItemHistoryWiringTest` 中三个失败路径（点位不存在 / 空公式 / 非法汇总方式）均 `verify(historyService, never()).recordXxx(...)`；`IotDataItemHistoryServiceImplTest.rollsBackWithBusinessSave` 在 TransactionTemplate 内写入后抛异常，回滚后 `SELECT COUNT(*)`=0。写点在 `@Transactional` 方法内、落库与 ref 重建成功后调用 record；JdbcTemplate 复用调用方事务连接。
- **before 快照时序**：`IotDataItemHistoryWiringTest` 用 `InOrder` 锁定 `baseMapper.update` → `recordXxx` 顺序；before 全部取 update 前 `item` 快照（BIND 旧点位编码由旧 pointId 反查，FORMULA 旧公式、ROLLUP 旧 SUM/AVG），首次绑定 before=null。
- **项删除后按 full_key 仍可查**：`orderedFilteredAndByFullKey` 插入同 full_key、不同 itemId（200 与 999，模拟删后重建）三行，按 fullKey 查询返回全部 3 行；DDL 的 `full_key` 为 NOT NULL 冗余列 + `idx_tenant_fullkey_time` 索引。
- **跨租户隔离**：`tenantIsolation` 插入 tenant=1 与 tenant=2 的同名 fullKey，按 tenant=1 查仅 1 行、tenant=2 仅 1 行、tenant=999 空集；tenantId 由 `IotDataItemHistoryServiceImpl` 从 `TenantContext.getTenantId()` 取，请求不带 tenant 参数。
- **倒序 + 时间窗 + 分页**：`orderedFilteredAndByFullKey` 断言 FORMULA→BIND 的 `change_time DESC` 顺序、窗口过滤只留 FORMULA、第 2 页取次新。
- **读不审、不双写**：历史端点无 `@OperateLog`（`IotDataItemHistoryControllerTest.endpointShape` 反射断言）；三条写点既有 `@OperateLog` 保持不变，history 写入独立于 operate_log；事件照发（`DataItemConfigChangedEvent` 仍在 record 之前 publish），二者并存不替代。
- **NoOp 移除**：`NoOpIotDataItemHistoryService.java` 与其测试已删除，全仓 `grep -rn NoOpIotDataItemHistoryService` 零命中；接口注释承诺「调用方零改动」兑现——PARAM 调用点未改一行。

## 6. 遗留问题

- 生产库需由部署方执行 DDL 后，时间线表才可用；当前代码在未升级库上调用历史端点/写点会因表不存在报错（与本项目其它「DDL 随升级脚本交付」任务一致的部署前提）。
- `operatorName` 从 `sys_user.real_name` 尽力解析，取不到（用户已删/表不可用）回退 `operatorId`；一期 VO 同时暴露两字段，前端按 `operatorName` 优先展示。
