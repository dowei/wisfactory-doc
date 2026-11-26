# 实施报告：微改批 TASK-0079 + TASK-0080

- BASE：`cebf17a`（开工时主仓 HEAD）；提交：`a0a92b7`（TASK-0079）+ `9197e7e`（TASK-0080），均在 main
- 执行日期：2026-11-23；统一测一次（两任务全部改完后同一轮全量测试）；未用 subagent
- 范围核对：`git diff --name-only cebf17a..HEAD` 共 7 文件，全部落 brief「改动范围」声明内（credential/** 与 publish/** 主代码 + ot-admin test/java/**），零越界；workitems 库仅写本报告

## 1. 改动文件清单

| # | 文件（backend/ot-admin/src） | 任务 | 变更 |
| --- | --- | --- | --- |
| 1 | main/java/…/iot/credential/dto/CredentialPullVO.java | 0079 | 扩 `gatewayStatus` 字段（iot_gateway.status 原值；null=网关行缺失状态未知）+ 两参工厂 `of(entity, gatewayStatus)` + javadoc 契约注记 |
| 2 | main/java/…/iot/credential/service/impl/IotCredentialServiceImpl.java | 0079 | pull 时按通道所属网关带出状态（新增 `gatewayMapper` 依赖一处；`gatewayId` 缺失防御分支=null）；不据此改判拉取成败 |
| 3 | test/java/…/iot/credential/service/impl/IotCredentialServiceImplTest.java | 0079 | 红线测试 2 条（禁用网关 gatewayStatus=0 / 启用=1，响应 JSON 字段名锚点同 Go `puller_gatewaystatus_contract_test`）+ gateway 桩 |
| 4 | main/java/…/iot/publish/dto/InternalSnapshotVO.java | 0080 | `snapshot` 字段 `tools.jackson.databind.JsonNode` → `tools.jackson.databind.util.RawValue`（原始存储字节透传，Jackson 3 API）；javadoc 改写口径（BUG-0005） |
| 5 | main/java/…/iot/publish/service/impl/IotPublishServiceImpl.java | 0080 | `retrieveSnapshotInternal` 出站体改 `new RawValue(json)` 原样内嵌；解析保留仅作 schemaVersion 提取 + 存储损坏探测（不参与出站）；信封 version/snapshotHash/snapshotUri/schemaVersion 不动 |
| 6 | test/java/…/iot/publish/service/impl/IotPublishServiceImplInternalQueryTest.java | 0080 | 新增可重算断言 `internalSnapshotOutboundBodyHashRecomputable`（Result 包装 + 运行时同源 Jackson 3 mapper 序列化出站 → 按字段名花括号配对提取原始子串 → SHA-256 == snapshotHash）+ 既有 3 条树断言适配 RawValue API（`snapshotTree` 辅助） |
| 7 | test/java/…/iot/publish/controller/IotPublishInternalControllerTest.java | 0080 | 既有 fixture 构造适配（ObjectNode 包 `RawValue`，wire 契约不变 `$.data.snapshot.isMap()`） |

## 2. 统一验收原始输出

### 2.1 全量构建与测试（brief 验收 1）

```
$ cd backend && mvn install -pl ot-core -DskipTests
[INFO] Installing …\ot-core-1.0.0.jar to C:\Users\dowil\.m2\repository\net\otlab\ot-core\1.0.0\ot-core-1.0.0.jar
[INFO] BUILD SUCCESS
[INFO] Total time:  1.915 s

$ mvn test -pl ot-admin
[INFO] Results:
[INFO]
[INFO] Tests run: 2301, Failures: 0, Errors: 0, Skipped: 0
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] Total time:  58.712 s
```

关键类 surefire 分表（同一轮全量运行）：

```
IotCredentialServiceImplTest          Tests run: 27, Failures: 0, Errors: 0, Skipped: 0
IotPublishServiceImplInternalQueryTest Tests run: 18, Failures: 0, Errors: 0, Skipped: 0
IotPublishInternalControllerTest       Tests run: 15, Failures: 0, Errors: 0, Skipped: 0
```

### 2.2 TASK-0079 红线测试：先红后绿（brief 验收 3）

**红**（旧代码——无 gatewayStatus 字段/无网关填充；运行时断言失败，非编译失败）：

```
$ mvn test -pl ot-admin -Dtest='IotCredentialServiceImplTest#testPull_DisabledGatewayCarriesGatewayStatus+testPull_EnabledGatewayCarriesGatewayStatus'
[ERROR] Tests run: 2, Failures: 1, Errors: 1, Skipped: 0
[ERROR] …IotCredentialServiceImplTest.testPull_DisabledGatewayCarriesGatewayStatus:523
  pull 响应必须携带 gatewayStatus 字段（TASK-0079 契约——通道启用但网关禁用时 Go 侧据此 deny）
  ==> expected: not <null>
[ERROR] …IotCredentialServiceImplTest.testPull_EnabledGatewayCarriesGatewayStatus:538
  NullPointer Cannot invoke "tools.jackson.databind.JsonNode.asInt()" … JsonNode.get(String)" is null
[INFO] BUILD FAILURE
```

**绿**（实现落地后，全类含既有凭据测试）：

```
$ mvn test -pl ot-admin -Dtest='IotCredentialServiceImplTest'
[INFO] Tests run: 27, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

### 2.3 TASK-0080 可重算断言：先红后绿（brief 验收 2 / BUG 修复类加严 §13-2）

**红**（旧 JsonNode 代码——出站体经解析再序列化，复现 BUG-0005 数字形态改写 5.000000→5.0，重算哈希必失配）：

```
$ mvn test -pl ot-admin -Dtest='IotPublishServiceImplInternalQueryTest#internalSnapshotOutboundBodyHashRecomputable'
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
org.opentest4j.AssertionFailedError: 出站快照体必须与其 snapshotHash 互相验证（拉快照比对哈希兜底——REQ-0013/REQ-0033 验收 2）
  ; raw snapshot={"schemaVersion":"1.0.1",…,"points":[{"id":12020,"pointCode":"P1","minValue":5.0}],…}
  ==> expected: <36a0995bdf3825c351fb926384fc896b5870f9e5dadfb4f651df3efe123cba82>
      but was: <f5257b0452b9412c24960362c0185e6474c79ef0ddc5920f7907174eba47997f>
[INFO] BUILD FAILURE
```

**绿**（RawValue 原始字节透传后，同断言转绿；0026 既有 snapshot/changes 测试同轮回归）：

```
$ mvn test -pl ot-admin -Dtest='IotPublishServiceImplInternalQueryTest,IotPublishInternalControllerTest'
[INFO] Tests run: 15,  Failures: 0, Errors: 0, Skipped: 0 -- IotPublishInternalControllerTest
[INFO] Tests run: 18,  Failures: 0, Errors: 0, Skipped: 0 -- IotPublishServiceImplInternalQueryTest
[INFO] BUILD SUCCESS
```

（日志中的 ERROR 行为既有负路径夹具的预期告警输出——篡改探测/凭据终检/哈希核对案例。）

### 2.4 全仓同契约点扫描证据（BUG 修复类加严 §13-1）

扫描命令与命中（主代码，iot 域 + admin 全域核对）：

```
$ grep -rn "private.*JsonNode" ot-admin/src/main/java/net/otlab/admin/
→ VO 字段级命中仅 3 处：
  publish/dto/InternalSnapshotVO.java:47  snapshot   —— 本修复点（0080）
  channel/dto/IotChannelCreateForm.java:53 connConfig —— 入站表单（请求侧），无哈希契约，不适用
  channel/dto/IotChannelUpdateForm.java:46 connConfig —— 同上（入站）
  （其余命中均为私有方法参数/局部变量，非出站 VO 字段）

$ grep -rln "readTree" ot-admin/src/main/java/net/otlab/admin/module/iot/
→ 6 文件逐一归类：
  publish/service/impl/IotPublishServiceImpl.java —— 出站快照解析（唯一再序列化丢字节点=0080 修复；
      其 BODY_MAPPER/DIFF_MAPPER 两条 readTree 为回滚 id 提取与增量 diff，纯服务端计算不回写哈希、不出站原文）
  publish/snapshot/SnapshotCredentialGuard.java   —— 凭据终检（服务端检测，不出站）
  channel/schema/ChannelConnConfigValidator.java  —— 入站 connConfig 校验
  point/service/support/PointCurveQueryClient.java、tdengine/HttpTdengineAdmin.java —— TDengine 查询客户端（非快照链路）
  template/service/impl/IotTemplateVersionServiceImpl.java —— 模板结构化 diff（确认门，无哈希重算诉求）

管理面 /iot/publish/snapshot（brief 点名核对项）：
  retrieveSnapshot → SnapshotVO.snapshotBody 为 String 原文口径（代码级确认），不经 JsonNode
  再序列化，原始字节在 JSON 字符串转义下无损——不属 BUG-0005 同类缺陷，不改（未做项 3）。
/internal/v1/config/changes（brief 点名核对项）：
  changesSince 出站为五段结构化摘要（added/removed/changed，type+key 字符串），无快照原文
  出站、无哈希重算契约——不适用。
```

**结论：同契约点全仓唯一命中即本修复点，无待裁决项，无需扩散修复。**

### 2.5 live curl 复核（brief 验收 2 后半）

起停纪律：8098 起前探测空闲（netstat 无 LISTENING）→ `java -jar ot-admin/target/ot-admin-1.0.0.jar`（03:14 以 `mvn package -pl ot-admin -DskipTests` 重建、含本批修复；cwd=backend/ot-admin 使 `./data/snapshots` 落既有存储根）+ `OT_GATEWAY_INTERNAL_TOKEN=tok-0080-live`（仅进程环境变量，未落任何配置文件）→ 启动日志 `Started OtAdminApplication in 11.573 seconds` → 用毕即停，停后 netstat 8098 零 LISTENING（仅 TIME_WAIT 客户端残留）。WSL wisfactory-mysql/redis 全程在跑；Go 道在途任务零接触（未碰 ingest/**）。

快照拉取（TC-0026 步骤 2/3，iot_default scope=1:9111 最新版 v5——存量已发布数据，零数据变更）：

```
$ curl -s -H "X-Internal-Token: tok-0080-live" \
  "http://127.0.0.1:8098/internal/v1/config/snapshot?scopeType=1&scopeId=9111&tenant=default" → HTTP 200
响应信封：version=5, snapshotHash=415c24f63d45fd7b410949d72156854a0012a0a14172da5e85185200738da473,
          snapshotUri=1/1-9111-5.json, schemaVersion=1.0.1, snapshot={…嵌套对象…}

对响应内 "snapshot" 字段原文子串（花括号配对提取，不经二次解析）重算 SHA-256：
SHA256(raw)  : 415c24f63d45fd7b410949d72156854a0012a0a14172da5e85185200738da473
resp.hash    : 415c24f63d45fd7b410949d72156854a0012a0a14172da5e85185200738da473
MATCH        : True
出站体数字形态：响应含 "minValue":5.000000（原样，未被改写为 5.0）
存储文件旁证：sha256sum backend/ot-admin/data/snapshots/1/1-9111-5.json
  = 415c24f6…d473（出站体 = 响应 hash = 存储文件 三方一致）
```

凭据拉取（TASK-0079 live 侧证，密文脱敏）：

```
$ curl -s -H "X-Internal-Token: tok-0080-live" \
  "http://127.0.0.1:8098/internal/v1/credential/pull?channelId=21&credType=MQTT_PASSWORD" → HTTP 200
{"code":200,"data":{"channelId":21,"ciphertext":"{v1}…（脱敏）","credType":"MQTT_PASSWORD",
 "gatewayStatus":1,"keyVersion":"1"},"message":"操作成功","timestamp":…}
```

—— 网关 9111（status=1）原值带出；JSON 字段名与 TASK-0027 Go 契约（`GatewayStatus *int`，0 禁用/1 启用/缺失未知）逐字对齐。

## 3. 未做项与原因

1. **0079 禁用网关的 live 案例**：需把共享库 gateway 9111 置 0——该行为 Go 道（TASK-0028 在途）authmqtt 链路所用的共享状态，本批不改 SQL/数据；禁用场景红线由单测先红后绿承载（断言锚点=响应 JSON 字段名，与 Go `puller_gatewaystatus_contract_test` 同锚）。live 仅证启用网关 gatewayStatus=1 原值带出 + 字段名契约。
2. **TASK/BUG/TC 状态流转**：契约明文——执行者不置「已完成」、不改 workitems 档案；BUG-0005 验证结果与 TC-0026 回归登记归协调者收尾。
3. **管理面 /iot/publish/snapshot 不改**：扫描证实为 String 原文口径（§2.4），无原始字节丢失，非 BUG-0005 同类。

## 4. 偏离计划之处

1. **0079 红阶段测试形态**：红线测试红阶段版本未含 `vo.getGatewayStatus()` getter 断言与 `gatewayMapper` 桩接线（旧代码中字段/依赖不存在——否则会以编译失败或反射字段缺失异常充当红，证据不纯）；绿阶段补齐并随之提交。红→绿两次运行的差异仅此。
2. **live 用 boot jar 重建**：brief 只要求「backend dev 起」，未禁 mvn（0027 的禁 mvn 是其任务特定约束）；旧 jar（01:15 构建）不含修复，live 前以 `mvn package -pl ot-admin -DskipTests` 重建（03:14），零源码写入。
3. **live scope 取 1:9111 v5**（存量已发布数据，与 TASK-0027 冒烟同 scope 同版本）——省去发布造数，零数据变更。
4. **出站体哈希取证方式**：单测与 live 均以「字段名 + 花括号配对（字符串/转义感知）」提取原始子串后重算——不经 JSON 二次解析（那会再次改写数字形态使断言失真）。

## 5. 遗留问题

1. `InternalSnapshotVO.snapshot` 类型为 Jackson 3 `RawValue`，springdoc/OpenAPI 文档对其类型展示可能退化为 string——内部服务间端点、无 UI 消费面，本批不处理。
2. 快照体出站口径并存：管理面 String 原文（SnapshotVO.snapshotBody）/ 内部端点嵌套对象原文透传（RawValue）——两者均保原始字节，行为契约无变化，仅登记。
3. Go 侧（TASK-0027 交付的「对账锚点口径」）在 gatewayStatus + 哈希可重算两项交付后，可升级为原始完整性验证（重算哈希兜底恢复）——归 Go 道接线，本批不涉。
4. 0079 禁用网关 live 案例未做（见未做项 1）；如需活体验证可在 Go 道空闲窗口置 gateway status=0 后复跑 curl。

## 结论

**通过**——0079/0080 共 7 文件改动全部落 brief 声明范围；统一全量测试 2301/0/0 BUILD SUCCESS；0080 可重算断言先红（出站体 5.000000→5.0 重算哈希失配，即 BUG-0005 复现）后绿（RawValue 原始字节透传），live curl 对出站体重算 SHA-256 == snapshotHash == 存储文件哈希三方一致且 8098 用毕即停；0079 红线测试先红（pull 响应缺 gatewayStatus）后绿（禁用网关 gatewayStatus=0、启用=1 原值带出，字段名与 Go 侧契约逐字对齐），既有凭据/发布/内部端点测试全数回归绿。
