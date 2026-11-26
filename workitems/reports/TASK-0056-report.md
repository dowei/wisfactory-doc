# TASK-0056 实施报告（管理页第一组——租户/字典/模板/实例/网关/通道基础 CRUD 页）

- 执行者：implementer（frontend 独立道，与 backend 0019 并行互不干扰）
- BASE：`fa12df9`；HEAD：`424031b`（main，提交注明 TASK-0056）
- brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0056-brief.md`
- 验收口径：三条静态门（build:test / lint / typecheck）+ 改动范围核对 + 页面-接口映射核对表；**不起 8098 联调**（brief 决策 2）

---

## 一、改动文件清单（30 文件，全部落在 brief 声明范围）

范围声明：`frontend/src/views/iot/**`、`frontend/src/api/iot/**`、`frontend/src/components/iot/**`。`git diff --name-only fa12df9..HEAD` 输出（= 第四条验收命令原始输出）逐条核对无越界。

### api/iot（6 新 1 改）

| 文件 | 内容 |
| --- | --- |
| `frontend/src/api/iot/tenant-api.ts`（改） | 补 `initTenantAdmin`（POST /iot/tenant/{id}/init-admin，4017） |
| `frontend/src/api/iot/dict-api.ts`（新） | /iot/dict 五接口 + 导入三接口（multipart / 模板下载 / 错误行清单下载） |
| `frontend/src/api/iot/template-api.ts`（新） | /iot/template 五接口 + /iot/template/item 四接口（查询共用 list、写共用 item:edit） |
| `frontend/src/api/iot/instance-api.ts`（新） | /iot/instance tree/list/{id}/add/edit/remove |
| `frontend/src/api/iot/item-api.ts`（新） | /iot/item/list + regenerate / params 覆盖 / effective-params |
| `frontend/src/api/iot/gateway-api.ts`（新） | /iot/gateway 四接口（**不提供 GET /{id}**——后端无 detail 端点，0017 口径） |
| `frontend/src/api/iot/channel-api.ts`（新） | /iot/channel 五接口（list gatewayId 必传、detail 含 conn_config 原文） |

### components/iot（3 新）

| 文件 | 内容 |
| --- | --- |
| `iot-consts.ts` | 枚举与后端注解逐条对照（节点类型/数据类型/类别/值性质/取值方式/聚合/协议四值与预留协议/maxLimit=500/item 批量上限 500） |
| `iot-optimistic-lock.ts` | 乐观锁冲突提示组件化预留（brief 决策 3：iot 域无 @Version——`IOT_OPTIMISTIC_LOCK_ENABLED=false` 登记不触发，各写操作 catch 统一接入，后端引入 @Version 后开开关即生效） |
| `iot-conn-config-editor.vue` | 通道 conn_config JSON 编辑器：协议联动前置提示（mqtt/http/modbus/opcua-v1.json 语义镜像）、必含键/`/` 开头/凭据键全深度扫描（镜像 ChannelConnConfigValidator 黑名单）、提交前 validate() 兜底；文案均标注「后端校验为准」 |

### views/iot（15 新 6 改）

| 页 | 文件 | 交付内容 |
| --- | --- | --- |
| 租户 | `tenant/index.vue`（改）、`components/tenant-form-modal.vue`（改）、`components/tenant-admin-modal.vue`（新） | 0055 骨架升级完整页：查询按钮挂 iot:tenant:list、行选择+批量删除、初始化租户管理员弹窗（SM4 密文 password，TenantAdminCreateForm 口径）、表单校验对齐 SysTenant 注解（编码 ≤32 且 [A-Za-z0-9_]、name ≤100、contact ≤50/32、remark ≤500）、删除二次确认补「编码不复用」文案 |
| 字典 | `dict/index.vue`（改）、`components/dict-form-modal.vue`（新）、`components/dict-import-modal.vue`（新） | 完整 CRUD+导入：data_code/data_type 编辑禁输（锁定）；「禁用」按钮按 status 显隐（DELETE 语义=置禁用，无启用路径）；导入弹窗模板下载→上传→逐行回执渲染→错误行清单下载全闭环（整批拒绝 body code=400 + data=receipt，silentError 自渲染） |
| 模板 | `template/index.vue`（改）、`components/template-form-modal.vue`（新）、`components/template-item-modal.vue`（新） | 完整 CRUD+数据项子表管理弹窗（字典选码下拉仅启用项、待保存清单+一次批量 POST ≤500、编辑 data_code 禁输、删除逻辑删二次确认）；**状态规则入口占位按钮**（点击提示待 TASK-0048） |
| 实例 | `instance/index.vue`（改）、`components/instance-form-modal.vue`（新）、`components/instance-detail-modal.vue`（新）、`components/instance-item-drawer.vue`（新）、`components/instance-params-modal.vue`（新） | 左树（iot:instance:tree，名称过滤+点击按编码前缀检索子树）+右列表/批量删除；详情弹窗（iot:instance:query）；表单：新增只传 shortCode、nodeType 联动模板必填（4/5）与部门必填（1/2）、编辑五字段一致性快照回传、locked=1 仅 status 可改；数据项抽屉（清单/按模板重生成）；参数覆盖弹窗（effective-params 三态来源标注 + 四字段覆盖，留空=null=清除回归继承） |
| 网关 | `gateway/index.vue`（改）、`components/gateway-form-modal.vue`（新）、`components/gateway-detail-modal.vue`（新） | 完整 CRUD：gateway_code 编辑禁输（不可改）、回写字段组五字段只读展示、详情弹层用列表行数据（后端无 detail 端点）、新建不携带 status（一律启用） |
| 通道 | `channel/index.vue`（改）、`components/channel-form-modal.vue`（新）、`components/channel-detail-modal.vue`（新） | 网关下拉必选（gatewayId 必传前置拦截，未选不发请求）；conn_config 走公共编辑器（协议联动）；MODBUS/OPCUA 标注「未启用适配器」+status 强制 0（radio 禁用）；编辑 channel_code/gateway_id 禁输；conn_config 未变更不携带（后端 null 不进更新集保持原值）；详情含 conn_config JSON 美化展示 |
| 占位 | `point/index.vue`、`metric/index.vue` | **维持占位未动**（后端未交付，0021/0043 后升级） |

未动文件：router 既有文件（菜单动态注册已就绪，0055 口径）、views 非 iot 目录、backend/、ingest/、resources/。

---

## 二、验收命令原始输出

### 1. `cd frontend && pnpm build:test` → EXIT=0

```text
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
transforming...

./icon.svg referenced in D:/workspace/WisFactory/frontend/node_modules/.pnpm/vue3-json-viewer@2.3.1_vue@3.4.27_typescript@5.6.3_/node_modules/vue3-json-viewer/dist/index.css didn't resolve at build time, it will remain unchanged to be resolved at runtime
...（660 行 chunk 产物清单，中略）
dist/js/index.esm-E1hPCUwe.js         458.39 kB │ gzip: 137.18 kB
dist/js/index.esm-B7hxPDTL.js         547.28 kB │ gzip: 157.18 kB
dist/js/index-BNgoIe2l.js             652.71 kB │ gzip: 205.35 kB
dist/js/code-generator-preview-modal-C-j3dAkv.js  915.72 kB │ gzip: 303.29 kB
dist/js/index-DwN9dPDV.js             928.82 kB │ gzip: 274.02 kB
dist/js/index-Df3fix1U.js             992.25 kB │ gzip: 159.93 kB
dist/js/index-DP6JzB6w.js           1,002.50 kB │ gzip: 323.21 kB
dist/js/index-BAsMtFso.js           1,015.77 kB │ gzip: 329.40 kB
dist/js/index-fapfwMnJ.js           1,316.16 kB │ gzip: 432.25 kB

(!) Some chunks are larger than 500 kB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking: https://rollupjs.org/configuration-options/#output-manualchunks
- Adjust chunk size limit via build.chunkSizeWarningLimit. (chunk warning 可忽略——brief 验收口径)
✓ built in 2m 21s

BUILD_EXIT=0
```

（注：`vue3-json-viewer ./icon.svg` 与 chunk>500kB 两条 warning 均为基线既有，非本任务引入；brief 明示 chunk warning 可忽略。）

### 2. `cd frontend && pnpm lint` → EXIT=0，0 error（9 warning 全部为基线既有 tests/ 文件 prettier 格式告警，改动范围核对确认本任务未触碰 tests/）

```text
$ eslint .

D:\workspace\WisFactory\frontend\tests\unit\api\survey\edit-lock-api.spec.ts
  13:9  warning  Replace `␍⏎··acquireEditLockRequest,␍⏎··heartbeatEditLockRequest,␍⏎··releaseEditLockRequest,␍⏎··takeoverEditLockRequest,␍⏎` with `·acquireEditLockRequest,·heartbeatEditLockRequest,·releaseEditLockRequest,·takeoverEditLockRequest·`  prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\layout\setting-drawer-persist.spec.ts
  136:36  warning  Replace `"import·HeaderSetting"` with `'import·HeaderSetting'`  prettier/prettier
  137:35  warning  Replace `"import·HeaderSetting"` with `'import·HeaderSetting'`  prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\lib\axios.spec.ts
   44:137  warning  Replace `␍⏎··await·import('/@/lib/axios'` with `·await·import(␍⏎··'/@/lib/axios'␍⏎`  prettier/prettier
  366:23   warning  Replace `blobResponse({ ... })` with `␍⏎········blobResponse({ ... })␍⏎······`  prettier/prettier
  376:21   warning  Replace `·Promise.resolve(blobResponse({ ... }))` with `␍⏎······Promise.resolve(blobResponse({ ... }))␍⏎····`  prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\survey\redpack-list-helper.spec.ts
  12:76  warning  Delete `␍⏎·`  prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\views\survey\redpack-columns.spec.ts
  35:4  warning  Replace `·dataId:·1, ...` with `␍⏎····dataId:·1,␍⏎···· ...`  prettier/prettier
  37:4  warning  Replace `·dataId:·2, ...` with `␍⏎····dataId:·2,␍⏎···· ...`  prettier/prettier

✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.

EXIT=0
```

（本任务 iot 文件曾产生的 27 条 prettier warning 已 `eslint --fix` 清零；`npx eslint "src/views/iot/**" "src/api/iot/**" "src/components/iot/**"` 复核输出为空=0 问题。）

### 3. `cd frontend && pnpm typecheck` → EXIT=0

```text
$ vue-tsc --noEmit
TYPECHECK_EXIT=0
```

（中途曾报 4 个 TS 错误：tenant/index.vue 对 `ref([])` 推断 never 的行属性访问 3 处、instance-detail-modal 对 `Record<string, unknown>` 的 label 传参 1 处——分别以 `TenantRow` 行接口与 `as number` 收敛后复跑通过。）

### 4. 改动范围核对：`git diff --name-only fa12df9..HEAD` → 全部 30 文件落在声明内

```text
frontend/src/api/iot/channel-api.ts
frontend/src/api/iot/dict-api.ts
frontend/src/api/iot/gateway-api.ts
frontend/src/api/iot/instance-api.ts
frontend/src/api/iot/item-api.ts
frontend/src/api/iot/template-api.ts
frontend/src/api/iot/tenant-api.ts
frontend/src/components/iot/iot-conn-config-editor.vue
frontend/src/components/iot/iot-consts.ts
frontend/src/components/iot/iot-optimistic-lock.ts
frontend/src/views/iot/channel/components/channel-detail-modal.vue
frontend/src/views/iot/channel/components/channel-form-modal.vue
frontend/src/views/iot/channel/index.vue
frontend/src/views/iot/dict/components/dict-form-modal.vue
frontend/src/views/iot/dict/components/dict-import-modal.vue
frontend/src/views/iot/dict/index.vue
frontend/src/views/iot/gateway/components/gateway-detail-modal.vue
frontend/src/views/iot/gateway/components/gateway-form-modal.vue
frontend/src/views/iot/gateway/index.vue
frontend/src/views/iot/instance/components/instance-detail-modal.vue
frontend/src/views/iot/instance/components/instance-form-modal.vue
frontend/src/views/iot/instance/components/instance-item-drawer.vue
frontend/src/views/iot/instance/components/instance-params-modal.vue
frontend/src/views/iot/instance/index.vue
frontend/src/views/iot/template/components/template-form-modal.vue
frontend/src/views/iot/template/components/template-item-modal.vue
frontend/src/views/iot/template/index.vue
frontend/src/views/iot/tenant/components/tenant-admin-modal.vue
frontend/src/views/iot/tenant/components/tenant-form-modal.vue
frontend/src/views/iot/tenant/index.vue
```

判定：30/30 全落在 `views/iot`、`api/iot`、`components/iot` 三目录内，无越界。提交 `424031b` 仅含上述文件（git add 逐目录指定）。

---

## 三、未做项与原因

| 未做项 | 原因 |
| --- | --- |
| 8098 后端联调 / 8099 dev server 冒烟 | brief 决策 2（关键裁决）：backend 串行道被 0019 占用，8098 本轮不归前端；运行期验证归集成轮（FEAT-0006 全齐后的 0068 或专项联调）。8099 预览对三条静态门无增益，未启用 |
| 点位/指标两域完整页 | brief 决策 1：两域后端未交付，占位页维持（0021/0043 交付后再升级） |
| 状态规则配置界面实体化 | iot_status_rule 后端未交付（TASK-0048）——仅按 brief 交付入口占位按钮 |
| 乐观锁冲突提示实际触发 | brief 决策 3：iot 域无 @Version 字段——提示逻辑组件化（`iot-optimistic-lock.ts`）但登记不触发 |
| TC-0018 全流程手工冒烟 | 用例为 UAT 手工场景且依赖真实接口联调，本轮验收口径不含（见决策 2） |
| 模板数据项公式编辑器（语法校验/引用选择器/试算） | 归 TASK-0070 第二组；本期 defaultFormula 仅文本域+长度 ≤2000 前置校验（与后端一期口径一致：仅长度+基础格式校验） |

---

## 四、偏离计划之处

1. **状态规则入口按钮的 v-privilege 绑定**：brief 要求「入口按钮 v-privilege 占位」，但状态规则独立权限点（REQ-0034 高危操作）尚未随 0048 登记种子。暂绑既有 `iot:template:edit`（4027，规则配置属模板编辑域），点击提示「待 TASK-0048 交付」；0048 交付权限位后替换绑定（遗留问题 4）。
2. **实例编辑无 remark 字段**：brief 任务正文有「remark 类」措辞——按后端 0012 落地口径（IotInstanceCreateForm/UpdateForm javadoc：iot_model_instance 附录 A p31 逐字 DDL 无 remark 列），表单未提供 remark，可变集=instanceName/deptId/sort/status。
3. **模板数据项新增交互形态**：brief 未规定具体交互——落地为「待保存清单 + 一次批量提交」以真实消费批量接口（POST {templateId, items[]}，单批上限 500 与 IotTemplateItemBatchForm 对齐）；编辑/删除单条直连对应接口。
4. **字典页「删除」按钮文案与显隐**：DELETE 语义=置禁用（不可逆、无启用路径）——按钮文案用「禁用」并按 `status===1` 显隐，二次确认文案明示「不可恢复启用，编码永不复用」（比 brief「停用/删除二次确认」更贴合后端语义）。
5. **工作区并行道隔离**：实施期间工作区存在 backend 道未提交文件（gateway/auth 4 个 java 修改 + credential 域未跟踪文件 + 20261122_iot_credential_domain.sql）——本任务未触碰、未纳入提交；`git add` 仅限 frontend/src/{api,views,components}/iot。

---

## 五、遗留问题（含页面-接口映射核对表）

### 页面-接口映射核对表（静态证据：每页前端调用 vs controller 注解逐条对齐）

**租户页**（SysTenantController @RequestMapping("/iot/tenant")）

| 前端调用（api/iot/tenant-api.ts） | 后端端点 | 权限位（sys_menu） | 对齐 |
| --- | --- | --- | --- |
| queryTenantList → GET /iot/tenant/list（pageNum/pageSize/tenantCode/tenantName/status） | @GetMapping("/list") Result\<Page\<SysTenant\>\> | iot:tenant:list 4002 | ✓ |
| queryTenantById → GET /iot/tenant/{id} | @GetMapping("/{id}") | iot:tenant:query 4003 | ✓（封装提供；页面用行数据回显未消费） |
| addTenant → POST /iot/tenant（SysTenant 校验注解全对齐） | @PostMapping @Valid | iot:tenant:add 4004 | ✓ |
| updateTenant → PUT /iot/tenant（tenant_code 禁输回传） | @PutMapping @Valid | iot:tenant:edit 4005 | ✓ |
| changeTenantStatus → PUT /iot/tenant/{id}/status {status} | @PutMapping("/{id}/status") | iot:tenant:status 4007 | ✓ |
| deleteTenant → DELETE /iot/tenant/{ids}（单/批逗号分隔） | @DeleteMapping("/{ids}") | iot:tenant:remove 4006 | ✓ |
| initTenantAdmin → POST /iot/tenant/{id}/init-admin（username/nickname/password SM4 密文） | @PostMapping("/{id}/init-admin") @Valid TenantAdminCreateForm | iot:tenant:admin 4017 | ✓ |

**字典页**（IotDataDictController @RequestMapping("/iot/dict")）

| 前端调用（dict-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryDictList → GET /iot/dict/list（dataCode/dataName/category/status） | @GetMapping("/list") | iot:dict:list 4018 | ✓ |
| queryDictById → GET /iot/dict/{id} | @GetMapping("/{id}") | iot:dict:query 4019 | ✓（封装提供；页面用行数据回显未消费） |
| addDict → POST /iot/dict（dataType/category Pattern 前置对齐） | @PostMapping @Valid | iot:dict:add 4020 | ✓ |
| updateDict → PUT /iot/dict（data_code/data_type 禁输必携带——服务层「含未携带即拒」口径） | @PutMapping @Valid | iot:dict:edit 4021 | ✓ |
| disableDict → DELETE /iot/dict/{id}（禁用语义） | @DeleteMapping("/{id}") | iot:dict:remove 4022 | ✓ |
| importDict → POST /iot/dict/import（multipart，silentError 自渲染回执） | @PostMapping("/import") @RequestParam("file") | iot:dict:import 4023 | ✓ |
| downloadImportTemplate → GET /iot/dict/import/template | @GetMapping("/import/template") | iot:dict:import 4023 | ✓ |
| downloadImportErrorReport → POST /iot/dict/import/error-report（body=回执） | @PostMapping("/import/error-report") @RequestBody DictImportReceipt | iot:dict:import 4023 | ✓ |

**模板页**（IotModelTemplateController + IotDataItemTemplateController，/iot/template、/iot/template/item）

| 前端调用（template-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryTemplateList → GET /iot/template/list（templateCode/templateName/nodeType/status） | @GetMapping("/list") | iot:template:list 4024 | ✓ |
| queryTemplateById → GET /iot/template/{id}?withItems | @GetMapping("/{id}") Result\<IotModelTemplateDetailVO\> | iot:template:query 4025 | ✓（封装提供；子表清单走 /item/list，未消费） |
| addTemplate → POST /iot/template（新增不携带 status——后端强制 1） | @PostMapping @Valid | iot:template:add 4026 | ✓ |
| updateTemplate → PUT /iot/template（template_code 禁输） | @PutMapping @Valid | iot:template:edit 4027 | ✓ |
| deleteTemplates → DELETE /iot/template/{ids} | @DeleteMapping("/{ids}") | iot:template:remove 4028 | ✓ |
| queryTemplateItems → GET /iot/template/item/list?templateId | @GetMapping("/list")（查询共用 iot:template:list） | iot:template:list 4024 | ✓ |
| addTemplateItems → POST /iot/template/item {templateId, items[]}（批量 ≤500） | @PostMapping @Valid IotTemplateItemBatchForm | iot:template:item:edit 4029 | ✓ |
| updateTemplateItem → PUT /iot/template/item | @PutMapping @Valid | iot:template:item:edit 4029 | ✓ |
| deleteTemplateItems → DELETE /iot/template/item/{ids} | @DeleteMapping("/{ids}") | iot:template:item:edit 4029 | ✓ |

**实例页**（IotModelInstanceController，/iot/instance）

| 前端调用（instance-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryInstanceTree → GET /iot/instance/tree | @GetMapping("/tree") Result\<List\<IotInstanceTreeVO\>\> | iot:instance:tree 4035 | ✓ |
| queryInstanceList → GET /iot/instance/list（instanceCode/instanceName 模糊、nodeType/status 精确） | @GetMapping("/list") | iot:instance:list 4030 | ✓ |
| queryInstanceById → GET /iot/instance/{id}（详情弹窗） | @GetMapping("/{id}") | iot:instance:query 4031 | ✓ |
| addInstance → POST /iot/instance（只传 shortCode；parentId 空→0；模板/部门必填联动=服务层口径前置） | @PostMapping @Valid IotInstanceCreateForm | iot:instance:add 4032 | ✓ |
| updateInstance → PUT /iot/instance（五字段一致性快照回传；locked=1 仅携带 status） | @PutMapping @Valid IotInstanceUpdateForm | iot:instance:edit 4033 | ✓ |
| deleteInstances → DELETE /iot/instance/{ids} | @DeleteMapping("/{ids}") | iot:instance:remove 4034 | ✓ |

**实例数据项/参数覆盖**（IotDataItemController，无类级前缀）

| 前端调用（item-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryItemList → GET /iot/item/list?instanceId | @GetMapping("/iot/item/list") | iot:item:list 4036 | ✓ |
| regenerateItems → POST /iot/instance/{instanceId}/items/regenerate | @PostMapping | iot:item:add 4038 | ✓ |
| overrideItemParams → PUT /iot/instance/{instanceId}/items/{itemId}/params（四字段，空→null=清除覆盖） | @PutMapping @RequestBody IotItemParamOverrideForm | iot:item:edit 4039 | ✓ |
| queryEffectiveParams → GET /iot/instance/{instanceId}/items/{itemId}/effective-params | @GetMapping | iot:item:query 4037 | ✓ |
| （无删除调用） | 后端 remove 为预留行（绑定/删除归 0023/0015） | iot:item:remove 4040 | ✓ 前端不消费（无接口） |

**网关页**（IotGatewayController，/iot/gateway）

| 前端调用（gateway-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryGatewayList → GET /iot/gateway/list（gatewayCode/gatewayName/status/onlineStatus） | @GetMapping("/list") | iot:gateway:list 4041 | ✓ |
| （无 GET /{id} 调用） | 后端不设 detail 端点（0017 口径） | iot:gateway:query 4042（预登记） | ✓ 详情弹层用列表行数据渲染，封装不提供不存在接口 |
| addGateway → POST /iot/gateway（不携带 status——新建一律启用） | @PostMapping @Valid IotGatewayCreateForm | iot:gateway:add 4043 | ✓ |
| updateGateway → PUT /iot/gateway（gateway_code 禁输回传；可变白名单五字段） | @PutMapping @Valid IotGatewayUpdateForm | iot:gateway:edit 4044 | ✓ |
| deleteGateway → DELETE /iot/gateway/{id} | @DeleteMapping("/{id}") | iot:gateway:remove 4045 | ✓ |

**通道页**（IotChannelController，/iot/channel）

| 前端调用（channel-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryChannelList → GET /iot/channel/list（gatewayId 必传——未选网关不发请求） | @GetMapping("/list")（gatewayId 必传 PARAM_ERROR） | iot:channel:list 4046 | ✓ |
| queryChannelById → GET /iot/channel/{id}（详情/编辑回显 conn_config 原文） | @GetMapping("/{id}") | iot:channel:query 4047 | ✓ |
| addChannel → POST /iot/channel（connConfig 以 JSON 对象承载=JsonNode 直收；MODBUS/OPCUA 不携带 status） | @PostMapping @Valid IotChannelCreateForm | iot:channel:add 4048 | ✓ |
| updateChannel → PUT /iot/channel（channel_code/gateway_id 禁输回传；conn_config 变更才携带） | @PutMapping @Valid IotChannelUpdateForm | iot:channel:edit 4049 | ✓ |
| deleteChannel → DELETE /iot/channel/{id} | @DeleteMapping("/{id}") | iot:channel:remove 4050 | ✓ |

**页面内跨域消费（合法联动，非 iot 新端点）**

| 页面 | 消费 | 说明 |
| --- | --- | --- |
| instance-form-modal / gateway-form-modal | deptApi.queryDeptTree → GET /dept/tree | 绑部门树选择（框架接口） |
| instance-form-modal | templateApi.queryTemplateList（按 nodeType 联动） | 选模板 TD-0008 |
| template-item-modal | dictApi.queryDictList（status=1，pageSize=500） | 字典选码下拉（ADR-0003 唯一来源） |
| channel/index、channel-form-modal | gatewayApi.queryGatewayList | 网关下拉（gatewayId 必传前置） |
| tenant-admin-modal | encryptData（/@/lib/encrypt SM4） | password 前端密文（TenantAdminCreateForm 口径） |

### 遗留问题清单

1. **运行期联调未做**（本轮口径内）：上表为静态正确性证据；集成轮（0068 或专项）需按表逐条冒烟，重点验证：字典导入整批拒绝回执的 body 结构（code=400 + data=receipt）、通道预留协议成功文案（message 含「未启用适配器」）、实例参数覆盖 null=清除语义、实例树数据权限裁剪展示。
2. **字典选码下拉 500 上限边界**：字典条目超 500 时下拉不全（后端 maxLimit=500 硬上限）。集成轮评估改为远程搜索加载（show-search + 后端模糊查询）。
3. **iot:item:remove（4040）/iot:gateway:query（4042）为预留权限行**：后端分别无删除端点/详情端点——前端未封装未消费（网关详情用行数据、数据项删除无入口），0021/0023 交付对应接口时补齐。
4. **状态规则入口权限位待替换**：现绑 iot:template:edit（偏离项 1）——TASK-0048 交付独立权限位与菜单种子后替换绑定并实体化配置界面。
5. **乐观锁提示开关**：`IOT_OPTIMISTIC_LOCK_ENABLED=false`（iot 域无 @Version）——任一 iot 实体引入 @Version 后置 true 即全页生效，无需改页面代码。
6. **lint 基线 9 条 prettier warning**（tests/unit/** 既有文件）：不属于本任务改动范围，未修（改 tests/ 会违反改动范围契约）——可在后续工程化任务统一 `lint:fix`。
7. **workspace 未跟踪的 `.agents/` 目录**：实施前已存在（本任务未创建、未触碰、未提交）。

---

## 修订轮（§13 退回一轮：评审 I-1 修复 + 同类错位清查）

- 评审报告：`TASK-0056-review.md`（Critical 0 / Important 1 / Minor 9）
- 修订提交：`11e200c`（main，注明 TASK-0056 修订轮，3 文件 +10 −4，全部在 `frontend/src/views/iot/**` 声明范围内）
- 修复内容（统一事件名取多数既有约定 **reloadList**——六域 8 个 emit 组件全部归一）：
  1. **I-1**：`dict-import-modal.vue` `defineEmits(['reload'])`→`['reloadList']`，两处 `emit('reload')`→`emit('reloadList')`（导入成功与 close 的 failCount===0 分支两处；父页 dict/index.vue `@reloadList="ajaxQuery"` 监听不变）——导入成功后列表恢复自动刷新。
  2. **同类错位 ①**：`instance-params-modal.vue` **原先未声明、未触发任何事件**（抽屉监听落空，参数覆盖保存后清单不刷新）——补 `defineEmits(['reloadList'])` 并在覆盖保存成功后 emit。
  3. **同类错位 ②**：`instance-item-drawer.vue` 对 params-modal 的监听 `@reload`→`@reloadList`，与子组件两侧统一。
- Minor M-1~M-9 按裁决**不动记 deferred**（含 M-4 中 template-item-modal `emit('reloadList')` 无父级监听一项——属「emit 无消费者」类，非两侧命名错位，维持 Minor 处置）；M-1 勘误（§一计数应 13 新 + 7 改）随本节记录在案。

### 修订轮补充验收 1：三条静态门（原始输出）

```text
$ cd frontend && pnpm build:test
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
transforming...
...（chunk 产物清单中略，基线既有 icon.svg / chunk>500kB 两条 warning 同前）
✓ built in 1m 28s
BUILD_EXIT=0
```

```text
$ cd frontend && pnpm lint
$ eslint .
（9 条 warning 全部同首轮——tests/unit/** 基线既有 prettier 告警，文件与行号逐一相同，本修订未触碰 tests/）
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.
LINT_EXIT=0
```

```text
$ cd frontend && pnpm typecheck
$ vue-tsc --noEmit
TYPECHECK_EXIT=0
```

### 修订轮补充验收 2：emit-监听配对核对清单（grep 全量，逐条对齐结果）

grep 口径：emit 侧 `grep -rn "defineEmits|emit('reload" src/views/iot --include="*.vue"`；监听侧 `grep -rn "@reloadList|@reload" src/views/iot --include="*.vue"`。修订后 **六域弹窗/抽屉刷新事件统一 `reloadList`**，8 个 emit 组件 × 8 个监听位全部对齐：

| # | 子组件（emit 侧） | defineEmits / emit 触发 | 父页监听侧 | 对齐结果 |
| --- | --- | --- | --- | --- |
| 1 | tenant-form-modal | `['reloadList']` ×1 | tenant/index.vue L112 `@reloadList="ajaxQuery"` | ✓ 对齐 |
| 2 | tenant-admin-modal | 无 emit（初始化管理员不影响列表数据） | tenant/index.vue 无监听 | ✓ 无错位 |
| 3 | dict-form-modal | `['reloadList']` ×1 | dict/index.vue L102 `@reloadList="ajaxQuery"` | ✓ 对齐 |
| 4 | **dict-import-modal** | 修前 `['reload']` ×2 → 修后 `['reloadList']` ×2（L126 成功 / L160 close 分支） | dict/index.vue L103 `@reloadList="ajaxQuery"` | **✓ 修复（I-1）** |
| 5 | template-form-modal | `['reloadList']` ×1 | template/index.vue L119 `@reloadList="ajaxQuery"` | ✓ 对齐 |
| 6 | template-item-modal | `['reloadList']` ×1（updateItem 后） | template/index.vue 未监听（弹窗经 ref 打开，清单弹窗内部自刷新） | emit 无监听 = 评审 M-4 组成部分，按 Minor 裁决**不动记 deferred** |
| 7 | instance-form-modal | `['reloadList']` ×1 | instance/index.vue L156 `@reloadList="reloadAll"` | ✓ 对齐 |
| 8 | instance-detail-modal | 无 emit | instance/index.vue 无监听 | ✓ 无错位 |
| 9 | instance-item-drawer | 无 emit（自身无刷新事件） | instance/index.vue 无监听 | ✓ 无错位 |
| 10 | **instance-params-modal** | 修前无声明无触发 → 修后 `['reloadList']` ×1（L153 保存成功后） | instance-item-drawer.vue L48 `@reload`→修后 `@reloadList="ajaxQueryItems"` | **✓ 修复（同类错位，两侧统一）** |
| 11 | gateway-form-modal | `['reloadList']` ×1 | gateway/index.vue L106 `@reloadList="ajaxQuery"` | ✓ 对齐 |
| 12 | gateway-detail-modal | 无 emit | gateway/index.vue 无监听 | ✓ 无错位 |
| 13 | channel-form-modal | `['reloadList']` ×1 | channel/index.vue L108 `@reloadList="ajaxQuery"` | ✓ 对齐 |
| 14 | channel-detail-modal | 无 emit | channel/index.vue 无监听 | ✓ 无错位 |

grep 修订后复核（emit 侧，10 处声明 / 12 处触发全部 reloadList）：

```text
src/views/iot/channel/components/channel-form-modal.vue:67:  const emit = defineEmits(['reloadList']);
src/views/iot/channel/components/channel-form-modal.vue:185:      emit('reloadList');
src/views/iot/dict/components/dict-form-modal.vue:50:  const emit = defineEmits(['reloadList']);
src/views/iot/dict/components/dict-form-modal.vue:112:          emit('reloadList');
src/views/iot/dict/components/dict-import-modal.vue:90:  const emit = defineEmits(['reloadList']);
src/views/iot/dict/components/dict-import-modal.vue:126:      emit('reloadList');
src/views/iot/dict/components/dict-import-modal.vue:160:      emit('reloadList');
src/views/iot/gateway/components/gateway-form-modal.vue:75:  const emit = defineEmits(['reloadList']);
src/views/iot/gateway/components/gateway-form-modal.vue:152:          emit('reloadList');
src/views/iot/instance/components/instance-form-modal.vue:95:  const emit = defineEmits(['reloadList']);
src/views/iot/instance/components/instance-form-modal.vue:251:          emit('reloadList');
src/views/iot/instance/components/instance-params-modal.vue:77:  const emit = defineEmits(['reloadList']);
src/views/iot/instance/components/instance-params-modal.vue:153:      emit('reloadList');
src/views/iot/template/components/template-item-modal.vue:170:  const emit = defineEmits(['reloadList']);
src/views/iot/template/components/template-item-modal.vue:351:      emit('reloadList');
src/views/iot/template/components/template-form-modal.vue:55:  const emit = defineEmits(['reloadList']);
src/views/iot/template/components/template-form-modal.vue:117:          emit('reloadList');
src/views/iot/tenant/components/tenant-form-modal.vue:56:  const emit = defineEmits(['reloadList']);
src/views/iot/tenant/components/tenant-form-modal.vue:117:          emit('reloadList');
```

监听侧（8 处 @reloadList，无 @reload 残留）：

```text
src/views/iot/channel/index.vue:108:    <ChannelFormModal ref="channelFormModalRef" :gateway-options="gatewayOptions" @reloadList="ajaxQuery" />
src/views/iot/gateway/index.vue:106:    <GatewayFormModal ref="gatewayFormModalRef" @reloadList="ajaxQuery" />
src/views/iot/dict/index.vue:102:    <DictFormModal ref="dictFormModalRef" @reloadList="ajaxQuery" />
src/views/iot/dict/index.vue:103:    <DictImportModal ref="dictImportModalRef" @reloadList="ajaxQuery" />
src/views/iot/instance/index.vue:156:  <InstanceFormModal ref="instanceFormModalRef" @reloadList="reloadAll" />
src/views/iot/instance/components/instance-item-drawer.vue:48:    <InstanceParamsModal ref="instanceParamsModalRef" @reloadList="ajaxQueryItems" />
src/views/iot/template/index.vue:119:    <TemplateFormModal ref="templateFormModalRef" @reloadList="ajaxQuery" />
src/views/iot/tenant/index.vue:112:    <TenantFormModal ref="tenantFormModalRef" @reloadList="ajaxQuery" />
```

### 修订轮改动范围核对

```text
$ git diff --name-only 424031b..11e200c（修订提交自身）
frontend/src/views/iot/dict/components/dict-import-modal.vue
frontend/src/views/iot/instance/components/instance-item-drawer.vue
frontend/src/views/iot/instance/components/instance-params-modal.vue
```

3/3 全在 `views/iot` 声明范围内。注：`fa12df9..HEAD` 全区间现含并行 backend 道（0019 凭据域）在 main 的独立提交，非本任务产物；本任务两笔提交（`424031b` 30 文件 + `11e200c` 3 文件）合计 33 文件均在 `frontend/src/{api,views,components}/iot` 内。

### 修订轮未动项

- Minor M-1~M-9 全部按裁决不动记 deferred（M-2/M-9 建议并入遗留 2/集成轮清单跟踪；M-1 勘误随本节在案）。
- 契约不变：未碰 backend/、ingest/、workitems 库其他文件；TASK 状态未改（仍由协调者收尾）。
