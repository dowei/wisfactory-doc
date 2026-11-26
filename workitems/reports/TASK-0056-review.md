# TASK-0056 diff 评审报告（管理页第一组——租户/字典/模板/实例/网关/通道 CRUD 页）

- 评审者：fresh reviewer（不共享协调者上下文；no-subagents；只评审不修改）
- 评审对象：单提交 `424031b`（BASE `fa12df9`..HEAD），30 文件 / +4278 −48，与实施报告 §四 文件清单逐条一致，全部落在 brief 声明范围（`frontend/src/{views,api,components}/iot/**`），无越界
- 输入：评审 brief + 实施 brief + TASK-0056 详情 + 实施报告 + diff 包（全文通读）+ 后端七控制器 / DTO / Schema / 菜单种子 SQL 现场核对
- 依据：`L1-execute.md` §13（契约一致性 + 明显质量缺陷，分级 Critical / Important / Minor）

---

## 一、结论

**Critical 0 / Important 1 / Minor 9。**

六域页面-接口映射核对表与 controller 实际注解逐条吻合（本评审未抽样、全量核对七控制器全部端点），权限位与菜单种子逐一对齐，表单校验与 DTO 注解逐条对齐，brief 八项契约点全部落实；质量面发现 1 处功能缺陷（字典导入成功后列表不刷新——事件名错位）与 9 处 Minor。按 §13：Important 进退回路径（scoped 修复：一处 emit 名或监听名二选一改齐 + 复评）；Minor 记 TASK `任务结果` deferred，不进修订轮。

---

## 二、发现清单

### Critical（0）

无。

### Important（1）

| # | 出处 | 说明 |
| --- | --- | --- |
| I-1 | `frontend/src/views/iot/dict/components/dict-import-modal.vue`（L89 `defineEmits(['reload'])`，L125/L159 两处 `emit('reload')`）vs `frontend/src/views/iot/dict/index.vue` L103（`<DictImportModal ref="dictImportModalRef" @reloadList="ajaxQuery" />`） | 子组件声明并触发的事件名是 `reload`，父页监听的是 `reloadList`——导入成功（含 close 时 failCount===0 分支）的刷新事件全部落空，导入后表格停留旧数据，用户须手动点「查询」才能看到结果，易误判导入失败而重复导入；typecheck/lint 均不拦截此类错位（对照：dict-form-modal、其余五个表单弹窗的 `reloadList` 链路均正确，仅导入弹窗错位）。 |

### Minor（9）

