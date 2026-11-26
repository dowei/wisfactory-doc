# TASK-0003 实施报告（租户开通触发 TDengine 建库与配额强控）

- 执行者：implementer（subagent）
- BASE：`17e09b1`（主仓库 HEAD）；提交：`88caf08`（main，注明 TASK-0003，22 files / +1876 −3）
- 验收口径（人类裁决 2026-11-21）：TDengine 本地不可用（6041/6030 不通，实测探证）→ 对 TDengine 依赖项走「接口抽象 + mock 单测验收」；mock 证明**触发与编排逻辑**（谁在何时以什么参数调了什么），真实库行为登记 TD-0053 挂 TASK-0068 联调闭合。
- 总体结论：**5 条验收 4 条完全通过、1 条通过但含 1 处声明外文件（偏离 1，需协调者追认）**——验收 5 的 diff 中 `SysJobServiceImpl.java`（sys_job 调用白名单 2 行扩展）落在改动范围声明之外，它是 brief 强制的「框架 sys_job 注册」机制在框架侧的必要扩展点（不扩即被框架运行时拒绝，验收 3 无法成立），理由与影响面见偏离 1。

---

## 1. 改动文件清单

提交：`88caf08 feat(iot): 租户开通触发 TDengine 建库（异步化+扫描重试）与配额软限告警（TASK-0003）`

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/.../iot/tdengine/TdengineAdmin.java` | 新增 | 管理抽象接口（brief 裁决 1 三方法）：`createDatabase` / `storageUsageBytes` / `databaseExists` + 静态 `databaseName(tenantCode)`——库名派生**唯一入口**（iot_&lt;code&gt;，REQ-0001 核心规则禁外部传参），派生时做 TDengine 标识符安全校验（字母/数字/下划线 ≤32，结果要拼 DDL） |
| `backend/ot-admin/.../iot/tdengine/TdengineProperties.java` | 新增 | `ot.tdengine.*` 配置（base-url/username/password/connect-timeout-ms/read-timeout-ms） |
| `backend/ot-admin/.../iot/tdengine/HttpTdengineAdmin.java` | 新增 | taosAdapter REST 实现（JDK HttpClient + Basic 认证）：建库=CREATE DATABASE（KEEP 1825d/DURATION 10d/BUFFER 256 全写齐，IF NOT EXISTS 幂等）+ 四张超级表 DDL 经 `/rest/sql/iot_<code>` 库上下文逐条执行（契约附录 B 逐字）；判存=SHOW DATABASES；用量=`ins_disk_usage` 求和（KB×1024）；**一切失败（不可达/非 0 code/HTTP 4xx/非 JSON）映射 BusinessException**（fail-soft，含派生库名与异常摘要） |
| `backend/ot-admin/.../iot/service/TenantDbProvisionService.java` + `impl/TenantDbProvisionServiceImpl.java` | 新增 | 建库触发与扫描编排：`scanPendingDatabases()`（sys_job 入口）扫 status=1 且 db_created=0 → 逐租户先判存（已存在自愈置位，覆盖部署脚本/人工补建）再建库 → 成功置 1 / 失败 WARN（tenant_code + iot_&lt;code&gt; + 异常摘要）留待下轮，**连续失败不升级不阻断、单租户失败不中断同轮**；`provisionImmediately(tenantId)`（事件式即时触发，不替代扫描兜底）；自持 SysTenantMapper 避免与 SysTenantService 循环依赖 |
| `backend/ot-admin/.../iot/service/SysTenantService.java` + `impl/SysTenantServiceImpl.java` | 修改 | ① insertTenant：db_created 强制置 0（请求体携带值一律忽略——mass assignment 防护）；事务提交后（TransactionSynchronization.afterCommit）即时触发 `provisionImmediately`，触发异常收敛不炸穿主流程，status=null 视同启用（API 常省略 status、DDL 默认 1——dev 冒烟实证修复）；② 新增 `checkPointQuota(Long tenantId)`（brief 裁决 3） |
| `backend/ot-admin/.../iot/quota/PointQuotaCounter.java` | 新增 | 点位计数源接口——校验器与提示口径先行（mock 计数验收），**真实实现与调用方接线归 TASK-0021**（brief 裁决 3 明文）；未装配过渡期计数视为 0 放行不误伤 |
| `backend/ot-admin/.../iot/quota/QuotaProperties.java` | 新增 | `ot.quota.*` 配置（storage-warn-ratio 默认 0.8；fail-log-every-n-rounds 默认 20） |
| `backend/ot-admin/.../iot/service/IotQuotaStatsService.java` + `impl/IotQuotaStatsServiceImpl.java` | 新增 | 统计任务（sys_job 入口 `collectAndWarn`）：逐启用租户调 storageUsageBytes → 与 storage_quota_gb×ratio 比对 → 超阈值 WARN 固定格式 `[QUOTA-WARN] tenant=<code> type=storage usage=<bytes> quota=<bytes>`（吞吐同型 type=throughput，`warnQuota` 方法已就位待 ingest 数据源接线）；**只告警不停写**；不可用轮次防日志风暴（连续失败仅首轮与每 N 轮 WARN，恢复清零）；阈值比对用除法形态（乘法在 2 次幂边界受双精度舍入漏报） |
| `backend/ot-admin/.../iot/entity/SysTenant.java` | 修改 | +`dbCreated` 字段；tenantCode 加 `@Pattern([A-Za-z0-9_]+)`（库名派生安全的入口侧防线，与 databaseName 派生校验双保险） |
| `backend/ot-core/.../result/ResultCode.java` | 修改 | **仅追加枚举** `QUOTA_EXCEEDED(4003, "配额不足")`（4001-4099 参数与校验段，语义全局） |
| `backend/ot-admin/.../system/service/impl/SysJobServiceImpl.java` | 修改 | **INVOKE_BEAN_WHITELIST 追加 2 个 bean 名**（tenantDbProvisionServiceImpl / iotQuotaStatsServiceImpl）——⚠ 声明外文件，见偏离 1 |
| `backend/ot-admin/src/main/resources/application.yml` | 修改 | **仅新增 `ot.tdengine.*`（base-url/username/password/connect-timeout-ms/read-timeout-ms）与 `ot.quota.*`（storage-warn-ratio/fail-log-every-n-rounds）**，env 可注入形态，brief 授权的两个键族 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/...`（6 个新测试类，41 例） | 新增 | 见下表 |
| `resources/sql/upgrade/20261121_tenant_db_task.sql` | 新增 | sys_tenant 加列 db_created tinyint NOT NULL DEFAULT 0（information_schema 判存条件化存储过程，幂等）；default 租户置 1；sys_job 播种两任务（invoke_target 判存 INSERT...SELECT WHERE NOT EXISTS，幂等） |

