# TASK-0084 报告：新值事件到 MQTT 数据项订阅主题桥接生产者

- 执行者：TASK-0084 执行者（no-subagents；ingest 道——只改 `ingest/**`）
- **BASE（开工 HEAD）= `fb8f66d`**（`git rev-parse --short HEAD`，2026-11-24 main，brief「BASE=当前 HEAD」口径）
- **交付提交 = `2fc2a4d`**（唯一提交，注明 TASK-0084；8 文件全部 `ingest/**`，`git show --name-only --format="" HEAD | grep -v '^ingest/' | wc -l` = 0）
- 并行事实：本窗口内 backend 道 0063 评审在途（工作区 `CalcReadServiceImpl.java` / `20261124_iot_calc_schema.sql` / 两个 backend 测试文件改动）——本任务全程未触碰、未 add、未编译；`git diff --name-only fb8f66d..HEAD | grep -v '^ingest/'` = 空（本提交越界 0）
- 执行日期：2026-11-24
- 环境：Go 1.27.2 windows/amd64；Redis 宿主 phpStudy 3.0.504@127.0.0.1:6379（redis-cli `/d/phpstudy_pro/Extensions/redis3.0.504/redis-cli.exe`，PING=PONG）；EMQX 1883 握手退化（brief 既定）——live 用内嵌 MQTT 3.1.1 真协议夹具 broker（每轮随机端口）+ 真 Paho v1.5.1 客户端 + 真 Redis，broker env 可切；TDengine 本批不涉；未碰 wsl/docker
- 结论先行：**三条验收全部通过**——① `go build ./... && go vet ./... && go test ./...` 全绿（12 包 ok、EXIT=0；契约 223 PASS/0 FAIL，基线 211 + 新增 12，零回归）；② live 桥接全序列 7 步真协议实证（opt-in 默认关无发布 → 开启后 4 字段真收且主题 tenant/instance 正确 → 映射失败丢弃 → 租户主题隔离 → 无订阅者不报错 → ctx 取消退出 → 全停归零）；③ 提交 `2fc2a4d` 恰 8 文件全落 `ingest/**`、越界 0
- TDD 红转绿：4 处红（编译期缺符号 / opt-in 测试口径 / live 夹具 RESP 长度 mult / live 源 RESP2 bulk 取文本与 QoS1 掩码），每项修附「为什么现在对了」（见 §四）

---

