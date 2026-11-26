# TASK-0028 diff 评审报告（代码评审）

- 评审对象：提交 `8a837aa` + `b4e877a`（BASE=`cebf17a`，18 文件全落 `ingest/**`）
- 评审口径：`L1-execute.md` §13「diff 评审」+ 评审 brief 评审重点 1-9
- 评审者：fresh 子代理（只依据评审 brief「必读」与工作区源码判断；未派 subagent；未修改任何业务文件）
- 评审日期：2026-11-23

## 0. 越界反查（复核）

- `git diff --name-only cebf17a..b4e877a -- ingest/` = 18 文件，与 diff 包一致。
- 同区间非 ingest 文件 7 个（全部 backend/ot-admin），经 `git log` 归属并行道提交 `a0a92b7`（TASK-0079）/`9197e7e`（TASK-0080），与本任务无关——与评审 brief「越界反查」预判一致，不列入本评审。
- 工作区 `ingest/` 与 `b4e877a` 零 diff（评审所读源码即被评审提交）。

## 1. 评审重点逐条核对

### 1.1 量程判定 vs CONV-02 冻结枚举（重点 1）——落实正确

- 方向核实：`internal/quality/range.go:27-40` MergeRange 为 value>max→`OverMax`(1)、value<min→`UnderMin`(2)，与冻结枚举 `internal/quality/quality.go:14-15`（1=超上限、2=超下限）及 `FromTextCode` OVER_RANGE 映射同向；brief 行文反向已在执行者报告 §4.1 与 range.go 包注释登记为勘误，以冻结枚举为准——处置正确，未触发「报告并停」（CONV-02 枚举本身无问题，quality.go 本批零改动）。
- 边界：==min/==max 不越界（`value > *maxValue` / `value < *minValue` 严格不等号），契约测试 `TestContractQualityRangeBoundaryInclusiveKeepsGatewayQuality` + 流水线级 `TestContractRangeJudgeBoundaryInclusive` 双锁定。
- 缺失透传：`*float64` nil 判空（快照 Schema minValue/maxValue 可空），双侧缺失整跳过、单边按存在侧判定（`TestContractQualityRangeMissingBoundsPassthrough`）。
- 原值保留：`pipeline.go:495` 行构造 `Value: pt.Value` 原值、缓存 `updateCache` 同值——SQL `VALUES (ts,150,1)`/`(ts,-10,2)` 与缓存 `V=-10 Q=2` 断言齐备，无假 0。
- 越域码防御收敛：非超限且网关码越出 0-10 → `Good`(0)（`TestContractQualityRangeInvalidGatewayQualityNormalized`），超限覆盖写优先于收敛（越域码+越限值仍打越限码，有测试）。

### 1.2 quality 合并规则合并矩阵（重点 2）——完整

网关先填+快照 min/max 二次判定覆盖写（统一消息模型 §3.2 逐字口径）：

| 网关码 \ 判定 | 越限 | 范围内/边界 | min/max 缺失 |
| --- | --- | --- | --- |
| 有效码 0-10 | 覆盖写 1/2（3 例测试） | 保留网关值（测试） | 透传（测试） |
| 越域码（99/-1/11 等） | 覆盖写 1/2（测试） | 收敛 Good(0)（测试） | 收敛 Good(0)（注释口径） |

「网关没填」= JSON quality 缺省 → Go 零值 0 = Good，协议无「缺省」标记位，矩阵自然闭合，无缺口。

### 1.3 时间三分裁决（重点 3）——三路径边界正确、去重键一致性锁定

- 边界值：`timepolicy.go:51-67` judge 严格不等号——ts==now+5m→纠偏（`bandUpperMs` 测试）、+5m+1ms→ts_overflow、ts==now-30d→豁免（`maxAgeEdgeMs` 测试）、-1ms→ts_too_old、ts==now→豁免。与裁决字面（「>」才拒绝）一致。
- 路径 1 越界拒绝：阈值 env 配置化（`INGEST_TS_FUTURE_REJECT`/`INGEST_TS_MAX_AGE`，`cmd/ingestd/main.go:124-133`；pipeline `Config.TSFutureReject/TSMaxAge`，Assemble 零值取默认、负值报错）；拒绝不落库、拒绝明细携带消息原始 ts（`TestContractTimeRejectionDetailCarriesMessageTs` + live 冒烟 B3/B4）。
- 路径 2 未来窄带纠偏：rowTs=接收时间、原始 ts 不落库（测试双断言）、`Stats.TimeCorrected` 独立计数器 + `CorrectionTagger` 采样 WARN（`[ts-corrected]` 固定格式）；CONV-02 码域未动（quality 枚举零改动），显式「已纠偏」标记按 brief 走契约变更另议（报告 §5.1 登记）。
- 路径 3 补传豁免：原始 ts 落库不纠偏不打标（`TestContractTimeBackfillExemptionKeepsOriginalTs`），§1.9A「原始 ts 不可被纠偏破坏」锚点在 timepolicy.go/pipeline.go javadoc 登记。
- **去重键一致性**：`pipeline.go:444` 先存 `origTs`，纠偏只改 `rowTs`，`pipeline.go:472` `MarkIfSeen(pt.PointCode, origTs)` 取消息原始 ts；`TestContractTimeDedupKeyedOnOriginalTs` 锁定未来漂移点跨批重报恰一落库。一致。