| # | 出处 | 说明 |
| --- | --- | --- |
| M-1 | `TASK-0056-report.md` §一「views/iot（15 新 6 改）」 | 报告文件计数自相矛盾：实际 13 新 + 7 改（tenant-form-modal 是修改、占位两页不在 diff），§四 30 文件清单本身正确——不影响验收，建议收尾时勘误。 |
| M-2 | `frontend/src/components/iot/iot-conn-config-editor.vue`（a-textarea 无 maxlength、basicFormatCheck 无长度分支） | conn_config 前置提示缺 8192 字符上限镜像（后端 `ChannelConnConfigValidator.MAX_CONN_CONFIG_LENGTH=8192` 硬拒）：超长文本要到提交后才被后端 PARAM_ERROR 拒绝，brief 质量项「超大文本」的前置纠错缺位（fail-closed 安全无风险）。 |
| M-3 | `dict/components/dict-form-modal.vue`、`template/components/template-form-modal.vue`、`gateway/components/gateway-form-modal.vue`、`template/components/template-item-modal.vue`（itemForm） | 回显统一 `Object.assign(form, rowData/detail)`、reset 用 `Object.assign(formDefault)`（不删多余键）→ 编辑态整行回传（createTime/deleted/tenantId 等随 payload 上行），且「编辑已落库数据项→新增待保存行」会把上一行 `id` 带进批量保存 payload。已逐一核对六域后端写路径全部字段白名单（`updateDict`/`updateTemplate`/`saveItems.buildEntity`/各 UpdateForm），当前无数据影响，属「类型与 DTO 对齐」卫生问题与后端契约演进隐患。 |
| M-4 | `components/iot/iot-consts.ts`（`IOT_PAGE_MAX`/`IOT_STATUS_ENABLE`/`IOT_STATUS_DISABLE`）、`channel/components/channel-detail-modal.vue`、`instance/components/instance-detail-modal.vue`、`instance/components/instance-params-modal.vue`（a-descriptions `:loading`）、`instance/components/instance-form-modal.vue`（`submitLoading`）、`template/components/template-item-modal.vue`（updateItem 的 `emit('reloadList')` 父级未监听） | 一批无效/死代码：三个常量全仓零引用；antd Descriptions 无 loading prop（:loading 为 no-op，详情弹窗加载期无反馈）；submitLoading 定义未绑定；一处无消费者的 emit。 |
| M-5 | `frontend/src/views/iot/instance/index.vue`（onSelectTree、placeholder「含前缀检索子树」） | 树节点点击过滤文案称「按编码前缀检索」，后端实际是 `like %code%` 包含匹配（`IotModelInstanceServiceImpl` L612，SqlLikeUtils.escape）——点 F01 会命中 F010 等以其为子串的兄弟子树，结果为超集（展示过滤精度问题，非数据错误）。 |
| M-6 | `frontend/src/views/iot/channel/components/channel-form-modal.vue`（onSubmit 固定 `message.success('添加/修改成功')`） | 后端对 MODBUS/OPCUA 返回定制成功文案「通道保存成功：该协议一期未启用适配器…」，前端丢弃 response.message 不透传（表单内静态 alert 已有前置提示，实施报告遗留 1 已登记集成轮验证该文案）。 |
| M-7 | `frontend/src/views/iot/channel/components/channel-form-modal.vue`（create 非预留协议 status 单选初始 undefined、无默认选中、无必填规则） | 不触摸状态直接提交则不携带 status→后端落库 1（启用），行为正确但单选组初始无高亮，与 template-form-modal（formDefault.status=1）不一致，属 UX 一致性。 |
| M-8 | `frontend/src/views/iot/instance/components/instance-params-modal.vue`（mode='view'） | 「有效值」（view）入口打开后四个输入框仍可编辑、OK 文案仍为「保存」，点击仅关闭不落库——view 模式应禁输或改文案，易误导。 |
| M-9 | `channel/index.vue`（loadGatewayOptions pageSize=500）、`instance/components/instance-form-modal.vue`（loadTemplateOptions pageSize=500）、`template/components/template-item-modal.vue`（loadDictOptions pageSize=500） | 三个下拉同受 maxLimit=500 截断限制，实施报告遗留 2 只登记了字典一处——网关/模板条目超 500 时下拉同样不全，建议与字典一并进集成轮评估远程搜索。 |

---

## 三、契约一致性核对证据（brief「依据①」逐项）

### 1. 页面-接口映射核对表 vs controller 实际注解（全量，超出「六域各两接口」抽样要求）

