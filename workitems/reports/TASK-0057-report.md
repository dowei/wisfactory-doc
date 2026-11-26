# TASK-0057 实施报告（公式编辑器组件——编辑器/引用选择器/客户端语法校验 + 模板数据项弹窗接入）

- 执行者：implementer（frontend 独立道）
- BASE：`46d9391`（开工时 `git rev-parse --short HEAD`，main）；HEAD：`587d5e6`（提交注明 TASK-0057，单 commit 6 文件 +967/-7）
- brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0057-brief.md`
- 验收口径：`pnpm build:test` / `pnpm lint` / `pnpm typecheck` 三门 + `pnpm test -- --run`（vitest）+ 改动范围核对；**不起 8098 联调**（同 0056 裁决）
- **验收结论：五条验收命令全部通过（EXIT=0 / 0 error / 全绿 / 无越界）**

---

## 一、改动文件清单（6 文件，范围核对见第五节）

| 文件 | 新/改 | 内容 |
| --- | --- | --- |
| `frontend/src/components/iot/formula/formula-rules.ts` | 新 | **规则表共享常量文件（brief 决策 1）**：`FORMULA_MAX_LENGTH=2000`（0011 口径）、相对引用形态 `^\{(SELF\|PARENT)\.([A-Za-z0-9_]+)\}$/i` 与花括号 token 字符集 `[A-Za-z0-9_.]+`（均 0013 RelativeRefExpander 同款含 CASE_INSENSITIVE）、`validateFormula()` 逐条规则（长度→花括号配平/嵌套→token 字符集/前缀白名单→圆括号配平）、错误文案常量 `FORMULA_ERROR_MESSAGES`（与后端 4001 族文案对齐）、`extractBraceTokens`（引用收集预演）、`buildRefToken`/`buildDefaultRefGroups`（选择器包装与两级分组构建，供 0036/0016 复用）、占位入口文案 `FORMULA_PENDING_TIPS` |
| `frontend/src/components/iot/formula/iot-formula-editor.vue` | 新 | 公式编辑器：monospace `a-textarea` + 实时校验提示（`validateFormula` computed，错误逐条渲染、语法通过显示语义级归属提示）+ 工具行（引用选择器 + 影响预览/试算占位按钮）。props：`modelValue` / `refGroups` / `rows` / `disabled` / `showPendingActions`；emits：`update:modelValue` / `validate-change` / `impact-preview` / `trial-calc`；expose `insertToken`（光标处插入，未聚焦退化为尾部追加）。**不引 CodeMirror**（一期 textarea+覆盖提示，见偏离节） |
| `frontend/src/components/iot/formula/iot-formula-ref-picker.vue` | 新 | 引用选择器（编辑器子件、可独立复用）：数据源由调用方传入（不发请求），两级分组=引用域（SELF 本实例/PARENT 父实例/ABSOLUTE 绝对全键）→组内数据项；**禁手输**（`a-select` 非 tags/combobox，选项仅来自传入清单，show-search 仅过滤）；选中自动包装 token：SELF/PARENT → `{SELF.code}`/`{PARENT.code}`，ABSOLUTE → `{code}`（0013 口径）；组级置灰带接线提示；插入后复位占位态 |
| `frontend/src/views/iot/template/components/template-item-modal.vue` | 改 | 默认公式字段由纯 `a-textarea` 替换为 `IotFormulaEditor`（`v-model:modelValue="itemForm.defaultFormula"`，rows=3，占位入口开启）；新增 `formulaRefGroups` computed：SELF 组=本模板数据项清单（已落库+待保存行，随增删实时刷新）、PARENT 组置灰提示「待 TASK-0036 接线」；表单规则 `defaultFormula max 2000` 保留（双网）；头部注释登记 TASK-0057 变更 |
| `frontend/tests/unit/components/iot/formula-rules.spec.ts` | 新 | 校验规则单测 32 例：合法形态（含大小写不敏感/绝对全键/函数包裹）、长度 2000 边界（2000 过/2001 拒/文案逐字对齐 0011）、花括号配平与顺序从严（`}{`）、嵌套拒绝、token 空与字符集、相对引用前缀白名单（`{SELF.}`/`{SELF.A.B}` 拒、`{OTHER.X}` 绝对键放行）、圆括号配平、多 issue 聚合、`extractBraceTokens`、`buildRefToken`/`buildDefaultRefGroups`、占位文案归属断言 |
| `frontend/tests/unit/components/iot/iot-formula-editor.spec.ts` | 新 | 编辑器组件级测试 15 例：渲染与实时校验（错误文案/通过文案/validate-change emit）、输入回传、选择器两级分组选项仅来自传入清单（禁手输断言 `mode` 为空）、选中包装 `{SELF.}`/`{PARENT.}`、父清单缺位组置灰含 TASK-0036 提示、占位按钮点击提示文案（message.info 打桩断言「待 TASK-0036 交付」「待 FEAT-0012 交付」）+ emit、`insertToken` 光标插入与 disabled 空操作 |

未动：`frontend/src/api/iot/**`（无需扩展——选择器数据由调用方以 props 传入，实例侧复用走既有 `itemApi.queryItemList`）、backend/、ingest/、其他域页面。

---

## 二、验收命令原始输出（五条全过）

### 1. `cd frontend && pnpm build:test` → EXIT=0

```text
$ pnpm build:test
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
transforming...
...（chunk 产物清单中略）
dist/css/iot-placeholder-Bmzk0EMC.css            0.06 kB │ gzip:   0.08 kB
dist/css/template-item-modal-Bujj7HnT.css        0.63 kB │ gzip:   0.31 kB
dist/js/iot-optimistic-lock-CHOgIYgR.js          0.04 kB │ gzip:   0.06 kB
dist/js/template-item-modal-DFzLtVk_.js          0.34 kB │ gzip:   0.23 kB
dist/js/iot-placeholder-DcV2i-ap.js              0.49 kB │ gzip:   0.43 kB
dist/js/iot-consts-P6ASvpZ9.js                   1.11 kB │ gzip:   0.86 kB
dist/js/template-item-modal.vue_vue_type_script_setup_true_lang-znEOCgFr.js   15.88 kB │ gzip:   6.30 kB
dist/js/index-BAsMtFso.js                      1,015.77 kB │ gzip: 329.40 kB
dist/js/index-UNe20RQA.js                      1,316.16 kB │ gzip: 432.29 kB

(!) Some chunks are larger than 500 kB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking: https://rollupjs.org/configuration-options/#output-manualchunks
- Adjust chunk size limit via build.chunkSizeWarningLimit.
✓ built in 1m 3s

EXIT=0（编辑器接入后 template-item-modal 构建通过——上述 template-item-modal chunk 即该弹窗产物）
```

### 2. `cd frontend && pnpm lint` → 0 error

```text
$ pnpm lint
$ eslint .
（警告明细 9 条，全部为存量文件——edit-lock-api.spec.ts / setting-drawer-persist.spec.ts /
 axios.spec.ts / redpack-list-helper.spec.ts / redpack-columns.spec.ts 的 prettier 格式警告，
 与本次 6 个改动文件无关，本次改动文件 0 warning）
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.

EXIT=0（0 error 达标；9 warnings 为存量，未清理——越界）
```

### 3. `cd frontend && pnpm typecheck` → 通过

```text
$ pnpm typecheck
$ vue-tsc --noEmit
（无任何输出，即 0 类型错误）

EXIT=0
```

### 4. `cd frontend && pnpm test -- --run` → 全绿（vitest run，工程有 vitest 配置：vitest.config.ts，include `tests/unit/**/*.spec.ts`）

```text
$ pnpm test -- --run
$ vitest run --run
RUN v2.1.9 D:/workspace/WisFactory/frontend

 ✓ tests/unit/components/iot/formula-rules.spec.ts (32 tests) 17ms
 ✓ tests/unit/components/iot/iot-formula-editor.spec.ts (15 tests) 1117ms
（其余 57 个既有 spec 文件全过，中略——59 文件合计）

 Test Files  59 passed (59)
      Tests   573 passed (573)
（本次新增 47 例：formula-rules 32 + iot-formula-editor 15；全量 573 无回归）

EXIT=0
```

### 5. 改动范围核对：`git diff --name-only 46d9391..HEAD`

```text
$ git diff --name-only 46d9391..HEAD
frontend/src/components/iot/formula/formula-rules.ts
frontend/src/components/iot/formula/iot-formula-editor.vue
frontend/src/components/iot/formula/iot-formula-ref-picker.vue
frontend/src/views/iot/template/components/template-item-modal.vue
frontend/tests/unit/components/iot/formula-rules.spec.ts
frontend/tests/unit/components/iot/iot-formula-editor.spec.ts

EXIT=0
```

逐条核对：前 4 条落在 brief 声明范围（`frontend/src/components/iot/formula/**` + `template-item-modal.vue`）；2 个测试文件在 `frontend/tests/unit/components/iot/`——brief 改动范围清单未列 tests/，但验收口径第 4 条强制要求「校验规则表单测/组件级测试（vitest）」，按工程 vitest.config 约定位置放置，登记为验收口径要求的必要产物，未触碰其他任何范围文件（backend/、ingest/、resources/、api/iot 均未动）。

---

## 三、前端校验文案 ↔ 后端错误分段映射表（brief 决策 1 第 3 项）

后端分段口径：详细设计 §3.4 错误码分段 **4001-4099 = 参数与校验段**（ResultCode 注释），公式语法错误族即 4001 段；`BusinessException(ResultCode.PARAM_ERROR)` 携带具体文案。

| 前端 issue 码（`FORMULA_ERROR_MESSAGES` 键） | 前端文案 | 后端同源依据 | 后端接管方 |
| --- | --- | --- | --- |
| `FORMULA_LENGTH_EXCEED` | `默认公式长度不能超过 2000` | `IotDataItemTemplateServiceImpl.validateFormula`（0011）逐字同文案；DDL default_formula varchar(2000) | 后端保存即校验（现役） |
| `FORMULA_BRACE_UNBALANCED` | `默认公式基础格式非法：括号/占位符不配平（花括号 { } 数量或顺序不配对）` | 0011「默认公式基础格式非法：括号/占位符不配平（…）」同族，前端按花/圆拆分细化 | 后端保存即校验（现役，计数口径） |
| `FORMULA_PAREN_UNBALANCED` | `默认公式基础格式非法：括号/占位符不配平（圆括号 ( ) 数量或顺序不配对）` | 同上（0011 括号配平） | 后端保存即校验（现役） |
| `FORMULA_TOKEN_EMPTY` / `FORMULA_TOKEN_CHARSET` | 空占位符 / `花括号占位符内容仅允许字符 A-Z a-z 0-9 _ .` | 0013 `RelativeRefExpander.BRACE_REF` `\{([A-Za-z0-9_.]+)\}` 字符集口径镜像（后端不拒畸形 token——落生成链/FEAT-0012，前端从严前置） | FEAT-0012 公式校验器（4001） |
| `FORMULA_RELATIVE_SHAPE` | `相对引用编码仅允许字符 A-Z a-z 0-9 _（形态 {SELF.code} / {PARENT.code}…）` | 0013 `RELATIVE_REF` `\{(SELF\|PARENT)\.([A-Za-z0-9_]+)\}` 含 `CASE_INSENSITIVE` 同款镜像（含前缀白名单：token 以 SELF./PARENT. 开头必须满足完整形态；`{OTHER.X}` 类按绝对全键放行——0013 口径） | FEAT-0012（4001）；引用存在性同归 FEAT-0012/REQ-0016 |
| `FORMULA_BRACE_NESTED` | `花括号占位符不允许嵌套（如 {SELF.{X}}）` | 前端从严项（0011 纯计数不查嵌套，brief 决策 1 明文「花括号配对/**嵌套合法**」） | FEAT-0012（4001） |
| ——（运行时，前端不预判） | —— | 0013 `expand`：顶级实例 PARENT 引用抛 `BusinessException(PARAM_ERROR)`「顶级实例不支持 PARENT 相对引用（无父实例可解析）…」 | 0013 生成链 fail-closed（400 段） |
| `FORMULA_SEMANTIC_NOTICE`（语法通过时展示） | `语法检查通过（引用存在性/循环依赖/函数集白名单以后端保存校验为准——4001 族，FEAT-0012）` | REQ-0016 验收 1/5（循环依赖/引用不存在/函数集外拒存 4001） | FEAT-0012（4001 族） |

不变式：**前端拒绝集 ⊇ 后端拒绝集**（后端拒的前端全拒；前端另拒嵌套/字符集/前缀形态/配对序等后端一期放行的畸形形态——前置纠错，保存校验一律以后端为准）。

---

## 四、未做项与原因——三项接线义务明细（brief 决策 2，占位实现）

| 项 | 前端现状（本期交付） | 缺口（后端 API） | 接线归属 | 接线点 |
| --- | --- | --- | --- | --- |
| 影响预览（受影响上级项与指标，REQ-0016 验收 3） | 编辑器工具行「影响预览」占位按钮：点击 `message.info`「影响预览待 TASK-0036 交付（引用图反查 API 未提供，当前为占位入口）」+ emit `impact-preview` | 引用图反查 API——0013 已建 `iot_data_item_ref` 表但无查询端点 | **TASK-0036**（绑定/公式/汇总配置接口与引用边维护） | 调用方监听编辑器 `impact-preview` 事件调 0036 端点后，将 `showPendingActions` 收回为自有按钮/加载态；`FORMULA_PENDING_TIPS` 同步更新 |
| 试算入口（单点 + 窗口曲线，REQ-0016 / ADR-0013 增补 5） | 「试算」占位按钮：点击提示「试算待 FEAT-0012 交付（Aviator 试算端点未提供，当前为占位入口）」+ emit `trial-calc` | Aviator 试算端点（validate/preview 同引擎同口径）+ 窗口曲线沙箱试算 | **FEAT-0012 / TASK-0045** | 同上（`trial-calc` 事件） |
| 循环依赖即时反馈（TC-0012 步骤 2） | 仅语法级提示：状态行常显「引用存在性/循环依赖/函数集白名单以后端保存校验为准——4001 族，FEAT-0012」；前端不实现语义级循环检测（brief 决策 2 明文） | 保存时循环检测 4001 拒存（REQ-0016 验收 1；0013 生成链 fail-closed 终检已兜底生成态） | **FEAT-0012**（详细设计 §1.7 计算引擎） | 无前端改造需求；可选优化为 4001 返回后将后端文案透传进编辑器错误区（`validate-change` 已 emit，父级可注入外部错误） |

---

## 五、偏离计划之处

1. **语法高亮后置**（brief 决策 1 明示可后置，此处按约定声明）：一期 textarea + 实时校验提示层，未引 CodeMirror 重依赖；语法高亮（覆盖层或编辑器内核）留待后续任务。
2. **超长不再静默截断**：旧 textarea `:maxlength=2000` 静默剪裁（2000 长度规则实际永不触发）；编辑器不截断，超长实时显示 0011 同文案「默认公式长度不能超过 2000」，后端仍兜底拒存——校验规则因此真正可被用户看见。
3. **前端从严项**（后端一期计数口径放行、前端拒绝，均已登记）：花括号嵌套拒绝；`)(`/`}{` 顺序颠倒拒绝（0011 为纯计数配平）；token 字符集/相对引用前缀形态校验（后端对畸形 token 不拒，落 FEAT-0012/生成链）。不改变「保存以后端为准」。
4. **测试文件落位**：brief 改动范围清单未显式列 `frontend/tests/**`，但验收口径第 4 条要求 vitest 校验规则单测——按工程 `vitest.config.ts` include 约定放 `tests/unit/components/iot/`（工程既有测试全部位于该目录树）。
5. **未动 `frontend/src/api/iot/**`**（brief 允许「如需扩展」）：引用选择器采用数据注入式设计（`refGroups` props），模板弹窗用弹窗内已加载清单、实例侧（0036）用既有 `itemApi.queryItemList` 即可，组件不与具体 API 耦合，更利 0036/0016 复用。
6. 提交按契约走 main 单 commit（`587d5e6`），未派 subagent、未改 workitems 库文件、未把 TASK 置「已完成」（状态仍为协调者管辖）。

## 六、遗留问题

1. **模板态 PARENT 分组置灰**：模板弹窗拿不到「父实例数据项清单」（父实例模板随实例树而定，模板无父概念）——组件已支持 `parentItems` 传入，等 TASK-0036 在实例侧接线后生效；本期 PARENT 组显示置灰提示「父实例数据项清单待 TASK-0036 接线」。
2. **影响预览/试算为占位**：按钮仅提示不调接口（第四节明细）；编辑器 emit 已备好，后端交付后前端接线量约一处事件监听 + 一处 API 调用 + 按钮态改造。
3. **SELF 自引用不拦截**：公式引用本项自身编码（自环）前端不报错——循环依赖归后端 FEAT-0012（brief 决策 2：前端仅语法级）。
4. **lint 存量 9 warnings**：survey/layout/lib 既有测试文件的 prettier 格式警告，与本次改动无关，未清理（越界）。
5. **复用说明（供 0036/0016）**：`formula-rules.ts` 全部为纯函数/常量可直接单测引用；`iot-formula-editor.vue` props 即接入面——`modelValue`、`refGroups`（`buildDefaultRefGroups` 或自组）、`showPendingActions`、`disabled`；`validate-change` 可用于外层表单联动（如保存前阻断）。
6. 实施中途发现并修正一处与后端口径不一致的实现（相对引用关键字大小写敏感→改为与 Expander `CASE_INSENSITIVE` 一致），由单测先行暴露——即 TDD 红绿记录（测试先行 32 例，红→绿）。
