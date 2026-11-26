# 评审 brief：TASK-0041 diff 评审（代码评审）

> 实现提交 9a6cda6（BASE=3cdde58，单提交 23 文件纯 calc/**）——双触发执行器（周期调度+到达驱动+节流幂等+汇总算子+失败隔离）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-diff.txt`（3cdde58..9a6cda6，23 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-report.md`（含 4 处 TDD 红转绿与 live 冒烟全序列）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0041.md`
- 关联：FEAT-0013/REQ-0020/TC-0013/ADR-0014（workitems 库）
- 上游：`reports\TASK-0040-report.md`（calc 工程/GraphHolder/AppliedGraph/ACK 对账）+ `reports\TASK-0029-report.md`（iot:newvalue:{tenant} 发射侧）+ calc/src/** 既有图代码
- 契约：CONV-02 质量码 + 详细设计 §1.6 计算周期

## 越界反查（协调者已比对，评审者复核）

- 23 文件全落 `calc/**`；并行道 0083 backend 改动未混入（区间仅一提交）；backend/frontend/ingest/契约零触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **两通道统一引擎**：DualTriggerExecutor 拓扑序执行复用 0040 图结构是否正确；周期通道与到达通道是否真正入同一执行器（非两套逻辑）；ResultSink 边界是否清晰（不写 TDengine/MySQL 业务表——实证）
2. **周期调度**：整点判据（calc_period HOUR/DAY/MONTH/YEAR，结果 ts 对齐周期边界）——自实现线程的启停/异常处理/优雅退出；`executor-enabled` 默认关闭对 0040 既有行为零变化
3. **到达驱动**：真订阅 Redis `iot:newvalue:{tenant}`（0029 发射侧）的 RESP2 客户端正确性（解析/重连/取消订阅）；同秒合并节流（key=全键+窗）+ 有界 TTL 幂等（同窗只算一次保留最后正确结果）——多实例 Redis 化演进位登记
4. **失败隔离**：单节点失败 quality=6/不发假 0/不阻塞其他分支——"失败只影响下游"语义是否正确（下游是否真的被标记为不可算）；与 CONV-02 质量码不冲突
5. **汇总算子**：SUM/AVG/MAX/MIN 四法 + quality=7 缺失打码（不纳入但保留打标）+ 开关关闭不纳入（ADR-0008 T1 保险）；动态查找子级正确性
6. **ArithmeticEvaluator/CalcPeriod**：白名单算术求值（非法表达式 fail-closed，非 eval 注入）；CalcPeriod 与 0081 mapCalcPeriod 口径一致性（MONTH/YEAR 在 calc 侧独立实现）
7. **并发与资源**：调度线程/Redis 订阅线程生命周期（无泄漏）；有界 TTL map 的内存界；优雅停机（8095/18041 全停实证）
8. **测试有效性**：134 测试（1 skip=env 门控）零回归对照 0040 的 115；live 冒烟证据（topo 链/10→1 节流/周期窗右边界/失败链隔离/SUM=14 q7=6.0）与单测断言对应；4 处红转绿留痕真实
9. **观测视图**：/calc/v1/status 执行器视图是否泄露内部状态/性能影响

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