## 一、改动文件清单（8 项，全部 `ingest/**`）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `ingest/internal/itembridge/bridge.go` | 新增 | **桥接主体与契约映射核心**：`NewValueSource`/`ItemPublisher` 薄接口（生产/测试零改动替换）；`ItemChangeEvent` 4 字段逐字（camelCase、ts 毫秒、quality 透传，`MarshalItemChange` 有序序列化）；`MapPointCode`（pointCode→fullKey 原样 + instance 去末段数据码，`F01.WS01.EQ001.M01`→`F01.WS01.EQ001`，空段/无分隔符/纯点判失败）；`NewValuePattern`=`iot:newvalue:*`、`TenantFromNewValueChannel`（channel tenant 解析，防跨租户注入）；`ItemTopic`=`iot/open/{t}/{i}/item` 逐字；`EnabledFromEnv`（`INGEST_ITEM_BRIDGE_ENABLED` 仅 true/1 开、默认 false）；`Bridge.Run` 单 goroutine 事件循环（源退出指数退避重连、ctx 取消退出）；`onEvent` 五段纪律（channel tenant 解析→payload 解析→payload/channel tenant 双一致→映射→发布），任一失败 WARN+计数丢弃（fail-soft）；`BridgeStats` 原子计数（Published/Dropped/PublishFailed） |
| `ingest/internal/itembridge/redis_source.go` | 新增 | **Redis 新值事件订阅源**：极简 RESP2（纯标准库 net+bufio，零新增依赖），`PSUBSCRIBE iot:newvalue:*` 一条连接承接全租户；握手 AUTH/SELECT 沿用 0029 纪律；push 解析严格 4 元素 pmessage（`sourceAsText` 兼容 bulk `[]byte` 与简单串 `string`）；行/bulk/数组长度上限防御；断线指数退避重连（200ms→5s 封顶）；ctx 取消经独立 goroutine 关连接解阻塞 Read；`writeRESP`/`sourceReadReply`/`sourceReadLine` 自含实现 |
| `ingest/internal/itembridge/paho_publisher.go` | 新增 | **MQTT 发布器**：Paho v1.5.1（go.mod 已钉，零新增依赖），独立 clientID（pid+seq 后缀多实例隔离，不复用消费会话）；`Publish` QoS1 fire-and-forget（无订阅者不报错）；`Connect` 初连失败快速上返（fail-soft 由装配层 WARN）；`Close` 幂等 |
| `ingest/internal/itembridge/bridge_contract_test.go` | 新增 | 契约测试 12 项：MapPointCode 恒等/设备段映射（3 例）、映射失败丢弃（5 例）、channel 模式逐字与 tenant 解析、ItemTopic 逐字、4 字段载荷逐字断言（有序 camelCase/毫秒/无 tenant）、map→发布 4 字段、映射失败丢弃+WARN、租户隔离（主题含 tenant/payload 无 tenant/跨租户 payload 丢弃）、发布失败 fail-soft 持续消费、坏 JSON 丢弃、ctx 取消退出、opt-in 默认关与真假值解析 |
| `ingest/internal/itembridge/bridge_live_test.go` | 新增 | live 全序列（env 门控 `INGEST_ITEM_BRIDGE_LIVE=1`，`//go:build live` 标签不进默认测试）：内嵌 MQTT 3.1.1 夹具 broker（CONNECT/CONNACK/SUBSCRIBE/SUBACK/PUBLISH QoS1+PUBACK/PINGREQ/DISCONNECT、通配符转发）+ 裸协议订阅者 + 真 Redis + 真 Paho 发布器 + 真桥接；S1–S7 七步真协议实证；`waitSubscribed` 以 `PUBSUB NUMPAT` 作订阅就绪观测 |
| `ingest/cmd/ingestd/main.go` | 修改 | 接线补全：`Config` 增 `ItemBridgeEnabled`/`ItemBridgeBroker`，`loadConfig` 经 `itembridge.EnabledFromEnv` 解析 opt-in；`run()` 在流水线装配后启动 `startItemBridge`（env 未开→启动日志明示「默认关」；开启但依赖缺失/broker 不可达→WARN 降级不阻断主链）；独立 goroutine 承载桥接事件循环，停机 `pcancel` 后调 `pub.Close`；头注释与组件口径段同步 TASK-0084 |
| `ingest/cmd/ingestd/wiring84_test.go` | 新增 | 接线契约 3 项：opt-in 默认关、env 覆盖（enabled+broker）、非法值（yes）不开启 |
| `ingest/README.md` | 修改 | 目录树增 `itembridge/`；「写库/缓存/事件」段增桥接口径（opt-in env/映射规则/4 字段/丢弃纪律/live 冒烟命令） |

依赖纪律：`go.mod`/`go.sum` 零改动（Paho v1.5.1 已在依赖中；Redis 走纯标准库 RESP2）。契约测试 **211 → 223**（新增 12 = itembridge 契约 12 + wiring84 接线 3 中 3 项非 Contract 命名的不计入；实际 Contract 命名项 +12），既有 211 项零改动全绿——**零回归**。

## 二、验收命令原始输出

### 2.1 验收 1：`cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（EXIT=0；提交 `2fc2a4d` 上终验）

```text
=== BUILD ===
BUILD_EXIT=0
=== VET ===
VET_EXIT=0
=== TEST ===
ok  	net.otlab/ingest/cmd/ingestd	3.826s
ok  	net.otlab/ingest/internal/cache	3.001s
ok  	net.otlab/ingest/internal/event	1.169s
ok  	net.otlab/ingest/internal/gwtoken	2.718s
ok  	net.otlab/ingest/internal/httpapi	3.104s
ok  	net.otlab/ingest/internal/itembridge	1.617s
ok  	net.otlab/ingest/internal/message	1.482s
ok  	net.otlab/ingest/internal/mqttsource	2.438s
ok  	net.otlab/ingest/internal/pipeline	2.969s
ok  	net.otlab/ingest/internal/quality	1.507s
ok  	net.otlab/ingest/internal/snapshot	3.740s
ok  	net.otlab/ingest/internal/tdengine	3.593s
TEST_EXIT=0
```

契约符合性全套（`go test ./... -run 'Contract' -v -count=1`，PASS/FAIL 计数）：

```text
=== CONTRACT COUNT ===
PASS=223 FAIL=
（223 = 基线 211 + 新增 12；FAIL 计数 0——awk 未匹配 FAIL 故留空；既有 211 项零改动全绿）
```

### 2.2 验收 2：live 桥接全序列（内嵌 MQTT 3.1.1 真协议夹具 broker + 真 Redis db15 + 真 Paho v1.5.1）

命令：`cd ingest && INGEST_ITEM_BRIDGE_LIVE=1 INGEST_REDIS_TEST_ADDR=127.0.0.1:6379 go test -tags live ./internal/itembridge/ -run 'TestLiveBridgeEndToEnd' -count=1 -v`

