# TASK-0089 复评报告（round2 · 终审）

> 评审对象：隔离修订提交 `ba0a908`（父 `ee767db`），2 文件（index.vue + 多行测试）。首评 1 Important（I-1）退回一轮，本轮为唯一复评、终审（「1 退回 + 1 复评」上限已用尽）。
> 角色：独立 fresh 评审（no-subagents，只评审不修改，报告除外）。所有结论均回到源码、git 与**本地实测**独立核实，未采信执行者报告自述。

## 结论

**通过（I-1 闭合，无新 C/I）。**

- I-1（Important）**真修复**：`valueOf()` 由取 ts 升序首行改为取同 `metricCode` **最大 ts 行**（本期行），旧值问题消除；仅按 metricCode 定位、value/yoy/mom 同源口径不变；仅 1 行 / 无对比行 / 无匹配三种场景均正确，未引入新空态或错值。
- 多行测试**真实红转绿、非恒真**：独立实测旧版 `.find()` 下该用例必红且实际渲染去年旧值 `123.4`，修后绿。
- 其余五项首评通过项零回归；四门全绿（631 = 630 基线 + 1）；范围恰 2 文件、越界 0、区间夹入 0037 backend 已隔离。
- 无 Critical、无新 Important、无 Minor 需 deferred。按终审口径判定通过。

---

## 一、发现清单

| 编号 | 严重级 | 出处（绝对路径 + 位置） | 一句话说明 |
| --- | --- | --- | --- |
| — | — | — | 本轮无新增发现（0 C / 0 I / 0 Minor）。首评 I-1 已闭合，见第二节判定。 |

---

## 二、I-1 闭合判定（终审核心）

### 2.1 修复实现（回到源码核实）

`D:\workspace\WisFactory\frontend\src\views\display\metrics\index.vue:122-133`：

- 旧版（9459e34）：`return valueRows.value.find((r) => r.metricCode === metric.metricCode) ?? null;` —— 取 ts 升序首行。
- 修订版（ba0a908）：遍历 `valueRows.value`，对同 `metricCode` 行取 `row.ts > latest.ts` 的最大 ts 行返回；无匹配时 `latest` 仍为 `null` 返回 null。
- 口径保持：仅按 `metricCode` 定位（未引入 period/instance 约束），整行同源喂给卡片，value/yoy/mom 未现算。改动仅在 `index.vue` 内联，未 import、未改任何组件/纯函数。

### 2.2 「最大 ts 行是否真是本期行」——成立（独立回到后端源码核实）

证据链逐条实测：

1. `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/support/OpenApiTimeseriesClient.java:207-212`：`queryMetrics` SQL `... AND ts >= startMs AND ts <= endMs ORDER BY ts ASC LIMIT ...`。同码多行按 ts 升序返回。
2. 同文件 `:228` `vo.setWindowStart(formatWindowStart(row.get(0)))` —— VO 的 `windowStart` 取自结果行 `ts` 列，即该行统计窗口起始时间；`:229-235` value/yoy/mom 取自同一行，同源。
3. `backend/.../iot/displayread/impl/IotDisplayReadServiceImpl.java:96-99`：`new MetricValueVO(..., parseWindowStart(vo.getWindowStart()), vo.getValue(), vo.getYoy(), vo.getMom(), ...)` —— 只做 `windowStart → ts` 形态映射、period 回填，**不改顺序、不改值**；`parseWindowStart`（:135-145）按 Asia/Shanghai 转 epoch 毫秒，不可解析返回 null（不现算）。
4. `backend/.../openapi/service/impl/OpenApiQueryServiceImpl.java:285-295`：按 code 循环 `result.addAll(page.rows())`，保持每个 code 内部 ts ASC 的子序列。
5. `frontend/src/api/display/display-api.ts:238-253` `MetricValueRow.ts` 注为「窗口起始 epoch 毫秒（ts 列）」；`index.vue:159-171` 查询窗 `start = end - 366d`，`valueRows.value = valueResults.flat()` 无排序无截断，保序装入。
6. 结论：366 天窗内同一 `metricCode` 的行按 ts 升序排列，去年同 ts 对比行的 ts 恰为对应本期行 ts − 1 年（更小）。因此**最大 ts 行就是本期最新窗口行**，取最大 ts 必然避开去年对比行；与 `metric-utils.ts:112-120` 的 comparisonTs 识别口径（ts 恰晚 1 年者为对比行）语义一致且更稳健。I-1 成因（取首行=最旧=去年对比行）被消除。

### 2.3 「仅 1 行 / 无对比行」场景仍正确（逐场景核实）

- **仅 1 行**：循环命中唯一同码行 → `latest = 该行`，返回该行，与旧版 `.find()` 行为完全一致，无回归；既有 412.6 单行用例全绿佐证。
- **无匹配**（该 metricCode 无任何值行）：`latest` 始终为 `null` → 返回 null → `metric-card` 走「无本期值」空态；空态纪律保留，未引入新空态/伪造值。
- **多行但无去年对比行**（全是本期不同 ts）：取最大 ts 行（最新本期值），比旧版取最早行更准，无错值——旧版在此场景本就取到较旧本期值，修订后更正确。

