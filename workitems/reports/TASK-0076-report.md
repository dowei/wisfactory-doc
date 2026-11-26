# TASK-0076 报告：ingest HTTP 上报端点 + 网关令牌链 + TDengine 写客户端（0027 前端半 mock 先行）

- 执行者：implementer（TASK-0076 brief 驱动）
- BASE：`d74d56f`（主仓 main 开工时 HEAD）
- HEAD：`f0f4533`（本次唯一提交，仅 ingest/** 路径）
- 执行日期：2026-11-22
- 验收结论：**4/4 验收命令全部通过**（原始输出见 §2）

## 1. 改动文件清单（14 个，全部落 ingest/**）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `ingest/internal/pipeline/pipeline.go` | 修改 | 内核增量：`Submit` 同步提交入口（与 Run 共用同一 `processOne` 五步冻结顺序）+ `StartFlusher`；`processOne` 增加返回值 `SubmitResult`（受理/拒绝/逐点拒绝明细）——行为保持重构，0073 既有测试零改动全绿 |
| `ingest/internal/pipeline/pipeline_submit_contract_test.go` | 新增 | Submit 契约测试 5 项（五步顺序同构、batchId/点级幂等口径、信封级整批拒绝、与 Run 并发双入口） |
| `ingest/internal/gwtoken/gwtoken.go` | 新增 | 网关令牌链：`GatewayChannelResolver`（静态映射 mock 口径）/`CredentialPuller`/`Validator` 三接口留位 + `TokenChainValidator`（TTL 本地缓存 + 常量时间比对，fail-closed 五闸） |
| `ingest/internal/gwtoken/puller.go` | 新增 | 0019 凭据拉取客户端：GET `/internal/v1/credential/pull?channelId=&credType=`，请求头 `X-Internal-Token`（配置化），`Result<CredentialPullVO>` 信封解析（响应形状以 0019 代码为准），非 200/非 200 业务码/空密文一律转错误 |
| `ingest/internal/gwtoken/gwtoken_contract_test.go` | 新增 | 令牌链契约测试 5 项（校验+缓存、TTL 过期重拉、拉取失败 fail-closed、网关未知 fail-closed、不匹配/空参数 fail-closed） |
| `ingest/internal/gwtoken/puller_contract_test.go` | 新增 | 0019 锚点契约测试 3 项（请求形态 httptest 断言、403/401/业务码失败转错误、后端不可达有限时间转错误） |
| `ingest/internal/httpapi/report.go` | 新增 | POST `/ingest/v1/report`：X-Gateway-Token → 令牌链（fail-closed 401）→ 信封纪律（空 points 4001 / 超 500 整批 400 契约文案）→ 统一流水线入口（租户=令牌侧产物）→ 响应{accepted, rejected, rejections[], duplicate} |
| `ingest/internal/httpapi/authmqtt.go` | 新增 | POST `/ingest/v1/auth/mqtt`：端点自身内部令牌鉴权（常量时间比较，缺失/不匹配 401+deny）→ username/password（gatewayCode/网关令牌）→ 令牌链 → allow+主题范围（发自己的 data/status、收自己的 config）/ deny |
| `ingest/internal/httpapi/pipeline_sink.go` | 新增 | `PipelineSink`：ReportSink 的内核流水线实现（经 `pipeline.Submit`，与 MQTT 消费循环同一 processOne） |
| `ingest/internal/httpapi/mux.go` | 新增 | 路由装配（两端点 + /healthz，方法精确匹配 405） |
| `ingest/internal/httpapi/report_contract_test.go` | 新增 | 上报端点契约测试 8 项（契约样例 mock sink 断言、超限 400 契约文案、空 points 4001、令牌失败 fail-closed、未知编码死信（真流水线）、端到端写库+缓存、batchId 幂等重报、入口不可用 503） |
| `ingest/internal/httpapi/authmqtt_contract_test.go` | 新增 | 认证回调契约测试 6 项（allow+主题范围、令牌错 deny、拉取失败 deny、内部令牌缺失/不匹配 401、空三要素 deny、真令牌链端到端 fail-closed） |
| `ingest/internal/tdengine/rest.go` | 新增 | TDengine REST 写客户端：POST `/rest/sql`（禁用库上下文路径——9750 教训）、Basic 认证、读超时默认 30s 配置化、code 24/918 → `EnvError` 环境级 fail-soft（WARN+重试计数）、`EnsureSchema` USE 多语句复用附录 B DDL 构造器、实现 `Executor` 可直配 0073 `tdengine.Writer` |
| `ingest/internal/tdengine/rest_contract_test.go` | 新增 | REST 客户端契约测试 7 项（全限定名+Basic 认证头、EnsureSchema 多语句、默认 30s 读超时、短超时失败、24/918 环境级识别+WARN、9750 非环境级、fail-soft 经流水线重试计数收敛） |

依赖纪律：新增第三方依赖 **0**（全部标准库）；`go.mod` 未改动。新增契约测试共 **34 项**（全部以 `Contract` 命名）；0073 既有 55 项契约测试零改动全绿。

## 2. 验收命令原始输出

### 2.1 `cd ingest && go build ./... && go vet ./...` → 双零退出

```text
（无任何输出，两命令均零退出；以下为命令回显确认行）
ACCEPT-1 BUILD+VET EXIT=0
```

### 2.2 `cd ingest && go test ./... -count=1` → 全绿（EXIT=0）

```text
?   	net.otlab/ingest/cmd/ingestd	[no test files]
ok  	net.otlab/ingest/internal/cache	3.991s
ok  	net.otlab/ingest/internal/gwtoken	3.706s
ok  	net.otlab/ingest/internal/httpapi	5.072s
ok  	net.otlab/ingest/internal/message	1.513s
ok  	net.otlab/ingest/internal/pipeline	3.607s
ok  	net.otlab/ingest/internal/quality	1.352s
ok  	net.otlab/ingest/internal/snapshot	3.358s
ok  	net.otlab/ingest/internal/tdengine	3.569s
ACCEPT-2 EXIT=0
```

### 2.3 `cd ingest && go test ./... -run 'Contract' -v` → 契约符合性清单（EXIT=0）

89 项契约测试全 PASS（0073 既有 55 + 本次新增 34）。以下为逐项原始输出：

```text
?   	net.otlab/ingest/cmd/ingestd	[no test files]
=== RUN   TestContractPointCacheValueShape
--- PASS: TestContractPointCacheValueShape (0.00s)
=== RUN   TestContractPointCacheRoundTripSameShape
--- PASS: TestContractPointCacheRoundTripSameShape (0.00s)
PASS
ok  	net.otlab/ingest/internal/cache	4.070s
=== RUN   TestContractTokenChainValidatesAndCachesPull
--- PASS: TestContractTokenChainValidatesAndCachesPull (0.00s)
=== RUN   TestContractTokenChainCacheTTLExpiryRepull
--- PASS: TestContractTokenChainCacheTTLExpiryRepull (0.06s)
=== RUN   TestContractTokenChainFailClosedOnPullFailure
--- PASS: TestContractTokenChainFailClosedOnPullFailure (0.00s)
=== RUN   TestContractTokenChainFailClosedOnUnknownGateway
--- PASS: TestContractTokenChainFailClosedOnUnknownGateway (0.00s)
=== RUN   TestContractTokenChainFailClosedOnMismatchAndEmpty
--- PASS: TestContractTokenChainFailClosedOnMismatchAndEmpty (0.00s)
=== RUN   TestContractCredentialPullHTTPRequestShape
--- PASS: TestContractCredentialPullHTTPRequestShape (0.01s)
=== RUN   TestContractCredentialPullFailClosedOnFailureResponses
--- PASS: TestContractCredentialPullFailClosedOnFailureResponses (0.01s)
=== RUN   TestContractCredentialPullBackendUnreachableTimesOut
--- PASS: TestContractCredentialPullBackendUnreachableTimesOut (0.05s)
PASS
ok  	net.otlab/ingest/internal/gwtoken	3.864s
=== RUN   TestContractMQTTAuthAllowWithTopicScope
--- PASS: TestContractMQTTAuthAllowWithTopicScope (0.00s)
=== RUN   TestContractMQTTAuthDenyOnWrongGatewayToken
--- PASS: TestContractMQTTAuthDenyOnWrongGatewayToken (0.00s)
=== RUN   TestContractMQTTAuthDenyOnPullFailureFailClosed
--- PASS: TestContractMQTTAuthDenyOnPullFailureFailClosed (0.00s)
=== RUN   TestContractMQTTAuthRejectsMissingOrWrongInternalToken
--- PASS: TestContractMQTTAuthRejectsMissingOrWrongInternalToken (0.00s)
=== RUN   TestContractMQTTAuthDenyOnEmptyUsernameOrPassword
--- PASS: TestContractMQTTAuthDenyOnEmptyUsernameOrPassword (0.00s)
=== RUN   TestContractMQTTAuthRealTokenChainFailClosed
--- PASS: TestContractMQTTAuthRealTokenChainFailClosed (0.01s)
=== RUN   TestContractReportEndpointContractSampleViaMockSink
--- PASS: TestContractReportEndpointContractSampleViaMockSink (0.00s)
=== RUN   TestContractReportEndpointRejectsOverLimitWith400ContractWording
--- PASS: TestContractReportEndpointRejectsOverLimitWith400ContractWording (0.00s)
=== RUN   TestContractReportEndpointRejectsEmptyPointsWith4001
--- PASS: TestContractReportEndpointRejectsEmptyPointsWith4001 (0.00s)
=== RUN   TestContractReportEndpointFailsClosedOnTokenFailure
--- PASS: TestContractReportEndpointFailsClosedOnTokenFailure (0.00s)
=== RUN   TestContractReportEndpointUnknownCodeDeadLetterViaRealPipeline
--- PASS: TestContractReportEndpointUnknownCodeDeadLetterViaRealPipeline (0.01s)
=== RUN   TestContractReportEndpointEndToEndWritesThroughPipeline
--- PASS: TestContractReportEndpointEndToEndWritesThroughPipeline (0.01s)
=== RUN   TestContractReportEndpointBatchIdIdempotentReplay
--- PASS: TestContractReportEndpointBatchIdIdempotentReplay (0.01s)
=== RUN   TestContractReportEndpointSinkError503
--- PASS: TestContractReportEndpointSinkError503 (0.00s)
PASS
ok  	net.otlab/ingest/internal/httpapi	3.592s
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
ok  	net.otlab/ingest/internal/message	2.785s
=== RUN   TestContractUnknownCodeDeadLetterNotWritten
--- PASS: TestContractUnknownCodeDeadLetterNotWritten (0.04s)
=== RUN   TestContractEmptyPointsRejected
--- PASS: TestContractEmptyPointsRejected (0.01s)
=== RUN   TestContractOverLimitRejected
--- PASS: TestContractOverLimitRejected (0.01s)
=== RUN   TestContractBatchIdDedup
--- PASS: TestContractBatchIdDedup (0.04s)
=== RUN   TestContractPointTsDedup
--- PASS: TestContractPointTsDedup (0.03s)
=== RUN   TestContractDisabledPointDiscarded
--- PASS: TestContractDisabledPointDiscarded (0.04s)
=== RUN   TestContractDispatchByDataTypeAndCacheUpdate
--- PASS: TestContractDispatchByDataTypeAndCacheUpdate (0.04s)
=== RUN   TestContractQualityJudgeHookApplied
--- PASS: TestContractQualityJudgeHookApplied (0.04s)
=== RUN   TestContractFlushBySizeWatermark
--- PASS: TestContractFlushBySizeWatermark (0.01s)
=== RUN   TestContractFlushByInterval
--- PASS: TestContractFlushByInterval (0.04s)
=== RUN   TestContractBufferCapDropsOldestAndCounts
--- PASS: TestContractBufferCapDropsOldestAndCounts (0.07s)
=== RUN   TestContractWriteFailureRequeued
--- PASS: TestContractWriteFailureRequeued (0.04s)
=== RUN   TestContractTenantFromConsumerRequired
--- PASS: TestContractTenantFromConsumerRequired (0.00s)
=== RUN   TestContractWriteRetriesExhaustedDeadLetter
--- PASS: TestContractWriteRetriesExhaustedDeadLetter (0.22s)
=== RUN   TestContractNonAsciiPointCodeEndToEnd
--- PASS: TestContractNonAsciiPointCodeEndToEnd (0.04s)
=== RUN   TestContractSubmitSyncEntryMirrorsRunPath
--- PASS: TestContractSubmitSyncEntryMirrorsRunPath (0.01s)
=== RUN   TestContractSubmitBatchIdDedupReportsDuplicate
--- PASS: TestContractSubmitBatchIdDedupReportsDuplicate (0.01s)
=== RUN   TestContractSubmitPointTsDupCountedAccepted
--- PASS: TestContractSubmitPointTsDupCountedAccepted (0.01s)
=== RUN   TestContractSubmitEnvelopeRejectedWhole
--- PASS: TestContractSubmitEnvelopeRejectedWhole (0.00s)
=== RUN   TestContractSubmitConcurrentWithRun
--- PASS: TestContractSubmitConcurrentWithRun (0.00s)
PASS
ok  	net.otlab/ingest/internal/pipeline	2.994s
=== RUN   TestContractQualityCodesZeroToTen
--- PASS: TestContractQualityCodesZeroToTen (0.00s)
=== RUN   TestContractQualityTextCodeMapping
--- PASS: TestContractQualityTextCodeMapping (0.00s)
=== RUN   TestContractQualityCalcErrNotMappable
--- PASS: TestContractQualityCalcErrNotMappable (0.00s)
PASS
ok  	net.otlab/ingest/internal/quality	2.713s
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
ok  	net.otlab/ingest/internal/snapshot	4.647s
=== RUN   TestContractRESTWriteFullyQualifiedNamesAndBasicAuth
--- PASS: TestContractRESTWriteFullyQualifiedNamesAndBasicAuth (0.01s)
=== RUN   TestContractRESTEnsureSchemaUsesUseMultiStatement
--- PASS: TestContractRESTEnsureSchemaUsesUseMultiStatement (0.00s)
=== RUN   TestContractRESTDefaultReadTimeoutIs30s
--- PASS: TestContractRESTDefaultReadTimeoutIs30s (0.08s)
=== RUN   TestContractRESTConfiguredShortTimeoutFails
--- PASS: TestContractRESTConfiguredShortTimeoutFails (0.30s)
=== RUN   TestContractRESTEnvErrorRecognition24And918
--- PASS: TestContractRESTEnvErrorRecognition24And918 (0.00s)
=== RUN   TestContractRESTSQLErrorNonEnvCode9750
--- PASS: TestContractRESTSQLErrorNonEnvCode9750 (0.00s)
=== RUN   TestContractRESTEnvFailSoftRetryCountThroughPipeline
--- PASS: TestContractRESTEnvFailSoftRetryCountThroughPipeline (0.03s)
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
=== RUN   TestContractChildTableNameProbePairNotColliding
--- PASS: TestContractChildTableNameProbePairNotColliding (0.00s)
=== RUN   TestContractChildTableNameInjective
--- PASS: TestContractChildTableNameInjective (0.00s)
=== RUN   TestContractDBNameInjective
--- PASS: TestContractDBNameInjective (0.00s)
=== RUN   TestContractProbePairDistinctInsertSQL
--- PASS: TestContractProbePairDistinctInsertSQL (0.00s)
PASS
ok  	net.otlab/ingest/internal/tdengine	4.859s
ACCEPT-3 EXIT=0
```

### 2.4 改动范围核对：`git diff --name-only d74d56f..HEAD` → 全落 ingest/**（EXIT=0）

```text
ingest/internal/gwtoken/gwtoken.go
ingest/internal/gwtoken/gwtoken_contract_test.go
ingest/internal/gwtoken/puller.go
ingest/internal/gwtoken/puller_contract_test.go
ingest/internal/httpapi/authmqtt.go
ingest/internal/httpapi/authmqtt_contract_test.go
ingest/internal/httpapi/mux.go
ingest/internal/httpapi/pipeline_sink.go
ingest/internal/httpapi/report.go
ingest/internal/httpapi/report_contract_test.go
ingest/internal/pipeline/pipeline.go
ingest/internal/pipeline/pipeline_submit_contract_test.go
ingest/internal/tdengine/rest.go
ingest/internal/tdengine/rest_contract_test.go
ACCEPT-4 EXIT=0
```

14 个文件全部在 `ingest/**` 内，越界 0。

## 3. 未做项与原因（brief 明确归后，非缺口）

| 未做项 | 归属 | 原因 |
| --- | --- | --- |
| EMQX 真回调联调（真认证回调载荷/主题授权 ACL 生效） | 0068 | brief 裁决 2：测试以自定契约锁形状，真 EMQX 联调归 0068 |
| TDengine 真库写入（EnsureSchema/INSERT 真执行） | 0068 | brief 裁决 4：fd 待修复，本任务以 httptest 假服务断言请求形态 |
| MQTT 四主题消费、快照拉取接线、端到端（MQTT↔HTTP 同构验证） | 0027 正式认领（0026 收口后） | brief 边界：本任务 0027「前端半」mock 先行 |
| 真网关令牌体系（SM4 密文解密、令牌归属通道校验、轮换并行） | 0020/0027 | brief 裁决 3：明文比对占位，接口化留位（Resolver/Puller/Validator 三接口） |
| `cmd/ingestd/main.go` 接线 | 0027 | README 口径「占位，TASK-0027 接线」；`httpapi.Mux` 已提供装配函数，接线即组装 |
| 校验/质量判定完整逻辑（量程二次判定、时间纠偏） | 0028 | 0073 口径沿用 PassthroughJudge |
| 崩溃安全幂等（batchId/(pointCode,ts) 真存储实现） | 0028 | 沿用 0073 内存假实现（内核简化口径） |
| TC-0009 步骤 8 跨租户快照隔离的完整演示 | 0027 | 需按租户分片快照缓存的部署架构（0027 定版）；本任务令牌链已保证租户来自令牌侧、未知编码经单快照缓存死信路径已锁 |
| `-race` 并发检测 | （环境限制） | 本机无 gcc（cgo 缺失）race 检测器不可用；并发安全由 `TestContractSubmitConcurrentWithRun`（32 并发 Submit vs Run，计数严格分批）行为断言覆盖 |

## 4. 偏离计划之处

1. **pipeline 内核增加 `Submit` 同步入口（非纯增量装配，但为行为保持重构）**：计划中 HTTP 上报走「统一流水线入口」，若经消息通道（Run 消费循环）异步处理，契约 §3.2（续）承诺的响应「拒绝明细（未知编码、时间戳越界）」无法同步返回，且会出现第二份五步顺序逻辑（漂移风险）。故在 `pipeline.go` 内为 `processOne` 增加返回值并暴露 `Submit`（与 Run 共用同一实现），外加 `StartFlusher`（纯 HTTP 接线场景单独启动 flusher）。0073 既有 55 项契约测试零改动全绿，处理顺序单点实现未复制。
2. **HTTPPuller 带自有超时（默认 10s）**：0073 `snapshot.HTTPFetcher` 范式为可注入 `http.Client`（默认无超时）；令牌链 fail-closed 要求后端不可达在有限时间内转为拒绝，故 `NewHTTPPuller` 内建超时（可配置）。
3. **响应/回调形状为自定契约**（brief 授权「测试以自定契约锁形状」）：
   - 上报响应 `{"accepted","rejected","rejections":[{pointCode,ts,reason}],"duplicate"}`（契约 §3.2 续只定内容不定字段名）；信封级错误 `{"code","message"}`（code 照 `message.Error` 纪律：4001 原样、未定码 0 不臆造）；令牌失败 401、入口不可用 503。
   - 认证回调 `{"result":"allow"|"deny","topics":{publish[],subscribe[]}}`（EMQX 惯例 result 键 + 主题范围扩展）；端点自身内部令牌失败 401 + 响应体 deny 双信号。
4. **受理计数口径（自定契约的一部分）**：QoS1 重报（同 batchId / 同 (pointCode,ts)）按「成功无操作」计入 accepted 并以 `duplicate:true` 标记整批重放——幂等不是拒绝，避免网关把正常重投当失败；拒绝只计死信/丢弃点。

## 5. 遗留问题

1. **工作区存在并行会话的未提交改动**（`backend/.../dataperm/*.java` 修改、`backend/.../openapi/` 未跟踪目录，属并行的 backend 0051 道）：与本次改动零交集，本次未触碰、未提交；请协调者知悉，后续提交归位各自道。
2. **令牌链凭据缓存 TTL 内无法即时感知吊销**：吊销立即生效口径依赖 TTL 到期重拉（默认 30s，可配小）；如需强即时需 0020 定版失效通知机制（缓存失效/版本比对）。
3. **auth/mqtt 的主题范围目前为响应字段**（EMQX ACL 生效需 0068 配置 HTTP ACL 回调或映射规则）；0068 联调时若 EMQX 版本要求独立 ACL 端点，可基于同一令牌链新增回调面（接口已留位）。
4. **`ingest/ingestd.exe` 为本地构建产物**（gitignore 未跟踪、未提交），与仓库无关。
5. **race 检测未跑**（本机无 C 编译器）；建议在具备 gcc 的环境对 `pipeline`/`httpapi` 补一次 `go test -race`（Submit 与 Run 双入口并发是重点面）。

## 可追溯性

- **git 提交**：`f0f4533`（main，仅 ingest/**，注明 TASK-0076；曾为 c4a519a，amend 仅修正提交信息中新增测试计数 42→34，文件与范围未变）；修订轮 `4b2d59d`（见 §6）
- **修改文件**：§1 清单 14 项（10 新增源码/测试文件 + 1 内核修改 + 3 新增测试文件全部列名）；修订轮 7 项（见 §6.4）
- **验收**：§2 四条命令原始输出，全部通过；修订轮补充验收见 §6.3

## 6. 修订轮（§13 验收退回：评审 Critical 1 / Important 3 / Minor 8 deferred）

- 评审报告：`workitems/reports/TASK-0076-review.md`（fresh reviewer，对象 f0f4533；C1 有 overlay 探针三组复现数据：800 轮×24 goroutine 下同 batchId 被当新批 8-11 次、唯一 (pointCode, ts) 键重复写出 5-7 行）
- 修订轮提交：`4b2d59d`（仅 ingest/** 7 文件，注明 TASK-0076 修订轮）
- 修复范围：必修四项 C1/I1/I2/I3 全部闭合；Minor 8 条按裁决记 deferred 不动

### 6.1 修复内容

**C1（dedup TOCTOU 破坏 QoS1 幂等）**
- `pipeline.go`：去重挂点接口收敛为原子 `MarkIfSeen`——`BatchDeduper`（SeenBatch+MarkBatch 两段式 → 单方法）与 `PointDeduper`（SeenPoint+MarkPoint → 单方法）各自一次调用完成首见判定+标记（单锁内，含 FIFO 淘汰）；`MemoryBatchDeduper`/`MemoryPointDeduper` 实现同步收敛；`processOne` 两处调用点改原子口径。接口文档明示：真实存储实现（TASK-0028）必须保持同原子语义，不得降级为先查后写两段式。
- `Submit` 并发注释改为真实承诺：原子性前提逐项列明（去重挂点 MarkIfSeen 单锁原子、攒批缓冲/统计锁内原子、SnapshotLookup 读原子快照、DeadLetterSink 调用方保证并发安全）——承诺在 MarkIfSeen 原子化后成立才写。
- 竞态回归测试（探针思路转正，新增文件 `pipeline_submit_race_contract_test.go`）2 项：
  - `TestContractSubmitRaceSameBatchIdExactlyOnce`：30 轮 × 12 goroutine 同刻齐发同 batchId，每轮恰 1 个 non-duplicate、11 个 Duplicate；累计账面 ReceivedBatches=360、DuplicateBatches=330、AcceptedPoints=30（零重复入攒批）；
  - `TestContractSubmitRaceSamePointTsExactlyOnce`：30 轮 × 12 goroutine 同 (pointCode, ts) 跨批并发重报（互异 batchId），累计 AcceptedPoints=30、DuplicatePoints=330，写出行数收敛恰 30（唯一键恰一次写出）。
  - 稳定性：`-run 'Race' -count=3` 与 `-count=10` 复跑均全绿（修复前评审探针 300/800 轮即可复现）。

**I1（令牌缓存全局锁横跨网络回源）**
- `gwtoken.go`：`credential()` 改 per-key singleflight——同键并发回源收敛为一次 Pull（等待者复用结果），异键互不阻塞；Pull 移出元数据锁（锁只护 cache/inflight 短临界区）；拉取失败不缓存口径不变。已知取舍（singleflight 惯例：等待者复用发起者结果，含其 ctx 取消错误）已注明。
- 新增测试 2 项：`TestContractTokenChainPerKeyParallelism`（A 键回源悬挂期间 B 键缓存命中 300ms 内返回——修复前该断言实证失败「B 键缓存命中超时」）、`TestContractTokenChainSameKeySingleflight`（同键 8 并发恰 1 次 Pull）。

**I2（401/503 回显内部错误细节）**
- `report.go`：401/503 对外固定文案常量「网关令牌校验失败」/「服务暂不可用」；err 细节（后端 status/业务 message/响应体原文——`Result<CredentialPullVO>` 含 ciphertext，mock 口径下即期望网关令牌；sink 错误含内部拓扑）只进服务端日志（`ReportHandler.Logger` 可注入，nil → log.Default()）。
- 新增测试 1 项：`TestContractReportEndpointFixedExternalErrorMessages`——401 响应体不含 `SECRET-MATERIAL`（潜在密文材料）、message 恰为固定文案、日志承载细节；503 同口径（不含 `internal-detail-X`）。

**I3（认证回调缺网关 status 校验且注释失实）**
- 代码改动=注释修正+报告登记（按裁决，status 实校验不强制本期实现）：
  - `authmqtt.go` 包注释改为实际行为（校验网关编码与设备令牌；通道/凭据禁用经后端 403 覆盖；网关级禁用会被 allow——缺口明示）；
  - `gwtoken.go` `GatewayIdentity` 扩展位注释：需后端 credential/pull 响应扩 gateway status 字段（归 0027 后端接线）后在此承载并在回调实校验；
  - 登记义务：**0027 后端接线时扩 credential/pull 响应 gateway status 字段 → 0020/0027 定版网关令牌体系时于 GatewayIdentity 承载并在 auth/mqtt 实校验**（本报告 §5 遗留问题同步登记）。

### 6.2 修订轮改动文件清单（7 个，全落 ingest/**）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `ingest/internal/pipeline/pipeline.go` | 修改 | C1：去重挂点接口/实现/调用点收敛为原子 MarkIfSeen；Submit 并发注释真实化；包文档同步 |
| `ingest/internal/pipeline/pipeline_submit_race_contract_test.go` | 新增 | C1 竞态回归 2 项（30 轮 × 12 goroutine 齐发，恰一处理断言） |
| `ingest/internal/gwtoken/gwtoken.go` | 修改 | I1：credential() per-key singleflight（inflight 表 + 锁外回源）；I3：GatewayIdentity 扩展位注释 |
| `ingest/internal/gwtoken/gwtoken_contract_test.go` | 修改 | I1：per-key 并行断言 + 同键 singleflight 断言 2 项 |
| `ingest/internal/httpapi/authmqtt.go` | 修改 | I3：流程注释改为实际行为 + 缺口登记（网关 status 归 0020/0027） |
| `ingest/internal/httpapi/report.go` | 修改 | I2：401/503 固定文案 + Logger 字段 + err 细节进服务端日志 |
| `ingest/internal/httpapi/report_contract_test.go` | 修改 | I2：固定文案/零泄露/日志承载断言 1 项 |

### 6.3 补充验收原始输出（三条全过）

**① `cd ingest && go build ./... && go vet ./...` → 双零退出**

```text
（无输出，两命令零退出；回显确认行）
REV-ACCEPT-1 BUILD+VET EXIT=0
```

**② `cd ingest && go test ./... -count=1` → 全绿（含 C1 竞态回归 / I1 per-key 并行 / I2 固定文案）**

```text
?   	net.otlab/ingest/cmd/ingestd	[no test files]
ok  	net.otlab/ingest/internal/cache	5.440s
ok  	net.otlab/ingest/internal/gwtoken	6.516s
ok  	net.otlab/ingest/internal/httpapi	6.231s
ok  	net.otlab/ingest/internal/message	3.833s
ok  	net.otlab/ingest/internal/pipeline	6.035s
ok  	net.otlab/ingest/internal/quality	2.318s
ok  	net.otlab/ingest/internal/snapshot	6.003s
ok  	net.otlab/ingest/internal/tdengine	6.645s
REV-ACCEPT-2 EXIT=0
```

C1 竞态回归稳定性补跑（-count 迭代断言零复现）：

```text
go test ./internal/pipeline/ -run 'Race' -count=3  →  ok  net.otlab/ingest/internal/pipeline	2.174s
go test ./internal/pipeline/ -run 'Race' -count=10 →  ok  net.otlab/ingest/internal/pipeline	2.172s
```

**③ `cd ingest && go test ./... -run 'Contract' -v` → 契约测试 94 项全 PASS（89+新增 5）**

新增 5 项逐项：

```text
=== RUN   TestContractSubmitRaceSameBatchIdExactlyOnce
--- PASS: TestContractSubmitRaceSameBatchIdExactlyOnce (0.01s)
=== RUN   TestContractSubmitRaceSamePointTsExactlyOnce
--- PASS: TestContractSubmitRaceSamePointTsExactlyOnce (0.01s)
=== RUN   TestContractTokenChainPerKeyParallelism
--- PASS: TestContractTokenChainPerKeyParallelism (0.00s)
=== RUN   TestContractTokenChainSameKeySingleflight
--- PASS: TestContractTokenChainSameKeySingleflight (0.02s)
=== RUN   TestContractReportEndpointFixedExternalErrorMessages
--- PASS: TestContractReportEndpointFixedExternalErrorMessages (0.00s)
```

全包统计（`-run 'Contract' -v` 汇总）：`--- PASS` 94 行、`--- FAIL` 0 行；8 包 ok：

```text
?   	net.otlab/ingest/cmd/ingestd	[no test files]
ok  	net.otlab/ingest/internal/cache	5.083s
ok  	net.otlab/ingest/internal/gwtoken	5.935s
ok  	net.otlab/ingest/internal/httpapi	5.603s
ok  	net.otlab/ingest/internal/message	1.463s
ok  	net.otlab/ingest/internal/pipeline	5.753s
ok  	net.otlab/ingest/internal/quality	1.313s
ok  	net.otlab/ingest/internal/snapshot	5.908s
ok  	net.otlab/ingest/internal/tdengine	5.972s
```

逐项清单与首轮 §2.3 相同（89 项全 PASS）加以上 5 项；本次修订未删除或改名任何既有契约测试（0073 的 55 项与首轮新增 34 项全部原样通过）。

### 6.4 遗留登记（修订轮更新）

- **I3 缺口（登记归 0020/0027）**：网关 status 校验需后端 credential/pull 响应扩 gateway status 字段（0027 后端接线）→ GatewayIdentity 扩展位承载 → auth/mqtt 实校验。本期通道/凭据禁用经 403 覆盖，网关级禁用会 allow。
- **Minor 8 条**：按裁决记 deferred（recover 双写头部+panic 值外泄、RejectedWhole→503 语义错位、puller 空令牌注释与行为不符、鉴权前全量解析+双重解析、双 flusher 仅注释防守、主题范围无通配符防线、accepted≠落库的缓冲丢最旧静默缺口、REST 客户端 code=0+非 200 混码与部署硬化注记）——收尾进 LOG。
- **race 检测**：本机仍无 gcc（cgo 缺失）；C1 修复的 `-race` 回归面建议在有 gcc 环境补跑（评审建议同此口径）。
- 并行 backend 会话工作区改动（dataperm/application.yml/redline 等）与本修订轮零交集，未触碰、未提交。

## 7. 修订轮可追溯性

- **git 提交**：`4b2d59d`（main，仅 ingest/** 7 文件，注明 TASK-0076 修订轮）
- **核对**：`git diff --name-only f0f4533..HEAD` → 7 文件全落 `ingest/**`（见 §6.2 清单），越界 0