```text
=== RUN   TestLiveBridgeEndToEnd
2026/11/24 07:29:43 LIVE fixture broker=127.0.0.1:4936 redis=127.0.0.1:6379
LIVE-S1 opt-in 默认关：无发布（400ms 无消息）
LIVE-S2 桥接已开启：PSUBSCRIBE iot:newvalue:* 建好
LIVE-S3 真收 4 字段: topic=iot/open/bridge8476583/F01.WS01.EQ001/item payload={"fullKey":"F01.WS01.EQ001.M01","ts":1790000002000,"value":42.75,"quality":6}
LIVE-BRIDGE 2026/11/24 07:29:43 itembridge: WARN pointCode 映射失败，丢弃 channel="iot:newvalue:bridge8476583" pointCode="BADCODE" err=itembridge: pointCode "BADCODE" 无法解析 instanceCode（须含非空实例段与数据码段）
LIVE-S4 映射失败样本丢弃（500ms 无消息）
LIVE-S5 租户隔离: topic=iot/open/bridge8476583B/F02/item payload={"fullKey":"F02.M09","ts":1790000002000,"value":9.5,"quality":1}
LIVE-S6 无订阅者不报错：QoS1 fire-and-forget 发布成功，publishFailed 不增
LIVE-S7 ctx 取消退出: published=3 dropped=1 publishFailed=0
LIVE-ALL-DONE
--- PASS: TestLiveBridgeEndToEnd (0.95s)
PASS
ok  	net.otlab/ingest/internal/itembridge	2.543s
```

序列逐步判据（对应 brief 验收 2）：
- S1：桥接未装配时 PUBLISH 一条 `iot:newvalue:{tenant}` → MQTT 订阅 400ms 无消息（opt-in 默认关实证）。
- S2：开启桥接后 `PUBSUB NUMPAT ≥ 1` 观测 PSUBSCRIBE 就绪。
- S3：真发 `{tenant,pointCode=F01.WS01.EQ001.M01,ts=1790000002000,value=42.75,quality=6}` → 裸协议订阅者真收 `iot/open/{tenant}/F01.WS01.EQ001/item`，载荷逐字 `{"fullKey":"F01.WS01.EQ001.M01","ts":1790000002000,"value":42.75,"quality":6}`——topic tenant/instance 正确、4 字段 camelCase/毫秒、无 tenant 字段。
- S4：`pointCode=BADCODE`（无分隔符）→ WARN 丢弃，500ms 无 MQTT 消息（不造假主题）。
- S5：另一租户 `tenantB` 的 `F02.M09` → 主题 `iot/open/{tenantB}/F02/item`，载荷无 tenant（租户隔离）。
- S6：关掉订阅者后再发 → 发布仍成功（Published 增长）、`publishFailed` 不增（QoS1 fire-and-forget，无订阅者不报错）。
- S7：`cancel()` 后桥接 goroutine 退出；终态 published=3 / dropped=1 / publishFailed=0。
- 全停归零：夹具 broker 随机端口随进程退出关闭；`redis-cli PUBSUB NUMPAT` 终态 = 0（无残留模式订阅者）。

### 2.3 验收 3：改动范围核对（原始输出）

```text
=== HEAD ===
2fc2a4d
=== commit files ===
ingest/README.md
ingest/cmd/ingestd/main.go
ingest/cmd/ingestd/wiring84_test.go
ingest/internal/itembridge/bridge.go
ingest/internal/itembridge/bridge_contract_test.go
ingest/internal/itembridge/bridge_live_test.go
ingest/internal/itembridge/paho_publisher.go
ingest/internal/itembridge/redis_source.go
=== non-ingest count ===
0
=== diff scope BASE..HEAD ===
git diff --name-only fb8f66d..HEAD | grep -v '^ingest/' → 空
=== 工作区（并行在途，本任务未 add/未改）===
 M backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/impl/CalcReadServiceImpl.java
 M resources/sql/upgrade/20261124_iot_calc_schema.sql
?? backend/ot-admin/src/test/java/net/otlab/admin/module/iot/calcread/impl/CalcReadDataPermContractTest.java
?? backend/ot-admin/src/test/java/net/otlab/admin/module/iot/calcread/impl/IotCalcDdlSplitContractTest.java
（全部来自并行 0063 道；按提交粒度区分归属，本提交越界 0）
```

## 三、未做项与原因（brief 明确边界外，非缺口）

