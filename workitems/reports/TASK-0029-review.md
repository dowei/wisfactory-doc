# diff 评审：TASK-0029（攒批写 TDengine、Redis 点位缓存与新值事件）

- 评审对象：`cfed417`（BASE=`b4e877a`，单提交 19 文件，限 `ingest/**`；同区间 backend 13 文件属并行 0016 道提交 `082d694`，按 brief 不在本评审范围——已用 `git diff --name-only b4e877a..cfed417` 复核，19 ingest 文件与 diff 包逐一对应）
- 评审口径：`L1-execute.md` §13（与验收标准/brief 接口一致性 + 明显质量缺陷；分级 Critical/Important/Minor）
- 评审者立场：fresh，只依据 brief 必读清单与工作区源码；live 冒烟（真库/真 Redis 段）以报告 §2.2 原始输出为证据评估，未在本机复跑（环境依赖），其余结论均以工作区源码与独立验证运行为准
- 评审日期：2026-11-23

## 结论（先行）

**无 Critical；1 项 Important + 7 项 Minor。** 整体实现质量高：九项评审重点中八项全部核对通过，唯一 Important（I1：状态事件发射同步阻塞 MQTT 分发回调）是本 diff 新引入的同步队头网络调用，属 brief 交付面「sink 队头约束复核」的漏网面，修复面很小。按 §13，Important 应进 §12 退回路径（一轮修复）或由协调者裁决降级登记 deferred；仅 Minor 不足以阻断收尾。

## 1 独立验证记录（评审者自跑，非转抄报告）

| 命令 | 结果 |
| --- | --- |
| `cd ingest && go build ./... && go vet ./...` | 双零退出（无输出） |
| `go test ./...` | 11 包全部 ok |
| `go test ./... -run 'Contract' -v -count=1` | `--- PASS` 计数 = **205**（与报告一致） |
| `git grep "^func TestContract" b4e877a` / `cfed417` | 168 → 205（新增 37；唯一被改写既有测试=EnsureSchema，`git diff --stat -- '*_test.go'` 中唯一带删除行的既有测试文件即 `rest_contract_test.go`——「167 零改动 + 1 重写」口径与实物一致） |
| `git status --porcelain`（ingest/） | 空（工作区干净） |
| `git grep strings.Join`（非测试代码） | 零命中——多语句信封全仓仅 EnsureSchema 一处（已修正），**无遗漏面** |
| go.mod/go.sum | 不在 19 文件 diff 内——零新增依赖（paho/websocket/x-net/x-sync 为 0027 前既有） |

## 2 评审重点逐条核对

### 2.1 真执行器 —— 通过

- **攒批形态与语句锁**：INSERT 构造与分组下发均为 0073 既有件（`ingest/internal/tdengine/tdengine.go:256-282 InsertPointSQL`、`:370-407 Writer.WritePoints`，本 diff 零改动）：按（租户，子表，kind）分组、单条 INSERT 多 VALUES（`VALUES (ts,v,q)(ts,v,q)`）、组间字典序稳定。多 VALUES 语句形态由既有 `tdengine_contract_test.go`（"同一子表多行合并单条 INSERT 多 VALUES 元组"，BASE 既有、零改动）逐字锁定；wiring29 端到端测试再以真 run() 装配捕获 `INSERT INTO iot_group_5fa.W_5fM01 USING iot_group_5fa.st_point_num TAGS (...) VALUES (` 请求体。禁止逐条写成立（写路径无逐条分支）。
- **kind/表映射与附录 B 逐字**：TAGS 列序 `tenant_id, point_code, kind, instance_key, dept_id`、VALUES 列序 `ts, value, quality` 与契约附录 B（`resources/contract/tdengine-super-tables.md` §32-35）逐字一致；kind→stable 映射（INSTANT/CUMUL→st_point_num、STATE/SWITCH→st_point_state）与 §2.2 表一致；state 表 INT 值经 `formatValue` 最短浮点（"1"）写入 INT 列合法。live 断言 2（报告 §2.2.b）值/kind/标签三方核对实证。
- **子表首写自动创建**：`INSERT INTO db.child USING db.stable TAGS(...)` 形态即契约「子表由写入方首次写入自动创建（USING 超级表 TAGS(...)）」口径；标签取自快照（pipeline.go:494-503 标签纪律），live SHOW TABLES 4 子表 `_5f` 转义实证。
- **EnsureSchema 逐语句修正**：`rest.go:188-198` CREATE DATABASE 一请求 + 四表各一请求（`QualifiedCreateStableSQLs` 仅做库名全限定单一变换，DDL 主体复用附录 B 构造器逐字）；任一失败上返。契约测试重写（`TestContractRESTEnsureSchemaSplitsPerQualified`，5 请求/全限定/无 USE/TAGS 行逐字核对）红转绿留痕清晰，重写理由（多语句信封真库 code=0 affected=0 静默空操作）成立且是唯一能与 3.4.3.9 真库对话的形态。**遗漏面核查：全仓非测试代码无任何其他多语句拼接（`strings.Join` 零命中），无第二处同族隐患。** 契约原文未动（st_metric_value `period` 保留字发现按冻结纪律登记报告 §5.2，不擅改）。

