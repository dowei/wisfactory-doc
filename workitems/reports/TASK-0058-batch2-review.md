# 评审报告：TASK-0058 批次二 diff 评审（代码评审）

- 评审口径：L1-execute §13「diff 评审」；fresh、no-subagents、只评审不修改（本报告除外）。
- 评审对象：提交 `8660ee0d`（BASE=`ba0a908`，区间单提交，14 文件）——工厂状态数聚合读口 + 总览/工厂卡片接真值。
- 方法：**回到源码独立核实**（不采信实施报告自述）+ 独立重跑前后端验收命令。
- 关联验收：FEAT-0019 / REQ-0029 验收 2「卡片运行/异常/离线数与状态判定结果一致」/ TC-0019 / ADR-0011。

## 结论

**通过（无 C/I）**。头号重点三项——①状态数一致性与 0050 同源、UNKNOWN 不冒充正常；②fail-soft 不造假（真 0 显示「0」、fail-soft 走空态）；③空态诚实（能耗/关键数据项/上看板标记无源不伪造、工厂页指标 0087 同源不现算）——均回到源码核实成立。14 文件全落声明范围，越界 0，区间 `ba0a908..HEAD` 干净单提交。独立重跑：backend 2613 用例 0F0E；frontend 四门 exit 0、649 用例全绿。发现 1 条 Minor（deferred，不阻塞验收）。

## 一、越界反查与范围

- `git rev-parse HEAD` = `8660ee0d1e394681c3e280cb4dd1c99de0b16860`；`git log ba0a908..HEAD` 仅 1 条提交；`git diff --name-only ba0a908..HEAD` 恰 14 文件。
- 14 文件分布：backend `iot/statusread/**` 6（VO/Service/ServiceImpl/Controller + 2 测试）；frontend display 8（display-api.ts、display-factory-card.vue、display-slot-card.vue、factory/index.vue + 4 spec）。全部落在 brief 声明范围。
- 零触碰反查：`frontend/src/views/display/device/**`、`metrics/**`、`workshop/index.vue`、backend 其他模块（calcread/displayread/openapi/dataitem/inspect/alarm）、calc/ingest/resources/contract、组件纯函数（metric-utils）均无 diff。工作区仅存在未跟踪 `.agents/`、`.tmp-t0044v/`（未入提交）。
- 结论：范围合规，越界 0。

## 二、头号重点判定

### 1. 状态数一致性（REQ-0029 验收 2）——通过

- **真同源、无逻辑复制**：`FactoryStatusSummaryServiceImpl.readStatusCode()`（`/d/.../FactoryStatusSummaryServiceImpl.java:99-108`）对每个设备调用 `StatusReadService.currentForClient(tenantId, deviceId)` 取 `statusCode`。回到权威源 `StatusReadServiceImpl.java:111-114, 129-167` 确认：`currentForClient` 与 0050 `GET /status/current`（`current()`→同核心 `current()`）走**同一状态读口**——读 Redis `iot:st:{tenant}:{instance}` 解析 statusCode + 字典/规则动态翻译。聚合层只读 statusCode 分类，**不读 Redis 原值、不复制状态判定、不硬编码状态名/色**。满足 brief「复用 0050 取 statusCode 的方法，不复制判定逻辑」。
- **分类精确、UNKNOWN 不冒充正常**：`summary()` switch（`:82-87`）`RUNNING→running / FAULT→fault / OFFLINE→offline / default→unknown`；空/空白/读异常统一归 "UNKNOWN"（`:102-103, 104-107`）。RUNNING/FAULT/OFFLINE/UNKNOWN 是本仓 0048 字典、0049 写侧、0050 读口、LOG-0060（"RUNNING→FAULT→LWT OFFLINE→UNKNOWN"）确立的规范码；自定义态/停机/custom 落 `unknown`，不进任何「正常」类。测试 `classifiesStatusCodesExactly`（ServiceImplTest:123-137）mock 7 台设备 `RUNNING×2/FAULT/OFFLINE/UNKNOWN/STOPPED/CUSTOM_STATE` 断言 `running=2,fault=1,offline=1,unknown=3,total=7`，与实现一致且精确。
- **工厂子树遍历正确**：`collectDeviceIds`（`:111-131`）以迭代栈从工厂节点出发，只收 `nodeType==4` 设备；组件（nodeType=5）与兄弟厂不在子树内自然排除。`findNode`+`isFactory`（`:133-155`）校验节点存在且为工厂，否则 fail-closed 403。测试 `traversesSubtreeOnly` 另用 `verify(never())` 断言兄弟厂设备 998、组件 1100 不被读取（ServiceImplTest:169-182）。
- **数据权限裁剪**：先 `requireTenant()`（TenantContext 无上下文 403），再 `requireDataPerm()`（`dataPermService.resolve(userId)`，factoryId ∉ 可见集即 403 通用文案，`:157-176`）；实例树本身由 `IotModelInstanceServiceImpl.selectInstanceTree()`（`:456-506`，实证按可见集 + 到根骨架裁剪，无权兄弟节点不返回）裁剪，聚合只在裁剪后树内遍历、不二次补节点。测试 `forbiddenWhenFactoryNotVisible/forbiddenWhenNoTenantContext/usesSessionTenantForEveryDevice/forbiddenWhenFactoryMissing` 覆盖。
- 判定：**通过，与 0050 同源，UNKNOWN 不冒充正常**。

