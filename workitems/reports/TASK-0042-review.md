# TASK-0042 diff 评审报告（代码评审）

- 评审者：fresh 评审子代理（no-subagents；只依据评审 brief 指到的文件 + 工作区源码判定，不共享协调者上下文）
- 评审口径：`.agents/protocol/L1-execute.md` §13「diff 评审」（与 TASK 验收标准 / brief 接口一致性 + 明显质量缺陷）
- 输入：
  - diff 包 `reports/TASK-0042-diff.txt`（e49fea8..e387c3b，24 文件限 `calc/**`）
  - 实施 brief `reports/TASK-0042-brief.md`；执行者报告 `reports/TASK-0042-report.md`；TASK 详情 `tasks/TASK-0042.md`
  - 决策 FEAT-0013 / REQ-0020 / REQ-0019 / TC-0013 / ADR-0010
  - 上游 reports/TASK-0041、TASK-0049（含修订轮 R1-R5）+ 工作区 `calc/src/**`
  - 权威：CONV-02（workitems/conventions.md:10）、config-snapshot-schema.md:165、tdengine-super-tables.md:15/36-37、详细设计 V1.1 §1.6（PDF）
- 被评审提交：`e387c3b`（父提交 `e49fea8`）；HEAD 实测 = `e387c3b`。
- 评审日期：2026-11-24

---

## 一、越界反查复核（评审重点 8 前半）