### 2.2 攒批 —— 通过

- **双触发**：`Assemble` 默认 BatchSize=500 / FlushInterval=200ms（pipeline.go:257-262，FEAT-0009 口径）；`push` 水位触发 flushCh + `flusher` ticker（pipeline.go:544-577）。
- **整批重排受同上限约束**：`requeueFront`（pipeline.go:613-637）重排放回头部、combined 超 BufferCap 丢最旧计数——与 ADR-0014 FIFO 口径一致；单行重排 >3 转死信不回灌（maxWriteRequeues）。
- **写失败重试与 0027 幂等协同（零幻影机制核实）**：结构性成立——`flush`（pipeline.go:600-608）中 `updateCache`/`publishEvents` 只在 `WritePoints` 返回 nil 后执行，失败路径 requeue 后 return；重排行直接回缓冲、不再过 Submit/去重（pipeline.go:41-42 注释口径），不会被幂等键自吞。TDengine 同子表同 ts 重写为覆盖语义，部分组成功+整组重排的重试不产生重复行；缓存 SET 为覆盖写幂等。报告 §2.2.c live 三证（零幻影事件/缓存零污染/真库 count 不变）与本机制互洽。**执行者「零幻影零污染」声明核实为真。**

### 2.3 Redis 点位缓存 —— 通过（附 M3/M4/M5）

- 键 `iot:pt:{tenant}:{code}`（cache.go:47-52 定版）+ 值 JSON 恰 v/q/ts 三键（cache.go:22-26 json tag；契约测试逐字锁定）；写穿透=落库成功后更新（flush 成功路径内）；失败 fail-soft 上返计数不阻塞（updateCache 两分支均计 CacheFailures 继续）；未命中回源=懒 read-through（GetPoint miss → Backfill.LatestPointValue → 写穿 → 返回；源无数据转 ErrCacheMiss 不给零值）。
- **纯标准库 RESP2 客户端正确性**（cache/redis.go）：信封编码（`appendRESPCommand`）规范；回复解析覆盖 +,-,:,$,* 五类型、`$-1`=nil、整数/长度非法拒绝；错误回执上返并断连；惰性建连 AUTH→SELECT 序正确；单连接互斥串行 + 失败断连下次重连自愈（契约测试覆盖失败注入/重连/坏地址快速失败/管线单段断言）；`Pipelined` 一次写全批逐条读回，队头约束批量化落点成立。缺陷见 M3（长度无上限）、M4（装配日志失真）。

### 2.4 新值事件与 ADR-0010 —— 通过（I1 属发射通道形态缺陷，语义面通过）

- payload `{tenant,pointCode,ts,value,quality}` 恰五键（event.go:17-27 + 契约测试正反双向断言）、channel `iot:newvalue:{tenant}` 逐字；发射时机=写库成功后（flush 内，2.2 已核）；批量管线优先、单条回退、失败 fail-soft 计 EventFailures。
- **ADR-0010 兑现（TD-0049）**：`EventStatusSink`（mqttsource/status_event.go）对 LWT/graceful/心跳统一只发 `iot:gwstatus:{tenant}` 事件（payload 五键、graceful 字段逐字承载、ts=接收时刻并注明未冻结），**无任何状态存储写依赖**（契约测试结构性断言）——「接入服务不成为状态第二写入方」纪律成立；channel/payload 为发射侧拟定、0040s 对接定版，登记（报告 §4.4/§5.4）充分。评估：拟定形态与 newvalue 族命名同构，可接受。
- **无订阅者行为**：PUBLISH 回执 :0 不视为失败（fire-and-forget），event.go 包注释与报告 §5.4 均明确「无订阅者=丢弃」语义与演进位——事件仅为计算触发输入、权威值在 TDengine，可接受。

