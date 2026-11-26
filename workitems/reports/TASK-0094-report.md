# TASK-0094 执行报告

- 任务：实例数据项公式/汇总自定义入口（前端补齐）
- BASE：5813fa58
- 提交：4e4c1461cdff7d9e8fbed02a474f17b9528fcb50（`feat(TASK-0094): 实例数据项公式/汇总自定义入口（前端补齐）`）
- 状态：实现完成，四门验收全绿；按契约不置 TASK 已完成

## 1 改动文件清单（7 个，全 frontend）

| 文件 | 说明 |
| --- | --- |
| `frontend/src/api/iot/item-api.ts` | 新增 `saveFormula(instanceId,itemId,formula)` → PUT `/iot/item/formula`；`saveRollup(instanceId,itemId,rollupMethod)` → PUT `/iot/item/rollup`，字段对齐后端 DTO |
| `frontend/src/components/iot/iot-consts.ts` | 新增 `IOT_ROLLUP_ALLOWED_METHODS = ['SUM','AVG']`，镜像后端 `IotRollupRules.ALLOWED_METHODS` 单一来源 |
| `frontend/src/views/iot/instance/components/item-formula-rollup-modal.vue`（新） | 一组件两模式（formula/rollup）；formula 复用 `IotFormulaEditor` 并按当前实例清单构造 SELF 引用分组；rollup SUM/AVG 单选；空/语法错误前置拦截；保存失败 `handleIotConflictError`；`defineExpose({showModal})` |
| `frontend/src/views/iot/instance/components/instance-item-drawer.vue` | 按 sourceType=2/3 加「编辑公式」「编辑汇总」（`v-privilege iot:item:edit`），保存后 `reloadList` 刷新；locked=1 时按钮禁用 + 包裹层点击提示 |
| `frontend/tests/unit/api/iot/point-bind-api.spec.ts` | 扩展 saveFormula/saveRollup 参数断言（沿用 0092 测试文件） |
| `frontend/tests/unit/views/iot/item-formula-rollup-modal.spec.ts`（新） | 弹窗 5 例：defineExpose/回填与 SELF 分组、空与语法错误拦截、公式保存、汇总保存、失败提示 |
| `frontend/tests/unit/views/iot/instance-item-value-editor.spec.ts`（新） | 抽屉 4 例：按钮按 sourceType 渲染、公式编辑保存刷新、汇总编辑保存刷新、locked 禁用与提示 |

范围核对：`git diff --name-only 5813fa58..HEAD` 仅上述 7 个前端文件；无 backend/calc/ingest/contract 改动。

## 2 验收命令原始输出

```
$ cd frontend && pnpm build:test
✓ built in 1m 45s
build exit=0

$ pnpm lint
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.
lint exit=0
（9 条 warning 均为既有文件 prettier 提示：edit-lock-api.spec.ts / setting-drawer-persist.spec.ts /
 axios.spec.ts / redpack-list-helper.spec.ts / redpack-columns.spec.ts；本次改动文件 0 warning）

$ pnpm typecheck
$ vue-tsc --noEmit
typecheck exit=0

$ pnpm test
Test Files  80 passed (80)
Tests       677 passed (677)
Duration    82.93s
test exit=0
```

基线 666 + 本次新增 11 例 = 677，零失败；0092 的 `point-bind-api.spec.ts`、`point-bind-modal.spec.ts`、`point-bind-drawer.spec.ts` 全部通过（point-bind-api 同文件扩展）。

## 3 未做项与原因

- 取值方式「切换」入口（绑定项 ↔ 公式项 ↔ 汇总项互转）未实现：按任务实施步骤 4 说明，范围过大（需新增切换确认 UI 与互斥语义提示），留演进位。后端接口已支持切换时清退互斥字段。
- 既有 5 个测试文件的 9 条 prettier warning 未修：属既有代码，超出本任务范围，且 lint 0 error 不阻塞验收。

## 4 偏离计划之处

- 涉及文件比任务书多两处：`iot-consts.ts`（新增 rollup 方法枚举，保持前后端枚举单一来源）与 `point-bind-api.spec.ts`（沿用 0092 测试文件追加 API 断言，避免同主题重复文件）；均在 frontend 范围内。
- 弹窗文件名采用 brief 建议的 `item-formula-rollup-modal.vue`（任务详情正文写的是 `item-formula-edit-modal.vue`），与 brief 第 2 条一致。
- locked 提示实现：按钮 `disabled`，其外包 `<span @click>` 触发 message.warning——antd disabled button 不派发 click，故用包裹层承接提示。rowData 来自列表/树完整记录含 locked；骨架路径节点缺 locked 时按未锁处理，后端仍以 locked 为准拒改并走冲突提示。

## 5 遗留问题

- 取值方式切换入口待后续任务（见未做项）。
- `IotFormulaEditor` 的影响预览/试算仍是占位入口（历史 FI-0036 接线义务，本任务未触碰）。
- 父实例 PARENT 引用分组在实例取值编辑弹窗未接线，保留置灰提示；SELF 组已按当前实例清单接通。
