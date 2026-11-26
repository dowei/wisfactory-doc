# EMQX 真联调验证报告（TD-0056 backend 真发 / TD-0057 ingest 真收 / calc 订阅缝观察）

> 批次：纯验证（零业务源码改动；只 env/配置 + 临时脚本）
> BASE HEAD：`09357d1`（2026-11-23 10:11:20 +0800，TASK-0065）
> 执行日期：2026-11-23 22:47–23:01（Git Bash / Windows）
> 唯一输入：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\EMQX-INTEGRATION-brief.md`

---

## 一、改动文件清单（应为零或仅配置）

| 项 | 状态 |
| --- | --- |
| 业务源码（backend/ingest/calc/frontend） | **零改动**（`git status --short` 仅 `?? .agents/`，为开工前既有未跟踪目录） |
| 配置文件 | **零改动**——`application.yml` / `application-dev.yml` 未编辑；broker URL 与开关全部 env 注入 |
| 临时脚本 | 全部在仓库外 `D:\tmp\emqx-verify\`（rawmqtt.py / httpclient.py / item_a.py / item_b.py / item_c_pub.py / item_c_event.py / redis_sub.py / ingestd-real.exe 等），**不入库** |
| 构建产物 | `ingest/cmd/ingestd` 从**未改源码** `go build -o D:\tmp\emqx-verify\ingestd-real.exe ./cmd/ingestd`（仓库内 `ingest/ingestd.exe` 为陈旧 skeleton，仅 /healthz，故必须重编到 temp；零源码改动） |
| backend jar | 直接 `java -jar backend/ot-admin/target/ot-admin-1.0.0.jar`（mtime 2026-11-23 10:09，含 notify 类；**未跑 mvn**，避开 0081 在途 agent） |

**执行命令与原始输出**
```
$ git rev-parse --short HEAD && git status --short && git branch --show-current
09357d1
?? .agents/
main

$ python -c "import zipfile; ..." # jar 内 notify 类
BOOT-INF/classes/net/otlab/admin/module/iot/publish/notify/MqttNotifySender.class
BOOT-INF/classes/net/otlab/admin/module/iot/publish/notify/MqttNotifySender$NotifyPayload.class
BOOT-INF/classes/net/otlab/admin/module/iot/publish/notify/PahoMqttPublisher.class
BOOT-INF/classes/net/otlab/admin/module/iot/publish/notify/NotifySender.class
BOOT-INF/classes/net/otlab/admin/module/iot/publish/notify/NoOpNotifySender.class
...
```

---

## 二、每条验证命令原始输出

### 启动前端口探测（原始输出）
```
$ netstat -ano | grep LISTENING | grep -E ':(8098|8090|8095|8099|1883|18083|3307|6379|6041)\b'
  TCP    0.0.0.0:3307    0.0.0.0:0    LISTENING    65616
  TCP    0.0.0.0:8090    0.0.0.0:0    LISTENING    91192   # nord-scrm backend（无关项目，未触碰）
  TCP    127.0.0.1:6379  0.0.0.0:0    LISTENING    5508
```
- 8098/8095/8099 探测空闲；8090 被**其他项目** `D:\workspace\nord-scrm\backend\...nord-scrm-1.0.0.jar`（PID 91192）占用 → C 项用 `INGEST_HTTP_ADDR=:18090` 绕开（18090 亦先探测为空闲）。未停/动 8090 占用进程。

### A. 连通性与 broker 基本行为（复审）——PASS
```
$ python -I D:/tmp/emqx-verify/item_a.py
=== A) EMQX 裸 MQTT 复审 ===
CONNECT -> 00 00 (CONNACK return code=0 accepted)
PUB -> SUB received topic=iot/verify/a_roundtrip qos=0 payload={"probe": "A-roundtrip", "ts": 1795445291852}
PING -> PINGRESP ok
A RESULT: PASS — CONNECT accepted / SUBACK QoS1 / PUB-SUB 往返收到 / PINGRESP
```
脚本用纯 stdlib 手写 MQTT 3.1.1 报文（paho 未安装），CONNECT(clean session) → CONNACK 00 00 → SUBSCRIBE `iot/verify/a_roundtrip` QoS1 → SUBACK granted=01 → QoS0 PUB → 对方 SUB 收到 → PING/PINGRESP。与协调者 2026-11-23 实测一致。

### B. backend → EMQX 真发（TD-0056）——PASS

**启动（env 注入，零源码改动）**
```
$ cd backend/ot-admin && OT_PUBLISH_NOTIFY_ENABLED=true \
    OT_PUBLISH_NOTIFY_BROKER_URL=tcp://192.168.31.145:1883 \
    OT_GATEWAY_INTERNAL_TOKEN=verify-internal-token-20261123 \
    java -Dfile.encoding=UTF-8 -jar target/ot-admin-1.0.0.jar --spring.profiles.active=dev
