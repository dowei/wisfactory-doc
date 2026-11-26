# 评审 brief：TASK-0007 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断口径不对称修复是否正确、有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原实施报告（观察项与红线套件记录）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0007-report.md`
2. 修复 diff 包（5 文件约 225 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0007-fixdiff.txt`
3. 协调者裁决口径：详情判定补 create_by=me 分支（instance 详情 + dataitem 清单/params 读路径同口径——自建实例可见即可开，与列表 OR 分支对齐，ADR-0016 裁决 3「仅本人适用集=iot_ 配置表的管理面查询」含详情类）；写防线不动。
4. 实施报告修订轮节（末尾）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0007-report.md`

## 判定标准

- 口径修齐是否正确：instance 详情与 dataitem 清单/params 三处读路径一致接受 create_by=me；他建实例仍 403；写防线未动。
- 修复 diff 有无新引入缺陷（create_by 语义正确性=创建者 userId、范围蔓延、断言放松）。
- 红线套件观察项转正的断言质量。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0007-review-round2.md`，含：结论（口径修齐完成/未完成）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
