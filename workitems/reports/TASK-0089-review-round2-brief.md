# 评审 brief：TASK-0089 scoped 复评 round2（修订轮）

> 隔离修订 diff：9459e34..ba0a908，2 文件（metrics/index.vue + 多行测试）。首评 I-1（Important，指标卡片取去年旧值）退回一轮，本轮为唯一复评。区间夹入的 17 个 backend 文件属已闭环的 TASK-0037（ee767db），已用单提交隔离排除。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」；scoped 复评只覆盖 I-1 修复 + 回归确认）

## 必读（只给路径，不粘正文）

- **本轮隔离 diff**：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0089-diff-round2.txt`（2 文件，58 行）
- 首评报告：`reports\TASK-0089-review.md`（I-1 原文 + 五项通过判定）
- 退回 brief：`reports\TASK-0089-revision-brief.md`（裁决 + 修复方案）
- 修订报告段：`reports\TASK-0089-report.md`（追加的「修订轮」段）
- TASK 详情：`tasks\TASK-0089.md`（任务结果段含裁决）
- 权威证据：`frontend/src/views/display/metrics/index.vue:122-124`（原 valueOf .find）、`:152/:163`（366 天窗 + flat）、`backend/.../openapi/support/OpenApiTimeseriesClient.java:212`（ORDER BY ts ASC）、`frontend/src/views/display/metrics/metric-utils.ts:100-120`（去年对比行识别口径）

## 越界反查

- 修订 ba0a908 恰 2 文件（index.vue + metrics-board-read-ports.spec.ts）；区间 9459e34..HEAD 中的 17 个 backend 文件属 TASK-0037（ee767db，已独立闭环评审），本 diff 用单提交隔离、零 backend 内容；组件/纯函数/backend/calc/ingest/contract 本轮零触碰。

## 复评重点

1. **I-1（Important）真修复**：`valueOf()` 是否从 `.find()` 改为取同 metricCode **最大 ts 行**（本期行），避免 ts 升序首行=去年对比行；保持按 metricCode 定位口径、value/yoy/mom 同源不现算。核实修复是否会在"仅 1 行"或"无对比行"场景仍正确（不应引入新空态/错值）。
2. **多行测试有效性（红转绿）**：新测试是否造同 metricCode 2 行（去年对比行 ts=T-1年 + 本期行 ts=T，按后端 ts ASC 顺序注入）、断言 valueOf 取本期行（value/ts 都是本期）；旧版 `.find()` 必红（显示去年旧值 123.4 而非本期 412.6）、修后绿；非恒真、非改测试凑绿。
3. **零回归**：其余五项首评通过项（安全红线/空态诚实/端点对齐/组件纯函数零改/范围）本轮零回归；pnpm 四门 0 错误、631 用例（630 基线+1）全绿；menu-cache-frame 高负载偶发超时隔离通过按环境抖动登记合理。
4. **范围**：仅 2 文件、越界 0、区间夹入 0037 已隔离。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0089-review-round2.md`。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。对 I-1 是否闭合、旧版必红是否真实、是否有新 C/I 给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
