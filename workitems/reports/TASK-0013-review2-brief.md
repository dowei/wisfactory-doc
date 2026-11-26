# 评审 brief：TASK-0013 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断原两条 Important 与 M-5/M-6 是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1/I-2 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-review.md`
2. 修复 diff 包（2 文件约 252 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-fixdiff.txt`
3. 协调者裁决口径（四项）：I-1 生成循环末尾展开终检（formula 残留 `{`/`}` 整批拒，文案含原始编码与 token 示例）；I-2 selfScope 只并 newItems 编码（引用已删占用码 → 展开期失败整批，ref 不落悬空边）；M-5 剥壳键 >180 补长度校验入 paramError；M-6 source_type=2 且公式空白整批拒。
4. 实施报告修订轮节（末尾，含红转绿与冒烟原始输出）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-report.md`

## 判定标准

- 原 I-1 是否消除：畸形/超纲 token（含空格、非白名单字符集）不再静默落库——终检兜底 + 生成整批拒；正常公式路径不误伤。
- 原 I-2 是否消除：引用已删占用码的公式在展开期失败；ref 图无悬空边（ref_item_id 不再指向死键）。
- M-5/M-6 是否落地且无新缺陷（校验位置、文案、断言收紧、对既有正常路径无回归）。
- 原 Minor #3/#4/#7 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-review-round2.md`，含：结论（四项已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
