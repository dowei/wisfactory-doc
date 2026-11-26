# TASK-0084 diff 评审退回修订 brief（一轮退回）

> 主体提交 2fc2a4d 验收三条自测全过，但 diff 评审 `TASK-0084-review.md` 判 C-1（Critical，核心成立）+ I-1/I-2（Important）+ M-1（Minor）。本 brief 为**一轮退回修订**，只修评审发现，不重开全场。续用原执行者上下文。

## 必读（只给路径，不粘正文）

- 原实施 brief：`reports\TASK-0084-brief.md`
- 主体报告：`reports\TASK-0084-report.md`
- 评审报告：`reports\TASK-0084-review.md`（C-1/I-1/I-2/M-1 原文）
- diff 包：`reports\TASK-0084-diff.txt`（fb8f66d..2fc2a4d，8 文件）
- TASK 详情：`tasks\TASK-0084.md`（任务结果段已写裁决）
- **权威口径证据**（本次已核查，逐字为准，勿再信评审的错误论证）：
  - `resources/contract/unified-message-model.md:28-35`（pointCode 示例下划线 + 字段纪律）
  - `resources/contract/config-snapshot-schema.md:67/95/165`（pointCode 只有长度约束无字符集；fullKey = instance_code + "." + data_code）
  - `resources/contract/push-payload-evaluation.md:33`（pointCode 与 fullKey 是两个编码空间）
  - `ingest/internal/pipeline/pipeline.go:680`（newvalue 的 pointCode = PointRow.PointCode 原样）
  - `reports/TASK-0029-report.md:142-153`（live 真样本 `V29_NUM_01` 等下划线形态）
  - `ingest/internal/snapshot/snapshot.go:32-43/47-70/86-103/510-560`（Snapshot/Point/DataItem/Cache.LookupPoint 结构）
  - `ingest/cmd/ingestd/main.go:277/340/444-450`（snapCache 已装配、startItemBridge 装配点）

## 裁决摘要（C-1 核心成立，评审论证有失真已纠正）

1. **C-1 核心成立，必须修**：`MapPointCode` 用 `LastIndex(".")` 解析 pointCode 是错的。
   - **契约事实**：pointCode 是**点位编码**（现场层，下划线形态如 `V29_NUM_01` / `F01_WS01_EQ001_M01`，无 `.`）；fullKey 是**数据项全键**（`instance_code + "." + data_code`），两者是不同编码空间。0029 实测 newvalue 的 pointCode 全是下划线无点号 → 当前映射把这些真实事件全丢弃，opt-in 打开后主路径发布恒为 0，端到端缺口没闭合。
   - **根因**：原 brief 决策 2 / TASK 技术方案写「pointCode 形如 F01.WS01.EQ001.M01 → 去末段」是**我（协调者）的口径错误**；你照示例实现且测试只用点分号样本，形成假阳性。责任不全在你，但主路径空转是实打实的，必须修。
   - **不要采信评审这三处失真**：①评审称「pointCode 字符集 `[A-Za-z0-9_]`、`.` 是 pointCode 禁用保留符」——错，pointCode 白名单实为 `[A-Za-z0-9_.]`（`IotPointServiceImpl.java:164`），契约从未禁 pointCode 含点号；②「0034/0036 既定映射口径」——0034 未实施、0036 讲实例全码/数据项 fullKey，均非 pointCode 映射依据；③评审报告引用的行号（bridge_contract_test.go:538 等）与真实文件不符。**按上面"权威口径证据"的原文为准。**

