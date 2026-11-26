# TASK-0057 diff 评审报告（fresh reviewer）

- 评审对象：单提交 `587d5e6`（6 文件，diff 包 `reports/TASK-0057-diff.txt`，已与仓库 `git diff --name-only 46d9391..587d5e6` 核对一致，无越界——全部落在 brief 声明的 `frontend/src/components/iot/formula/**`、`template-item-modal.vue` 及验收要求的 tests）
- 依据：实施 brief 决策 1/2/3 + L1-execute §13；语法口径权威逐条对照 `backend/.../dataitem/service/support/RelativeRefExpander.java`（0013）与 `IotDataItemTemplateServiceImpl.validateFormula`（0011，实测源码 MAX_LEN_FORMULA=2000、文案逐字核对）
- 实测核验：fresh 重跑两个新增 spec（`formula-rules` 32 例 + `iot-formula-editor` 15 例）→ 47/47 全绿；6 个改动文件 eslint EXIT=0（0 error 0 warning）

## 结论

**通过（无 Critical / 无 Important，Minor 5 条记 TASK `任务结果` deferred，不进修订轮）。**

### 硬约束核对（校验规则与后端同口径，逐条对照源码）

| 规则 | 后端依据 | 前端实现 | 判定 |
| --- | --- | --- | --- |
| {TOKEN} 字符集 | Expander `BRACE_REF=\{([A-Za-z0-9_.]+)\}` | `FORMULA_BRACE_TOKEN_PATTERN=/^[A-Za-z0-9_.]+$/` | 同口径 |
| 相对引用形态/前缀白名单 | Expander `RELATIVE_REF=\{(SELF\|PARENT)\.([A-Za-z0-9_]+)\}` 含 CASE_INSENSITIVE | `FORMULA_RELATIVE_REF_PATTERN` 同款 /i；token 以 self./parent. 开头（大小写不敏感）必须整体匹配该形态，其余按绝对全键放行（`{OTHER.X}` 不拒，与后端一致） | 同口径 |
| 长度 ≤2000 | 0011 `MAX_LEN_FORMULA=2000`，文案「默认公式长度不能超过 2000」 | `FORMULA_MAX_LENGTH=2000`，文案逐字一致 | 同口径 |
| 括号配平 | 0011 纯计数（`)(` 放行） | 配对序扫描（计数不转负 + 归零），从严超集 | 从严登记（brief 决策 1 明文） |
| 花括号配对/嵌套 | 0011 纯计数不查嵌套 | 配对序 + 嵌套拒绝；从严项已在文件头与报告映射表登记 | 从严登记（brief 决策 1 明文） |
| 不变式「前端拒绝集 ⊇ 后端拒绝集」 | 0011 拒绝集（超长/计数不配平）+ 0013 运行时（顶级 PARENT 抛 400，前端不预判、登记归属） | 逐条推演：`{SELF.{X}}` 后端展开成残留花括号废式、`{SELF.A.B}` 后端当绝对键、畸形字符集 token 后端残留花括号——前端全拒且后端拒的前端全拒 | 成立 |

### brief 契约其余各条

- 引用选择器禁手输：`a-select` 非 tags/combobox，搜索仅过滤，选项仅来自传入分组（组件测试断言 `mode` 为空）——达标；上下文自动包装 `{SELF.code}`/`{PARENT.code}`/ABSOLUTE `{code}`（`buildRefToken`）——达标。
- 接线义务三件（影响预览/试算/循环依赖）：占位按钮仅 `message.info` 提示 + emit `impact-preview`/`trial-calc`，无任何假 API 调用；循环依赖仅语法级提示并声明归后端 FEAT-0012——达标（无假功能）。
- template-item-modal 接入：仅替换字段组件，既有编辑流（回填/待保存清单/批量提交/表单规则）未破坏；`defaultFormula max 2000` 表单规则保留（双网）——达标。
- 组件 props 契约（0036/0016 复用面）：`FormulaRefOption/FormulaRefGroup/FormulaRefType`、`buildRefToken/buildDefaultRefGroups/extractBraceTokens` 导出齐全，数据注入式设计（组件不发请求）——达标。
- vitest 断言真实性：断言为行为级（真实挂载 Antd、经 picker `onSelect` 真链路 emit、文案逐字断言、长度 2000/2001 边界），fresh 重跑 47/47 全绿——真实。

## 发现清单

| 严重级 | 出处 | 说明 |
| --- | --- | --- |
| Minor | `frontend/src/components/iot/formula/iot-formula-editor.vue` 错误列表 `v-for :key="issue.code"` | 多个 token 触发同码问题（如两个 `{}`、多处字符集违规）时 issue 码重复，Vue 重复 key 有 dev 告警与 patch 隐患——建议按 index 或去重聚合。 |
| Minor | `frontend/src/views/iot/template/components/template-item-modal.vue` `formulaRefGroups` | SELF 组未传 `selfEmptyHint`：模板尚无任何数据项（新增首项场景）时 SELF 组显示为「可展开的空分组」且无提示，与 PARENT 组的空态置灰提示行为不一致。 |
| Minor | `frontend/src/components/iot/formula/iot-formula-editor.vue` `insertToken` | 注释称「取不到光标时退化为尾部追加」，但 `selectionStart` 恒为 number，尾部追加分支仅在 textarea ref 未解析时触达，未聚焦时实际插在保留光标位/0——守卫与注释不符（行为无害）。 |
| Minor | `iot-formula-editor.vue` emit `validate-change` + `template-item-modal.vue` `saveItem` | 实时语法错误不阻断弹窗保存（表单规则仅 max 2000，语法错误可入待保存清单、由后端 4001 拒存）——与「保存校验以后端为准」口径一致且报告已登记为可选联动，属 0036 接线时的 UX 待办。 |
| Minor | `template-item-modal.vue` `formulaRefGroups` 未去重 | 待保存清单出现重复 dataCode（后端 4002 才拒存）时，SELF 组出现重复选项，纯展示层瑕疵。 |