... Tomcat started on port 8098 (http) with context path '/'
... Started OtAdminApplication in 146.858 seconds (process running for 149.701)
就绪判据：GET /auth/captchaImage -> HTTP 200（首轮 ~130s 冷启；二轮 ~35s）
```
（首轮仅 notify 两 env；二轮补 `OT_GATEWAY_INTERNAL_TOKEN` 供 C 项快照拉取；两轮都只用 env。）

**真调 publish + 外部订阅断言（原始输出）**
```
$ python -I D:/tmp/emqx-verify/item_b.py
=== B) backend -> EMQX 真发验证 (TD-0056) ===
B1 login OK token=f095cbce-f59...
B2 baseline version: scope2=6 scope1(gw9111)=5
B3 subscriber CONNACK=0000
B3 subscribed: iot/config.changed, iot/+/+/config
B4 publish scope_type=2 scopeId=0 ...
   POST /iot/publish/publish -> HTTP 200 code=200
   version row: id=42 scope=2-0 version=7 hash=ed595758c6fcf5b2a6ab211346362ab45b0811e04534eee1f536b16b8b285451
B5 publish scope_type=1 scopeId=9111 ...
   POST /iot/publish/publish -> HTTP 200 code=200
   version row: id=43 scope=1-9111 version=6 hash=f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae
B6 collecting MQTT notifications (8s)...
   received 3 message(s):
   topic=iot/config.changed qos=1 payload={"tenant":"default","scopeType":2,"scopeId":0,"version":7,"hash":"ed595758c6fcf5b2a6ab211346362ab45b0811e04534eee1f536b16b8b285451"}
   topic=iot/default/SMOKE0026_GW/config qos=1 payload={"tenant":"default","scopeType":1,"scopeId":9111,"version":6,"hash":"f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae"}
   topic=iot/config.changed qos=1 payload={"tenant":"default","scopeType":1,"scopeId":9111,"version":6,"hash":"f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae"}
B7 ASSERT five-key order=['tenant', 'scopeType', 'scopeId', 'version', 'hash']
   shared scope2 msg={'tenant': 'default', 'scopeType': 2, 'scopeId': 0, 'version': 7, 'hash': 'ed595758c6fcf5b2a6ab211346362ab45b0811e04534eee1f536b16b8b285451'}
   gateway topic msg={'tenant': 'default', 'scopeType': 1, 'scopeId': 9111, 'version': 6, 'hash': 'f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae'}
   shared scope1 msg={'tenant': 'default', 'scopeType': 1, 'scopeId': 9111, 'version': 6, 'hash': 'f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae'}
B RESULT: PASS — 五键逐字 + hash 与版本行一致 + 网关专属主题收到（QoS1 由 broker SUBACK/PUBACK 流程见证）
B8 record version rows for report:
   scope2 versionId=42 version=7 hash=ed595758c6fcf5b2a6ab211346362ab45b0811e04534eee1f536b16b8b285451
   scope1 versionId=43 version=6 hash=f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae
