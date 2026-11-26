# TASK-0073 实施报告：ingest 流水线内核先行（契约驱动 mock TDD）

- **BASE**：`c1b2615`（主仓 main，开工时 HEAD）
- **提交**：`cff0642`（首轮）、`c2ff33e`（修订轮：I1 + I2）；两者均 main、只 add `ingest/` 明确路径、信息注明 TASK-0073
- **同期他道提交（非本任务）**：`d971c10`（TASK-0008 修订轮）、`384d58f`（TASK-0005），验收命令 4 区间内出现 backend 文件均属这两个提交，本任务两个提交的边界核对见 §2.4
- 执行者：Go 道执行者（implementer），2026-11-22
- 工具链：go 1.27.2 windows/amd64；全程无第三方依赖（`go.mod` require 段为空）
- 评审：`TASK-0073-review.md`（FAIL，0 Critical / 2 Important / 9 Minor）→ 本报告 §8 为退回轮修复记录；Minor M1-M9 由协调者记 deferred（M8 数字修正已并入本报告）

## 0. 交付概述

按 brief 决策 2 的管道形态交付内核：`Consumer(假 EMQX=内存 channel) → Parser → [查快照→未知死信→禁用丢弃→(pointCode,ts)/batchId 去重→质量判定挂点] → Batcher(有界缓冲/水位/定时/丢最旧/失败重排+重排上限) → Writer(假 TDengine 执行器) → PointCache 更新`，外加 `SnapshotFetcher(HTTP + httptest 假端点) → 快照缓存原子替换`。组件间只经接口交互，0027-0029 接线时假实现零改动替换。质量判定只留挂点与 CONV-02 枚举/映射常量，完整判定逻辑归 0028。TDD 全程红→绿（每个包先写失败测试，红确认后实现；修订轮两条 Important 同样先红后绿）。

## 1. 改动文件清单（全部在 `ingest/**`）

首轮提交 `cff0642`（15 个文件）：

| 文件 | 性质 | 说明 |
| --- | --- | --- |
| `ingest/README.md` | 修改 | 目录与内核流水线说明（组件、冻结处理顺序、并发模型、Contract 测试口径） |
| `ingest/internal/quality/quality.go` | 新增 | CONV-02 质量码 0-10 枚举、文本码映射（GOOD→0、OVER_RANGE→1/2、STALE/OFFLINE→3、JUMP→8、MANUAL→5、CALC_ERR 不映射）、0-10 取值域判断 |
| `ingest/internal/quality/quality_contract_test.go` | 新增 | 契约符合性测试 ×3 |
| `ingest/internal/message/message.go` | 新增 | 统一消息模型 v1.1.0 §3.2 解析器：信封/点位类型、ts 与 sentAt 分开保留、空 points=4001 整批拒绝、超 500 整批拒绝、tenant/batchId 必填、未知字段宽松（网关演进不打停接入面） |
| `ingest/internal/message/message_contract_test.go` | 新增 | 契约符合性测试 ×5（§3.2 JSON 示例逐字逐字段断言） |
| `ingest/internal/snapshot/snapshot.go` | 新增（修订轮扩） | 配置快照 v1.0.1 五对象类型 + 字段级校验（required 存在性、additionalProperties=false、enum、maxLength）+ 快照体解析 + config.changed 载荷 + 本地缓存原子替换（校验失败保旧版）+ pointCode 索引 |
| `ingest/internal/snapshot/fetch.go` | 新增 | §3.3 拉取客户端：GET /internal/v1/config/snapshot?scopeType=&scopeId=&version=（version=0 不带参=最新版）、信封三部分解析、信封 version 与快照体 version 同值校验；三个契约路径常量（changes/ack 留常量） |
| `ingest/internal/snapshot/snapshot_contract_test.go` | 新增（修订轮扩） | 契约符合性测试 ×12（顶层结构逐字示例、装载键省略、五对象 Schema、enum 违例、additionalProperties=false、required 存在性 ×2、required 违例缓存保旧版） |
| `ingest/internal/snapshot/snapshot_fetch_contract_test.go` | 新增 | 契约符合性测试 ×7（httptest 假端点回放：请求 URL、不带 version、信封同值、错版拒收、缓存原子替换保旧版、未知编码未命中、通知载荷） |
| `ingest/internal/tdengine/tdengine.go` | 新增（修订轮扩） | 超级表契约 v1.0.0 写入侧：dataType→kind/超级表映射（1 INSTANT/2 CUMUL→st_point_num，3 STATE/4 SWITCH→st_point_state）、附录 B 建库建表 DDL 逐字（库名 iot_<tenant> 参数化）、INSERT INTO 子表 USING 超级表 TAGS(...) VALUES 多行合并、子表名/库名确定性映射（修订轮 I2）、Writer 按子表分组、FakeExecutor（记录谁在何时以什么参数调了什么 + 失败注入） |
| `ingest/internal/tdengine/tdengine_contract_test.go` | 新增（修订轮扩） | 契约符合性测试 ×7（INSERT 形态逐字对照、kind 映射、附录 B DDL 归一空白逐字对照、同子表多行合并、按子表分组、pointCode 映射不拒收、映射确定性/长度上限） |
| `ingest/internal/cache/cache.go` | 新增 | Redis 点位缓存抽象：Value{v,q,ts}（§3.5 realtime 返回口径）、键拟定 scheme、内存假实现（JSON 字节存储模拟 Redis 字符串语义） |
| `ingest/internal/cache/cache_contract_test.go` | 新增 | 契约符合性测试 ×2（JSON 恰 v/q/ts 三键、同形态读回）+ 行为测试 ×2 |
| `ingest/internal/pipeline/pipeline.go` | 新增（修订轮扩） | 内核流水线组装：Consumer/Parser/SnapshotLookup/BatchDeduper/PointDeduper/DeadLetterSink/Writer/PointCache 八接口 + QualityJudge 函数挂点；冻结五步处理顺序；单消费循环+单 flusher 并发模型（doc comment 全文写明）；有界攒批缓冲（满丢最旧计数，ADR-0014 FIFO 口径）、水位+定时双触发、写失败整批重排 + 行级重排上限 3 次/超限死信（修订轮 I2）；内存去重假实现（FIFO 淘汰）；Stats 计数（FEAT-0020 数据源挂点） |
| `ingest/internal/pipeline/pipeline_contract_test.go` | 新增（修订轮扩） | 契约符合性测试 ×15（含重排超限死信、契约合法 pointCode 端到端）+ 假实现/断言辅助 |

