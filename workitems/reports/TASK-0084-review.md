# TASK-0084 代码评审报告（diff 评审 · L1-execute §13）

- 评审对象：实现提交 `2fc2a4d`（BASE=`fb8f66d`，8 文件全 `ingest/**`）
- 评审角色：fresh 代码评审子代理（no-subagents / 只依据 brief 指到的文件与工作区源码 / 只评审不改文件）
- 评审日期：2026-11-24
- 评审口径：`TASK-0084-review-brief.md`「必读 / 评审重点 / 报告 / 契约」逐项执行；严重级 C（阻断）/ I（重要）/ M（次要）
- 独立验证手段（只读）：`git show/cat-file` 核提交范围；`go build ./... && go vet ./... && go test ./...`；`go test ./... -run Contract -v -count=1` 计数；`INGEST_ITEM_BRIDGE_LIVE=1 ... go test -tags live` 复跑 live 7 步；源码/contract/workitems 逐文件比对

## 总判定

**不通过（Critical 1）**——实现在 opt-in 开关、4 字段载荷、租户隔离、生命周期、零新增依赖、测试有效性与范围控制上均达标且经独立复跑证实；但**重点 2（pointCode→instance 映射与 0034/0036 口径一致性）存在阻断级口径冲突**：桥接按「点分号（`.`）去末段」取 instance，与项目既定且冻结契约/快照 schema/流水线实际使用的 pointCode 编码体系（`.` 是**数据项 fullKey/实例路径**分隔符，`_` 或字母数字才是点位 pointCode 的现场编码）不一致；结果是**主路径上真实事件会被映射失败丢弃，端到端链路在 opt-in 打开后仍不成立**，且主题 instance 段的语义与 0034/0036/0053 口径漂移。详见发现 C-1 与「关于重点 2 的明确判定」。

---

## 发现清单

### [C-1] pointCode→instance 映射口径错误：用数据项全键规则解析点位编码，真实事件将被丢弃（评审重点 2）