```

**服务端日志（[PUBLISH-NOTIFY] afterCommit + 真连接 + 三条 sent）**
```
2026-11-23 22:52:33.951 [http-nio-8098-exec-7] INFO  n.o.a.m.i.p.notify.MqttNotifySender - [PUBLISH-NOTIFY] config.changed deferred to afterCommit topic=iot/config.changed versionId=42 version=7 target=3-calc
2026-11-23 22:52:34.386 [http-nio-8098-exec-7] INFO  n.o.a.m.i.p.notify.PahoMqttPublisher - [PUBLISH-NOTIFY] mqtt publisher connected broker=tcp://192.168.31.145:1883 clientId=ot-admin-notify-91799bf7
2026-11-23 22:52:34.396 [http-nio-8098-exec-7] INFO  n.o.a.m.i.p.notify.MqttNotifySender - [PUBLISH-NOTIFY] config.changed sent topic=iot/config.changed versionId=42 version=7 target=3-calc hash=ed595758...
2026-11-23 22:52:34.756 [http-nio-8098-exec-8] INFO  n.o.a.m.i.p.notify.MqttNotifySender - [PUBLISH-NOTIFY] config.changed deferred to afterCommit topic=iot/default/SMOKE0026_GW/config versionId=43 version=6 target=1-SMOKE0026_GW
2026-11-23 22:52:34.776 [http-nio-8098-exec-8] INFO  n.o.a.m.i.p.notify.MqttNotifySender - [PUBLISH-NOTIFY] config.changed deferred to afterCommit topic=iot/config.changed versionId=43 version=6 target=2-ingest
2026-11-23 22:52:34.791 [http-nio-8098-exec-8] INFO  n.o.a.m.i.p.notify.MqttNotifySender - [PUBLISH-NOTIFY] config.changed sent topic=iot/default/SMOKE0026_GW/config versionId=43 version=6 target=1-SMOKE0026_GW hash=f0655b3f...
2026-11-23 22:52:34.797 [http-nio-8098-exec-8] INFO  n.o.a.m.i.p.notify.MqttNotifySender - [PUBLISH-NOTIFY] config.changed sent topic=iot/config.changed versionId=43 version=6 target=2-ingest hash=f0655b3f...
```

**版本行与下发记录（DB 原始输出）**
```
$ backend/scripts/sql-exec.sh "SELECT ... FROM iot_config_version WHERE id IN (42,43); ..."
id  tenant_id  scope_type  scope_id  version  snapshot_hash                                                      snapshot_uri     publish_status
42  1          2           0         7         ed595758c6fcf5b2a6ab211346362ab45b0811e04534eee1f536b16b8b285451  1/2-0-7.json     1
43  1          1           9111      6         f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae  1/1-9111-6.json  1

id  version_id  target_type  target_id       ack_status
23  42          3            calc            0
24  43          1            SMOKE0026_GW   0
25  43          2            ingest          0
DONE
```
> **验证产生的版本行（id=42/43）与 publish_log 行（id=23/24/25）登记如上，未清理**。既有 21 行版本数据未动（验证前 21 行，验证后 23 行）。

**fail-soft 旁证**：按 brief「本轮真机只做正向」，未制造 broker 断连；既有单测 `MqttNotifySenderTest` 已覆盖（TASK-0026 报告）。

### C. ingest ← EMQX 真收（TD-0057）——PASS

**启动（env 注入 + temp 构建二进制）**
```
$ cd ingest && INGEST_HTTP_ADDR=:18090 INGEST_MQTT_BROKER=tcp://192.168.31.145:1883 \
  INGEST_MQTT_CLIENT_ID=verify-ingest-c INGEST_REDIS_ADDR=127.0.0.1:6379 INGEST_REDIS_DB=0 \
  INGEST_TDENGINE_BASE_URL=http://192.168.31.145:6041 INGEST_TDENGINE_USER=root \
  INGEST_TDENGINE_PASSWORD=taosdata INGEST_BACKEND_BASE_URL=http://127.0.0.1:8098 \
  INGEST_INTERNAL_TOKEN=verify-internal-token-20261123 INGEST_SNAPSHOT_SCOPE=1:9111 \
  INGEST_SNAPSHOT_TENANT=default INGEST_SNAPSHOT_INTERVAL=2s \
  INGEST_DEADLETTER_WARN_EVERY=1 INGEST_GATEWAY_MAP='SMOKE0026_GW:21:default' \
  D:/tmp/emqx-verify/ingestd-real.exe