### 2.5 0027 遗留两项承接 —— 通过（M7）

- **SIGTERM 排空**：`Pipeline.Drain`（pipeline.go:713-728）同步循环 flush 至缓冲空或 ctx 超时；写失败走与运行期一致的重排上限（持续失败 4 次 flush 内收敛转死信，契约测试 `DrainExhaustsRetriesToDeadLetter` 验证）；超时滞留 WARN 不阻塞退出（main.go:376-383）；live §2.2.d 真实 SIGTERM 证据（缓冲清零落库 4 行+真库核对）。边界（MQTT 通道积压不排空、QoS1 重投兜底）在报告 §5.8 登记准确。
- **sink 队头约束复核**：数据面两处批量化落地（缓存 SetPoints 按租户管线、事件 PublishNewValues 管线）——已复核通过；**漏网面见 I1**（状态事件 sink 同步网络调用在 broker 分发回调内）。

### 2.6 口径固化三项 —— 通过（无代码变更核实）

diff 未触碰 `internal/quality/**`、CorrectionTagger 与 processOne 拒绝路径（19 文件清单核实）；运行期语义与三项裁决逐条一致：①纠偏标记=CorrectionTagger 独立计数+采样 WARN（main.go:264）；②被拒点不标去重键、每次到达计入原因码（pipeline.go:456 注释+实现未变）；③纠偏计数=TimeCorrected 每动作加一（pipeline.go:466）。README 已登记裁决日期。

### 2.7 清理纪律 —— 通过

scratch 库 `iot_verify29` DROP + `SHOW DATABASES` 终态与开工一致（报告 §2.2.e）；Redis db0 租户键 DEL 归零、db15 scan=0。测试代码侧复核：`redis_live_test.go`（db15、仅 DEL 自建键）、`redis_backfill_live_test.go`（db15 + Delete 测后清理）、`wiring29_test.go` live 段（db15、t.Cleanup 仅 DEL `iot:pt:group_a:W_M01` 单键）——**清理面均限自建键/自建库，无任何触碰共享 iot_default/db0 业务键空间的语句**。

### 2.8 零改动替换 —— 通过

契约测试 168→205、唯一重写=EnsureSchema（live 发现驱动，§2.1 已核重写合理性）；`git diff --stat -- '*_test.go'` 唯一带删除行的既有文件即 `rest_contract_test.go`；go.mod/go.sum 零改动（零新增依赖）；既有假执行器/内存缓存保留为降级形态（装配面不变）。

### 2.9 并发模型 —— 基本通过（I1/M7）

- sink 调用形态：数据面单 flusher 串行，Redis 客户端互斥串行单连接，批量管线一次往返——无队头放大。
- Pub/Sub 连接生命周期：cache/event 各持独立惰性连接，纯同步 net.Conn 无 goroutine；无订阅侧常驻 goroutine，**无泄漏面**；进程退出随连接关闭（未显式 Close——进程退出语义下无害）。
- SIGTERM 与在途写：HTTP Shutdown(5s)→pcancel→Drain 顺序正确；flusher 与 Drain 共存期缓冲操作由 bufMu 串行化（Drain 直读 buf 长度持锁）。残留计数口径瑕疵见 M7。
- 附：paho 分发模型核实——`paho.mqtt.golang@v1.5.1 router.go matchAndDispatch`，默认 `Order=true`（ingest `paho.go` 未设 SetOrder(false)）为**单 goroutine 串行调用 handler、handler 返回后才 Ack**——这是 I1 成立的前提，已实证。

## 3 发现清单

### Critical（0 项）

无。

### Important（1 项）