新增测试（41 例，TDD 先红后绿，红灯证据见附 1 根因链）：

| 测试类 | 例数 | 覆盖 |
| --- | --- | --- |
| `iot.tdengine.HttpTdengineAdminTest` | 10 | 假 taosAdapter（JDK HttpServer）记录「谁在何时以什么参数调了什么」：建库 5 条语句逐字断言（iot_acme + KEEP 1825d/DURATION 10d/BUFFER 256 + 四张超级表 TAGS 含 tenant_id）、Basic 认证头、非 0 code/连接拒绝/非 JSON/HTTP 5xx 均映射业务异常、库名派生与非法编码拒绝、判存命中/未命中、用量 KB→字节与 NULL→0 |
| `iot.service.impl.TenantDbProvisionServiceImplTest` | 10 | 扫描只扫 status=1 且 db_created=0（查询条件断言）；成功置 1；库已存在自愈不重建；失败 WARN（tenant_code/iot_acme/摘要/留待下轮）留 0；单租户失败不中断其他；非法编码 fail-soft；即时触发四态（启用/禁用/已建/不存在）；先判存后建库次序 |
| `iot.service.impl.SysTenantOpenTriggerTest` | 5 | 开通强制 db_created=0（携带 1 被忽略）；提交后即时触发；status=0 不触发；status=null 视同启用（冒烟实证修复，红→绿）；触发异常不炸穿；编码冲突不触发 |
| `iot.service.impl.SysTenantPointQuotaTest` | 8 | ResultCode 4003 在位；空/0 不限不触发计数；99/100 放行；100/100 拒且提示「点位配额不足（上限 100）」；101/100 拒；租户缺失 400；计数源未装配视为 0 |
| `iot.service.impl.IotQuotaStatsServiceImplTest` | 7 | 只查启用租户；9GB/10GB@0.8 告警格式逐字断言；恰达阈值（8GB）告警/低于不告警；空/0 配额永不告警；防风暴（首轮 WARN→静默→第 N 轮再 WARN→恢复清零再首轮）；部分失败不误报；空租户静默 |
| `iot.service.impl.SysJobInvokeWhitelistContractTest` | 1 | 锁定 sys_job 调用白名单含两个新 bean（防后续清理误删致任务恒失败） |