### 1.4 死信统一通道（重点 4）——四要素齐备且有界

- 原因码对齐：`DeadLetterReason`（string 型）同源用于死信出口与 `SubmitRejection.Reason`——ts_overflow/ts_too_old/unknown_code/format_invalid/disabled_point/invalid_config/invalid_envelope 与 README 登记一致；HTTP 拒绝明细与死信计数同字符串（`TestContractDeadLetterReasonEnumValues` + wiring 测试逐 reason 断言）。
- 租户+原因隔离计数：`deadKey{tenant, reason}` 有界 map（默认 Cap=1024，FIFO 淘汰最旧键，ADR-0014 口径，淘汰/重计测试覆盖）。
- WARN 采样防风暴：首条 + 每 N 条（`n==1 || n%N==0`），N=INGEST_DEADLETTER_WARN_EVERY env 可配默认 100；固定格式含 tenant/reason/count/pointCode/batchId（N=100 与 N=3 两档测试 + live 冒烟 N=2 实证 count=3 不出日志）。
- 环形缓冲排查读口：固定容量切片（默认 128）构造时一次分配，`Recent(n)` 时序读出（容量 3 收 5 条→[P3,P4,P5] 测试）——无泄漏、有界。

### 1.5 四步校验顺序与去重位置（重点 5）——嵌入位置正确

- 实际顺序（`pipeline.go:409-499`）：查快照/未知编码死信 → 禁用点位 → 格式（点位级 isFinite 防御）→ 量程（judge 挂点）→ 时间（三分裁决）→ 去重（MarkIfSeen）→ kind → 写行。冻结五步骨架未动（令牌验签在 Consumer/httpapi 面、分发写库+缓存为第 5 步），四步嵌入位置与 README/pipeline.go 包注释声明一致。
- 格式步先于时间步：NaN+陈旧 ts → format_invalid 不被 ts_too_old 遮蔽（`TestContractFormatStepPrecedesTimeStep`）。
- 去重靠后口径：同 batchId 重报整批幂等吸收（零逐点、零新增死信，`TestContractSameBatchReplayNoDuplicateDeadLetter` + live B1 重报）；跨 batchId 重报已受理点过四步后被点级去重吸收（`TestContractCrossBatchDuplicateNoDeadLetter`）。

### 1.6 ADR-0009 边界注记（重点 6）——四处登记，无入口

pipeline.go 包注释（「本流水线不提供任何人工补录入口」）、cmd/ingestd/main.go 头注释、ingest/README.md、quality/range.go 包注释（MANUAL 码按「网关先填」透传）四处登记；httpapi 未新增任何人工补录端点（diff 仅测试注入时钟）。

### 1.7 墙钟定时炸弹拆除（重点 7）——注入完整、断言未弱化

- 5 处验证 harness 注入 `Now=1795212000000`（=统一消息模型契约样例 sentAt，见 unified-message-model.md:26）：httpapi realPipelineFixture、mqttsource twopath、pipeline 0073 harness、ttl race harness、tdengine fail-soft——grep 全仓核对：**全部 6 个 `pipeline.Assemble` 测试装配点均含时钟注入，无遗漏**。
- 残留炸弹排查：其余含硬编码样例 ts 的测试文件（message/cache/gwtoken/tdengine/dedup_ttl/mqttsource 单组件测试）不经流水线时间裁决，无墙钟依赖。
- 断言弱化评估：时钟注入仅改变裁决基准、既有断言零改动；`TestContractBufferCapDropsOldestAndCounts` 以放开阈值（24h/100 年）替代注入，理由成立（该测专查攒批 FIFO，纪元 ts 与时间裁决正交，注释已说明）。

### 1.8 Race 竞态定位（重点 8）——测试侧竞态实锤，修复不掩盖缺陷

`pipeline_submit_race_contract_test.go:117-135`：`st` 在 Submit 循环后捕获，AcceptedPoints/DuplicatePoints 为 Submit 同步递增（该两断言用 `st` 有效）；WrittenPoints 由 flusher 异步递增——旧断言用收敛前 `st` 读 WrittenPoints 属测试侧 stale 读。修复后复核取收敛后新快照。恰一写出三重锁定充分：① AcceptedPoints=30（同步账面，零重复入攒批）② DuplicatePoints=330（同步账面，幂等键恰一放行）③ eventually 收敛 WrittenPoints==30 + 收敛后复核——若真有重复写出，②/③ 必失败，故非流水线缺陷被掩盖。`-count=10×2 + -count=3` 稳定复跑留痕（执行者报告 §2.1）。

