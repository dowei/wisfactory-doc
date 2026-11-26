# 评审 brief：TASK-0062 diff 评审（代码评审）

> 实现提交 2755060（BASE=24b5c44，区间干净单提交，26 文件）——系统事件告警记录闭环：backend iot/alarm 新域 + frontend 告警列表页。FEAT-0020 最后一块。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0062-diff.txt`（2755060 单提交，26 文件）
- 实施 brief + 报告：reports\TASK-0062-brief.md / TASK-0062-report.md（含 D1-D4 落地 + TDD 红转绿 + DDL 未执行声明 + 边界遵守声明）
- TASK 详情 + FEAT-0020/REQ-0031/TC-0020 + ADR-0013 增补 6（系统事件告警边界）
- 事件源（只读消费）：`backend/.../iot/gateway/event/GatewayOfflineEvent.java` + `GatewayOfflineReason.java` + `GatewayOfflineEventListener.java`（fail-closed + try/catch 范式）
- 范式对照：`iot/inspect/controller/IotInspectController.java`（0061 读不审 + tenant 会话注入 + iot:instance:query）、`iot/publish/notify/PublishAckTimeoutScanner.java`（@Scheduled 范式）、`resources/sql/upgrade/20261124_iot_inspect.sql:31-41`（菜单种子幂等写法）
- 审计：BaseEntity / @OperateLog 范式

## 越界反查

- 提交 2755060 恰 26 文件全落 brief 声明范围（backend iot/alarm 13 主 + 8 测试、frontend alarm 页/api/测试、resources/sql/upgrade/20261124_iot_alarm.sql）；区间 24b5c44..HEAD 干净单提交；ingest/contract/0061 inspect/0063 calcread/gateway 既有事件代码/sys_notice 公告表零触碰。

## 评审重点

1. **告警记录表（D1）**：`iot_alarm_record` 字段/索引/唯一键去重合理；审计四件套用显式 JdbcTemplate 列（执行者称工程无全局 MetaObjectHandler，MP fill 不落值——核实这个判断与实现一致）；**重复事件不把已确认告警打回待确认**（状态不回退）；DDL 逐行切分兼容 sql-exec.sh、**只产出未执行**。
2. **告警生成边界（D2，ADR-0013 增补 6）**：仅系统事件三类；GATEWAY_OFFLINE listener 消费 GatewayOfflineEvent、**GRACEFUL 正常下线不生成告警**、fail-closed 缺字段忽略 + try/catch 不外溢；DATA_TIMEOUT @Scheduled 扫描 iot_calc_state 滞后/停算、按 full_key 去重；缓存积压无真源→诚实空态不造假（核实没编造数据）；**业务阈值告警/通知矩阵/升级策略未混入**（三期边界）。
3. **端点与权限（D3，REQ-0031 验收 4/5）**：list 分页 tenant 会话注入 + 数据权限裁剪 + 跨租户 403；confirm/handle 状态流转守卫（非法跃迁拒绝、不静默改）+ handler_id/handle_note/时间戳留痕；**写口挂 @OperateLog**（写要审，与 0061 读不审相反）；站内出口用独立 iot_alarm_notice 表不污染 sys_notice 公告、不外发邮件/企微；权限码沿用 iot:instance:query。
4. **前端告警页（D4）**：筛选/列表/确认/处理 + 备注；空态 a-empty；API 只调管理面端点、**不持 appKey/secret、不直连 openapi HMAC、不自造代理**；菜单种子 id=172 幂等（WHERE NOT EXISTS）、挂 110 系统监控下、避开 0061 的 171。
5. **测试有效性与零回归**：backend 2513 用例 0F0E（含新增）、frontend build:test + lint（0 error）+ typecheck + pnpm test 610 passed；TDD 红转绿真实（去重/状态流转/越权403/GRACEFUL不生成/留痕有断言）、调试日志清理；0061/0063/gateway 既有零回归。
6. **范围**：26 文件全落声明范围、越界 0、区间干净。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0062-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。特别对第 2 点（仅系统事件边界 + GRACEFUL 不生成 + 不造假数据）与第 3 点（跨租户 403 + 写口审计留痕）给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
