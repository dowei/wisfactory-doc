# TASK-0075 实施报告（sys_operate_log user_id 可空化与未认证操作日志落库修复）

- 执行者：implementer（subagent）
- BASE：`e30de0a`（主仓 HEAD，派发态）；提交：`9754e0e`（main，注明 TASK-0075 / BUG-0003，4 files / +360 −10）
- 总体结论：**全部验收通过**——TC-0024 先红后绿闭环（红：prod /auth/login/test → `操作日志写入失败 … Field 'user_id' doesn't have a default value` + 库零新增；绿：同请求落库 user_id=NULL、user_name=probeUser 尝试值、无写库失败 ERROR；认证态写操作回归 user_id=1 既有行为不变）；全量 1233 例 0 失败 0 错误。

---

## 1. 改动文件清单（提交 9754e0e，4 个，全部落在 TASK 风险核对三行声明范围）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/system/aspect/OperateLogAspect.java` | 修改 | 切面 null 容忍（BUG-0003 修复方案②）：① `fillUserInfo` 未认证分支不再静默 return——身份列留空（user_id/user_type/user_name=null 落库，列已可空）并尽力记尝试上下文；② 新增 `resolveAttemptUserName`：从切面已捕获的方法参数 JSON 提取 username 尝试值（多参逐个查找，截断 500，取不到不造数，全程兜底不影响落库主流程）；③ `StpUtil.isLogin()` 异常兜底按未认证处理（不因取身份失败丢日志）；认证态分支逻辑零改动 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysOperateLog.java` | 修改 | 实体注释口径更新（userId/userType/userName 三列可空语义，BUG-0003）；无结构改动（MyBatis Plus NOT_NULL 字段策略下 null 列本就不进 INSERT 列清单，与可空列配合即落库成功） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/system/aspect/OperateLogAspectTest.java` | 新增 | 4 例：未认证态登录失败（身份列 null + user_name=尝试用户名 probeUser + 失败记录语义 + 尝试上下文 url/ip/param 齐）；无参数不造数；参数无 username 字段不造数；认证态行为回归（user_id=1/用户/部门齐备）。真实 Method 反射 + RequestContextHolder + mockStatic(StpUtil/OtIpUtils)，沿 PermissionDeniedAuditListenerTest 范式 |
| `resources/sql/upgrade/20261122_operate_log_identity_nullable.sql` | 新增 | 存量库升级脚本（BUG-0003 修复方案①）：sys_operate_log 身份三列 `user_id bigint` / `user_type int` / `user_name varchar(500)` 改 NULL 可空；information_schema.IS_NULLABLE 判存幂等（条件化存储过程，单行纪律兼容 sql-exec.sh）；user_name 保持 utf8mb4/utf8mb4_general_ci 原口径；01_base.sql 不改 |

改动范围核对：`git show 9754e0e --stat` → 恰上述 4 文件；切面/实体一处（OperateLog 域）、增量 SQL 一份（判存幂等）、测试一类——与风险核对第 1 行逐字对应；未碰 01_base.sql、未碰 LoginLogService（两日志不混存口径不变）、未碰登录业务逻辑、frontend/ 与 ingest/ 零接触、application*.yml 未动。

## 2. 验收命令原始输出

### 验收 0（BUG-0003 加严证据 1）：全库 sys_operate_log 引用 grep 扫描

```
=== A) 全库 sys_operate_log 引用（排除 target/node_modules/.git，全部文件类型）===
backend\ot-admin\src\main\java\net\otlab\admin\module\system\entity\SysOperateLog.java:22:@TableName("sys_operate_log")
backend\ot-admin\src\main\java\net\otlab\admin\module\system\listener\PermissionDeniedAuditListener.java:23: * <p>把 core 发布的 {@link PermissionDeniedEvent} 落到 sys_operate_log（status=0 失败记录），
backend\ot-core\src\test\java\net\otlab\core\tenant\OtTenantLineHandlerTest.java:71: "sys_config", "sys_dict_data", "sys_file", "sys_operate_log", "sys_login_log",
resources\sql\01_base.sql:438:DROP TABLE IF EXISTS `sys_operate_log`;
resources\sql\01_base.sql:439:CREATE TABLE `sys_operate_log` (
=== B) frontend 单独补扫（前端是否引用表名）===（frontend/src 扫描 operate_log/sys_operate_log：0 命中，前端仅走 REST API）
=== C) Java 侧写入/查询点（SysOperateLog|OperateLogService|saveOperateLog）===
backend/ot-admin/.../system/aspect/OperateLogAspect.java          ← 写入点 1（本任务落点）
backend/ot-admin/.../system/listener/PermissionDeniedAuditListener.java ← 写入点 2（未登录分支本就 userId=null，受益于列可空，代码无需改）
backend/ot-admin/.../system/service/impl/OperateLogServiceImpl.java     ← selectList（userName/部门/业务类型/状态筛选，全部可选项）+ selectMyList（eq user_id，仅已认证态触发，NULL 身份行天然不入结果）→ 既有查询兼容
backend/ot-admin/.../system/controller/OperateLogController.java  ← 列表页参数 userName（前端 operate-log-list.vue 按用户名称 LIKE 筛选，非 user_id；可选项）
```

