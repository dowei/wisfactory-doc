# 评审 brief：TASK-0008 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断原 Important 发现是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-review.md`
2. 修复 diff 包（仅 2 文件，约 209 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-fixdiff.txt`
3. 实施报告的「修订轮」一节（末尾，含补充验收原始输出）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-report.md`
4. 协调者裁决口径：新增与编辑全量字段白名单（照 SysTenantServiceImpl.updateTenant 既有范式）——新增仅收 dataCode/dataName/category/dataType/defaultUnit（status 强制 1、deleted 强制 0、审计字段框架填充）；编辑仅 dataName/category/defaultUnit 可变（dataCode/dataType/status/deleted 排除在更新集外，status 变更只走 DELETE 禁用端点）。

## 判定标准

- 原 I-1 是否消除：POST/PUT 任何载荷组合下，落库行不可能出现 deleted=1 或非预期 status；「编码永不复用」不变量不再可能被幽灵行破坏。
- 修复 diff 有无新引入缺陷（白名单遗漏字段、校验顺序、测试断言放松等）。
- 原 3 条 Minor 不在本轮判定范围（已由协调者记 deferred），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-review-round2.md`，含：结论（I-1 已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
