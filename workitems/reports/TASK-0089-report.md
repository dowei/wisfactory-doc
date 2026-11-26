# TASK-0089 执行报告：展示端接入管理面值读口（设备实时区与指标看板接真数据）

- 执行者：单执行者（no-subagents）
- BASE：`2d0f1a9`
- 提交：`9459e34fc26f71bab8e7e7daa2219eac5680bc1c`（父 `2d0f1a9`；单一提交，未 push）
- 改动文件数：**6**（源码 3 + 新增测试 3），全部前端，全部落 brief 声明范围
- TASK 状态：保持「进行中」（执行者不置已完成、不改 workitems 索引）
- 注：开工时工作区已有别的任务预暂存的 `backend/ot-admin/.../dataitem/**` 变更（17 项）；本任务用路径受限提交严格隔离，这些文件从未进入本提交，收尾时仍保持原暂存状态。

## 一句话结论

**通过**——display-api.ts 新增四封装（itemsRealtime/itemHistory/metricValues/metricCompleteness，只增不改既有签名），设备实时区改走 realtime 真值（绑定/公式/汇总同口径、来源如实标注）、指标看板接 metric/values 与 metric/completeness（同源不现算、available=false 保持空态+告警）；组件与纯函数零改动；`pnpm build:test && lint && typecheck && test` 四门全 exit 0（630 用例 = 610 基线 + 20 新增，零回归）；安全红线与空态纪律经源码级 + 测试级双重核实成立。

## 一、改动文件清单（6 个，全部落声明范围）

| 文件 | 性质 | 内容 |
| --- | --- | --- |
| `frontend/src/api/display/display-api.ts` | 修改 | 新增类型 `RealtimeItemVO`/`IotHistoryPoint`/`MetricCompletenessVO` + 四封装；既有封装签名零改动（diff 无删除行） |
| `frontend/src/views/display/device/device-realtime-zone.vue` | 修改 | load() 由「pointCurve 末点派生」改为「一次 itemsRealtime 批量真值」，来源标注改「实时值读口」；v=null/无该项/失败→空态 |
| `frontend/src/views/display/metrics/index.vue` | 修改 | loadAll() 接 metricValues（按 period 分组查询，valueRows 同源）+ metricCompleteness（available=true 填 map / false 保持空态+readGap 告警）；告警文案改「完整度接入中（读口待接入）」 |
| `frontend/tests/unit/display/display-read-ports.spec.ts` | 新增 | 10 用例：四封装 URL/参数/响应容错 + 安全红线代码扫描 |
| `frontend/tests/unit/display/device-realtime-zone.spec.ts` | 新增 | 5 用例：真值渲染（绑定+公式）、v=null 空态、无该项空态、读口失败降级、无 items 不调用 |
| `frontend/tests/unit/display/metrics-board-read-ports.spec.ts` | 新增 | 5 用例：同源映射、null 不冒充 0、available=true 填 map、available=false 告警空态、读口失败降级 |

**范围核对（`git diff --name-only 2d0f1a9..HEAD`，6 行全落声明范围）：**

```
frontend/src/api/display/display-api.ts
frontend/src/views/display/device/device-realtime-zone.vue
frontend/src/views/display/metrics/index.vue
frontend/tests/unit/display/device-realtime-zone.spec.ts
frontend/tests/unit/display/display-read-ports.spec.ts
frontend/tests/unit/display/metrics-board-read-ports.spec.ts
```

## 二、验收命令原始输出

命令：`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`

### build:test（exit 0）

```
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
transforming...
✓ 6004 modules transformed.
rendering chunks...
computing gzip size...
✓ built in 1m 26s
```

### lint（exit 0，0 errors）

```
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.
```