### 1.9 零回归与依赖纪律（重点 9）——复跑实证

- 独立复跑（本评审机）：`go build ./... && go vet ./...` 双零退出；`go test ./...` 10 包全 ok；`-run Contract -v` 计数 **PASS=168 / FAIL=0**——与执行者报告一致。
- 既有测试改动面核对（对照 diff）：既有文件触及 7 处——5 处单行时钟注入、1 处阈值放开（1.7 已述）、1 处 race 复核修正（1.8 已述）、main_test.go 既有两测为**增量断言**（新键位默认值/env 覆盖值，原有断言未动）。既有 135 项核心断言语义零改动、全绿——「零回归」成立；「零改动」按字面不成立但全部改动必要（时间裁决默认开启使旧硬编码 ts 在 2026-12-23 后必入 ts_too_old，注入是确定性修复非回避）、且已在报告 §4.2/§4.4 透明登记，不构成缺陷。
- 零新增依赖：go.mod/go.sum 在 cebf17a..b4e877a 零 diff。
- gofmt：`gofmt -l` 告警 9 文件全部**不在**本任务 18 文件清单内（0027 既有状态/并行道文件），与报告 §4.6 回退声明吻合；本任务文件零格式告警。

## 2. 发现清单

无 Critical、无 Important。Minor 5 条（建议记收口 deferred，不进修订轮）：

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | `ingest/internal/pipeline/pipeline.go:32`（对照 `:478-485`） | 包注释「被拒点（时间越界/未知编码等）不标记去重键」对 invalid_config 不成立——kind 查找在 MarkIfSeen 之后（0027 既有顺序本批未改），该类被拒点已标记去重键、跨 batchId 重发会被吸收而非按新事件计数；注释以「等」概括与实现不一致，建议改为点名枚举或把 kind 查找挪到去重前 |
| 2 | Minor | `ingest/internal/pipeline/timepolicy.go:84-100` | CorrectionTagger 纠偏 WARN 采样周期固定 100、无 env 配置位，与死信侧 INGEST_DEADLETTER_WARN_EVERY 形成运维控制面不对称（brief 裁决 3 仅要求死信 N 配置化，非违约，登记性缺口） |
| 3 | Minor | `ingest/internal/pipeline/deadletter.go:128-136`（调用点 `:103`） | appendOrder 每次 OnDeadLetter 对 order 做线性扫描（≤Cap=1024），死信风暴场景每事件 O(Cap) 比较；量级可接受，演进位可用 map 索引降为 O(1) |
| 4 | Minor | `ingest/internal/quality/range.go:27-40` | MergeRange 未防御退化配置 max<min（快照 Schema 层亦未约束 min≤max）：此时 value>max 恒判 OverMax、value<min 恒判 UnderMin，判定次序隐含 OverMax 优先；极端配置防御缺口，建议快照装载侧补 min≤max 校验或注释声明假设 |
| 5 | Minor | `ingest/cmd/ingestd/main.go:124-141` | INGEST_TS_FUTURE_REJECT / INGEST_TS_MAX_AGE / INGEST_DEADLETTER_WARN_EVERY 非法值（解析失败或 ≤0）静默回落默认、无告警日志——与既有 SNAPSHOT_INTERVAL/DEDUP_TTL 解析风格一致，但时间阈值属脏数据防线参数，误配置不易察觉，建议回落时打一行 WARN |

## 3. 执行者登记的裁决留白（复核确认为留白非缺陷）

- §5.2 被拒点跨 batchId 重发按新死信事件计数（观测口径）：与实现一致（TimeReject/format_invalid/unknown_code 均不标记去重键，仅 1.9-发现#1 所述 invalid_config 例外为 0027 既有）。
- §5.3 纠偏计数含被去重吸收的重报点（纠偏步在去重步前）：实现与登记一致，语义=「执行过的纠偏次数」；落库恰一由去重键保证（测试锁定）。

## 4. 评审复跑证据（本评审机独立执行）

```text
go build ./... && go vet ./...      → 无输出，EXIT 0
go test ./...                        → 10 包全 ok（ingestd/httpapi/mqttsource/pipeline/tdengine/snapshot 等含新跑 6 包）
go test ./... -run Contract -v       → --- PASS 168 行 / --- FAIL 0 行
gofmt -l ./cmd ./internal            → 9 文件均非本任务 18 文件清单内
git diff b4e877a -- ingest/          → 空（工作区=被评审提交）
```

## 5. 结论

**通过**——评审重点 1-9 逐条核对全部落实且与契约/裁决口径一致（量程冻结枚举方向勘误处置正确、合并矩阵完整、时间三路径边界与去重键一致性锁定、死信通道四要素有界、四步顺序嵌入正确、ADR-0009 登记、墙钟炸弹拆除彻底、Race 修复为测试侧且三重锁定充分、168 契约测试独立复跑全绿零新增依赖）；无 Critical / Important 发现，Minor 5 条建议记 TASK 收口 deferred。