- `git show --name-only e387c3b` 实测 24 项，**全部 `calc/**`**：README 1 + main（application/配置/executor×3/operator×4/storage×5/web）13 + resources/application.yml 1 + test 8（config 1 改 + executor 1 + operator 3 + storage 2 + 集成 1 ... 实测清单见 diff 包）。越界 0。
- `e49fea8`（TASK-0050，backend/** 状态读口）是 `e387c3b` 的**父提交**，不在本提交内：协调者「区间内 backend 已隔离」的判断成立。
- 结论：**越界反查通过**。注：执行者报告 §首部把 BASE 记为 `331f860`，而 diff brief 与提交父节点均为 `e49fea8`；报告 §验收 3 / 遗留 6 已自行澄清「按提交哈希而非区间首末判断归属」，属报告字段口径不一致，不影响越界判定（见 Minor M-3）。

## 二、评审重点逐条核对

### 1. DELTA 五异常语义 —— 基本一致（1 处端到端缺口见 M-1）
`DeltaOperator.evaluate`（diff 325-324 行区）逐条落地：正常差分 `value=当前-上周期`（NORMAL/0）；回退 `diff<0 → 0 + 码8`（BACKFLOW，不写负值）；换表 `markParamChange`/`paramChangeEvent` 命中即重建基线、跨段不差分（NEW_BASELINE/0，与 ADR-0010 TD-0019「PARAM 变更时间点即分段边界」一致）；溢出 `|diff|>maxRange → 0 + 码1`（OVERFLOW）；跨周期 `gap > expectedPeriodMs×1.5 → 0 + 码9` 并重建基线（CROSS_PERIOD，恢复后从最近样本起算）；坏样本保留上游码。与 CONV-02（0 正常/1 超上限/8 突变待核查/9 跨周期用量）逐条对齐，质量码无冲突。契约测试 6 项 + live 生产类探针覆盖。
- **缺口**：溢出码 1 在执行器主链不可达（`maxRange` 恒传 `null`，见 M-1）；live 端到端 quality 序列仅出现 0/5/8/9。

### 2. INTEGRAL 三场景 —— 基本一致（1 处语义缺口见 M-2）
`IntegralOperator.accept`（diff 617-640 行区）：真实 ts 间隔梯形 `((v0+v1)/2)*dt`（测试 `(2+4)/2*3=9` 与 `(4+4)/2*2=8` 逐段锁定，非固定周期）；缺口 `quality!=0`（当前或上一跳）→ GAP_SKIPPED、累计不变、重建基线（测试码 3 覆盖，不拿最后流量填满）；停机 `0+0 → STOPPED_ZERO/quality=0`，过渡段仍按梯形计增量，与断线 GAP_SKIPPED + 质量码严格区分。`isHandwritable()=false` 与 0040 白名单（`FormulaExpressionValidator.WHITELIST={DELTA:1}`、`BANNED={INTEGRAL}`，`GraphBuilder:145-146` 在快照侧强制调用）共同保证 ADR-0010「INTEGRAL 不开放手写」，集成测试 `handWrittenIntegralIsRejectedByWhitelist` 实证。
- **缺口**：「样本缺失间隔跳过」仅由 quality 非 0 表达，纯时间戳跳变且 quality=0 时不跳过（见 M-2）。

### 3. 同构写纪律 —— 通过
`TdengineItemValueWriter.insertSql`（diff 1206-1217 行区）：`fullKey = instanceCode + "." + dataCode`，**无 `full:` 前缀**，与冻结契约 config-snapshot-schema.md:165 及读侧等值查询一致（0049 F-02 教训已吸收）；六元组 tag 顺序 tenant_id/full_key/instance_code/data_code/category/source_type 与 tdengine-super-tables.md:36-37 完全一致，契约测试 `tagsSixTupleUsesContractFullKeyWithoutPrefix` **逐值精确断言**（`tags[1]` 精确等于规范值，非 `contains` 形态断言）；逐语句单 INSERT（0 分号、无 USE、库名全限定、子表首写 USING TAGS，0029 纪律）；`requireTaosSuccess` 正则解析 taosAdapter code（0049 F-01 已吸收，冒号后空格变体 + 错误码 fail-closed 离线单测）。`HomogeneousResultSink` 把成功结果按 `sourceType`（1/2/3）同构写 + Redis `iot:item:{tenant}:{fullKey}` JSON；失败结果不写假 0；TDengine/Redis 单条写失败 WARN 隔离。status 留痕（0049，fullKey=`instance.STATUS`，source_type=1）与算子结果写（本任务，各自数据项 fullKey）键空间不同，不冲突。

### 4. 派生项换向 —— 通过（契约测试层面）
`EnergyDerivation`（diff 450-541 行区）：绑定方向 CUMULATIVE→DELTA / FLOW→INTEGRAL，`Direction.FORWARD/REVERSE` 决定 ±sign，`apply(v)=sign*v`；反绑只翻符号，算子与 `derivedDataCode()` 不变（契约测试逐项断言）；`Template.resolve` 实例 null 继承模板默认、显式覆盖生效，与 0081 三方合并口径一致。**限制**：`EnergyDerivation` 在 `calc/src/main` 中除自身外零引用（实测 grep），换向仅由契约测试/live 探针以生产类直跑锁定，未接入执行器派生链（报告 未做项 3 同源）。按期口径「预置公式落地后接线」属登记演进位，不构成越界或错值。

### 5. 失败语义 —— 通过
算子异常/失败结果不写假 0（`HomogeneousResultSink.accept` 对 `!r.isSuccess()` 只 WARN 跳过，契约测试 `failedResultNeverWritesFakeZero`）；单节点失败隔离沿用 0041（无算子桥时 DELTA 隔离失败，集成测试 `withoutOperatorBridgeDeltaStillIsolatedAsFailure`）；TDengine/Redis 单条写失败只 WARN 不阻塞、不上抛（`storeFailureDoesNotBlockOtherResultsOrThrow` / `redisCacheFailureAlsoIsolated`）。**null-metadata NPE 修复充分**：live 首轮 `[CALC-ARRIVAL] 事件解析失败 ... "this.metadata" is null` 为真红，`HomogeneousResultSink` 构造器 `metadata == null → defaultMetadata()` 兜底（diff 771-778 行区）+ 回归测试 `nullMetadataFallsBackToDefaultProjection` 锁定，live 重跑 writes 逐事件递增，根因（传 null 元数据 → 异步订阅器 fail-soft 静默吞掉）与回归覆盖均到位。

### 6. opt-in 接线 —— 通过
`CalcProperties` 三 env（operators-enabled/homogeneous-storage-enabled/tenant-id=1）默认关（`CalcPropertiesTest.task0042DefaultsKeepExistingBehavior` + 0049 的 163 基线逐类零回归实测）；`storageRuntime` Bean 仅任一开关为真才装配，`storageDecorator` 仅 `homogeneousStorageEnabled` 时注入。协同顺序（实测源码）：`ExecutorRuntime` 先套 status 桥（`sinkDecorator`）再套同构写（`storageDecorator`），链为 `HomogeneousResultSink → BridgingSink → base sink`；同批数据先写数据项结果（各自 fullKey）后触发状态判定（STATUS fullKey），两者写口/fullKey 不同，**无重复写、无顺序冲突**。`operatorsEnabled=true` 但 `homogeneousStorageEnabled=false` 时仅接算子桥、不写同构（独立开关语义自洽，非缺陷）。

### 7. 测试有效性 —— 见第三节专门判定
194 = 163 基线 + 31 新增，逐类求和实测 194、Skipped 3（0041 门控 1 + 0049 门控 2），BUILD SUCCESS 与报告一致；22/22 live 断言覆盖 DELTA 五异常 + INTEGRAL 三场景 + 派生换向 + tag/full_key SQL；红转绿留痕含 NPE 修复。**TDengine 真机写入+读回未验真——证据充分性判定见下。**

### 8. 零回归与范围 —— 通过
163 基线逐类零回归（报告 Results 段逐类与 TASK-0049 修订轮 R2.1 一致）；24 文件全 `calc/**`，backend 0050 未触碰（见第一节）。

---

## 三、特别判定：评审重点 7 —— TDengine 真机未验真的证据充分性

**判定：充分（在环境阻断前提下达到可达上限），不构成 Critical / Important；与 0049 修订后同口径一致。** 但「真实写入可用」必须在补验前保持「未验真（环境阻断）」标记，不得据本报告宣称真机已通。

判定依据（4 项替代证据 + 1 项禁止性条件均满足）：

1. **full_key 契约由精确断言锁定**（0049 F-02 直接教训）：`tagsSixTupleUsesContractFullKeyWithoutPrefix` 对六元组 `tags[0..5]` 逐值断言、`tags[1]` 精确等于 `instance_code + "." + data_code`、全文无 `full:`；非 0049 初版那种 `contains("'status'")` 形态断言（该形态正是当初漏掉 `full:` 前缀的根因）。
2. **taosAdapter 真应答解析离线锁定**（0049 F-01 直接教训）：`requireTaosSuccess` 从不可测 HTTP 闭包抽为静态可测方法，`restParsesTaosAdapterResponses` 用真应答样本（含 `{"code": 0}` 冒号后空格变体、错误码 9728、空串）验证 code 判定 fail-closed；`writeCapturesSingleSqlViaInjectableRunner` / `writeFailurePropagatesForCallerWarnNotBlock` 锁定单语句与失败上抛语义。
3. **SQL 形态在 live 链路经生产类捕获**：`StorageRuntime.RecordingWriter.write()` 调用真 `TdengineItemValueWriter.insertSql()`，live 步骤 3/4/5 逐事件 quality 0/5/8/9 落 SQL，六元组、无前缀、0 分号、全限定库名逐条核对，`/calc/v1/status.storage` 提供观测面。形态侧闭合。
4. **真 Redis 点位缓存确有真实验证**（区别于 TDengine）：live 真 SET/GET `iot:item:t42:F01.WS01.EQ001.FLOW`，JSON 与跨周期 SQL 同值同码，步骤 6 DEL 清理。
5. **禁止性条件已落实**：报告 §一环境事实、§三未做项 1、§五遗留 1 与 §三末补验条件 1-5 明确「当前未验真、不宣称已验真」；补验序列覆盖建 scratch 库 → 生产 `rest()` 真写 → `WHERE full_key='F01.WS01.EQ001.FLOW'` 读回断言 value/quality(8/9)/source_type=2/六元组 → 同键重写覆盖语义 → DROP 库，与 0049 R4 同构且完整。

**边界说明（不构成重要缺陷，但需如实知悉）**：真写口 `TdengineItemValueWriter.rest()` 的 HTTP POST / Basic 认证 / 超时 / `requireTaosSuccess` 调用链在本次交付中**从未对任何服务实际执行过**——live 全程走 `RecordingWriter`（仅复用静态 `insertSql`），契约测试走注入 `SqlRunner`；离线只验证了 `requireTaosSuccess` 静态方法。因此上述「充分」仅指**代码形态与解析语义的证据充分**，不指「真机连通性与写入效果已验真」。这正是补验条件不可省略、且在此之前必须维持「未验真」标记的原因。环境为服务器侧故障（curl `SELECT 1` HTTP=000、6s reset，TD-0053 登记），非实现问题，按 brief 决策 6 与 0049 先例不阻断代码收尾。

---

## 四、发现清单

| 级别 | 编号 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| **Important** | I-1 | `calc/src/main/java/net/otlab/calc/executor/DualTriggerExecutor.java:192-195, 229-246`（`containsFunction` + `evaluateDeltaFormula`），对照 `calc/src/main/java/net/otlab/calc/graph/FormulaExpressionValidator.java:24, 154-163` 与 REQ-0019「用量÷周期时长=平均流量」 | DELTA 公式被 `containsFunction` 广谱命中后，`evaluateDeltaFormula` 用正则只取首个 `DELTA(x)` 就直接 `CalcResult.success(key,2,sample)` **丢弃表达式其余部分**——0040 校验器已接受单参 `DELTA(x)/3600`（其 javadoc 自称支持「DELTA(x) 作为更大表达式的一部分」），会被静默算成原始差分、无质量码，违背 REQ-0019「用量÷周期时长=平均流量」与 CONV-02「不写假值」/0041「函数调用隔离失败」纪律；属静默错值而非隔离失败。缓解：本期预置为纯 DELTA、报告已登记遗留 5；最小在范围修正为对非纯 `DELTA(x)` 公式 fail-closed 报错（无需 FEAT-0012）。 |
| Minor | M-1 | `DualTriggerExecutor.java:246`（`operators.delta(..., periodMs, null, false)`），对照 `DeltaOperator.java:98, 140-145` 与报告 §三未做项 2 | DELTA 溢出码 1 在执行器主链恒不可达（`maxRange` 硬编码 `null`，快照无满量程字段），五异常中的「溢出归零」仅在算子契约测试/live 生产类探针覆盖，端到端 live quality 序列只有 0/5/8/9；报告已如实登记为演进位。 |
| Minor | M-2 | `calc/src/main/java/net/otlab/calc/operator/IntegralOperator.java:75-85`（`accept` 仅查 `quality!=0`），对照 brief 重点 2「样本缺失间隔跳过」 | INTEGRAL 缺口跳过只认质量码非 0，不像 DELTA 那样有时间容差检测：两样本均 quality=0 但 ts 大幅跳变（样本缺失未补标）时仍按真实间隔整段积分，存在「跨缺口累计」风险；依赖上游断线补标 quality=3 的约定，且 INTEGRAL 主链未接线，影响有限（上游缺口表示契约是否保证补标，工作区资料无法确认，标**无法判定**）。 |
| Minor | M-3 | `reports/TASK-0042-report.md` §首部 BASE=`331f860` vs 评审 brief BASE=`e49fea8` / 提交父节点=`e49fea8` | 报告 BASE 字段与 diff brief/提交父节点不一致（报告 §验收 3 已说明按提交哈希判断归属），属可追溯性字段口径瑕疵，不影响越界与零回归结论。 |

**Critical：无。**

---

## 五、结论与建议

- **整体结论：不通过（1 Important，按 L1-execute §13 走 §12 退回一轮；复评 scoped 只看 I-1 修复 diff）。**
  - 必改：**I-1**——DELTA 非纯调用公式的静默错值。建议按 fail-closed 处置：`evaluateDeltaFormula` 严格限定公式必须整体匹配 `DELTA(key)`（可允许包裹空白），否则抛异常由单节点失败隔离（不打假值），并补一条红转绿契约测试（如 `DELTA(x)/3600` 必须隔离失败，而非写出原始差分）。如协调者确认该形态完全由 0081 预置公式约束且不可能出现，可降级为 deferred 并登记「校验器已接受但执行器错算」的风险，但需在报告/ADR 留痕。
  - deferred（不进修订轮）：M-1 / M-2 / M-3 记 TASK `任务结果`，收尾进 LOG。M-1 待快照契约增加满量程字段（CONV-03）后接 `maxRange`；M-2 待确认上游缺口补标契约后决定是否需要 ts 跳变保护。
- **评审重点 7 明确判定：证据充分（环境阻断下的可达上限），与 0049 修订后同口径，不阻断收尾；真机写入+读回保持「未验真（环境阻断）」，补验条件 1-5 为环境恢复后强制交付后条件，补验前不得宣称「真实写入可用」。**
- 修订轮上限 1 轮退回 + 1 轮 scoped 复评（L1-execute §13）。
