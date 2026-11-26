# TASK-0026 验收报告：config.changed 通知、快照拉取接口与回滚（发布链收官）

> 执行者：TASK-0026 执行；验收日期：2026-11-23；工作区 D:\workspace\WisFactory（main 分支）。
> 开工 BASE = **d08ec02**（`git rev-parse --short HEAD`，与协调者记录一致）；实现提交 = **2e4a1b9**（21 文件）。
> 环境：WSL docker wisfactory-mysql（MySQL 8.0.42，127.0.0.1:3307）/ wisfactory-redis（6379）；8098 启动前探测空闲、用毕即停（dev×2 轮 + prod×2 轮，四次手动释放均确认端口归零）；TDengine 未触碰；本地无 EMQX——通知按 brief D5 mock 口径（MqttPublisherPort 薄接口隔离 paho，单测 mock port），真 broker 验证登记 TD-0056。

---

## 一、改动文件清单（验收 5：改动范围核对）

`git diff --name-only d08ec02..HEAD` 原始输出（21 文件）：

```
backend/ot-admin/pom.xml
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/config/PublishProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/controller/IotPublishInternalController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/ChangeEntryVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/ChangeItemVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/ChangeSummaryVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/ChangesListVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/InternalSnapshotVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/MqttNotifySender.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/MqttPublisherPort.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/NoOpNotifySender.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/NotifySender.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/notify/PahoMqttPublisher.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/IotPublishService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishLogServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/controller/IotPublishInternalControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/notify/MqttNotifySenderTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishLogServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImplInternalQueryTest.java
```

**比对结论：通过。** 全部落在 brief「改动范围」声明内：`/iot/publish/**`（主代码 14 文件：notify 4 + dto 5 + service 3 + controller 1 + config 1）、`src/test/.../iot/publish/**`（测试 4 文件，新增 2 + 调整 2）、`backend/ot-admin/pom.xml`（仅新增 paho 依赖）、`application.yml`（仅新增 ot.publish.notify.* 与 ot.publish.changes-* 键族）。不碰 frontend/、ingest/、openapi、dataperm、credential/gateway/channel 域、ot-core。

