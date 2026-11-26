# TASK-0092 执行报告（BUG-0009 微改修复）

- BASE：a09e258699bd62ed236c48cfb4447b1442b5cdab
- 提交哈希：
  - `bb013a9c92baf79f2ef2ad905e11fc29ac90cc2c`（主体实现 + 新测试）
  - `4de39ec796853437c2b547357654aafaeb3d7e64`（既有 BUG-0008 测试补 point-api mock，消除新增网络请求噪声）
- 最终 HEAD：`4de39ec796853437c2b547357654aafaeb3d7e64`
- 执行时间：2026-11-25
- 状态：代码已提交；TASK 状态未改（按契约由协调者后续验收收尾）

## 1 改动文件清单

- `frontend/src/api/iot/item-api.ts`：新增 `bindPoint(instanceId, itemId, pointId)` → `PUT /iot/item/bind`，三字段对齐后端 `IotItemBindForm`（绑定/换绑共用 `iot:item:edit`）。
- `frontend/src/api/iot/point-api.ts`：新增 `queryList(params)` → `GET /iot/point/list`，透传 `pageNum/pageSize/status/boundInstanceId` 等过滤。
- `frontend/src/views/iot/instance/components/item-point-bind-modal.vue`（新增）：`a-modal` + `a-table` 单选点位弹窗；拉 `status=1` 启用点位，展示 pointCode/pointName/valueType；选中后调 `bindPoint`，成功 message + `reloadList`/`bindSuccess` 事件 + 关窗；失败 `message.error` 且不关窗；`defineExpose({ showModal })` 供父 ref 调用（BUG-0008 教训）。
- `frontend/src/views/iot/instance/components/instance-item-drawer.vue`：每行操作列加「绑定点位」按钮（`v-privilege="'iot:item:edit'"`），点击带当前实例/项打开弹窗；新增「绑定点位」列，优先显示行上 pointCode，其次用点位清单映射 pointId→pointCode，映射失败退化显示 `点位 <id>`；绑定成功后刷新清单并即时补上点位编码映射。
- `frontend/tests/unit/api/iot/point-bind-api.spec.ts`（新增）：API 参数单测（2 例）。
- `frontend/tests/unit/views/iot/point-bind-modal.spec.ts`（新增）：弹窗单测（4 例）——status=1 列表、选行→bindPoint 三元参数正确→成功提示/事件/关窗、未选不提交、失败提示不关窗、父 ref defineExpose。
- `frontend/tests/unit/views/iot/point-bind-drawer.spec.ts`（新增）：抽屉集成单测（2 例）——真实 `privilegeDirective` 验证无 `iot:item:edit` 时按钮被移除；点击→拉点位→选点→bindPoint(1001,2002,301)→刷新清单→行内显示 P-001。
- `frontend/tests/unit/views/iot/instance-item-drawer-ref.spec.ts`：既有 BUG-0008 回归测试补 point-api mock（因抽屉新增加载点位映射，避免引发真实 HTTP 请求噪声；断言不变）。

## 2 验收原始输出

验收命令：`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`

```
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
...（省略产物清单与既有 esbuild 警告）
✓ built in 1m 15s
BUILD_OK
$ eslint .
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.
LINT_OK
$ vue-tsc --noEmit
TYPECHECK_OK
...
 Test Files  77 passed (77)
      Tests  658 passed (658)
   Duration  55.61s
TEST_OK
```

说明：lint 9 条 warning 均来自既有测试文件的 prettier 提示（0 errors），本批新增/修改文件无 lint 问题；typecheck 无输出即通过；658 = 基线 650 零回归 + 本批新增 8 例。build/test 的完整原始日志分别见 `/tmp/task0092-build2.log`、`/tmp/task0092-lint2.log`、`/tmp/task0092-typecheck2.log`、`/tmp/task0092-test2.log`（本机临时文件，不入仓）。

## 3 范围核对

命令：`git diff --name-only a09e2586..HEAD`

```
frontend/src/api/iot/item-api.ts
frontend/src/api/iot/point-api.ts
frontend/src/views/iot/instance/components/instance-item-drawer.vue
frontend/src/views/iot/instance/components/item-point-bind-modal.vue
frontend/tests/unit/api/iot/point-bind-api.spec.ts
frontend/tests/unit/views/iot/instance-item-drawer-ref.spec.ts
frontend/tests/unit/views/iot/point-bind-drawer.spec.ts
frontend/tests/unit/views/iot/point-bind-modal.spec.ts
```

共 8 个文件，全部在 frontend 范围；未 `git add -A`、未 push、未置 TASK 已完成。工作区另有既存未跟踪目录 `.agents/`、`.tmp-t0044v/`（非本次改动，未触碰）。

## 4 未做项与偏离计划

- 未做：浏览器端手动 UI 回归（本执行者只保证组件/API 单测覆盖，界面实测由协调者验收）；后端未改（接口已具备，本任务纯前端）。
- 偏离/补充：
  1. 既有 BUG-0008 回归测试补了 point-api mock（计划未列，属保持既有测试隔离性的必要补充，不改变该测试断言）。
  2. 点位列表 API 封装到 `point-api.ts`（技术方案允许的可选项）；抽屉展示已绑点位采用「行上 pointCode 优先 + 点位清单映射 pointId」，映射请求 fail-soft，不阻塞清单。
  3. 提交拆为两个：主体 + 测试 hygiene 补充，最终 HEAD 为第二个提交。

## 5 遗留问题

- 点位清单按 `pageSize=500` 全量拉取用于编码映射/弹窗选择；点位规模超 500 时列表只显示前 500 条（后端支持分页，但弹窗当前未实现分页/搜索）。若后续出现超千点位场景，可扩展弹窗分页/关键字查询。
- 抽屉 `showDrawer` 仍会额外拉一次点位清单用于点Id→编码映射；若映射接口失败，行内退化为 `点位 <id>` 展示，不影响绑定功能。
