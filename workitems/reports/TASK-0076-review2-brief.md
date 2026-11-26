# 评审 brief：TASK-0076 修订轮 scoped 复评

## 角色

- 代码评审（修订轮 scoped 复评：只判断 C1/I1/I2/I3 四项是否修复、修复本身有无新缺陷；不重开全场）

## 必读（只给路径，不粘正文）

1. 原评审报告（原发现 C1/I1/I2/I3 与 Minor 清单）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-review.md`
2. 修复 diff 包（7 文件约 725 行）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-fixdiff.txt`
3. 协调者裁决口径：C1 原子 MarkIfSeen（单锁内 check+mark）+ Submit 并发注释修正 + 竞态回归测试转正；I1 per-key 串行化（singleflight/双检锁——同键回源串行异键并行）；I2 对外固定文案细节进服务端日志；I3 注释改实际行为 + 0020/0027 接线义务登记（status 实校验不强制本期）。
4. 实施报告修订轮节（TASK-0076-report.md §6）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-report.md`

## 判定标准

- C1 是否消除：MarkIfSeen 原子性（check+mark 单锁内）；并发 Submit 同 batchId 恰一处理（竞态回归 30 轮×12 goroutine 零复现 + -count=10 复跑）；Submit 注释承诺与实际一致。
- I1/I2/I3 是否落地且无新缺陷（per-key 语义正确性、固定文案不泄露、I3 注释与登记）。
- 修复 diff 有无新引入缺陷（锁粒度、per-key map 本身的并发安全、死锁面、断言放松）。
- 原 8 条 Minor 不在本轮判定范围（已 defer），除非修复使其恶化。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-review-round2.md`，含：结论（四项已修复/未修复）、新发现（分级，没有写无）、一句判定理由。

## 契约

- fresh——不向协调者追问；no-subagents；只评审不修改任何文件。

完成后，你的最终回复只需一句话：复评结论（已修复/未修复 + 新发现数）。