| 域 | controller（现场读取） | 核对结果 |
| --- | --- | --- |
| 租户 | `SysTenantController` @RequestMapping("/iot/tenant")：GET /list、GET /{id}、POST、PUT、PUT /{id}/status、DELETE /{ids}、POST /{id}/init-admin（@Valid TenantAdminCreateForm） | 与 tenant-api.ts 7 封装一一对应 ✓ |
| 字典 | `IotDataDictController` @RequestMapping("/iot/dict")：GET /list、GET /{id}、POST、PUT、DELETE /{id}（=禁用）、POST /import（@RequestParam("file")）、GET /import/template、POST /import/error-report（@RequestBody DictImportReceipt） | 与 dict-api.ts 8 封装一一对应 ✓ |
| 模板 | `IotModelTemplateController` /iot/template：GET /list、GET /{id}?withItems、POST、PUT、DELETE /{ids}；`IotDataItemTemplateController` /iot/template/item：GET /list?templateId、POST（IotTemplateItemBatchForm）、PUT、DELETE /{ids} | 与 template-api.ts 9 封装一一对应 ✓ |
| 实例 | `IotModelInstanceController` /iot/instance：GET /tree、GET /list、GET /{id}、POST（IotInstanceCreateForm）、PUT（IotInstanceUpdateForm）、DELETE /{ids} | 与 instance-api.ts 6 封装一一对应 ✓ |
| 数据项 | `IotDataItemController`（无类级前缀）：GET /iot/item/list?instanceId、POST /iot/instance/{instanceId}/items/regenerate、PUT /iot/instance/{instanceId}/items/{itemId}/params（IotItemParamOverrideForm）、GET …/effective-params | 与 item-api.ts 4 封装一一对应 ✓ |
| 网关 | `IotGatewayController` /iot/gateway：GET /list、POST（IotGatewayCreateForm）、PUT（IotGatewayUpdateForm）、DELETE /{id}；**无 GET /{id}** | gateway-api.ts 正确不封装 detail（详情用行数据）✓ |
| 通道 | `IotChannelController` /iot/channel：GET /list（gatewayId 必传）、GET /{id}、POST、PUT、DELETE /{id} | 与 channel-api.ts 5 封装一一对应 ✓ |

响应形状核对：`IotItemEffectiveParamsVO`（validMin/Max、alarmLow/High + 各 *Source 三态）、`IotInstanceTreeVO`（id/instanceCode/instanceName/status/children）、`IotGateway`（onlineStatus/lastHeartbeatTime/version/configVersion/ipAddr 五字段组）、`IotChannel.connConfig` 为 String（详情弹窗 `JSON.parse` 美化与表单回显口径正确）、`DictImportReceipt`（rows/rowIndex/dataCode/reasonCode/message/success）——前端消费字段全部存在 ✓。分页 `PAGE_SIZE_OPTIONS` 上限 500 = 后端 `MybatisPlusConfig` maxLimit ✓。

### 2. v-privilege 权限位 vs 菜单种子（resources/sql/upgrade/*.sql 逐行）

4002-4007（iot:tenant:*）、4017（iot:tenant:admin）、4018-4023（iot:dict:* 含 import）、4024-4029（iot:template:* 含 item:edit）、4030-4035（iot:instance:* 含 tree）、4036-4040（iot:item:*，remove 4040 预留未消费）、4041-4045（iot:gateway:*，query 4042 预留被详情按钮消费）、4046-4050（iot:channel:*）——diff 内全部 38 处 v-privilege 绑定与种子行一一对应，无悬空位、无越权位 ✓。0055 租户页权限显隐骨架未破坏（行操作 edit/status/remove 原绑定保留，仅查询/重置补挂 iot:tenant:list）✓。

### 3. 表单校验 vs DTO 注解（逐条）

- 租户 = `SysTenant`（code ≤32 + `^[A-Za-z0-9_]+$`、name ≤100、contactName ≤50、contactPhone ≤32、remark ≤500）✓
- 租户管理员 = `TenantAdminCreateForm`（username/nickname ≤50 必填、password 必填；SM4 encryptData 与 system/user 创建同款已核）✓
- 字典 = `IotDataDict`（dataCode ≤64 必填、dataName ≤100、dataType 四值、category 七值 ≤32、defaultUnit ≤32；编辑 data_code/data_type 禁输+必携带，服务层「未携带即拒」口径对齐）✓
- 模板 = `IotModelTemplate`（code ≤64、name ≤100、nodeType 1-5、version ≥1、remark ≤500；新增不携带 status）✓；数据项 = `IotDataItemTemplate`（含 defaultFormula ≤2000 一期降级口径、批量 ≤500 = `ITEM_BATCH_MAX`）✓
- 实例 = `IotInstanceCreateForm`（shortCode `^[A-Za-z0-9_]+$`、name ≤100、nodeType 1-5；设备/组件 templateId 必填、工厂/车间 deptId 必填的条件 validator 与服务层口径一致；编辑五字段一致性快照回传、locked=1 仅携带 status）✓
- 网关 = `IotGatewayCreateForm/UpdateForm`（code ≤64 + 字符集、name ≤100、心跳周期 1-86400、白名单五字段、新建不携带 status）✓
- 通道 = `IotChannelCreateForm/UpdateForm`（code ≤64、name ≤100、protocolType ≤16 四值、gatewayId 必传前置拦截、编辑 code/gateway_id 禁输、conn_config 未变更不携带=null 不进更新集）✓
- conn_config 编辑器镜像 = `ChannelConnConfigValidator` + mqtt/http-v1.json：8 个凭据关键词逐字一致（全深度递归含数组元素）、required（MQTT broker/topic、HTTP reportPath）、reportPath `^/` 前缀、MODBUS/OPCUA 无 required ✓

