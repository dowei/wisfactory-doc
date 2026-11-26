# 评审报告：TASK-0012 修订轮 scoped 复评

- 评审者：fresh reviewer（brief：reports/TASK-0012-review2-brief.md，四项裁决口径 + 原 8 条 Minor 是否恶化）
- 评审对象：修复 diff 包 `TASK-0012-fixdiff.txt`（提交 46e923d，2 文件 503 行，已完整读入），并实测核对仓库 HEAD（46e923d）与 diff 逐点一致（`git diff c9770ef..46e923d --stat` = 2 文件，均在 instance 域声明内）；独立复跑 `IotModelInstanceServiceImplTest`：**tests=47, failures=0, errors=0**（surefire 实测，与报告 §6.3 一致）。
- 范围：仅判定原 I-1/I-2 与 M-3/M-7 是否修复、修复本身有无新缺陷；不重开全场，原 8 条 Minor 中余 6 条（M-1/M-2/M-4/M-5/M-6/M-8）deferred 项未动、未恶化。

## 一、逐项判定（全部已修复）

1. **I-1 写路径数据权限防线：已修复。** 三写路径全部补 resolve+可见集成员校验（`requireDataPerm`，allTenant 直过）：`updateInstance`（存在性校验后、不可变校验前，`IotModelInstanceServiceImpl.java:287`）；`deleteInstances`（存在性后、锁定前，批内任一不可见即整批 403，:373-382）；`createInstance` 仅 parent!=null 时校验父实例 ∈ 可见集（:146-148），顶级创建不受限——与裁决「update/delete 目标 ∈ 可见集、create 父 ∈ 可见集、顶级创建不受限」逐字吻合。403 文案（「无权操作该实例（数据权限不命中）」/「无权在该实例下创建子实例（数据权限不命中）」）不含实例编码/名称/id，与详情同范式。防线顺序 fail-closed 且不泄露状态：编辑=存在→权限→不可变（测试 verify deptMapper/templateMapper/instanceCodeMapper 均未触）；删除=存在→权限→锁定→子保护（locked+不可见对受限用户 403 不泄露锁定状态、对 admin 400，冒烟 R2j/R4 双形态取证）。评审指出的「改绑 dept_id 反向扩大可见集」升级链：目标不在可见集即 403，链路封死（冒烟 R2h + DB 复核 R2k）。
2. **I-2 混合角色可见集：已修复。** 树/列表短路条件改仅 `!scope.allTenant() && scope.instanceIds().isEmpty()`（:420、:546），createByOnly 不再触发清空——混合角色（测试夹具 instanceIds={2,4}+createByOnly=true）树返回部门可见集+骨架（红转绿用例 testTree_MixedRoleKeepsDeptVisibleSet，冒烟 R1b：WS01/EQ001 在、FMIX 不在），列表走 `AND (id IN 可见集 OR create_by=me)`（:552-561，`wrapper.and` 正确括号包裹、无 IN () 空集 SQL、参数绑定无注入；冒烟 R1c：IN 分支 3/6 + create_by 分支 FMIX）。树不挂 create_by 分支——层级导航视图 scope=4 不贡献树可见性，与裁决「树不挂」一致。**口径备注（非缺陷）**：纯 scope=4 用户的列表行为由空集变为 create_by=me——此为裁决「列表 OR create_by 分支」按 ADR-0016 裁决 3/4 配置类语义的退化形（可见集空 → 仅 create_by=me），实施在提交说明/用例 DisplayName/报告 §6.1/冒烟全程明示，无隐瞒；判定标准「纯 scope=4 行为不变」按短路语义读（树仍空树、不查库，实测保持），若协调者另有意裁需显式翻案。
3. **M-3 锁定禁删：已修复。** `deleteInstances` 增 locked=1 → 400「实例已锁定，禁止删除」（:384-390），批内任一锁定即整批拒；位置在数据权限防线之后——无权者不见锁定状态（测试 testDelete_LockedAndInvisible403 断言 403 且文案不含「锁定」）。admin 亦拦（冒烟 R4）。
4. **M-7 骨架字段收窄：已修复。** `buildTree/assemble` 增加 fullFieldIds 参数：allTenant 全字段；受限用户 fullFieldIds=可见集，可见集之外的到根路径节点走 `skeletonVo`（仅 id/instanceCode/instanceName/nodeType/status，:524-531），与裁决字段面逐字一致且与详情 403 不泄露面对齐。测试对骨架节点断言 deptId/templateId/appliedTemplateVersion/locked/ancestors/parentId/sort 全 null、可见节点全字段对照；冒烟 R5 逐节点键清单实况吻合。

## 二、新发现（修复自身引入）

无。

核查过并排除的点（留痕）：修复 diff 未触碰 deferred 六项（M-1/M-2/M-4/M-5/M-6/M-8 均原样，未恶化）；删除批重复 id（原 Minor 2）路径未变；红转绿 9 条红均恰为四项新行为的被测断言（其余 4 条新增用例为不误伤守卫，旧实现本就绿，符合预期）；报告自陈的测试内夹具笔误修正（"0,2"→"0,1"）系测试 bug 修正非断言放松；混合角色「列表可见（create_by 分支）但写需 ∈ 可见集」的不对称是裁决口径的字面结果，非本修复缺陷；I-1 存在性校验先于权限校验的 400/403 区分与详情读路径同范式。

## 三、结论

四项（I-1/I-2/M-3/M-7）**全部已修复**，修复本身未引入新缺陷，新发现 0；判定理由：diff 与裁决四项逐字吻合、防线顺序与不泄露面经测试与冒烟双证，仓库 HEAD 实测一致且测试类独立复跑全绿（47/47）。
