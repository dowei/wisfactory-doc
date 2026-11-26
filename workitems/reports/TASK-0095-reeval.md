# TASK-0095 scoped 复评报告

- 复评对象：修复增量 `09ae85cc → 57804b47`（2 文件），以及原 Important 发现。
- 依据：复评 brief、原评审报告 Important 段、修复增量 diff、修复后报告「评审退回修复」小节及修复后四门原始输出。
- 契约：fresh、no-subagents、只评审不修改。scoped 范围：只看修复增量与原 Important，不重开全场。

## 一、原 Important 是否解决：resolved

原发现：新增态（无 rowData）`loadItemOptions` 只在 `showModal` 的 `if (rowData)` 编辑分支调用，新增态无法加载「来源数据项 / 比值分子 / 比值分母」下拉，aggType=5 比值型指标无法直接新建。

修复增量（`metric-form-modal.vue`）：
- 归属实例 `a-select` 新增 `@change="onInstanceChange"`。
- 新增 `onInstanceChange(instanceId?)`：同步清空 `form.sourceItemId / ratioNumItemId / ratioDenItemId`，再 `await loadItemOptions(instanceId ?? form.instanceId)`；`loadItemOptions` 复用项目既有 `itemApi.queryItemList`（真实后端端点），无实例时空态。
- `defineExpose` 增列 `onInstanceChange`，原有 `showModal` / `validateRatio` 保留。

逐条判定：

1. **新增态是否加载下拉**：resolved。无实例时 `itemOptions` 诚实为空；用户选实例触发 `@change` → `onInstanceChange` → `loadItemOptions(instanceId)` → 三项下拉填充。回归用例断言 `queryItemList(1001)` 被调用且 `itemOptions === [{ label: 'I-001（用电量）', value: 11 }]`。
2. **instanceId 变化是否重载/清空失效引用**：resolved。`onInstanceChange` 先清空 `sourceItemId / ratioNumItemId / ratioDenItemId` 再按新实例重载，避免提交跨实例引用（后端 4001）。
3. **aggType=5 能否在新建表单选分子分母并提交**：resolved。回归用例在新增态（无 rowData）经选实例加载选项后，以 `ratioNumItemId=11 / ratioDenItemId=11` 直接调 `onSubmit`，断言 `addMetric` 收到含 `aggType=5` 与两项比值的 payload。先红后绿证据充分：修复前仅该用例 `queryItemList` 调用数为 0 而失败，修复后通过；修复后整组 metric 单测 2 文件 / 16 用例全绿，四门链式 exit 0。

## 二、修复增量是否引入新问题：无

Critical / Important / Minor 均无发现。逐项核查：

- **造假数据 / 错误端点**：无。仅复用 `itemApi.queryItemList(instanceId)` 真实端点，测试断言调用参数，无 mock 数据写进实现、无 `/openapi` 或自造代理。
- **破坏编辑态原有加载路径**：无。`showModal` 的 `if (rowData)` 分支中 `loadItemOptions(form.instanceId)` 原路径未被改动；实例 select 为 `:disabled="isEdit"`，编辑态不触发 `@change`，故 `onInstanceChange` 的清空逻辑不会在编辑态误触发。
- **defineExpose**：无回退，仅新增 `onInstanceChange`，`showModal` / `validateRatio` 保留，父级 ref 调用不受影响。
- **权限**：增量 2 文件无权限相关改动，无越权可见性回退。
- **校验回退**：增量未改 `onSubmit` / 校验逻辑，回归用例仍走正常提交并被后端 DTO 口径 payload 断言覆盖。

说明（非问题）：`onInstanceChange` 注释写「编辑态初始化」，但编辑态实际仍由 `showModal` 直接调用 `loadItemOptions`，注释与实际调用点略有出入，不影响功能，不构成缺陷。

## 三、结论

**可收尾。** 原 Important 已由真实 `@change` 加载路径 + 回归用例先红后绿确实解决；修复增量 2 文件未引入新的 Critical / Important / Minor 问题。