修订轮提交 `c2ff33e`（6 个文件）：`snapshot.go` / `snapshot_contract_test.go` / `tdengine.go` / `tdengine_contract_test.go` / `pipeline.go` / `pipeline_contract_test.go`（内容见 §8）。

未改动：`ingest/go.mod`（require 段保持为空）、`ingest/cmd/ingestd/main.go`（占位入口原样，接线归 0027）。

## 2. 验收命令原始输出（修订轮提交 c2ff33e 后复跑）

### 2.1 `cd ingest && go build ./... && go vet ./...`

```
=== CMD1: go build ./... && go vet ./... ===
EXIT=0 (no output above = clean)
```

退出码 0，build 与 vet 均无输出。

### 2.2 `cd ingest && go test ./... -count=1`

```
=== CMD2: go test ./... -count=1 ===
?   	net.otlab/ingest/cmd/ingestd	[no test files]
ok  	net.otlab/ingest/internal/cache	1.861s
ok  	net.otlab/ingest/internal/message	1.850s
ok  	net.otlab/ingest/internal/pipeline	3.022s
ok  	net.otlab/ingest/internal/quality	1.202s
ok  	net.otlab/ingest/internal/snapshot	3.546s
ok  	net.otlab/ingest/internal/tdengine	1.243s
EXIT=0
```

全绿，退出码 0。（契约符合性断言的 -v 明细见 2.3。）

### 2.3 `cd ingest && go test ./... -run 'Contract' -v`（单独复跑）

> **M8 勘误**：首轮报告本节结论行误写「共 36 项」，与其自身输出不符；评审复跑实测首轮为 **44 项 PASS**。修订轮新增 7 项后，本轮实测 **51 项 PASS / 0 FAIL**，以下为修订轮原始输出。

