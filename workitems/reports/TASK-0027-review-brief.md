# 评审 brief：TASK-0027 diff 评审（代码评审）

> 实现提交 e1b6266（BASE=d02ac44，单提交，23 文件纯 ingest/** +3747 行级规模）——Go 道真集成第一环。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0027-diff.txt`（e1b6266^..e1b6266，23 文件全落 ingest/**——协调者已隔离 0015 提交）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0027-brief.md`（接口与决策 8 条——决策 5 为 gatewayStatus Go 侧先行口径）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0027-report.md`（含 §5.2 snapshotHash 发现登记与环境处置偏离）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0027.md`
- 契约三份：`D:\workspace\WisFactory\resources\contract\`（统一消息模型 v1.1.0 / config-snapshot-schema v1.0.1 / 超级表 v1.0.0）
- 上游代码：`D:\workspace\WisFactory\ingest\internal\**`（0073/0076 既有件——零改动替换原则的核对基准）；backend 端点参照 iot/publish/controller/IotPublishInternalController.java

## 越界反查（协调者已比对，评审者复核）

- `git diff --name-only e1b6266^..e1b6266` 共 23 文件 100% 落 `ingest/**`，无越界（并行 0015 道的 backend 改动未混入）。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **MQTT 消费者**：§3.2 主题路由矩阵逐字（data/status 上行 + config 下行，LWT 同 status 主题语义）；paho v1.5.1 钉版本 + MqttConsumerPort 薄接口；消费源进 pipeline 的 Consumer 接口零改动替换
2. **两路上报一致性**：MQTT 信封与 HTTP body 同构进同一流水线——假 broker 契约测试是否真实锁住「逐字段一致」（还是只测了各自能跑）
3. **幂等真实现**：(pointCode, ts) 单条 + batchId 整批；MarkIfSeen 原子+per-key 串行化（BUG-0002 红线范式沿用）；单实例进程内 TTL 的内存界与淘汰策略（无泄漏）；多实例=Redis 化演进位登记
4. **快照真客户端与哈希口径**：**关键发现评估**——snapshotHash=原始存储字节 SHA-256，内部端点出站体经 Jackson 再序列化（5.000000→5.0）下游不可重算；执行者改为「对账锚点口径」（响应 hash 字段比对 + 形态校验 + 同版本漂移探测）。评估：该口径在此契约现状下是否正确且防御充分（哪些完整性威胁仍敞开）；「真修」（端点原始字节透传）已由协调者另立微改 TASK-0080——评审者只需确认 Go 侧口径未过度声明、未虚构校验能力
5. **gatewayStatus**：字段缺失/空=状态未知不 fail-closed 不放行扩大（语义明确性）；禁用网关 deny 红线（fake puller）真实有效
6. **ingestd 接线**：env 配置完整性/优雅停机（SIGTERM 排空）/端口纪律（8090）
7. **fail-soft**：broker 不可达指数退避（500ms→30s 封顶）期间 HTTP 路径零受扰；重连无 goroutine 泄漏
8. **零改动替换原则**：0073/0076 既有 94 项契约测试零改动全绿（对照 git diff 确认既有文件改动面）
9. **并发模型**：单消费循环+有界队列+背压（README 冻结口径）；Race 探针测试有效性

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0027-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行 或 diff 位置）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