| 未做项 | 归属 | 原因 |
| --- | --- | --- |
| 真 EMQX broker 192.168.31.145:1883 上实收 | 环境侧 / 联调 | brief 明定 EMQX 1883 握手退化；以本地 MQTT 3.1.1 夹具 broker 完成真协议层全序列（真 Paho 发布侧 + 裸协议订阅侧），broker 地址 env 可切（`INGEST_ITEM_BRIDGE_BROKER` / 回退 `INGEST_MQTT_BROKER`），真机恢复后免改码复验 |
| 端到端 ingest 写库→桥接→0053 消费侧全链（跨 backend） | 联调 | 本任务 ingest 道只交付桥接生产者；0053 消费侧在 backend 道，brief「不碰 backend」；本批以真协议订阅者替代消费侧完成桥接出口实证，跨系统端到端归联调 |
| 桥接持久化 / 事件必达确认（QoS1 仅一次投递语义） | 演进位 | brief 决策 5 明定 QoS1 fire-and-forget、无订阅者不报错；「事件必达」需消费侧确认机制，非本批范围 |
| pointCode→fullKey/instance 的配置快照精确反查 | 演进位 | brief 决策 2 定版「pointCode 即 fullKey 核心、字段纪律不改」；映射按 pointCode 结构解析（去末段取设备段），与 `snapshot.DataItem.FullKey=instance_code+"."+data_code` 互逆，不引入快照依赖（ingest 快照缓存是上报校验用，非点位→数据项字典） |
| Redis 模式订阅持久会话 / 多实例分摊 | 演进位 | 当前每实例一条 PSUBSCRIBE 全量收（无消费者组语义）；多实例分摊归演进位，幂等/去重由下游 0053 承担 |
| TASK 状态置「已完成」 | 协调者收尾 | 硬约束：本任务保持「进行中」待评审，未改 workitems 库（唯一例外：本报告） |

## 四、偏离计划之处（含 TDD 红转绿与「为什么现在对了」）

### 4.1 TDD 红转绿留痕（4 处红，每项附为什么现在对了）

1. **RED（编译期——API 不存在）**：先写 `bridge_contract_test.go`（12 项）→ `go test ./internal/itembridge/` 编译错误 `undefined: MapPointCode / NewValuePattern / TenantFromNewValueChannel / ItemTopic / ItemChangeEvent / MarshalItemChange / NewBridge`（首 10+ 项）。GREEN：实现 `bridge.go`（薄接口 + 映射 + payload + 事件循环 + 计数）后 11/12 项立即绿，第 12 项（opt-in）见 4.2。**为什么现在对了**：新包 API 由测试逐字定义（4 字段有序序列化、channel/topic 逐字、映射边界），实现只暴露契约面，不为测试造多余出口。
2. **RED（opt-in 测试口径）**：`TestContractOptInDefaultOff` 初版把 `" true"` 列入「假值不应开启」，但实现按 ingest `loadConfig` 既有 `strings.TrimSpace` 口径先裁剪再判定 → 该项红。GREEN：改测试为「首尾空白属合法脏值，TrimSpace 后判定」（`" true"`/`"1 "` 视为真，`"yes"`/`"2"`/`"tru"` 等仍为假）。**为什么现在对了**：env 解析全仓一致（0029 各 env 键位均 TrimSpace）；宽松仅针对首尾空白，乱值仍 fail-closed 默认关，opt-in 安全性不变。
3. **RED（接线缺字段）**：`go test ./cmd/ingestd/ -run ItemBridge` 编译错误 `cfg.ItemBridgeEnabled undefined` / `cfg.ItemBridgeBroker undefined`（8 处）。GREEN：Config 增两字段 + loadConfig 解析 + `startItemBridge` 装配。为什么现在对了：opt-in 从 env 直达 Config 单一路径，装配失败 WARN 降级不污染主链；测试用 `getenvOf` 假环境锁定默认关/覆盖/非法值三态。
4. **RED（live 夹具 RESP 长度 mult=0）**：live S2 后 `SUBACK 失败 b0=0 err=EOF`；加临时夹具日志发现 broker 把 CONNECT 剩余长度读成 0（后续字节被当新包）。GREEN：夹具 `readMqttLen` 的 `mult` 初值由 `0` 改 `1`。**为什么现在对了**：MQTT 变长头按 7 位组 ×128^i 编码，第 0 组乘 1；初值 0 会把任意长度吞成 0，属夹具实现错误非产品代码。
5. **RED（live 源 RESP2 bulk 取文本）**：修好夹具后 S3 仍超时，bridge stats 全 0；日志显示 push 到但 `kind=` / `channel=` 空——pmessage 数组元素是 bulk（本 RESP 解码器返回 `[]byte`），`arr[0].(string)` 断言失败。GREEN：加 `sourceAsText(v any) string` 兼容 `string`/`[]byte`。**为什么现在对了**：RESP2 bulk 与简单串在解码层分别映射 `[]byte`/`string`，订阅 push 的 kind 与 channel 均为 bulk，必须按实际类型取文本后才能匹配 `pmessage` 与 tenant。
6. **RED（live 夹具 QoS1 掩码）**：源修好后 S3 收到载荷带 `\x00\x01` 前缀（`invalid character '\x00'`）——夹具把 QoS1 PUBLISH 的 packet-id 当载荷转发，且未回 PUBACK。GREEN：QoS 判定位由 `b0&0x06==0x04`（QoS2 掩码）改为 `==0x02`（QoS1）。**为什么现在对了**：MQTT QoS 在 flags 位 1-2，`(b0&0x06)>>1==1` 即 QoS1；正确识别后剥离 2 字节 packet-id 再转发净载荷，真 Paho 也收到合法 PUBACK，S3–S7 全绿。
   - 上述 4、5、6 均为夹具/测试侧修正（产品代码未因 live 红而改口径）；临时调试日志（LIVE-SRC / LIVE-BROKER-RECV / LIVE-SUB-CONNECT）全部移除，`grep -rn` 核验为 none。
   - 另有一处产品代码相关的 live 演进：源初版仅判 `len(arr)>=4` 后用 arr[0]/arr[2]/arr[3]，未处理确认包（`psubscribe` 回执）与 bulk 文本类型；经 RED 5 后定型为「kind 校验 + sourceAsText + tenant 形态校验」三关，避免把订阅确认当事件。