判定：**通过**——引用面 = 表定义（01_base.sql，不改）+ 写入点 2 处（切面=本任务落点；监听器已 null 容忍）+ 查询（筛选条件均为可选项，无 NOT NULL 假设）； OtTenantLineHandlerTest 的租户忽略表清单仅字符串引用不受列可空影响。风险核对第 2 行「user_id 可空对既有查询兼容」成立。

### 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
mvn install -pl ot-core -DskipTests:
[INFO] BUILD SUCCESS
===CORE-INSTALL-DONE===

mvn test -pl ot-admin（Results 段）:
[INFO] Results:
[INFO]
[INFO] Tests run: 1233, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
[INFO] Total time:  01:08 min
[INFO] Finished at: 2026-11-22T02:14:55+08:00
```

新测试类定向：

```
[INFO] Running net.otlab.admin.module.system.aspect.OperateLogAspectTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.372 s -- in net.otlab.admin.module.system.aspect.OperateLogAspectTest
[INFO] BUILD SUCCESS
```

判定：**通过**。1233 = 基线 1229（TASK-0009 收口态）+ 新增 4（OperateLogAspectTest；未动任何既有测试文件，git diff 可核）。（日志中的「业务异常：…」ERROR 行为既有测试的预期异常路径打桩输出，非失败。）

### 验收 2：升级脚本执行 + 二次幂等 + 列状态核对

第 1 次：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_operate_log_identity_nullable.sql`

```
OK(0)   ← SET NAMES utf8mb4
OK(0)   ← DROP PROCEDURE IF EXISTS ot_ol_nullable
OK(0)   ← CREATE PROCEDURE ot_ol_nullable
OK(0)   ← CALL ×3（三列 IS_NULLABLE='NO' 命中 → 存储过程内执行 ALTER MODIFY NULL；CALL 语句自身 updateCount 恒 0）
OK(0)
OK(0)
OK(0)   ← DROP PROCEDURE
DONE    ← exit 0，无 SQLERROR
```

第 2 次（幂等）：

```
OK(0) ×3（SET NAMES / DROP PROC / CREATE PROC）
OK(1) OK(1) OK(1)   ← CALL ×3（三列已 IS_NULLABLE='YES' → 存储过程内跳过 ALTER；CALL 报 OK(1) 为 JDBC 对 CALL 结果计数口径，非再修改）
OK(0)   ← DROP PROC
DONE    ← exit 0，无 SQLERROR
```

列状态核对（原始输出）：

```
SELECT COLUMN_NAME, IS_NULLABLE, COLUMN_TYPE, COLUMN_COMMENT …
COLUMN_NAME	IS_NULLABLE	COLUMN_TYPE	COLUMN_COMMENT
user_id	YES	bigint(20)	用户id（BUG-0003：未认证操作尝试无登录身份，可空落库）
user_type	YES	int(11)	用户类型（BUG-0003：未认证操作尝试无用户类型，可空）
user_name	YES	varchar(500)	用户名称（BUG-0003：未认证态记登录尝试用户名，取不到为空）

SHOW CREATE TABLE sys_operate_log（节选）:
  `user_id` bigint(20) DEFAULT NULL COMMENT '…可空落库）',
  `user_type` int(11) DEFAULT NULL COMMENT '…可空）',
  `user_name` varchar(500) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL COMMENT '…取不到为空）',
```