- 严重级：**Critical（阻断）**
- 出处：
  - `D:\workspace\WisFactory\ingest\internal\itembridge\bridge.go:53-58`（`MapPointCode`：`strings.LastIndex(pointCode, ".")`，无点即失败）
  - `...itembridge\bridge.go:164-201`（`onEvent`：`MapPointCode` 失败 → dropped++、丢弃）
  - `...itembridge\bridge_contract_test.go:538-551`（测试夹具只用 `F01.WS01.EQ001.M01` 这类点分号样本）
  - `...itembridge\bridge_live_test.go:1110-1119, 1185-1189, 1195`（live 注入的也是点分号样本）
  - 对照：`D:\workspace\WisFactory\resources\contract\unified-message-model.md:28` 冻结契约示例 pointCode=`F01_WS01_EQ001_M01`（下划线）；`D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md:67,95`（pointCode 仅约束 `string(96)`、租户内唯一，无「必含点号」规则）；同文件 `:165,195`（**fullKey = instance_code + "." + data_code**，即 `.` 是数据项全键分隔符）；`D:\workspace\WisFactory\ingest\internal\snapshot\snapshot.go:91`（`FullKey // = instance_code + "." + data_code`）
  - 对照：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0036-report.md:96-116`（**F01.WS13.EQ36 是 instanceCode，F01.WS13.EQ36.ELEC_METER 是数据项 fullKey**——点分号路径是实例/数据项编码，非点位 pointCode）；`workitems\reports\TASK-0053-report.md:147`（消费侧实测载荷 fullKey=`F01.M01` 即点分号路径，但那是**数据项**，不是点位）；`D:\workspace\WisFactory\backend\...\openapi\subscribe\SubscriptionConsumer.java:49`（消费侧断言 `fullKey.startsWith(instanceCode + ".")`）
- 说明：0029 发射的 `iot:newvalue` payload 中 `pointCode` 来自流水线 `PointRow.PointCode`（`ingest/internal/pipeline/pipeline.go:680-686`），即**点位编码**；项目里点位编码的权威形态是下划线/字母数字（冻结契约 `F01_WS01_EQ001_M01`、0029 live `V29_NUM_01/V29_STATE_01`、流水线/快照测试 `F01_WS01_EQ001_M01`）。`.` 是**保留分隔符**：`IotInstanceCreateForm.java:15`、`IotModelInstanceServiceImpl.java:48` 明定实例短码字符集 `[A-Za-z0-9_]` 就是为「防 `.` 路径分隔符注入编码体系」，`.` 专用于 instance 路径与 fullKey=instance_code+"."+data_code。桥接按 `.` 去末段解析点位编码，等于把「数据项全键」规则套到点位编码上，导致：
  1. **主路径丢弃**：冻结契约示例 `F01_WS01_EQ001_M01`、0029 live `V29_NUM_01` 这类真实 pointCode 无 `.` → `MapPointCode` 返回错误 → 每条新值事件都被 WARN 丢弃（实测：`go run` 模拟 `MapPointCode("F01_WS01_EQ001_M01")` → `err=no dot`，`V29_NUM_01` 同样丢弃）。opt-in 打开后桥接**发布数恒为 0**，0053 评审 I-2 的端到端缺口**实际未闭合**。
  2. **口径漂移**：即便某些现场 pointCode 恰用点号，桥接发布的 `fullKey=pointCode 原样`（点位编码）、instance=去末段（数据项式实例前缀）——这与消费侧断言（`fullKey` 必须 `startsWith(instanceCode+".")`，即 fullKey 是数据项键）在概念上不同源；且 0053 消费侧 live 的 `iot/open/default/F01/item` + `fullKey=F01.M01` 表明数据项实例段与点位编码并非简单同一物。brief 决策 2 明写「具体以 0036/0034 映射函数为准，开工先读 ingest/internal/tdengine 的 childTable/fullKey 映射复用」，而 `tdengine.ChildTableName`（`tdengine.go:241`）是 `escapeIdentifier(pointCode)` 的**恒等/转义映射**，从不做「去末段取实例」——实现未复用既有映射口径，而是新造了一套与 frozen fullKey 定义冲突的解析。
  3. **测试未覆盖真实域**：12 项契约测试与 live 7 步全部使用点分号 pointCode 样本（`F01.WS01.EQ001.M01` 等），对契约示例/0029 live 的下划线形态零覆盖，形成「测试全绿但主路径空转」的假阳性。
- 处置建议（供协调者裁决，评审不改码）：明确 pointCode→(fullKey,instance) 的**权威映射来源**。按 brief 契约纪律，正确方向应是「点位 pointCode → 其绑定的数据项 fullKey + instanceCode」经**配置快照反查**（`snapshot.DataItem` 带 `pointId` 绑定关系与 `fullKey/instance_code`），而**非**从 pointCode 字符串结构臆造；若协调者确认「现场 pointCode 一律就是 `instance.dataCode` 形态的数据项键」，则需以裁决形式改写 brief/契约示例并补齐下划线形态的丢弃语义说明，否则当前实现应判不通过。

### [I-1] Redis 订阅源 Password 恒空、装配未传密码，与发射侧/握手纪律不对称（评审重点 3/5 边界）

- 严重级：**Important**
- 出处：
  - `D:\workspace\WisFactory\ingest\internal\itembridge\redis_source.go:20-26`（`RedisSourceOptions.Password` 字段存在）、`:58-62`（`runOnce` 中 `if s.opts.Password != ""` 才 AUTH）
  - `D:\workspace\WisFactory\ingest\cmd\ingestd\main.go:444-448`（`startItemBridge` 构造 `RedisSourceOptions{Addr, DB}`，**未传 Password**）
  - 对照：`cmd/ingestd/main.go:97-98,128` 的 `Config` 无 RedisPassword 字段；发射侧 `event/redis.go` 与 `cache.RedisOptions`（`internal/cache/redis.go:34-35`）同样无密码入口——属既有形态
- 说明：若 ingest 部署的 Redis 配置了 requirepass，发射侧与缓存侧的现有实现也无法 AUTH（既有缺口，非 0084 新引入）；但 0084 源定义了 Password 字段却未接线，会给人「支持鉴权」的错觉。**属装配完整性问题而非本任务硬要求**（brief 决策 1 只说复用 0029 客户端、未提密码），记 Important 供确认：要么显式 javadoc 标注「与发射侧同口径，仅匿名形态」，要么统一补 RedisPassword env 接线。

### [I-2] 桥接发布用 `context.Background()` 而非传入的 ctx，取消时最多延迟 5s（评审重点 4 边界）

- 严重级：**Important**
- 出处：`D:\workspace\WisFactory\ingest\internal\itembridge\bridge.go:204`（`pubCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)`）
- 说明：`Bridge.Run(ctx)` 持有生命周期 ctx，但每条发布用的是独立 background+5s；`PahoItemPublisher.Publish`（`paho_publisher.go:1328-1345`）在 `tok.Wait()` 上等待，ctx 取消不会立即打断在途发布——最坏停机多等 5s。契约测试 `TestContractBridgeContextCancelExits` 在空闲源上通过，未覆盖「发布卡住时取消」。属生命周期严谨性缺口，建议把 `onEvent` 持有 Run 的 ctx 并 `context.WithTimeout(ctx, ...)`。

### [M-1] WARN 日志无采样，异常流量下可能刷日志

- 严重级：**Minor**
- 出处：`bridge.go:168,178,186,192,199,210`（每条失败均 `logger.Printf("WARN ...")`）；报告 §5.4 自认「仅进程内原子计数 + WARN，无采样」
- 说明：与 0029 `DeadLetterWarnEvery` 采样纪律不一致，桥接失败风暴下日志放大。演进位，不阻断。

---

## 评审重点逐项结论

| # | 评审重点 | 结论 | 关键依据 |
| --- | --- | --- | --- |
| 1 | 4 字段映射精确（fullKey=pointCode / ts/value/quality 透传 / payload 无 tenant / 失败丢弃） | **通过（字段层），但 fullKey 取值来源见 C-1** | `bridge.go:244-254` `ItemChangeEvent` 4 字段 camelCase/ts 毫秒；`:164-201` 五段纪律；契约测试 `...FourFieldsOrderedCamelCaseMilliseconds` 逐字断言；`go test` 复跑 PASS |
| 2 | pointCode→instance 映射与 0034/0036 口径一致性 | **不通过（Critical，见 C-1）** | `MapPointCode` 点分号去末段 vs 冻结契约 pointCode=`F01_WS01_EQ001_M01`（下划线）与 snapshot fullKey=instance+`.`+dataCode；实测真实形态被丢弃；测试仅覆盖点分号样本 |
| 3 | 租户隔离（主题含 tenant / payload 无 tenant / channel-payload tenant 一致性校验 / 与 0053 双保险） | **通过** | `bridge.go:74-84,183-190`；`TenantFromNewValueChannel` 拒绝 `:`/空白 tenant；契约测试 `TenantIsolation` + live S5 真收；0053 侧 `SubscriptionConsumer.java:42-56` 主题+实例+fullKey 校验保持 |
| 4 | opt-in 与生命周期（默认 false / 独立 goroutine+ctx 取消 / 装配失败只 WARN） | **基本通过（取消有 5s 上界缺口 I-2）** | `bridge.go:91-93` 仅 true/1 开；`main.go:112,139,338-340,403-404,424-455`；契约 `OptInDefaultOff`+wiring 3 项 + live S1/S7；`startItemBridge` 依赖缺失/连接失败均 WARN 不阻断 |
| 5 | 零新增依赖（RESP2 复用/ Paho 复用 / pom+go.mod 零改） | **通过** | `git show --name-only 2fc2a4d` 无 go.mod/go.sum/pom；Paho v1.5.1 已在 `go.mod:6`/`go.sum`；`redis_source.go` 纯标准库 RESP2；live 真 Paho 复跑 PASS |
| 6 | 测试有效性（223=211+12 / live 7 步 / TDD 红转绿 / 调试日志清理） | **通过（有效性），但样本偏差见 C-1** | 独立复跑契约 223 PASS/0 FAIL、itembridge 12 项逐字；live 7 步独立复跑全绿（S1–S7 与报告逐字一致，`LIVE-ALL-DONE`）；`grep` 临时调试日志 `LIVE-SRC/LIVE-BROKER-RECV/LIVE-SUB-CONNECT` 为 none（仅保留 S1–S7 正式输出） |
| 7 | 零回归与范围（0027/0029/0041/0042 契约零回归 / commit 8 文件） | **通过** | 独立 `go build/vet/test ./...` 全绿（12 包 ok，EXIT=0）；`git show --name-only 2fc2a4d` 恰 8 文件全 ingest/**，非 ingest=0；`unified-message-model.md`/backend/frontend/calc 零触碰 |

## 关于重点 2 的明确判定（应 brief 特别要求）

**判定：不通过（Critical）**——pointCode→instance 映射与 0034/0036 既定口径**不一致，且存在两套映射漂移**。

- **0034/0036 既定口径**：`.` 是实例路径/数据项全键分隔符——`F01.WS13.EQ36` 是 instanceCode、`F01.WS13.EQ36.ELEC_METER` 是数据项 fullKey（0036-report S2/impact 实证）；冻结契约 `config-snapshot-schema.md` 明定 `fullKey = instance_code + "." + data_code`；点位 pointCode 是独立的现场编码（下划线/字母数字，无 `.` 要求）。
- **0084 实现口径**：`MapPointCode` 把 pointCode 当数据项 fullKey 处理（`去最后 "." → instance`），`fullKey=pointCode 原样`。这是把「数据项全键」规则套到「点位编码」上，`ingest/internal/tdengine.ChildTableName`（既有 pointCode 映射）从不做此事——所谓「复用 0034/0034 映射」未落实，而是新造口径。
- **后果**：真实主路径的 pointCode（契约示例 `F01_WS01_EQ001_M01`、0029 live 全部 `V*_*`、流水线/快照测试 `F01_WS01_*`）**全部无 `.` → 被丢弃**，桥接空转；测试/live 仅用点分号样本掩盖了该缺陷；主题 instance 段语义与 0053 消费侧断言（`fullKey startsWith instance+"."`）不是同一命名空间。
- **公平性说明（责任不全在实现方）**：brief 决策 2 / TASK 技术方案给出的示例 `F01.WS01.EQ001.M01 → F01.WS01.EQ001` 本身用的是点分号形态，实现严格照该示例与「以 0036/0034 映射函数为准」的字面要求落地；但 brief 同时要求「开工先读 ingest/internal/tdengine 的 childTable/fullKey 映射复用」——该既有映射（`ChildTableName`=escapeIdentifier 恒等/转义）从不做去末段，且冻结契约/快照/0029 live 的真实 pointCode 是下划线形态。这是 brief 示例与项目既定口径之间的内在矛盾，实现选择了示例口径而未核验权威口径，故判阻断并要求裁决，非单纯实现粗心。
- **口径一致性交叉验证**：与 0053 消费侧（fullKey 必须是数据项键且落在订阅实例命名空间）对比，0084 的 fullKey=点位编码 也不等价——即便放宽 pointCode 形态，两端对 fullKey 的定义仍不同源。
- **结论**：此项不能按「通过」处理。要么由协调者裁决 pointCode 与 instance/fullKey 的权威映射（推荐经配置快照 pointId→dataItem 反查），并要求实现按裁决重做 + 补真实 pointCode 形态测试；要么明确接受「本桥接仅支持点分号形态点位编码、下划线形态整批丢弃」的边界并更新 TASK 验收标准与契约说明（不推荐，因与 TASK-0029 已实证的事件形态直接矛盾）。

## 说明

- 评审为 fresh 视角，仅依据 brief 指到的 diff/报告/workitems/契约/源码与工作区源码判断；未向协调者追问、未派 subagent、未修改任何文件（本报告除外）。
- 独立复跑的 live 7 步与契约 223 计数与执行报告逐字一致，测试有效性与 TDD 留痕属实；C-1/I-1/I-2/M-1 为本次评审新增发现，报告 §五「遗留问题 3（映射口径配置化）」已自查到映射风险但判定为「演进位」，评审认为该风险已构成主路径阻断，级别应上调为 Critical。
