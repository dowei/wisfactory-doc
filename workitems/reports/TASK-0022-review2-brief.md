# 评审 brief：TASK-0022 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断 I-1 导入引用存在性校验是否修复、修复本身有无新缺陷；不重开全场；0023 节不在范围）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-0023-review.md`
2. 修复 diff 包（3 文件约 213 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-fixdiff.txt`
3. 协调者裁决口径：导入逐行校验补 deptId/boundInstanceId 存在性/同租户/未删（一次 IN 批查）、新原因码 INVALID_REF、整批拒。
4. 实施报告修订轮节（末尾）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-0023-report.md`

## 判定标准

- I-1 是否消除：导入含不存在/他租户/已删 deptId 或 boundInstanceId 的行 → 该行 INVALID_REF 失败整批拒；悬空引用通道封死。
- 修复 diff 有无新引入缺陷（IN 批查性能、原因码枚举一致性、断言放松）。
- 原 Minor 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-review-round2.md`，含：结论（I-1 已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