### 2.4 多行红转绿——本地独立实测（非采信报告）

新增用例：`frontend/tests/unit/display/metrics-board-read-ports.spec.ts:78-100`，造同 `M_ENERGY` 两行、按 ts ASC 注入（去年对比行 ts=T−1年 value=123.4 yoy/mom=null；本期行 ts=T value=412.6 yoy=0.083 mom=-0.021），断言含 412.6/+8.3%/-2.1%、不含 123.4/+0.0%。

- **修后绿**：主工作区 `pnpm exec vitest run tests/unit/display/metrics-board-read-ports.spec.ts` → `Test Files 1 passed / Tests 6 passed (6)`。
- **旧版必红（独立构造）**：用 `git worktree add --detach D:/audit-0089-round2 ba0a908` 建隔离工作树，仅把该 worktree 内的 `valueOf` 回退为旧版 `.find()`（保留新增多行测试），复用已装依赖运行同一用例 → `Tests 1 failed | 5 passed (6)`，失败断言 `expected ... to contain '412.6'`，且 Received 实际文本渲染为 `用电日123.4kWh 同比 — 环比 —`。**实测证明旧版确把去年旧值 123.4 当本期值显示**，测试非恒真、非改测试凑绿。验证后已移除该 worktree 及软链，主工作区 `index.vue` 的 sha256 前后一致（`fc62d4fd…769f8c`），tracked 文件零改动。

---

## 三、零回归核实（本地实测）

| 门禁 | 命令 | 结果 |
| --- | --- | --- |
| typecheck | `pnpm typecheck`（vue-tsc --noEmit） | exit 0，无输出 |
| lint（本轮 2 文件） | `pnpm exec eslint <index.vue> <spec.ts>` | exit 0，0 warning/error |
| lint（全量） | `pnpm lint` | exit 0；9 problems(0 errors, 9 warnings) 全为既存无关文件 |
| build:test | `pnpm build:test` | exit 0，`✓ built in 1m 26s` |
| test（全量） | `pnpm test` | exit 0，`Test Files 69 passed (69)`、`Tests 631 passed (631)` = 630 基线 + 1 新增，零回归 |

- 其余五项首评通过项（安全红线 / 空态诚实 / 端点对齐 / 组件纯函数零改 / 范围）本轮 diff 未触碰：修订仅改 `valueOf` 与新增 1 用例，未引入新 URL/凭据/代理，未改空态与 readGap 判定，`metric-card.vue`/`metric-trend.vue`/`metric-utils.ts`/`device-utils.ts` 零改动（git 范围反查见第四节）。
- `menu-cache-frame.spec.ts` 高负载偶发 5000ms 超时：本轮全量连跑**未复现**（69/69 绿）；首评与修订报告均按环境抖动隔离登记，未改测试掩盖，判定合理。

---

## 四、范围 / 越界反查（git 独立核实）

- `git diff-tree --no-commit-id --name-status -r ba0a908` = 2 行：M `frontend/src/views/display/metrics/index.vue`、M `frontend/tests/unit/display/metrics-board-read-ports.spec.ts`。
- `git diff --name-only ee767db..ba0a908` = 上述 2 行，全落 brief 裁决「只改 index.vue + 相关测试」；组件/纯函数/backend/calc/ingest/contract 零触碰。
- 提交链：`9459e34`（主体 6 文件）→ `ee767db`（TASK-0037，17 文件全 `backend/.../dataitem/**` + `application.yml`）→ `ba0a908`（本轮 2 文件）。区间 9459e34..ba0a908 共 19 文件 = 2（本轮）+ 17（0037 已隔离），0037 属已独立闭环任务，不在本轮范围。
- 隔离 diff 包 `TASK-0089-diff-round2.txt`（58 行）内容与 ba0a908 实际 diff 一致，隔离可信。

---

## 五、契约核对与终审口径

- **fresh / no-subagents / 只评审不修改（报告除外）**：本轮新会话，未派子代理；曾在隔离 worktree 内临时回退 `valueOf` 做红转绿实测，验证后完整移除，主工作区源码字节级未变、tracked 文件零改动；仅写本报告。
- **终审口径**：0089 已用尽「1 退回 + 1 复评」上限。I-1 闭合且无新 C/I → 通过；否则如实不通过。本轮 I-1 已闭合、无新 C/I，故判**通过**。TASK 状态、索引、提交推进均不在本评审职责内，未改动。

---

## 一句话结论

**终审通过**——I-1 经源码+后端语义+本地红转绿实测三重独立核实确认真修复（取同码最大 ts 本期行，旧版必红/修后绿、非恒真，仅 1 行/无对比行场景均正确），四门全绿、631 用例零回归、范围恰 2 文件零越界，无新 C/I，I-1 闭合。