### 2. fail-soft 不造假——通过

- **单设备失败按 unknown、不炸全厂**：`readStatusCode` 内层 try/catch（`:100-107`）把状态读异常吞为 "UNKNOWN"→`unknown++`；测试 `singleDeviceFailureCountsAsUnknown`（ServiceImplTest:186-197）断言一台 Redis down 时其余设备仍精确、`available=true`、`running=1/unknown=1/total=2`。
- **整厂异常全 0 + 接入中**：`summary()` 外层 catch（`:92-95`）捕获非 BusinessException → `FactoryStatusSummaryVO.unavailable()`（VO.java:91-96，四类与 total 恒 0、`available=false`、`message="状态数据接入中"`）；403 BusinessException 显式 rethrow 不吞成 fail-soft（`:90-91`）。测试 `wholeFactoryFailureIsFailSoft`（:201-213）逐字段断言。
- **真 0 显示「0」不是空态**：backend `available=true` 全 0 是真 0（VO 构造器 `:81-88` 置 `available=true`；空厂测试 `emptyFactoryIsRealZero`）；frontend 三段一致——API 包装 `data.available===true` 严格判定、缺数据才 fail-soft（display-api.ts:744-759）；卡片 `if (out.available !== true) return` 保留 null 空态（display-factory-card.vue:55-70）；槽位 `typeof value==='number' && isFinite` 才进真值模式，0 必然渲染「0」（display-slot-card.vue:23-28, 49-50）。测试 `realZeroIsZero`/`available=true 且全 0`/`value=0 显示「0」` 三端断言。
- 判定：**通过，无伪造状态数路径**；`available` 标志贯穿 backend→API→卡片→槽位，fail-soft 的 0 与真 0 被严格区分。

### 3. 空态诚实——通过

- **能耗构成**：工厂页 `<display-slot-card title="电 / 水 / 气 / 蒸汽" />` 不传 value（factory/index.vue:64-68）→ 保持骨架 +「实时数据接入中」；测试 `factory-page-metrics.spec.ts` 断言其 `value` 为空且显示接入中。无专用端点/无数据源，未伪造占比。
- **关键数据项 / 核心指标（卡片）**：`display-factory-card.vue:37-38` 两个 slot-card 均不传 value → 空态；测试断言 `value` 为 undefined 且含空态文案。无上看板标记数据源（mark 占位未落列），未伪造「哪些是关键」。
- **上看板标记（ADR-0011）**：mark 依赖 node.mark，后端未落列恒缺省=普通卡片；mark 放大/置顶渲染逻辑保留但造不出标记。测试 `mark 保留` 仅验证「有标记时的视觉分支」未伪造标记来源。
- **工厂页指标接 0087 metric/values 同源展示、不现算**：`loadMetrics()`（factory/index.vue:115-147）按工厂 `queryMetricList(factoryId)` 取定义、按 period 分组调 `metricValues`，`valueOf()` 取同码最大 ts 为本期行（:105-113），`metric-card` 原样展示 value/yoy/mom。回到 `MetricCard`（metric-card.vue:33-49）确认：无值行走「指标数据接入中」空态、`null` 不冒充 0；`trendDisplay`（metric-utils.ts:38-46）仅做 ×100 百分比格式化，不重算同比环比。后端 0087 `IotDisplayReadServiceImpl.metricValues`（:89-102）只做 windowStart→ts 字段映射、value/yoy/mom 同源透传。null 值测试（M_WATER value:null）断言显示空态而非 0。
- 判定：**通过，空态诚实**；能耗/关键数据项/上看板标记无源保持空态并登记，工厂页指标同源不现算、null 不冒充 0。

