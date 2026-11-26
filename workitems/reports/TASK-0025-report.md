# TASK-0025 验收报告：下发记录 ACK 回执与超时重发

> 执行者：TASK-0025 执行；验收日期：2026-11-22；工作区 D:\workspace\WisFactory（main 分支）。
> 开工 BASE = da830e4（`git rev-parse --short HEAD`，与协调者记录一致）；实现提交 = **d08ec02**（24 文件）。
> 环境：WSL docker wisfactory-mysql（MySQL 8.0.42，127.0.0.1:3307）/ wisfactory-redis（6379）；8098 启动前探测空闲、用毕即停（两次手动释放，见「四」备注 8）；TDengine 未触碰。

---

## 一、改动文件清单（验收 5：改动范围核对）

`git diff --name-only da830e4..HEAD` 原始输出（24 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/config/PublishProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/config/PublishSchedulingConfig.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/controller/IotPublishController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/controller/IotPublishInternalController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/AckForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/PublishLogListVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/PublishLogRowVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/ResendForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/entity/IotConfigPublishLog.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/mapper/IotConfigPublishLogMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/NoOpNotifySender.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/NotifySender.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/PublishAckTimeoutScanner.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/IotPublishLogService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishLogServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/support/PublishScopeGuard.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/config/IotPublishMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/controller/IotPublishControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/controller/IotPublishInternalControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishLogServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImplTest.java
resources/sql/upgrade/20261122_iot_publish_log.sql
```

**比对结论：通过。** 全部落在 brief「改动范围」声明内：`/iot/publish/**`（主代码 17 文件）、`src/test/.../iot/publish/**`（测试 5 文件，新增 2 + 调整 3）、`application.yml`（仅新增 ot.publish.ack-* ×2 与 ot.publish.target-* ×2 共 4 键——diff 逐行核对，见验收 5）、升级 SQL。不碰 frontend/、ingest/、openapi、dataperm、credential、gateway 域、ot-core。

application.yml 新增行（验收 5 附加核对）：

```
+    ack-scan-interval-seconds: ${OT_PUBLISH_ACK_SCAN_INTERVAL_SECONDS:60}
+    ack-timeout-seconds: ${OT_PUBLISH_ACK_TIMEOUT_SECONDS:300}
+    target-ingest-id: ${OT_PUBLISH_TARGET_INGEST_ID:ingest}
+    target-calc-id: ${OT_PUBLISH_TARGET_CALC_ID:calc}
```

---

## 二、每条验收命令原始输出

### 验收 1：单测全量（BUILD SUCCESS、0 失败 0 错误）

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
[INFO] Tests run: 2205, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  01:29 min
```

**通过**：2205 通过 / 0 失败 / 0 错误 / 0 跳过（0024 交付时 2165，本任务净增 40 个测试用例）。

TDD 红转绿留痕（brief 测试口径）：先行落测试（`IotPublishLogServiceImplTest` 等），stub 实现阶段首跑为红——

```
[ERROR] Tests run: 23, Failures: 9, Errors: 14, Skipped: 0 <<< FAILURE! -- in ...IotPublishLogServiceImplTest
（其余 4 个测试类同期已绿：guard 4 / controller 11 / internal controller 6 / service impl 23 全过）
```

实现真实逻辑后转绿：67 个测试（23+11+6+4+23）全过。期间两处修复均附根因：
1. `PublishProperties.java` javadoc 中「ack-*/target-*」内含 `*/` 提前闭合注释导致编译失败——改为「ack- 与 target-」表述（为什么现在对了：注释不再被截断）。
2. 新测试三处断言「按行 id 定位」失败：根因是 MyBatis-Plus 条件参数惰性求值——`eq()` 的 WHERE 参数在 `getSqlSegment()` 渲染时才落入 `paramNameValuePairs`（以独立 WrapperDebugTest 实证：SQLSET 占 MPGENVAL1-3、SEGMENT 占 MPGENVAL4-5）；测试先渲染 WHERE 段再断言参数（为什么现在对了：断言与 MP 真实行为对齐，同时顺带锁定了 WHERE 带条件守卫）。

### 验收 2：升级脚本二次幂等 + 表结构 + 4074

命令：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_iot_publish_log.sql` 连续两次。

第一次（建表 0 行变更 + 菜单/授权各插 1 行）：

```
OK(0)
OK(0)
OK(1)
OK(1)
DONE
```

第二次（幂等零变更零报错）：

```
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

表结构核对（`SHOW CREATE TABLE iot_config_publish_log` 尾段）：