- **I1** `ingest/internal/mqttsource/status_event.go:43`（配合 `internal/mqttsource/mqttsource.go:308-315` route 同步调用 + `paho.go` 未设 `SetOrder(false)`）——`EventStatusSink.OnStatus` 在 paho 有序分发回调内**同步**执行 Redis PUBLISH（`context.Background()` 无自身超时，仅靠客户端 3s 命令超时兜底）：Redis 半死（TCP 可达不响应）时单条状态消息最多阻塞分发循环 ~9s（建连 3s+握手 3s+命令 3s），期间同连接后续**数据消息的投递与 QoS1 Ack 全部排队**（router order=true 串行、handler 返回后才 Ack）；多网关心跳叠加时停滞按 9s×N 放大，broker inflight 填满后靠重投+幂等兜底。与该文件注释「发射失败 fail-soft……不阻塞路由」的口径不符，且本批交付面明确含「sink 队头约束复核（如为同步队头瓶颈则批量化）」（brief 决策 4②）——数据面两 sink 已批量化，此处为唯一未复核的同步网络 sink。建议修复面很小：OnStatus 注入短超时 ctx（如 500ms）或改为有界异步交接（路由内只入队、独立 goroutine 发射）。

### Minor（7 项）

- **M1** `ingest/cmd/ingestd/wiring29_test.go:294` —— 多 VALUES 攒批断言 `Count("VALUES (") != Count(") VALUES (")+1 && Count("(") < 2` 对任何 INSERT 均为永假式（`InsertPointSQL` 产出 `VALUES (..)(..)` 形态，"VALUES (" 恒 1 次、"(" 恒 ≥2），是无效护栏；真锁在 0073 既有契约测试（零改动），建议删除或改为有效计数断言。
- **M2** `ingest/internal/tdengine/rest.go:23` —— 包头 TD-0053 教训注释仍背书「或 USE 多语句（EnsureSchema）」，与 §4.1 live 修正（多语句信封=静默空操作、已废弃）自相矛盾；连带 `tdengine.go:200 UseSQL` 成为死代码（唯一调用方已随重写移除）——后续维护者可能据此回退出同族缺陷。
- **M3** `ingest/internal/cache/redis.go:236`（另 `readReplyLine` 行读无界）—— bulk/数组长度直接 `make([]byte, n+2)` 无上限校验：受损/恶意 Redis 响应可触发巨量分配甚至 panic，与「缓存失败不阻塞落库」的 fail-soft 纪律相比防御面偏弱（可信内网前提下缓解），建议加长度上限。
- **M4** `ingest/cmd/ingestd/main.go:255` —— Redis 装配日志无条件声称「未命中回源」，但 TDengine 未配置（exec 非 RESTClient）时 backfill=nil、回源实际不可用——日志口径与装配事实不符。
- **M5** `ingest/internal/tdengine/rest.go:314-326` —— `parseTsCell` 对无时区 ISO 串按 UTC 兜底解析：上游 taosAdapter 若输出服务器本地时区形态，回源 ts 将整体偏移；live 实测形态为 RFC3339Z（安全路径），兜底仅在上游版本变更时可达——报告 §5.7 已登记，复核认可，维持为已知残余风险（上游升级时须复验）。
- **M6** `ingest/internal/tdengine/rest.go:34` —— EnsureSchema 的 CREATE DATABASE（brief 口径 20-25s 慢特性、live 实测 16.8s）对默认 30s 读超时裕度极小，慢盘首建可超时失败；报告 §5.5 runbook 登记宜补注 `RESTConfig.Timeout` 可调。
- **M7** `ingest/internal/pipeline/pipeline.go:713-728` + `cmd/ingestd/main.go:372-377` —— pcancel 后未等待 flusher goroutine 退出即 Drain，Drain 的 `written` 按 WrittenPoints 全局增量计，可把 flusher 尾写计入「本次落库」——仅影响停机日志计数（缓冲安全由 bufMu 保证，无正确性影响）。

## 4 处置建议

- I1 按 §13 进退回路径：小修（短超时 ctx 或有界异步发射 + 对应契约测试）后 scoped 复评（只看修复 diff 与本发现）。
- M1-M7 记 TASK `任务结果` 的 deferred（收尾进 LOG），不进修订轮；其中 M2 建议随 I1 顺手修正（同文件一处注释+删除死函数，零风险）。
