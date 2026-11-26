# 评审 brief：TASK-0019 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断 I-1 并发轮换丢失更新是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-review.md`
2. 修复 diff 包（3 文件约 225 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-fixdiff.txt`
3. 协调者裁决口径：轮换改原子条件 UPDATE（WHERE id=? AND key_version=<前置读> AND deleted=0，key_version 服务端自增）——0 行 = 并发冲突 → 5001 VERSION_CONFLICT（不引入 @Version）；密文/掩码落库与响应回填取舍已在报告说明。
4. 实施报告修订轮节（末尾，含红转绿与冒烟原始输出）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-report.md`

## 判定标准

- 原 I-1 是否消除：并发轮换恰一方成功（key_version 恰 +1）、败者 5001 不覆盖、刷新重试恢复——丢失更新通道封死。
- 原子 UPDATE 的正确性（WHERE 谓词完整性/事务边界/密文与掩码一致性——密文更新后掩码不同步是隐患）。
- 修复 diff 有无新引入缺陷（5001 语义全局性、断言放松、审计不受影响）。
- 原 7 条 Minor 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-review-round2.md`，含：结论（I-1 已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
