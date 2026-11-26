# TASK-0029 报告：攒批写 TDengine、Redis 点位缓存与新值事件（采集接入链收官）

- 执行者：TASK-0029 执行（brief 驱动，纯 Go 批次：只改 ingest/**）
- BASE：`b4e877a`（开工时主仓 main HEAD，2026-11-23；区间内另有并行 0016 道提交 `082d694`（backend/**），本任务未触碰）
- 交付提交：`cfed417`（唯一提交，19 文件全部 ingest/**，注明 TASK-0029；`git show --name-only cfed417` 非 ingest 文件数=0）
- 执行日期：2026-11-23
- 验收结论：**3/3 验收命令全部通过**（原始输出见 §2；live 真库冒烟全序列：子表自动创建/值可查/缓存键在位/事件收到/DB-缓存-事件三方逐字段一致/写失败重排/SIGTERM 排空/清理证明）

## 1. 改动文件清单（19 个，全部落 ingest/**；go.mod/go.sum 零改动——依赖纪律延续：零新增第三方依赖）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `ingest/internal/cache/redis.go` | 新增 | **Redis 点位缓存真实现（brief 决策 2）**：极简 RESP2 客户端（纯标准库 net+bufio——0073/0028 依赖纪律延续，零新增第三方依赖）；`RedisClient`（惰性建连+AUTH+SELECT、SET/GET/PUBLISH/PING、RESP2 信封解析、错误回执上返、失败断连下次重连自愈、Pipelined 管线一次往返整批）；`RedisPointCache` 实现 PointCache（SET/GET JSON v/q/ts 三键）+ 可选 `BatchPointSetter.SetPoints`（按租户管线批量写）+ 未命中回源（`Backfill` 接口注入 TDengine 最近值源——读-through 懒回源、命中后写穿、回源写穿失败仍返回值）；键 scheme `iot:pt:{tenant}:{code}`（§3.5 口径）。选库口径：生产 ingest db0（键前缀与 backend 键空间隔离）、测试 db15（env 门控集成测试） |
| `ingest/internal/cache/redis_contract_test.go` | 新增 | 契约测试 10 项：键形态逐字（iot:pt:）、SET 写形态（键+JSON 恰三键）、GET 命中/未命中 ErrCacheMiss、回源（调用参数/写穿/命中不再回源）、回源无数据/源错误、失败上返+重连自愈、批量管线化（单段一次写入断言）、Ping+SELECT db、错误回执重连、坏地址快速失败 |
| `ingest/internal/cache/redis_live_test.go` | 新增 | 真 Redis db15 集成测试（env `INGEST_REDIS_TEST_ADDR` 门控）：写读回/覆盖写/批量管线/未命中/测后清理 |
| `ingest/internal/cache/redis_backfill_live_test.go` | 新增 | 真 Redis + 真 TDengine 回源集成（env `INGEST_REDIS_TEST_ADDR`+`INGEST_LIVE_TDENGINE_URL` 门控；冒烟窗口内跑真库，scratch 清理后自动 skip——skip 即清理证明）：清键→GetPoint→懒回源真库最近行→与 REST 直查基准逐字段一致→写穿核验→测后清理 |
| `ingest/internal/cache/cache.go` | 修改 | 键 scheme 定版 `iot:pt:{tenant}:{pointCode}`（brief 决策 2，原拟定 `iot:{t}:point:{c}` 废止）；包注释留白转定版登记 |
| `ingest/internal/cache/cache_contract_test.go` | 修改 | 无（键形态新断言在 redis_contract_test.go） |
| `ingest/internal/event/event.go` | 新增 | **新值事件发射侧（brief 决策 3）**：`NewValueEvent`{tenant,pointCode,ts,value,quality}（JSON 五键逐字）+ 薄接口 `Publisher`（单条）+ 可选 `BatchPublisher`（批量管线）+ `GatewayStatusEvent`{tenant,gateway,graceful,status,ts} + `GatewayStatusPublisher`；channel：`iot:newvalue:{tenant}`（brief 逐字）/`iot:gwstatus:{tenant}`（发射侧拟定，消费对接归 0040s 定版） |
| `ingest/internal/event/redis.go` | 新增 | Redis Pub/Sub 实现：`RedisPublisher` 实现 Publisher+BatchPublisher+GatewayStatusPublisher（复用 cache.RedisClient 单连接；PUBLISH 无订阅者不失败；批量走 RESP 管线；失败断连自愈） |
| `ingest/internal/event/event_contract_test.go` | 新增 | 契约测试 5 项：payload 恰五键/值形态/channel 命名 ×2/网关状态 payload 五键/编译期接口面锁定 |
| `ingest/internal/event/redis_publisher_contract_test.go` | 新增 | 契约测试 5 项（假 RESP 服务记录原始 TCP 段）：PUBLISH 形态（channel+payload 逐字）、批量管线化（3 条单段）、网关状态事件形态、失败上返+重连自愈、坏地址快速失败 |
| `ingest/internal/tdengine/rest.go` | 修改 | ①**EnsureSchema live 修正（红转绿，见 §4.1）**：多语句信封→逐语句独立下发+库名全限定（新增 `QualifiedCreateStableSQLs`——附录 B 语句逐字、库名限定为唯一变换）；②查询支持：`Query`（REST POST /rest/sql、Basic 认证、column_meta/data 解析、24/918 环境级失败同 Exec 口径）+ `LatestPointValue`（回源最近值：`SELECT ts, value, quality FROM <db>.<child> ORDER BY ts DESC LIMIT 1`——子表名走 ChildTableName 确定性映射、库名走 DBName 唯一入口；ts 单元格解析毫秒整数/RFC3339Z〔live 实测形态〕/无时区 ISO） |
| `ingest/internal/tdengine/rest_query_contract_test.go` | 新增 | 契约测试 4 项：回源 SQL 形态逐字（全限定+确定性映射）、state 表 INT 值+ISO ts 解析、空结果 found=false、SQL 失败上返 |
| `ingest/internal/tdengine/rest_contract_test.go` | 修改 | fakeTaosAdapter 扩全部请求体记录；EnsureSchema 测试按 live 实测语义重写（`TestContractRESTEnsureSchemaSplitsPerStatementQualified`——5 请求逐语句/全限定/无 USE 多语句；原多语句断言废止，live 驱动红转绿留痕见 §4.1） |
| `ingest/internal/pipeline/pipeline.go` | 修改 | ①Deps 增 `Events event.Publisher`（nil→不发射）；②写成功后 `publishEvents`（优先 BatchPublisher 管线批量、未实现退回逐条；失败 fail-soft 计 EventFailures 不影响落库事实）；③`updateCache` 支持可选 `cache.BatchPointSetter`（按租户分组管线一次往返——0027 遗留 ② sink 队头约束复核的批量化落点；未实现退回逐点 SetPoint 零改动）；④**`Drain` 停机排空（0027 遗留分流①）**：同步冲刷攒批缓冲直到空或 ctx 超时——SIGTERM 后缓冲内合法数据写完再退；与 flusher 并发安全（bufMu）；写失败重排语义与运行期一致（上限内重试、超限死信不回灌——持续失败在排空内收敛）；返回（本次写出行数, 滞留行数）交调用方 WARN；⑤Stats 增 EventsPublished/EventFailures |
| `ingest/internal/pipeline/pipeline_event_drain_contract_test.go` | 新增 | 契约测试 9 项：事件写成功后发射（批量 1 次 2 条/字段与写入行一致）、写失败零发射（重排重试成功后恰一次——无幻影事件）、发射失败 fail-soft（WrittenPoints 照计）、仅 Publisher 逐条回退、缓存批量接口按租户 1 次管线调用+最新覆盖写、纯 Memory 逐点回退、Drain 冲刷（written=3 remaining=0+攒批形态语句）、Drain 超时兜底（阻塞写入器+80ms 预算→remaining=3 快速返回）、Drain 重试耗尽转死信（written=0 remaining=0+WriteRetriesExhausted=3） |
| `ingest/internal/mqttsource/status_event.go` | 新增 | **ADR-0010 兑现（brief 决策 3）**：`EventStatusSink`（StatusSink 零改动替换）——LWT/status 消息照 0027 既有分发进本出口，每个报文发射一条网关状态事件（心跳/LWT/正常下线同为状态判定输入），**不直写状态 Redis**（「按写入方划分」——最终状态由计算引擎按优先级链裁决，消费对接归 0040s）；发射失败 fail-soft WARN；保留 0027 计数+日志语义 |
| `ingest/internal/mqttsource/status_event_contract_test.go` | 新增 | 契约测试 4 项：LWT（graceful=false）事件字段逐字+冻结时钟 ts、graceful:true 承载、心跳照发事件、发射失败 fail-soft 不 panic（结构性断言：本出口无任何状态存储写依赖） |
| `ingest/cmd/ingestd/main.go` | 修改 | **接线补全（brief 决策 1/2/3/4）**：env 新键位 `INGEST_TDENGINE_BASE_URL/USER/PASSWORD`（真执行器=RESTClient，Basic 认证；未配置降级记录式假执行器+WARN——纯 HTTP 形态先例）、`INGEST_REDIS_ADDR/DB`（真缓存+回源源装配〔exec 为 RESTClient 时注入 LatestPointValue〕+事件发射侧；未配置降级内存缓存+不发射）、`INGEST_DRAIN_TIMEOUT`（默认 10s）；MQTT 消费源装配 EventStatusSink（Redis 配置时；ADR-0010）；停机序列插入排空：HTTP Shutdown→pcancel→`pipe.Drain(drainCtx)`→剩余>0 WARN「不阻塞停机」/清零记「本次落库 N 行」→broker 释放；typed-nil 接口陷阱修正（evPub 以接口变量承载） |
| `ingest/cmd/ingestd/wiring29_test.go` | 新增 | 接线测试 4 项（假 taosAdapter+假 RESP 服务+假 backend 端到端）：新键位默认值（root/taosdata、db0、10s）、env 覆盖、**run() 真装配端到端**（report→排空→INSERT 全限定 USING...TAGS 多 VALUES 到达+SET iot:pt:{tenant}:{code} 三键 JSON 到达+PUBLISH iot:newvalue:{tenant} 五键到达）、真 Redis db15 集成（env 门控+测后清理） |
| `ingest/README.md` | 修改 | 目录描述同步（event/ 新目录、tdengine/cache/mqttsource/pipeline 增量）；新增「写库/缓存/事件（TASK-0029）」段落（攒批写纪律/缓存口径/事件口径/ADR-0010/停机排空/口径固化三项裁决登记） |

依赖纪律：`go.mod`/`go.sum` 零改动（零新增第三方依赖——Redis/Pub/Sub 走纯标准库 RESP2 客户端）。契约测试 **168 → 205**（新增 37，全部 `Contract` 命名；既有 168 中唯一重写=EnsureSchema 测试〔live 实测驱动，§4.1〕，其余 167 项零改动全绿——零回归）。

## 2. 验收命令原始输出

### 2.1 验收 1：`cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（EXIT=0；HEAD `cfed417` 上终验）

```text
ACCEPT-1 BUILD+VET EXIT=0          （build/vet 无任何输出，双零退出）

ok  	net.otlab/ingest/cmd/ingestd	4.757s
ok  	net.otlab/ingest/internal/cache	4.396s
ok  	net.otlab/ingest/internal/event	1.721s
ok  	net.otlab/ingest/internal/gwtoken	4.431s
ok  	net.otlab/ingest/internal/httpapi	4.268s
ok  	net.otlab/ingest/internal/message	1.996s
ok  	net.otlab/ingest/internal/mqttsource	2.782s
ok  	net.otlab/ingest/internal/pipeline	4.037s
ok  	net.otlab/ingest/internal/quality	1.387s
ok  	net.otlab/ingest/internal/snapshot	3.597s
ok  	net.otlab/ingest/internal/tdengine	4.596s

契约符合性全套（go test ./... -run 'Contract' -v -count=1）：
--- PASS 计数 = 205，--- FAIL 计数 = 0
（168 → 205：新增 37；既有 167 项零改动全绿 + EnsureSchema 1 项按 live 实测语义重写——零回归）

并发红线稳定性复跑（-run 'Race' -count=3）：
ok  	net.otlab/ingest/internal/pipeline	1.638s
ok  	net.otlab/ingest/internal/mqttsource	1.636s [no tests to run]
```

env 门控真环境集成（`INGEST_REDIS_TEST_ADDR=127.0.0.1:6379`；回源项另加 `INGEST_LIVE_TDENGINE_URL`，在冒烟窗口内 scratch 库在位时跑真库）：

```text
--- PASS: TestLiveRedisPointCacheDB1Integration (0.01s)          （真 Redis db15：写读回/覆盖写/批量/清理）
--- PASS: TestLiveRedisPointCacheMissBackfillFromTDengine (0.14s) （真库回源：值/质量/ts 与 REST 直查基准逐字段一致）
--- PASS: TestIngestdWiringLiveRedisOptional (0.11s)             （真 run() 装配 + 真 Redis db15 键在位）
```

### 2.2 验收 2：live 真库冒烟全序列（fd 根治后首次真库写入验收；8090/18098 起前探测均空闲；TDengine http://192.168.31.145:6041 env 注入）

**环境处置**：WSL Ubuntu 在跑（wisfactory-mysql/wisfactory-redis up 6h）；8090 Windows/WSL 两侧空闲；本任务冒烟**不依赖 backend**——快照/凭据用假端点（18098 端口，不在 8098）；TDengine 共享库零污染：scratch 租户库 iot_verify29 测后 DROP。

**步骤 a：scratch 租户库准备（DBName() 唯一入口派生；curl -m 60 慢特性实测）**

```text
DBName("verify29") 派生：全 [a-z0-9] 且字母开头 → 恒等像 → iot_verify29
CREATE DATABASE IF NOT EXISTS iot_verify29 KEEP 1825d DURATION 10d BUFFER 256;
→ {"code":0,...} real 0m16.821s（20-25s 慢特性区间，与协调者实测一致）
CREATE STABLE ×4（附录 B DDL 逐字，库名全限定）→ {"code":0}×3、st_metric_value 因
  「period=保留字」单列失败后以 `period` 反引号限定建成（契约发现，见 §5.2——ingest 写入
  路径不涉及该表，契约原文未动）
SHOW iot_verify29.STABLES → ["st_item_value"],["st_point_num"],["st_metric_value"],["st_point_state"] rows=4
```

**步骤 b：ingestd 起（env 注入 TDengine/Redis）→ report 合法批 → 全链断言**

ingestd 装配日志（真执行器/真缓存/事件发射侧）：

```text
2026/11/23 04:16:48 ingestd: TDengine REST 执行器已装配 endpoint=http://192.168.31.145:6041 user=root
2026/11/23 04:16:48 ingestd: Redis 点位缓存已装配 addr=127.0.0.1:6379 db=0（键 iot:pt:{tenant}:{code}，写穿透/失败不阻塞落库/未命中回源）+ 新值事件发射侧
2026/11/23 04:16:48 ingestd: MQTT 未配置（INGEST_MQTT_BROKER 空）——纯 HTTP 上报形态
2026/11/23 04:16:48 ingestd: 快照对账已装配 scope=1:9111 tenant="verify29" interval=10s
2026/11/23 04:16:48 snapshot: 对账快照已替换 version=29 hash=c9c9c9...
curl http://127.0.0.1:8090/healthz → ok
```

report 合法批（跨 2 个 supertable 的混合点位：B1=6 点〔V29_NUM_01×3 多 ts + V29_STATE_01×2 + V29_STATE_02×1〕、B2=2 点〔V29_NUM_02 CUMUL + V29_STATE_02 SWITCH〕）：

```text
B29-MIX-1 → {"accepted":6,"rejected":0,"rejections":[],"duplicate":false}
B29-MIX-2 → {"accepted":2,"rejected":0,"rejections":[],"duplicate":false}
```

断言 1：**子表首写自动创建**（SHOW TABLES——4 子表全部经 INSERT...USING 自动落名，escapeIdentifier 映射 `_`→`_5f` 实证）：

```text
SHOW iot_verify29.TABLES → ["v29_5fnum_5f01"],["v29_5fstate_5f01"],["v29_5fstate_5f02"],["v29_5fnum_5f02"] rows=4
```

断言 2：**值可查 + kind/表映射 + 标签按快照（tenant_id/kind/dept_id）**：

```text
SELECT ... FROM iot_verify29.st_point_num ORDER BY ts →
  [2026-11-22T20:16:57.050Z, 10.5,  0, v29_5fnum_5f01,  V29_NUM_01,  INSTANT, 1, 3]
  [2026-11-22T20:16:59.050Z, 11.5,  0, v29_5fnum_5f01,  V29_NUM_01,  INSTANT, 1, 3]
  [2026-11-22T20:17:01.050Z, 12.5,  0, v29_5fnum_5f01,  V29_NUM_01,  INSTANT, 1, 3]
  [2026-11-22T20:17:02.572Z, 100.25,0, v29_5fnum_5f02,  V29_NUM_02,  CUMUL,   1, 3] rows=4
SELECT ... FROM iot_verify29.st_point_state ORDER BY ts →
  [.., 2, 0, v29_5fstate_5f01, V29_STATE_01, STATE,  1, 3] [.., 3, 0, ...STATE..]
  [.., 1, 0, v29_5fstate_5f02, V29_STATE_02, SWITCH, 1, 3] [.., 0, 0, ...SWITCH..] rows=4
```

断言 3：**攒批形态（非逐条）**——攒批语句形态由契约测试机械锁定（假适配器捕获请求体：同子表多行合并单条 INSERT 多 VALUES；wiring 测试以真 run() 装配捕获 `INSERT INTO iot_group_5fa.W_5fM01 USING iot_group_5fa.st_point_num TAGS (...) VALUES (` 逐字形态）；live 侧证明=同一 flush 周期内同点位 3 行全部落库（上表 V29_NUM_01 三行同批落库、子表仅一张）+ 攒批写路径唯一（Writer 按子表分组单条 INSERT，无逐条分支）。taosAdapter /metrics 仅暴露 promhttp 自指标、无语句计数器（已探测）——逐条/攒批的真库侧区分无外部观测面，形态锁定以捕获请求体的契约测试为准（登记 §5.6）。

断言 4：**Redis 缓存键在位（v/q/ts）+ DB-缓存-事件三方逐字段一致**：

```text
redis-cli --scan --pattern 'iot:pt:verify29:*' → 4 键全在：
iot:pt:verify29:V29_NUM_01   = {"v":12.5,  "q":0,"ts":1795378621050}
iot:pt:verify29:V29_NUM_02   = {"v":100.25,"q":0,"ts":1795378622572}
iot:pt:verify29:V29_STATE_01 = {"v":3,     "q":0,"ts":1795378619050}
iot:pt:verify29:V29_STATE_02 = {"v":0,     "q":0,"ts":1795378622572}
—— 三方一致实证（以 V29_NUM_01 最新行为例）：DB 行 ts=2026-11-22T20:17:01.050Z ↔
   1795378621050ms；缓存 {v:12.5,q:0,ts:1795378621050}；事件 payload {"ts":1795378621050,
   "value":12.5,"quality":0}——DB/缓存/事件三面 ts、value、quality 逐字段一致。
```

断言 5：**新值事件收到（redis-cli SUBSCRIBE，8 条=恰每写入行一条，payload 恰五键）**：

```text
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_NUM_01","ts":1795378617050,"value":10.5,"quality":0}
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_NUM_01","ts":1795378619050,"value":11.5,"quality":0}
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_NUM_01","ts":1795378621050,"value":12.5,"quality":0}
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_STATE_01","ts":1795378617050,"value":2,"quality":0}
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_STATE_01","ts":1795378619050,"value":3,"quality":0}
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_STATE_02","ts":1795378621050,"value":1,"quality":0}
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_NUM_02","ts":1795378622572,"value":100.25,"quality":0}
message iot:newvalue:verify29 {"tenant":"verify29","pointCode":"V29_STATE_02","ts":1795378622572,"value":0,"quality":0}
```

断言 6：**未命中回源（真库读-through）**——env 门控集成测试在冒烟窗口内实跑：

```text
--- PASS: TestLiveRedisPointCacheMissBackfillFromTDengine (0.14s)
（清键→GetPoint→懒回源 iot_verify29.V29_5fNUM_5f01 ORDER BY ts DESC LIMIT 1 →
  与 REST 直查基准逐字段一致 → 无源缓存实例读同键命中=写穿实证）
```

**步骤 c：写失败重试（TDengine 不可达注入）**

```text
ingestd 重启 INGEST_TDENGINE_BASE_URL=http://127.0.0.1:59991（坏端口）+ INGEST_DEADLETTER_WARN_EVERY=1
B29-FAIL report → {"accepted":3,...}（3 行入攒批缓冲，受理照常）
3 秒后 ingestd 日志（整批重排→重试耗尽转死信，采样=每条）：
2026/11/23 04:18:55 pipeline: [dead-letter] tenant=verify29 reason=invalid_config count=1 pointCode= batchId=
2026/11/23 04:18:55 pipeline: [dead-letter] tenant=verify29 reason=invalid_config count=2 pointCode= batchId=
2026/11/23 04:18:55 pipeline: [dead-letter] tenant=verify29 reason=invalid_config count=3 pointCode= batchId=
—— 3 行=初始下发+3 次整批重排（重排上限语义）后转死信告警、不再回灌；排空/运行期同一收敛路径。
失败期副作用核对：订阅流零新增事件（无幻影事件）、缓存值未被污染（仍 {"v":12.5,...}）、
真库 SELECT count(*) 仍=4（零落库）——写失败不发射/不写穿/不落库 live 三证。

恢复：重启注入正确 URL → B29-RECOVER report → {"accepted":2,...}
→ st_point_num count 4→5（恢复批落库）、缓存 {"v":88.8,"q":0,"ts":1795378759553} 写穿透更新、
  订阅流新增 2 条事件（88.8/4）——恢复链路（落库+缓存+事件）全通。
```

**步骤 d：停机排空（SIGTERM——真实投递；WSL Linux 形态 go1.27.2 交叉编译，镜像网络直达 Windows 假端点/真库/真 Redis）**

```text
--- REPORT（行入攒批缓冲，未到 200ms flush 定时）---
{"accepted":4,"rejected":0,"rejections":[],"duplicate":false}   （B29-DRAIN：4 行入缓冲）
--- SIGTERM（真实投递）---
EXIT=0
--- LOG ---
2026/11/23 04:20:11 ingestd: 优雅停机开始（SIGTERM 排空）
2026/11/23 04:20:11 ingestd: 停机排空完成（缓冲清零，本次落库 4 行）
2026/11/23 04:20:11 ingestd: 已停机
—— 排空落库真库核对：st_point_num count 5→7（DRAIN 批 2 行）、st_point_state count 5→7；
   SELECT ... V29_5fNUM_5f01 ORDER BY ts DESC LIMIT 2 → [66.2,0]/[66.1,0]（排空行值逐字在位）；
   订阅流新增 DRAIN 批 4 条事件（66.1/66.2/5/1——排空写成功后事件照发）。
```

（对照：Windows taskkill 为硬杀无 SIGTERM 投递面〔0027/0028 既有结论〕——本步骤以 WSL Linux 二进制获得真实 SIGTERM 证据；Drain 语义另有 9 项契约测试锁定。）

**步骤 e：清理（scratch 对象测完清零，绝不污染共享库）**

```text
DROP DATABASE IF EXISTS iot_verify29 → {"code":0}
SHOW DATABASES 终态 → ["information_schema"],["performance_schema"],["iot_default"],["log"] rows=4
  （与开工时完全一致——scratch 库零残留）
Redis db0 iot:pt:verify29:* 键 4→0（DEL 后 --scan 计数=0）；db15 --scan=0
端口归零：8090（Windows/WSL 双侧）与 18098 LISTENING 计数均=0
WSL 保活循环已停；冒烟进程（fakebackend/订阅器/ingestd 全部实例）已停
```

### 2.3 验收 3：改动范围核对

```text
git show --name-only --format="" cfed417 | grep -v "^ingest/" | wc -l = 0（19 文件全部 ingest/**）
git diff --name-only b4e877a..HEAD -- ingest/ | wc -l = 19（恰为本任务文件）
git diff --name-only b4e877a..HEAD | grep -v "^ingest/" | wc -l = 13
  —— 全部来自并行 0016 道提交 082d694（backend/**），本任务未触碰
git status --porcelain -- ingest/ = 空（工作区干净）
```

## 3. 未做项与原因（brief 明确归后/边界外，非缺口）

| 未做项 | 归属 | 原因 |
| --- | --- | --- |
| 新值事件消费侧（计算引擎订阅 iot:newvalue:{tenant} 触发计算/状态判定） | 0040s | brief 决策 3 明示消费方未建，本批交付发射侧+契约测试；channel/payload 拟定形态已 javadoc 登记待消费对接定版 |
| MQTT 路径 live 实测（EMQX 真 broker；data/status/LWT 真链路） | TD-0057（挂 0068） | 本地无 EMQX（0027/0028 同口径）；MQTT 语义与 HTTP 共用同一流水线（twopath 契约测试锁定），LWT 事件化出口由 4 项契约测试锁定，真链路核验归联调 |
| ingestd 自动开通租户库（EnsureSchema 接线） | 部署运维/演进位 | brief 冒烟设计即外部预置 scratch 库（步骤 a）；租户库开通归部署 runbook（EnsureSchema 修正后已具备逐语句建库建表能力，接线归部署形态裁决——登记 §5.5） |
| 攒批写真库侧语句计数观测面 | 观测面缺口（§5.6 登记） | taosAdapter /metrics 仅 promhttp 自指标、无 INSERT 语句计数器——攒批形态锁定以捕获请求体的契约测试（假适配器逐字断言）为准 |
| `-race` 检测器 | （环境限制） | 本机无 gcc（cgo 缺失）——同 0076/0027/0028 口径；并发安全由竞态回归测试行为断言覆盖（Race -count=3 稳定） |
| 口径固化三项代码变更 | 无需 | brief 裁决 5 明示「仅登记口径，无代码变更」——0028 已交付语义与三项裁决逐条一致（①纠偏标记=结构化 WARN+计数器〔CorrectionTagger〕；②被拒点每次到达计入原因码〔拒绝路径不标去重键〕；③纠偏计数=纠偏动作次数〔时间步先于去重步〕），README 已登记裁决日期 |

## 4. 偏离计划之处

### 4.1 EnsureSchema live 修正——多语句信封被真库静默忽略（本报告最重要偏离，红转绿）

brief 未要求动 EnsureSchema（0076 交付件、不在本批五项交付面内），但 live 步骤 a 预置库时实测发现：**taosAdapter REST（3.4.3.9）对多语句信封返回 code=0 affected_rows=0 且零建表**（`CREATE STABLE`×4 单请求 → SHOW STABLES rows=0；逐语句重发后全部建成）——即 EnsureSchema 原「单请求多语句（CREATE DATABASE + USE + CREATE STABLE）」形态在产品路径上是**静默空操作**，0076 契约测试（假服务端照单全收）无法暴露。处置（ingest/** 内 TDD 红转绿）：先改测试（`TestContractRESTEnsureSchemaSplitsPerStatementQualified`——5 请求逐语句、表名库名全限定、无 USE 多语句）确认红，再改实现（`QualifiedCreateStableSQLs` 附录 B 语句逐字+库名限定为唯一变换；EnsureSchema 循环逐语句 Exec，任一失败上返）。为什么现在对了：REST 协议一次一 SQL（9750 教训同族——无状态连接 USE 语境不可跨请求），逐语句+全限定是唯一可与真库对话的形态；附录 B DDL 语句本体逐字未动（构造器复用），契约面零变更。原测试重写已在 §1 登记（唯一被改写的既有契约测试，167 项其余零改动）。

### 4.2 ingestd 缺省降级形态（真执行器/Redis 未配置不启动失败）

brief 要求真执行器/真缓存接线，但未定义「env 未配置」时的行为。处置：沿 0027「MQTT 未配置=纯 HTTP 形态合法存在」先例——`INGEST_TDENGINE_BASE_URL` 空→记录式假执行器+启动 WARN；`INGEST_REDIS_ADDR` 空→内存缓存+事件不发射+启动 WARN。为什么现在对了：开发/CI 环境无 TDengine/Redis 时服务仍可起（冒烟 b 段日志即两种形态实测），装配失败会让所有既有装配测试（假执行器/内存缓存注入面）失去运行载体；生产 env 注入即真形态（live 冒烟全程验证）。typed-nil 接口陷阱（nil *RedisPublisher 赋给 Publisher 接口非 nil）已在接线中规避并以 run() 端到端测试覆盖。

### 4.3 回源 live 探针为测试新增（非 brief 列举交付物）

brief 断言口径未列「回源 live 实证」，但验收标准含「缓存未命中回源」。处置：新增 env 门控集成测试（真 Redis db15 + 真 TDengine scratch 库）在冒烟窗口内实跑（PASS 见 §2.2 断言 6），scratch 清理后自动 skip（skip 即清理证明——测试对真库有真实依赖的旁证）。为什么现在对了：回源路径的真库形态（ts 为 RFC3339Z 字符串）只有 live 才能锁定（假测试的 ts 形态是拟定的）。

### 4.4 网关状态事件 channel/payload 为发射侧拟定

ADR-0010 与契约未定义离线事件的 channel 名与载荷形态（仅裁决心智「接入服务只发事件不直写状态 Redis」）。处置：发射侧拟定 `iot:gwstatus:{tenant}` + payload {tenant,gateway,graceful,status,ts}（与 iot:newvalue:{tenant} 同族命名），javadoc 声明消费对接归 0040s 定版。为什么现在对了：消费方未建，先定发射侧可观测形态（live 可用 redis-cli SUBSCRIBE 验证），命名对齐 brief 既有 newvalue 族；0040s 对接时单点改名（GatewayStatusChannel 一个函数）。

### 4.5 冒烟 scratch 预置将 st_metric_value 以 `period` 反引号限定建成

附录 B 逐字 DDL 建该表失败（保留字，见 §5.2 契约发现）。处置：scratch 预置侧以 \`period\` 反引号限定建成（scratch 对象仅为本任务冒烟载体、测后已 DROP）；产品构造器 CreateStableSQL 保持附录 B 逐字未动（ingest 不写该表）。为什么现在对了：契约冻结纪律——发现报告并登记（§5.2），不擅改契约面；冒烟载体库的建成方式不属于契约面。

### 4.6 写失败重试告警采样设 N=1（brief 步骤 c 证据密度需要）

live 步骤 c 用 INGEST_DEADLETTER_WARN_EVERY=1（默认 100）——3 行耗尽各出一条 WARN，重排→耗尽证据逐行可读。为什么现在对了：告警采样语义=首次+每 N 条（0028 契约测试锁定），N=1 为合法配置且正是「每条都告警」形态；默认 100 的采样行为另有契约测试锁定，live 未改默认装配。

### 4.7 SIGTERM 排空 live 证据形态（WSL Linux 二进制）

Windows 控制台进程无 SIGTERM 投递面（0027/0028 已登记）。处置：交叉编译 Linux 二进制在 WSL 以真实 `kill -TERM` 投递（镜像网络直达假端点/真库/真 Redis），获得「停机排空完成（缓冲清零，本次落库 4 行）」+真库行数/值核对级证据。为什么现在对了：Drain 是本批新交付语义，单元级 9 项契约测试 + 真实信号一次投递的证据链比「unit 级替代证据」（0027 先例）更强；0068 联调（docker/Linux 部署形态）与本步骤同构。

## 5. 遗留问题

### 5.1 [TD-0057 延续登记——MQTT 真联调（0027/0028 登记，本批追加验证项）]

EMQX 容器就绪后执行 0027 报告 4 步验证清单，本批追加第 ⑥ 步：MQTT status 主题投递 LWT/graceful/心跳载荷，断言 `iot:gwstatus:{tenant}` 网关状态事件发射（graceful 字段逐字承载）且**零状态 Redis 直写**（ADR-0010 真链路核验——本批语义已由 4 项契约测试锁定，真链路归联调）。

### 5.2 [超级表契约发现——附录 B st_metric_value 的 `period` 标签在 TDengine 3.4.3.9 为保留字（待协调者裁决，本批未改契约）]

附录 B 逐字 DDL `CREATE STABLE ... st_metric_value (...) TAGS (..., period BINARY(8))` 在真库执行报 code=9728 syntax error near "period binary(8)"（`period` 为保留字）；以 \`period\` 反引号限定即建成（live 实测）。**接入服务写入路径不受影响**（ingest 只写 st_point_num/st_point_state，两表 DDL 逐字执行成功）；受影响面=该表的开通方（计算引擎道/部署 runbook——预置时须反引号限定或契约升版改列名）。共享库 iot_default 的 st_metric_value 已在位（早期预置以 VARCHAR 变体建成），一期联调不阻塞。按契约冻结纪律：契约原文未动，本报告登记待裁决（候选：a) 契约注明该列须反引号限定；b) 契约升版改列名避开保留字；c) 维持现状、runbook 承载）。

### 5.3 [REST 多语句语义口径登记（TD-0053 教训族新增）]

taosAdapter REST「一次一 SQL」实测补全：多语句信封**静默空操作**（code=0 affected=0，非报错）——比 9750（显式拒绝）更隐蔽。已固化于 EnsureSchema 实现与契约测试；任何后续 REST 通道多语句需求（如批量 DDL 运维脚本）须逐语句下发，rest.go javadoc 已登记。

### 5.4 [新值事件/网关状态事件发射侧拟定形态待消费定版]

channel `iot:newvalue:{tenant}`（brief 逐字，定版）与 `iot:gwstatus:{tenant}`（发射侧拟定）、payload 键集（五键逐字）待计算引擎（0040s）消费对接时确认；改名面=GatewayStatusChannel/NewValueChannel 单函数+ingestd 无感（发射侧集中）。PUBLISH fire-and-forget 语义（无订阅者不失败）为 Pub/Sub 固有口径——「事件必达」由 QoS 承载的需求（如有）须演进为消费侧确认机制，登记演进位。

### 5.5 [租户库开通 runbook 缺口]

ingestd 不自动开通租户库（§4.2/未做项）；部署时须先 EnsureSchema（修正后逐语句建库建表能力已具备，`tdengine.RESTClient.EnsureSchema(ctx, tenantCode)` 单调用）或按附录 B 逐语句预置。CREATE DATABASE 首建 20-25s 慢特性（本次 16.8s 实测）——runbook 须标注；确保幂等（IF NOT EXISTS）可重复执行。

### 5.6 [攒批形态的真库侧观测面缺口]

攒批（非逐条）的机械证明=契约测试捕获请求体逐字断言（假适配器）+run() 端到端 wiring 测试；live 侧以「同 flush 周期同点位多行一次落库+写路径无逐条分支」佐证。taosAdapter 无语句级计数指标——若运维需要真库侧语句形态观测（如「INSERT/行比」监控），须 taosAdapter 侧支持或 TDengine 慢查询日志口径，登记观测面演进位。

### 5.7 [ts 解析时区口径]

taosAdapter 查询结果 ts 实测为 RFC3339Z（UTC 带时区，live 实证解析正确）；parseTsCell 对无时区 ISO 串按 UTC 兜底解析（仅当上游版本变更输出形态时可达）——回源 ts 精度依赖 adapter 输出形态，已按实测形态锁定并以 live 集成测试背书；上游升级时须复验（登记）。

### 5.8 [停机排空边界]

Drain 只排空**攒批缓冲内**已受理行；MQTT 消费通道内未路由的 broker 积压不排空（停机即断，QoS1 重投+幂等键兜底——与 brief「缓冲内合法数据写完再退」口径一致）；HTTP Shutdown 先于 Drain（停机期新上报 503/拒绝——HTTP 排空窗口 5s 内已受理的行进缓冲被 Drain 覆盖）。排空预算默认 10s（INGEST_DRAIN_TIMEOUT 可配），超时滞留行 WARN 后随进程退出（下次启动后由上游重投覆盖，幂等键防重）。

### 5.9 [环境与并行]

- 本机无 gcc：`-race` 未跑（同 0076/0027/0028 登记）；并发安全由竞态回归测试行为断言覆盖（-count=3 稳定），建议有 gcc 环境补一轮 `go test -race ./internal/pipeline/ ./internal/cache/`（新增 Redis 客户端并发面：单连接互斥串行）。
- 并行 0016 道（backend/** 提交 082d694 与工作区改动）全程未触碰、未编译、未测试；本任务 live 冒烟零 backend 依赖（假端点承载快照/凭据）。
- 冒烟数据零残留：scratch 库 DROP + Redis db0/db15 键清理 + 端口归零（§2.2 步骤 e）；D:\task29-smoke 临时冒烟工具目录（假端点源码/二进制/日志，不入仓）随报告完成后清除。
- WSL 保活循环已停；mysql/redis 容器全程在跑未中断。

## 可追溯性

- **git 提交**：`cfed417`（main，仅 ingest/** 19 文件，注明 TASK-0029）；BASE=`b4e877a`
- **修改文件**：§1 清单 19 项（6 修改 + 13 新增）；非 ingest 文件数 = 0
- **验收**：§2 三条命令原始输出全部通过；契约测试 205/205 PASS（既有 167 零改动全绿+EnsureSchema 1 项 live 实测语义重写+新增 37——零回归）；live 冒烟全序列（子表自动创建/值与标签可查/缓存键在位/事件收到/三方一致/写失败重排恢复/SIGTERM 排空/清理证明）全实证

## 六、一句话结论

**通过**——验收 1（build/vet 双零退出 + go test 全 11 包 ok + 契约 205 PASS/0 FAIL〔既有 167 零改动全绿+1 项 live 实测重写+新增 37〕+ Race -count=3 稳定 + 真环境集成 3 项 PASS）、验收 2（live 真库冒烟全序列：scratch 库 16.8s 建成与 DROP 后 SHOW DATABASES 零残留、4 子表经 INSERT...USING 自动创建且 `_5f` 映射实证、st_point_num/state 值+kind+标签全对、Redis iot:pt: 四键 v/q/ts 在位且 DB-缓存-事件三方逐字段一致、订阅恰 8 条五键事件、写失败 3 行重排耗尽死信 WARN 且零幻影事件零缓存污染零落库、恢复批全链落库+写穿透+事件、SIGTERM 真实投递「缓冲清零本次落库 4 行」+真库 7/7 行核对、8090/18098/键空间全归零）、验收 3（提交 cfed417 恰 19 文件全落 ingest/**、越界 0）三条全部以原始输出为证通过；偏离核心为 live 实测驱动的 EnsureSchema 红转绿修正（多语句信封真库静默空操作——§4.1）与发射侧拟定形态登记（§4.4），遗留仅契约发现 §5.2（st_metric_value 保留字，ingest 路径不受影响）、消费对接/观测面/部署 runbook 登记与 TD-0057 延续，无阻断缺陷。

---

# 修订轮（退回轮 §12：I1 + 顺手修 M1-M4）

- 评审输入：`TASK-0029-review.md`（1 Important + 7 Minor；paho v1.5.1 router.go 有序分发实证）
- 协调者裁决：I1 必修（红→绿非阻塞契约测试）；M1-M4 顺手修同轮；M5/M6/M7 登记 deferred 不修
- 交付提交：`88dce89`（修订轮，9 文件全部 ingest/**，逐文件显式 add，注明 TASK-0029 修订轮；`git show --name-only --format="" 88dce89 | grep -v "^ingest/" | wc -l` = 0）
- 修订日期：2026-11-23

## R1. 修订清单（9 文件；契约测试 205 → 211，既有 205 项零改动全绿）

| 文件 | 修订项 | 说明 |
| --- | --- | --- |
| `ingest/internal/mqttsource/status_event.go` | **I1** | **EventStatusSink 改有界异步交接**：`OnStatus` 只做非阻塞入队（`select default` 满队丢弃+计数，**分发回调内零网络调用**）+ `NewEventStatusSink(ctx, pub, opts)` 启动独立发射 goroutine（逐条带短超时 ctx〔DefaultStatusPublishTimeout=3s〕、失败 fail-soft WARN、ctx 取消退出、`Done()` 可观测、`Dropped()` 满队丢弃计数〔首次+每 100 条采样 WARN〕）；QueueDepth 默认 256。javadoc 全文重写为与实现一致（队头约束论证：paho router Order=true 串行分发、handler 返回后才 Ack——同步 PUBLISH 在 Redis 半死时阻塞分发循环 ~9s、Ack 排队、9s×N 放大；异步面内事件为尽力面、pending 停机丢弃）。为什么现在对了：路由回调与 Redis 延迟彻底解耦——最坏情况下 OnStatus 耗时=一次 channel 入队（纳秒级），Redis 半死只拖慢发射 goroutine 自身且逐条有界，数据消息投递与 QoS1 Ack 不再排队 |
| `ingest/internal/mqttsource/status_event_contract_test.go` | **I1 红→绿** | 重写为异步语义 + 新增 3 项：**非阻塞契约测试**（慢发布者 publish 长阻塞下 3 条 OnStatus 累计 <100ms〔同步实现必红——首条即阻塞 ≥ publish 时长〕+ 放行后 3 条经发射 goroutine 送达）、满队丢弃计数（深度 1 + gate 编排确定性：worker 取走在途后 #2 入队 #3 丢弃，Dropped=1、OnStatus <50ms）、worker ctx 取消退出（Done 2s 内关闭）；既有 4 项语义断言保留（改经 eventually 收敛——异步面） |
| `ingest/cmd/ingestd/main.go` | **I1 接线 + M4** | 状态出口装配改 `mqttsource.NewEventStatusSink(pctx, rp, ...)`（ctx=pctx 停机即退）；M4：backfill=nil 时装配日志显式标注「回源不可用——INGEST_TDENGINE_BASE_URL 未配置」（原无条件声称「未命中回源」与装配事实不符）；头注释补「有界异步交接」 |
| `ingest/cmd/ingestd/wiring29_test.go` | **M1** | 攒批断言改**真护栏**：VALUES 元组数 = `")(" 连接子计数+1`（InsertPointSQL 产物 `VALUES (..)(..)` 形态有区分力；原 `Count("VALUES (") != Count(") VALUES (")+1 && Count("(") < 2` 对任何单条 INSERT 恒假=无效护栏）；上报改 W_M01 同点位双 ts + W_M02 单行——断言 W_M01 元组数 ≥2（多行同批合并端到端实证）、W_M02 =1（批内单行合法对照）。取舍说明：选「改真护栏」而非删除——端到端形态锁与 0073 单元锁（真形态权威）互补，且改红过程中真护栏反检出「装载探针行与 B1 批同 flush 窗口合并」的真实行为（元组数=3 非拟想的 2），断言收敛为 ≥2 防御性口径 |
| `ingest/internal/tdengine/rest.go` | **M2** | 包注释 TD-0053 教训清单删除「或 USE 多语句（EnsureSchema）」背书、补记修订轮实测（多语句信封真库静默空操作——USE 多语句路径已废止勿回退） |
| `ingest/internal/tdengine/tdengine.go` | **M2** | `UseSQL` 死代码删除（唯一生产调用方已随 EnsureSchema 重写移除），留注释登记废止理由（REST 无状态连接 USE 语境不可跨请求 + 多语句信封静默空操作——程序化建表走 QualifiedCreateStableSQLs 全限定形态） |
| `ingest/internal/tdengine/tdengine_contract_test.go` | **M2 关联** | 附录 B 逐字锁测试收敛为建库+四表 5 语句（USE 行仅存契约原文语境——交互式会话语义，代码不再产出；全限定形态断言在 rest_contract_test 既有项），四表 DDL 逐字锁零损失 |
| `ingest/internal/cache/redis.go` | **M3** | RESP 长度上限防御：bulk `>1 MiB` 拒绝、数组 `>1024` 拒绝、回复行 `>1 MiB` 拒绝（readReplyLine 改 ReadSlice 有界累积循环）——受损/恶意响应立即断连报错，不巨量分配（`make(n)` 无校验）不无界行缓冲。为什么现在对了：缓存失败不阻塞落库的 fail-soft 纪律要求解析面自身有界——畸形长度声明（如 `$9999999999`）原实现会挂到读超时（3s×N 拖慢 worker/flusher 面），现立即拒绝 |
| `ingest/internal/cache/redis_contract_test.go` | **M3 红→绿** | 新增 3 项（raw 回放假服务构造畸形响应）：超长 bulk 声明拒绝（红→绿：实现前挂 3s 读超时非「超上限」拒绝）、超长数组声明拒绝、4 MiB 超长回复行拒绝 |

M5（parseTsCell 时区残余）/ M6（EnsureSchema 建库超时裕度）/ M7（Drain written 计数口径）按裁决**不修**，维持原报告 §5.7/§5.5/§5.8 登记（M6 补注：runbook 登记已含「EnsureSchema 可用」，RESTConfig.Timeout 可调随 §5.5 runbook 承载）。

## R2. 验收命令原始输出（修订轮 HEAD `88dce89` 上终验）

### R2.1 `cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（EXIT=0）

```text
ACCEPT-R BUILD+VET EXIT=0          （build/vet 无任何输出，双零退出）

ok  	net.otlab/ingest/cmd/ingestd	1.669s
ok  	net.otlab/ingest/internal/cache	1.395s
ok  	net.otlab/ingest/internal/event	0.744s
ok  	net.otlab/ingest/internal/gwtoken	1.610s
ok  	net.otlab/ingest/internal/httpapi	1.406s
ok  	net.otlab/ingest/internal/message	0.463s
ok  	net.otlab/ingest/internal/mqttsource	1.353s
ok  	net.otlab/ingest/internal/pipeline	1.785s
ok  	net.otlab/ingest/internal/quality	0.439s
ok  	net.otlab/ingest/internal/snapshot	1.711s
ok  	net.otlab/ingest/internal/tdengine	1.812s
（11 包 ok，EXIT=0）
```

契约符合性全套（`go test ./... -run 'Contract' -v -count=1`）：

```text
--- PASS 计数 = 211，--- FAIL 计数 = 0
（205 → 211：新增 6 = I1 三项〔非阻塞/满队丢弃/worker 退出〕+ M3 三项〔bulk/数组/回复行超上限〕；
 既有 205 项零改动全绿——零回归）

修订轮新增项逐条（-run 'StatusSink|Oversized' -v）：
--- PASS: TestContractStatusSinkLWTEmitsGatewayStatusEvent (0.00s)
--- PASS: TestContractStatusSinkGracefulEmitsEventWithFlag (0.00s)
--- PASS: TestContractStatusSinkHeartbeatEmitsEvent (0.00s)
--- PASS: TestContractStatusSinkOnStatusNonBlockingUnderSlowPublisher (0.00s)
--- PASS: TestContractStatusSinkQueueFullDropsAndCounts (0.00s)
--- PASS: TestContractStatusSinkPublishFailureFailSoft (0.00s)
--- PASS: TestContractStatusSinkWorkerExitsOnContextCancel (0.00s)
--- PASS: TestContractRedisClientRejectsOversizedBulk (0.00s)
--- PASS: TestContractRedisClientRejectsOversizedArray (0.00s)
--- PASS: TestContractRedisClientRejectsOversizedLine (0.03s)

并发红线稳定性复跑（pipeline/mqttsource/cache -run 'Race' -count=3）：
ok  	net.otlab/ingest/internal/pipeline	1.264s
ok  	net.otlab/ingest/internal/mqttsource	1.203s [no tests to run]
ok  	net.otlab/ingest/internal/cache	1.740s [no tests to run]
```

I1 红留痕：修订前实现（同步 PUBLISH）下 `TestContractStatusSinkOnStatusNonBlockingUnderSlowPublisher` 必红——首条 OnStatus 阻塞在慢发布者 gate 上直至放行（累计耗时 > 100ms 阈值触发 Fatalf）；API 面（`NewEventStatusSink`/`EventStatusSinkOptions` 未定义）为第一重编译红，行为红为第二重（语义锁）。M3 红留痕：实现前超长 bulk 用例实测挂 3s 读超时（err=i/o timeout ≠「超上限」拒绝）。

### R2.2 范围核对

```text
git show --name-only --format="" 88dce89 → 9 文件全部 ingest/**（非 ingest 计数=0）
git status --porcelain -- ingest/ = 空（工作区干净）
主提交 cfed417（19 文件）+ 修订提交 88dce89（9 文件）= 本任务全部交付；均注明 TASK-0029
```

## R3. 修订轮一句话结论

**修订通过**——I1（有界异步交接：OnStatus 分发回调内零网络调用〔慢发布者 3 条 <100ms 契约锁定〕+ 满队丢弃计数可观测 + worker 有界发射/可观测退出 + javadoc 与实现一致）、M1-M4（真护栏元组计数〔反检出探针行合并真实行为〕/USE 背书清理+死代码删除/RESP 三面长度上限红转绿/装配日志与事实一致）全部以红→绿留痕落地；契约 211 PASS/0 FAIL（既有 205 零回归）、11 包 ok、Race 稳定、提交恰 9 文件全落 ingest/**；M5/M6/M7 按裁决登记 deferred。待 scoped 复评（只看 88dce89 修复 diff 与本发现）。