```
=== CMD3: go test ./... -run 'Contract' -v ===
?   	net.otlab/ingest/cmd/ingestd	[no test files]
=== RUN   TestContractPointCacheValueShape
--- PASS: TestContractPointCacheValueShape (0.00s)
=== RUN   TestContractPointCacheRoundTripSameShape
--- PASS: TestContractPointCacheRoundTripSameShape (0.00s)
PASS
ok  	net.otlab/ingest/internal/cache	2.656s
=== RUN   TestContractParseUpstreamJSONExample
--- PASS: TestContractParseUpstreamJSONExample (0.00s)
=== RUN   TestContractTsAndSentAtKeptDistinct
--- PASS: TestContractTsAndSentAtKeptDistinct (0.00s)
=== RUN   TestContractParseRejectsEmptyPointsWith4001
--- PASS: TestContractParseRejectsEmptyPointsWith4001 (0.00s)
=== RUN   TestContractParseRejectsOverLimitWholeBatch
--- PASS: TestContractParseRejectsOverLimitWholeBatch (0.00s)
=== RUN   TestContractParseAcceptsExactly500Points
--- PASS: TestContractParseAcceptsExactly500Points (0.00s)
PASS
ok  	net.otlab/ingest/internal/message	2.465s
=== RUN   TestContractUnknownCodeDeadLetterNotWritten
--- PASS: TestContractUnknownCodeDeadLetterNotWritten (0.05s)
=== RUN   TestContractEmptyPointsRejected
--- PASS: TestContractEmptyPointsRejected (0.01s)
=== RUN   TestContractOverLimitRejected
--- PASS: TestContractOverLimitRejected (0.01s)
=== RUN   TestContractBatchIdDedup
--- PASS: TestContractBatchIdDedup (0.03s)
=== RUN   TestContractPointTsDedup
--- PASS: TestContractPointTsDedup (0.04s)
=== RUN   TestContractDisabledPointDiscarded
--- PASS: TestContractDisabledPointDiscarded (0.04s)
=== RUN   TestContractDispatchByDataTypeAndCacheUpdate
--- PASS: TestContractDispatchByDataTypeAndCacheUpdate (0.03s)
=== RUN   TestContractQualityJudgeHookApplied
--- PASS: TestContractQualityJudgeHookApplied (0.03s)
=== RUN   TestContractFlushBySizeWatermark
--- PASS: TestContractFlushBySizeWatermark (0.00s)
=== RUN   TestContractFlushByInterval
--- PASS: TestContractFlushByInterval (0.03s)
=== RUN   TestContractBufferCapDropsOldestAndCounts
--- PASS: TestContractBufferCapDropsOldestAndCounts (0.07s)
=== RUN   TestContractWriteFailureRequeued
--- PASS: TestContractWriteFailureRequeued (0.03s)
=== RUN   TestContractTenantFromConsumerRequired
--- PASS: TestContractTenantFromConsumerRequired (0.01s)
=== RUN   TestContractWriteRetriesExhaustedDeadLetter
--- PASS: TestContractWriteRetriesExhaustedDeadLetter (0.22s)
=== RUN   TestContractNonAsciiPointCodeEndToEnd
--- PASS: TestContractNonAsciiPointCodeEndToEnd (0.04s)
PASS
ok  	net.otlab/ingest/internal/pipeline	3.793s
=== RUN   TestContractQualityCodesZeroToTen
--- PASS: TestContractQualityCodesZeroToTen (0.00s)
=== RUN   TestContractQualityTextCodeMapping
--- PASS: TestContractQualityTextCodeMapping (0.00s)
=== RUN   TestContractQualityCalcErrNotMappable
--- PASS: TestContractQualityCalcErrNotMappable (0.00s)
PASS
ok  	net.otlab/ingest/internal/quality	2.151s
=== RUN   TestContractSnapshotTopLevelStructure
--- PASS: TestContractSnapshotTopLevelStructure (0.00s)
=== RUN   TestContractSnapshotScopeLoadingOmittedKeys
--- PASS: TestContractSnapshotScopeLoadingOmittedKeys (0.00s)
=== RUN   TestContractPointObjectSchema
--- PASS: TestContractPointObjectSchema (0.00s)
=== RUN   TestContractPointObjectEnumViolations
--- PASS: TestContractPointObjectEnumViolations (0.00s)
=== RUN   TestContractPointObjectAdditionalPropertiesFalse
--- PASS: TestContractPointObjectAdditionalPropertiesFalse (0.00s)
=== RUN   TestContractChannelObjectSchema
--- PASS: TestContractChannelObjectSchema (0.00s)
=== RUN   TestContractDataItemObjectSchema
--- PASS: TestContractDataItemObjectSchema (0.00s)
=== RUN   TestContractFormulaObjectSchema
--- PASS: TestContractFormulaObjectSchema (0.00s)
=== RUN   TestContractMetricObjectSchema
--- PASS: TestContractMetricObjectSchema (0.00s)
=== RUN   TestContractPointObjectRequiredEnforced
--- PASS: TestContractPointObjectRequiredEnforced (0.00s)
=== RUN   TestContractAllFiveObjectsRequiredEnforced
--- PASS: TestContractAllFiveObjectsRequiredEnforced (0.00s)
=== RUN   TestContractSnapshotCacheKeepsOldOnRequiredViolation
--- PASS: TestContractSnapshotCacheKeepsOldOnRequiredViolation (0.00s)
=== RUN   TestContractSnapshotFetchRequestURL
--- PASS: TestContractSnapshotFetchRequestURL (0.01s)
=== RUN   TestContractSnapshotFetchWithoutVersionReturnsLatest
--- PASS: TestContractSnapshotFetchWithoutVersionReturnsLatest (0.00s)
=== RUN   TestContractSnapshotFetchEnvelope
--- PASS: TestContractSnapshotFetchEnvelope (0.00s)
=== RUN   TestContractSnapshotFetchVersionMismatchRejected
--- PASS: TestContractSnapshotFetchVersionMismatchRejected (0.00s)
=== RUN   TestContractSnapshotCacheAtomicReplaceKeepsOldOnInvalid
--- PASS: TestContractSnapshotCacheAtomicReplaceKeepsOldOnInvalid (0.00s)
=== RUN   TestContractSnapshotCacheLookupUnknownPoint
--- PASS: TestContractSnapshotCacheLookupUnknownPoint (0.00s)
=== RUN   TestContractConfigChangedNotificationPayload
--- PASS: TestContractConfigChangedNotificationPayload (0.00s)
PASS
ok  	net.otlab/ingest/internal/snapshot	4.235s
=== RUN   TestContractInsertShapeNumInstant
--- PASS: TestContractInsertShapeNumInstant (0.00s)
=== RUN   TestContractInsertShapeStateAndKindMapping
--- PASS: TestContractInsertShapeStateAndKindMapping (0.00s)
=== RUN   TestContractSuperTableDefinitionsMatchAppendixB
--- PASS: TestContractSuperTableDefinitionsMatchAppendixB (0.00s)
=== RUN   TestContractBatchInsertMultiRowSameChild
--- PASS: TestContractBatchInsertMultiRowSameChild (0.00s)
=== RUN   TestContractWriterGroupsByChildTable
--- PASS: TestContractWriterGroupsByChildTable (0.00s)
=== RUN   TestContractPointCodeMappedNotRejected
--- PASS: TestContractPointCodeMappedNotRejected (0.00s)
=== RUN   TestContractChildTableNameEscaping
--- PASS: TestContractChildTableNameEscaping (0.00s)
PASS
ok  	net.otlab/ingest/internal/tdengine	1.577s
EXIT=0
```