```
  PRIMARY KEY (`id`),
  KEY `idx_version` (`version_id`),
  KEY `idx_ack` (`ack_status`,`ack_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='配置下发记录表'
```

（brief DDL 逐字在位：idx_version / idx_ack 双索引；COLLATE 为 MySQL 8 默认补齐，脚本与 brief 均未显式指定。）菜单种子与授权核对：

```
id    parent_id  menu_name  menu_type  perms
4074  4015       下发重发    3          iot:publish:resend
role_id  menu_id
1        4074
```

**通过**：两次 DONE（二次幂等零变更）；idx_version/idx_ack 在位；4074 一行挂 4015 + 超管授权在位。

### 验收 3：dev 冒烟全序列

dev 档启动（mvn spring-boot:run，默认 profile dev，环境变量注入 `OT_GATEWAY_INTERNAL_TOKEN=tok-0025-smoke` 与 `OT_PUBLISH_ACK_TIMEOUT_SECONDS=20` 调短超时窗以便观察扫描）：`Started OtAdminApplication in 17.169 seconds`，启动日志 **ERROR_COUNT=0**。登录走 `/auth/login/test`（dev 专用）。建链：新建网关 SMOKE0025_GW → id=9110（status=1 启用）。

**步骤 1：发布网关 scope → 版本 1（versionId=22）→ publish_log 两行（待确认）**

```
POST /iot/publish/publish {"scopeType":1,"scopeId":9110}
{"code":200,"data":{"id":22,...,"version":1,"snapshotHash":"ad5c5fb0...","snapshotUri":"1/1-9110-1.json","publishStatus":1,...}}
SQL: SELECT id, version_id, target_type, target_id, ack_status, ack_time, error_msg
     FROM iot_config_publish_log WHERE version_id = 22 ORDER BY target_type;
id  version_id  target_type  target_id      ack_status  ack_time  error_msg
1   22          1            SMOKE0025_GW   0           null      null
2   22          2            ingest         0           null      null
—— 两行待确认落行（网关行取 gateway_code、接入行取配置标识 ingest；头注②语义实证）。
   同事务性亦实证：落行伴随版本落库成功（无 tenant_id 列的表在会话租户下 INSERT 成功
   即 @InterceptorIgnore 旁路端到端生效——不旁路则报 Unknown column 'tenant_id'）。
```

**步骤 2：状态查询三态可查（versionId 与三元组两种定位）**

```
GET /iot/publish/log/list?versionId=22
{"code":200,"data":{"versionId":22,"scopeType":1,"scopeId":9110,"version":1,"publishState":0,"rows":[
 {"id":1,"targetType":1,"targetId":"SMOKE0025_GW","ackStatus":0,"timedOut":false,...},
 {"id":2,"targetType":2,"targetId":"ingest","ackStatus":0,"timedOut":false,...}]}}
GET /iot/publish/log/list?scopeType=1&scopeId=9110&version=1
{"code":200,"data":{...同上..."publishState":0,...}}  —— 三元组定位等价
```

**步骤 3：内部令牌拦截（无令牌/错令牌拒绝）+ ackStatus=0 拒绝**

```
POST /internal/v1/config/ack（无 X-Internal-Token）
{"code":401,"data":null,"message":"内部令牌无效"}
POST /internal/v1/config/ack（X-Internal-Token: wrong-token）
{"code":401,"data":null,"message":"内部令牌无效"}
POST /internal/v1/config/ack（正确令牌 + ackStatus=0）
{"code":400,"data":null,"message":"ackStatus=0 不接受（回执只有 1 生效 / 2 失败）"}
```

**步骤 4：带 token curl /internal/v1/config/ack 置生效 → 行变已生效 + ack_time**

```
POST /internal/v1/config/ack {"versionId":22,"targetType":1,"targetId":"SMOKE0025_GW","ackStatus":1}
{"code":200,"data":true}
SQL 复查：id=1 SMOKE0025_GW ack_status=1 ack_time=2026-11-22 23:47:42 error_msg=null
状态查询：SMOKE0025_GW 1 ackTime=2026-11-22 23:47:42 / ingest 0 —— publishState=0（尚有未回执）
```

**步骤 5：再造一行失败回执 + 已生效终态保护 + 找不到行 400**

```
POST /internal/v1/config/ack {"versionId":22,"targetType":2,"targetId":"ingest","ackStatus":2,
 "errorMsg":"smoke failure: snapshot hash mismatch"} → {"code":200,"data":true}