判定：**通过**（两次 DONE 无 SQLERROR；三列 IS_NULLABLE=YES 一次到位；字符集/长度/其余列属性不变；幂等口径成立——实际效果以 information_schema 列状态为准，CALL 的 OK(0)/OK(1) 展示差异不影响判定）。

### 验收 3：TC-0024 步骤 2 基线红（修复前 prod 实测，jar 构建自 BASE=`e30de0a`）

打包：`mvn clean package -pl ot-admin -DskipTests` → `ot-admin-1.0.0.jar`（132,622,241 字节）。启动（TASK-0009 验收 4 命令口径 + prod fail-fast 三密钥，PowerShell Start-Process 全脱链）：

```
启动前 8098 探测：无 LISTENING
READY（captchaImage HTTP 200）
2026-11-22T02:09:27.171+08:00  INFO 17108 --- Started OtAdminApplication in 14.617 seconds
启动段 ERROR 计数 = 0（grep -c ERROR → 0）
DB 基线：SELECT COUNT(*), MAX(id) FROM sys_operate_log → 96 / 96
```

TC-0024 步骤 2 触发：`POST /auth/login/test`（body 为合法 LoginDTO，过 @Valid 进入方法体被 prod 守卫拒绝）：

```
{"code":500,"data":null,"message":"测试登录接口仅限开发环境使用","timestamp":1795284582861}
```

应用日志（原始输出）：

```
2026-11-22T02:09:42.855+08:00 ERROR 17108 --- [nio-8098-exec-4] n.o.c.exception.GlobalExceptionHandler : 业务异常：auth.testLoginDevOnly
2026-11-22T02:09:43.091+08:00 ERROR 17108 --- [t-operate-log-1] n.o.a.m.s.s.impl.OperateLogServiceImpl : 操作日志写入失败

org.springframework.dao.DataIntegrityViolationException:
### Error updating database.  Cause: java.sql.SQLException: Field 'user_id' doesn't have a default value
### The error may involve net.otlab.admin.module.system.mapper.OperateLogMapper.insert-Inline
### SQL: INSERT INTO sys_operate_log (business_type, module, method, content, url, request_method, param, ip, ip_region, user_agent, status, fail_reason, cost_time, create_time, create_by, update_time, update_by, deleted) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
### Cause: java.sql.SQLException: Field 'user_id' doesn't have a default value
```

DB 复核：

```
SELECT COUNT(*), MAX(id) FROM sys_operate_log → 96 / 96（无新增行——审计日志丢失，红）
```

判定：**通过（红）**——与 TASK-0009 验收 4 P4 01:57:31 复现完全同款（ERROR「操作日志写入失败 … Field 'user_id' doesn't have a default value」且 sys_operate_log 无新增行）。**关键旁证**：报错 SQL 列清单不含 user_id/user_type/user_name 三列（MyBatis Plus NOT_NULL 策略把 null 列整体省略），证明只放宽 user_id 一列 INSERT 仍会撞 user_type/user_name（均 NOT NULL 无默认）——直接支撑偏离 1 的三列放宽决策。

### 验收 4：TC-0024 步骤 3-4 绿（修复后 prod 实测，jar 重新构建自修复后工作树）

打包：`mvn clean package -pl ot-admin -DskipTests` → `ot-admin-1.0.0.jar`（132,622,893 字节）。启动：

```
启动前 8098 探测：无 LISTENING
READY（captchaImage HTTP 200）
2026-11-22T02:16:24.591+08:00  INFO 50484 --- Started OtAdminApplication in 23.309 seconds
DB 基线：96 / 96
```

TC-0024 步骤 3（绿）：同请求 `POST /auth/login/test`：

```
{"code":500,"data":null,"message":"测试登录接口仅限开发环境使用","timestamp":1795284991004}

应用日志：grep -c "操作日志写入失败" → 0；grep -c "DataIntegrityViolation|doesn't have a default value" → 0
（全日志唯一 ERROR 行 = 02:16:30.996 业务异常：auth.testLoginDevOnly——P4 故意触发的 dev-only 拒绝本身，预期）

SELECT id, user_id, user_type, user_name, url, request_method, module, status, fail_reason, ip, deleted
  FROM sys_operate_log ORDER BY id DESC LIMIT 2:
id	user_id	user_type	user_name	url	request_method	module	status	fail_reason	ip	deleted
97	null	null	probeUser	/auth/login/test	POST	用户登录(测试)	0	auth.testLoginDevOnly	127.0.0.1	0
96	1	3	admin	/auth/login	POST	用户登录	1	null	127.0.0.1	0
COUNT/MAX → 97 / 97（新增 1 行）
```