9 条 warning 全为既存无关文件的 prettier 格式提示（`tests/unit/api/survey/edit-lock-api.spec.ts`、`tests/unit/layout/setting-drawer-persist.spec.ts`、`tests/unit/lib/axios.spec.ts`、`tests/unit/survey/redpack-list-helper.spec.ts`、`tests/unit/views/survey/redpack-columns.spec.ts`）；本任务 6 个改动文件经 `eslint --fix` 后零 warning（`ZERO_WARNINGS_IN_MY_FILES`）。

### typecheck（exit 0）

```
$ vue-tsc --noEmit
```

（无输出即通过）

### test（exit 0）

```
✓ tests/unit/display/display-read-ports.spec.ts (10 tests)
✓ tests/unit/display/device-realtime-zone.spec.ts (5 tests)
✓ tests/unit/display/metrics-board-read-ports.spec.ts (5 tests)
...
 Test Files  69 passed (69)
      Tests  630 passed (630)
   Duration  59.12s
```

**610 基线 + 20 新增 = 630，既有 0058/0059/0060 display 单测零回归。**

说明：`menu-cache-frame.spec.ts`（无关既存文件）在「build 刚结束、机器仍高负载」时偶发单次 5000ms 超时；该文件隔离运行 `pnpm exec vitest run tests/unit/views/system/menu-cache-frame.spec.ts` → `1 passed (7 tests)`，且最终四门连跑 exit 全 0、630 全绿，判定为环境负载抖动，非本任务改动引入。

## 三、TDD 红转绿留痕（每修复附「为什么现在对了」）

**红（先写 20 个测试再实现）**：三份测试文件先落，`pnpm exec vitest run` → `Test Files 3 failed (3) / Tests 16 failed | 4 passed (20)`。典型红输出：

```
TypeError: displayApi.metricCompleteness is not a function
AssertionError: expected "spy" to be called with arguments: [['M_ENERGY','M_WATER'],...]  Number of calls: 0
TypeError: The URL must be of scheme file   ← 安全红线测试用 fileURLToPath(new URL(...)) 取源码路径
```

**逐修复 · 为什么现在对了：**

1. **四封装不存在红** → display-api.ts 末尾新增四方法 + 三个类型，全部走 `getRequest`；`asArray` 复用既有容错助手，realtime 的 data 非对象归一 `{}`。
   *为什么现在对了*：URL/参数逐字对齐 0087 controller（`/iot/instance/{id}/items/realtime` 的 `fullKeys` 逗号串、`/iot/item/history` 的 fullKey/start/end/interval/agg、`/iot/metric/values` 的 metricCodes/period/start/end、`/iot/metric/completeness` 的 metricCodes/period），10 个封装用例全绿。
2. **安全红线测试路径红（URL scheme）** → 改为 `resolve(process.cwd(), 'src/api/display/display-api.ts')`；代码区正则剔除 `/* */` 与 `//` 注释后扫描。
   *为什么现在对了*：命中源真实文件、只查可执行代码（注释里的「不持有」纪律说明不算命中），零 appKey/secret/HMAC/`/openapi/v1`/`fetch(`；原生 `getRequest` 是唯一通道。
3. **实时区仍走 pointCurve 红** → load() 改一次 `itemsRealtime(instanceId, fullKeys)`，按 `fullKey` 取 `{v,q,ts}`；来源文案改「实时值读口」；删除 `PointCurvePointVO` 导入。
   *为什么现在对了*：绑定类与公式类同一读口都出真值（`23.5℃`/`100℃`）、质量码 1 仍标红、不再出现「曲线末点」；测试断言 `pointCurve` 零调用。
4. **实时区测试零调用红** → 组件 `watch(..., { deep:false })` 无 `immediate`，父组件是显式 `realtimeRef.value?.load()`；测试改为 `await wrapper.vm.load()`（与生产同路径）。
   *为什么现在对了*：测试模拟真实调用路径，5 用例全绿，不靠改组件迁就测试。
5. **指标看板恒空态红** → loadAll() 按定义 period 分组查 metricValues → `valueResults.flat()` 填 valueRows；completeness 按 period 查，available=true 才填 map。
   *为什么现在对了*：`412.6`/`kWh`/`+8.3%`/`-2.1%` 全部原样来自读口（同源不现算），null 伴随值仍空态；有源时低于阈值标记可见。
