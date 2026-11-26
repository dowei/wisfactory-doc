# 评审 brief：TASK-0005 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断原两条 Important 是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1/I-2 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-review.md`
2. 修复 diff 包（约 516 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-fixdiff.txt`
3. 裁决载体：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\decisions\ADR-0016.md` 文末「增补（2026-11-22）」第 8/9 条
4. 实施报告的「六、修订轮」一节（末尾，含补充验收原始输出）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-report.md`

## 判定标准

- 原 I-1 是否消除：中台路径（resolveClientScope 衍生的点位/时序条件）任何情况下不得出现非空 deptIds；点位条件仅实例分支；被修正的测试断言与 ADR-0016 增补第 8 条一致且未放松其他既有断言。
- 原 I-2 是否消除：resolve/resolveClientScope 无租户上下文时返回空 VisibleScope + WARN（格式与增补第 9 条一致）；守卫不得误伤正常登录上下文路径。
- 修复 diff 有无新引入缺陷（守卫的空值语义、测试隔离、断言放松等）。
- 原 6 条 Minor 不在本轮判定范围（已记 deferred），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-review-round2.md`，含：结论（两条 Important 已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
