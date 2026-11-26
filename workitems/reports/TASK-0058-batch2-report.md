# TASK-0058 批次二实施报告（总览/工厂卡片接真数据——工厂状态数聚合 + 核心指标）

- 任务：TASK-0058 批次二（续做，TASK 保持「进行中」，不置已完成）
- 执行者：批次二执行者（no-subagents，一个执行者做完整、统一测一次）
- BASE：`ba0a908`（开工时 HEAD 恰为 BASE，批次一 9aed9a4 已在其历史内）
- 提交：`8660ee0d`（main，注明 TASK-0058 batch2；`git add` 只列本批 14 个改动文件，不含 `.agents/`、`.tmp-t0044v/`）
- 范围：backend statusread/** + frontend 展示端总览/工厂卡片/工厂页/api/单测；未碰 device/metrics 页面、backend 其他模块、calc/ingest/resources/contract、组件纯函数

## 一、改动文件清单（14 个，全落声明范围）

### backend（6 个，`backend/ot-admin/src/{main,test}/.../iot/statusread/**`）

| 文件 | 作用 |
| --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusread/FactoryStatusSummaryVO.java` | 聚合载荷 `{running,fault,offline,unknown,total,available,message}`；`available=false` 的 `unavailable()` 工厂全 0 + 「状态数据接入中」，不伪造真 0 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusread/FactoryStatusSummaryService.java` | 读口服务接口（summary） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusread/impl/FactoryStatusSummaryServiceImpl.java` | 核心实现：实例树取 factory 子树 nodeType=4 设备 → 复用 0050 `StatusReadService.currentForClient(会话tenant, deviceId)` 的 statusCode 分类计数；迭代式子树遍历；单设备失败 unknown；整厂异常 fail-soft；会话租户 + requireDataPerm 403 通用文案；读不审 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusread/controller/FactoryStatusSummaryController.java` | `GET /api/v1/iot/status/factory-summary?factoryInstanceId=`，`@SaCheckPermission("iot:instance:query")`、无 `@OperateLog`（读不审 0079 范式），参数空 400 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusread/impl/FactoryStatusSummaryServiceImplTest.java` | 10 用例：精确分类/真 0/空厂/子树遍历/单设备失败/整厂 fail-soft/403/无租户/tenant 注入/工厂缺失 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusread/controller/FactoryStatusSummaryControllerTest.java` | 4 用例：参数透传/fail-soft 载荷/403 直达/缺参 400 |

### frontend（8 个）

| 文件 | 作用 |
| --- | --- |
| `frontend/src/api/display/display-api.ts` | 只增不改既有签名：新增 `FactoryStatusSummaryVO` 类型 + `displayApi.factoryStatusSummary(factoryInstanceId)`（URL/参数逐字对齐 D1；data 缺失/非对象 → fail-soft 全 0 + available=false）；安全红线沿用 |
| `frontend/src/views/display/components/display-slot-card.vue` | 新增可选 `value?: number \| null`：仅 `typeof number && finite` 进真值模式（0 显示「0」）；缺省/null 保持骨架+「实时数据接入中」空态 |
| `frontend/src/views/display/components/display-factory-card.vue` | 挂载/节点变更即按 `node.id` 调 factory-summary 填运行/异常/离线三槽位；`available!==true`/读口失败走空态；关键数据项/核心指标仍空态；mark 放大保留 |
| `frontend/src/views/display/factory/index.vue` | 工厂指标看板接 0087 `/iot/metric/values`（按工厂取定义 + 值行，最大 ts 为本期行，复用 metric-card 同源展示 value/yoy/mom）；能耗构成/关键数据项保持诚实空态 |
| `frontend/tests/unit/display/factory-summary-read-port.spec.ts` | 5 用例：URL/参数、真 0、fail-soft、响应容错、安全红线 |
| `frontend/tests/unit/display/factory-card-summary.spec.ts` | 5 用例：0 真值/12 真值/缺省空态/null 空态/mark |
| `frontend/tests/unit/display/factory-card-status-values.spec.ts` | 5 用例：三槽位真值、全 0 真 0、fail-soft 空态、读口失败、mark 保留 |
| `frontend/tests/unit/display/factory-page-metrics.spec.ts` | 3 用例：工厂级指标同源、能耗空态、失败空态 |

> 范围核对（验收命令 3）：`git diff --name-only ba0a908..HEAD`（提交后）+ 工作区 `git status` 全落上表 14 文件；`frontend/src/views/display/device/**`、`frontend/src/views/display/metrics/**`、`frontend/src/views/display/workshop/index.vue` 零改动（零 diff 反查通过）；backend 仅 statusread/**，calcread/displayread/openapi/dataitem/inspect/alarm 等零触碰；calc/ingest/resources/contract 零触碰。

## 二、验收命令原始输出

### 1）`cd backend && mvn -pl ot-admin test`

```
[INFO] Running net.otlab.admin.module.iot.statusread.controller.FactoryStatusSummaryControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.079 s -- in ...FactoryStatusSummaryControllerTest
[INFO] Running net.otlab.admin.module.iot.statusread.impl.FactoryStatusSummaryServiceImplTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in ...FactoryStatusSummaryServiceImplTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 2613, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] BUILD SUCCESS
[INFO] Total time:  01:56 min
```
0 失败 0 错误；含 factory-summary 聚合契约 14 用例 + 既有 2599 基线零回归（2599 + 14 = 2613）。

### 2）`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`

```
===== pnpm build:test =====
✓ built in 1m 3s
BUILD_EXIT=0
===== pnpm lint =====
✖ 9 problems (0 errors, 9 warnings)
LINT_EXIT=0
===== pnpm typecheck =====
TYPECHECK_EXIT=0
===== pnpm test =====
✓ tests/unit/display/factory-summary-read-port.spec.ts (5 tests)
✓ tests/unit/display/factory-card-summary.spec.ts (5 tests)
✓ tests/unit/display/factory-card-status-values.spec.ts (5 tests)
✓ tests/unit/display/factory-page-metrics.spec.ts (3 tests)
 Test Files  73 passed (73)
      Tests  649 passed (649)
TEST_EXIT=0
```
四门 0 error；lint 9 warning 全部位于未改动的存量测试文件（edit-lock-api/setting-drawer-persist/axios/redpack-*），本批新增与改动文件经 `eslint --fix` 后独立复检 0 error 0 warning。649 = 631 基线 + 本批 18 新用例，既有 display 单测零回归。`pnpm test` 期间存量用例向 8098 发请求产生 ECONNREFUSED stderr（与批次一同款存量噪音），不影响 73 文件 649 用例全绿。

### 3）范围核对

见一末尾；device/metrics/workshop 页面零 diff 反查通过。

## 三、TDD 红转绿留痕（每修复附「为什么现在对了」）

### backend

- **红**：仅新增 2 个测试文件时跑 `mvn -pl ot-admin -Dtest='FactoryStatusSummary*Test' test` → testCompile 5 errors（找不到 `FactoryStatusSummaryVO/Service/ServiceImpl/Controller` 符号），BUILD FAILURE。
- **绿**：补 4 个生产类后重跑 → `Tests run: 14, Failures: 0, Errors: 0`，BUILD SUCCESS；全量 2613 全绿。
  - 为什么现在对了：分类计数只调用 `StatusReadService.currentForClient(tenantId, deviceId)` 取 statusCode 再归类（RUNNING/FAULT/OFFLINE/default→unknown），**没有复制状态判定逻辑、不读 Redis 原值**，与 0050 同源；子树遍历用迭代栈只对 nodeType=4 计数（排除组件/兄弟厂）；单设备异常在 `readStatusCode` 内 catch→UNKNOWN；整厂异常在 `summary` 外层 catch→`unavailable()`（BusinessException 403 不被吞成 fail-soft）；tenant 从 `TenantContext.getTenantId()` 会话注入并逐个传给 status 读口；`requireDataPerm` 用 `dataPermService.resolve(userId)`，不在可见集即 403 通用文案「无权访问该资源」（不回显工厂 id）。

### frontend

- **红**：先写 4 个 spec（factory-summary-read-port / factory-card-summary / factory-card-status-values / factory-page-metrics）→ `vitest run` 得 `3 failed files / 10 failed tests`：`displayApi.factoryStatusSummary is not a function`、slot-card 无 `.display-slot-card__value`、工厂页未调 `queryMetricList`。
- **绿阶段 1**：补 `displayApi.factoryStatusSummary` + slot-card `value` 真值模式 + 工厂卡片接数 → 15/15 通过。修正 3 条测试期望（真值卡整卡仍含关键数据项空态属**要求行为**；fail-soft 传 null 是合法空态信号，null/undefined 均可）后——为什么现在对了：`typeof value==='number' && Number.isFinite` 才进真值模式，`0` 必然显示「0」，`null/undefined/NaN` 保持「实时数据接入中」；卡片 `available!==true` 直接 return 保持 null，不把 fail-soft 全 0 渲染成真 0。
- **绿阶段 2**：工厂页接 0087 metric/values + 复用 metric-card → 首轮 2 failed（未接指标）；补齐后在 metricValues 各口 `.catch()` 内补 `console.error` 使失败可观测 → 3/3 通过。为什么现在对了：值行按 metricCode 取最大 ts 为本期行，value/yoy/mom 原样透传 metric-card（不现算），null 行显示「指标数据接入中」不冒充 0；读口失败逐口 fail-soft 空数组、外层 try/catch 记日志、整页不炸。

## 四、状态数一致性证据（REQ-0029 验收 2）

1. **同源**：`FactoryStatusSummaryServiceImpl.readStatusCode()` 调用 `StatusReadService.currentForClient(tenantId, deviceId)` —— 与 `GET /api/v1/iot/status/current`（0050）为同一服务同一 statusCode 取法（同一 `current()` 核心读 Redis `iot:st:{tenant}:{code}` + 动态翻译），聚合层只读 statusCode 分类，**零状态判定复制**。
2. **分类精确**（测试 `classifiesStatusCodesExactly`）：mock 7 台设备 statusCode = `RUNNING×2 / FAULT×1 / OFFLINE×1 / UNKNOWN / STOPPED / CUSTOM_STATE` → `running=2, fault=1, offline=1, unknown=3, total=7`，断言 UNKNOWN/停机/自定义一律 unknown，不并入任何「正常」类。
3. **设备口径**：`collectDeviceIds` 只收 `nodeType=4`（跨车间/产线多层），组件 nodeType=5 与兄弟厂设备 `verify(never())` 断言不被读取；空厂 `available=true` 全 0 是真 0。
4. **失败不造假**：单设备读异常按 unknown 计（全厂仍精确）；整厂异常返回 `available=false` + 全 0 + 「状态数据接入中」，前端据此走空态，**不把 fail-soft 的 0 冒充真 0**；`available=true` 时 `0` 由 slot-card 渲染为「0」。
5. **权限/租户**：无权工厂 403 通用文案（不回显 id）、无租户上下文 403、tenant 会话注入逐个传给设备读口（verify 断言），读不审（controller 无 `@OperateLog`）。

## 五、诚实空态与未落地登记

| 项 | 处理 | 原因 / 登记 |
| --- | --- | --- |
| 能耗构成（按车间占比） | 工厂页 `<display-slot-card title="电 / 水 / 气 / 蒸汽" />` 保持空态；测试断言 value 为空且显示接入中 | 无专用端点/无数据源（后端 mark 占位未落列）、非硬验收——不伪造占比（REQ-0029 业务能力有描述，本批登记缺口） |
| 关键数据项（卡片/工厂页） | 保持空态（无 value 传入） | 无上看板标记数据源（后端 mark 占位未落列，ADR-0011/ADR-0014 登记），不伪造「哪些是关键」 |
| 上看板标记 node.mark | 卡片放大/置顶渲染逻辑保留；接口未交付前恒缺省=普通卡片 | ADR-0011 定版但后端 mark 未落列，本批不造标记；批次一 M-1（放大仅视觉强调无 CSS 尺寸）因 mark 恒缺省、接口未交付，本批不额外返工（沿用 deferred） |
| M-2 展示端入口形态 | 不改动 | 协调者已裁决「批次一预期，归批次二或独立入口，不返工」；本批按要求不碰路由 |
| D3 车间页设备状态色块 | 保持现状不改 | 批次一车间页已用实例树 `dev.status` 字段展示（见源码），符合 brief D3「已用实例树 status 则保持现状」；且车间页不在本批改动范围 |
| 联调冒烟 | 缺席登记 | 与批次一同款：8098 无监听，本批未启动后端；验收以单测契约为准 |

一句话结论：**通过**——backend `mvn -pl ot-admin test` 2613 用例 0 失败 0 错误（含 factory-summary 14 条精确契约），frontend 四门 exit 0（649 用例 = 631 基线 + 18 新，0 error），14 文件全落声明范围、device/metrics 页面与冻结契约零触碰；卡片运行/异常/离线三槽位已接与 0050 同源的真值（真 0 显示「0」、fail-soft 走空态），能耗构成/关键数据项/上看板标记因无数据源保持诚实空态并登记，TASK 保持「进行中」。

---

提交哈希：`8660ee0d1e394681c3e280cb4dd1c99de0b16860`（短 `8660ee0d`，main）
改动文件数：14（backend 6 + frontend 8）
