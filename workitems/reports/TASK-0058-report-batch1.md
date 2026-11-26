# TASK-0058 批次一实施报告（展示端工程与模型树 + 总览/工厂/车间页骨架 · 前端道）

- 任务：TASK-0058（状态保持「进行中」，本批为批次一，卡片数据接入留批次二）
- 执行者：batch1 前端道执行者（no-subagents）
- BASE：`5758d5f`（开工时 main HEAD）
- 提交：`9aed9a4`（main，注明 TASK-0058 batch1；`git add` 只含 frontend/）
- 范围：只改 `frontend/**`，未碰 backend/**、ingest/**、calc/**、workitems 库（本报告除外）

## 一、改动文件清单（13 个，全在 frontend/）

| 文件 | 作用 |
| --- | --- |
| `frontend/src/api/display/display-api.ts` | 展示端数据服务：只封装已交付的 `GET /iot/instance/tree`；定义 `DisplayTreeNode`（含 ADR-0011 可选 `mark` 字段）；纯函数 `childrenOf`/`findNode`/`descendantsOfType` 支撑 lazy load 与按类型取子节点。不写未交付端点占位调用 |
| `frontend/src/router/display.ts` | 独立 `/display/**` 静态路由段：总览 / 工厂 / 车间三页，需登录、hideInMenu、与管理端 `/iot/**` 不混用 |
| `frontend/src/router/index.ts` | 接线：`import { displayRouters }` + `buildRouteList()` 内 `push(...displayRouters)`（与管理端 iotUpgradeListRouter 同范式，挂 OtLayout 下） |
| `frontend/src/views/display/overview/index.vue` | 总览页：根级工厂卡片网格（真数据名+下钻）；运行/异常/离线、关键数据项、核心指标均为空态槽位；mark 放大+置顶；点卡片→工厂页（第 1 次点击） |
| `frontend/src/views/display/factory/index.vue` | 工厂页：车间列表（实例树二级真数据，nodeType=2）+ 指标看板/能耗构成空态；点车间→车间页（第 2 次点击） |
| `frontend/src/views/display/workshop/index.vue` | 车间/产线页：设备列表（跨层级取 nodeType=4 真数据）+ 状态色块/实时值空态；不写设备详情入口死链 |
| `frontend/src/views/display/components/display-factory-card.vue` | 工厂卡片：真名/编码/禁用态，内部数据槽位空态，mark 驱动放大 |
| `frontend/src/views/display/components/display-slot-card.vue` | 通用数据槽位：骨架屏 +「实时数据接入中」空态（绝不造假数字），mark 放大/置顶 Class |
| `frontend/src/views/display/composables/use-display-tree.ts` | 三页共用模型树加载（OtLoading 包裹，一次性拉取，error/empty 态）；后端已裁剪权限，前端不二次过滤 |
| `frontend/src/views/display/display.less` | 展示端页面公共样式 |
| `frontend/tests/unit/display/display-tree.spec.ts` | 5 用例：lazy load 按 sort、无权节点查不到、跨层取车间/设备、mark 透传 |
| `frontend/tests/unit/display/display-routes.spec.ts` | 3 用例：三路由路径命名、下钻≤3且无设备详情死链、需登录+hideInMenu+不混用 /iot/ |
| `frontend/tests/unit/display/display-slot-card.spec.ts` | 2 用例：空态不造假数字、mark 放大/置顶渲染 |

> 验证命令 3（范围核对）：`git diff --name-only 5758d5f..HEAD` 输出即上表 13 个文件，全部落在 `frontend/**`（src/** 与路由配置；含 `tests/unit/display/**` 前端单测）。并行 TASK-0054 的 backend/openapi 改动仍在工作区未提交，未被本任务触碰或纳入提交。

## 二、验收命令原始输出

### 1）pnpm 三门

```
===== pnpm build:test =====
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported ...
vite v5.2.12 building for test...
...（既有 survey panel.vue 常量赋值警告、vue3-json-viewer icon.svg 提示、chunk>500kB 警告均为存量）
✓ built in 2m 21s
BUILD_EXIT=0
```

```
===== pnpm lint =====
$ eslint .
...（9 条 prettier/prettier warning 全部位于未改动的存量测试文件：
     edit-lock-api.spec.ts / setting-drawer-persist.spec.ts / axios.spec.ts /
     redpack-list-helper.spec.ts / redpack-columns.spec.ts）
✖ 9 problems (0 errors, 9 warnings)
LINT_EXIT=0
```
（本任务新增文件经 `npx eslint <新文件> --fix` 后复检 0 error 0 warning，仅 .less 因无匹配配置为 ignored）

```
===== pnpm typecheck =====
$ vue-tsc --noEmit
TYPECHECK_EXIT=0
```

补充全量单测（brief 测试口径要求 mock 断言）：

```
===== pnpm test =====
$ vitest run
✓ tests/unit/display/display-tree.spec.ts (5 tests)
✓ tests/unit/display/display-routes.spec.ts (3 tests)
✓ tests/unit/display/display-slot-card.spec.ts (2 tests)
 Test Files  62 passed (62)
      Tests  583 passed (583)
TEST_EXIT=0
```

三门 + 单测全部 0 error / 全绿。

### 2）联调冒烟（缺席登记，待补）

```
$ curl -s -o /dev/null -w "HTTP_CODE=%{http_code}\n" --max-time 5 http://127.0.0.1:8098/
HTTP_CODE=000
CURL_EXIT=7
$ netstat -ano | grep ":8098"
NO_8098_LISTENER
```

结论：8098 无监听（协调者未启动后端）。按 brief「8098 探测，忙则跳过登记待补」，**本批联调冒烟缺席**；且 backend 道 TASK-0054 在途（openapi 模块改动仍在工作区），执行者不擅自启动 backend、不跨道触碰。待补项：后端就绪后执行「登录→展示端模型树真数据→三级页面下钻断言→空态卡片→全停」。

> 附带观察：`pnpm test` 全程有存量单元测试向 8098 发起请求产生 ECONNREFUSED stderr（user-module 等旧用例），但 62 文件 583 用例全部通过，与本任务无关。

## 三、未做项与原因 / 批次二登记清单

| 未做项 | 原因 | 批次二承接 |
| --- | --- | --- |
| 总览工厂卡片「运行/异常/离线数」真值 | 依赖 TASK-0050 状态展示读口（未交付）；硬编码假数字违反数据纪律 | 0050 交付后接数据，布局/槽位不动 |
| 关键数据项实时值、当日核心指标与同比环比真值 | 依赖 TASK-0045 指标聚合（未交付） | 0045 交付后接数据 |
| 工厂页指标看板、能耗构成真值 | 同上（0045） | 0045 交付后接数据 |
| 车间/产线页设备状态色块、实时值真值 | 依赖 0050 状态读口 + 0045 指标聚合 | 0050/0045 交付后接数据 |
| 设备详情页入口与页面 | 0059 拆出，本批不写死链 | TASK-0059 |
| 指标看板页 | 0060 拆出，本批不写死链 | TASK-0060 |
| 上看板标记真值 | ADR-0011 标记字段接口未交付；本批仅在数据结构预留可选 `mark` 并渲染放大/置顶逻辑 | 接口交付后填值，渲染逻辑不动 |

数据纪律声明：本批所有未接数据的槽位均为骨架屏 +「实时数据接入中」空态，**无一处硬编码假数字、无未交付页死链**。

## 四、偏离计划之处

1. **联调冒烟缺席**：brief 明示「8098 探测，忙则跳过登记待补」——实测无监听，已登记待补（见二.2）。属计划内允许的缺席，非执行偏离。
2. **单测文件位置**：brief 改动范围写「frontend/src/** 与路由配置」，本批在 `frontend/tests/unit/display/**` 新增 3 个单测文件以覆盖 brief「测试口径」（mock 树响应、下钻点击数、空态、mark 渲染）。仍全部位于 frontend/ 内、随 frontend/ 提交，未越出前端道；在此显式标注。
3. **路由挂载方式**：展示端三页未走 sys_menu 动态菜单注册（brief 要求「独立路由、与管理端并存不混用」），采用与 `iotUpgradeListRouter` 同构的静态路由段、挂 OtLayout 下、hideInMenu。这样无需新增菜单种子即可直达 `/display/**`，且不污染管理端侧栏；若协调者要求展示端走独立菜单/独立布局外壳，需另行裁决（不影响本批功能与验收）。
4. **无实质性技术偏离**：未改 backend、未置 TASK 已完成、未造假数据、未写死链、只 add frontend/。

## 五、遗留问题

1. 联调冒烟待补：需后端 8098 就绪 + TASK-0054 落定后，由协调者安排端到端冒烟（登录→模型树真数据→三级下钻→空态→全停）。
2. 批次二（0050/0045 到位）只需在 `useDisplayTree` / 各页槽位处替换数据源，卡片布局与下钻交互无需改动；上看板标记 `mark` 字段来源待接口交付后接入。
3. 并行提醒：backend/openapi 的 0054 改动与 `.agents/` 仍在工作区未提交，属其他任务/环境目录，本任务未触碰；协调者合并时留意文件集不相交。

---

一句话结论：**通过**——pnpm build:test/lint/typecheck 三门 exit 0（lint 0 error），pnpm test 62 文件 583 用例全绿（含本批 10 条新断言）；`git diff --name-only 5758d5f..HEAD` 13 个文件全落 frontend/**，模型树真接入、三页骨架、空态无假数据、下钻≤3、无死链均成立；联调冒烟因 8098 无监听按 brief 登记缺席待补，不影响本批骨架验收。