## 2. 验收命令原始输出

### 验收 1：`cd backend && mvn test -pl ot-admin`（最终态 = HEAD `88caf08`，与打包/冒烟同一代码状态）

```
[INFO] Tests run: 1105, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Finished at: 2026-11-21T23:38:00+08:00
```

判定：**通过**。1105 = TASK-0002 修订轮基线 1064 + 新增 41（逐类数见上表）。存量无回归（SysTenantControllerTest 16 / SysTenantServiceImplTest 13 / SysTenantInitAdminTest 7 均原样全绿）。ot-core 另测 95/95（ResultCode 仅追加枚举，BUILD SUCCESS）。

说明：全量共跑两轮——首轮 1104 例（23:26，其后 1 例）；dev 冒烟暴露 1 缺陷修复（status=null 触发，附 1 R7）后**按同一提交态复跑全量**（23:38，1105 例），验收 1 以最终态为准。

### 验收 2：升级脚本二次运行幂等 + 表结构核对

第 1 次：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261121_tenant_db_task.sql`（9 条语句：SET NAMES / 过程建删 + CALL / UPDATE / 2 条 INSERT）：

```
OK(0)   ← SET NAMES
OK(0)   ← DROP PROCEDURE ot_tdb_addcol
OK(0)   ← CREATE PROCEDURE
OK(0)   ← CALL：db_created 列缺失 → ALTER 执行（DDL 受影响行数 0）
OK(0)   ← DROP PROCEDURE
OK(1)   ← UPDATE default 租户 db_created=1（首次改 1 行）
OK(1)   ← INSERT sys_job 租户库建库扫描（首插 1 行）
OK(1)   ← INSERT sys_job 租户配额存储统计（首插 1 行）
DONE
```

第 2 次（幂等）：

```
OK(0)
OK(0)
OK(0)
OK(1)   ← CALL：列已存在 → 跳过 ALTER（OK(1) 取过程内末条 SELECT..INTO 受影响行数=1，非重复 DDL，无 SQLERROR 为准；TASK-0002 同款口径）
OK(0)
OK(1)   ← UPDATE default：值已为 1，MySQL 默认返回匹配行数 1（结果幂等，非重复改写）
OK(0)   ← INSERT：invoke_target 已存在 → NOT EXISTS 假 → 0 行
OK(0)
DONE    ← exit 0，无 SQLERROR
```

核对 1：`SHOW CREATE TABLE sys_tenant`（db_created 列在位）：

```
sys_tenant	CREATE TABLE `sys_tenant` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '租户 ID',
  ...
  `deleted` tinyint(4) DEFAULT '0' COMMENT '删除标识',
  `db_created` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'TDengine 建库标志 0 待建 1 已建（定时扫描任务维护，TASK-0003）',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_code` (`tenant_code`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='租户表'
```

核对 2：租户行与播种任务：

```
SELECT id, tenant_code, status, db_created FROM sys_tenant ORDER BY id;
1	default	1	1    ← default 置 1（私有化约定初始库由部署脚本建）
2	acme	0	0 / 4 revtmp 0 0 / 5 tenanta 0 0 / 7 tna 0 0   ← 禁用租户不扫描
6	tenantb	1	0 / 8 tnb 1 0 / 9 tenantc 1 0 / 10 e 1 0 / 11 f 1 0  ← 启用待建，由扫描任务接管（本地无 TDengine → WARN 重试，见验收 3）

SELECT job_name, invoke_target, cron_expression, status FROM sys_job;
租户库建库扫描	tenantDbProvisionServiceImpl.scanPendingDatabases	0 * * * * ?	0   ← 60s 间隔可配（框架 sys_job）
租户配额存储统计	iotQuotaStatsServiceImpl.collectAndWarn	0 0/5 * * * ?	0   ← 5min 间隔可配
```