## 三、其它评审点

- **端点与权限**：`FactoryStatusSummaryController`（:32-46）`GET /api/v1/iot/status/factory-summary?factoryInstanceId=`、`@SaCheckPermission("iot:instance:query")`（与 0087 展示读口同码）、无 `@OperateLog`（读不审 0079 范式，与 status current 一致）；缺参 400；403 文案「无权访问该资源」不回显工厂标识（ServiceImpl:174-176 + 测试断言）。租户经 `TenantContext` 会话注入并逐个传给设备读口。合规。
- **测试有效性与零回归**：独立重跑（非采信报告）——
  - `mvn -pl ot-admin test`：`Tests run: 2613, Failures: 0, Errors: 0`，BUILD SUCCESS；surefire 报告确认 `FactoryStatusSummaryServiceImplTest` 10/0/0、`FactoryStatusSummaryControllerTest` 4/0/0（=2599 基线 +14）。
  - `pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`：BUILD_EXIT=0 / LINT_EXIT=0（9 warning 全在存量未改文件）/ TYPECHECK_EXIT=0 / TEST_EXIT=0；`Test Files 73 passed，Tests 649 passed`（=631 基线 +18）。lint 期间存量用例 8098 ECONNREFUSED 属同款噪音，不影响全绿。
  - TDD 红转绿：单提交 Squash 无法从历史独立复核「红」阶段，但新增测试覆盖完整、断言精确、且独立跑全绿，不影响结论。
- **D3（车间页设备状态色块）**：brief D3「若车间页已用实例树 status 字段展示则保持现状」。源码 `workshop/index.vue:41-43` 确实用 `dev.status` 展示「已禁用/状态待接入」，符合保持现状条件；本批未改车间页，合规。

## 四、发现清单

| 级别 | 出处 | 一句话说明 |
| --- | --- | --- |
| Minor（deferred，不阻塞） | `FactoryStatusSummaryServiceImpl.java:82-87` + `resources/sql/upgrade/20261122_iot_status_rule_domain.sql:135`（`iot_status_dict.status_code varchar(32)` 租户可配） | 分类用 RUNNING/FAULT/OFFLINE 三个字面量精确匹配，与 0050 及本仓规范码一致、满足本批硬验收；但状态字典是租户可配置的，若某租户新增语义为「故障/离线」的自定义码，会被诚实归入 `unknown` 而非 fault/offline——属「其余→unknown」既定口径（不造假），无需本批返工，登记为演进观察项。 |

除上述 Minor 外，无 C/I/Alpha/Beta 发现。

## 五、判定汇总

| 评审重点 | 判定 |
| --- | --- |
| ① 状态数一致性（与 0050 同源、UNKNOWN 不冒充正常） | 通过 |
| ② fail-soft 不造假（真 0/空态/fail-soft 三态分明） | 通过 |
| ③ 空态诚实（能耗/关键数据项/上看板标记 + 工厂页指标同源） | 通过 |
| ④ 端点与权限（GET/权限码/403/租户/读不审） | 通过 |
| ⑤ 测试有效性与零回归（独立重跑 2613 + 649 全绿） | 通过 |
| ⑥ 范围（14 文件、越界 0、区间干净） | 通过 |

**总评：通过**。无 C/I，按标准任务首评口径不退回、不触发复评；1 条 Minor 记 deferred。TASK 状态与 workitems 索引本次评审未改动。