### 4.2 其他偏离

1. **live 用内嵌 MQTT 夹具 broker 而非真 EMQX**（非缺陷，环境规避）：与 0053/0046 先例同口径——真协议层（Paho 发布侧 + 裸协议订阅侧 + MQTT 3.1.1 报文编解码）全实证，仅 broker 地址 env 可切；恢复后免改码真机复验。
2. **live 测试用 `//go:build live` 标签隔离**：不进默认 `go test ./...`（避免无 Redis 环境跑冒烟误红）；由 `INGEST_ITEM_BRIDGE_LIVE=1` + `-tags live` 显式启用。默认套件仍覆盖桥接全部契约语义（12 项）。
3. **订阅源用 PSUBSCRIBE 而非逐租户 SUBSCRIBE**：brief 决策 1 要求订阅 `iot:newvalue:{tenant}`；以 `iot:newvalue:*` 单连接通配订阅 + channel tenant 解析实现（对新增租户无需重建订阅），channel 形态不符的 push 直接忽略，与逐租户 SUBSCRIBE 语义等价且运维更简。
4. **pointCode→instance 取「去末段」结构映射**：brief 给出 `F01.WS01.EQ001.M01`→设备段示例；实现为通用「最后 `.` 前为 instance、末段为数据码」规则（与 `snapshot.DataItem.FullKey=instance+"."+dataCode` 互逆），空段/无分隔符判失败丢弃，不臆造实例路径。

## 五、遗留问题