51 项 `Contract` 命名断言全部通过（cache 2 / message 5 / pipeline 15 / quality 3 / snapshot 19 / tdengine 7）。附加证据（非验收项）：

```
CGO_ENABLED=1 go test ./... -count=1 -race
→ cgo: C compiler "gcc" not found: exec: "gcc" is not in %PATH%（本机无 C 编译器，-race 不可用，见 §6 遗留）
```

### 2.4 `git diff --name-only c1b2615..HEAD`

```
=== CMD4: git diff --name-only c1b2615..HEAD ===
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermAccessStrategy.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermConditionAssembler.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermDeptRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermInstanceRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermRoleRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/NoOpVisibleScopeCache.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/TdengineTagConditionRenderer.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/VisibleScope.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/VisibleScopeCache.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermDeptMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermInstanceMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermRoleDeptMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermRoleMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermUserMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermAccessStrategyTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermClientScopeTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermConditionAssemblerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermResolveScopeTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermResolveSubtreeTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermTagConditionRendererTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImplTest.java
ingest/README.md
ingest/internal/cache/cache.go
ingest/internal/cache/cache_contract_test.go
ingest/internal/message/message.go
ingest/internal/message/message_contract_test.go
ingest/internal/pipeline/pipeline.go
ingest/internal/pipeline/pipeline_contract_test.go
ingest/internal/quality/quality.go
ingest/internal/quality/quality_contract_test.go
ingest/internal/snapshot/fetch.go
ingest/internal/snapshot/snapshot.go
ingest/internal/snapshot/snapshot_contract_test.go
ingest/internal/snapshot/snapshot_fetch_contract_test.go
ingest/internal/tdengine/tdengine.go
ingest/internal/tdengine/tdengine_contract_test.go
EXIT=0
```