2026/11/23 22:57:31 ingestd: TDengine REST 执行器已装配 endpoint=http://192.168.31.145:6041 user=root
2026/11/23 22:57:31 ingestd: Redis 点位缓存已装配 addr=127.0.0.1:6379 db=0（键 iot:pt:{tenant}:{code}，写穿透/失败不阻塞落库/未命中回源 TDengine 最近值）+ 新值事件发射侧
2026/11/23 22:57:31 ingestd: MQTT 消费源已装配 broker=tcp://192.168.31.145:1883
2026/11/23 22:57:31 ingestd: 快照对账已装配 scope=1:9111 tenant="default" interval=2s
2026/11/23 22:57:31 ingestd: HTTP 接入面已启动 [::]:18090（report/auth-mqtt/healthz）
2026/11/23 22:57:31 snapshot: 对账快照已替换 version=6 hash=f0655b3f7de8c9bdeec31eb7db2ea0cf98cf1b6889ca3991d144333c5bfc34ae
$ curl http://127.0.0.1:18090/healthz  -> HTTP 200
```
> 快照版本 6 是 B 项刚发布的计算/网关版本（网关 scope 1:9111 version=6）；DBName() 派生 `iot_default`（未手拼），子表 `SMOKE0026_5fP1`（`escapeIdentifier` 把 `_` 转 `_5f`）。

**真 pub §3.2 payload（外部 raw MQTT 客户端，QoS1）**
```
$ python -I D:/tmp/emqx-verify/item_c_pub.py
=== C) ingest <- EMQX 真收验证 (TD-0057) ===
C1 publisher CONNACK ok
C2 pub topic=iot/default/SMOKE0026_GW/data payload={"tenant": "default", "gateway": "SMOKE0026_GW", "batchId": "VERIFY-C-1795445875066", "sentAt": 1795445875066, "points": [{"pointCode": "SMOKE0026_P1", "ts": 1795445875066, "value": 42.5, "quality": 0}]}
   PUBLISH QoS1 sent
C3 pub unknown-code topic=iot/default/SMOKE0026_GW/data payload={"tenant": "default", "gateway": "SMOKE0026_GW", "batchId": "VERIFY-C-UNKNOWN-1795445875067", "sentAt": 1795445875067, "points": [{"pointCode": "NO_SUCH_POINT_XYZ", "ts": 1795445875067, "value": 1.0, "quality": 0}]}
   PUBLISH QoS1 sent
C4 publisher closed
   expected child table = SMOKE0026_5fP1 (escapeIdentifier maps _ -> _5f)
   expected ts = 1795445875066, value = 42.5, quality = 0