**无新 SQL 核对**（验收 2 前半）：`git diff --name-only d08ec02..HEAD | grep -c "resources/sql"` → **0**（无任何 resources/sql/** 文件出现）。

application.yml 新增行（逐行 diff 核对，验收 2 后半——仅允许键族）：

```
+    changes-max-span: ${OT_PUBLISH_CHANGES_MAX_SPAN:10}   # 增量摘要最大跨度（TASK-0026）：...严格大于本值 → needFullSync=true...
+    notify:
+      enabled: ${OT_PUBLISH_NOTIFY_ENABLED:false}   # config.changed MQTT 通知开关（TASK-0026 默认 false——未启用走 NoOp 语义路径...）
+      broker-url: ${OT_PUBLISH_NOTIFY_BROKER_URL:}   # EMQX broker 地址...
+      username: ${OT_PUBLISH_NOTIFY_USERNAME:}   # broker 用户名（空=匿名）
+      password: ${OT_PUBLISH_NOTIFY_PASSWORD:}   # broker 口令（空=匿名；经环境变量注入——凭据不进快照/通知载荷，ADR-0006）
+      client-id: ${OT_PUBLISH_NOTIFY_CLIENT_ID:}   # MQTT clientId（空=进程内随机...）
+      keep-alive-seconds: ${OT_PUBLISH_NOTIFY_KEEP_ALIVE_SECONDS:60}   # MQTT keep-alive（秒）
+      timeout-seconds: ${OT_PUBLISH_NOTIFY_TIMEOUT_SECONDS:30}   # MQTT 连接超时（秒）；连接懒建立+自动重连+句柄复用...
```

（恰为 `ot.publish.changes-max-span` ×1 + `ot.publish.notify.*` ×7——无任何其他键改动。）

pom.xml 新增行（仅 paho，版本显式钉死）：

```
+        <!-- MQTT 客户端（TASK-0026 config.changed 下行通知，ot.publish.notify.enabled=true 时启用；
+             版本显式钉死——1.2.5 为 org.eclipse.paho.client.mqttv3 最新稳定版） -->
+        <dependency>
+            <groupId>org.eclipse.paho</groupId>
+            <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
+            <version>1.2.5</version>
+        </dependency>
```

---

## 二、每条验收命令原始输出

### 验收 1：单测全量（BUILD SUCCESS、0 失败 0 错误）

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`（在最终提交代码 2e4a1b9 上完整重跑；中途曾因冒烟发现的 Jackson 3 出站序列化问题修补后复跑，本节为准最终轮）

ot-core install 段：BUILD SUCCESS（静默 -q 无 ERROR）。

ot-admin 测试 Results 段（全量日志尾段）：

```
[INFO] Tests run: 2244, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:03 min
[INFO] Finished at: 2026-11-23T00:42:36+08:00
```

**通过**：2244 通过 / 0 失败 / 0 错误 / 0 跳过（0025 交付时 2205，本任务净增 39 个测试用例：MqttNotifySenderTest 13 + IotPublishServiceImplInternalQueryTest 17 + 内部控制器测试新增 9）。

TDD 红转绿留痕（brief 测试口径）：先行落测试（MqttNotifySenderTest / IotPublishServiceImplInternalQueryTest），stub 实现（UnsupportedOperationException）阶段首跑为红——

```
[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotPublishInternalControllerTest（薄透传先行实现，绿）
[ERROR] Tests run: 13, Failures: 4, Errors: 9, Skipped: 0 <<< FAILURE! -- in ...MqttNotifySenderTest
[ERROR] Tests run: 17, Failures: 8, Errors: 9, Skipped: 0 <<< FAILURE! -- in ...IotPublishServiceImplInternalQueryTest
[INFO] Results:
[ERROR] Tests run: 45, Failures: 12, Errors: 18, Skipped: 0
[INFO] BUILD FAILURE
```

实现真实逻辑后转绿（目标 4 测试类）：

```
[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotPublishInternalControllerTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0 -- in ...MqttNotifySenderTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotPublishLogServiceImplTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotPublishServiceImplInternalQueryTest
[INFO] Tests run: 68, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

期间修复均附根因与「为什么现在对了」：

1. **PahoMqttPublisher 缺 import / paho 常量名**：`MqttException.REASON_CODE_INVALID_CLIENT` 在 paho 1.2.5 无此常量（为什么现在对了：改用 `REASON_CODE_CLIENT_EXCEPTION`——该语义为「客户端配置/状态异常」，与「broker 未配置即刻上报」匹配，且常量在 1.2.5 真实存在，编译通过）。
2. **受检异常**：接口 `MqttPublisherPort.publish` 声明 `throws Exception`，Mockito verify/doThrow 语法上要经真实签名调用（为什么现在对了：触 publish 的测试方法补 `throws Exception`——编译期受检契约与运行期拦截无关，声明即可）。
3. **注解断言**：`@ConditionalOnProperty.name()` 返回 `String[]`（为什么现在对了：断言 `name()[0]`——与注解元数据真实形态对齐）。
4. **载荷序列化为空对象 `{}`**：NotifyPayload 字段为包私有 final，Jackson 默认字段可见性 PUBLIC_ONLY 不可见（为什么现在对了：字段升 public final——默认字段可见性命中，五键逐字出站）。
5. **毒载荷打错防线**（红转绿后冒烟前单测修正）：首版毒 JSON 用了非法顶层键 `remark`——被终检第一道「顶层键白名单」拦截，未走到凭据关键词（为什么现在对了：毒点改藏合法键 `connConfig` 的嵌套键 `mqttPassword`——专测出站凭据终检本身，而非顶层键白名单）。
6. **冒烟发现的出站序列化缺陷（修正轮，详见「四」7）**：`InternalSnapshotVO.snapshot` 用 Jackson 2 `com.fasterxml JsonNode` 被运行时 Jackson 3（Spring Boot 4 / tools.jackson）按 POJO 序列化成属性散列（冒烟实测输出 `{"array":false,"bigDecimal":false,...}`）。修复：字段类型改 `tools.jackson.databind.JsonNode` + 出站解析用 Jackson 3 `JsonMapper`（为什么现在对了：出站节点与 HTTP 序列化栈同源，JSON 树原样输出——dev 重启复测 snapshot 嵌套对象正确）。

### 验收 2：无新 SQL + application.yml 仅允许键

- 无 SQL：`git diff --name-only d08ec02..HEAD` 中 resources/sql/** 文件数 = **0**（本任务无建表/加列——实现全程未发现需要新表新列的点，快照域/下发记录表 0024/0025 已备齐）。
- application.yml 新增键逐行核对：见「一」——恰为 `ot.publish.changes-max-span` ×1 + `ot.publish.notify.{enabled,broker-url,username,password,client-id,keep-alive-seconds,timeout-seconds}` ×7，全部在 brief 允许键族内。

**通过。**

### 验收 3：dev 冒烟全序列（每步原始输出）

**启动轮 1（notify.enabled=false 默认）**：`mvn spring-boot:run`（默认 dev，注入 `OT_GATEWAY_INTERNAL_TOKEN=tok-0026-smoke`）：

```
2026-11-23 00:30:46.977 [main] INFO ... Started OtAdminApplication in 13.539 seconds
grep -c " ERROR " → 0
```

建链：登录走 `/auth/login/test`（dev 专用）→ 新建网关 SMOKE0026_GW → **id=9111**（status=1）→ 通道 SMOKE0026_CH1（MQTT）→ id=21 → 点位 SMOKE0026_P1（DOUBLE，min=0 max=100，channelId=21）→ **id=12021**。

**步骤 1：发布网关 scope（NoOp 路径）→ 版本 1（versionId=25）**

```
POST /iot/publish/publish {"scopeType":1,"scopeId":9111}
{"code":200,"data":{"id":25,...,"version":1,"snapshotHash":"8e921f8360a080f1...","snapshotUri":"1/1-9111-1.json","publishStatus":1,...}}
服务日志（NoOp 零异常、零 broker 连接尝试）：
00:31:55.527 DEBUG ...NoOpNotifySender - [PUBLISH-NOTIFY] NoOp notify versionId=25 targetType=1 targetId=SMOKE0026_GW
00:31:55.533 DEBUG ...NoOpNotifySender - [PUBLISH-NOTIFY] NoOp notify versionId=25 targetType=2 targetId=ingest
grep -cE "PUBLISH-NOTIFY|MqttException|mqtt" → 仅上述 2 行 NoOp DEBUG（无任何 MQTT 连接痕迹）
```

**步骤 2：置 enabled=true + 假 broker（tcp://127.0.0.1:18830，无监听）重启 → 再发布 → WARN fail-soft 证据**

```
启动轮 2（OT_PUBLISH_NOTIFY_ENABLED=true OT_PUBLISH_NOTIFY_BROKER_URL=tcp://127.0.0.1:18830）：
Started OtAdminApplication in 11.562 seconds；ERROR_COUNT=0
POST /iot/publish/publish {"scopeType":1,"scopeId":9111}
{"code":200,"data":{"id":26,"version":2,"snapshotHash":"0389cbb9a62aa346...","snapshotUri":"1/1-9111-2.json",...}}
—— 发布仍 200（通知失败不阻断）
服务日志（afterCommit 延迟 + 提交后 WARN，发布事务未受扰）：
00:32:48.440 INFO ...MqttNotifySender - [PUBLISH-NOTIFY] config.changed deferred to afterCommit topic=iot/default/SMOKE0026_GW/config versionId=26 version=2 target=1-SMOKE0026_GW
00:32:48.451 INFO ...MqttNotifySender - [PUBLISH-NOTIFY] config.changed deferred to afterCommit topic=iot/config.changed versionId=26 version=2 target=2-ingest
00:32:48.494 WARN ...MqttNotifySender - [PUBLISH-NOTIFY] config.changed publish failed (fail-soft, 发布不阻断——下游拉取对账哈希兜底 REQ-0033) topic=iot/default/SMOKE0026_GW/config versionId=26 version=2 target=1-SMOKE0026_GW err=无法连接至服务器
00:32:48.497 WARN ...MqttNotifySender - [PUBLISH-NOTIFY] config.changed publish failed (fail-soft, ...) topic=iot/config.changed versionId=26 version=2 target=2-ingest err=无法连接至服务器
—— topic 映射实证：网关目标→专属主题 iot/{tenantCode=default}/{gatewayCode}/config；接入→共享主题 iot/config.changed
DB 复核（版本落库 + 两行待确认）：
SELECT id,version_id,target_type,target_id,ack_status FROM iot_config_publish_log WHERE version_id=26 →
8  26  1  SMOKE0026_GW  0
9  26  2  ingest        0
SELECT id,version,publish_status FROM iot_config_version WHERE id=26 → 26 | 2 | 1
```

**步骤 3：curl 内部令牌拉 snapshot（不带 version=最新 / 带 version=指定 / 坏 version=400 / 错令牌 401）**

```
GET /internal/v1/config/snapshot?scopeType=1&scopeId=9111（不带 version → 最新 v2）
{"code":200,"data":{"version":2,"snapshotHash":"0389cbb9a62aa34661c154d4cbcf5b0648ca6b1c1bc05bc0248b56c56be3962c",
 "snapshotUri":"1/1-9111-2.json","schemaVersion":"1.0.1",
 "snapshot":{"schemaVersion":"1.0.1","tenantId":1,"scopeType":1,"scopeId":9111,"version":2,
   "points":[{"id":12021,...,"pointCode":"SMOKE0026_P1",...,"minValue":0.0,...}],"channels":[...]}}}
—— 快照体为嵌套 JSON 对象（Jackson 3 修复后复测），schemaVersion 一并下发
GET .../snapshot?scopeType=1&scopeId=9111&version=1 → 200 version=1 hash=8e921f8360a080f1 points=1 min=0.0（指定版）
GET .../snapshot?...&version=99 → {"code":400,"data":null,"message":"指定版本不存在: 99"}
GET .../snapshot（X-Internal-Token: wrong-token）→ {"code":401,...,"message":"内部令牌无效"}
GET .../snapshot（无令牌）→ {"code":401,...,"message":"内部令牌无效"}
```

**步骤 4：changes 查询（diff 摘要 / 空数组 / 基线缺失 needFullSync）**

```
GET /internal/v1/config/changes?scopeType=1&scopeId=9111&sinceVersion=1
{"code":200,"data":{"needFullSync":false,"latestVersion":2,"changes":[
  {"version":2,"hash":"0389cbb9...","publishTime":"2026-11-23 00:32:48",
   "summary":{"added":[],"removed":[],"changed":[{"type":"point","key":"SMOKE0026_P1"}]}}]}}
—— 相邻版本 diff 摘要实证（v1→v2 内容差异=点位对象 configVersion 随版本递增，符合预期）
GET .../changes?...&sinceVersion=2 → 200 needFullSync=false latest=2 changes=0（since≥最新 → 空数组）
GET .../changes?...&sinceVersion=0 → 200 needFullSync=true latest=2 changes=[(2, summary 三段全空)]
（基线版本行不存在——下游状态不可信 → needFullSync 兜底，摘要留空）
```

**步骤 5：ACK 生效 → 变更点位 → 再发布 v3（fail-soft WARN 再证）**

```
POST /internal/v1/config/ack {"versionId":26,"targetType":1,"targetId":"SMOKE0026_GW","ackStatus":1} → {"code":200,"data":true}
POST /internal/v1/config/ack {"versionId":26,"targetType":2,"targetId":"ingest","ackStatus":1} → {"code":200,"data":true}
DB：SELECT ... WHERE version_id=26 → 两行 ack_status=1 ack_time=2026-11-23 00:36:14
PUT /iot/point（minValue 0→5）→ {"code":200}
POST /iot/publish/publish {"scopeType":1,"scopeId":9111} → {"code":200,"data":{"id":27,"version":3,"snapshotHash":"34b6ecb53aa81563...",...}}
WARN×2（versionId=27 网关/接入 topic 与上同形，err=无法连接至服务器）—— 发布 200 不阻断
```

**步骤 6：回滚 → 版本 4（复用旧快照）→ 拉取新版本快照与旧内容一致（E2E 收口）**

```
POST /iot/publish/rollback {"scopeType":1,"scopeId":9111,"targetVersion":1}
{"code":200,"data":{"id":28,"version":4,"snapshotHash":"8e921f8360a080f1...","snapshotUri":"1/1-9111-1.json","publishStatus":1,...}}
—— 新版本行 hash/uri 与版本 1 完全相同（复用旧快照文件，历史版本不改写）
GET /internal/v1/config/snapshot?scopeType=1&scopeId=9111&version=4（内部令牌）
→ 200 version=4 hash=8e921f8360a080f1 min=0.0 uri=1/1-9111-1.json
GET .../version=1 → 200 hash=8e921f8360a080f1 min=0.0 —— **拉取新版本快照与旧内容一致（哈希/体/点位值三方一致）**
DB：SELECT ... WHERE version_id=28 → 12|28|1|SMOKE0026_GW|0 与 13|28|2|ingest|0（回滚=重新下发，两行待确认落行）
服务日志：versionId=28 的 config.changed publish failed WARN×2（网关专属 topic + 共享 topic）
—— 真 sender 上线后回滚即自动重新下发（通知尝试按行触发实证；真 broker 归 TD-0056）
```

**步骤 7：跨度超限 needFullSync（重启注入 OT_PUBLISH_CHANGES_MAX_SPAN=2）**

```
GET /internal/v1/config/changes?scopeType=1&scopeId=9111&sinceVersion=1（latest=4，跨度 3 > 2）
{"code":200,"data":{"needFullSync":true,"latestVersion":4,"changes":[
  {"version":4,"hash":"8e921f83...","publishTime":"2026-11-23 00:36:26",
   "summary":{"added":[],"removed":[],"changed":[]}}]}}
—— 只给 latest 摘要、不给逐版 diff；latest 摘要=基线(v1)→latest(v4) 累计 diff——v4 为回滚复用 v1 快照，
   两体逐字节一致故三段空（needFullSync 语义与回滚复用语义互证的天然案例）
服务日志：[SNAPSHOT] changes needFullSync tenantId=1 scope=1-9111 since=1 latest=4 baselineKnown=true maxSpan=2
```

**验收 3 全序列通过。**（备注：步骤 1-2 在 Jackson 3 出站修复前的构建上采集——NotifySender/NoOp/fail-soft 代码在该修复中零改动，证据有效；步骤 3 起均在新构建（=最终提交代码）上复测。）

### 验收 4：prod 档真实启动回归

干净复跑（本地回归口径，环境变量注入本地连接与密钥；首轮曾因冒烟探针自身打错 captcha 字段产生一条业务异常日志——为使 0 ERROR 证据无歧义，修对字段后完整重跑，本轮为准）：

```
cd backend/ot-admin && DB_HOST=127.0.0.1 DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=127.0.0.1 REDIS_PORT=6379 REDIS_DATABASE=0 SM4_KEY=0123456789abcdef \
CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 WECHAT_MP_ENABLED=false \
OT_GATEWAY_INTERNAL_TOKEN=tok-0026-smoke mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

启动日志：

```
2026-11-23T00:39:43.292+08:00  INFO ... The following 1 profile is active: "prod"
2026-11-23T00:39:55.432+08:00  INFO ... Started OtAdminApplication in 12.588 seconds (process running for 12.943)
ERROR_COUNT=0 （`grep -c ' ERROR '` = 0——含启动与全部探针，全程无 ERROR）
```

prod 下认证走正式链：`GET /auth/captchaImage`（取 uuid）→ WSL redis-cli 读 `captcha:code:<uuid>`（码 KZKS 等）→ SM4 传输密文（javac/java + hutool/BouncyCastle 对 admin123 做 encryptHex 后 Base64）→ `POST /auth/login` 登录成功（token 248d0d83-...）。

prod 各端点断言（本任务新增接口在 prod 生效——部署/配置类加严条款：application.yml 新键族触及）：

```
GET /internal/v1/config/snapshot?scopeType=1&scopeId=9111（内部令牌）
→ {"code":200,version=4,hash=8e921f8360a080f1,schema=1.0.1,points=1,min=0.0}
GET /internal/v1/config/changes?scopeType=1&scopeId=9111&sinceVersion=3
→ {"code":200,needFullSync=false,latest=4,changes=[(4,changed=[point:SMOKE0026_P1])]}
GET /iot/publish/list?scopeType=1&scopeId=9111&pageNum=1&pageSize=10（管理面，正式登录）
→ {"code":200,versions=[(4,8e921f83,1/1-9111-1.json),(3,34b6ecb5,1/1-9111-3.json),
                        (2,0389cbb9,1/1-9111-2.json),(1,8e921f83,1/1-9111-1.json)]}
—— v4 与 v1 hash/uri 完全一致（回滚复用），版本列表 4 条单调递增
```

**通过**：prod 档真实启动 0 ERROR + snapshot 拉取 200。验毕后 backend 已停止（8098 监听确认归零，纪律保持）。

### 验收 5：改动范围核对

见「一、改动文件清单」——`git diff --name-only d08ec02..HEAD` 21 文件全部落在 brief 声明范围内；pom 仅新增 paho 1.2.5、yml 仅新增允许键族、无 resources/sql 文件，**通过**。

---

## 三、未做项与原因

1. **EMQX 真 broker 联调**：本地无 EMQX（GO-LANE-HANDOFF 实测，协调者指令「不要试图连任何真实 broker」）——按 brief D5 mock 口径交付：`MqttPublisherPort` 薄接口隔离 paho，`MqttNotifySenderTest` 13 个单测 mock port 验 topic 映射/载荷五键/fail-soft/afterCommit；连接生命周期（懒建立/自动重连/句柄复用）实现于 `PahoMqttPublisher`（paho 1.2.5），真 broker 端到端验证登记 TD-0056（文案见「五」1）。
2. **5002 错误码**：不在本任务范围（归 TASK-0015 确认门，0024/0025 口径维持）；本任务全部接口裁决按 brief 走 400/500。
3. **灰度/审批流**：REQ-0033 非目标，未实现（brief D7）。
4. **TDengine**：本任务发布链不涉 TDengine 写路径（快照组装与拉取数据源为 MySQL 业务表 + 本地快照存储），按协调者指令不触碰。

## 四、偏离计划之处

1. **0025 评审 Minor ④ 接线约束——裁决=a（afterCommit）**：现 NotifySender 调用点在事务提交前，真实现把**发送动作移至 afterCommit**（`MqttNotifySender` 内 `TransactionSynchronizationManager.isSynchronizationActive()` 判定：事务激活→组装载荷捕获后注册 `afterCommit` 回调，提交后才做网络发送；无事务链路〔重发 resend〕立即发送）。javadoc 已在 `NotifySender` 接口与 `MqttNotifySender` 类注释双处登记取舍与理由（杜绝「通知已发、行已回滚」伪通知；载荷捕获于调用点、回调内零 DB 读取）。冒烟日志 `deferred to afterCommit` ×2 行为证。
2. **NotifySender 接口签名变更**：`notify(Long versionId, ...)` → `notify(IotConfigVersion version, Integer targetType, String targetId)`——载荷五键（tenant 编码/版本号/哈希）需版本行数据，传行免发送方回查（调用点数量与位置不变：落行逐行×2 + 重发×1，均在声明范围内调整；`IotPublishLogServiceImplTest` 相应 verify 改形，23 测试全绿背书）。
3. **内部端点新增可选 `tenant` 查询参数（spec 兼容扩展）**：§3.3 p25 的 `GET /internal/v1/config/snapshot?scopeType=&scopeId=&version=` 与 `changes?sinceVersion=` 未含租户维度——内部链路无会话（多租户插件跳过注入），计算 scope（scopeId=0 恒定）跨租户同名必然歧义。处理：两端点增可选 `tenant`（租户编码，反查 sys_tenant→id 过滤）；缺省且命中多租户行 → 400 fail-closed（不猜租户，与 0025 三元组跨租户歧义同纪律）。spec 三参数全部支持，参数为纯增量。租户编码权威性：sys_tenant 为平台级表（OtTenantLineHandler 白名单外，无租户注入），会话内外一致；通知载荷 tenant 键同源（tenant_id 反查）。
4. **changes 端点补 scope 定位参数**：brief/上游 p25 仅提 `sinceVersion`，但增量摘要本质是 per-scope 快照 diff——实现按 `scopeType`/`scopeId`（必填）+`sinceVersion`（必填）+`tenant`（可选）定位，与 snapshot 端点同口径（spec 参数为子集，不冲突）。
5. **needFullSync 触发面**：brief 列「跨度超限 / 中间快照缺失」两类，实现按同语义另收两类退化：sinceVersion 版本行不存在（下游基线不可信）、版本序列空洞（异常防御）——均为 needFullSync=true 只给 latest 摘要，同一兜底语义面，javadoc 登记。
6. **summary 元素形态**：brief 字面 `summary:{added:[],removed:[],changed:[]}` 保留，元素定为 `{type,key}` 对象（type=point/channel/dataItem/formula/metric；key=pointCode/channelCode/fullKey/行 id/metricCode——公式引用边无业务 code 以行 id 对齐；code 缺失回退行 id），段内/段间按 (type,key) 排序确定性输出。五段计变更对象与类型由元素 type 承载（brief「分五段计」语义保留）。
7. **出站快照体对象化撞上 Jackson 3（修正轮，计划外发现）**：Spring Boot 4 运行时 HTTP 序列化栈为 Jackson 3（tools.jackson）——`InternalSnapshotVO.snapshot` 初版用 Jackson 2 节点被按 POJO 序列化（冒烟实测输出属性散列）。修复：字段类型与出站解析切 tools.jackson（`parseSnapshotBody` 用 Jackson 3 `JsonMapper` + 捕获 unchecked `JacksonException`）；diff/摘要路径仍用 com.fasterxml（与 0024 组装/终检同源，纯服务端内部）。此修正轮在冒烟步骤 3 发现、单测先行修正毒载荷用例后统一转绿，并触发验收 1 在最终代码上重跑（见验收 1 备注）。
8. **出站凭据终检响应码**：复用 0024 `SnapshotCredentialGuard.check`（抛 PARAM_ERROR 400），但出站命中=存储被篡改（服务端故障语义）——服务层捕获后转 ERROR 500 + `[SNAPSHOT] outbound credential guard tripped` ERROR 告警日志（文案保留 ADR-0006 防线语义）。brief 未规定该处响应码，按「不一致 500 告警」同族语义裁决，javadoc 登记。
9. **retrieveSnapshot 管理面取回落库行语义微调（0024 行为不变量内）**：0024 取回 400 用 `ResultCode.NOT_FOUND`（版本不存在），本任务内部端点同场景用 PARAM_ERROR 400（brief 口径「scope 无版本/指定版本不存在 → 400」）——管理面旧路径原样保留（NOT_FOUND 亦 400 族），未动 0024 测试。
10. **冒烟 API 备注**：点位创建踩两处既有校验（单位必须取自 iot_unit 字典→去掉 unit；dataType 为整型枚举非字符串），ASCII 载荷按 0024 经验执行——冒烟建链数据（网关 9111/通道 21/点位 12021/版本 25-28）留存库内，与 0024/0025 冒烟遗留同口径隔离。
11. **环境备注**：mvn spring-boot:run fork 子进程不随外层任务终止——dev×2 + prod×2 轮均按 8098 监听 PID 手动 Stop-Process 释放（四次确认归零）；冒烟快照产物（backend/ot-admin/data/snapshots/1/ 下 4 文件）已清理（.gitignore backend/*/data/ 覆盖，git status 无残留）；WSL 保活循环全程未复发空闲关停。