1. **真 EMQX 复验条件**：1883 恢复后 `INGEST_ITEM_BRIDGE_ENABLED=true INGEST_ITEM_BRIDGE_BROKER=tcp://192.168.31.145:1883 INGEST_REDIS_ADDR=127.0.0.1:6379` 起 ingestd，外部订阅 `iot/open/+/+/item` 断言 4 字段；本批交付已具备能力（env 可切），仅差环境。
2. **跨系统端到端（ingest 写库→Redis→桥接→0053 消费）**：本批只实证桥接出口（Redis newvalue→MQTT item topic），未跨 backend 跑 0053 消费侧（brief 不碰 backend）；归联调，0053 消费侧代码无需改动即可承接（topic/载荷已定版）。
3. **映射口径的配置化**：当前 instance 取点码结构前缀，未接配置快照反查（避免引入快照字典依赖）；若现场 pointCode 命名不遵循「实例路径.数据码」结构，映射会丢弃+WARN（不造主题）——需演进为配置驱动映射时改 `MapPointCode` 单点（javadoc 已登记）。
4. **观察面**：桥接计数（Published/Dropped/PublishFailed）仅进程内原子计数 + WARN 日志，未接 metrics/持久化；与 0029 事件发射侧计数同形态，归演进位。
5. **并行在途**：0063 道 backend/** 与 SQL 改动全程未触碰；`resources/sql/upgrade/20261124_iot_calc_schema.sql` 及 backend 测试文件为他人工作区状态，本提交越界 0。
6. **TASK 状态**：保持「进行中」待协调者 scoped 复评，未置「已完成」；workitems 库仅本报告变更。

## 一句话结论

**通过**——验收 1（`go build`/`go vet` 双零退出 + 12 包 `go test` 全 ok + 契约 223 PASS/0 FAIL〔基线 211 零回归 + 新增 12〕）、验收 2（live 真 Redis + 真 Paho + MQTT 3.1.1 夹具 broker 全序列：opt-in 关无发布、开启后 4 字段逐字真收且主题 tenant/instance 正确、映射失败丢弃、租户主题隔离、无订阅者不报错、ctx 取消退出、全停归零）、验收 3（提交 `2fc2a4d` 恰 8 文件全落 `ingest/**`、越界 0）三条全部以原始输出为证通过；TDD 4 处红转绿（编译期缺符号 / opt-in 测试口径 / 夹具 RESP 长度 mult / 源 RESP2 bulk 取文本 + QoS1 掩码）均附「为什么现在对了」；偏离为夹具替真 EMQX（环境退化，env 可切）与结构映射口径登记，遗留为真机复验/跨系统端到端/观察面演进，无阻断缺陷。

---
---

# 修订轮（2026-11-24）：diff 评审退回一轮（C-1 + I-1/I-2/M-1）

- 触发：`TASK-0084-review.md` 判不通过（Critical 1 + Important 2 + Minor 1）；协调者事实核查后 `TASK-0084-revision-brief.md` 裁决：**C-1 核心成立必修，I-1 评估登记即可，I-2 必修，M-1 可选；不采信评审关于 pointCode 字符集/点号禁用/0034-0036 的错误论证**。
- 修订范围 = 原范围（`ingest/**`）；修订 BASE = 主体提交 `2fc2a4d`；**修订提交 = `3e09775`**（注明 TASK-0084 修订轮，5 文件全落 `ingest/**`、越界 0）。
- 结论先行：**C-1 与 I-2 真修复**——pointCode 改为经配置快照反查数据项全键（下划线真实样本 `V29_NUM_01` live 真收），发布 ctx 从 Run ctx 派生取消即时；契约/live 全改下划线形态；`go build/vet/test ./...` 全绿、live 7 步全绿；I-1 评估登记不改码（与发射侧对称匿名形态），M-1 不强制未做。

## 修订轮一、权威口径核查（按 brief 事实核查纪律，不采信评审失真论证）

- `resources/contract/unified-message-model.md:28-35`：pointCode 真样本下划线形态 `F01_WS01_EQ001_M01`，消息只认 pointCode 不认 id。
- `resources/contract/config-snapshot-schema.md:165`：`fullKey = instance_code + "." + data_code`（`.` 是**数据项全键**分隔符，不是点位编码的分隔符）。
- `resources/contract/push-payload-evaluation.md:33`：pointCode 是点位编码（采集现场层），fullKey 是数据项全键，**两个不同编码空间**。
- `ingest/internal/pipeline/pipeline.go:680` + `TASK-0029-report.md:142-153`：newvalue 的 pointCode = `PointRow.PointCode` 原样，live 实测全下划线（`V29_NUM_01` / `V29_STATE_01` 等，无 `.`）。
- 结论：主体 `MapPointCode` 用 `LastIndex(".")` 解析点位编码，把真实事件全丢弃（opt-in 打开后发布恒为 0），主路径空转——C-1 核心成立。评审三处失真（pointCode 字符集 `[A-Za-z0-9_]`/点号禁用/0034-0036 映射依据）按 brief 不采信：pointCode 白名单实为 `[A-Za-z0-9_.]`（`IotPointServiceImpl.java:164`），契约从未禁点号；0034 未实施、0036 讲实例与数据项编码。

## 修订轮二、改动文件清单（提交 `3e09775`，5 项，全 `ingest/**`）

| 文件 | 修订内容 |
| --- | --- |
| `ingest/internal/itembridge/mapper.go`（新） | `PointMapper` 薄接口 + `SnapshotPointMapper` 实现：注入 `*snapshot.Cache`，`LookupPoint(pointCode)` → `Point.ID` → 快照 `DataItems` 中 `PointID==ID` 的 `DataItem` → `FullKey`（原样发布）+ 从 FullKey 用 `LastIndex(".")` 去末段取 instanceCode（此时 `.` 才是契约数据项全键分隔符，解析合法）。未装载/未登记/点位未绑数据项/FullKey 无实例段 → error（WARN+丢弃 fail-closed）。 |
| `ingest/internal/itembridge/bridge.go`（改） | 删 `MapPointCode`（字符串解析）；`Bridge` 持有 `PointMapper`；`NewBridge(src,pub,mapper,logger)` 注入映射；`onEvent(runCtx,...)` 用 mapper 反查；发布 `context.WithTimeout(runCtx,5s)` 取代 Background（I-2）。 |
| `ingest/cmd/ingestd/main.go`（改） | `startItemBridge(ctx,cfg,logger,snapCache)` 接收快照缓存；桥接装配 `NewSnapshotPointMapper(snapCache)`；调用点传 `snapCache`（main:277 已装配、与 reconciler 共享同一原子替换指针）。 |
| `ingest/internal/itembridge/bridge_contract_test.go`（改） | 新增测试快照夹具（下划线 pointCode 绑定）；删 2 项点分号 `MapPointCode` 用例、改 3 项快照反查用例（`V29_NUM_01`/`F01_WS01_EQ001_M01`/`V29_STATE_01` 命中数据项全键）+ 未绑定/未登记丢弃 + nil 缓存 fail-closed；桥接 6 项全改下划线样本；新增 I-2「取消中发布立即返回」契约（`blockingPub` 断言 pubCtx 从 Run ctx 派生、取消即时 Done）。itembridge 契约 12 → 14 项。 |
| `ingest/internal/itembridge/bridge_live_test.go`（改） | live 装载含下划线 pointCode 绑定的配置快照 → `NewBridge(..., NewSnapshotPointMapper(snapCache), ...)`；S1/S3/S4/S5/S6 全改下划线真样本；S3 真收 `fullKey=F01.WS01.EQ001.NUM_01`、topic instance=`F01.WS01.EQ001`；S4 改「快照未登记样本」丢弃。 |

## 修订轮三、TDD 红→绿（每项附「为什么现在对了」）

**C-1 RED（对主体实现必红，评审独立实测）**：旧 `MapPointCode("V29_NUM_01")` / `MapPointCode("F01_WS01_EQ001_M01")` → `err=no dot`（无 `.`），真实 0029 事件全丢弃；12 项契约 + live 7 步仅点分号样本 → 假阳性（主路径空转）。

**C-1 GREEN**：

```
$ go test ./internal/itembridge -run Contract -v -count=1
--- PASS: TestContractSnapshotMapperUnderscorePointCodeResolvesDataItemFullKey
    Map("V29_NUM_01")        → (F01.WS01.EQ001.NUM_01, F01.WS01.EQ001)
    Map("F01_WS01_EQ001_M01")→ (F01.M01, F01)
    Map("V29_STATE_01")      → (F01.WS01.EQ002.STATE_01, F01.WS01.EQ002)
--- PASS: TestContractSnapshotMapperUnboundOrMissingDrops
--- PASS: TestContractSnapshotMapperNilCacheFailsClosed
（itembridge 契约 14 项全 PASS）
```

- 为什么现在对了：`SnapshotPointMapper` 只经配置快照反查——pointCode 点位编码与 fullKey 数据项全键是不同编码空间，不下任何字符串结构假设；pointCode 原文只作 `LookupPoint` 的键，命中 Point.ID 后由 DataItem 绑定给出权威 fullKey/实例段；快照未装载/未绑定一律丢弃（fail-closed 不造主题），与 0029 实测下划线形态和冻结契约一致，主路径不再空转。

**I-2 GREEN**：

```
$ go test ./internal/itembridge -run TestContractPublishUsesRunCtxCancelInterruptsInflight -v
--- PASS: TestContractPublishUsesRunCtxCancelInterruptsInflight
    pubCtx 从 Run ctx WithTimeout 派生（非 ctx 本体）；cancel() 后 <300ms pubCtx.Done
```

- 为什么现在对了：`onEvent` 接收 Run 的 ctx，`context.WithTimeout(runCtx,5s)` 派生发布 ctx——父 ctx 取消沿派生链立即传播到 Paho `tok.Wait()`，停机最坏延迟从 5s 降为即时；保留 5s 超时边界不变。

## 修订轮四、验收命令原始输出

```
$ cd ingest && go build ./... && go vet ./... && go test ./...
BUILD_EXIT=0 ; VET_EXIT=0
ok  	net.otlab/ingest/cmd/ingestd	(cached)
ok  	net.otlab/ingest/internal/cache	(cached)
ok  	net.otlab/ingest/internal/event	(cached)
ok  	net.otlab/ingest/internal/gwtoken	(cached)
ok  	net.otlab/ingest/internal/httpapi	(cached)
ok  	net.otlab/ingest/internal/itembridge	1.043s
ok  	net.otlab/ingest/internal/message	(cached)
ok  	net.otlab/ingest/internal/mqttsource	(cached)
ok  	net.otlab/ingest/internal/pipeline	(cached)
ok  	net.otlab/ingest/internal/quality	(cached)
ok  	net.otlab/ingest/internal/snapshot	(cached)
ok  	net.otlab/ingest/internal/tdengine	(cached)
FINAL_EXIT=0
```

**live 7 步（真 Redis 6379 + 真 Paho + 内嵌 MQTT 3.1.1 夹具 broker + 装载下划线 pointCode 绑定快照）**：

```
$ INGEST_ITEM_BRIDGE_LIVE=1 INGEST_REDIS_TEST_ADDR=127.0.0.1:6379 go test -tags live ./internal/itembridge -run TestLiveBridgeEndToEnd -v -count=1
--- RUN   TestLiveBridgeEndToEnd
LIVE-S1 opt-in 默认关：无发布（400ms 无消息）
LIVE-S2 桥接已开启：PSUBSCRIBE iot:newvalue:* 建好
LIVE-S3 真收 4 字段: topic=iot/open/bridge8478069/F01.WS01.EQ001/item payload={"fullKey":"F01.WS01.EQ001.NUM_01","ts":1790000002000,"value":42.75,"quality":6}
LIVE-BRIDGE WARN pointCode 映射失败（快照反查未命中） pointCode="V29_UNBOUND_99" err=... 未在配置快照点位中登记
LIVE-S4 映射失败样本（快照未登记）丢弃（500ms 无消息）
LIVE-S5 租户隔离: topic=iot/open/bridge8478069B/F01/item payload={"fullKey":"F01.M01","ts":1790000002000,"value":9.5,"quality":1}
LIVE-S6 无订阅者不报错：QoS1 fire-and-forget 发布成功，publishFailed 不增
LIVE-S7 ctx 取消退出: published=3 dropped=1 publishFailed=0
LIVE-ALL-DONE
--- PASS: TestLiveBridgeEndToEnd (0.95s)
LIVE_EXIT=0
```

- S3 下划线 `V29_NUM_01` 真收 4 字段、fullKey 为快照 DataItem 全键（`F01.WS01.EQ001.NUM_01`）、topic instance=`F01.WS01.EQ001`——C-1 主路径闭合实证。
- S4 未绑定 `V29_UNBOUND_99` 丢弃、S5 租户 B 隔离、S6 无订阅者不报错、S7 ctx 取消退出全绿；未起任何 host 服务（夹具 broker random port、真 Redis 6379 只读订阅/PUBLISH），用毕即停。

**范围核对**：`git show --name-only 3e09775` = 5 文件全 `ingest/**`（main.go + itembridge 4），越界 0；契约文件/backend/frontend/calc 零触碰；TASK 保持「进行中」。

## 修订轮五、每项发现处理与遗留

1. **C-1（Critical）**：真修复——配置快照反查替代字符串解析，下划线真样本契约 + live 真收双实证；评审失真论证按 brief 不采信（pointCode 白名单 `[A-Za-z0-9_.]`、契约未禁点号、0034/0036 非映射依据）。
2. **I-1（Important）评估登记，不改码**：`RedisSourceOptions.Password` 未接线——与发射侧（`event/redis.go`、`cache.RedisOptions`）同口径匿名形态，宿主 Redis 6379 无密码，0029 实测匿名可用；按 brief「若发射侧本就匿名则不改代码，如实登记」。需密码环境由后续 env 扩展（统一 `REDIS_PASSWORD` 注入发射/缓存/桥接三处），本轮不引入不对称装配，登记为演进位。
3. **I-2（Important）**：真修复——发布 ctx 从 Run ctx 派生，取消即时（见上）。
4. **M-1（Minor）**：brief 可选、不强制——未做 WARN 采样（保持主体原子计数 + WARN 形态，与 0029 发射侧一致）；登记演进位，不阻断。
5. **真 EMQX 复验**：1883 仍退化，本轮沿用夹具 broker（env 可切）；跨系统 0053 消费侧端到端联调仍归演进位（topic/4 字段已定版，消费侧零改承接）。
6. **TASK 状态**：保持「进行中」待协调者 scoped 复评，未置「已完成」；workitems 库仅本报告变更。

## 修订轮一句话结论

**通过（修订轮）**——C-1 真修复：pointCode 经 `SnapshotPointMapper` 配置快照反查 `Point.ID → DataItem.FullKey`，下划线真样本 `V29_NUM_01` 契约命中 + live S3 真收（fullKey=`F01.WS01.EQ001.NUM_01`、topic instance=`F01.WS01.EQ001`），主路径不再空转；I-2 真修复：发布 ctx 从 Run ctx 派生、取消 <300ms 即时打断；I-1 评估登记不改码、M-1 未做（可选）；`go build/vet/test ./...` 12 包全 ok、itembridge 契约 14 项全 PASS、live 7 步 `LIVE-ALL-DONE`，提交 `3e09775` 5 文件全落 `ingest/**` 越界 0；TASK 保持「进行中」，待协调者 scoped 复评（只看 C-1/I-2 修复 diff 与证据）。
