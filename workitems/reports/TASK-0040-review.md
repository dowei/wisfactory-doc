# TASK-0040 diff 评审报告（代码评审）

- 评审对象：提交 `04e6acd`（BASE `537366e`，49 文件全部 `calc/**`）——计算引擎服务引导：新顶层 Maven 工程 + 快照消费 + 依赖图 + 新图原子切换 + ACK 回执闭环
- 评审口径：`.agents/protocol/L1-execute.md` §13「diff 评审」；fresh 评审（仅依据 brief 指到的文件与工作区源码）；no-subagents；只评审不改文件
- 评审日期：2026-11-23
- 输入：diff 包、实施 brief、执行者报告、TASK-0040 详情、FEAT-0013/REQ-0020/TC-0013/ADR-0008、`config-snapshot-schema.md` v1.0.1、上游 `IotPublishInternalController.java` / `AckForm.java` / `InternalSnapshotVO.java` / `IotPublishLogServiceImpl.java` / `PublishAckTimeoutScanner.java`

## 结论：通过

无 Critical / Important。8 个评审重点逐项核验均满足验收标准与契约；发现 4 条 Minor（均为防御纵深 / 工程卫生 / 文档一致性，不影响有效数据下的正确性与本次验收）；1 项根因「无法判定」（ACK 首次未受理，calc 侧行为正确，登记观察即可，无需为 calc 新建 BUG）。按 §13，Minor 记入 TASK `任务结果` 的 deferred（收尾进 LOG），不进修订轮。

---

## 一、发现清单

| # | 严重级 | 出处（文件:行） | 一句话说明 |
| --- | --- | --- | --- |
| F1 | Minor | `calc/src/main/java/net/otlab/calc/graph/GraphBuilder.java:239`（对照 `FailureReason.java:18`、`GraphBuilder.java:60-63`） | 类注/FailureReason 声明 `DUPLICATE_NODE` 覆盖「fullKey/metricCode 重复」，但仅 fullKey 做了存在性判重；指标节点用 `nodes.putIfAbsent(metricCode, …)` 静默保留首个重复 metricCode，不抛 `DUPLICATE_NODE`。仅在违反契约唯一性的坏快照下静默合并两指标的边，有效数据不受影响；属防御纵深声明与实现不一致，缺一条重复 metricCode 的负向测试。 |
| F2 | Minor | `calc/src/main/java/net/otlab/calc/runtime/SnapshotReconciler.java:223`（对照 `CalcEngineApplication.java:85-91`、`SnapshotReconciler.java:101`） | `destroy()` 未被 Spring 调用：`@Bean` 默认 `destroyMethod` 只推断 `close`/`shutdown`（不推断 `destroy`），且无 `@PreDestroy`/`DisposableBean`/`AutoCloseable`。javadoc 所称「Spring 容器关闭钩子」实际未接线，优雅停机未显式 `shutdownNow()`；因调度线程为 daemon（:101）JVM 仍可正常退出、无挂起，故仅文档/意图不一致，低风险。 |
| F3 | Minor | `calc/src/test/java/net/otlab/calc/CalcEngineApplicationTest.java:28,42`（对照 `calc/src/main/java/net/otlab/calc/config/ServerPortCustomizer.java:26-28`） | `@SpringBootTest(RANDOM_PORT)` 的随机端口被 `ServerPortCustomizer` 的 `factory.setPort(8095)` 覆盖（自定义izer无 `@Order`、默认最低优先级，在 `ServerProperties` 置 0 之后执行），实际绑定固定 8095（执行者报告 surefire 段亦显示「Tomcat started on port 8095」）。测试丧失端口隔离：8095 被占用时该用例失败；读取 `local.server.port` 仍取到实际端口，故当前环境通过。 |
| F4 | Minor | `calc/src/main/java/net/otlab/calc/snapshot/HttpSnapshotFetcher.java:142-145`（对照 `GraphBuilder.java:1258`） | 快照体 version < 信封 version 时仅 WARN 接受，偏离 brief 决策 2 字面「信封与快照体 version 同值校验」。有 TASK-0024 回滚复用裁决依据、危险方向（体>信封，:138）仍 fail-closed，并有契约测试 `bodyVersionBelowEnvelopeAcceptedAsRollbackShape` 覆盖，属已记录的有意偏离。附带一致性：`DependencyGraph` 的 version 取体 version（:1258）而 `AppliedGraph` 取信封 version，回滚形态下两者不同值；本任务未消费前者，暂无功能影响。 |

