# 评审 brief：TASK-0021 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断 I-1 remark 丢失与 I-2 写路径无数据权限是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1/I-2 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-review.md`
2. 修复 diff 包（约 348 行，BASE a61fd24 → a961fd1）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-fixdiff.txt`
3. 协调者裁决口径：I-1 createPoint 白名单补 setRemark + 用例；I-2 写路径三方法（updatePoint/changePointStatus/deletePoints）补 resolve 防线——可见口径=（allTenant）或（dept_id ∈ 部门集）或（bound_instance_id ∈ 可见集），与列表同条件；不可见 403 不泄露标识；资产类无 create_by 分支（fail-closed）。
4. 实施报告修订轮节（末尾）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-report.md`

## 判定标准

- I-1：remark 落库路径打通（建档携带→回读一致）；白名单与表单字段一一对应原则恢复。
- I-2：三写路径任何受限上下文下不可写不可见点位（横向越权写封死）；403 文案不泄露点位标识；可见点位写操作不误伤；Redline 28 用例含新增写越权。
- 修复 diff 有无新引入缺陷（可见口径与列表一致性、事务边界、断言放松）。
- 原 8 条 Minor 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-review-round2.md`，含：结论（两项已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