POST /internal/v1/config/ack（对已生效的网关行再发失败回执——过期失败回执）
{"code":200,"data":false,"message":"操作成功"}  —— data=false=忽略不倒退，行保持 ack_status=1
POST /internal/v1/config/ack {"versionId":22,"targetType":3,"targetId":"calc","ackStatus":1}
{"code":400,"data":null,"message":"下发记录不存在: (versionId=22, targetType=3, targetId=calc)"}
```

（时序备注：失败回执到达前，超时扫描已在 scheduling 线程把 ingest 行 0→2——服务日志
`23:47:51.101 [scheduling-1] INFO ... [PUBLISH-ACK] timeout scan marked 1 stale row(s)
timeoutSeconds=20 candidate=1`——即步骤 7 的扫描置 2 在 v1 行上已先行实证；本步骤的失败
回执为 2→2 覆盖（语义允许），error_msg 更新为显式失败原因。）

**步骤 6：调短 ack-timeout-seconds 验证扫描置 2（error_msg 含超时）**

```
POST /iot/publish/publish {"scopeType":1,"scopeId":9110} → versionId=23 version=2
立即查询：publishState=0；行 3/SMOKE0025_GW 0 timedOut=false、行 4/ingest 0 timedOut=false（窗内）
等待 65 秒（>20s 超时窗 + 60s 扫描周期）后查询：
publishState=2
3 SMOKE0025_GW 2 timedOut=false err= ACK 超时（超过 20 秒未回执）
4 ingest       2 timedOut=false err= ACK 超时（超过 20 秒未回执）
—— 扫描把逾窗待确认行置 2，error_msg 含「ACK 超时（超过 N 秒未回执）」（N=配置值 20 实测）；
   已置 2 的行 timedOut=false（该派生只针对待确认行）。
```

**步骤 7：重发 → 重置待确认**

```
POST /iot/publish/resend {"versionId":23,"targetId":"SMOKE0025_GW"}
{"code":200,"data":{"id":3,"targetType":1,"targetId":"SMOKE0025_GW","ackStatus":0,"ackTime":null,
 "errorMsg":null,...,"updateTime":"2026-11-22 23:49:39"}}
立即查询 v23：行 3 SMOKE0025_GW 0 timedOut=true（create_time 已逾 20s 窗——派生即时标红，覆盖扫描间隙）
              行 4 ingest 2 err=ACK 超时... publishState=2（ingest 行仍失败）