**范围说明**：`c1b2615..HEAD` 区间含四个提交——本任务 `cff0642`（15 个 ingest 文件）与 `c2ff33e`（6 个 ingest 文件），以及同期他道提交 `d971c10`（TASK-0008 修订轮）与 `384d58f`（TASK-0005），全部 backend/application.yml 文件属后两者。本任务两个提交逐提交边界核对（`git show --name-only`）：`cff0642` 15 文件全部 `ingest/**`（backend 计数 0）、`c2ff33e` 6 文件全部 `ingest/**`（backend 计数 0）。brief 已预期「主仓同期有 backend 道提交」。

### 2.5 `cat go.mod`

```
=== CMD5: cat go.mod ===
module net.otlab/ingest

go 1.27
EXIT=0
```

require 段为空：零第三方依赖（生产与测试均纯标准库）。

## 3. TDD 红绿与修复记录（每个修复附「为什么现在对了」）

首轮按包先写测试→运行确认红→实现→绿；修订轮两条 Important 同样先红后绿（见 §8）。过程中共 6 处测试侧修正，无一处为凑输出改实现、也无一处为实现缺陷回改测试掩盖：

1. **tdengine DDL 语句数 5≠6**：测试组装时漏拼 `USE iot_group_a;`（实现侧 UseSQL 独立存在）。修正测试组装补入 USE。为什么现在对了：附录 B 原文为「CREATE DATABASE / USE / 4×CREATE STABLE」共 6 句，组装补齐后与契约逐字一致。
2-4. **pipeline 三处 WrittenPoints 断言早于 flush 完成**（batchId 去重、(pointCode,ts) 去重、禁用点位三测）：原断言只等去重/丢弃计数，但落库发生在 30ms 定时 flush 上，存在先断言后落库的时序错位。修正为在 eventually 条件中同时等待「计数 + WrittenPoints」。为什么现在对了：流水线行为本就正确（计数与落库分别发生），错的是断言时序；合并等待后消除竞态窗口。
5. **dispatch 测试误期望「M01+M02 合并单条 INSERT」**：M01 与 M02 是不同点位，§2.2 子表粒度为「每个数值型点位一张」，不同点位分属不同子表，Writer 按子表分组下发 2 条 INSERT 是契约正确行为。修正期望为 2 条并逐条断言。同子表多行合并的正确性由 tdengine 包的 TestContractBatchInsertMultiRowSameChild / TestContractWriterGroupsByChildTable 覆盖。
6. **失败重排测试把失败调用计入成功语句数**：FakeExecutor 记录全部调用（含注入失败的那次），原断言对所有调用数匹配首行 VALUES 得 2。修正为断言「恰 2 次调用 + 第 2 次按原序含两行」。为什么现在对了：记录式假执行器的口径就是记录全部调用（谁在何时以什么参数调了什么），失败与成功调用都应有记录；重排语义由调用序列 + 计数（WriteFailures=1、RequeuedRows=2、WrittenPoints=2）共同证明不丢不重。

## 4. 未做项与原因

