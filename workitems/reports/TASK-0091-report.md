# TASK-0091 执行报告（BUG-0008 微改修复）

- BASE：abad464e2e4c34f17eae62654fa756005462b169
- 提交哈希：a09e258699bd62ed236c48cfb4447b1442b5cdab
- 执行时间：2026-11-25
- 状态：代码已提交；TASK 状态未改（协调者后续验收并置「已完成」）

## 修复内容

1. `frontend/src/views/iot/instance/components/instance-item-drawer.vue`
   - `<script setup>` 末尾（`openParams` 之后、`</script>` 之前）新增：
     ```
     defineExpose({
       showDrawer,
       onClose,
     });
     ```
   - 对齐同目录 instance-form-modal.vue:265 / instance-detail-modal.vue:62 / instance-params-modal.vue:162 范式；组件内部逻辑零改。
2. 新增 `frontend/tests/unit/views/iot/instance-item-drawer-ref.spec.ts`
   - 通过真实父组件 template ref（而非 VTU `wrapper.vm`，后者在 dev 模式会绕过 expose 边界）复现 BUG-0008。
   - 断言父 ref 拿到的 `showDrawer` / `onClose` 为 function；调用 `showDrawer(INSTANCE_ROW)` 后抽屉 open、Teleport 到 body 的标题「实例数据项 - INS-001」可见；`onClose()` 后关闭。
   - 红/绿验证：将组件回退到 HEAD（无 defineExpose）跑该 spec → `expected 'undefined' to be 'function'` 必红；恢复修复后 → 1 passed。

## 验收 1：四门全绿（原样贴回输出）

命令：`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`

```
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
✓ built in 1m 36s
$ eslint .
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.
$ vue-tsc --noEmit
✓ tests/unit/views/iot/instance-item-drawer-ref.spec.ts (1 test) 516ms
   ✓ 实例数据项抽屉父组件 ref 暴露（BUG-0008） > 父组件 ref 可拿到 showDrawer/onClose；调用后抽屉打开且标题可见 513ms
 Test Files  74 passed (74)
      Tests  650 passed (650)
```

lint 9 条 warning 均为既有文件的 prettier 提示（0 errors）；typecheck 无输出即通过；650 = 既有 649 零回归 + 新测试 1。

## 验收 2：范围核对

命令：`git diff --name-only abad464e..HEAD`

```
frontend/src/views/iot/instance/components/instance-item-drawer.vue
frontend/tests/unit/views/iot/instance-item-drawer-ref.spec.ts
```

仅 2 个文件，符合契约；未 `git add -A`、未 push、未置 TASK 已完成。

## 备注

- 浏览器实测「数据项」按钮打开抽屉由协调者在修复后自行验证；本执行者仅保证组件层 ref 调用测试覆盖。
- 工作区另有既存未跟踪目录 `.agents/`、`.tmp-t0044v/`（非本次改动，未触碰）。
