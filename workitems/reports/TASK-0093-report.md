# TASK-0093 执行报告（实例树节点编辑入口 · 微改）

- BASE：`4de39ec796853437c2b547357654aafaeb3d7e64`
- 提交哈希：`5813fa5884a74b3236eef30ff73123d8ac76b867`
- 最终 HEAD：`5813fa5884a74b3236eef30ff73123d8ac76b867`
- 执行日期：2026-11-25
- 状态：代码已提交；TASK 状态未改（按契约由协调者验收收尾），未 push、未 `git add -A`

## 1 改动文件清单

- `frontend/src/views/iot/instance/index.vue`：a-tree 用 `#title` 插槽（选中方案，见 §4）在节点标签旁渲染编辑小图标按钮（`EditOutlined` + `v-privilege="'iot:instance:edit'"` + `@click.stop`）；新增 `editFromTree(treeNode)`：按 key 在树中回查原始 `IotInstanceTreeVO`（`raw` 字段），补 `shortCode`（从 instanceCode 末段推导，仅为通过弹窗必填校验，后端 update 白名单不含此字段）后调既有 `addInstance(record)` 打开编辑弹窗；骨架路径节点（后端数据权限裁剪只给 id/code/name/nodeType/status，缺 deptId/sort）按 id 调 `instanceApi.queryInstanceById` 拉详情补齐，失败上报 otSentry 且不开空弹窗；保留原 expand/select/filter 行为不变。加 hover 显隐编辑图标样式，不改树缩进。
- `frontend/tests/unit/views/iot/instance-tree-edit.spec.ts`（新增，8 例）：树节点编辑按钮在工厂/车间/设备各级渲染（EditOutlined）、点击工厂节点调 `showModal` 传完整 record（edit 模式）且不触发 select、骨架节点拉详情补齐、详情失败不开弹窗且上报、locked=1 透传、无权限按钮被真实 v-privilege 指令移除、过滤保留命中+祖先且编辑按钮随树保留、点击标签仍触发 onSelectTree。

## 2 验收命令原始输出

命令：`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`（按序逐项执行，退出码均 0）

```
$ pnpm build:test
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
./icon.svg referenced in .../vue3-json-viewer/dist/index.css didn't resolve at build time, it will remain unchanged to be resolved at runtime
(!) Some chunks are larger than 500 kB after minification. ...
✓ built in 1m 31s
BUILD_EXIT=0

$ pnpm lint
$ eslint .
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.
LINT_EXIT=0

$ pnpm typecheck
$ vue-tsc --noEmit
TYPECHECK_EXIT=0

$ pnpm test
 ✓ tests/unit/views/iot/instance-tree-edit.spec.ts (8 tests)
 Test Files  78 passed (78)
      Tests  666 passed (666)
   Duration  67.46s
TEST_EXIT=0
```

结论：0 错误 0 失败。666 = 基线 658 零回归 + 新增 8 例。lint 的 9 条 warning 全来自既有测试文件（edit-lock-api / setting-drawer-persist / axios / redpack-list-helper / redpack-columns）的 prettier 提示，本批改动文件 lint 干净（单独 `eslint src/views/iot/instance/index.vue tests/unit/views/iot/instance-tree-edit.spec.ts` 无输出）。完整原始日志：`/tmp/task0093-build.log`、`/tmp/task0093-lint.log`、`/tmp/task0093-typecheck.log`、`/tmp/task0093-test.log`（本机临时文件，不入仓）。

## 3 范围核对

命令：`git diff --name-only 4de39ec7..HEAD`

```
frontend/src/views/iot/instance/index.vue
frontend/tests/unit/views/iot/instance-tree-edit.spec.ts
```

仅 2 个文件，全部在 TASK 声明范围内（index.vue + instance*.spec.ts）；未触碰 backend / 其它前端页。工作区另有既存未跟踪目录 `.agents/`、`.tmp-t0044v/`（非本次改动，未触碰）。

## 4 方案选择与偏离说明

- **方案二选一结论**：采用 **a-tree `#title` 插槽**（非右键 a-dropdown）。理由：项目已有同款范式（`help-doc-catalog-tree.vue` title 插槽内嵌操作按钮），Ant Design Vue 4.2.5 的 title 插槽直接支持；图标默认 `opacity:0`、hover 节点显现，不改变树原有样式与缩进，无需引入右键菜单状态。测试证明 expand/select/filter 未受影响。
- **最小补充 1**：树 title 插槽只拿到加工后的 `TreeNode`（title/key/instanceCode/status），故在映射时为每节点保留 `raw`（完整 VO）供编辑弹窗回填，避免改树数据结构语义。
- **最小补充 2**：编辑弹窗 `rules.shortCode` 为必填（编辑态仅展示不可改），树 VO 无 shortCode，故在 `editFromTree` 从 `instanceCode` 末段推导补上；后端 `IotInstanceUpdateForm` 白名单不含 shortCode，不产生写路径影响。
- **最小补充 3**：后端 M-7 修订将受限用户的路径祖先收窄为骨架 VO（缺 deptId/sort），直接传弹窗会导致 deptId 必填校验失败/排序回填丢失；按 id 拉详情补齐（复用已存在的 `queryInstanceById`），失败不开窗。全权限用户（工厂级实例为顶级、常见场景）走 raw 分支，无额外请求。
- 未做浏览器端人工 UI 点击实测（执行者以组件单测覆盖入口行为，界面实测留协调者验收）；后端未改（PUT /iot/instance 已具备）。

## 5 遗留问题

- 编辑图标默认 hover 才显现，触屏 / 键盘 Tab 导航场景可见性弱于列表常显的「编辑」按钮；如需无障碍强化，可后续改为常显或加 focus-within 样式（本任务未要求）。
- 骨架节点编辑依赖详情接口（权限点 `iot:instance:query`）；仅有 edit 权限但无 query 权限的受限用户点击骨架（祖先路径）节点时会 403 → 不开窗并上报。全权限/自身可见节点不受影响；如需放宽，应由后端细化树 VO 字段口径，非本任务范围。