| 项 | 归属 | 原因 |
| --- | --- | --- |
| 真 EMQX 订阅、HTTP `/ingest/v1/report` 与 `/ingest/v1/auth/mqtt`、网关令牌认证链 | TASK-0027 | brief 决策 2/3：内核只做管道形态与假实现；本任务不宣称任何真实连通性 |
| 质量判定完整逻辑（量程二次判定覆盖、时间校验纠偏/补传豁免、JUMP 检测） | TASK-0028 | brief 决策 2 与执行纪律 10：只留挂点（QualityJudge）与 CONV-02 枚举/映射常量 |
| 真 TDengine 连接（真 Executor）、建库建表的实际执行、真 Redis 客户端实现 | TASK-0029 | 同上；假执行器仅记录调用形态 |
| 真实联调验证 | TD-0053（TASK-0068） | 本地无 TDengine/EMQX，mock 口径 |
| config.changed 通知的订阅消费循环（通知→拉新→校验→原子替换的常驻流程） | TASK-0027 接线 | 内核已交付触发信号的载荷类型（ParseConfigChanged）、拉取客户端与 Cache.Replace 原语，循环组装属接线 |
| `/internal/v1/config/changes` 增量查询与 `/internal/v1/config/ack` 回执上报的调用实现 | 0027/0029 | 契约路径常量已留（ChangesPath/AckPath），调用时机（发布确认链）属接线职责 |
| `instance_key` 标签赋值 | 待上游/0029 | 配置快照 v1.0.1 点位对象无 instance_code 字段（契约留白），写入行暂以空串占位（包注释与 PointRow 注释声明） |
| `st_item_value` / `st_metric_value` 的写入语句构造 | 计算引擎道 | 契约写入方为计算引擎；本任务只交付其附录 B DDL 逐字对照 |
| Redis db1+ 隔离口径落地 | TASK-0029 | 本任务未连真实 Redis（纯内存假实现，不碰 localhost:6379 db0）；隔离参数随 0029 真 Redis 实现落地 |
| Minor M1-M9 | 协调者记 deferred | 评审裁决 Minor 不进修订轮（M3 租户假设声明、M5 DefaultClient 超时等归 0027 接线闭合；M8 报告数字修正已并入本报告 §2.3） |

## 5. 偏离计划之处

1. **未引入 miniredis**（brief 决策 4 提及的测试依赖示例）：内核只交付 PointCache 抽象 + 内存假实现，没有 Redis 客户端实现可供 miniredis 检验（真 Redis 客户端是 0029 的生产依赖，非测试依赖，引入即违反「外部依赖仅限测试用途」）。结果：go.mod require 为空，比计划更干净；miniredis 随 0029 与其 Redis 客户端一起引入时仍属测试依赖。
2. **快照响应信封包装键名 `"snapshot"` 为拟定**：契约 §3.3 只承诺响应含 version、snapshotHash 与快照体三部分内容，未定信封键名（快照体内部 version 与信封 version「同值」证明快照体必然嵌套）。本实现拟用 `"snapshot"` 键（fetch.go envelope 结构体 + 包注释声明留白）；上游定版后单点修改该结构体即可，接口不变。
3. **子表命名 = 点位编码的确定性映射**：契约只定「每点位一张子表、首次写入自动创建」，未定命名方案。映射规则见 §8 I2（恒等/转义/前缀/截断加哈希），0029 如需调整只改 `ChildTableName` 一个函数；INSERT 形态断言锚点（USING 超级表、TAGS 值）不受影响。
4. **幂等标记口径**：batchId / (pointCode,ts) 内存假实现采用「接受即标记」而非「写入成功后标记」（崩溃窗口可能丢批），doc comment 已声明为内核简化口径，真实去重的崩溃安全实现归 0028。
5. **Config 未含 QueueDepth**：并发模型落地为「消费源自带有界通道（假 EMQX depth=64）+ 单消费循环内联处理」，攒批缓冲 BufferCap 才是本实现的第二个有界点；QueueDepth 配置项无落点故未设，模型全文写入 pipeline 包 doc comment。

## 6. 遗留问题

