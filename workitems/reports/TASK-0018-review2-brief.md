# 评审 brief：TASK-0018 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断原两条 Important 是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1/I-2 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-review.md`
2. 修复 diff 包（约 217 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-fixdiff.txt`
3. 协调者裁决口径：required 判定改「键存在且值非 null 且字符串非空白」（显式 null 按 missing 处理）+ 凭据关键词扫描全深度递归（数组元素与嵌套对象都进）。
4. 实施报告修订轮节（末尾）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-report.md`

## 判定标准

- 原 I-1 是否消除：显式 null/空白字符串的 required 字段按缺失拒绝；正常值不误伤。
- 原 I-2 是否消除：任意层级的凭据关键词键名（数组元素/嵌套对象）一律拒；正常嵌套业务键不误伤。
- 修复 diff 有无新引入缺陷（递归深度限制、判定顺序、断言放松）。
- 原 5 条 Minor 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-review-round2.md`，含：结论（两项已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