TC-0024 步骤 4（正常认证态回归）：admin 登录（验证码经 Redis 键 captcha:code:task0075a 预置 JSON 字符串 "AB12"，jshell 回读验证；密码 admin123 经 hutool SM4-ECB 加密 hex 再 base64，本地 ROUNDTRIP=admin123 验证）→ 认证态写操作 `DELETE /monitor/cache/key?key=captcha:code:task0075x`（不存在键，@OperateLog(title=缓存监控, businessType=DELETE)，无状态副作用）：

```
POST /auth/login → {"code":200,"data":{"nickname":"管理员","token":"5c7dd0d9-…","userId":1,"username":"admin"},…}
DELETE /monitor/cache/key → {"code":200,"data":null,"message":"操作成功"}

SELECT id, user_id, user_name, url, request_method, module, status FROM sys_operate_log ORDER BY id DESC LIMIT 3:
id	user_id	user_name	url	request_method	module	status
99	1	admin	/monitor/cache/key	DELETE	缓存监控	1
98	1	admin	/auth/login	POST	用户登录	1
97	null	probeUser	/auth/login/test	POST	用户登录(测试)	0
COUNT/MAX → 99 / 99
```

判定：**通过（绿 + 回归）**——步骤 3：同请求落库 user_id=NULL、url=/auth/login/test（TC 笔误 request_url 实为 url 列，按实际列核对）、且 user_name=probeUser 证明「username 尝试值能取则记」落地、无写库失败 ERROR；步骤 4：认证态登录与写操作均落库 user_id=1，既有行为不变。8098 纪律：红/绿两实例启动前均无监听、用毕即停（taskkill 17108/50484）、复核均 0 监听。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| PermissionDeniedAuditListener 代码改动 | 其未登录分支本就以 userId/userName=null 构造落库（代码已 null 容忍，单元测试 testAuditLoggedWithoutUser 在位）——过去同样会被 NOT NULL 约束拒绝，本任务列可空化后自然恢复，属收益方而非改动面；无控制流变化 |
| ot-core 任何改动 | @OperateLog 注解（ot-core）为纯标记注解无需变更；实际切面落点在 ot-admin（见偏离 2） |
| OperateLogServiceImpl 写入逻辑改动 | 异步线程池 + try/catch 兜底口径不变（ERROR 兜底仍是最终防线，修复后不再触发）；无控制流变化 |
| 01_base.sql 修改 | TASK 风险核对第 2 行明确不改；新库按部署口径执行 01_base.sql + 全部 upgrade 脚本后与本脚本口径一致（脚本头部已注明） |
| 登录失败场景接 LoginLogService | BUG 明确「两日志不混存口径不变」；本修复只保证 OperateLog 不再丢弃 |

## 4. 偏离计划之处

1. **放宽列数：3 列而非 brief 字面的 user_id 1 列（证据性必要扩展）**：红阶段报错 SQL（验收 3 原始输出）证明 MyBatis Plus 把 user_id/user_type/user_name 三个 null 列全部从 INSERT 列清单省略，三列均 NOT NULL 无默认——只改 user_id 可空则同请求仍会报 `Field 'user_type' doesn't have a default value`。BUG-0003 预期行为（「审计完整性不受认证态影响」）的必要条件即三列一起放宽；user_name 在未认证态由切面尽力记尝试用户名、取不到为空（与 BUG 方案②逐字一致），user_type 未认证态无可取值只能为空。脚本头部已完整记录推理链。
2. **切面落点更正（证据性）**：TASK 风险核对第 1 行写「ot-core OperateLog 切面/实体一处」——实际切面在 ot-admin（`net.otlab.admin.module.system.aspect.OperateLogAspect`），ot-core 仅有 @OperateLog 注解（无需改）。范围语义（切面/实体一处 + 增量 SQL + 测试、无控制流变化）不变。
3. **切面改动形态澄清**：切面未认证态「身份列留空」本就存在（原代码 `if (!StpUtil.isLogin()) return;`），本修复新增的是 ① 未认证分支尽力提取尝试用户名（resolveAttemptUserName，落实 BUG 方案②「能取则记，取不到不造数」）② isLogin() 异常兜底按未认证处理。非「把 null 改成可落库」的实体结构改动——实体无结构变化（仅注释）。
4. **先红后绿的实现口径**：红 = 修复代码未写、jar 构建自 BASE `e30de0a`；绿 = 修复后重新打包（两次独立 build，132,622,241 → 132,622,893 字节）。红转绿跨越 SQL 脚本执行 + 代码提交内容，证据链在验收 3/4。
5. **sql-exec.sh CALL 语句展示口径**：存储过程 CALL 的 OK(0)/OK(1) 为 JDBC 对 CALL 结果计数的表现差异，不直接表示是否执行 ALTER；幂等性以两遍执行无 SQLERROR + information_schema 列状态核对为准（验收 2）。
6. **TC-0024 步骤 3 SQL 列名笔误**：TC 写 `request_url`，实际列名 `url`（01_base.sql 450 行）；按实际列核对 url=/auth/login/test，语义与 TC 意图一致。
7. **jshell 冒烟准备工具坑**（沿 TASK-0009 同款链路）：hutool Padding 枚举常量为 `PKCS5Padding`（非 PKCS5）；SM4 需 BouncyCastle provider（bcprov-jdk18on-1.80 加入 jshell classpath 解决）；均为脚本侧修正，应用零改动。

