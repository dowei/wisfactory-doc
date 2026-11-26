# 评审报告：TASK-0073 diff 评审（fresh，角色=代码评审）

- 评审对象：`TASK-0073-diff.txt`（= `git diff d971c10..cff0642`，本任务唯一提交 cff0642，15 文件）
- 依据：L1-execute §13 diff 评审口径；TASK-0073 详情与验收标准；实施 brief（改动范围/接口与决策 1-6）；三份冻结契约（unified-message-model v1.1.0 / config-snapshot-schema v1.0.1 / tdengine-super-tables v1.0.0）；conventions.md CONV-02；关联 FEAT-0009 / REQ-0013 / TC-0009（验证测试断言出处用）
- 方法：diff 包 3339 行全量读完（未抽样）；关键声明（范围、依赖、测试）在本地复跑核实

## 0. 复跑核实（评审者独立执行，非转抄报告）

| 命令 | 结果 |
| --- | --- |
| `git log --oneline -3` / `git diff --name-only d971c10..cff0642` | 提交链 c1b2615→d971c10→cff0642 与报告一致；15 文件全部 `ingest/**`，非 ingest 文件 0（越界反查通过） |
| `cd ingest && go build ./... && go vet ./...` | 退出码 0，无输出 |
| `go test ./... -count=1` | 6 包全 `ok`，0 FAIL |
| `go test ./... -run 'Contract' -v -count=1` | **44 项 PASS / 0 FAIL**（报告 §2.3 结论行写「36 项」，与其自身输出和实测不符，见 M8） |
| `cat go.mod` | module `net.otlab/ingest`，require 段空，零第三方依赖（brief 决策 4 通过） |

## 1. 发现清单

### Critical（0 条）

无。

### Important（2 条）

1. **I1 · 快照字段级校验未实现契约 Schema 的 required 清单**
   - 出处：`ingest/internal/snapshot/snapshot.go`（ParseSnapshot + Point/Channel/DataItem/Formula/Metric 五个 `Validate`）vs `resources/contract/config-snapshot-schema.md` 各对象 JSON Schema 的 `"required"`（point: id/tenantId/pointCode/pointName/dataType/valueType；channel/dataItem/formula/metric 各自清单）。
   - 说明：Validate 只做 enum + maxLength，无 required 存在性校验——缺 `pointName`、缺 `pointCode`（空串通过 maxLength）、缺 `id/tenantId` 等的畸形快照会被判「有效」并原子替换缓存，§3.3「新快照校验失败则继续用旧版」防线对这一类输入静默失效；报告 §1 只声明 enum/maxLength/additionalProperties 三项口径，未把 required 缺口列为偏离或遗留。

2. **I2 · 子表名拒绝路径在写库层：契约合法的 pointCode 可造成整批无限重排、绕过死信口径**
   - 出处：`ingest/internal/tdengine/tdengine.go`（safeIdentifier/childTable/Writer.WritePoints）+ `ingest/internal/pipeline/pipeline.go`（flush/requeueFront）vs config-snapshot-schema.md point 对象（pointCode 仅 `maxLength 96`，无字符集约束）。
   - 说明：含连字符等非 `[A-Za-z0-9_]` 字符的 pointCode 按契约 Schema 合法、可通过快照校验进入缓存，但到 `childTable` 才被拒 → `Writer.WritePoints` 中途返回错误 → flush 整批重排回缓冲头部，每过一个 FlushInterval 重试一次（无重试上限、无退避），同 flush 内已成功下发的其他分组被反复重排重写；流水线自己定义的 `DeadLetterInvalidConfig`（「点位配置非法→死信计数告警」）对此类不可达——拒绝点在写库层而非第 5 步构造行时。

### Minor（9 条）