判定：**通过**（两次 DONE 无 SQLERROR、任务行不重复播种；db_created 列与默认租户置位在位）。

### 验收 3：dev 冒烟（真实验证触发与重试路径——本地无 TDengine 恰好验证 fail-soft）

启动前端口探测：8098 无 LISTENING、6041 探测 `HTTP 000`（本机无 TDengine，预期）。启动 `mvn spring-boot:run -pl ot-admin` 后台，就绪判据 `GET /auth/captchaImage` HTTP 200（READY after ~12s）。启动段日志：`定时任务[租户库建库扫描]已注册调度: 0 * * * * ?`、`定时任务[租户配额存储统计]已注册调度: 0 0/5 * * * ?`、`定时任务初始化完成，已注册 2 个任务`、`Started OtAdminApplication in 10.032 seconds`。

```
=== S1) admin login（dev /auth/login/test，tenantCode 缺省 -> default）
{"code":200,"data":{"nickname":"管理员","token":"0a3d8740-...","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795275039527}

=== S2) POST /tenant 建租户 E (code=e，请求省略 status)
首次尝试 {"code":4002,"data":null,"message":"租户编码已存在: e"}   ← 修复轮曾建过 e（编码创建后不复用的唯一闸生效，预期行为）
{"code":200,"data":null,"message":"操作成功","timestamp":1795275081769}   ← 建租户 f（status=1 显式）成功

=== S2b) GET /tenant/10 回读（租户 E）
{"code":200,"data":{"id":10,"tenantCode":"e",...,"status":1,...,"dbCreated":0,...},"message":"操作成功"}   ← db_created=0、status=1，开通主流程未受影响
```

（说明：租户 e 在本轮启动前的旧实例上已创建——旧实例跑的是「status=null 不触发」的缺陷版（附 1 R7），e 落库正确（db_created=0）仅即时触发未挂上；修复后重建同码被 4002 拒（符合编码不复用纪律），故 e 的即时触发证据以单测红→绿 + 租户 f 的实测即时触发为准，e 走扫描重试路径作为主证据。）

```
=== S3) PUT /job/run/1（runOnce 触发扫描轮）
{"code":200,"data":null,"message":"操作成功","timestamp":1795275064330}

=== S3b) 扫描轮 WARN（含 iot_e，日志原文 UTF-8 提取）：
2026-11-21 23:31:00.066 [ot-job-1] WARN  n.o.a.m.i.s.impl.TenantDbProvisionServiceImpl - TDengine 建库失败，留待下轮扫描重试（连续失败不升级不阻断） tenant=e db=iot_e 异常摘要：判存 iot_e失败：TDengine REST 不可达 http://localhost:6041（java.net.ConnectException）
2026-11-21 23:31:04.357 [ot-job-2] WARN  n.o.a.m.i.s.impl.TenantDbProvisionServiceImpl - TDengine 建库失败，留待下轮扫描重试（连续失败不升级不阻断） tenant=e db=iot_e 异常摘要：判存 iot_e失败：TDengine REST 不可达 http://localhost:6041（java.net.ConnectException）

=== S3c) 扫描轮 INFO（任务自身成功，失败收敛为租户级 WARN）：
2026-11-21 23:31:04.357 [ot-job-2] INFO  n.o.a.m.i.s.impl.TenantDbProvisionServiceImpl - 租户库建库扫描完成：待建 4 个，本轮成功 0 个

=== S3d) sys_job_log（runOnce 执行记录，status=1 成功、无 fail_reason）：
job_id	job_name	status	cost_time	fail_reason
1	租户库建库扫描	1	27	null
1	租户库建库扫描	1	66	null

=== S4) 事件式即时触发（租户 f 提交后立刻尝试一次，http-nio 线程即请求线程，非 ot-job 线程）：
2026-11-21 23:31:21.768 [http-nio-8098-exec-9] WARN  ... tenant=f db=iot_f 异常摘要：判存 iot_f失败：TDengine REST 不可达 http://localhost:6041（java.net.ConnectException）

=== S4c) db_created 仍 0：
id	tenant_code	status	db_created
10	e	1	0
```

