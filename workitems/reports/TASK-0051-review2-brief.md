# 评审 brief：TASK-0051 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断 I-1~I-4 是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 I-1~I-4 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-review.md`
2. 修复 diff 包（约 1135 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-fixdiff.txt`
3. 协调者裁决口径：I-1 openapi 块移出 dataperm 块顶层独立 + ot.dataperm 两键回归 + 命名空间锁测试；I-2 ORDER BY ts DESC LIMIT 1 + fake 单测；I-3 回填键 TTL 配置化默认 60s；I-4 fetch-one-extra 截断判定 + truncated 字段 + X-Truncated 响应头。
4. 实施报告修订轮节（末尾）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-report.md`

## 判定标准

- I-1：yml 结构正确（ot.openapi 顶层独立/ot.dataperm 两键回归原命名空间）+ 命名空间锁测试质量（锁的是结构还是脆断言）。
- I-2：「最新值」真是最新（超限场景实证）。
- I-3：回填键 TTL 在位且过期后回源路径不被破坏。
- I-4：截断判定正确（fetch-one-extra 语义——恰好 20000 行时不误标）+ 对外契约标记一致（字段+响应头）。
- 修复 diff 有无新引入缺陷（降序查询对既有调用方的影响、TTL 配置键命名空间、断言放松）。
- 原 11 条 Minor 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-review-round2.md`，含：结论（四项已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
