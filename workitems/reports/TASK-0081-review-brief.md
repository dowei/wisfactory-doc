# 评审 brief：TASK-0081 diff 评审（代码评审）

> 实现提交 b491e34（BASE=09357d1，9 文件 +1202−17）——确认升级数据面生效编排（用户确认开工，计划评审三裁决已定）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-diff.txt`（09357d1..b491e34，9 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-report.md`（含开工复核性实证四项/live 哈希变化证据/边界四类断言）
- TASK 详情（**正文含三裁决全文**）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0081.md`
- 计划评审：`reports\TASK-0081-plan-review.md`（三机制裁决的唯一结论）
- 决策：ADR-0008（三档政策原文）；关联 FEAT-0005/REQ-0006/TC-0005
- 上游代码（需要时直接读）：iot/template/**（IotTemplateVersionService.buildContentJson/版本历史）+ iot/dataitem/**（regenerate/引用边/FormulaCycleDetector）+ iot/publish/snapshot/ConfigSnapshotAssembler.java（组装器零改动实证）

## 越界反查（协调者已比对，评审者复核）

- 9 文件全落 backend template/dataitem 域+测试；执行者声明 `mapCalcPeriod` 仅 static→public static 可见性放宽（供复用）——核实无其他语义改动；publish/snapshot/SQL/yml/ot-core/前端/ingest/calc 零触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **开工复核性实证**：四项对照源码（基线 content 字段齐备/兜底形态复现/逻辑删双 uk 不含 deleted/组装器读实例行）——执行者称零偏差，评审者独立核实（有任何不符即 Important）
2. **T2 三方合并正确性（最关键）**：判定「未覆盖」的三形态——行值为空/等于旧默认（公式按本实例展开后比对/calc_period 经 mapCalcPeriod 换算后比对/source_type 复现 null→1 兜底）——是否有漏判导致误覆盖用户已改值（方向=隐私/正确性风险）；已覆盖保留路径；新默认写入+公式展开+边重建的完整链路
3. **T1 补插**：applied→target 新增项委托 regenerate 补插——相对引用展开+边落库；对已存在 (instance_id,data_code) 是否幂等（regenerate 既有语义）；补插项参与汇总默认关闭（ADR-0008 T1 保险）
4. **T3 逻辑删迁移**：删项=逻辑删（@TableLogic 退出快照+历史行自身承载+双 uk 键永久占用防复用污染）；改类型/改取值方式=结构迁移+边重建；入边与指标引用不清理（ADR 明示后果，逐台 diff 明示）；**逻辑删行的 uk 行为在 8.0.12 实测环境是否成立**（执行者 live 实证 deleted=1，评审者核机制）
5. **事务与失败语义**：确认接口同事务并入编排（簿记+行应用+留痕单原子）；任一实例失败整批回滚+4002 口径；乐观锁与并发确认/发布组装无半态
6. **可复用接缝 applyToVersion(instance,target)**：方法面是否真可复用（供 TASK-0083 发布自动跟随），无隐藏耦合（如依赖确认接口上下文/会话态）；本任务未实现发布侧接线（D-1 已拆 0083）实证
7. **留痕过渡口径**：逐台结果 VO（diff+迁移动作+rowUpdated）+逐台 [UPGRADE-APPLY] INFO+@OperateLog 请求级；不建表/不写 iot_data_item_history/不改 0014 NoOp——实证
8. **live 哈希变化证据**：确认前 ad7e6471…(v8)→确认后 64fdb216…(v9)，快照体 E8A 新公式 /1000 与 E8B 旧公式 /100 并存——核实该证据与单测断言对应（数据面生效真机实证）
9. **2384 测试零回归**（基线 2373+11；既有确认/预览/发布测试不受编排接线影响）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0081-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
