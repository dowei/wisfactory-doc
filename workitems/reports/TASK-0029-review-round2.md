# scoped 复评 round2：TASK-0029（I1 + M1-M4 打包修复）

- 复评对象：`88dce89`（修订轮，cfed417..88dce89，单提交 9 文件，限 `ingest/**`——diff 包 `TASK-0029-diff-round2.txt` 已与本仓 `git diff cfed417..88dce89` 逐字节比对一致，9 文件全落 ingest/**，越界 0，工作区干净）
- 复评基线：`TASK-0029-review.md` I1 + M1-M4（M5/M6/M7 裁决 deferred，不复评）
- 复评口径：`L1-execute.md` §13「复评是 scoped 的——只看修复 diff 与原发现，不重开全场」；本轮为终审（修订上限 1 退回 + 1 复评）
- 复评者立场：fresh，只依据复评 brief 必读清单与工作区源码（含 paho v1.5.1 模块源码独立取证）；不向协调者追问
- 复评日期：2026-11-23

## 0 独立验证记录（复评者自跑，非转抄修订轮报告）

| 命令 | 结果 |
| --- | --- |
| `git diff cfed417..88dce89 > /tmp/actual.txt` 与 diff 包 diff | 逐字节一致（DIFF PACKAGE MATCHES） |
| `git diff --name-only cfed417..88dce89` | 9 文件，全部 `ingest/**`，非 ingest 计数 = 0 |
| `git status --porcelain -- ingest/` | 空 |
| `cd ingest && go build ./... && go vet ./...` | 双零退出（EXIT=0） |
| `go test ./... -count=1` | 11 包全部 ok |
| `go test ./... -run 'Contract' -v -count=1` | `--- PASS` = **211**，`--- FAIL` = 0（与报告 205→211 一致；`git grep "^func TestContract"` cfed417→88dce89 = 205→211 个函数，+6 新增、0 删除） |
| wiring 测试压测 `-run TestIngestdRealExecutorCacheEventsWiring -count=200` | 全过（连同首轮 50 次，累计 250 次零失败） |
| sink 契约压测 `-run 'TestContractStatusSink' -count=30` / M3 `-run 'Oversized' -count=20` | 全过 |
| `go test ./internal/pipeline/ ./internal/mqttsource/ ./internal/cache/ -run 'Race' -count=3` | ok |
| paho v1.5.1 源码取证（模块缓存 `router.go matchAndDispatch`） | order=true：主分发 goroutine 同步串行调用 handler（`handler(client, m)` 后才 `m.Ack()`，router.go:196-202）——I1 前提与取舍论证独立复核成立 |
| `git grep UseSQL`（88dce89） | 仅 2 处注释引用（均为「已删除」留痕），零代码引用；cfed417 上唯一引用方即契约测试自身——「死代码」定性准确 |

## 1 三问逐条回答

### 1.1 问一：I1 是否真修复 —— **通过**

**结论：真修复。有界异步交接落地完整，分发回调内零网络调用，非阻塞契约测试真实锁住语义，取舍论证经独立取证成立。**

- **OnStatus 非阻塞 + 零网络调用**（`ingest/internal/mqttsource/status_event.go:96-120`）：OnStatus 只做三件事——构造事件、`select { case s.ch <- ev: default: 丢弃+计数 }` 非阻塞入队、本地日志；无任何网络调用（出处：实现 + `Consumer.route` 唯一调用点 `mqttsource.go:312`）。一句话说明：paho order=true 单分发 goroutine 内的最坏耗时=一次 channel 入队，I1 的队头阻塞路径被彻底移除。
- **发射 goroutine 独立、逐条短超时、ctx 取消退出**（status_event.go:63-140）：`NewEventStatusSink(ctx,...)` 启动单 worker（`loop`），每条 `context.WithTimeout(Background, 3s)` 发射、失败 fail-soft WARN、`defer close(s.done)`。关键核实：**3s 超时是真实生效的而非装饰**——`cache/redis.go:134-143`（ensureConnLocked 将拨号超时钳制到 ctx 剩余 deadline）与 `:163-172`（execLocked `SetDeadline(ctx.Deadline())`）表明拨号+握手+命令全被同一绝对 deadline 封顶，单条发射墙钟 ≤3s；main.go:297 接线 `NewEventStatusSink(pctx, rp, ...)`，停机序列 pcancel（main.go:381）后 worker 经 select 退出，无泄漏（在途发射被 deadline 封顶，Done 经 defer 必关）。一句话说明：worker 生命周期有界且可观测，pending 停机丢弃与 javadoc「尽力面」声明一致。
- **非阻塞契约测试真实锁语义**（`status_event_contract_test.go:123-139`）：慢发布者 gate（publish 阻塞直至 close(block)）下 3 条 OnStatus 累计 <100ms + 放行后 3 条经 worker 送达——同步实现在本测会死锁在首条 OnStatus 上（gate 放行代码位于耗时断言之后），必红，故对异步/同步形态有区分力。配套满队丢弃测试（:143-162，用 entered channel 确定性编排 worker 取走在途后 #2 占满、#3 丢弃，Dropped()==1 且 <50ms）与 worker 退出测试（:181-193）。一句话说明：三个新契约测试从「非阻塞」「有界丢弃」「生命周期」三面锁住 I1 语义，红→绿留痕可信。
- **丢弃计数与可观测性**（status_event.go:88-92,107-111）：`Dropped()` 原子计数 + 首次/每 100 条采样 WARN（与死信/纠偏采样同族）；`Done()` 供停机面观测，两者均有测试消费、非死代码。一句话说明：尽力面丢弃不再是静默行为，运维监控挂点齐备。
- **取舍论证成立**（status_event.go:8-15 javadoc）：paho v1.5.1 router.go 实证（见 §0）order=true 下 handler 返回后才 Ack——若改「OnStatus 内短超时 ctx 同步发射」，每次状态消息仍占住分发 goroutine 一个超时周期并推迟同连接数据消息 Ack；异步交接把路由与 Redis 延迟彻底解耦，论证成立。一句话说明：选异步交接而非短超时 ctx 是对的，javadoc 论证与源码实证一致。

### 1.2 问二：M1-M4 是否真修复 —— **通过（M1 附 Minor N1 备注）**

- **M1 攒批断言改真护栏**（`cmd/ingestd/wiring29_test.go:295-313`）：原断言独立复核确为恒假式——`Count("VALUES (") != Count(") VALUES (")+1` 对 `VALUES (..)(..)` 形态恒为 `1 != 1`＝false，护栏从不触发（连逐条写也检不出）；新断言按 `Count(")(")+1` 计元组数，W_M01 ≥2（多行同批合并实证）、W_M02 ==1（批内单行合法对照）。修订轮反检出「装载探针行与 B1 批同 flush 窗口合并（元组=3 非拟想 2）」的行为留痕清晰（注释 :295-300 完整记录恒假式归因与 ≥2 防御口径取舍）。一句话说明：从无效护栏变成有区分力的真护栏，端到端形态锁与 0073 单元锁互补，且 250 次压测零失败。
- **M2 背书删除 + 死代码清理 + 逐字锁零损失**（`rest.go:22-25`、`tdengine.go:199-202`、`tdengine_contract_test.go`）：包头注释不再背书 USE 多语句、改为「已废止勿回退」留痕；UseSQL 删除（§0 已核：删除前唯一引用方即契约测试自身，纯死代码）；契约锁测试收敛为建库+四表 5 语句，**四表 CREATE STABLE 语句逐字未动**（diff 仅两侧对称删 USE 行），全限定形态断言仍在 rest_contract_test 既有项。一句话说明：M2 三件事全落地，回退诱导面清除且契约锁零损失。
- **M3 RESP 三重上限红→绿**（`cache/redis.go:44-56,245-247,258-260,275-296`）：bulk >1MiB 拒绝（在 `n<-1` 非法与 `$-1`=nil 处理之后，`make(n+2)` 有界）、数组 >1024 拒绝、回复行 >1MiB 经 ReadSlice 有界累积循环拒绝；错误上返后 `Do/Pipelined` 走 `dropConnLocked`，「断连」口径与实现一致。三项新契约测试（raw 回放假服务构造 `$9999999999`/`*9999999999`/4MiB 行）全过，红留痕（原实现挂 3s 读超时非「超上限」拒绝）与机制吻合。一句话说明：三个解析面（bulk/数组/行）全部有界，受损/恶意响应不再触发巨量分配或无界缓冲，且与「缓存失败不阻塞落库」纪律自洽。
- **M4 日志措辞一致**（`cmd/ingestd/main.go:253-275`）：按 `backfill != nil` 分支——真执行器在位时报「未命中回源 TDengine 最近值」、未配置时报「回源不可用——INGEST_TDENGINE_BASE_URL 未配置」；`backfill` 为纯 nil 接口（仅在 `*RESTClient` 分支赋值，无 typed-nil 陷阱），日志与装配事实两分支均一致。连带 OnStatus graceful 日志措辞同步改为「事件交接发射」（不再声称「已发射」，与异步语义一致）。一句话说明：装配/语义日志不再超前于事实。

### 1.3 问三：修复 diff 是否引入新 Critical/Important —— **无**

- **顺序语义**：OnStatus 唯一调用点在 paho order=true 的单分发 goroutine（§0 取证）→ 单生产者 FIFO 入队 → 单 worker FIFO 发射，**状态事件发射顺序=broker 投递顺序，严格有序性未降级**；数据面与状态面本就不同通道，异步化不产生跨面乱序。ADR-0010（TD-0049）消费方语义核查：事件只是状态判定的「输入」、最终状态经优先级链裁决、接入服务不直写状态 Redis——payload/channel/纪律面零改动，仅发射时机改为异步尽力面且已 javadoc+报告双重登记，消费方语义不受影响。
- **goroutine 生命周期**：每 sink 恰一个 worker；ctx 取消后 select 必然收敛退出（Done 常备就绪），在途发射被 3s deadline 封顶；worker 退出后 OnStatus 仍安全（入队→满后丢弃+计数，无 panic 无阻塞）；main 侧 pctx 覆盖 sink 与 consumer/pipeline 生命周期，无泄漏面。
- **丢弃策略**：满队丢弃只在 Redis「半死」持续期可达——256 深度 × 单条 3s ≈ 12.8 分钟积压才开始丢；Redis 硬不可达时 publish 快速失败反而零丢弃；丢失窗口内首次+每 100 条 WARN 可观测。最坏情形（约 13 分钟半死期间末尾的 LWT 离线事件被丢）是已登记的尽力面代价，与 ADR-0010「事件=判定输入非权威写」口径自洽，且优于被 I1 修掉的「阻塞分发循环 9s×N」形态。
- **契约测试 205→211**：实测 211 PASS / 0 FAIL，+6 新增、0 删除。既有项严格核对：5 个既有 Contract 测试体被改（4 个状态 sink 测试为异步语义必然适配、语义断言保留改经 eventually 收敛；1 个 DDL 逐字锁测试因 UseSQL 删除被迫同步去 USE 行，四表逐字锁保留），加 1 个 wiring 测试（非 Contract 口径）。修订轮报告 R1 表已如实记载上述适配，但顶部「既有 205 项零改动全绿」一行按字面不精确——见 Minor N2。

## 2 新发现分级

### Critical（0 项）

无。

### Important（0 项）

无。

### Minor（2 项）

- **N1** `ingest/cmd/ingestd/wiring29_test.go:295-313` —— M1 断言的「合并确定性」属实证而非结构保证：探针成功行入缓冲与 B1 行入缓冲之间隔一次 HTTP 往返（本地 httptest 亚毫秒~毫秒级），若 200ms flush tick 恰落在该间隙，探针行将被**单独**刷出一条元组数=1 的 W_M01 INSERT，被捕获后 `tuples < 2` 即误报——`≥2` 防御口径只吸收「探针并入 B1」的不确定性，吸收不了「探针被单独刷出」。压测 250 次零失败（风险量级 ≲1%/次），不阻断。建议（登记不修亦可）：断言只针对承载 B1 值（42.5/43.5）的那条 INSERT，或按 batchId 时点排除探针期语句。
- **N2** `TASK-0029-report.md` 修订轮段顶部——「契约测试 205 → 211，既有 205 项零改动全绿」按字面与实物不符：实为「200 项零改动 + 5 项语义保留式适配（4 sink 异步收敛 + 1 DDL 锁去 USE 行）+ 6 新增」；R1 修订清单表内已如实记载两处适配，仅汇总行措辞以偏概全。属报告口径精度问题，零回归事实不受影响。

## 3 复评结论

**通过。** 三问全过（I1 真修复且取舍论证经 paho 源码独立取证成立；M1-M4 全部真修复；修复 diff 未引入新 Critical/Important），新发现仅 2 项 Minor（N1 测试防御口径的残余 flake 窗口、N2 报告汇总行措辞），按 §13 记 deferred 不进修订轮；M5/M6/M7 维持原裁决 deferred。契约 211 PASS/0 FAIL（自测实证）、11 包 ok、越界 0、diff 包与仓库逐字节一致——TASK-0029 修订轮收口条件满足。
