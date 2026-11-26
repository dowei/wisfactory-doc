# 评审 brief：TASK-0083 diff 评审（代码评审）

> 实现提交 4d4012e（BASE=3cdde58，隔离区间 9a6cda6..4d4012e——剔除并行道 0041 calc 交错提交；3 文件）——发布时未锁定实例自动跟随（TD-0059 处置，FEAT-0005 收官）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-diff.txt`（9a6cda6..4d4012e，3 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-report.md`（含上下文核实结论与"当期快照保留跟随前基线"遗留）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0083.md`
- 决策：ADR-0008（二档政策=未锁定实例随发布自动跟随；T3 永不自动）
- 接缝：reports\TASK-0081-report.md + TASK-0081-review-round2.md（applyToVersion 边界 + m-1 SecurityUtils/TenantContext 依赖——执行者称上下文核实适用）
- 上游代码（直接读）：IotPublishServiceImpl（接线点 :268/:290）+ IotUpgradeApplyService（0081 接缝）+ IotModelInstance（locked/applied）

## 越界反查（协调者已比对，评审者复核）

- 3 文件全落 backend publish/service/impl + 测试；0081 内核 IotUpgradeApplyServiceImpl 零改动；SQL/yml/ot-core/前端/ingest/calc 零触碰；0041 calc 交错提交已隔离剔除。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **接线时序（最尖锐）**：执行者按 brief「快照存储完成后」时序接线——故**当期发布快照保留跟随前基线，数据面推进体现在下一期快照**。评估：这与 ADR-0008「发布生效」语义是否真的成立（"发布 v10 后未锁定实例当期仍按旧公式跑、要等下次发布才进快照"是否构成语义缺口）；执行者已登记为遗留待裁决——评审者独立判断严重度（可能 Important）
2. **筛选口径**：scope_type=2 计算域才触发（网关 scope 不触发实证）、同租户、locked=0、applied<current、T3 判档跳过（逐台 skipped 实证）、乐观锁 4002 口径——筛选条件是否遗漏边界（如新实例 applied=0/null）
3. **事务与失败语义**：publish 现有 @Transactional 内 followUnlockedInstances——失败抛异常整批回滚（WEEKLY 坏码→400 且无 v13 版本行实证）；**回滚是否真的覆盖版本行+快照文件+publish_log 全链**（特别注意：0024 两段式落位的 promote 在事务提交前，若 follow 抛异常，文件已落位但行回滚——"已提交行指向缺失文件"的反面"孤立文件"是否可接受）
4. **上下文核实（0081 m-1）**：applyToVersion 经 regenerate 依赖 SecurityUtils/TenantContext——发布场景实测 userId=1 留痕正确、计算域管理员语义一致；评审者评估该核实是否充分（非登录态的定时任务/内部调用触发发布时是否风险）
5. **0081 内核零改动实证**：IotUpgradeApplyServiceImpl 未改（仅调用）——diff 核实
6. **测试有效性**：6 红线测试+25 补桩——哈希变化/锁定保持/网关不触发/T3 skipped/坏码回滚/上下文断言是否真断言；2401 测试零回归（基线 2395+6）
7. **对外契约**：publish 四端点签名不动；prod 登录链+列表/清单全 200

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。**特别对第 1 点接线时序给出明确严重级判定**——若判 Important 则按 §12 退回。若整体无 Critical/Important，给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