```

**断言 1：TDengine 真写（原始响应）**
```
$ curl -u root:taosdata -X POST http://192.168.31.145:6041/rest/sql --data-binary 'SELECT ts,value,quality FROM iot_default.SMOKE0026_5fP1 ORDER BY ts DESC LIMIT 5'
{"code":0,"column_meta":[["ts","TIMESTAMP",8],["value","DOUBLE",8],["quality","INT",4]],"data":[["2026-11-23T14:57:55.066Z",42.5,0]],"rows":1}
# 后续多次发布累积到 5 行：42.5 / 55.5 / 66.6 / 77.7 / 77.7（全部 quality=0）
```

**断言 2：Redis 点位缓存（原始输出）**
```
$ redis-cli -h 127.0.0.1 -p 6379 GET 'iot:pt:default:SMOKE0026_P1'
{"v":42.5,"q":0,"ts":1795445875066}
# 末次 GET: {"v":77.7,"q":0,"ts":1795445996030}
```

**断言 3：新值事件 Redis Pub/Sub（原始输出）**
```
$ python -I D:/tmp/emqx-verify/item_c_event.py
=== C event capture: Redis SUB iot:newvalue:default + MQTT pub ===
SUB ACK: ['subscribe', 'iot:newvalue:default', 1]
MQTT PUB: {"tenant": "default", "gateway": "SMOKE0026_GW", "batchId": "VERIFY-C-4-1795445996030", "sentAt": 1795445996030, "points": [{"pointCode": "SMOKE0026_P1", "ts": 1795445996030, "value": 77.7, "quality": 0}]}
RECV Redis Pub/Sub channel=iot:newvalue:default payload={"tenant":"default","pointCode":"SMOKE0026_P1","ts":1795445996030,"value":77.7,"quality":0}
C-EVENT RESULT: PASS — 五键逐字 tenant/pointCode/ts/value/quality 匹配
```

**断言 4：未知编码死信（日志原始输出）**
```
$ grep 'dead-letter' ingest-c.log
2026/11/23 22:57:56 pipeline: [dead-letter] tenant=default reason=unknown_code count=1 pointCode=NO_SUCH_POINT_XYZ batchId=VERIFY-C-UNKNOWN-1795445875067
```
未知编码批**未落库**（TDengine 查询只有 SMOKE0026_P1 子表行，无 NO_SUCH_POINT_XYZ 子表）；ingest 日志 error/panic 计数 = 0。

**scratch 数据清理（原始输出）**
```
$ redis-cli DEL 'iot:pt:default:SMOKE0026_P1'          -> 1
$ curl ... --data-binary "DELETE FROM iot_default.SMOKE0026_5fP1 WHERE ts >= '2026-11-23T14:57:00.000Z' AND ts <= '2026-11-23T15:00:00.000Z'"
{"code":0,...,"data":[[5]],"rows":1}                    # 删除 5 行验证数据
$ SELECT COUNT(*) FROM iot_default.SMOKE0026_5fP1       -> 0 行
```

### D. calc 订阅缝观察（仅观察，不验消费）——符合预期
```
$ grep -rIn 'ConfigChangedSubscriber' calc/src
calc/src/main/java/net/otlab/calc/runtime/ConfigChangedSubscriber.java:15: public interface ConfigChangedSubscriber {
calc/src/main/java/net/otlab/calc/CalcEngineApplication.java:95: public ConfigChangedSubscriber configChangedSubscriber() {
...  // NoOp 占位：本期拉取驱动，通知不消费
$ grep -n 'reconcile' calc/src/main/resources/application.yml
20:  reconcile-seconds: ${CALC_SNAPSHOT_RECONCILE_SECONDS:300}

# B 项外部订阅实收的 config.changed 真消息（格式观察对象，QoS1）：
topic=iot/config.changed qos=1 payload={"tenant":"default","scopeType":2,"scopeId":0,"version":7,"hash":"ed595758c6fcf5b2a6ab211346362ab45b0811e04534eee1f536b16b8b285451"}
```
结论：backend 发出的 `config.changed` 真消息格式正确（§3.3 五键逐字、hash 与版本行一致）；calc 侧当前仅有 NoOp 接口占位 + 300s 定时拉取，本轮**不验消费**，占位说明保留 TD 登记。

### 全停端口归零（原始输出）
```
$ taskkill //PID 82744 //F   # backend dev
$ taskkill //PID 84772 //F   # ingestd
$ taskkill //PID 91364 //F   # redis-cli subscriber
$ netstat -ano | grep -E ':(8098|18090) ' | grep LISTENING
(8098/18090 free)
# 预存进程未动：8090=91192(nord-scrm)、8091=91140(nord-scrm vite)、3307/6379=phpStudy
```

---

## 三、未验证项与原因

| 项 | 原因 |
| --- | --- |
| fail-soft（broker 不可达仍 200 / 自动重连） | brief 明确「本轮真机只做正向」；既有 `MqttNotifySenderTest` 单测已覆盖 |
| `retain=false` 晚接入不收旧信号 | 同上，非本轮必验项；实现走 `publish(topic, json, QoS1, false)`（源码确认） |
| calc 对 config.changed 的真消费 / 触发 reconcile | TASK-0040 遗留订阅缝占位（NoOp），本轮按 brief D 只观察格式，不验消费；消费实现到位后另验 |
| ingest HTTP 上报口（/ingest/v1/report）真收 | 本轮按 brief C 走 MQTT 真收链；HTTP 同构路径由既有契约测试覆盖 |
| 多实例/跨租户/并发场景 | 纯验证批次单实例正向，不在 A–D 范围 |
| 8090 默认 ingest 端口占用下的真连 | 端口被无关 `nord-scrm` 占用，改用 env `INGEST_HTTP_ADDR=:18090`（零源码改动）；不属实现缺陷 |

---

## 四、偏离计划之处

1. **ingest 端口**：brief/默认 8090 被无关项目占用 → 用 `INGEST_HTTP_ADDR=:18090`（先探测空闲，用毕即停）。属环境规避，非实现问题。
2. **.backend 启动两轮**：首轮仅 notify 两 env；为支撑 C 项快照拉取（`/internal/v1/config/snapshot` 需 `X-Internal-Token`），二轮追加 `OT_GATEWAY_INTERNAL_TOKEN=verify-internal-token-20261123` env 重启。两轮都未编辑配置文件。
3. **ingestd 二进制**：仓库内 `ingest/ingestd.exe` 为陈旧 skeleton（仅 /healthz，无流水线装配），故从**未改源码** `go build` 到 temp 目录运行；构建产物不入库。
4. **C 项发布次数**：为排除 Windows redis-cli 重定向缓冲假象、用 Python RESP 订阅进程内直连取证，共发 4 条合法批次（42.5/55.5/66.6/77.7），全部真落库；证据取末次 77.7。scratch 数据（5 行 + Redis 键）已按 brief 清理。
5. **版本/publish_log 行保留**：B 项产生 version id=42/43 与 publish_log id=23/24/25 按 brief「登记进报告、不清理既有数据」保留（既有 21 行未动）。

---

## 五、遗留问题

1. **TD-0056 可清账**：backend `ot.publish.notify.enabled=true` + env broker 下，真 broker 收到 3 条 QoS1 通知（计算 scope `iot/config.changed`、网关 scope 专属主题 + 共享主题各一），五键逐字、hash 与版本行一致；afterCommit 延迟发送、懒建立连接、句柄复用在真机日志可见。**建议 TD-0056 清账**（fail-soft 旁证由既有单测覆盖，brief 允许本轮不测）。
2. **TD-0057 可清账**：ingest PahoPort 真连 EMQX、订阅 `iot/+/+/data`、收到 §3.2 合法 payload，全链断言通过：TDengine `iot_default.SMOKE0026_5fP1` 真写（value/quality 正确）、Redis `iot:pt:default:SMOKE0026_P1` 写穿透、新值事件 `iot:newvalue:default` 五键逐字、未知编码 `reason=unknown_code` 死信且不落库。**建议 TD-0057 清账**（calc 订阅缝属 TASK-0040 遗留，保持登记不随本条清）。
3. **calc 订阅缝延续登记**：backend 真消息格式已确认正确，calc 侧 NoOp 占位未消费——该缝实现到位后，用本报告的同款 `config.changed` 真消息验证触发 reconcile；不阻塞 TD-0056/0057 清账。
4. **观察项（非本轮缺陷）**：backend 冷启 ~147s（首轮）/ ~35s（二轮），远慢于历史 ~10-20s；疑与宿主 phpStudy MySQL + 并行 0081 maven 负载相关，不影响功能，未定位/未修。
5. **环境观察**：`8090/8091` 被 `nord-scrm` 项目占用；若后续本仓 ingestd 需默认端口，需协调释放或继续用 env 覆盖。

---

**结论一句话：TD-0056/0057 两项均可清账（A/B/C 全 PASS，D 格式观察符合预期）；calc 订阅缝按 TASK-0040 遗留继续登记，不阻塞本次清账。**