1. **快照哈希未复算**：§3.3「校验哈希」依赖哈希算法口径（上游未定），当前 FetchResult 只透传 snapshotHash、Cache.Replace 校验字段级 Schema 与信封/快照体 version 同值；算法定版后需补哈希复算（ACK 失败回执依赖它）。
2. **instance_key 标签缺口**：快照点位对象无 instance_code（仅 boundInstanceId），st_point_num/state 的 instance_key 标签暂空。需上游补字段或 0027/0029 经数据项绑定关系解析；在此之前该标签为空串。
3. **HTTP 同步响应缺口**：契约要求 `/ingest/v1/report` 响应返回接收条数、拒绝条数与拒绝明细，而内核管道为异步计数口径；0027 设计 HTTP 层时需要同步校验路径（Parser 已可独立复用）或响应改异步查询，需任务单裁决。
4. **并发模型吞吐上限**：单消费循环为起步口径，消息吞吐受单线程处理限制；扩多消费者前必须重估 (pointCode,ts)/batchId 去重的一致性（当前内存假实现非共享存储），已在 doc comment 声明。
5. **-race 未跑**：本机无 C 编译器（cgo 不可用），竞态检测未执行；并发正确性目前靠「单写者（单消费循环 + 单 flusher）+ 有界互斥缓冲」的结构保证与功能测试覆盖，建议有 gcc 的环境补一轮 `go test -race`。
6. **消息模型未知字段宽松 vs 快照严格**：消息模型解析对未知字段宽松（网关演进不打停接入面），快照对象严格（additionalProperties=false 契约明文）。两者口径差异是有意为之，若上游对消息模型也要求严格需升契约版本后再收紧。
7. **TAGS 值含单引号的 SQL 字面转义未做**：TAGS 的 point_code 值按原编码放入单引号字面量，编码含 `'` 时会破坏语句形态（子表名经映射已无此问题）。快照 pointCode 含单引号属契约合法但现实中罕见的输入，随 0029 真执行器一并处理（含正式的字符串转义口径）。（修订轮新登记）

## 7. 可追溯性

- git 提交：`cff0642`（首轮，15 文件）+ `c2ff33e`（修订轮，6 文件）；均 main、只 add `ingest/` 明确路径、信息注明 TASK-0073
- 修改文件：§1 清单，全部 `ingest/**`；workitems 库与 `resources/**` 未动（契约只读），backend/frontend/application*.yml 未触碰
- 需求追溯：FEAT-0009 / REQ-0013 内核子集（五步处理顺序、批量上限 500、空 points 4001、batchId 与 (pointCode,ts) 幂等、未知编码死信不落库、禁用丢弃计数、按 data_type 分发两张点位超级表、攒批 100-500 条或 200ms、禁逐条写库）；TC-0009 的内核可验证子集（步骤 3/4/6/7 与无假 0 的质量码口径挂点）已入契约测试；步骤 1/2/5/8/9 涉及真通道与纠偏逻辑，归 0027/0028 集成验证

## 8. 修订轮（退回轮）修复记录

评审：`TASK-0073-review.md`（FAIL，0 Critical / 2 Important / 9 Minor）。本轮仅修复两条 Important + M8 报告数字修正；Minor M1-M9 按裁决记 deferred 不处理。修订提交：`c2ff33e`。

### 8.1 I1 · 快照字段级校验补 required 清单（红→绿）

**红**（先写测试，3 个新测试全部失败）：缺 pointName / 缺 tenantId / pointCode 空串均被判「有效」；channel 缺 hasCredential、dataItem 缺 fullKey、formula 缺 refFullKey、metric 缺 metricName 均未拒收；直接构造结构体的畸形快照经 `Cache.Replace` 未被拒收、未保旧版。

**实现**（`snapshot.go`，与现有 enum/maxLength 同层）：
- 五对象 JSON Schema `"required"` 清单逐字落为常量（point: id/tenantId/pointCode/pointName/dataType/valueType；channel: id/tenantId/gatewayId/channelCode/channelName/protocolType/hasCredential；dataItem: id/tenantId/instanceId/dataCode/fullKey/sourceType；formula: id/tenantId/itemId/refFullKey/refType；metric: id/tenantId/metricCode/metricName/instanceId/aggType/period）。
- `checkRequiredRaw`：在原始 JSON 层校验 required 键存在性（键缺失或显式 null 视为缺失）——结构体零值无法区分「缺失」与「零值」，bool（hasCredential）只能在此层判定；五个 `Parse*` 单对象入口与 `ParseSnapshot` 的对象数组逐元素路径都接入。
- `Validate()` 补必填字符串字段非空校验（pointCode/pointName/channelCode/channelName/dataCode/fullKey/refFullKey/metricCode/metricName；protocolType/valueType/period 由 enum 覆盖空串）——覆盖绕过 JSON 解析、直接构造结构体走 `Cache.Replace` 的路径（测试 `TestContractSnapshotCacheKeepsOldOnRequiredViolation` 正是此路径）。必填数值字段（id/tenantId 等）在原始层判定存在性，结构体层不做零值判断（id 类列均为正整数自增，避免把「缺失」与「零值」混为一谈的错误扩权）。
- **为什么现在对了**：契约 required 是存在性语义，原始 JSON 层才能精确判定键缺失（含 bool），字符串非空校验补齐结构体层缺口，两层的并集恰好覆盖「缺任一必填字段 → 校验失败 → 缓存保旧版（§3.3）」的全部入口；缺 tenantId 的点位快照现在进不了缓存，未知编码死信防线不再被畸形快照绕过。

