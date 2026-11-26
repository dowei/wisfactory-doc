# TASK-0075 diff 评审报告（BUG 修复类加严）

- 评审者：fresh reviewer（只依据评审 brief 的「必读」与「依据」）
- 评审对象：提交 `9754e0e`（BASE `e30de0a`，4 文件 +360 −10：OperateLogAspect.java / SysOperateLog.java / OperateLogAspectTest.java（新增） / 20261122_operate_log_identity_nullable.sql（新增））
- 评审依据：L1-execute.md §13（BUG 修复类加严）+ brief 两项加严核验 + 一致性/质量两样
- 评审方式补充（fresh 自证）：评审者在本仓独立执行了 `sys_operate_log` 全库 grep、`git diff --stat e30de0a..HEAD` 越界反查、01_base.sql 建表口径核对、sql-exec.sh 切分逻辑核对、升级脚本过程名冲突比对、前端 operate-log 页字段核对，并独立复跑 `OperateLogAspectTest`（Tests run: 4, Failures: 0, Errors: 0）。

## 一、加严核验两项（逐项结论）

### 核验 1：全仓同契约点扫描 —— **通过**

- 评审者独立 grep 结果与报告验收 0 一致：引用面 = `01_base.sql` 表定义（439-443 行三列 NOT NULL 无默认值，本 diff 未改）+ 实体 SysOperateLog + 写入点 2 处（`OperateLogAspect`＝本任务落点；`PermissionDeniedAuditListener` 未登录分支本就以 userId/userName=null 构造，过去同样撞 NOT NULL，列可空化后自然恢复，代码无需改，核实属实）+ 查询点（`OperateLogServiceImpl#selectList` 全部筛选条件可选；`selectMyList` eq(user_id) 仅认证态 /mine 可达，NULL 身份行天然不入结果——语义正确）+ `OtTenantLineHandlerTest` 租户忽略表清单（纯字符串引用）。
- 导出与前端联动：`OperateLogController` 无导出端点，`frontend/src` 对 sys_operate_log/userId 零引用，操作日志列表页仅 userName LIKE 筛选与 userName 列展示（可选条件，NULL 行空显示）——brief 点名的「导出列」风险不存在，查询兼容成立。
- 升级脚本面：`ot_ol_nullable` 过程名与既有 upgrade 脚本（ot_tn_addcol / ot_tn_swapuk / ot_tdb_addcol / wechat_*）无冲突；单行 CREATE PROCEDURE 纪律与 20261121 租户脚本同款，且经 sql-exec.sh 切分逻辑（`;\s*\n` 按行尾分号切、逐行剥 -- 注释）证实为必要而非风格偏好。

### 核验 2：回归测试先红后绿 —— **通过**

- 红：jar 构建自 BASE `e30de0a`，2026-11-22T02:09:43 prod 实测 `POST /auth/login/test` → `操作日志写入失败 … Field 'user_id' doesn't have a default value`，DB 96/96 零新增——与 BUG-0003 复现（TASK-0009 验收 4 P4，01:57:31）完全同款；报错 INSERT 列清单不含身份三列。
- 绿：重新构建（字节差 132,622,241→132,622,893 佐证两次独立 build），同请求落库 id=97：user_id=NULL、user_name=probeUser、url=/auth/login/test，grep 写库失败 ERROR=0；TC-0024 步骤 4 认证态回归 id 98/99 user_id=1 既有行为不变。
- 时间线自洽：红 02:09:43 → 单测（验收 1，finished 02:14:55）→ 脚本执行 + 绿 02:16:24 → 提交 02:18:23；红阶段报错本身即证明当时列仍 NOT NULL，脚本不可能先于红执行。
- 单测层面：评审者独立复跑 OperateLogAspectTest → 4/4 通过。

## 二、一致性评审（BUG-0003 方案 / TASK 风险核对）

