# 评审 brief：TASK-0029 diff 评审（代码评审）

> 实现提交 cfed417（BASE=b4e877a，单提交，19 文件纯 ingest/**，按路径隔离——中间 backend 提交属 0016 道已单独收尾）——采集接入链收官：真攒批写 TDengine、Redis 点位缓存与新值事件。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-diff.txt`（b4e877a..cfed417 限 ingest/，19 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-brief.md`（真执行器/Redis 缓存/新值事件/0027 遗留两项/口径固化三项/live 冒烟序列）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-report.md`（含 §4 EnsureSchema live 修正——taosAdapter 多语句信封静默空操作发现、§5.2 契约发现）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0029.md`
- 契约：`D:\workspace\WisFactory\resources\contract\`（超级表 v1.0.0——附录 B DDL/kind 映射/DBName 派生；统一消息模型 v1.1.0）
- 上游代码：`D:\workspace\WisFactory\ingest\internal\**`（0073/0027/0028 既有件——零改动替换核对基准）
- 关联决策：workitems 库 `decisions\ADR-0010.md`（LWT 离线发事件不直写状态 Redis）、`decisions\ADR-0014.md`（攒批 FIFO）

## 越界反查（协调者已比对，评审者复核）

- `git diff --name-only b4e877a..cfed417 -- ingest/` 共 19 文件全落 ingest/**；同区间 backend 文件（0016 道 082d694）不在本评审范围。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **真执行器**：单 supertable 一条 INSERT 多 VALUES（禁止逐条写——契约测试锁语句形态）；kind/表映射与附录 B 列序逐字；子表首写自动创建（INSERT...USING TAGS tenant_id/kind 与超级表契约一致）；**逐语句独立下发修正**（多语句信封静默空操作的 live 发现——评估修正正确性与遗漏面：还有没有别处用多语句信封）
2. **攒批**：水位 500/200ms 双触发；写失败整批重排受同上限约束（ADR-0014 FIFO）；**写失败重试与 0027 幂等的协同**（重排后的批再次到达，去重兜底防幻影事件/重复缓存——执行者报告称零幻影零污染，核实机制）
3. **Redis 点位缓存**：key `iot:pt:{tenant}:{code}` v/q/ts；写穿透（落库成功后更新）；缓存失败不阻塞落库（fail-soft WARN）；未命中回源（懒 read-through TDengine 最近值）；**纯标准库 RESP2 客户端**（零新增依赖）——协议实现正确性（解析/错误处理/连接管理）评审
4. **新值事件**：Redis Pub/Sub `iot:newvalue:{tenant}` payload 五键契约；发射时机（落库成功后）；ADR-0010 兑现（LWT/状态走事件不直写状态 Redis——`iot:gwstatus:{tenant}` 发射侧拟定口径评估）；订阅侧无人消费时的行为（Pub/Sub 无订阅者=消息丢弃，可接受性）
5. **0027 遗留两项承接**：SIGTERM 排空（缓冲清零+落库+超时兜底强制退出）；sink 队头约束复核
6. **口径固化三项**（brief 第 5 条）：①纠偏标记=日志+计数 ②被拒重发=到达次数视角 ③纠偏计数=动作次数——登记为口径即可，核实 0028 实现未被本 diff 破坏
7. **清理纪律**：scratch 库 iot_verify29 DROP + Redis 键清理的清理证明（报告 §2.2.e）——评审者核实清理代码/步骤不会误伤共享库
8. **零改动替换**：既有 167 契约测试零改动（唯一重写=EnsureSchema——live 发现驱动的例外，核实重写合理性）；零新增依赖（go.mod/go.sum 零改动）
9. **并发模型**：真执行器落位后的 sink 调用形态、Pub/Sub 连接生命周期（无 goroutine 泄漏）、SIGTERM 排空与在途写的关系

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