0→0 再重发（未回执但想重推）：{"code":200,"data":{...ackStatus":0...}} 200
—— 重置 ack_status=0、error_msg=NULL、update_time 刷新全部实证；超时窗内再未回执会被下轮
   扫描重新标 2（本例 65 秒后复查 v23 即如此——语义一致，非缺陷）。
```

**步骤 8：计算 scope 发布 → 一行**

```
POST /iot/publish/publish {"scopeType":2,"scopeId":0} → versionId=24 version=1 scopeId=0
GET /iot/publish/log/list?versionId=24
{"code":200,"data":{"versionId":24,"scopeType":2,"scopeId":0,"version":1,"publishState":0,
 "rows":[{"id":5,"targetType":3,"targetId":"calc","ackStatus":0,"timedOut":false}]}}
—— 计算行（target_type=3 / target_id=calc）恰好一行。
```

**步骤 9：重发已生效行 → 400**

```
POST /iot/publish/resend {"versionId":22,"targetId":"SMOKE0025_GW"}（该行 ack_status=1）
{"code":400,"data":null,"message":"已生效无需重发"}
```

**补充断言（计算行回执 + 全部生效汇总 + 定位一致性）**

```
POST /internal/v1/config/ack（三元组定位 {"scopeType":2,"scopeId":0,"version":1,"targetType":3,
 "targetId":"calc","ackStatus":1}）→ {"code":200,"data":true}
GET /iot/publish/log/list?versionId=24 → publishState=1（calc 1 ackTime=2026-11-22 23:50:00）
   —— 版本级汇总「全部 ACK 生效 → 已生效」口径实证
GET /iot/publish/log/list?versionId=23&scopeType=2&scopeId=0&version=1
{"code":400,"data":null,"message":"versionId 与 (scopeType+scopeId+version) 不一致"}
计算 scope 三元组重发：POST /iot/publish/resend {"scopeType":2,"scopeId":0,"version":1,
 "targetId":"calc"} → {"code":200,"data":{...ackStatus":0...}}
```

**验收 3 全序列通过。**

### 验收 4：prod 档真实启动回归

启动（本地回归口径，环境变量注入本地连接与密钥）：

```
cd backend/ot-admin && DB_HOST=127.0.0.1 DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=127.0.0.1 REDIS_PORT=6379 REDIS_DATABASE=0 SM4_KEY=0123456789abcdef \
CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 WECHAT_MP_ENABLED=false \
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

启动日志：

```
2026-11-22T23:51:41.392+08:00  INFO ... The following 1 profile is active: "prod"
2026-11-22T23:51:55.551+08:00  INFO ... Started OtAdminApplication in 14.975 seconds (process running for 15.877)
ERROR_COUNT=0 （`grep -c ' ERROR '` = 0；查询前与查询后两次复核一致）
```

prod 下认证走正式链：`GET /auth/captchaImage`（取 uuid）→ WSL redis-cli 读 `captcha:code:<uuid>`（码 DCY7）→ SM4 传输密文（jshell + hutool/BouncyCastle 对 admin123 做 encryptHex 后 Base64）→ `POST /auth/login` 登录成功：

```
{"code":200,"data":{"nickname":"管理员","token":"7ac11517-...","userId":1,"username":"admin"},...}
```

下发状态查询断言（本任务新增接口）：

```
GET /iot/publish/log/list?versionId=22
{"code":200,"data":{"versionId":22,...,"publishState":2,"rows":[
 {"id":1,"targetType":1,"targetId":"SMOKE0025_GW","ackStatus":1,"ackTime":"2026-11-22 23:47:42",...},
 {"id":2,"targetType":2,"targetId":"ingest","ackStatus":2,"errorMsg":"smoke failure: snapshot hash mismatch",...}]}}
GET /iot/publish/log/list?versionId=23 → code 200 publishState=2（两行 2）
内部端点 prod fail-closed 复核（未注入 OT_GATEWAY_INTERNAL_TOKEN）：
POST /internal/v1/config/ack → {"code":401,"data":null,"message":"内部令牌未配置，回写接口拒绝访问（fail-closed）"}
```

**通过**：prod 档真实启动 0 ERROR + /iot/publish/log/list 200。部署/配置类加严条款满足：application.yml（4 个新键在 prod 生效——超时扫描 scheduling 线程随 prod 启动、查询派生走新键默认值）与升级 SQL（验收 2 已在库验证）。验毕后 backend 已停止（8098 释放确认 False，纪律保持）。

### 验收 5：改动范围核对

见「一、改动文件清单」——`git diff --name-only da830e4..HEAD` 24 文件全部落在 brief 声明范围内，application.yml 仅新增允许键族，**通过**。

---

## 三、未做项与原因

1. **config.changed 真实通知与 /internal/v1/config/snapshot 下游拉取**：brief D8 明确归 TASK-0026，本任务只留 `NotifySender` 接缝（NoOp 占位，javadoc 载明接线义务）——计划内未做，非缺口。
2. **前端页面**：brief D8「不做前端页面（FEAT-0008 触点=后台）」——未做。
3. **5002 错误码使用**：brief「必读」段称「5002 下发未确认（ResultCode 已有）」，实测 ot-core ResultCode **无** 5002 枚举；且本任务全部接口裁决（ACK 拒绝/重发已生效/定位失败）均按 brief 自己的接口决策走 400，无需 5002。ot-core 属改动范围禁区，故未登记该枚举——5002 的登记义务照 0024 交付时 javadoc 口径归 TASK-0015（确认门专用码），报告在此登记口径差异，未做任何代码改动。
4. **TDengine**：本任务发布链不涉 TDengine 写路径（快照组装数据源为 MySQL 业务表），按协调者指令不触碰。

## 四、偏离计划之处

1. **多租户插件旁路（本表特例，登记为设计决策）**：brief DDL 逐字自上游 p33，`iot_config_publish_log` **无 tenant_id 列**——而 OtTenantLineHandler 对全部 iot_ 前缀表在会话有租户时注入 tenant_id（无该列即 SQL 报错）。处理：`IotConfigPublishLogMapper` 接口级 `@InterceptorIgnore(tenantLine = "true")`（照 IotGatewayCodeMapper / IotDataItemRefMapper 既有旁路范式）；租户边界由服务层保证——管理面（状态查询/重发）先按会话租户解析 version 行（iot_config_version 带租户注入，跨租户 versionId 解析不到即 400）再按 version_id 取本表行；内部 ACK 无管理面数据权限（brief D7 原文）；超时扫描系统级全表（无会话调度线程，上游 DDL 的天然语义）。已在 mapper javadoc 与升级脚本头注双处登记；dev 冒烟 INSERT/SELECT/UPDATE 全链路在会话租户下成功即旁路端到端实证。
2. **PublishScopeGuard 上提（对 0024 交付文件的小重构）**：下发状态查询/重发与发布/回滚/列表/取回同数据权限防线（brief D7「照 0024 范式」）——为防两份防线实现漂移，将 IotPublishServiceImpl 的私有 normalize/assertScope* 方法逐字平移为共享组件 `publish/support/PublishScopeGuard`，0024 侧改为委托（行为不变，既有 23 个测试全绿背书）。
3. **失败回执对已生效行的响应形态**：brief 未规定返回码，定为 **200 + data=false**（忽略不应用，非错误语义）——过期失败回执是分布式常态竞态，按错误处理会诱导内部调用方误重试；终态保护由「先查后跳 + 条件更新 WHERE ack_status IN (0,2)」双道保证。
4. **空行集的 publishState**：brief 三态未覆盖「版本存在但无下发记录行」（0024 冒烟遗留版本），定为 0 下发中（保守口径——非全部生效即不亮绿），javadoc 登记。
5. **冒烟中扫描先行于显式失败回执（步骤 5 时序）**：超时窗调短至 20s 后，v1 ingest 行在显式 ACK 前已被扫描标 2（服务日志 scheduling 线程实证）——后续失败回执为 2→2 覆盖（语义允许）。不影响验收有效性，反而提前实证了扫描真实运转。
6. **重发后的 timedOut 行为**：逾窗行被重发重置 0 后，若未在超时窗内回执，timedOut 派生立即为 true（且下轮扫描重新置 2）——brief D6 timedOut 口径「ack_status=0 且已超超时窗」的字面语义，如实实现；对 UI 阶段的展示含义（重发后红色立即复现）登记为观察项，非缺陷。
7. **TDD 红/绿证据形态**：红阶段以「测试先行 + stub 实现」跑出 9F+14E（UnsupportedOperationException），随后真实实现转绿——非纯「无实现编译失败」形态，留痕更完整。
8. **环境备注**：mvn spring-boot:run 的 fork 子进程不随外层任务终止——dev/prod 两次停止后均需按 8098 监听 PID 手动 Stop-Process 释放（已两次确认端口 False）；冒烟快照产物（backend/ot-admin/data/snapshots/ 下 3 文件）已清理（.gitignore 的 backend/*/data/ 已覆盖该路径——0024 报告遗留问题 1 已在仓内修复）。

## 五、遗留问题

1. **[接线义务（brief D5 声明）]** NotifySender 为 NoOp 占位：TASK-0026 交付真实 config.changed 通知（MQTT/消息通道，只带版本号与哈希）与 /internal/v1/config/snapshot 下游拉取时替换该 Bean，发布落行后/重发重置后的调用点不扩散——计划内占位，非缺陷。
2. **[口径差异登记]** brief 称 ResultCode「已有 5002 下发未确认」——实测无此枚举；本任务未使用亦未登记（ot-core 禁区），登记义务维持 0024 时口径归 TASK-0015。若后续任务确需 5002，请在该任务内扩 ot-core。
3. **[观察项]** 重发重置后的行在超时窗内未回执即再次被扫描标 2（timedOut 派生同步标红）——符合「超时未 ACK 标红可重发」闭环，但 UI 阶段需向用户明示「重发后须在窗内等待回执」，避免「刚重发就又红」的误解（可考虑前端倒计时提示）。
4. **[观察项]** 超时扫描按固定周期对历史逾窗行重复标红（幂等不重复更新已置 2 行，但 0 行重置回 0 后若长期无人处理会周期性翻红）——与上游「扫描幂等」语义一致，无数据副作用；运维侧建议对 error_msg=超时的行设告警阈值。
5. **[环境]** 同 0024 报告登记的 WSL2 空闲关停问题（本次以周期性 `wsl docker ps` 探活规避，未复发）。

## 六、一句话结论

**通过**——验收 1（2205 测试 0 失败 0 错误 BUILD SUCCESS，TDD 红 9F+14E→绿 67 全过留痕）、验收 2（升级脚本两次 DONE 幂等、idx_version/idx_ack 与 4074 种子在位）、验收 3（发布落行两行/一行、状态查询三态可查、无令牌 401/ackStatus=0 400、ACK 置生效+ack_time、失败回执+终态不倒退、扫描置 2 error_msg 含「ACK 超时（超过 20 秒未回执）」、重发重置 0、计算 scope 一行、重发已生效行 400，全序列原始输出为证）、验收 4（prod 档真实启动 0 ERROR + 正式登录链 + /iot/publish/log/list 200 + fail-closed 401）、验收 5（24 文件改动全落 brief 声明范围、application.yml 仅 4 个允许键）五条全部以原始输出为证通过；偏离仅为计划内接缝占位（NotifySender NoOp）、既有范式内的多租户旁路与防线上提共享，遗留仅观察项与 5002 口径差异登记，无阻断缺陷。
