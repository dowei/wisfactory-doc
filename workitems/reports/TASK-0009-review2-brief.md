# 评审 brief：TASK-0009 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断原三条 Important 是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1/I-2/I-3 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-review.md`
2. 修复 diff 包（8 文件约 1030 行，含新自实现的流式解析器）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-fixdiff.txt`
3. 协调者修复裁决：I-1 行号保真（物理行占位/关 ignoreEmptyRow）、I-2 关 alias 替换（首数据行同口径 MISSING_REQUIRED）、I-3 流式解析+5000 行 fail-fast（brief 提议 Hutool SAX，执行者实证 Hutool 5.8.25 SAX 在本 classpath 静默零回调，**改用 POI XSSFReader+安全 XMLReader 自实现流式**，偏离依据在实施报告修订轮节）
4. 实施报告的「修订轮」一节（末尾，含补充验收与偏离依据）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-report.md`
5. 首轮实施 brief（对照原契约）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-brief.md`

## 判定标准

- 原 I-1 是否消除：含空行文件回执行号=Excel 物理行号（实测 [2,4,5]）；行号契约不再依赖「无空行」前提。
- 原 I-2 是否消除：首数据行缺格触发 MISSING_REQUIRED，"A"/"B" 列名替换通道封死，垃圾码不可静默落库。
- 原 I-3 是否消除：解析流式化、5000 行 fail-fast 有界内存；**重点审自实现流式解析器**——POI XSSFReader + XMLReader 的资源关闭（流/包/临时文件泄漏）、XML 实体注入防护（XXE：XMLReader 特性配置）、异常路径的解析中断语义、与旧 DOM 路径的替换完整性（旧路径无残留）。
- 原 6 条 Minor 不在本轮判定范围（已记 deferred），除非修复使其恶化；Hutool→POI 偏离本身按「评审意图 vs 字面 brief」判定（意图=有界流式，达成即符合）。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-review-round2.md`，含：结论（三条 Important 已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