### 无法判定项

- **ACK 首次未受理根因（评审重点 8）**：报告记载 v41 首次失败回执出现一次 `ackFail=1`（09:59:07「未受理」），次周期即成功落库（id=22，`ack_status=2`+errorMsg）。仅凭 diff/报告无法判定是 backend 侧瞬时原因还是环境/DB 可见性窗口（需该时刻 backend 日志）。**评估**：从源码可确认 calc 侧行为正确——`HttpAckSender.send` 非 200/网络失败返回 false、`SnapshotReconciler` 计数告警且不改图状态（fail-soft），且失败版本每轮对账都会重发 ACK（版本仍 > 当前、构建仍失败 → 再次 `ackFailure`），天然重试使次周期自愈。结论：按执行者建议**登记观察即可，无需为 calc 新建 BUG 档案**；若后续接入 config.changed 高频触发再评估 ACK 重试退避。
- 执行者报告「既有快照含悬空引用种子 `R2_DEAD2→R2_DEAD1`」：报告与 diff 一致（calc 对其 fail-closed 为正确行为），属配置治理侧遗留数据，非本任务缺陷；无需判新缺陷。

---

## 二、评审重点逐项核验

1. **快照消费与哈希**：通过。`HttpSnapshotFetcher.fetch()` 带 `X-Internal-Token`（与上游 `InternalAuthChecker` 头一致）、`scopeType=2&scopeId=0[&tenant]`；`RawJsonExtractor` 逐字节提取 `data.snapshot`（不树再序列化，保 `5.000000` 形态）；`Hashes.sha256Hex` 对原文重算与 `snapshotHash` 比对，不一致/哈希形态非法/HTTP 或业务码非 200/data 缺失/体版本未来漏出全部 fail-closed 转 `SnapshotFetchException`；契约测试 + live 冒烟（envelope hash == recomputed）双证。错形态防御见 `checkHashForm` 与 `HttpSnapshotFetcherContractTest`（9 例含 hashMismatch/malformed/missingSnapshot/businessCode/bodyVersionGreater）。符合 BUG-0005 修复后口径。
2. **依赖图构建**：通过。五对象消费数据项/公式(引用边)/指标（README/契约差异第 11 条说明模型域不装载）；节点=fullKey/点位编码/metricCode，边型 `FORMULA_REF`/`POINT_REF`/`ROLLUP`/`METRIC_SOURCE` 映射正确；四类失败主因（悬空引用、实例化失败、循环、节点键冲突-fullKey）均 fail-closed 且有负向测试（含 `rollupFormulaCycle`、`identifierWithNodeButWithoutEdge` 漂移、`integralFormulaFails`、`pointRefNeverDangling`）；白名单 DELTA 可引用、INTEGRAL 显式禁用（`FormulaExpressionValidatorTest` 17 例）；空快照构建空图有测试；refType=2 点位按派生规则 7 不判悬空（合规）。唯一缺口见 F1（metricCode 判重）。
3. **新图原子切换**：通过。`GraphHolder` 用 `AtomicReference<AppliedGraph>.set()` 单写多读引用替换，读者无中间态；`DependencyGraph` 用 `Map.copyOf`/`List.copyOf` 不可变；FAIL 路径（`reconcileOnce` catch 块 :184-191）不调用 `switchTo`，旧图持续可用；并发读图/切换线程安全由 AtomicReference + 不可变对象保证；`CalcStatusController` 以 `GraphHolderSnapshot.of` 单读快照防撕裂。live 冒烟 `status.version=40` 全程未变、`replaced=1`、`buildFailures=3`，实证配置错误未打停计算。
4. **ACK 回执闭环**：通过。`targetType=3`、`targetId=CALC_TARGET_ID`（默认 calc，与 backend 上游 `ot.publish.target-calc-id` 派生口径同源）、`ackStatus=1/2`、`errorMsg`（失败必填、截断至 1000 对齐 `AckForm @Size`）；成功/失败两路径均发 ACK（:183/:190）；失败 errorMsg 含失败原因与定位；ACK 自身失败 fail-soft（返回 false→`incAckFail`+日志，不改图）；payload 字段与 `AckForm` 逐字段对齐并由 `HttpAckSenderTest` 假端点契约测试覆盖（含 data=false 受理、业务码非 200 返回 false、errorMsg 超限截断、生效回执省略 errorMsg）。live 冒烟 id=21（`ack_status=1`）、id=22（`ack_status=2`+errorMsg）双落库，成功行未被倒退。
5. **配置纪律**：通过。端口 8095（未占 8098/8099/8090），`CALC_HTTP_ADDR` 默认 `:8095`、`ServerPortCustomizer` 覆盖 `server.port`、`application.yml` 兜底 8095；env 键完整（BACKEND_BASE_URL/INTERNAL_TOKEN/TENANT/TARGET_ID/SNAPSHOT_RECONCILE_SECONDS/HTTP_ADDR）且启动 fail-fast 校验 baseUrl、缺 token/tenant WARN；`config.changed` 订阅缝 `ConfigChangedSubscriber` NoOp 占位按 TD-0057 登记、拉取驱动（启动即拉+默认 300s 对账）；优雅停机见 F2（daemon 保证可退出，显式钩子未接线）。
6. **测试有效性**：通过。`mvn test` 115 测试 0F0E0S（报告原始 surefire 段逐类列出）；契约命名（`SnapshotParserContractTest`/`HttpSnapshotFetcherContractTest`）fixture 取自 v1.0.1；live 冒烟 ACK 双落库与单测决策矩阵断言一一对应（首拉成功/幂等跳过/新版本切换/构建失败保旧/同版本漂移/版本回归/拉取失败/ACK 发送失败计数/调度生命周期）。偏离：仅重跑 `mvn test` 未单独留证 `mvn install`（`install` 产物 jar 已由 live 冒烟消费，源码未变），执行者已记为「偏离计划之处」，接收。
7. **新工程质量**：通过。pom 仅 `spring-boot-starter-web` + `spring-boot-starter-test`，Jackson 3（`tools.jackson`）走 Spring Boot 4 传递依赖、未显式引入 Jackson 2，Java 21，符合 TD-0058；README 体例（目录/配置/消费语义/构建与检查/边界）照 ingest；`.gitignore` 覆盖 `target/`/`**/target/`（报告说明避免构建产物入库，合理）；配置默认值安全（baseUrl 空白拒启、令牌/租户缺省仅 WARN 依赖服务端 fail-closed）。
8. **执行者登记项评估**：见「无法判定项」——悬空引用种子为治理侧遗留、calc fail-closed 正确；ACK 首次未受理根因无法判定但 calc 侧 fail-soft + 天然重试正确，登记观察即可。

## 三、越界反查复核

协调者已比对、评审者复核：`git diff --name-only 537366e..04e6acd` 49 项全部落 `calc/**`（含 `.gitignore`/`README`/`pom`/`src`），未触碰 backend/frontend/ingest/workitems/契约文件，符合 brief 授权范围与 §10。工作区并行道未跟踪文件未 add/未提交（报告实证），遵守 `git add calc/**`。