6. **完整度 available=false 仍静默红** → `readGap = !anyAvailable`，catch 分支 `readGap = true`（fail-closed）。
   *为什么现在对了*：当前 0087 诚实空集下页面保持「完整度接入中（读口待接入）」告警、map 空、不静默判完整（断言不出现「完整度 100%」）。
7. **指标组件测试 ECharts canvas 崩溃红**（`Cannot set properties of null (setting 'dpr')`）→ 测试层 mock `metric-trend.vue` 为纯文本桩（happy-dom 无 2d context），生产组件零改动。
   *为什么现在对了*：测试只验证看板传值与渲染，生产 ECharts 组件 untouched，符合「组件零改动」。
8. **读口失败测试期望 console.error 红** → 生产用逐读口 `.catch(() => 空态)` 局部降级（不冒泡），测试断言改为「页面不炸 + 空态」而非强求 console.error。
   *为什么现在对了*：断言贴合真实降级语义（值读口失败不影响完整度读口、反之亦然），不强改生产代码迎合测试。

**绿**：`pnpm build:test && pnpm lint && pnpm typecheck && pnpm test` → 四门 exit 0、630/630 用例全绿。

## 四、安全红线证据（零 appKey/secret/HMAC）

```
$ grep -vE '^\s*(\*|//|/\*)' frontend/src/api/display/display-api.ts \
    | grep -niE 'appKey|appSecret|createHmac|secret|/openapi/v1|fetch\(|axios'
2:import { getRequest } from '/@/lib/axios';        ← 唯一命中：管理面登录态 axios
（其余零命中）

两处视图 + 三份测试代码区：ZERO_HMAC_OR_CREDENTIAL_REFS
单测 display-read-ports.spec.ts「安全红线」用例：读源码剔除注释后
  expect(code).not.toMatch(/appKey|appSecret|createHmac|HMAC|secret/i)   ✓
  expect(code).not.toMatch(/\/openapi\/v1/)                              ✓
  expect(code).not.toMatch(/\bfetch\s*\(/)                               ✓
```

四封装全部为管理面已登录态 `getRequest('/iot/...')`；不持 appKey/secret、不直连 `/openapi/v1/**`、不自造代理。页面告警文案中的 `appKey+HMAC` 等字样仅为「不持有/不直连」的说明，无任何可执行实现。

## 五、空态诚实证据（无源不造假）

| 场景 | 表现 | 证据 |
| --- | --- | --- |
| realtime 无该项 | `live=null` → `valueText='数据接入中'`，`qualityLabel='—'` | device-realtime-zone.spec.ts「无该项 → 空态」✓ |
| realtime `v=null` | `valueText='数据接入中'`，断言不含 `0℃`/`null` | 「v=null → 空态，不冒充 0」✓ |
| realtime 读口失败 | catch → rows 空 + console.error | 「读口整体失败 → 空态」✓ |
| metricValues `value/yoy/mom=null` | 原样透传到卡片，卡片走空态；断言不冒充 0 | metrics-board-read-ports.spec.ts「null 不冒充 0」✓ |
| 完整度 `available=false`（0087 当前） | map 空、`readGap=true` 告警、不出现「完整度 100%」 | 「available=false 保持空态+告警」✓ |
| 完整度行 `completeness=null` | displayApi 层 rows 原样保留；看板不写进 map | display-read-ports.spec.ts「null 完整度不伪造 0/1」✓ |
| 任一读口失败 | 局部 catch → 对应数据空态，不炸页面、不影响另一读口 | 「读口失败降级」✓ |

## 六、组件/纯函数零改动证据

