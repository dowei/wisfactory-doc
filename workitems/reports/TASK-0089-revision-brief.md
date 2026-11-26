# TASK-0089 diff 评审退回修订 brief（一轮退回：I-1 指标卡片取到去年旧值）

> 主体提交 9459e34（6 文件纯 frontend），四门验收全绿但评审发现 1 Important。本轮只修 I-1。续用原执行者上下文（agent_56f76a37）。

## 必读（只给路径，不粘正文）

- 首评报告：`reports\TASK-0089-review.md`（I-1 原文 + 五项通过判定 + 安全红线/空态通过）
- TASK 详情：`tasks\TASK-0089.md`（任务结果段含裁决）
- 实施 brief + 主体报告：`reports\TASK-0089-brief.md` / `TASK-0089-report.md`
- diff：`reports\TASK-0089-diff.txt`
- **缺陷证据（已独立核实）**：
  - `frontend/src/views/display/metrics/index.vue:122-124` `valueOf()` 用 `.find(r => r.metricCode===...)` 取首行；`:152` 查询窗 366 天；`:163` `valueRows.value = valueResults.flat()`
  - `backend/.../openapi/support/OpenApiTimeseriesClient.java:199-212` `queryMetrics` 用 `ORDER BY ts ASC`
  - `frontend/src/views/display/metrics/metric-utils.ts:100-120` 已有"去年同期行识别"口径（ts 恰晚 1 年 → 较早行是对比行）；`:146 oneYearBefore`

## I-1（Important，必须修）：指标卡片"本期值"取到一年前旧值

**问题**：本批把 valueRows 从恒空改成真值后，metricValues 查 366 天窗（覆盖去年同 ts 对比行），后端按 ts 升序返回同 metricCode 多行。`valueOf()` 的 `.find()` 取**最早行**（去年对比行）喂给 metric-card 当"本期值"显示。数值真实但周期错误，属核心交付物错误结果。该 valueOf 在 BASE 即存在（0060 review D-4 已登记 deferred："读口返回多行需补 period/instance 约束"），本批首次激活后暴露，判为本批引入的实际缺陷。新增测试每码仅 1 行未覆盖多行。

**修复方案（协调者裁决）**：
1. `index.vue` 的 `valueOf(metric)` 改为选**同 metricCode 的本期最新行**——按 ts 升序时取**最大 ts 行**（本期行 ts 一定晚于去年同期对比行；与 metric-utils 的 comparisonTs 识别口径一致但更简单稳健）。若同 metricCode 有多实例行，保持现有"按 metricCode 定位"口径不改（tenant 内 metricCode 唯一，value/yoy/mom 同源）。
2. **不改组件/纯函数**（metric-utils.ts/buildMetricTrend/metric-card.vue 等仍零改动）——修复只在 index.vue 的 valueOf 内联完成；如需复用 oneYearBefore 可从 metric-utils import（该函数已是纯函数、只读，不改它）。
3. 补多行契约测试：同 metricCode 造 2 行（去年对比行 ts=T-1年 + 本期行 ts=T）→ `valueOf` 取**本期行**（断言 value 与 ts 都是本期的）；旧版 `.find()` 必红（取到去年行）。其余 5 项评审通过项（安全红线/空态/组件零改/端点对齐/范围）零回归。

## 验收命令（沿用原始）

1. `cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test` → 0 错误 0 失败（含新增多行断言，旧版必红；既有 630 零回归），贴原始输出
2. 范围核对：`git diff --name-only 9459e34..HEAD` 只落 index.vue + 相关测试，组件/纯函数/backend/calc/ingest/contract 零触碰

## 交付

- 修订提交（注明 TASK-0089 修订轮 I-1）；报告追加「修订轮」段：I-1 修复 + 多行红转绿 + 为什么现在对了 + 其余零回归。
- no-subagents；只改最小范围；不置 TASK 已完成；报告写到 `reports\TASK-0089-report.md`（追加段）。