**base-url 两轮失败路径**（不改文件，命令行参数覆盖，每轮重启实例并 runOnce）：

```
=== S5) 不可达地址 --ot.tdengine.base-url=http://127.0.0.1:9
2026-11-21 23:32:00.086 [ot-job-1] WARN  ... TDengine 建库失败，留待下轮扫描重试（连续失败不升级不阻断） tenant=e db=iot_e 异常摘要：判存 iot_e失败：TDengine REST 不可达 http://127.0.0.1:9（java.net.ConnectException）
db_created 仍 0（id=10 e 1 0）；GET /tenant/10 → HTTP 200

=== S6) 可达假地址 --ot.tdengine.base-url=http://127.0.0.1:3307（MySQL 端口，可达非 taosAdapter）
2026-11-21 23:32:45.345 [ot-job-1] WARN  ... TDengine 建库失败，留待下轮扫描重试（连续失败不升级不阻断） tenant=e db=iot_e 异常摘要：判存 iot_e失败：TDengine REST 不可达 http://127.0.0.1:3307（Invalid status line: "J    "）
db_created 仍 0（id=10 e 1 0）；GET /tenant/10 → HTTP 200

=== S7) 统计任务防风暴实测（runOnce job id=2 连续两轮）：
2026-11-21 23:33:00.173 [ot-job-3] WARN  n.o.a.m.iot.service.impl.IotQuotaStatsServiceImpl - TDengine 存储统计不可用，本轮跳过（连续失败第 1 轮，仅首轮与每 20 轮告警防风暴）：失败租户 6/6，最后失败 tenant=f 异常摘要：存储用量 iot_f失败：TDengine REST 不可达 http://127.0.0.1:3307（Invalid status line: "J    "）
第 2 轮后 WARN 计数不变（1 → 1，静默）——防日志风暴生效
```

判定：**通过**（dev 冒烟恰好验证 fail-soft 重试路径：建库尝试失败 → WARN（含 tenant_code、iot_e、异常摘要）→ db_created 仍 0 → 开通主流程不受影响（租户 E status=1 可查、list 200）——**这是预期行为不是故障**；60s cron 自动轮与 runOnce 手动轮均实证；即时触发以租户 f 实证）。三实例（首轮/修复轮/base-url×2 轮）完成后均已停止，8098 复核 0 监听。

### 验收 4：prod 档真实启动回归（BUG-0001 已修原命令口径）

打包：`cd backend && mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS，`ot-admin/target/ot-admin-1.0.0.jar`（132,567,915 字节；-DskipTests 因全量测试已在验收 1 同一代码态覆盖）。

启动（原命令 + TASK-0001/0002 已确认的 prod fail-fast 必需三密钥环境变量）：

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
READY after ~15s (captchaImage HTTP 200)
2026-11-21T23:36:32.970+08:00  INFO 68008 --- [ot-framework] [main] net.otlab.admin.OtAdminApplication : Started OtAdminApplication in 16.894 seconds (process running for 17.626)
grep -c "ERROR" /tmp/task0003_prod.log → 0

=== P1) prod POST /auth/login（图形验证码经 Redis 键 captcha:code:task0003a 预置 JSON 字符串 "AB12"，密码 admin123 经 hutool SM4-ECB 加密 hex 再 base64；不传 tenantCode → 走 default）
{"code":200,"data":{"nickname":"管理员","token":"ff9da26c-...","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795275405315}

=== P2) prod GET /tenant/list（带 token）→ HTTP_STATUS:200，total=8（dbCreated 字段随实体正常序列化，default dbCreated=1）

=== P3) prod /auth/login/test 拒绝（dev-only 防线回归）
{"code":500,"data":null,"message":"测试登录接口仅限开发环境使用"}
```

判定：**通过**（Started 无 ERROR + 真实登录 200 + list 200）。prod 实例完成后停止，8098 复核 0 监听。

### 验收 5：改动范围核对