### 4. 其余契约点

- 停用/删除二次确认：六域全部 Modal.confirm（字典禁用文案含「不可恢复启用，编码永不复用」、租户删除含「编码不复用」、批量删除均确认）✓
- 乐观锁预留登记：`iot-optimistic-lock.ts` `IOT_OPTIMISTIC_LOCK_ENABLED=false` 登记不触发，全部写操作 catch 接入 `handleIotConflictError` ✓
- 点位/指标占位维持：point/metric 不在 diff ✓
- 状态规则入口占位：template 页占位按钮 + 点击提示待 TASK-0048；v-privilege 暂绑 iot:template:edit 为已登记偏离（实施报告偏离 1/遗留 4），可接受 ✓
- 三条静态门：实施报告贴原始输出 EXIT=0（build:test / lint 0 error / typecheck），本评审未复跑（只评审不修改契约）；中途 4 个 TS 错误的收敛方式（TenantRow 行接口、as number）在 diff 中可见 ✓
- 后端联调不在本轮：未起 8098，映射表即静态证据，口径符合 brief 决策 2 ✓

## 四、质量面核对证据（brief「依据②」）

- **响应解包一致性**：拦截器 resolve 响应体、code!==200 reject 整个 AxiosResponse（`/@/lib/axios.ts`）——各页 `responseData.data.records/total` 消费口径统一；错误统一 catch + `otSentry.captureError`；字典导入 silentError 自渲染回执取 `e.data.data` 与拦截器契约吻合 ✓；网络类错误由拦截器全局提示，无双重弹错。
- **conn_config 编辑器安全**：JSON.parse（无 eval）、提交前 `JSON.parse` 失败有 SyntaxError 兜底、凭据键深度扫描与后端逐字镜像、必含键/前缀校验一致；缺口仅 8192 长度（M-2）。
- **内存泄漏**：弹窗/抽屉常驻挂载仅靠 :open 切换，close 均清空 detail/tableData/form；无新增定时器/监听器；下载走封装（URL.revokeObjectURL 在框架层）——未发现泄漏。gateway 表单每次打开重拉实例树/部门树为性能小疵非泄漏。
- **组件复用度**：枚举/文案（iot-consts）、连接配置编辑器、乐观锁提示三件公共件已抽出；六个列表页的查询/表格/分页/确认删除脚手架为逐页复制——与框架既有 system/monitor 页范式一致（对照 system/dict 同构），非本批引入的偏离。
- **错误处理边界**：channel 编辑时详情接口失败降级用行数据回显、gatewayId 未选不发请求、instanceCode 空/JSON 非法各有前置拦截；M-5/M-7/M-8 为残留边界瑕疵（见清单）。

## 五、处置建议

- I-1 → §12 退回路径（scoped）：`dict-import-modal.vue` 的 emits/emit 改 `reloadList`（或父页改听 `reload`）一处改动 + 复评仅看该 diff。
- M-1~M-9 → 记 TASK `任务结果` deferred（收尾进 LOG）；其中 M-2/M-9 建议并入实施报告遗留 2/集成轮清单一并跟踪。