## 五、遗留问题

1. **[TD-0056 登记文案]** TASK-0026 通知真联调缺口：config.changed 下行通知已按 mock 口径交付（MqttPublisherPort 薄接口隔离 paho；topic 映射/载荷五键/fail-soft/afterCommit 由 13 个单测锁定），`PahoMqttPublisher`（paho 1.2.5，懒建立+自动重连+句柄复用）**未经真实 broker 验证**。登记动作：部署/联调环境补 `wisfactory-emqx` 容器后，置 `ot.publish.notify.enabled=true` + `ot.publish.notify.broker-url=tcp://<emqx>:1883`（账号经 OT_PUBLISH_NOTIFY_USERNAME/PASSWORD 注入），执行——①发布网关 scope → 用 mosquitto_sub 订阅 `iot/<tenantCode>/<gatewayCode>/config` 验证 QoS 1 收到五键载荷 `{"tenant","scopeType","scopeId","version","hash"}`；②发布计算 scope → 订阅 `iot/config.changed` 同验；③kill broker 后发布 → 服务日志 WARN fail-soft、发布 200、broker 恢复后自动重连（paho automaticReconnect）；④retain=false 复核（晚接入订阅不收旧信号）。登记人：TASK-0026 执行者；建议归属：接入服务联调任务（或环境就绪后的修订轮）。
2. **[口径登记]** 通知载荷 version 键=业务版本号（非版本行 id）——`MqttNotifySender.NotifyPayload.version` javadoc 已明示「注意非版本行 id」；下游以 (tenant, scopeType, scopeId, version) 三元组或拉取响应信封对账，勿混 versionId。
3. **[观察项]** changes 摘要的 changed 判定=快照体结构化全等（含对象内 configVersion）——同 scope 连续两次「无业务变更」发布会因 configVersion 递增使全量对象计入 changed（冒烟 v1→v2 即此形态）。属快照 Schema 自洽语义（哈希亦随版本变化），非缺陷；下游若以「业务字段 diff」为增量应用依据，可按 type+key 过滤 configVersion 差异，登记为下游对接注意项。
4. **[观察项]** `PahoMqttPublisher` 断连期间的发布失败不做同步重试（单发单弃，WARN 兜底）——paho automaticReconnect 后台恢复后续发正常；极端场景（broker 长时间离线）通知连续丢失由拉取对账兜底（REQ-0033 验收 2 设计语义）。如需投递可靠性升级（离线缓冲重发），登记为二期增强。
5. **[环境]** 同 0024/0025 报告登记的 WSL2 空闲关停问题（本次以 80 分钟保活循环规避，全程未复发）。