1. **M1** · `ingest/internal/snapshot/snapshot.go`：`Channel.Validate` 未校验 `status` 枚举、`DataItem.Validate` 未校验 `enabled` 枚举（契约 Schema 均为 enum [0,1]），声明口径内（enum+maxLength）的枚举覆盖不全。
2. **M2** · `ingest/internal/snapshot/snapshot.go`：`Cache.Replace` 只复验 points 数组，channels/dataItems/formulas/metrics 不复验，与注释「校验并原子换入」口径不符；绕过 `ParseSnapshot` 直接构造的 `Snapshot` 对其余对象无校验入口。
3. **M3** · `ingest/internal/pipeline/pipeline.go`：`SnapshotLookup.LookupPoint(pointCode)` 与 batchId/(pointCode,ts) 去重键均不含租户，TC-0009 步骤 8「跨租户编码判死信」依赖「一 Pipeline 实例 = 一个租户/网关快照范围」的组装假设，包文档未声明该假设，0027 组装单实例承载多租户时会静默串租户。
4. **M4** · `pipeline.go` + `tdengine.go`：Writer 分组串行下发、中途失败即返回，flush 整批重排会把已 Exec 成功的分组一并重写；失败注入测试仅单分组场景，多分组部分成功路径未覆盖，收敛性依赖 0029 真 TDengine 对同（子表, ts）的覆盖语义（契约未定）。
5. **M5** · `ingest/internal/snapshot/fetch.go`：`HTTPFetcher` Client 为 nil 时回退 `http.DefaultClient`（无超时），快照端点异常时调用方无限期阻塞；0027 接线须显式配超时。
6. **M6** · `ingest/internal/pipeline/pipeline.go`：`Run` 因消息通道关闭返回时 flusher goroutine 不退出（仅随 ctx 退出），调用方只关通道不取消 ctx 会泄漏；Run 注释未声明此生命周期。
7. **M7** · `ingest/internal/tdengine/tdengine.go`：状态/开关量写 `st_point_state`（value INT 列）未校验 value 整数性，float64 直接 `'g'` 格式化写入，异常小数将进 INT 列（真执行时的行为契约未定义）。
8. **M8** · `TASK-0073-report.md` §2.3：结论行「共 36 项 Contract 命名断言」与其自身粘贴输出不符——实测 44 项 PASS（本评审复跑核实）；报告数字错误，不涉实现。
9. **M9** · `ingest/internal/message/message.go`：注释把「空数据=空 points 4001」归属 REQ-0013，REQ-0013 正文无 4001（权威出处为 FEAT-0009「异常与边界」；TC-0009 步骤 6 引「REQ-0013 验收 3」同样无此码）；实现取值本身正确（4001 与 FEAT-0009 一致）。

## 2. 留白确认（非缺陷，按 brief 口径标注）

- **哈希复算未做**：§3.3「校验哈希」的哈希算法上游未定版，FetchResult 仅透传 snapshotHash；报告遗留 1 已声明，0027 收口。
- **instance_key 空串占位**：快照点位对象无 instance_code 字段（上游留白）；报告遗留 2 已声明。
- **快照响应信封包装键名 `"snapshot"` 为拟定**：契约只承诺 version/snapshotHash/快照体三部分内容、未定键名；报告偏离 2 已声明。
- **子表命名方案 = 点位编码为实现口径**：契约未定版命名方案（报告偏离 3 已声明）；其拒绝路由缺陷单列 I2，命名本身不判缺陷。
- **消息模型未知字段宽松**：契约未对消息模型规定 additionalProperties，与快照严格口径的差异为有意为之（报告遗留 6 已声明）。
- **推送频率等契约未定字段**：未做假设，符合 brief 决策 1。

## 3. 一致性正面确认（与 TASK 验收标准 / brief 接口逐项对上）

- **改动范围**：15 文件全落 `ingest/**`（本地核实），README/go.mod 之外未触碰 backend、frontend、resources、application*.yml；提交信息注明 TASK-0073。
- **契约符合性（brief 决策 1）**：消息解析样例逐字取自 §3.2 JSON 示例并逐字段断言（含 tenant/gateway/batchId/sentAt/points[].ts/value/quality）；空 points 4001（FEAT-0009 口径）、恰 500 放行 / 501 整批拒绝（契约批量上限 + REQ-0013 验收 3）；ts 与 sentAt 分开保留；CONV-02 质量码 0-10 枚举与文本码映射（GOOD→0、OVER_RANGE→1/2、STALE/OFFLINE→3、JUMP→8、MANUAL→5、CALC_ERR 不映射）逐字一致；INSERT 形态（子表 USING 超级表、TAGS 值顺序=附录 B、同子表多行合并 VALUES）与附录 B DDL（归一空白后逐句）一致；标签取自快照不取消息自带字段；每租户一库 `iot_<tenant_code>`。
- **brief 接口与决策 2/5**：八接口（Consumer/Parser/SnapshotLookup/BatchDeduper/PointDeduper/DeadLetterSink/Writer/PointCache）+ QualityJudge 函数挂点齐备；处理顺序五步一致（含租户只信任令牌侧、未知编码死信不落库、禁用点位丢弃计数）；攒批水位默认 500、定时默认 200ms（FEAT-0009 100-500/200ms）；BufferCap 有界、满丢最旧并计数（ADR-0014 FIFO 口径）；写失败整批重排且受同上限约束；单消费循环 + 单 flusher 并发模型成文于 doc comment。
- **决策 3/4**：假 EMQX=内存 channel、假 TDengine=记录式执行器（含时间戳）、假快照端点=httptest 回放契约样例；go.mod 零第三方依赖（未引 miniredis，理由在报告偏离 1 成立）。
- **验收命令**：build/vet 干净、测试全绿、Contract 断言 44/44（评审者复跑）；报告五项齐备、TDD 红绿记录含「为什么现在对了」。
- **遗留与偏离声明**：HTTP 同步响应缺口、通知消费循环、changes/ack 调用、-race 未跑（本机无 gcc）等均已如实声明且归属清晰。

