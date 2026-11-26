# TASK-0028 报告：校验流水线、死信计数与质量判定（Go 道第二环）

- 执行者：TASK-0028 执行（brief 驱动，纯 Go 批次：只改 ingest/**）
- BASE：`cebf17a`（开工时主仓 main HEAD，2026-11-23；同期 0079/0080 道后继提交为 backend/**，与本任务无关）
- 交付提交：`8a837aa`（主提交，17 文件全部 ingest/**）+ `b4e877a`（修订：并发红线测试收敛后复核取最新快照，1 文件）——两提交均注明 TASK-0028
- 执行日期：2026-11-23
- 验收结论：**3/3 验收命令全部通过**（原始输出见 §2；live 冒烟全序列含纠偏打标/死信采样告警/幂等重报实证）

## 1. 改动文件清单（18 个，全部落 ingest/**；`git show --name-only 8a837aa b4e877a` 非 ingest 文件数=0）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `ingest/internal/quality/range.go` | 新增 | **量程质量判定核心（裁决 1）**：`MergeRange(gatewayQuality, value, min*, max*)`——value>max→OverMax(1)、value<min→UnderMin(2)（超限类接入覆盖写，§3.2 p24 口径）；边界值（==min/==max）不算越界；min/max 缺失跳过该侧判定（双侧缺失整跳过、网关码透传）；越域网关码（非 0-10）防御收敛为 Good（CONV-02 唯一存储口径）；只产质量码不碰数值——原值保留、不丢弃、不假 0；包注释登记 ADR-0009 TD-0048 边界（人工补录走管理面，ingest 不产生 MANUAL 码） |
| `ingest/internal/quality/range_contract_test.go` | 新增 | 契约测试 5 项：超上限/超下限覆盖写（含网关非 0 码被覆盖）、边界包含、缺失透传+单边判定、越域码收敛 |
| `ingest/internal/pipeline/timepolicy.go` | 新增 | **时间三分裁决（裁决 3）**：`timePolicy.judge(now, ts)` 三路径——ts>now+FutureReject（默认 5m）→ts_overflow、ts<now-MaxAge（默认 30 天）→ts_too_old、now<ts≤上界→纠偏为接收时间并打标、其余（含边界 ==now±阈值）→原始 ts 落库；`CorrectionTagger` 纠偏打标（独立计数器 + 结构化 WARN 采样 `pipeline: [ts-corrected] tenant= pointCode= origTs= correctedTs= count=`，首次+每 N 条，不改 CONV-02 码域）；javadoc 登记三路径与边界及依据（§1.9A 原始 ts 不可被纠偏破坏） |
| `ingest/internal/pipeline/deadletter.go` | 新增 | **死信统一通道（裁决 2）**：原因码扩展 `ts_overflow`/`ts_too_old`/`format_invalid`（与 HTTP report 拒绝明细原因码同源对齐）；`DeadLetterCounter` 实现 DeadLetterSink——按租户+原因隔离计数（进程内有界 map，key=tenant+reason，FIFO 淘汰最旧键对齐 ADR-0014；Redis 化登记演进位）、WARN 采样告警（固定格式 `[dead-letter] tenant= reason= count= pointCode= batchId=`，首次+每 N 条，N 默认 100）、最近明细有界环形缓冲（默认 128，`Recent(n)` 排查读口）；可链式 Next 下游 |
| `ingest/internal/pipeline/pipeline.go` | 修改 | **四步校验重接（裁决 1/3/4）**：processOne 逐点流改为 查快照/禁用 → 格式（点位级非有限值防御→format_invalid）→ 量程（judge）→ 时间（三分裁决：拒绝→死信+拒绝明细不落库；纠偏→rowTs=接收时间+打标+TimeCorrected 计数）→ 去重（`MarkIfSeen(pointCode, 原始ts)`——**去重靠后、键取消息原始 ts**：纠偏只改落库 ts 不改幂等键）→ kind → 行构造（原值+裁决后 ts）；Config 增 TSFutureReject/TSMaxAge（0 取默认、负值装配报错）；Deps 增 `TimeTag func(tenant, pointCode, origTs, correctedTs)`（nil 安全）与 `Now func() time.Time`（nil→time.Now）；**Assemble 缺省挂点换 `RangeQualityJudge` 真实现**（PassthroughJudge 保留直通形态）、缺省 Dead 换 DeadLetterCounter；Stats 增 `TimeCorrected`；包注释登记四步校验口径、去重位置口径、时间语义三分与 ADR-0009 边界注记；修正 0027 留置的过时 javadoc（去重 TTL 真实现已于 0027 交付） |
| `ingest/internal/pipeline/pipeline_quality_range_contract_test.go` | 新增 | 量程流水线契约 5 项 + 验证 harness（valPipe：Submit 同步面+冻结时钟+TimeTag/Dead 假记录）：越限原值保留（SQL `VALUES (ts, 150, 1)`/`(ts, -10, 2)` 断言 + 缓存 V=-10 Q=2 无假 0）、合并规则（网关 3 范围内保留/越限覆盖 1/无量程透传 4）、单边量程、**缺省挂点=真实现**、边界包含 |
| `ingest/internal/pipeline/pipeline_time_contract_test.go` | 新增 | 时间三分契约 10 项：窄带纠偏（落库 ts=接收时间、原始 ts 不落库、TimeCorrected=1、Tag 回调字段全对）、上界 ==now+5m 纠偏/+5m+1ms 拒绝、ts_overflow/ts_too_old 拒绝不落库（明细携带原始 ts）、补传豁免原始 ts 落库（-10d）、MaxAge 边界（==30d 豁免/-1ms 拒绝）、ts==now 保留、**阈值配置化**（FutureReject=1m/MaxAge=2h 改判）、**去重键取原始 ts**（未来漂移点跨批重报恰一落库）、格式步先于时间步（NaN+陈旧 ts→format_invalid 非遮蔽） |
| `ingest/internal/pipeline/pipeline_deadletter_contract_test.go` | 新增 | 死信通道契约 9 项：租户隔离、原因分列、采样告警（N=100：count=1 与 count=100 两条，格式含租户/原因/计数/固定前缀）、N 可配置（N=3：1/3/6）、环形缓冲（容量 3 收 5 条→Recent=[P3,P4,P5]）、有界 map FIFO 淘汰（淘汰后重计）、链式 Next、原因码字面值、**流水线端到端**（unknown_code 经计数器：Count/Recent/WARN 全对） |
| `ingest/internal/pipeline/pipeline_order_contract_test.go` | 新增 | 顺序与去重位置契约 4 项（裁决 4）：同批三分流（越限受理+越旧拒绝+同批重复点吸收，拒绝明细顺序与 points 一致）、**同 batchId 重报零新增死信**（重复上报不产生重复死信计数）、跨 batchId 重报已受理点零死信恰一落库、时间拒绝明细携带原始 ts |
| `ingest/cmd/ingestd/main.go` | 修改 | env 接线：`INGEST_TS_FUTURE_REJECT`（默认 5m）/`INGEST_TS_MAX_AGE`（默认 720h=30 天）/`INGEST_DEADLETTER_WARN_EVERY`（默认 100）；run() 装配 `Judge: RangeQualityJudge`、`Dead: DeadLetterCounter{WarnEvery:env}`、`TimeTag: CorrectionTagger.Tag`、`pipeCfg.TSFutureReject/TSMaxAge`；头注释登记新键位与组件口径 + ADR-0009 边界注记 |
| `ingest/cmd/ingestd/main_test.go` | 修改 | 配置默认值/env 断言扩展（5m/30d/100；90s/72h/7）；新增 **`TestIngestdValidationPipelineWiring`** 端到端装配测试：假 backend（httptest 起 credential/pull+config/snapshot）→ 真 run() 启动 → 快照装载探针 → 越限值受理（只打码不拒绝）/未来窄带受理（纠偏）/+10m→ts_overflow/-40d→ts_too_old/未知编码→unknown_code 全断言 → 优雅停机 |
| `ingest/internal/pipeline/pipeline_submit_race_contract_test.go` | 修改 | 修订 1 处（修订提交 b4e877a）：`TestContractSubmitRaceSamePointTsExactlyOnce` 收敛后复核由「收敛前快照 st」改取最新 `Stats()`——原写法在 -count=10 下偶发 26/30 误报（stale 竞态读，见 §4.4），恰一写出语义三重锁定不变 |
| `ingest/internal/pipeline/pipeline_contract_test.go` | 修改 | harness 注入冻结时钟 `Now: clockAt(1795212000000)`（契约样例 sentAt——硬编码 ts 恒落补传豁免带，消除墙钟漂移定时炸弹，见 §4.2）；`TestContractBufferCapDropsOldestAndCounts` 显式放开时间阈值（该测专查攒批缓冲语义与时间裁决正交） |
| `ingest/internal/pipeline/pipeline_submit_ttl_race_contract_test.go` | 修改 | ttlHarness 注入冻结时钟（同上口径） |
| `ingest/internal/mqttsource/twopath_contract_test.go` | 修改 | 镜像流水线注入冻结时钟（两路一致性与时间裁决共存） |
| `ingest/internal/httpapi/report_contract_test.go` | 修改 | realPipelineFixture 注入冻结时钟 |
| `ingest/internal/tdengine/rest_contract_test.go` | 修改 | fail-soft 经流水线测试注入冻结时钟 |
| `ingest/README.md` | 修改 | 目录描述同步（quality 增量程合并规则、pipeline 增四步校验/死信统一通道/时间三分裁决）；新增「入库前校验四步」「死信统一通道」「人工补录不走 ingest（ADR-0009）」段落 |

依赖纪律：零新增第三方依赖。契约测试 **135 → 168**（新增 33，全部 `Contract` 命名）；既有 135 项零语义改动全绿（零回归，见 §2.1）。

## 2. 验收命令原始输出

### 2.1 验收 1：`cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（EXIT=0；提交 HEAD `b4e877a` 上终验）

```text
ACCEPT-1 BUILD+VET EXIT=0          （build/vet 无任何输出，双零退出）

ok  	net.otlab/ingest/cmd/ingestd	1.659s
ok  	net.otlab/ingest/internal/cache	0.439s
ok  	net.otlab/ingest/internal/gwtoken	1.620s
ok  	net.otlab/ingest/internal/httpapi	1.511s
ok  	net.otlab/ingest/internal/message	0.480s
ok  	net.otlab/ingest/internal/mqttsource	1.152s
ok  	net.otlab/ingest/internal/pipeline	1.632s
ok  	net.otlab/ingest/internal/quality	0.387s
ok  	net.otlab/ingest/internal/snapshot	1.620s
ok  	net.otlab/ingest/internal/tdengine	1.603s
ACCEPT-1 TEST EXIT=0
```

契约符合性全套（`go test ./... -run 'Contract' -v -count=1`）：**`--- PASS` 168 行、`--- FAIL` 0 行**（既有 135 零回归 + 本次新增 33）。分包：pipeline 60、snapshot 34、httpapi 18、gwtoken 16、mqttsource 8、message/quality/cache/tdengine 32（quality 含新增 5）。

并发红线稳定性复跑：

```text
go test ./internal/pipeline/ ./internal/mqttsource/ -run 'Race' -count=3
ok  	net.otlab/ingest/internal/pipeline	1.115s
ok  	net.otlab/ingest/internal/mqttsource	1.075s [no tests to run]

go test ./internal/pipeline/ -run 'Race' -count=10（修复后复跑两轮）
ok  	net.otlab/ingest/internal/pipeline	1.306s
ok  	net.otlab/ingest/internal/pipeline	1.196s
```

（初轮 -count=10 曾出现一次偶发 FAIL——经根因定位为**测试侧 stale 快照竞态读**、非流水线行为缺陷，红转绿修正留痕见 §4.4；修正后 -count=10 两轮稳定。）

### 2.2 验收 2：live 冒烟全序列（8098/8090 起前探测均空闲；每步原始输出）

**环境处置**：探测 8098/8090 全空闲、WSL wisfactory-mysql/wisfactory-redis 在跑（与协调者 2026-11-23 实测一致）；backend dev 以**既有构建产物** `backend/ot-admin/target/ot-admin-1.0.0.jar`（2026-11-23 03:14 构建，含 0079/0080 道代码）`java -jar` 起 dev 实例（零编译零 mvn、对 backend 源码零写入——同 0027 报告 §4.1 处置先例），`OT_GATEWAY_INTERNAL_TOKEN=tok-0028-smoke`，启动日志 `Started OtAdminApplication`×1、启动段 `grep -c " ERROR "`=**0**。

**步骤 0：凭据与快照（0027 冒烟遗留数据复用）**

```text
POST /auth/login/test {"username":"admin","password":"admin123","tenantCode":"default"} → token ok
GET /internal/v1/credential/pull?channelId=21&credType=HTTP_SECRET（X-Internal-Token: tok-0028-smoke）
→ {"code":200,"data":{"channelId":21,"ciphertext":"{v1}P5+Wy8...rrS0=","credType":"HTTP_SECRET","gatewayStatus":1,"keyVersion":"1"}}
—— gatewayStatus:1 已随响应携带（0079 道交付后形态，网关状态已知路径真联调）
快照点位核对：backend/ot-admin/data/snapshots/1/1-9111-5.json 中 SMOKE0026_P1
  = {"minValue":5.0,"maxValue":100.0,"dataType":1,"status":1}（量程冒烟直接可用）
```

**步骤 1：ingestd 起（8090）→ 快照真拉取（version=5 哈希 415c24f6... 与 0027 报告锚点一致）**

```text
2026/11/23 03:25:20 ingestd: MQTT 未配置（INGEST_MQTT_BROKER 空）——纯 HTTP 上报形态
2026/11/23 03:25:20 ingestd: 快照对账已装配 scope=1:9111 tenant="default" interval=10s
2026/11/23 03:25:20 ingestd: HTTP 接入面已启动 [::]:8090（report/auth-mqtt/healthz）
2026/11/23 03:25:20 snapshot: 对账快照已替换 version=5 hash=415c24f63d45fd7b410949d72156854a0012a0a14172da5e85185200738da473
curl http://127.0.0.1:8090/healthz → ok
（env：INGEST_TS_FUTURE_REJECT=5m、INGEST_TS_MAX_AGE=720h、INGEST_DEADLETTER_WARN_EVERY=2、其余同 0027 冒烟）
```

**步骤 2：/ingest/v1/report 混合批分流（X-Gateway-Token=credential/pull 密文）**

```text
=== B1 范围内+双边界（v50 / v150 超上限 / v1 越下限 / v100==max / v5==min）===
{"accepted":5,"rejected":0,"rejections":[],"duplicate":false}
—— 量程越界只打码不拒绝（accepted=5）；边界值 v100/v5 不算越界照常受理

=== B2 未来漂移 ts(+2m，窄带纠偏) ===
{"accepted":1,"rejected":0,"rejections":[],"duplicate":false}

=== B3 越未来界 ts(+10m > 5m) ===
{"accepted":0,"rejected":1,"rejections":[{"pointCode":"SMOKE0026_P1","ts":1795376136947,"reason":"ts_overflow"}],"duplicate":false}
—— 拒绝明细携带消息原始 ts（1795376136947 = 发送侧 +10m 值）

=== B4 陈旧 ts(-40d < 30 天) ===
{"accepted":0,"rejected":1,"rejections":[{"pointCode":"SMOKE0026_P1","ts":1791919536947,"reason":"ts_too_old"}],"duplicate":false}

=== B5a/B5b/B5c 未知编码 ×3（告警采样 N=2）===
{"accepted":0,"rejected":1,"rejections":[{"pointCode":"SMOKE_X99_UNKNOWN","ts":...,"reason":"unknown_code"}],"duplicate":false} ×3

=== B1 重报（同 batchId 整批幂等）===
{"accepted":5,"rejected":0,"rejections":[],"duplicate":true}
```

**步骤 3：ingestd 日志（纠偏打标 + 死信按租户隔离计数采样告警）**

```text
2026/11/23 03:25:37 pipeline: [ts-corrected] tenant=default pointCode=SMOKE0026_P1 origTs=1795375656947 correctedTs=1795375537113 count=1
2026/11/23 03:25:37 pipeline: [dead-letter] tenant=default reason=ts_overflow count=1 pointCode=SMOKE0026_P1 batchId=B0028-3
2026/11/23 03:25:37 pipeline: [dead-letter] tenant=default reason=ts_too_old count=1 pointCode=SMOKE0026_P1 batchId=B0028-4
2026/11/23 03:25:46 pipeline: [dead-letter] tenant=default reason=unknown_code count=1 pointCode=SMOKE_X99_UNKNOWN batchId=B0028-U1
2026/11/23 03:25:46 pipeline: [dead-letter] tenant=default reason=unknown_code count=2 pointCode=SMOKE_X99_UNKNOWN batchId=B0028-U2
```

读证：`[ts-corrected]` origTs=+2m 值 → correctedTs=接收时刻（纠偏打标实证）；`[dead-letter]` 固定格式含租户/原因/计数/点位/批次；unknown_code×3 只出 count=1、count=2 两行（N=2 首次+每 2 条采样，count=3 不出——防风暴采样实证）；**B1 重报后日志零新增死信行（重复上报不产生重复死信计数 live 实证）**。

**步骤 4：全停**

```text
taskkill /PID 54880（ingestd）/ taskkill /PID 54332（backend）→ 均「成功: 已终止」
netstat 8090/8098 LISTENING → 0（端口归零，用毕即停）
backend 全程日志 grep -c " ERROR " = 0
```

### 2.3 验收 3：改动范围核对

```text
git show --name-only 8a837aa b4e877a | grep -v "^ingest/"（剔除提交头后非 ingest 行）= 0
git diff --name-only cebf17a..HEAD -- ingest/ | wc -l = 18（恰为本任务 17+1 修订文件）
git status --porcelain -- ingest/ = 空（工作区干净）
cebf17a..HEAD 中另有的 backend/** 变更属并行 0079/0080 道提交（a0a92b7/9197e7e），本任务未触碰
```

## 3. 未做项与原因（brief 明确归后/边界外，非缺口）

| 未做项 | 归属 | 原因 |
| --- | --- | --- |
| 死信计数 Redis 化（多实例聚合） | 演进位 | brief 裁决 2 明示一期=进程内有界 map（key=tenant+reason），Redis 化登记演进位（deadletter.go/pipeline.go javadoc 已登记） |
| 攒批写入真 TDengine、Redis 点位缓存真实现、新值事件 | 0029 | brief 裁决 6 不碰；writer 仍为记录式假执行器、cache 内存假实现原样保留 |
| sink 同步调用队头约束 / 停机排空 | 0029 | brief 分流明确（0027 报告遗留 §3 同口径） |
| MQTT 路径 live 实测（EMQX 真 broker） | TD-0057（挂 0068） | 本地无 EMQX（0027 已登记）；时间/量程/死信语义 MQTT 与 HTTP 共用同一 processOne（两路一致性由 twopath 契约测试锁定），本批 live 冒烟走 HTTP 面 |
| 纠偏点的显式「已纠偏」质量码/标记字段 | 契约变更另议 | brief 裁决 3：CONV-02 码域冻结不自造码——本批打标=结构化 WARN + 独立计数器（§5.1 登记） |
| `-race` 检测器 | （环境限制） | 本机无 gcc（cgo 缺失）——同 0076/0027 口径；并发安全由竞态回归测试行为断言覆盖并已 -count=10×2 稳定复跑 |
| `format_invalid` 经 JSON 的自然触发路径 | （防御分支定位） | JSON number 语义下 NaN/Inf 不可达（encoding/json 拒收）；该原因码按 brief 枚举要求保留， producer=流水线点位级格式防御（防注入式 Parser/未来解析器），以假 Parser 契约测试锁定（§4.5） |

## 4. 偏离计划之处

### 4.1 量程码位映射以冻结枚举为准（brief 行文勘误，最重要偏离）

brief 裁决 1 行文「value<min → quality=1（下限越界）、value>max → quality=2（上限越界）」与 CONV-02 冻结枚举（workitems/conventions.md：**1 超上限、2 超下限**；ingest/internal/quality 枚举与 FromTextCode 映射同口径）文字相反。处置：以冻结枚举为准——value>max→OverMax(1)、value<min→UnderMin(2)。为什么现在对了：CONV-02 是 brief 自指认的契约权威且枚举冻结（「量程外打码 1/2 不丢弃」验收锚点两侧语义一致，均满足）；FromTextCode("OVER_RANGE", overMax=true)→1 先例同向。CONV-02 枚举本身经核对无问题（未触发「报告并停」条款），属 brief 转写勘误，已在本报告登记。

### 4.2 既有验证类 harness 冻结时钟（定时炸弹拆除）

时间三分裁决默认开启（脏数据防线缺省装配），而既有 135 契约测试大量硬编码 ts=1795211995000（契约样例，2026-11-21）——按真实墙钟将在 2026-12-23 后集体落入 ts_too_old（测试定时炸弹）。处置：5 处验证类 harness（pipeline 0073/竞态 harness、twopath、httpapi realPipeline、tdengine fail-soft）注入冻结时钟 `Now=契约样例 sentAt(1795212000000)`，硬编码 ts 恒落补传豁免带，断言零改动。为什么现在对了：Deps.Now 为本批新增的裁决基准注入点，测试冻结时钟=裁决确定性的一部分；产品装配（ingestd）不注入走 time.Now。既有测试文件改动仅此一行注入 + `TestContractBufferCapDropsOldestAndCounts` 显式放开时间阈值（该测专查攒批缓冲 FIFO，与时间裁决正交；其 ts=1000/2000/3000 纪元值在新默认阈值下会误入 ts_too_old）。

### 4.3 Assemble 缺省挂点与缺省死信出口替换

0027 的 Assemble 缺省 Judge=PassthroughJudge（直通占位）、Dead=no-op。本批按「挂点真实现」缺省替换为 RangeQualityJudge 与 DeadLetterCounter（ingestd 同时显式装配以注入 env 阈值）。既有测试显式注入 Judge/Dead 的不受影响；未显式注入的 4 处装配经核对快照量程与样例值均不越限，行为等价全绿。为什么现在对了：缺省真实现是「内核默认即产品语义」的正确缺省（防未来装配路径漏配时间/量程面 fail-open）。

### 4.4 并发红线测试 stale 快照竞态读修正（live 复跑驱动的红转绿）

验收复跑 `go test ./internal/pipeline/ -run 'Race' -count=10` 偶发 `TestContractSubmitRaceSamePointTsExactlyOnce: WrittenPoints = 26, want 30`。根因：该测在收敛等待（eventually 2s 内 WrittenPoints==30 已通过）之后，用**收敛前**捕获的快照 `st`（30 轮 Submit 刚结束时）复核——攒批落库为异步，快照取值时 flusher 尚未写完，属测试侧竞态读而非流水线丢行（AcceptedPoints=30/DuplicatePoints=330 两项同步账面断言始终通过，恰一写出语义未被破坏）。修正：复核改取最新 `Stats()`（收敛判定后的新快照）。为什么现在对了：恰一写出由「同步账面（Accepted/Duplicate 计数）+ 收敛 eventually（WrittenPoints==30）+ 收敛后复核」三重锁定；旧快照值不构成任何账面事实。修正后 -count=10 两轮 + -count=3 复跑稳定全绿（§2.1）。该测试为 0027 交付既有项，本批在其归属文件内做最小修正并留痕。

### 4.5 format_invalid 的 producer 定位

brief 要求原因码枚举含 format_invalid（与 HTTP 拒绝明细对齐）。JSON number 语义下 NaN/±Inf 在 encoding/json 层不可达，若枚举无 producer 则为死码。处置：producer=processOne 点位级格式防御（非有限值→format_invalid 死信，嵌入四步校验之格式步、先于时间步——假 Parser 契约测试锁定「格式步先于时间步」），覆盖注入式 Parser/未来解析器路径；信封级格式非法仍走 invalid_envelope 整批拒绝（0027 语义不变）。

### 4.6 gofmt 全目录扫描回退

执行中对 ingest/internal+cmd 做了整体 `gofmt -w`，波及 9 个本任务未触碰文件（纯格式化漂移）。处置：git checkout 回退全部格式化漂移文件，使两提交 diff 严格限于任务文件；任务文件本身经 gofmt -l 核对零格式告警（`pipeline_submit_contract_test.go` 的告警为 0027 提交时的既有状态，未触碰不回填）。

### 4.7 backend dev 起动方式（同 0027 先例）

8098 空闲但任务禁跑 mvn/编译 backend——以既有 jar `java -jar` 起 dev 实例（零编译零源码写入），实例仅在冒烟段存活（起前探测空闲、用毕即停、端口归零确认）；对并行 0015/0079/0080 道零干扰（其工作区/提交全程未触碰）。

## 5. 遗留问题

### 5.1 [纠偏显式标记——契约留白待裁决]

时间三分裁决的「打标」一期落地为结构化 WARN 日志（采样）+ 独立计数器（Stats.TimeCorrected / CorrectionTagger.Count），落库行不含「已纠偏」痕迹（CONV-02 码域冻结不自造码，ts 本身已按裁决纠偏）。若产品要求纠偏行可查询追溯（如显式标记列/保留原 ts 影子行），须走 CONV-03 契约升版另议——本批未动契约。

### 5.2 [被拒点跨 batchId 重发的计数口径]

四步校验「去重靠后」口径下，被拒点（ts_overflow/ts_too_old/unknown_code 等）不标记去重键：同 batchId 重报由整批幂等吸收（零逐点处理、零新增死信，live+契约双实证）；跨 batchId 重发被拒点按新死信事件计数（观测口径——每次重发确为一次死信worthy事件），风暴由采样告警（首次+每 N 条）约束。若产品要求被拒点也吸收重发（不重复计数），需为拒绝路径引入独立去重键面——登记待裁决，本批按观测口径落地。

### 5.3 [纠偏计数包含被吸收重复点]

时间步在去重步之前（去重靠后口径），未来窄带点的跨批重报会再次执行纠偏（TimeCorrected/打标计数随之上调）后被点级去重吸收——即纠偏计数语义=「执行过的纠偏次数」而非「落库的纠偏行数」。落库恰一行由去重键（原始 ts）保证（契约测试锁定）；若产品要求纠偏计数只计落库行，需把纠偏打标挪到去重命中之后——登记待裁决，本批按裁决步骤顺序落地。

### 5.4 [TD-0057 MQTT 真联调（0027 登记，本批追加验证项）]

EMQX 容器就绪后执行 0027 报告 4 步验证清单，追加第 ⑤ 步：MQTT data 主题投递混合批（含超限值/未来漂移 ts/陈旧 ts），断言死信计数/纠偏打标/拒绝分流与 HTTP 面 two-path 一致（本批语义已由 twopath 契约测试锁定，真链路核验归联调）。

### 5.5 [环境与并行]

- 本机无 gcc：`-race` 未跑（同 0076/0027 登记）；并发安全由竞态回归测试行为断言覆盖（-count=10×2 稳定），建议有 gcc 环境补一轮 `go test -race ./internal/pipeline/`。
- 并行道（0015/0079/0080）backend/** 提交与工作区全程未触碰、未编译、未测试；本任务 backend 交互仅 live 冒烟段（既有 jar 起停 + HTTP API）。
- 冒烟数据零新增：复用 0027 遗留通道 21/网关 9111/快照 v5 与凭据；backend dev 实例已停、8090/8098 归零。
- WSL mysql/redis 全程在跑未中断；无长间隙未做额外保活。

## 可追溯性

- **git 提交**：`8a837aa` + `b4e877a`（main，仅 ingest/** 18 文件，均注明 TASK-0028）
- **修改文件**：§1 清单 18 项（10 修改 + 8 新增）；非 ingest 文件数 = 0
- **验收**：§2 三条命令原始输出全部通过；契约测试 168/168 PASS（既有 135 零回归）；live 冒烟混合批分流/纠偏打标/死信采样/幂等重报全实证

## 六、一句话结论

**通过**——验收 1（build/vet 双零退出 + go test 全 10 包 ok + 契约 168 PASS/0 FAIL〔既有 135 零回归+新增 33〕+ Race -count=10/-count=3 稳定）、验收 2（live 冒烟：量程五值批 accepted=5 只打码不拒绝含双边界、未来漂移 ts 纠偏打标〔ts-corrected origTs→correctedTs〕、+10m→ts_overflow/-40d→ts_too_old 拒绝明细携带原始 ts 不落库、未知编码死信按租户计数首条+每 N 条采样告警、B1 重报 duplicate=true 零新增死信、全停端口归零 backend 零 ERROR）、验收 3（两提交 18 文件全落 ingest/**、越界 0）三条全部以原始输出为证通过；偏离均为契约权威取舍（§4.1 冻结枚举为准）、测试确定性加固（§4.2 冻结时钟）与测试侧竞态读修正（§4.4 红转绿留痕），遗留仅契约留白待裁决项（§5.1-5.3）与 TD-0057 真联调登记，无阻断缺陷。