**新增测试 ×3**：`TestContractPointObjectRequiredEnforced`、`TestContractAllFiveObjectsRequiredEnforced`、`TestContractSnapshotCacheKeepsOldOnRequiredViolation`（缺 pointName/tenantId/pointCode 空串/五对象清单/保旧版）。

### 8.2 I2 · 子表名确定性映射 + 重排上限（红→绿）

**红**（先写测试）：`TestContractPointCodeMappedNotRejected`/`TestContractChildTableNameEscaping` 编译失败（ChildTableName 不存在）；实现前运行时表现为含连字符 pointCode 被 `childTable` 拒绝 → `WritePoints` 中途报错 → flush 整批无限重排（`TestContractWriteRetriesExhaustedDeadLetter` 超时、死信计数不增长）。

**实现一（`tdengine.go` childTable 改全映射）**：
- `escapeIdentifier` 确定性映射：全安全字节（[A-Za-z0-9_]）恒等；非安全字节逐字节 `_XX` 十六进制转义并追加 `_`+FNV-64 后缀（与恒等集区分，避免「F01-M01」撞上字面编码「F01_2dM01…」）；首字符数字加前缀 `t_`；超长截断+哈希后缀（TDengine 标识符上限 192 字节，覆盖 96 个中文汉字这类契约合法上限输入）。
- `safeIdentifier` 降级为映射后的不变式断言，不再构成对契约合法数据的拒绝路径；`DBName` 的租户码同样走映射（消除写库层最后一处字符集拒绝）；`ChildTableName` 导出供 0029 复用。
- TAGS 的 point_code 值保留原编码（标签取自配置快照的语义不变）。
- **为什么现在对了**：契约对 pointCode 只约束 maxLength 96，任何字符集都是合法数据；写库层的合法职责是把「任意契约合法编码」确定性映射为「合法标识符」，而不是拒绝数据。映射后含连字符/中文/首字符数字的编码端到端（快照→管道→WritePoints）不再产生写库层错误，I2 的无限重排触发源被消除。

**实现二（`pipeline.go` 重排上限）**：
- 攒批缓冲改为行级条目（row + requeues 计数），同一行重排超过 3 次（`maxWriteRequeues`）→ `DeadLetterInvalidConfig` 口径死信计数 + 告警（Sink，point=nil、batchID 空串）+ `WriteRetriesExhausted` 计数，不再回灌缓冲。
- **为什么现在对了**：写失败（含不可达写库层错误）的响应有界化——真写失败（假执行器注入 1 次）3 次以内仍重排重试（原 `TestContractWriteFailureRequeued` 语义保持绿），持续失败在第 4 次转死信，不再整批无限重排、不再重复下发同 flush 已成功分组，且死信有计数可告警，符合「死信/异常计数经运维监控页展示」的 FEAT-0020 口径。

**新增测试 ×4**：`TestContractPointCodeMappedNotRejected`、`TestContractChildTableNameEscaping`（连字符 `_2d`/中文逐字节转义/首字符数字前缀/恒等/96 汉字 ≤192 字节/两次调用可复现）、`TestContractWriteRetriesExhaustedDeadLetter`（重排 3 次后超限死信、计数稳定不再增长）、`TestContractNonAsciiPointCodeEndToEnd`（三类编码快照→管道→WritePoints 全通过、子表名确定性、TAGS 保留原编码、分发不受字符集影响）。

### 8.3 M8 · 报告数字修正

首轮 §2.3 结论行「共 36 项」系笔误（评审复跑实测 44 项）；已在 §2.3 勘误并更新为修订轮实测 51 项 PASS / 0 FAIL。

### 8.4 修订轮验收

五条验收命令重跑全部通过（原始输出见 §2，BASE=c1b2615，HEAD=c2ff33e）；`go vet` 无告警；`go.mod` require 仍为空；修订提交只含 6 个 `ingest/**` 文件（§2.4 边界核对）。