## 4. 结论

- Critical 0 条，Important 2 条（I1 快照 required 校验缺失、I2 子表名拒绝路径在写库层致整批无限重排并绕过死信口径），按 L1-execute §13 进入退回路径；Minor 9 条记 `任务结果` deferred，不进修订轮。
- 契约符合性与 brief 接口 overall 一致、验收命令实测全绿；两条 Important 均为实现侧缺陷（非计划缺陷），修复面集中在 snapshot 校验与 childTable 拒绝路由，修复后可复评。

VERDICT: FAIL

## 复评（round 2）

- 复评范围（scoped）：修复提交 `c2ff33e` 的 6 个 ingest 文件（`pipeline.go` / `pipeline_contract_test.go` / `snapshot.go` / `snapshot_contract_test.go` / `tdengine.go` / `tdengine_contract_test.go`），只核对 I1/I2 修复与协调者裁决语义的符合性 + 新缺陷，不重开全场（round 1 的 Minor 不复审）。
- **范围勘误（diff 包 vs 提交）**：`TASK-0073-fixdiff.txt` 实为 `git diff cff0642..c2ff33e`，该**区间**含同期 backend 道提交 `384d58f`（TASK-0005 数据权限组件，24 个 backend 文件）——包内 backend hunks 属他人任务，不纳入本轮评审。修复提交 `c2ff33e` 本身经 git 核实恰 6 个 ingest 文件、0 越界；「6 文件全落 ingest/**」对提交成立、对 diff 包不成立（打包用了区间而非 `git diff cff0642..c2ff33e -- ingest/` 或 `git show c2ff33e`）。非 TASK-0073 缺陷，知悉即可。
- **独立复跑**：`go build ./... && go vet ./...` 干净；`go test ./... -count=1` 6 包全绿；`go test ./... -run 'Contract' -v` **51/51 PASS**（44 旧 + 7 新，与提交说明一致）。M8 勘误已确认：TASK-0073-report.md §2.3 已加勘误注（36→44→51）。

### 原发现结论

1. **I1（快照 required 校验缺失）→ 已修复（关闭）**
   - 证据：五对象 required 清单与契约逐字一致（snapshot.go `pointRequired` 等 5 个清单 vs config-snapshot-schema.md 各对象 JSON Schema `"required"`，含 channel 的 hasCredential、metric 的 7 字段逐字核对）；`ParseSnapshot`/`ParsePoint` 等 6 个解析入口在原始 JSON 层做键缺失/显式 null 拒收（`checkRequiredRaw`/`checkArrayRequired`，null 元素同样被拒）；pointCode 空串视为缺失，且 `validateRequiredString` 在 Validate 层覆盖绕过 JSON 解析直接构造结构体的路径；缺 pointName 的点位使快照体整体拒收、`Cache.Replace` 拒收并保旧版（`TestContractPointObjectRequiredEnforced` / `TestContractAllFiveObjectsRequiredEnforced` / `TestContractSnapshotCacheKeepsOldOnRequiredViolation` 实测通过）。契约示例样例无一误伤（全绿）。

2. **I2（子表名拒绝路径 + 无限重排）→ 裁决两个面均已按语义落地，但 I2-1 修复引入 1 条新 Important（N1）**
   - I2-1 修复面成立：`childTable` 改 `ChildTableName` 确定性全映射（全安全字节恒等、非安全字节逐字节 `_XX` 十六进制转义 + FNV 后缀、首数字 `t_` 前缀、超长截断至 192 字节），契约合法 pointCode（连字符/中文/数字开头，含 96 字中文上限）端到端不再被写库层拒绝，TAGS 保留原编码（`TestContractNonAsciiPointCodeEndToEnd` / `TestContractPointCodeMappedNotRejected` / `TestContractChildTableNameEscaping` 实测通过）；`safeIdentifier` 保留为映射后不变式断言——与裁决语义一致。
   - I2-2 修复面成立：行级重排上限 3 次（`maxWriteRequeues=3` + `bufItem.requeues` 跨 flush 保留），超限按 `DeadLetterInvalidConfig` 死信计数 + 新增 `WriteRetriesExhausted` 计数、不再回灌（`TestContractWriteRetriesExhaustedDeadLetter` 实测：3 次重排后第 4 次失败死信、后续周期计数稳定不增长）；单次失败仍整批重排，原 `TestContractWriteFailureRequeued` 语义保持；与去重「接受即标记」交互无回归（死信行已标记，不重消费）；BufferCap 丢最旧口径不变——与裁决语义一致。

### 新发现

**Important（1 条）**

1. **N1 · `escapeIdentifier` 转义集与恒等集存在构造性碰撞，映射在契约合法域上不单射（含租户库域）**
   - 出处：`ingest/internal/tdengine/tdengine.go` 的 `escapeIdentifier` / `ChildTableName` / `DBName`（本修复引入；DBName 转义为裁决外的对称扩展，同类继承）。
   - 说明：转义路径输出 = 转义前缀 + `"_" + FNV64(原文)`，该输出本身是全 `[A-Za-z0-9_]` 字符串，故「字面编码恰好等于另一编码的完整转义结果」走恒等路径映射到同一标识符。评审在仓库外临时副本以探针实测复现：pointCode `"F01-WS01"` 与 `"F01_2dWS01_2aad8c0256245aa0"`（后者全安全、契约合法、27 字节）同映射子表名 `F01_2dWS01_2aad8c0256245aa0`；租户 `"t-1"` 与 `"t_2d1_5627091943b2a601"` 同库名 `iot_t_2d1_5627091943b2a601`。后果分级依据：同 flush 碰撞对触发 `InsertPointSQL`「混入不同点位」错误，重排 ≤3 次后死信（有界，但丢的是合法数据）；分 flush 写入时，0029 真接入后 TDengine 对已存在子表忽略 TAGS → 后写者行落先写者子表、`point_code` 标签错归属（静默串数据）；库域碰撞使两租户收敛同一库，破附录 B「每租户一库」隔离纪律。当前 mock 内核不落真实数据，但映射函数是本任务交付物且在声明域（契约合法编码全集）上不单射，裁决 I2-1「恒等集区分」对该构造族不成立。可修方向（0029 定版命名方案时落地）：转义集定长化（恒等像 ≤98 字节 vs 转义像恒 192 字节，长度域不相交即单射）或放弃恒等、对全部编码统一编码。

**Minor（2 条）**

2. **N2** · `ingest/internal/snapshot/snapshot.go`：required 非空校验从裁决点名的 pointCode 一刀切推广到全部必填字符串（pointName/channelName/dataCode/fullKey/refFullKey/metricCode/metricName）——契约 Schema 无 minLength，空串严格说合法；方向 fail-closed、契约样例零误伤，属超出裁决字面的从严，0027 接线若遇上游确有空名快照需回看此口径。
3. **N3** · `ingest/internal/pipeline/pipeline.go` `requeueFront`：重排超限死信记录 `OnDeadLetter(DeadLetterInvalidConfig, tenant, "", nil)` 不携带 batchID 与点位上下文（bufItem 未存），运维告警只能定位到租户粒度。

### 复评结论

- 原 I1 已修复关闭；原 I2 两个裁决面均已按语义落地且实测通过；修复引入新缺陷 N1（映射不单射、构造性碰撞、探针实测复现，point 与租户库两域）。
- 本轮计数：Critical 0 / Important 1（N1）/ Minor 2（N2、N3）。
- 修订轮上限（1 退回 + 1 复评）已用满，按 L1-execute §12 定性分流：N1 为实现缺陷，建 BUG 挂 FEAT-0009（附本节探针复现与可修方向），TASK-0073 照常收尾；N2/N3 记 TASK `任务结果` deferred（收尾进 LOG）。

VERDICT: FAIL
