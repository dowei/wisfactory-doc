# 评审 brief：TASK-0039 diff 评审（代码评审）

> 实现提交 57aeb36（BASE=7886325，区间干净单提交，21 文件）——iot_data_item_history 数据项变更时间线。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0039-diff.txt`（57aeb36 单提交，21 文件，1656 行）
- 实施 brief + 报告：reports\TASK-0039-brief.md / TASK-0039-report.md（含 D1-D3 落地 + TDD 红转绿 + DDL 未执行声明 + 偏离说明）
- TASK 详情 + FEAT-0012/REQ-0018/TC-0012 + ADR-0005
- 接缝基线（改动前）：`IotDataItemHistoryService.java`（原仅 recordParamChange）、`NoOpIotDataItemHistoryService.java`（已删）、`IotDataItemParamChange.java`（PARAM 载荷）
- 写点：`backend/.../dataitem/service/impl/IotDataItemServiceImpl.java`（BIND ~:718 / FORMULA ~:877 / ROLLUP ~:915 / PARAM ~:466）
- 范式对照：`resources/sql/upgrade/20261124_iot_alarm.sql`、`20261124_iot_inspect.sql`（DDL 逐行切分 + 菜单幂等）

## 越界反查（本任务有声明范围外连带，重点核实）

- 提交 21 文件：18 全落声明范围（backend iot/dataitem + resources/sql/upgrade/20261124_iot_data_item_history.sql），**3 个需判定必要性**：
  1. `backend/.../iot/redline/RedlineFixture.java`（M，声明范围外）——执行者称删除 NoOp 后该 fixture 必须改 Mockito mock 才能编译，报告 §4 已记录；核实是否**真为编译必需、改动最小、无测试弱化**；
  2. `backend/.../dataitem/config/IotDataItemMenuPermissionGuardTest.java`（M）——核实是否仅因 NoOp 删除/权限守卫适配，无放宽断言；
  3. `backend/.../dataitem/service/IotDataItemService.java`（M，接口）——核实改动必要性。
- 另：NoOp 实现与其测试均为删除（D），符合 brief「真实实现上线后移除 NoOp」；frontend/ingest/calc/contract/既有其他模块零触碰。

## 评审重点

1. **四类 change_type 全覆盖 + 同事务（D1，REQ-0018 验收 1）**：BIND/FORMULA/ROLLUP 三个写点是否在各自落库成功后、同一 @Transactional 内调用对应 record 方法；before 值是否在 update 前读 item 快照取得（不是后验伪造）；after 语义正确（bind=pointCode/formula=expanded/rollup=SUM-AVG）；**PARAM 唯一接线点（~:466）保持不变、未重复写**；构造保存失败路径验证无 history 行（同事务性）。是否四类齐全、缺一不可。
2. **真实实现与 NoOp 移除**：`IotDataItemHistoryServiceImpl` 用 JdbcTemplate 显式 tenant_id 统一落库、倒序分页查询；NoOp 实现与其测试删除后，全部调用方/测试仍能编译通过（mvn test 实证）；接口扩展的四个 record 方法签名清晰、载荷字段完整（itemId/fullKey/operatorId/before/after）。
3. **查询端点（D2，REQ-0018 验收 2/3）**：`GET /api/v1/iot/dataitem/history` 支持 itemId 或 fullKey 双定位（**项删除后按 full_key 仍可查**）、二者都不给参数错误；倒序 change_time DESC；tenant 会话强制注入 + 跨租户空集/403；权限码 iot:instance:query；**读不审**（无 @OperateLog）；VO 含 changeType/文本/before/after/操作人/时间。
4. **边界（D3）**：时间线不双写 sys_operate_log；发布动作不入时间线；与 DataItemConfigChangedEvent 并存不互相替代。
5. **测试有效性与零回归**：backend 2534 用例 0F0E（2516 基线 + 18 新增）、TDD 红转绿真实（四类写入/同事务回滚/倒序/删除后可查/跨租户有断言）、调试日志清理；既有测试零回归。
6. **DDL 与范围**：20261124_iot_data_item_history.sql 逐行切分、无同行多语句、**未在用户库执行**（报告声明）；21 文件里 3 个连带修改必要性判定（见越界反查）。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0039-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。特别对第 1 点（四类齐全 + 同事务 + PARAM 不重复）与越界反查 3 个连带文件必要性给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
