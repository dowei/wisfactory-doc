# 评审 brief：TASK-0028 diff 评审（代码评审）

> 实现两提交 8a837aa + b4e877a（BASE=cebf17a，18 文件纯 ingest/**，按路径隔离——中间的 backend 提交属微改批二/0016 道已单独收尾）——Go 道校验流水线/死信/质量判定。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0028-diff.txt`（cebf17a..b4e877a 限 ingest/，18 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0028-brief.md`（量程/死信/时间三分裁决/去重复核）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0028-report.md`（含 brief 越界码方向勘误登记、墙钟定时炸弹拆除、Race 竞态定位）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0028.md`
- 契约：`D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md`（min/max）+ 统一消息模型 v1.1.0 + CONV-02（ingest/internal/quality 枚举冻结）
- 上游代码：`D:\workspace\WisFactory\ingest\internal\**`（0073/0027 既有件）

## 越界反查（协调者已比对，评审者复核）

- `git diff --name-only cebf17a..b4e877a -- ingest/` 共 18 文件全落 ingest/**；同区间非 ingest 文件（7 个）属并行 backend 道任务（微改批二/0016），不在本评审范围。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **量程判定 vs CONV-02 冻结枚举**：执行者登记 brief 越界码方向勘误（brief 误写 value<min→1/value>max→2；冻结枚举 1=超上限/2=超下限）——核实实现以冻结枚举为准、边界（==min/max）不算越界、缺失透传、原值保留不丢弃不假 0、越域码防御收敛
2. **quality 合并规则**：网关先填+快照 min/max 二次判定覆盖写（§3.2 口径）——合并矩阵是否完整（网关填了/没填 × 越限/正常）
3. **时间三分**：越界拒绝（ts_overflow/ts_too_old 阈值可配）/未来窄带纠偏+打标（不改 CONV-02 码域——结构化 WARN+计数器）/补传窗原始 ts 落库（§1.9A 锚点）——三路径边界值正确性（恰好 5m/30d/now）与**去重键取消息原始 ts（纠偏不改幂等键）**的一致性
4. **死信统一通道**：原因码与 HTTP 拒绝明细对齐；按租户+原因隔离计数（有界 map 内存界）；WARN 采样防风暴（首条+每 N 条）；环形缓冲排查读口（有界、无泄漏）
5. **四步校验顺序**：格式→量程→时间→去重嵌入冻结五步的位置正确性；去重靠后——重复上报不产生重复死信计数的口径
6. **ADR-0009 边界注记**：无人工补录入口
7. **墙钟定时炸弹拆除**：5 处测试改冻结时钟注入（契约样例 sentAt）——评估注入方式不弱化断言
8. **Race 竞态定位**：b4e877a 的测试侧修复（复核取收敛后快照）——确认是测试竞态非流水线缺陷被掩盖（恰一写出语义三重锁定是否充分）
9. 零回归：既有 135 契约测试零改动（对照 diff 既有文件改动面）；零新增依赖

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0028-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