- **偏离 1「三列放宽」——依据充分、副作用评估到位**：01_base.sql 439-443 实证 user_id/user_type/user_name 三列均 NOT NULL 无默认值；红阶段报错 SQL 列清单实证 MyBatis Plus NOT_NULL 策略把三个 null 列整体省略——只放宽 user_id 必撞 user_type，三列一起放宽是 BUG-0003 预期行为（审计完整性不受认证态影响）的必要条件。user_name 保持 varchar(500)/utf8mb4_general_ci 原口径；对另一写入方（监听器）是纯收益；查询兼容已核（见核验 1）。
- **增量脚本幂等 / 01_base 不改**：information_schema.IS_NULLABLE='NO' 判存 + 条件化存储过程，报告两遍执行无 SQLERROR 且列状态核对三行 IS_NULLABLE=YES、字符集/长度不变；`01_base.sql` 不在 diff 中，纪律守住；脚本头部完整记录推理链与执行器约束。
- **切面 null 容忍落库字段完整性**：未认证态记录 url/method/ip/ip_region/user_agent/module/businessType/status=0/fail_reason/param（落库前仍走 maskJsonFields 脱敏，password 类全遮）/cost_time/create_time/update_time/deleted；user_name 仅取参数中 username 尝试值（截 500 对齐列宽，取不到不造数，全程兜底），user_id/user_type/createBy/updateBy 不造数。`isLogin()` 异常兜底按未认证处理，不因取身份失败丢日志。认证态分支零改动（单测 testAuthenticatedBehaviorUnchanged + prod id 98/99 双证）。
- **越界反查**：`git diff --stat e30de0a..HEAD` 恰 4 文件，全部落在 TASK 风险核对三行声明范围（切面/实体 + 增量 SQL + 测试）；未碰 01_base.sql、LoginLogService、登录业务逻辑、frontend/、application*.yml。
- **测试真实性**：真实 Method 反射 + RequestContextHolder + mockStatic(StpUtil/OtIpUtils)，未认证路径（含无参、无 username 字段两态）与认证态回归均有断言；`verify(operateLogService).saveOperateLog` 抓实体逐字段断言，覆盖身份列 null、失败语义、尝试上下文与审计字段。评审者复跑通过。

## 三、发现清单

| 级别 | 出处 | 说明 |
| --- | --- | --- |
| Critical | （无） | — |
| Important | （无） | — |
| Minor | OperateLogAspect.java L182-188（新增守卫）vs OperateLogAspectTest.java | 本次 diff 新增的 `StpUtil.isLogin()` 异常兜底分支（异常→按未认证处理）无单测覆盖，现有 4 例均走 isLogin 正常返回，该守卫路径的「不丢日志」语义只有代码兜底没有测试护栏。 |
| Minor | OperateLogAspect.java L228-240 vs OperateLogAspectTest.java | `resolveAttemptUserName` 的 JSONArray 多参数逐个查找分支与 `getRequestParams` 序列化失败回退（toString → 非 JSON → 不造数）两条兜底路径未测；主路径（单对象/无参/无 username 字段）已覆盖，属测试广度缺口非缺陷。 |
| Minor | TASK-0075-report.md §2 验收 2/3 | 报告章节编号把脚本执行（验收 2）排在红（验收 3）之前，实际时序为红（02:09）→ 单测（02:14）→ 脚本 + 绿（02:16），读者需靠时间戳自行还原执行顺序；证据链本身自洽，建议按时间线重排或加一行时序说明。 |
| Minor | TC-0024.md 步骤 3 | TC 原文 SQL 列名笔误 `request_url`（实际列为 `url`）已在报告偏离 6 记录但 TC 正文未修，后续回归照 TC 抄写 SQL 会报未知列错误。 |

## 四、结论

- 两项加严证据均通过（评审者独立复核：全库 grep 一致、红绿时间线自洽同款复现、单测复跑 4/4）。
- 与 BUG-0003 修复方案①②③及 TASK 风险核对三行一致；偏离 1「三列放宽」与偏离 2「切面落点」均有原始证据支撑并已完整记录。
- 发现 Critical 0 / Important 0 / Minor 4——Minor 不进修订轮，按 L1 §13 记 TASK `任务结果` deferred（收尾进 LOG），本 diff 可过。
