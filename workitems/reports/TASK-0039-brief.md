# 实施 brief：TASK-0039（iot_data_item_history 数据项变更时间线）

> 标准任务（backend iot/dataitem 域扩展，纯后端）。BASE=7886325。一个执行者做完整、统一测一次。前置 0036✅。

## 批次

- TASK-0039（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（backend mvn test）
- BASE：`7886325`
- 改动范围（文件/目录级，越界反查用）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/**`（history 服务真实实现、接口扩展、三个写点接线、查询端点/VO/mapper）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/**`
  - `resources/sql/upgrade/20261124_iot_data_item_history.sql`（history 表 DDL + 菜单种子如需要，**只产出不执行**）
  - **不得触碰**：frontend/**、ingest/**、calc/**、resources/contract/**（冻结）、既有 publish/inspect/alarm/calcread/statusread/template 模块、framework upstreams
  - 注意：`NoOpIotDataItemHistoryService.java` 需删除或停用（真实实现替换后移除，接口注释已声明）

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0039.md`
- 关联对象：`features/FEAT-0012.md`、`requirements/REQ-0018.md`、`testcases/TC-0012.md`、`decisions/ADR-0005.md`
- 接缝（已预留）：
  - `backend/.../iot/dataitem/service/IotDataItemHistoryService.java`（**仅 `recordParamChange` 一个方法**，PARAM 已接线）
  - `backend/.../iot/dataitem/service/NoOpIotDataItemHistoryService.java`（占位实现，真实实现上线后删除）
  - `backend/.../iot/dataitem/service/IotDataItemParamChange.java`（PARAM 载荷：itemId/fullKey/operatorId/before/after Thresholds）
  - `backend/.../iot/dataitem/service/impl/IotDataItemServiceImpl.java:434 overrideParams`（PARAM 已调 recordParamChange，:462）、`:668 bindPoint` / `:709 saveFormula` / `:862 saveRollup`（三类写点已发 DataItemConfigChangedEvent 但**未写 history**）
  - `backend/.../iot/dataitem/event/DataItemConfigChangedEvent.java`（ChangeType 枚举 BIND/FORMULA/ROLLUP，复用其语义）
- 控制器范式：`backend/.../iot/dataitem/controller/IotDataItemController.java`（各方法全路径，无类级 @RequestMapping）
- DDL/权限范式：`resources/sql/upgrade/20261124_iot_alarm.sql` / `20261124_iot_inspect.sql`（逐行切分、菜单种子幂等 WHERE NOT EXISTS、权限码沿用 iot:instance:query）
- 审计：BaseEntity / @OperateLog；多租户插件对 iot_ 表自动注入 tenant_id

## 接口与决策（逐条结论，无选项）

### D1 · 表与写入（四种 change_type 全覆盖，同事务）
- 新建 `iot_data_item_history`（业务库，DDL 写 `resources/sql/upgrade/20261124_iot_data_item_history.sql`）：字段至少 id/tenant_id/item_id/**full_key（冗余，项删除后仍可查）**/change_type（BIND/FORMULA/ROLLUP/PARAM）/before_json/after_json（TEXT，NULL=新建首条）/**change_summary**（可选简述）/operator_id/change_time/create_time；索引 idx_tenant_fullkey_time (tenant_id, full_key, change_time DESC)。审计口径参照 REQ-0018（时间线记业务值变化），与 sys_operate_log 分工不双写。
- **扩展 `IotDataItemHistoryService` 接口**：在现有 `recordParamChange` 之外新增 BIND/FORMULA/ROLLUP 三条 record 方法（载荷 record 各自带 itemId/fullKey/operatorId/before/after 值；形式自定但语义明确）。真实实现统一写 history 表；删除 NoOp 实现（接口注释承诺调用方零改动——PARAM 调用点保持不变）。
- **三个写点接线**：在 `IotDataItemServiceImpl` 的 `bindPoint`/`saveFormula`/`saveRollup` 各自**落库成功后、同一 @Transactional 内**调用对应 record 方法（before=变更前该配置值，after=本次值——变更前值在执行 update 前先读 item 快照取得；公式类 after 用展开后 expanded 公式，bind 类 before/after 用 pointCode，rollup 类用 SUM/AVG）。**PARAM 已在 :462 接线，保持不变勿重复写**。
- **同事务性**：保存失败（抛异常回滚）则无 history 行——TC 直接构造失败路径验证。发布动作不入时间线（REQ-0018 非目标）。
- 变更人：从 `SecurityUtils.getUserId()` / 会话取（调度线程无会话时按系统 0，但本任务写点都在 Web 会话内）。

### D2 · 查询端点（管理面按数据项倒序时间线）
- `GET /api/v1/iot/dataitem/history?itemId=&fullKey=&from=&to=&page=` → 分页倒序时间线（change_time DESC）。支持两种定位：按 itemId（现存项）或按 fullKey（**项删除后仍可查**，REQ-0018 验收 3）。二者至少给其一，否则参数错误。
- tenant 从会话强制注入（多租户插件 + 显式 tenant_id 条件），跨租户查询返回空集/403（仿 0061/0062 口径）；权限码沿用 `iot:instance:query`；**读不审**（0079 范式，时间线是只读查询，写动作的审计由各写点 @OperateLog 与 history 行本身承担）。
- VO 含 changeType/changeTypeText/beforeJson/afterJson/operatorName(或 operatorId)/changeTime。
- 菜单种子：若需控制台入口，加 sys_menu（id 避开 171/172，挂合适父节点）+ 幂等授权；若判断无需单独菜单（时间线作为数据项详情子功能）则不加并重报告说明——不强制新增菜单。

### D3 · 与 0062/ADR-0014 的边界
- 时间线 ≠ sys_operate_log：操作日志记"谁调了接口"，时间线记"业务值前后变化"，不重复双写；本任务只写 iot_data_item_history，不写 operate_log（写点既有 @OperateLog 保持）。
- 与 DataItemConfigChangedEvent（BIND/FORMULA/ROLLUP 发布联动）并存：事件继续发（发布域消费），history 另写，二者不互相替代。

## 验收命令（必须原样贴回输出）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（四类 change_type 写入 + 同事务回滚 + 倒序查询 + full_key 删除后可查 + 跨租户隔离有契约测试，TDD 红转绿留痕），贴 Results 段
2. DDL 静态检查：`20261124_iot_data_item_history.sql` 逐行切分、无同行多语句；**未在用户 phpStudy MySQL 执行**（报告声明）
3. 范围核对：`git diff --name-only 7886325..HEAD` 全落改动范围声明；frontend/ingest/calc/contract/既有其他模块零触碰；NoOp 实现已移除

## 报告

报告写到：`reports\TASK-0039-report.md`。五项齐全 + 一句话结论 + TDD 红转绿（每修复附"为什么现在对了"）+ DDL 未执行声明 + 同事务性/删除后可查/跨租户证据。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引；DDL 只产出不执行。
- 四种 change_type（BIND/FORMULA/ROLLUP/PARAM）全覆盖，缺一不可；PARAM 已接线勿重复。
- 同事务性（保存失败无 history）；项删除后按 full_key 仍可查；跨租户隔离；读不审、不双写 operate_log。
- 不造假数据、不硬编码凭据/文档库路径。
- 测试失败先定位根因，禁止为匹配输出改测试/改期望。