## 5. 遗留问题

1. **sys_operate_log 留痕 3 行**（id 97/98/99）：97=未认证尝试（user_id=NULL，正是本修复的审计语义载体，非测试垃圾）；98/99=admin 登录与缓存监控写操作（真实认证态操作）。操作日志为审计数据且表有 cleanAll 既有能力，保留不清理。
2. **Redis captcha 键 captcha:code:task0075a 与 token 会话**：5 分钟 TTL 自然过期 / 86400s 自然过期，无需清理。
3. **prod 启动段唯一 ERROR 为 dev-only 拒绝的业务异常日志**（GlobalExceptionHandler 对故意触发拒绝的 ERROR 级记录，TASK-0008/0009 冒烟同款）：属预期行为非缺陷；若希望降噪可在 GlobalExceptionHandler 对业务异常降级 WARN——超出本任务范围，供协调者归口。
4. **user_name 承载尝试用户名的语义口径**：未认证行 user_name=尝试值 + user_id=NULL 组合已在列表页可读辨识（状态=0、失败原因列、无部门）；前端如需显式「未认证尝试」标记可后续迭代（本任务不涉展示规则，风险核对第 3 行）。
5. **_PARAM 列含登录参数明文密码密文**（LoginDTO.password 为 SM4 密文，且 maskJsonFields 的 password 键规则会将其值替换为 ******——已脱敏，无新增暴露面）；历史口径未动。

---

## 附 1：契约自查

- 未派 subagent；未把 TASK-0075 状态改为「已完成」（详情留协调者收尾）；workitems 库仅新增本报告文件（TASK-0075-brief.md 派发时未落地，以 TASK-0075.md 详情为主体执行——已按派发消息口径）。
- 提交恰 4 文件（`git show 9754e0e --stat` 可核），全部落在风险核对三行声明内；`.agents/`（任务前已存在未跟踪）未纳入；8098 红绿两实例均「启动前探测无监听 → 用毕 taskkill → 复核 0 监听」；无残留本任务 java/mvn 进程（既有无关实例 PID 22592 为 framework 仓 8080 应用，非本任务产物，未触碰）。
- BUG 加严两项证据齐备：① 全库 sys_operate_log grep 扫描原始输出贴回（验收 0）；② 先红后绿（TC-0024 步骤 2 红：ERROR + 96/96 零新增；步骤 3 绿：id=97 user_id=NULL 无写库失败 ERROR；步骤 4：id 98/99 user_id=1 既有行为不变）——原始输出全部贴回（验收 3/4）。
- 验收命令原样执行：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 1233 例 0 失败 0 错误（+新增 4 与基线 1229 闭环）；prod 打包启动后 TC-0024 步骤 2-4 实测。
- 01_base.sql 未改；LoginLogService 未触碰（两日志不混存口径不变）；既有查询页兼容（userName LIKE 筛选为可选项，selectMyList 仅认证态触发）。
- 审计语义：认证失败尝试开始留痕（user_id=NULL + 尝试用户名 + url/ip/param/fail_reason），不涉权限/金额/日期/编号/展示规则（风险核对第 3 行）。
