# 评审 brief：TASK-0006 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断 I-2 是否修复、修复本身有无新缺陷；I-1 已由 TD-0055 裁决为残余风险不在判定范围）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-review.md`
2. 修复 diff 包（约 112 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-fixdiff.txt`
3. 裁决口径：I-1 接受为残余风险（TD-0055，todo.md）不改码；I-2 createInstance/deleteInstances 各一行 evictAll（授权扩展 brief「实例侧仅一处」至三处挂点）。
4. 实施报告修订轮节（末尾）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-report.md`

## 判定标准

- I-2 是否消除：新建绑部门实例与删除实例后全量失效即时生效（SCAN 为空），缓存与实例状态不再有 10 分钟错位窗。
- 修复 diff 有无新引入缺陷（evictAll 时机/事务边界、测试断言放松）。
- 原 6 条 Minor 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-review-round2.md`，含：结论（I-2 已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