```
$ git diff --name-only 2d0f1a9..HEAD  # 6 行，无下列任一文件
metric-card.vue / metric-trend.vue / metric-utils.ts / device-utils.ts /
display-factory-card.vue / trendDisplay / completenessDisplay / buildMetricTrend：UNCHANGED（逐一 git diff --quiet 核实）
backend/** / calc/** / ingest/** / resources/contract/**：working tree diff 0
display-api.ts：diff 无删除行（既有封装签名只增不改）
```

看板只替换 loadAll 取数处，`valueOf`/`completenessOf`/模板绑定 metric-card/metric-trend 均未改；实时区保留 `realtimeDisplayState`（同源不现算）与超限标红/离线灰显，仅换取数来源与来源文本。

## 七、未做项与原因

- TASK 收尾（置已完成 / 回填哈希 / LOG / 索引）：执行者契约禁止，未做。
- 完整度真值接入：0087 当前 `available=false`（calc 内网观测面未对管理面开放），前端接了也走空态——本任务只把取数逻辑接好，读口可用时自动出真值，属预期行为非遗漏。
- backend/calc/ingest/contract / 组件 / 纯函数改动：brief 明令不触碰，未做。
- push / git add -A：契约禁止，未做（提交为路径受限的 6 文件；工作区既存的他任务 backend 暂存变更未被纳入本提交）。

## 八、偏离计划之处

- 无实质偏离。两点说明：
  1. brief D3 的 `readGap` 告警文案要求「完整度接入中（读口待接入）」不变——原页面文案是更长的「管理面指标值读口缺口（空态登记，不造假）」，本任务按 brief 决策改为前者的短文案（描述同步更新为当前真实缺口：仅完整度无源，值已接通），测试断言同步。告警语义（available=false 保持 true、不静默判完整）未变。
  2. 验收链在「build 刚结束高负载」时出现一次无关既存测试 `menu-cache-frame.spec.ts` 的 5000ms 超时；隔离运行通过、最终四门连跑全绿，未为此改动测试或生产代码（遵循「测试失败先定位根因，禁止改期望」）。

## 九、遗留问题

- 完整度读口待 calc 提供管理面只读数据口后自动出真值（当前 honest empty，读口 available=true 时前端无需再改）。
- 四读口真实 TDengine/Redis 联调（fail-soft 空集如实）归 0068 环境恢复后联调，不在本任务验收范围。
- 趋势图去年同期对比依赖结果集中真实「同 ts 去年行」；若读口只返回本期行，Component 保持「不造假对比曲线」标注——既有纯函数口径，本任务不更改。

---

TASK 状态保持「进行中」，提交 `9459e34` 不 push；workitems 索引未动。

---

## 修订轮（I-1：指标卡片"本期值"取到去年旧值，BASE=9459e34）

> diff 评审退回一轮（1 Important）。本轮只修 I-1，其余五项评审通过项零回归。修订提交 `ba0a908c7c360cce8134d4a2d6170c9101e7e05e`（父 `ee767db`，即 9459e34 + 并行 TASK-0037 backend 提交；本轮仅 2 文件，不 push）。

### R1. I-1 修复（只在 `index.vue::valueOf` 内联完成）

| 位置 | 修改前 | 修改后 |
| --- | --- | --- |
| `frontend/src/views/display/metrics/index.vue:122` `valueOf()` | `valueRows.value.find((r) => r.metricCode === metric.metricCode) ?? null`（取 ts 升序首行 = 去年对比行） | 遍历同 `metricCode` 行，取 `ts` 最大者（本期行）：`if (latest == null \|\| row.ts > latest.ts) latest = row` |

- **不改组件/纯函数**：`metric-card.vue`/`metric-trend.vue`/`metric-utils.ts`（含 `buildMetricTrend`/`comparisonTs`/`oneYearBefore`）/`device-utils.ts` 全部零改动（逐一 `git diff --quiet` 核实）——修复完全在 `index.vue` 内联，未 import 也未改任何纯函数。
- **口径保持**：仍按 `metricCode` 定位（tenant 内唯一），不引入 period/instance 约束；value/yoy/mom 仍同源取自同一行，不现算。取最大 ts 与 `metric-utils` 的"去年对比行识别"（ts 恰晚 1 年者为对比行）语义一致，本期行 ts 一定更大。