## 六、一句话结论

**通过**——验收 1（最终提交代码 2e4a1b9 全量 mvn test：2244 测试 0 失败 0 错误 BUILD SUCCESS，TDD 红 12F+18E→绿 68 目标测试全过留痕，6 项修复均附根因）、验收 2（无 SQL=0 文件、yml 仅新增 changes-max-span×1 + notify.*×7 允许键、pom 仅 paho 1.2.5）、验收 3（dev 冒烟全序列：NoOp 零异常零连接痕迹→enabled=true+假 broker 发布 200 且 afterCommit 延迟+提交后 WARN fail-soft 双 topic 落库两行待确认→内部令牌 snapshot 最新/指定/400/401→changes diff 摘要/空数组/基线缺失 needFullSync→ACK 双行生效→变更点位再发布 v3→回滚 v4 复用 v1 快照（hash/体/uri 三方一致）且通知按行触发→max-span=2 跨度超限 needFullSync 只给 latest 摘要）、验收 4（prod 档干净启动全程 0 ERROR + 正式登录链 + snapshot/changes/list 全 200）、验收 5（21 文件全落 brief 声明范围）五条全部以原始输出为证通过；偏离均为计划内裁决（Minor ④ 决策 a afterCommit、接口签名传行、tenant 消歧参数、needFullSync 同族退化、summary 元素形态、Jackson 3 出站修正轮）与既有范式内小重构，遗留仅 TD-0056 真联调登记与观察项，无阻断缺陷。