命令：`git diff --name-only 17e09b1..HEAD`（HEAD=88caf08）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/SysTenant.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/quota/PointQuotaCounter.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/quota/QuotaProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/IotQuotaStatsService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/SysTenantService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/TenantDbProvisionService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotQuotaStatsServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/TenantDbProvisionServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdmin.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/TdengineAdmin.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/TdengineProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/impl/SysJobServiceImpl.java   ← ⚠ 声明外（偏离 1）
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/IotQuotaStatsServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysJobInvokeWhitelistContractTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantOpenTriggerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantPointQuotaTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/TenantDbProvisionServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdminTest.java
backend/ot-core/src/main/java/net/otlab/core/result/ResultCode.java
resources/sql/upgrade/20261121_tenant_db_task.sql
```

核对结论：22 文件中 **21 个全落在声明内**（iot 主代码 12 + iot 测试 6 + ResultCode 仅追加枚举 + upgrade SQL + application.yml）；application.yml 逐行核对**仅新增 ot.tdengine.* 与 ot.quota.* 两键族**；未碰 frontend/、ingest/、0002 登录链路与拦截器、application-prod.yml。**唯一声明外文件**：`SysJobServiceImpl.java`（+2 行白名单登记）——判定：**通过（含 1 处需协调者追认的框架扩展，理由见偏离 1；如不追认，备选方案见偏离 1 末段）**。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 真实 TDengine 建库/判存/统计行为验证 | 人类裁决（brief 口径）：mock 验收先做，真实库验证登记 TD-0053 挂 TASK-0068 联调闭合；`HttpTdengineAdmin` 的 REST 细节（Basic 认证口径、`ins_disk_usage` 字段形态、taosAdapter 多语句行为）均以假 taosAdapter 断言请求形态，真实性归联调 |
| 吞吐（type=throughput）软限告警的实际接线 | 吞吐用量源归接入侧（ingest），本批仅交付骨架（ingest 服务骨架未含用量上报）；告警格式与编排已就位（`IotQuotaStatsServiceImpl.warnQuota(code, "throughput", ...)` 直接可复用），数据源交付后一行接线 |
| 点位配额计数真实实现与 CRUD 调用方接线 | brief 裁决 3 明文「调用方接线归 TASK-0021（点位 CRUD），本任务只交付校验器与单测」；已交付 `PointQuotaCounter` 接口 + 校验器 + 8 例边界测试 |
| TC-0001 全用例联测 | TC-0001 为 FEAT 级联调用例（依赖点位 CRUD/采集上报等后续任务）；本任务落地其步骤 6 的服务侧能力（建库触发），brief 已裁决 mock 口径 |
| 超级表 DDL 的真实 TDengine 解析验证 | 契约附录 B 逐字实现并经假 taosAdapter 断言逐字下发；真实解析（KEEP 1825d 等参数在目标版本是否全部合法）归 TD-0053 |

## 4. 偏离计划之处

1. **`SysJobServiceImpl.java`（声明外，2 行）**：brief 强制「定时扫描任务（框架 sys_job 注册）」「定时任务（框架 sys_job）」，而框架 `SysJobServiceImpl.invokeTarget` 对调用目标 bean 有硬编码白名单（安全设计：仅白名单 bean 可被定时任务反射调用）——新 bean 不登记则播种的 sys_job 行每轮被框架拒绝（`调用目标不在允许的 bean 白名单内`），brief 裁决 2 的扫描链路与验收 3 全部无法成立。处置：白名单追加 2 个 bean 名（该清单即框架预留的业务任务登记点，注释注明 TASK-0003），并以 `SysJobInvokeWhitelistContractTest` 锁定防误删；验收 5 如实标注。**若协调者不追认**：备选方案是回退白名单改动、改用 Spring `@Scheduled` + `ot.tdengine.*` 间隔配置（完全在声明内，但偏离 brief 的「框架 sys_job 注册」且失去 sys_job_log 执行留痕与运营方改频能力）——二选一，请裁决。
2. **dev 冒烟暴露缺陷一处（先红后绿，已修）**：开通请求省略 status 时（DB 默认 1）即时触发不挂接——按内存值 `status==1` 判定漏判。修复为 `status==null || status==1`（null 视同启用，落库走 DDL 默认 1），补单测 `SysTenantOpenTriggerTest.testInsertTenant_StatusNull_TreatedAsEnabled`（红：注册数为空断言失败 → 绿）。详见附 1 R7。租户 e 的即时触发因此发生在修复轮的租户 f 上（编码不复用纪律使 e 无法重建），e 以扫描重试路径为主证据——未影响验收结论。
3. **`HttpTdengineAdmin` 异常摘要增强**：JDK `ConnectException` 某些形态 `getMessage()` 为 null，首轮冒烟 WARN 摘要为空串；修复为回退 `e.toString()` 保住异常类型（R8）。已入单测（连接拒绝用例断言消息含 "TDengine"）。
4. **storageUsageBytes 数据源选型**：TDengine `ins_databases` 无磁盘用量列，采用官方 meta 视图 `information_schema.ins_disk_usage`（按 db_name 汇总 WAL/多级存储/缓存/元数据，单位 KB，×1024 转字节）；该视图为较新 3.x 版本视图，版本不适配时表现为 SQLERROR → 业务异常 → 统计任务按不可用跳轮（fail-soft 不误报）。字段真实性归 TD-0053 实测。
5. **entity 硬编码防注入双保险**：`SysTenant.tenantCode` 增 `@Pattern([A-Za-z0-9_]+)`——库域名派生规则（REQ-0001 核心规则）要求派生结果可安全拼 DDL；`TdengineAdmin.databaseName` 派生入口另有同口径运行时校验（非法编码在扫描轮 fail-soft WARN 留 0，不冒充 TDengine 失败）。属防御性收紧，现有合法编码不受影响。
6. **验收 4 命令形态**（沿 TASK-0001/0002 已确认先例）：a) 打包 `-DskipTests`（全量测试已在同一提交态覆盖）；b) 启动补 `SM4_KEY`/`CRYPTO_DATA_KEY`/`CRYPTO_INDEX_KEY`（prod fail-fast 必需，取 dev 默认值，非新密钥）。
7. **构建顺序**：`mvn test -pl ot-admin` 前需 `mvn install -pl ot-core`（单模块构建引用 .m2 中的 ot-core；TASK-0001/0002 报告同款前置）。
8. **进程纪律**：dev 首轮/修复轮/base-url 不可达轮/可达假轮 + prod 共 5 个自起实例，均先探测端口再启动、完成后停止，8098 每次复核 0 监听。
9. **git commit 执行**（`88caf08`，main，注明 TASK-0003）：验收 5 的范围核对命令比较两个提交点，不提交则命令恒为空；提交内容严格限于上列 22 个文件（`.agents/` 等任务前已存在未跟踪文件未纳入）。

## 5. 遗留问题

1. **TD-0053 联调清单**（挂 TASK-0068）：a) 真实 taosAdapter Basic 认证与 `/rest/sql/<db>` 库上下文行为；b) `ins_disk_usage` 在现场 TDengine 版本的可用性与字段集；c) 建库语句在目标版本的真实解析（KEEP 1825d/DURATION 10d/BUFFER 256）；d) 吞吐用量源接线与 type=throughput 告警实测。
2. **存量启用租户的建库收敛**：dev 库现有 5 个启用待建租户（e/f/tenantb/tnb/tenantc）将持续每轮 WARN 重试直到现场 TDengine 可用（fail-soft 设计使然，不升级不阻断）；联调环境接入真库后首轮扫描即自愈置位。禁用租户（acme/revtmp/tenanta/tna）不扫描——若后续启用会在下轮被拾取，符合口径。
3. **PointQuotaCounter 过渡期口径**：TASK-0021 交付前无实现 bean，`checkPointQuota` 计数视为 0（放行）——校验器已就位但**运行时无真实强控**，勿在 TASK-0021 前对外宣称点位配额已强控生效（接线即生效，零改动）。
4. **统计任务的 per-round 内存态**：连续失败轮计数为实例内存（AtomicInteger），多实例部署时各实例独立计数（告警节流按实例生效，非全局）——单机/试点形态无影响；集群形态若需全局节流应改 Redis 计数（挂 TD-0053 后评估）。
5. **sys_job 白名单治理**：白名单现为 7 个 bean 的硬编码清单；若后续任务频繁新增业务任务，可考虑配置化（挂后续任务，本批不动框架结构）。
6. **冒烟数据留痕**：租户 e(id=10)/f(id=11)、任务执行日志若干、Redis 验证码键与 token（自然过期）——均为预期交付数据，重复验收可直接重跑（脚本与流程幂等）。

---

## 附 1：根因修复链（契约要求「为什么现在对了」）

| # | 现象（红） | 根因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- | --- |
| R1 | `HttpTdengineAdminTest` 编译失败：找不到 `HttpTdengineAdmin` 等 4 类符号 | TDD 红灯预期：接口与实现未写 | 实现 TdengineAdmin/TdengineProperties/HttpTdengineAdmin | 红→绿符合 TDD 流程 |
| R2 | `TenantDbProvisionServiceImplTest`/`SysTenantOpenTriggerTest` 编译失败（符号缺失、`when` 未导入、`close()` 未声明 throws） | TDD 红灯预期 + 测试代码机械缺漏 | 补实现与测试 imports/签名 | 红→绿；断言逻辑未放松 |
| R3 | `IotQuotaStatsServiceImplTest.testCollect_UnavailableStormGuard` ERROR：`BusinessException: TDengine REST 不可达` 从测试直抛 | Mockito `when(mock.method())` 在**再打桩**时会真实调用 mock——上一次 thenThrow 存根在打桩表达式求值时抛出 | 改 `doThrow/doReturn` 形态再打桩（不触发既有存根） | 打桩不再借用真实调用；断言逻辑（轮次计数）未变 |
| R4 | ot-admin 全量编译失败：`QUOTA_EXCEEDED` 找不到符号 | ot-admin 单模块构建引用 .m2 中旧版 ot-core | 先 `mvn install -pl ot-core` 再测（既定前置） | 构建顺序符合仓库现状 |
| R5 | 首次全量 1104 全绿；阈值边界用例初版以乘法形态 `usage >= quota*ratio` 预演推演可能漏报 8GB/10GB@0.8 恰达阈值 | 双精度 0.8 非精确表示，`quota*ratio` 会得到略大于 8GB 的积 | 实现即采用除法形态 `usage/quota >= ratio`（0.8 的真值与双精度最近值相等，边界判定精确） | 单测恰达阈值告警/低于不告警双向绿 |
| R6 | （设计缺陷，冒烟前推演发现）播种的 sys_job 行会被框架白名单拒绝 | 框架 invokeTarget 白名单未含新 bean | 白名单登记 2 bean + 契约测试锁定 | 冒烟 S3 runOnce 实测：任务执行成功（job log status=1）且产生租户级 WARN——链路贯通 |
| R7 | dev 冒烟：建租户（省略 status）后**无**即时触发 WARN | 触发条件按内存 `status==1` 判定；请求省略 status 时内存为 null，落库走 DDL 默认 1——判定与库内实态不一致 | 条件改 `status==null || status==1`；补单测（红→绿：注册同步器从 0 到 1） | API 建租户（无论显式 1 或省略）均即时触发；status=0 显式禁用仍不触发（行为面不扩大）；全量 1105 复跑绿 |
| R8 | dev 冒烟 WARN 异常摘要为空串（`...不可达 http://localhost:6041（）`） | JDK ConnectException 该形态 getMessage()==null | 摘要回退 `e.toString()`（保异常类型） | S3-S7 实测摘要含 `java.net.ConnectException` / `Invalid status line`；单测断言消息含 "TDengine" 绿 |

## 附 2：契约自查

- 未派 subagent；未把 TASK 置「已完成」；未改 workitems 库其他文件（仅本报告）。
- 改动 22 文件：21 个在 brief 声明内 + 1 个框架白名单扩展（偏离 1 如实申报待追认）；frontend/、ingest/、0002 登录链路与拦截器、application-prod.yml 未碰；application.yml 仅增 `ot.tdengine.*`/`ot.quota.*`。
- 全量 1105 例（最终提交态）+ ot-core 95 例 0 失败 0 错误；5 条验收命令全部执行且原始输出贴回；TDD 先红后绿（R1-R3/R7），测试与期望未被反向修改。
- 自起实例（dev×4 + prod×1）均已停止，8098 复核 0 监听。
- ADR-0011（TD-0035/TD-0041 只告警不停写）、ADR-0002（标签 tenant_id 属建表参数、建库参数无关——契约冻结稿已含）、库名只由 tenant_code 派生禁外部传参：全部照做。
