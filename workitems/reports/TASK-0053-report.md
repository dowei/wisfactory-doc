# TASK-0053 验证报告：推送订阅与指标产出事件——数据服务侧订阅消费

- 执行者：TASK-0053 执行者（no-subagents；backend 道——只改 `backend/ot-admin` openapi 模块 + 测试）
- **BASE（开工 HEAD）= `ebd443d`**（`git rev-parse --short HEAD`，2026-11-24 main，brief「BASE=当前 HEAD」口径）；**交付提交 = `a230f91`**（注明 TASK-0053，`git add` 只含 openapi 模块 22 文件）
- 并行事实：BASE 之后、本任务提交之前，并行 TASK-0047（calc 道）提交 `f19edc1` 落入主仓（19 文件全 `calc/**`）；本任务全程未 rebase/改写他人历史，`git diff f19edc1..HEAD` = 22 文件全落本任务声明范围（见 §二-4）
- 执行日期：2026-11-24
- 结论先行：**四条验收全部通过**——① `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 2445/0/0/0 BUILD SUCCESS（既有 0051/0054 openapi 测试逐类零回归 + 新增 22 用例）；② live 本地 MQTT 3.1.1 夹具 broker 真协议订阅全序列绿（数据项 4 字段、指标 9 扩展字段 camelCase/毫秒、全键+窗口幂等 2 发 1 条且以最新为准、租户 B 载荷跨主题丢弃、无订阅者不报错、HMAC 开通订阅成功、全停归零）；③ prod 档真实启动 16.169s + ERROR=0 + 签名真调 200 / 篡改 403+6001；④ 范围核对本提交 22 文件全落 backend openapi 模块+测试。
- 契约闭合：**字段名 camelCase + 时间毫秒时间戳**在本任务正式定版（通过事件 VO javadoc 实现，不改冻结契约文件本体；CONV-03 升版走裁决），见 §五-1。

---

## 一、改动文件清单（提交 `a230f91`，22 项，全部 openapi 模块 + 测试）

### 1.1 新增——事件 VO 与编解码（3）

| 文件 | 职责 |
| --- | --- |
| `dto/ItemChangeEventVO.java` | 数据项订阅消息体 4 字段逐字（fullKey/ts/value/quality），camelCase + ts 毫秒定版（闭合契约 §与上游差异第 7/8 条）；`toMap` 有序锁定字段序。 |
| `dto/MetricProducedEventVO.java` | 指标产出事件 9 扩展字段（metricCode/instanceKey/period/windowStart/windowEnd/value/yoy/mom/quality）camelCase + 窗口毫秒定版；含 tenant/tenantId 双隔离字段、yoy/mom null 如实保留、hash 非幂等键；`idempotencyKey()`=tenant\|metricCode\|instanceKey\|period\|windowStart\|windowEnd（全键+窗口，非 hash，ADR-0010 TD-0030）。 |
| `support/OpenApiEventCodec.java` | 入站 JSON→定版 VO 解析（缺契约必填字段 fail-closed 拒绝、多余/过渡字段忽略）；出站有序序列化（camelCase 保 null）；4/9 字段校验表。 |

### 1.2 新增——订阅消费领域（8）

| 文件 | 职责 |
| --- | --- |
| `subscribe/SubscriptionProperties.java` | `ot.openapi.subscribe.*` 配置：mqtt-enabled/kafka-enabled 默认 false、broker env 可切、clientId 前缀、环形缓冲容量。 |
| `subscribe/SubscriptionConfig.java` | 注册表/缓冲/消费入口/编解码器装配（proxyBeanMethods=false，条件传输通道各自装配）；默认关闭时零连接尝试。 |
| `subscribe/SubscriptionEventStore.java` | 内存环形缓冲 + 幂等登记：指标按全键+窗口、数据项按 fullKey+ts+quality；同键重发以最新覆盖不占位；不写任何业务表（ADR-0010 事件仅触发信号，物化表为准）；无订阅者 accept 恒真不报错。 |
| `subscribe/SubscriptionRegistry.java` | 订阅注册表：订阅时用 0051 `resolveClientScope(clientId, tenantId, scopeJson)` 显式租户重载校验 scope（空默认拒绝，subtree 组件展开），数据项实例解析到可见集合才允许（6003 拒绝）；主题构造/解析逐字（iot/open/{t}/{i}/item、iot/{t}/metric/out、iot.metric.result.{t}）；退订幂等。 |
| `subscribe/SubscriptionConsumer.java` | 事件消费入口（MQTT/Kafka 共用）：主题解析→租户双校验（载荷 tenant 与主题 tenant 不一致丢弃 + WARN）→载荷校验→落缓冲；返回投递计数供冒烟断言。 |
| `subscribe/SubscriptionService.java` + `subscribe/SubscriptionServiceImpl.java` | 订阅管理 + 事件读口门面（开通/关闭订阅、latest/ring 读口）。 |
| `subscribe/PahoSubscriptionBridge.java` | 条件装配（mqtt-enabled=true）：真 Paho 客户端连 broker（QoS1、自动重连、clientId 随机后缀多实例隔离），订阅 `iot/open/+/+/item` 与 `iot/+/metric/out` 通配符，消息转消费入口；连接失败 fail-soft WARN 不阻断启动。 |
| `subscribe/KafkaSubscriptionBridge.java` | 条件装配（kafka-enabled=true，默认关）：演进位占位，真 consumer 接线统一调 `SubscriptionConsumer.onMetricMessage`（同载荷/同幂等）。 |

### 1.3 新增——接口与内部驱动面（3）

| 文件 | 职责 |
| --- | --- |
| `controller/SubscriptionController.java` | `/openapi/v1/subscribe/**` 订阅管理（开通/关闭数据项/指标 + 事件读口）：类级 @Anonymous，走 0051 appKey+HMAC 过滤器全覆盖鉴权+限流+iot_api_call_log 全量审计；订阅动作挂 @OperateLog（机器无登录会话由切面 NULL 身份列落库，BUG-0003 既有行为）；requireIdentity fail-closed。 |
| `controller/SubscriptionObserveController.java` | dev 档内部驱动面 `/internal/v1/subscribe/{item,metric}`（@Profile("dev")，prod 不装配）：live 冒烟注入真协议路径之外的消息到消费入口，仿 0046 `/calc/v1/metric/recalc` 观测缝，非契约命名空间、非 /openapi/v1 前缀不经 HMAC。 |
| `exception/ScopeDeniedException.java` | 订阅 scope 拒绝异常（业务码 6003，与 0051 对外 scope 外同码）。 |

### 1.4 修改——配置与既有测试适配（2）

| 文件 | 改动 |
| --- | --- |
| `application.yml` | `ot.openapi.subscribe` 子块 5 键（mqtt-enabled/mqtt-broker/mqtt-client-prefix/kafka-enabled/ring-capacity），env 全覆盖；原 `ot.openapi.*` 五键与 dataperm 块零改动。 |
| `test/.../openapi/config/OpenApiYmlStructureTest.java` | openapi 块计数 5→6（新增 subscribe 子块仍在 ot.openapi.* 命名空间），并断言子块为 Map——原 I-1 命名空间归属断言不变。 |

### 1.5 新增——契约测试（22 用例，5 类）

| 文件 | 用例数 | 覆盖 |
| --- | --- | --- |
| `subscribe/ItemChangeEventContractTest.java` | 2 | 数据项 4 字段逐字/有序 camelCase/ts 毫秒、多余字段忽略、缺字段拒。 |
| `subscribe/MetricProducedEventContractTest.java` | 4 | 9 扩展字段 camelCase 逐字段、窗口毫秒、null yoy/mom 保留、序列化键序锁 snake_case 不出现、缺字段拒。 |
| `subscribe/EventRingBufferContractTest.java` | 5 | 全键+窗口幂等（值/hash 变化仍算重发、以最新为准）、不同窗口/指标/实例独立、数据项幂等与分流、环形有界、无订阅者不报错。 |
| `subscribe/SubscriptionIsolationContractTest.java` | 9 | scope 内/外/空拒绝、数据项主题租户隔离、指标主题+载荷双校验、Kafka 主题同口径、退订幂等、主题解析容错。 |
| `config/SubscriptionPropertiesTest.java` | 2 | MQTT/Kafka 默认关、broker env 可切、容量归一；yml subscribe 子块键名逐字 + 原五键不漂移。 |

计数：2+4+5+9+2=22 新增用例；全量 2423（0054 报告口径）→ 2445（含并行 0047 calc 增量与 0052/0058 等既有提交后的当前实测基线），本任务实测 2445/0/0/0（见 §二-1）。

---

## 二、验收命令原始输出

### 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 0 失败 0 错误

前置 `mvn install -pl ot-core -DskipTests -q` 成功（无输出）；终轮全量 Results 段（提交 `a230f91` 对应源码）：

```
$ cd backend && mvn install -pl ot-core -DskipTests -q
（无输出 = 成功）

$ mvn test -pl ot-admin
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 2445, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:55 min
[INFO] Finished at: 2026-11-24T06:20:25+08:00
[INFO] ------------------------------------------------------------------------
EXIT=0
```

**既有 0051/0054 测试逐类零回归**（终轮 openapi 包 Results 段，逐类计数与 0051/0054 报告一致）：

```
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.audit.OpenApiAuditWriterTest
[INFO] Tests run: 22, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.auth.HmacAuthStrategyTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.auth.OpenApiSeedLiteralTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.config.OpenApiYmlStructureTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.config.SubscriptionPropertiesTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.filter.OpenApiAuthFilterTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.management.OpenApiClientAuditNoPlaintextTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.management.OpenApiClientMenuPermissionGuardTest
[INFO] Tests run: 18, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.management.OpenApiClientServiceTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.mapper.OpenApiDictSyncMapperSqlTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.ratelimit.OpenApiRateLimiterTest
[INFO] Tests run: 31, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.service.impl.OpenApiQueryServiceImplTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.service.impl.OpenApiStatusQueryTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.subscribe.EventRingBufferContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.subscribe.ItemChangeEventContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.subscribe.MetricProducedEventContractTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.subscribe.SubscriptionIsolationContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.support.OpenApiRealtimeStoreTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.support.OpenApiTimeseriesClientTest
```

- HmacAuthStrategyTest 22 / OpenApiQueryServiceImplTest 31 / OpenApiDictSyncMapperSqlTest 3 / OpenApiAuthFilterTest 7 等 0051/0054 既有类逐类计数与各自报告终轮一致——零回归。新增 22 用例（上表 5 类）+ OpenApiYmlStructureTest 结构适配（2 例断言语义不变）。

**TDD 红转绿留痕（每项修附「为什么现在对了」）**：

1. **RED（编译期）**：先写 5 个契约测试类 → `mvn test -Dtest=...` `[ERROR] COMPILATION ERROR`（找不到符号 `ItemChangeEventVO` / `MetricProducedEventVO` / `OpenApiEventCodec` / `SubscriptionEventStore` / `SubscriptionRegistry` / `SubscriptionConsumer` / `SubscriptionProperties` / `ScopeDeniedException` 等 20+ 处）。GREEN：实现上述类后编译通过、24/24 聚焦测试绿（含 OpenApiYmlStructureTest）。
2. **RED（真实断言）**：首轮 `SubscriptionIsolationContractTest` 编译错——测试局部变量声明 `Subscription s` 而该 record 是 `SubscriptionRegistry.Subscription` 嵌套类型。GREEN：测试改为 `SubscriptionRegistry.Subscription s`。为什么现在对了：类型引用与声明处一致，记录类型属于注册表嵌套命名空间，不另建顶层类型。
3. **RED（live 启动）**：dev 档启动 `BeanDefinitionStoreException: Could not enhance configuration class [SubscriptionConfig] ... No visible constructors`（私有无参构造 + CGLIB 增强失败）。GREEN：`@Configuration(proxyBeanMethods=false)` + public 构造。为什么现在对了：该类各 @Bean 无互调（无 inter-bean reference），关闭代理无需 CGLIB 增强，且 public 构造可被容器/增强器构造——prod/dev 启动均通过（§二-3）。
4. **RED（live 协议夹具）**：首轮 live 冒烟 S3 数据项事件「载荷非合法 JSON」——根因在**冒烟脚本**（非主码）：QoS1 PUBLISH 自行多加 2 字节假 packet-id `P1`，夹具正确剥离 2 字节真 packet-id（`\x00\x01`）后把 `P1` 当载荷前缀转给 Paho。GREEN：去掉多余 `b'P1'`，QoS1 报文严格按 MQTT 3.1.1 变长头（topic + 2 字节 packet id + payload）。为什么现在对了：publish 报文与订阅端解析口径一致，真协议栈（Paho）收到逐字载荷，S3-S6 全绿。

### 验收 2：live 冒烟全序列（本地 MQTT 3.1.1 夹具 broker，每步原始输出）

**步骤 0：起前端口 + 夹具 broker + dev 启动（env 切 broker）**

```
PORT_18888_FREE
PORT_8098_FREE
FIXTURE_BROKER_READY 127.0.0.1:18888
OK   | FIXTURE_BROKER_READY
OK   | DEV_APP_READY  after wait
=== app log（真协议栈）===
INFO  PahoSubscriptionBridge - [OPENAPI-SUB] MQTT 订阅已建立 broker=tcp://127.0.0.1:18888 clientId=ot-admin-sub-5b960c46 topics=iot/open/+/+/item,iot/+/metric/out
```

> 环境事实：EMQX 192.168.31.145:1883 当前 MQTT 握手退化（0046 同窗口实证 TCP 可连但 CONNECT 无 CONNACK）；本任务按 brief 以本地 MQTT 3.1.1 夹具 broker 完成真协议订阅/发布全序列——Paho 消费侧与冒烟发布侧均为真协议栈，仅 broker 地址由 env `OT_OPENAPI_SUB_MQTT_BROKER` 切换，真机恢复后免改码复验（§三-1）。

**步骤 1–6：HMAC 订阅 + 真发事件 + 幂等 + 隔离 + 无订阅者（逐条原始输出）**

```
OK   | S1 开通数据项订阅（HMAC）  200 {'code': 200, 'data': {'topic': 'iot/open/default/F01/item'}, ...}
OK   | S1b 订阅主题逐字  iot/open/default/F01/item
OK   | S2 开通指标订阅（HMAC）  200 {'code': 200, 'data': {'mqtt': ['iot/default/metric/out'], 'kafka': ['iot.metric.result.default']}, ...}
OK   | S2b MQTT 主题逐字
OK   | S2c Kafka 主题逐字
OK   | S3 数据项事件真订阅落缓冲  1
OK   | S3b 4 字段逐字  [{'fullKey': 'F01.M01', 'quality': 0, 'ts': 1795211995000, 'value': 12.5}]
OK   | S4 全键+窗口幂等（2 发 1 条）  count=1
OK   | S4b 同窗重发以最新为准  150.0
OK   | S4c 扩展字段 metricCode  M_LIVE
OK   | S4c 扩展字段 instanceKey  F01.ELEC_METER
OK   | S4c 扩展字段 period  DAY
OK   | S4c 扩展字段 windowStart  1795305600000
OK   | S4c 扩展字段 windowEnd  1795392000000
OK   | S4c 扩展字段 value  150.0
OK   | S4c 扩展字段 yoy  0.25
OK   | S4c 扩展字段 mom  -0.2
OK   | S4c 扩展字段 quality
OK   | S4d 窗口毫秒  1795305600000
OK   | S5 租户 B 载荷跨主题丢弃（双校验）  1
OK   | S6 无订阅者不报错且不落缓冲
SMOKE_SEQUENCE_DONE
```

- S4 两条实收/落缓冲序列：首条 `value=100.0/hash=h1` → 修正重发 `value=150.0/hash=h2`（**同窗口 windowStart=1795305600000 / windowEnd=1795392000000**）——按全键+窗口只落 1 条且以 150.0 为准（若用 hash 为幂等键会落 2 条，实证非 hash 口径）。
- S5 载荷把 `tenant` 改为 `other` 但从 `iot/default/metric/out` 发入——消费侧丢弃并 WARN，缓冲仍 1 条（租户 B 收不到租户 A 的载荷面）：
  ```
  WARN SubscriptionConsumer - [OPENAPI-SUB] 指标事件租户不一致丢弃 topicTenant=default payloadTenant=other topic=iot/default/metric/out
  ```
- 审计链实证（0051 既有 iot_api_call_log，订阅动作全量落库，HMAC 链生效）：
  ```
  api_path                                result_code  tenant_id  app_key
  /openapi/v1/subscribe/metric            200          1          test_client
  /openapi/v1/subscribe/metric/events     200          1          test_client
  /openapi/v1/subscribe/item/events       200          1          test_client
  /openapi/v1/subscribe/item/F01          200          1          test_client
  ```
- dev 档全程 ERROR=0（`grep -c ' ERROR ' app.log` = 0）。

**步骤 7：全停归零**

```
PORT_18888_AFTER=FREE
PORT_8098_AFTER=FREE
ALL_STOPPED
```

### 验收 3：prod 档真实启动回归 → 0 ERROR

启动前探测 `PROD_PROBE_FREE`（netstat :8098 无 LISTENING）；打包沿用 `ot-admin-1.0.0.jar`（终码）；启动命令（0051/0054 确认口径 + 三密钥注入）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

原始输出：

```
2026-11-24T06:31:45.232+08:00  INFO 93316 --- [ot-framework] [main] o.s.boot.tomcat.TomcatWebServer : Tomcat started on port 8098 (http) with context path '/'
2026-11-24T06:31:45.250+08:00  INFO 93316 --- [ot-framework] [main] net.otlab.admin.OtAdminApplication : Started OtAdminApplication in 16.169 seconds (process running for 16.707)
--- liveness ---
200
--- ERROR count（启动段 + 探针后）---
0
0
PROD subscribe read: (200, '{"code":200,"data":[],"message":"操作成功",...}')
PROD tamper: (403, '{"code":6001,"message":"签名不匹配",...}')
PROD_PID=93316  成功: 已终止 PID 为 93316 的进程
PORT_8098_FREE
```

- 默认档（mqtt/kafka 均 false）下 PahoSubscriptionBridge / KafkaSubscriptionBridge 条件装配不触发，prod 启动零连接尝试、ERROR=0；签名真调 200（订阅读口空缓冲如实）+ 篡改 403+6001（0051 鉴权链 prod 档等价生效）。
- `/actuator/health` 聚合 503（MailHealthIndicator 对 prod 默认 smtp.example.com，0051 已登记非本任务引入）；liveness 200、应用本体健康，不阻塞验收。

### 验收 4：范围核对

```
=== BASE / 提交 ===
BASE（开工）= ebd443d
并行 0047 提交（BASE 之后）= f19edc1（19 文件全 calc/**）
本任务提交 = a230f91（22 文件，注明 TASK-0053）
=== 本提交文件数 / 越界 ===
git show --name-only --format="" a230f91 | grep -c .  → 22
git show --name-only --format="" a230f91 | grep -v '^backend/ot-admin/' → 空（ALL IN backend/ot-admin）
git diff --name-only f19edc1..HEAD → 22 项，全部 backend/ot-admin openapi 模块+测试/application.yml
=== 工作区（开工既存 + 并行在途，本任务未触碰）===
M calc/**（TASK-0047 并行在途，未 add/未改）
?? .agents/  ?? .tmp-t0044v/  ?? .tmp-t0047r/  ?? .tmp-t0053/（本任务临时冒烟目录，测后删除）
```

- `git diff ebd443d..HEAD` 含并行 0047 的 calc 文件（非本任务）——按提交粒度（f19edc1 并行 / a230f91 本任务）区分归属，本任务提交越界 0。

---

## 三、未做项与原因

1. **真 EMQX broker 192.168.31.145:1883 上的订阅实收**：本执行窗口 broker MQTT 握手退化（CONNECT 无 CONNACK，0046 同窗口实证），属环境侧退化非实现缺陷。按 brief 以本地 MQTT 3.1.1 夹具 broker 完成真协议层 subscribe/publish 全序列（Paho 消费侧 + 裸协议发布侧均为真协议栈）；broker 地址 env 可切（`OT_OPENAPI_SUB_MQTT_BROKER`，默认 `tcp://192.168.31.145:1883`），**恢复后同一交付无需改码真机复验**。补验条件见 §三末。
2. **Kafka 真 broker 消费**：本环境无 Kafka broker（brief 明定），`ot.openapi.subscribe.kafka-enabled` 默认 false——`KafkaSubscriptionBridge` 为条件装配演进位占位，真 consumer 接线统一调 `SubscriptionConsumer.onMetricMessage`（同载荷/同幂等/同租户校验）；契约测试覆盖 Kafka 主题解析同口径（live S2c 返回 `iot.metric.result.default`）。真 consumer（bootstrap/序列化/分区键）为演进位。
3. **字段 style 变更需升契约修订号**：本任务按 brief 定版 camelCase/毫秒并 javadoc 登记；冻结契约文件本体未改（brief 硬约束）。若后续中台对接要求 snake_case/字符串时间，需走 CONV-03 升版裁决并双方同步（§五-1）。
4. **多实例共享订阅表/环形缓冲/幂等状态**：当前为进程内（同 0046 幂等/留痕口径），多实例 Redis 化/持久化为演进位；租户内主题隔离与载荷双校验在单进程已实证，多实例共享归联调阶段。
5. **反向同步（中台建点回传）**：ADR-0010 TD-0032 一期不做，brief 明文不预留写接口，本任务零触碰。
6. **TASK 状态**：按硬约束未把 TASK-0053 置「已完成」，保持「进行中」待评审；未改 workitems 库（唯一例外：本报告）。

**环境恢复后真机补验条件（用户侧动作）**：

1. EMQX 1883 恢复（CONNECT 正常回 CONNACK 后）：`OT_OPENAPI_SUB_MQTT_ENABLED=true OT_OPENAPI_SUB_MQTT_BROKER=tcp://192.168.31.145:1883` 起 ot-admin，POST `/openapi/v1/subscribe/metric`（HMAC），外部 MQTT 发布 `iot/{tenant}/metric/out`，GET `/openapi/v1/subscribe/metric/events` 断言全载荷；本批交付已具备该能力，仅差环境。
2. Kafka broker 就绪后：设 `OT_OPENAPI_SUB_KAFKA_ENABLED=true` 并替换 `KafkaSubscriptionBridge` 为真 consumer，订阅 `iot.metric.result.{tenant}` 断言同载荷/同幂等。

---

## 四、偏离计划之处

1. **live broker 用本地 MQTT 夹具而非真 EMQX**（非缺陷，环境规避）：真 broker 握手退化为环境阻断，按 0046 先例用本地 MQTT 3.1.1 夹具完成真协议层验证；消费侧 Paho 真客户端 + 发布侧裸协议真栈，仅地址 env 切换，恢复后无需改码真机复验（§三-1）。
2. **新增 dev 内部驱动面 `/internal/v1/subscribe/{item,metric}`**（观测驱动缝）：brief live 冒烟需「真发数据项事件+指标事件」，但夹具 broker 真协议路径已覆盖发布侧；本端点仅作 dev 档额外注入缝（仿 0046 `/calc/v1/metric/recalc`），prod 档 `@Profile("dev")` 不装配，不属对外契约面，主链消息仍由 Paho 从 broker 收取（本批 live S3-S6 即真协议路径实收，观测面只用于读缓冲断言）。
3. **订阅管理接口放在 /openapi/v1 下**（brief 决策 5「走 0051 appKey+HMAC 链」）：过滤器 PATH_PREFIX=`/openapi/v1/` 只覆盖该前缀，故订阅开通/关闭置于 `/openapi/v1/subscribe/**` 方可复用验签+限流+全量审计链；@OperateLog 在机器无登录会话时由切面 NULL 身份列落库（BUG-0003 既有行为，审计不丢）。
4. **TDD 红转绿 4 处**：编译期缺符号、测试嵌套类型引用、配置 CGLIB 增强、冒烟 QoS1 假 packet-id，每项红→修→绿与「为什么现在对了」见 §二-1。
5. **并行交错提交**：BASE 后 TASK-0047（calc）提交 `f19edc1` 插队；本任务未 rebase/改写他人历史，范围按提交粒度核对（本提交 22 文件全落 backend openapi）。

---

## 五、遗留问题（字段名定版闭合登记）

1. **字段名定版闭合（本任务核心遗留项，已闭合登记）**：
   - 冻结契约 `unified-message-model.md` v1.1.0「与上游差异」第 7 条（指标推送载荷命名风格 snake_case vs camelCase 待定版）与第 8 条（window_start/window_end 线上类型未明示，待定版），及 `push-payload-evaluation.md` 遗留 1/2——**本任务正式定版：camelCase + 毫秒时间戳**。
   - 落点（不改冻结契约本体）：`ItemChangeEventVO` / `MetricProducedEventVO` javadoc + 字段 + `toMap()` 有序序列化；`OpenApiEventCodec` 4/9 字段逐字校验；契约测试 `ItemChangeEventContractTest` / `MetricProducedEventContractTest` 逐字段锁定（snake_case 不出现、窗口毫秒断言）；live S3b/S4c/S4d 真协议实收载荷逐字一致。
   - 与 0046 生产侧 `MetricProducedEvent` javadoc 登记的「最终风格归 0053 定版」对齐（0046 修订轮已按 camelCase/毫秒发出）。
   - **CONV-03 升版处置**：brief 明定冻结契约文件不改、风格变更走裁决——本定版为履约闭合，若需回写契约正文（第 7/8 条改为已定版）须由协调者对 CONV-03 升 v1.1.1（修订号）并通知各系统，本任务不擅自改冻结文件。
   - `push-payload-evaluation.md` 遗留 3（yoy/mom 为 null 是否随附 compareNote）：本任务定版为「载荷保留 null 字段、不另造 compareNote 字段」（与 0046 生产侧一致，VO 与 codec 如实承载 null，契约测试锁定），同样待 CONV-03 裁决回写。
   - Kafka/MQTT 双通道取舍（遗留 4）：按 brief 决策 2 双通道订阅、Kafka env 默认关，联调时按部署形态取舍。
2. **hash 非幂等键强化登记**（0046 修订轮 F-06 承接）：幂等严格按「指标全键 + 窗口」，`MetricProducedEventVO.idempotencyKey()` 实现 + live S4 实证（值 100→150/hash h1→h2 同窗只落 1 条）；4 项契约测试逐项断言。不得改用 hash（含值随修正重算变化）。
3. **事件仅触发信号、物化表为准**（ADR-0010）：本任务不写 st_metric_value（0045/0046 生产侧已写），缓冲只供触发/读口；中台消费成功语义、积压告警闭环均非一期范围（REQ-0028 非目标）。
4. **dev 库审计残留**：iot_api_call_log 新增订阅动作审计行（正常业务数据，0051/0052 既有链）；test_client 种子沿用 0051/0054 终态，本任务未改其 scope/rate_limit。
5. **TASK 状态**：保持「进行中」待协调者 scoped 复评，未置「已完成」；未改 workitems 库（本报告除外）。

---

## 一句话结论

**通过**——四条验收全部以原始输出为证：① 全量 `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` 2445/0/0/0 BUILD SUCCESS（既有 0051/0054 openapi 测试逐类零回归 + 新增 22 用例，TDD 红转绿 4 处留痕）；② live 本地 MQTT 3.1.1 夹具 broker 真协议全序列（数据项 4 字段 camelCase/毫秒、指标 9 扩展字段逐字、全键+窗口幂等 2 发 1 条且以最新为准、租户 B 载荷跨主题丢弃+WARN、无订阅者不报错、HMAC 开通订阅、全停归零）；③ prod 档真实启动 16.169s + ERROR=0 + 签名 200/篡改 403+6001；④ 提交 `a230f91` 22 文件全落 backend openapi 模块+测试、越界 0。字段名定版 camelCase/毫秒已在 VO+javadoc+测试闭合登记（冻结契约本体未改，CONV-03 升版走裁决），真 EMQX 握手退化与 Kafka 无 broker 按 brief 以夹具+演进位覆盖并登记真机补验条件，TASK 保持「进行中」待评审。

---
---

# 修订轮（2026-11-24）：diff 评审 F-01/I-1 退回修订

- 触发：`TASK-0053-review.md` 判定不通过（2 Important）——F-01（I-1，必修）：事件读口 `GET /openapi/v1/subscribe/{metric,item}/events` 只 `requireIdentity` 却丢弃身份、直接返回**全局**环形缓冲，租户 B 合法 appKey 可读租户 A 事件载荷（真越权）；F-02（I-2）：0029 ingest 发射 `iot:newvalue:{tenant}` 到本任务 MQTT `iot/open/{t}/{i}/item` 主题之间缺桥接生产者——**协调者已裁决拆出 TASK-0084 另行派发，本轮不改代码、只登记边界**。
- 修订 BASE = `ea34d02`（本轮开工 HEAD；`a230f91` 实现提交之后并行 0047 修订轮 `ea34d02` 落库，全 calc/** 未触碰）；**修订提交 = `e9cc251`**（`TASK-0053 修订轮: I-1 事件读口按身份订阅授权过滤(租户B读不到A事件)+4项越权红线测试`；7 文件全落 backend openapi 模块+测试，越界 0）。
- 结论先行：**I-1 已修复，4 项越权红线测试红→绿；全量 2449 测试 0F0E0Skip BUILD SUCCESS（2445 基线零回归 + 4 新增）；I-2 边界已登记（归 0084，本轮不动）。**

## 修订轮一、改动文件清单（提交 `e9cc251`，7 项，全部 openapi 模块）

| 文件 | 修订内容 |
| --- | --- |
| `subscribe/SubscriptionEventStore.java` | 读口隔离核心：新增 `metricEventsFor(allowedTenantCodes)`（按事件 `tenant` 过滤）/ `itemEventsFor(allowedProvenance)`（按 `(tenantCode,instanceCode)` 归属过滤）；`acceptItem` 增重载在摄入时记录主题归属（**不改幂等键**：仍 fullKey+ts+quality / 全键+窗口）；原无参快照 `metricRing()/itemRing()` 保留供内部/契约测试。 |
| `subscribe/SubscriptionRegistry.java` | 新增 `authorizedMetricTenants(appKey)` / `authorizedItemProvenance(appKey)`：从该 appKey **当前有效订阅**派生授权集（租户编码集 / `(tenant,instance)` 归属键集）；无订阅=空集=读不到任何事件（与订阅时 scope 三态同源——注册表只登记已通过 `resolveClientScope` 校验的订阅）。 |
| `subscribe/SubscriptionConsumer.java` | `onItemMessage` 落缓冲时把**已通过摄入侧双校验**的主题 tenant/instance 传给 `acceptItem(event, tenantCode, instanceCode)`——摄入侧双校验逻辑零改动，仅记录归属供读口过滤。 |
| `subscribe/SubscriptionService.java` + `subscribe/SubscriptionServiceImpl.java` | 读口签名改身份感知 `metricEvents(identity)` / `itemEvents(identity)`：以 identity.appKey 取注册表授权集 → 过滤缓冲；无身份/无订阅返回空。 |
| `controller/SubscriptionController.java` | 两个读口不再丢弃 `requireIdentity(request)` 的结果，把身份传入服务层过滤（@Operation 文案同步「按身份订阅授权隔离」）。 |
| `test/.../subscribe/SubscriptionReadIsolationContractTest.java`（新增，4 用例） | 越权红线：① 租户 B 合法 appKey 读不到 A 的指标事件；② 读不到 A 的数据项事件；③ 无订阅权限 appKey 读口空；④ 同租户只订 F01 读不到 F02（scope 实例裁剪）。对旧实现必红。 |

## 修订轮二、TDD 红→绿证据（附「为什么现在对了」）

**RED（编译期）**：先写 `SubscriptionReadIsolationContractTest`（4 用例）→ `mvn test -Dtest=SubscriptionReadIsolationContractTest`：

```
[ERROR] COMPILATION ERROR :
[ERROR] .../SubscriptionReadIsolationContractTest.java:[124,55] 无法将接口 SubscriptionService 中的方法 metricEvents 应用到给定类型
[ERROR] ...[125,55] 同类（metricEvents(identity) 不存在）
[ERROR] ...[137,51]/[138,51] itemEvents(identity) 不存在
[ERROR] ...[149,27]/[150,27]/[168,50] itemEvents/metricEvents 身份重载不存在
[INFO] BUILD FAILURE
```

- 根因（评审 F-01）：旧读口 `subscriptionService.metricEvents()` 无参、返回全局 `store.metricRing()`；身份在控制器被 `requireIdentity(request)` 校验后即丢弃。若仅把签名改为带 identity 但仍返回全局缓冲，`tenantBCannotReadTenantAMetricEvents` 的 `bEvents` 会含 tA 事件 → `assertTrue(bEvents.isEmpty())` 仍红——红线同时锁签名与行为。

**GREEN（实现后）**：

```
[INFO] Running net.otlab.admin.module.openapi.subscribe.SubscriptionReadIsolationContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
[INFO] Tests run: 24, Failures: 0, Errors: 0, Skipped: 0   （聚焦 5 类 subscribe 测试合计）
[INFO] BUILD SUCCESS
```

修复要点与「为什么现在对了」：
1. **授权集以有效订阅为准**（`authorizedMetricTenants` / `authorizedItemProvenance`）——为什么现在对了：注册表只登记已通过 0051 `resolveClientScope` 校验的订阅（scope 三态默认拒绝），从它派生读口授权集与订阅时授权同一来源、零行为分叉；租户 B 未订阅 tA 主题就不可能拿到 tA 的 tenantCode 授权，跨租户读取在授权集生成处即被切断。
2. **数据项事件记录主题归属**（摄入侧仅加记录、双校验不动）——为什么现在对了：数据项订阅消息体契约 4 字段不含 tenant（不能从载荷判租户），而事件到达的主题 `iot/open/{tenant}/{instance}/item` 已由摄入侧校验过 tenant+instance+fullKey 命名空间，把该主题归属随事件记录是最权威、无需改载荷契约的来源；同租户只订 F01 的 appKey 授权集不含 F02 归属键，读口自然裁剪。
3. **幂等键零改动**——为什么现在对了：归属记录走独立 `itemProvenance` Map，不进 `items` 的幂等键（fullKey+ts+quality）与指标幂等键（全键+窗口）；修订不触碰评审明确判定通过的「非 hash 幂等」语义，事件 repr 与 0046 生产侧仍逐字一致。
4. **读口空集 fail-closed**（无身份/无订阅/租户不匹配均返回空 Deque）——为什么现在对了：与 0051 scope 空默认拒绝、REQ-0028「租户 B 收不到租户 A 事件」同向；不给「订阅被拒后仍可读历史缓冲」的旁路。

## 修订轮三、全量验收命令原始输出

```
$ cd backend && mvn install -pl ot-core -DskipTests -q && mvn test -pl ot-admin
[INFO] Running net.otlab.admin.module.openapi.subscribe.SubscriptionReadIsolationContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.admin.module.openapi.subscribe.SubscriptionReadIsolationContractTest
[INFO] Running net.otlab.admin.module.openapi.subscribe.SubscriptionIsolationContractTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.017 s -- in net.otlab.admin.module.openapi.subscribe.SubscriptionIsolationContractTest
[INFO] Running net.otlab.admin.module.openapi.subscribe.EventRingBufferContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.010 s -- in net.otlab.admin.module.openapi.subscribe.EventRingBufferContractTest
[INFO] Running net.otlab.admin.module.openapi.subscribe.ItemChangeEventContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.005 s -- in net.otlab.admin.module.openapi.subscribe.ItemChangeEventContractTest
[INFO] Running net.otlab.admin.module.openapi.subscribe.MetricProducedEventContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.admin.module.openapi.subscribe.MetricProducedEventContractTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 2449, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
EXIT=0
```

- **2445 基线零回归**：2449 = 2445（本轮开工 HEAD `ea34d02` 实测基线，原 22 用例全绿）+ 4 新增红线用例；既有 0051/0054/openapi 逐类计数（HmacAuthStrategyTest 22 / OpenApiQueryServiceImplTest 31 / OpenApiDictSyncMapperSqlTest 3 / OpenApiAuthFilterTest 7 等）与上轮/评审独立复跑一致；本轮改动只在 openapi 读口与测试，未触碰 0051/0054 既有文件。

**范围核对（原始输出）**：

```
BASE（修订）= ea34d02   （并行 0047 修订轮提交，全 calc/**）
修订提交 = e9cc251     （7 文件）
git show --name-only --format="" e9cc251 → 7 项，全部 backend/ot-admin/src/{main,test}/java/.../openapi/**
越界（非 openapi）→ 空
工作区并行在途（本轮未 add/未改）：backend/ot-admin/pom.xml + iot/calcread/**（TASK-0063）、calc/**（TASK-0047 后续）——按提交粒度区分归属
```

## 修订轮四、F-02（I-2）边界登记——协调者已裁决拆出 TASK-0084（本轮不改代码）

- **边界事实**：0029 ingest 发射侧新值事件走 Redis channel `iot:newvalue:{tenant}`（payload 5 键 tenant/pointCode/ts/value/quality，见 TASK-0029-report §1 `event/event.go`）；本任务消费侧订阅 MQTT 主题 `iot/open/{tenant_code}/{instance_code}/item`（统一消息模型 4 字段 fullKey/ts/value/quality）。两主题间**缺一个桥接生产者**（订阅 `iot:newvalue:{tenant}`，把 pointCode 映射为 fullKey、按 target instance 路由后 publish 到 `iot/open/{t}/{i}/item`）。全仓无该桥接实现（评审 F-02 独立核验结论）。
- **归属裁决**：协调者已裁决该桥接归 **TASK-0084（ingest 侧桥接）另行派发**——本任务（TASK-0053）交付的是**消费侧就绪**（订阅注册/scope 校验/MQTT+Kafka 双通道消费/4 字段解析/租户双校验/幂等/读口），不负责生产侧桥接；brief「不自造新事件」「不碰 ingest」口径一致，本轮不改代码、不自造跨工程桥接。
- **live S3 的真实性质登记**：round1 live 冒烟 S3 由裸 MQTT 3.1.1 脚本直接 PUBLISH 注入 `iot/open/default/F01/item`——这是**协议层模拟**（验证消费侧真协议栈收/解/落缓冲的正确性），**非真实 0029 事件链端到端**；本轮如实登记，不宣称数据项订阅已在生产环境具备真实事件来源。
- **端到端补验条件（TASK-0084 交付后）**：ingest 桥接就绪后——ingest 写入新值 → `iot:newvalue:{tenant}` → 桥接 publish `iot/open/{t}/{i}/item` → 本任务消费侧订阅 → GET `/openapi/v1/subscribe/item/events` 断言 4 字段与 ingest 写入一致（fullKey=pointCode 映射口径由 0084 定版）；本任务消费侧代码无需改动即可承接（地址/topic/载荷契约已定版）。

## 修订轮五、遗留与状态

1. I-1 已闭环：读口全局缓冲越权面消除，4 项红线测试锁死（对旧实现必红）；摄入侧双校验与「非 hash 幂等」语义零改动，评审重点 3/4（所列判据）通过结论保持。
2. I-2 边界已登记（§修订轮四），本任务消费侧就绪、端到端生产者归 TASK-0084；不改代码、不自造桥接。
3. TASK 状态保持「进行中」，未置「已完成」；workitems 库仅本报告（修订轮段）变更。

## 修订轮一句话结论

**通过（修订轮）**——2 Important 处置完毕：I-1 真越权漏洞已修复（事件读口按身份有效订阅的 tenant/instance 授权集过滤全局缓冲，租户 B 读不到 A、无订阅读不到、同租户跨实例隔离），4 项越权红线测试编译红→行为绿并附「为什么现在对了」；I-2 按协调者裁决登记为 TASK-0084 边界（消费侧就绪、端到端生产者归 0084、live S3 为协议层模拟已如实登记），本轮不改代码。全量 2449 测试 0F0E0Skip BUILD SUCCESS（2445 基线零回归 + 4 新增），修订提交 `e9cc251` 7 文件全落 openapi 模块越界 0；TASK 保持「进行中」，待协调者 scoped 复评。
