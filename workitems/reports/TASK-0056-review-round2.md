# TASK-0056 修订轮 scoped 复评报告（仅判 I-1 修复 + 新缺陷）

- 评审者：fresh reviewer（不共享协调者上下文；no-subagents；只评审不修改任何代码文件）
- 评审范围：仅修订提交 `11e200c`（3 文件 +10 −4）；不重开首轮全场，原 9 条 Minor 按裁决 deferred 不在判定范围
- 输入：review2 brief + 原评审报告 + fixdiff 包（通读）+ 实施报告修订轮节；另对工作区现场独立 grep 复核（HEAD=`11e200c`，`git diff 11e200c -- frontend/src/views/iot/` 为空，fixdiff 与提交一致）

---

## 一、结论

**I-1 已修复；新发现 0 Critical / 0 Important / 2 Minor（不阻塞，建议记 deferred）。**

## 二、判定理由（一句）

`dict-import-modal.vue` 的 `defineEmits` 与两处 emit（L126 导入成功 / L160 close 的 failCount===0 分支）已统一改为 `reloadList`，与父页 `dict/index.vue` L103 `@reloadList="ajaxQuery"` 精确配对，导入成功后列表恢复自动刷新；修复 diff 三处改动均为事件名对齐/补齐，无事件语义破坏、无误修、无断言放松。

## 三、核对证据

### 1. I-1 消除（现场 grep + 文件复读）

- emit 侧：`dict-import-modal.vue` L90 `defineEmits(['reloadList'])`、L126/L160 `emit('reloadList')`（修前 `['reload']`×3 已清零）。
- 监听侧：`dict/index.vue` L103 `<DictImportModal ... @reloadList="ajaxQuery" />` 不变，两侧一致；全 `views/iot` 无 `@reload` 代码残留（仅注释中提及）。

### 2. 同类错位两处（随 I-1 清查）语义正确

- `instance-params-modal.vue`：原未声明/未触发任何事件 → 现补 L77 `defineEmits(['reloadList'])` + L153 保存成功后 emit、`onClose()` 前；catch 失败路径不 emit、非 edit 模式不 emit——语义正确。
- `instance-item-drawer.vue` L48：`@reload`（对原子组件是死监听）→ `@reloadList="ajaxQueryItems"`，`ajaxQueryItems`（L99）存在，两侧统一。

### 3. emit-监听配对全核（本评审独立 grep，非转录报告）

emit 侧现场 grep：9 个 emit 组件共 9 处 `defineEmits` / 10 处 emit 触发，全部 `reloadList`（tenant/dict-form/dict-import×2/template-form/template-item/instance-form/instance-params/gateway/channel）；`components/iot` 仅 `iot-conn-config-editor.vue` 的 `update:modelValue`（标准 v-model，非刷新事件，配对正常）。监听侧 8 处 `@reloadList` 全部有对应子组件 emit；`template-item-modal` emit 无父级监听保持原状（M-4 deferred，未恶化）。无其他事件名、无 `$emit`、无双引号变体。

### 4. 无新引入缺陷

- 改动范围：3/3 文件在 `views/iot` 声明范围内；未触碰 tests/（断言无放松，lint 基线 9 warning 与首轮逐一相同）。
- 三静态门复跑输出（build:test EXIT=0 / lint 0 error / typecheck EXIT=0）贴于实施报告修订轮节，本评审未复跑（只评审契约）。

## 四、新发现

| 级别 | # | 出处 | 说明 |
| --- | --- | --- | --- |
| Minor | R-1 | `frontend/src/views/iot/dict/components/dict-import-modal.vue` L126 + L160 | 导入全成功路径存在双 emit：`doImport` 成功即 emit 刷新一次，回执展示后用户点关闭时 `close()` 的 `failCount===0` 分支再 emit 一次 → 同一次导入触发两次列表刷新。双 emit 结构系修前既有（修前因事件错位整体落空），本次修复使其成为可观察行为；两次均为幂等列表 GET，无数据影响，仅冗余请求，建议后续收敛为单 emit 点（如 close 分支清除 receipt 或删除其一）。不阻塞本轮。 |
| Minor | R-2 | `TASK-0056-report.md` 修订轮节 prose | 计数失准：修订轮节称「8 个 emit 组件 × 8 个监听位」「10 处声明 / 12 处触发」，现场 grep 实为 **9 个 emit 组件、9 处声明 / 10 处触发**（其逐字贴出的 grep 块与 14 行配对表本身完整、正确，仅 prose 计数错误）——与首轮 M-1 同类报告卫生问题，不影响配对结论，建议随 M-1 一并勘误。 |

（原 9 条 Minor 均未复查、未见因本次修复恶化：`template-item-modal` emit 无监听保持原状。）
