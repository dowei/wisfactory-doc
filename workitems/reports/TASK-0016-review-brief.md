# 评审 brief：TASK-0016 diff 评审（代码评审）

> 实现提交 082d694（BASE=b4e877a，单提交，13 文件纯 backend）——实例锁定/待升级清单/逐台批量确认接口组。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0016-diff.txt`（082d694^..082d694，13 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0016-brief.md`（三接口决策/权限复用/边界/测试口径）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0016-report.md`（含 5 项偏离登记与关键裁决项——数据面生效编排缺口的发现）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0016.md`
- 决策权威：workitems 库 `decisions\ADR-0008.md`（锁定/待升级/逐台迁移语义）；关联 `features\FEAT-0005.md`、`requirements\REQ-0006.md`、`testcases\TC-0005.md`
- 上游代码：iot/template/**（0015 版本服务/diff 引擎/tier 表复用面）、iot/publish/snapshot/ConfigSnapshotAssembler.java（组装读实例行——0016 执行者据此外推「数据面生效缺口」，评审者独立核实该外推）

## 越界反查（协调者已比对，评审者复核）

- 082d694^..082d694 共 13 文件全落 backend/**（template 域为主）；执行者偏离 1（锁定控制器落位 template/controller——instance 包在 brief 范围外）属 brief 笔误的合理执行，评审者核实落位合理性。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **锁定开关**：PUT /iot/instance/lock 复用 iot:instance:edit；locked 列白名单更新（0012 口径 locked=1 编辑仅 status/remark 的接口层校验）；数据权限照 0012 写路径防线
2. **待升级清单**：applied<current 筛选正确性（无差异不出现/回退防御跳过 WARN）；diff 复用 0015 引擎逐条目带档位——**当前侧 content 构建与版本历史行同一 canonical 构建器**（buildCurrentContentJson 扩展方法）的正确性；无基线降级；分页/实例域读裁剪
3. **确认流**：targetVersion==模板当前版校验；逐台乐观锁（冲突 4002 口径）；**整批原子不半成功**（事务边界）；locked 实例可确认；@OperateLog+服务逐台留痕；applied==target 按 4002 拒绝（消除驱动 affected 歧义——执行者偏离 3）
4. **并发安全**：确认与发布组装并发时的竞态（applied 更新瞬间组装读到新旧半态？）；确认与锁定开关并发
5. **5002 确认门联动**：本任务接口不触发布门（确认升级只是簿记+数据面编排入口）——核实无意外联动
6. **执行者裁决项核实**：报告「关键登记」的**数据面生效缺口**外推是否成立（ConfigSnapshotAssembler 读实例行、确认升级仅簿记——该缺口影响 ADR-0008「确认后按新版本生效」语义，协调者将据此立后续任务；评审者独立确认代码事实）
7. 0015 引擎复用无改动面（版本服务/diff 引擎/tier 表是否被本 diff 修改——应只读复用）
8. 数据权限/租户插件一致性（清单与确认的租户边界）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0016-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