2. **正确映射方案（协调者裁决）**：改为**配置快照反查**，不再从 pointCode 字符串结构臆造 instance：
   - 通过已装配的 `snapshot.Cache`（`main.go:277` 的 `snapCache`）`LookupPoint(pointCode)` 拿 `Point`（含 `ID`）；
   - 在快照 `DataItems` 中找 `PointID == point.ID` 的 `DataItem`（`snapshot.go:86-103`，`PointID *int64` / `FullKey string` / `InstanceID int64`）；
   - `fullKey = dataItem.FullKey`（契约数据项全键，原样发布，不再等于 pointCode）；
   - `instanceCode` = 从 `dataItem.FullKey` 用 `LastIndex(".")` 去末段取前缀——**此时点号是契约定义的数据项全键分隔符，解析合法**；或直接用 `dataItem.InstanceID` 经实例表反查 instanceCode（若快照带 instance 短码映射则优先，避免 ID 数字当短码）。
   - 未命中（快照未装载 / pointCode 未配点位 / 点位未绑数据项）→ WARN + 丢弃（不造假主题），与现有 fail-soft 一致；opt-in 默认关不变。
   - 桥接组件需持有 `snapCache` 引用（经构造参数注入，`startItemBridge` 装配时传入），保持"桥接失败不影响 ingest 主流水线"。

3. **I-1（Important）——评估后处理**：评审指 Redis 订阅源定义了 Password 却未装配（与发射侧同为匿名形态）。这是 ingest 既有边界，若发射侧 0029 本就匿名、宿主 Redis 无密码，则**不改代码**，在报告里如实登记「与发射侧对称的匿名形态，需在需要密码的环境由 env 扩展」即可；若你认为装配成本低且不影响零新增依赖，也可一并补 `REDIS_PASSWORD` env 注入（可选，不强制）。

4. **I-2（Important）——必须修**：桥接 MQTT 发布用 `context.Background()+5s` 而非 Run 传入的 ctx，`ctx` 取消最坏延迟 5s。改为 Run 的 ctx 贯穿发布调用（保留超时边界但用 `context.WithTimeout(runCtx, ...)`），使取消即时生效。补一条「取消中发布立即返回」的契约测试。

5. **M-1（Minor，可选）**：失败 WARN 无采样，异常流量下日志放大。加一个简单计数采样（如每 N 条打一条，N 常量可配）即可；不强制，做了在报告说明。

## 测试口径（TDD，红转绿留痕）

- **新增必须覆盖下划线形态**：契约测试用真实样本 `V29_NUM_01` / `F01_WS01_EQ001_M01`（无点号），经快照反查命中 DataItem → 发布 fullKey + topic instance 正确；原来那 12 项点分号样本**保留但改语义**——点分号形态此时应视为"未经快照绑定的原始 pointCode"，若无快照绑定也应丢弃（或删除纯字符串解析用例，改为快照反查用例）。
- live 7 步序列重跑，S2/S3 的下划线 pointCode 真收 4 字段；映射失败样本仍丢弃；租户隔离/无订阅者/ctx 退出/全停不变。
- 既有 ingest 契约测试零回归（`go test ./...` 全绿）。

## 验收命令（沿用原始三条，必须原样贴回输出）

1. `cd ingest && go build ./... && go vet ./... && go test ./...` → 0 失败 0 错误（Results 段贴回）
2. live 冒烟：本地夹具 broker + 真 Redis + **装载含下划线 pointCode 绑定的快照** → 开启桥接 → PUBLISH `iot:newvalue:{tenant}`（pointCode 为 `V29_NUM_01` 下划线形态）→ MQTT 订阅 `iot/open/+/+/item` 真收 4 字段（fullKey 为快照 DataItem 的 fullKey、topic instance 正确）→ 未绑定 pointCode 丢弃 → 关闭开关无发布 → 全停
3. 范围核对：`git diff --name-only <主体BASE>..HEAD` 全落 `ingest/**`

## 交付

- 修订提交（注明 TASK-0084 修订轮）；报告追加「修订轮」段：每项发现的处理（C-1 必含快照反查实现 + 下划线测试红转绿）、全量 Results、为什么现在对了、遗留。
- **不要**把 TASK 置「已完成」；只改 ingest/**；no-subagents；契约冻结。
- 报告写到 `reports\TASK-0084-report.md`（追加段）。
