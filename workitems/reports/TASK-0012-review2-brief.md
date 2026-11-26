# 评审 brief：TASK-0012 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断原两条 Important 与 M-3/M-7 是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1/I-2 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-review.md`
2. 修复 diff 包（2 文件约 503 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-fixdiff.txt`
3. 协调者裁决口径（四项）：I-1 写路径 resolve 校验（update/delete 目标 ∈ 可见集、create 父 ∈ 可见集、顶级创建不受限、403 不泄露标识）；I-2 短路仅 instanceIds().isEmpty() + 列表 OR create_by 分支（树不挂——层级导航不贡献）；M-3 locked 禁删；M-7 骨架字段收窄（id/instanceCode/instanceName/nodeType/status）。
4. 实施报告修订轮节（末尾，含红转绿与冒烟原始输出）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-report.md`

## 判定标准

- 原 I-1 是否消除：三写路径任何受限上下文下不可写不可见目标（含改绑部门升级链封死）；403 文案不泄露实例标识；顶级创建不受限口径落地。
- 原 I-2 是否消除：混合角色（scope=3+4 并存）树/列表不被清空；列表含 create_by=me 自建实例、树不含（裁决分野正确）；纯 scope=4 行为不变。
- M-3/M-7 是否落地且无新缺陷（守卫位置、VO 字段面、断言收紧）。
- 原 8 条 Minor 不在本轮判定范围（余 4 条 deferred、M-6 层级转 0013），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-review-round2.md`，含：结论（四项已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