### R2. 多行契约测试（TDD 红转绿）

- 新增用例 `同 metricCode 多行 ts 升序（去年对比行在前）→ 本期值取最大 ts 行，不取首行旧值（I-1）`，夹具：同码两行 `ts=T-1年,value=123.4,yoy=null`（去年对比行）与 `ts=T,value=412.6,yoy=0.083,mom=-0.021`（本期行），按后端 `ORDER BY ts ASC` 顺序注入。
- **红（旧版 `.find()`）**：`pnpm exec vitest run tests/unit/display/metrics-board-read-ports.spec.ts` → `Tests 1 failed | 5 passed (6)`：
  ```
  AssertionError: expected '指标看板卡片本期值…' to contain '412.6'
  ```
  旧版首行是去年对比行（123.4），本期值 412.6 未渲染 → 必红。
- **绿（修复后）**：同命令 → `Test Files 1 passed (1) / Tests 6 passed (6)`，断言 `412.6`/`+8.3%`/`-2.1%` 全出、不含 `123.4`、不含无伴随值的 `+0.0%`。

### R3. 为什么现在对了

- `valueRows` 由 366 天窗 + `valueResults.flat()` 装入，同码多行按 `ts ASC`；旧 `.find()` 命中窗口最早行（去年对比行），把一年前的值标为"本期值"——数值真实但周期错误。
- 改为"同码取 ts 最大行"后：本期行（ts 最大）整行喂给卡片，value/yoy/mom 一起同源切换（412.6 / +8.3% / -2.1%），不再出现旧值；遍历在无匹配时返回 null（卡片走空态），保留空态纪律。
- 选择"最大 ts"而非复用 `buildMetricTrend` 的 comparisonTs 集合：后者按"存在 ts+1 年同伴"识别，等价但更重；最大 ts 在"窗口只返本期行"或"含去年行"两种形态下都稳健，且无需触碰纯函数，符合 brief 裁决。

### R4. 验收原始输出（修订轮）

```
$ cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test
build:test exit=0   ✓ 6004 modules transformed. ✓ built in 2m 21s
lint exit=0         ✖ 9 problems (0 errors, 9 warnings)   ← 9 条均 unrelated 既存文件，本轮 2 文件零 warning
typecheck exit=0    $ vue-tsc --noEmit（无输出）
```
`pnpm test` 连跑（build 后高负载）复现一次无关既存 `menu-cache-frame.spec.ts` 5000ms 超时（`1 failed | 630 passed`）；隔离重跑该文件 `1 passed (7 tests)`，单独重跑全量（冷却 5s）：
```
$ pnpm test
 Test Files  69 passed (69)
      Tests  631 passed (631)     ← 630 主体基线 + 1 新增多行断言，零回归
  Duration  59s
 exit=0
```

### R5. 范围核对（修订轮）

- 修订提交 `ba0a908` 仅 2 文件：`frontend/src/views/display/metrics/index.vue`、`frontend/tests/unit/display/metrics-board-read-ports.spec.ts`。
- `git diff --name-only ee767db..ba0a908` = 上述 2 行，全落 brief 裁决"只改 index.vue + 相关测试"；组件/纯函数/backend/calc/ingest/contract 零触碰。
- 说明：主体提交 9459e34 之后、修订之前落了并行 TASK-0037 backend 提交 `ee767db`（17 文件 `backend/.../dataitem/**`+`application.yml`），不属本轮范围、本轮零触碰；首评报告 §六已记载该并行提交隔离事实。
- 安全红线与五项已通过判定沿用首评结论；本轮 diff 未引入任何新 URL/凭据/代理，未改空态与 readGap 判定。

TASK 状态保持「进行中」，修订提交 `ba0a908` 不 push；workitems 索引未动。
