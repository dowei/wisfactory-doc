# 评审报告：TASK-0097 点位管理页前端补齐

- 角色：fresh 代码评审代理（不共享协调者上下文；仅依据评审 brief 指定路径与 `.agents/protocol/L1-execute.md` §13 diff 评审口径）
- 评审输入：
  - 评审 brief：`reports/TASK-0097-review-brief.md`
  - TASK 详情：`tasks/TASK-0097.md`
  - 实施 brief：`reports/TASK-0097-brief.md`
  - diff 包：`reports/TASK-0097-diff.txt`（BASE `4e4c1461` → HEAD `449d7eb3`，7 个前端文件）
  - 实施报告：`reports/TASK-0097-report.md`
  - 后端事实源：`IotPointController.java` 及 `dto/`（CreateForm / UpdateForm / StatusForm / DetailVO / CurveVO / PointCurvePointVO / PointReferenceVO / StatusChangeVO）、`entity/IotPoint.java`
  - 协议：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 结论：**无 Critical / Important / Minor 发现**。
- 说明：以下「核查结论」为逐项核对记录，不作为发现；判断依据不足处标注为「无法判定」。

## 1 越界反查（独立复核）

- `git diff --name-only 4e4c1461..449d7eb3`（在 `D:\workspace\WisFactory` 执行）结果为 7 个文件：
  - `frontend/src/api/iot/point-api.ts`
  - `frontend/src/components/iot/iot-consts.ts`
  - `frontend/src/views/iot/point/index.vue`
  - `frontend/src/views/iot/point/components/point-form-modal.vue`
  - `frontend/src/views/iot/point/components/point-detail-modal.vue`
  - `frontend/tests/unit/api/iot/point-api.spec.ts`
  - `frontend/tests/unit/views/iot/point.spec.ts`
- 与实施 brief §改动范围声明逐项比对：全部落在声明集内；无 backend / calc / ingest / contract / metric 页改动。diff 包正文中的 hunk 也仅涉及上述 7 个文件，未发现声明集之外的越界修改。

## 2 发现清单

无 Critical / Important / Minor 发现。

## 3 评审重点逐条核查（brief 第 36–43 行）

### 3.1 point-api 端点/动词/参数与 IotPointController 对齐（重点 1）

diff 行 30–88（`frontend/src/api/iot/point-api.ts`）：

| 封装 | diff 行 | 调用 | Controller 事实源 | 一致性 |
| --- | --- | --- | --- | --- |
| queryList | 40–42 | `getRequest('/iot/point/list', params)` | `@GetMapping("/list") list(pageNum, pageSize, IotPoint query)` | 一致；`pageNum/pageSize` 与 DTO 字段同在 params 透传 |
| getById / detail | 45–52 | `getRequest('/iot/point/${id}', undefined)` | `@GetMapping("/{id}") detail(Long id)` | 一致；第二参 `undefined` 匹配 `getRequest(url, params, config?)` 签名 |
| references | 55–57 | `getRequest('/iot/point/${id}/references', undefined)` | `@GetMapping("/{id}/references")` | 一致 |
| curve | 60–62 | `getRequest('/iot/point/${id}/curve', undefined)` | `@GetMapping("/{id}/curve")` | 一致 |
| add | 65–67 | `postRequest('/iot/point', form)` | `@PostMapping` `@SaCheckPermission("iot:point:add")` | 一致 |
| update | 70–72 | `putRequest('/iot/point', form)` | `@PutMapping` `@SaCheckPermission("iot:point:edit")` | 一致 |
| changeStatus | 75–77 | `putRequest('/iot/point/changeStatus', form)` | `@PutMapping({"/changeStatus", "/status"})`，主路径即 `/changeStatus` | 一致（未用别名） |
| remove | 80–83 | `deleteRequest('/iot/point/${idsParam}')`，数组 `join(',')` | `@DeleteMapping("/{ids}")` `List<Long> ids` | 一致；Spring 对 `@PathVariable List<Long>` 的逗号分隔绑定是既有契约，brief 明确指定逗号拼接 |
| export | 86–88 | `getDownload('/iot/point/export', params)` | `@GetMapping("/export")` `void export(IotPoint query, response)` | 一致；`getDownload(url, params?)` 支持 GET 下载、内部完成保存与错误提示 |
| 导入三方法 | 90+ 保留 | 原有 `getDownload/postRequest/postDownload` | `@GetMapping("/import/template")`、`@PostMapping("/import")`、`@PostMapping("/import/error-report")` | 保留未改，单测覆盖仍通过 |

- `silentError`：本 diff 未对 point 的 CRUD / 详情 / 导出方法使用 `silentError`；仅导入方法沿用原有 `silentError: true`（多处业务错误需逐行回执自定义处理，合理）。导入方法调用形态与 diff 前一致，单测断言（diff 行 865–869）也匹配。
- 安全：全 diff 未出现 appKey / secret / HMAC / `/openapi/v1/**` / 自造代理，全部走 `/iot/point` + `/@/lib/axios` 管理面登录态。符合 brief 安全红线。

### 3.2 表单校验与后端 DTO 一致性、pointCode 编辑锁定、status 不进提交（重点 2）

diff 行 256–453（`point-form-modal.vue`）、101–116（`iot-consts.ts`）：

- 必填与长度：`pointCode` 必填 ≤96、`pointName` 必填 ≤100、`unit` ≤32、`protocolAddr` ≤8192、`remark` ≤500 —— 与 `IotPointCreateForm` / `IotPointUpdateForm` 注解逐条一致。
- `pointCode` 格式 `/^[A-Za-z0-9_.]+$/`：与后端服务层单点建档 `validatePointCode`（仅非空/≤96，格式未单点强制）相比，前端更严；与导入侧 `POINT_CODE_PATTERN = [A-Za-z0-9_.]+` 及导入模板说明行一致，且后端 DTO 注释与模板文案均声明该格式。前端前置收紧不构成与 DTO 冲突（后端仍逐字段校验为准）；未发现错误拒绝合法值的情形（合法锚点即字母/数字/下划线/点）。
- `dataType`：`type:'number' min:1 max:4`，选项 1 瞬时/2 累计/3 状态/4 开关 —— 与 `@Min(1) @Max(4)` 及 `DATA_TYPE_VALUES = Set.of(1,2,3,4)` 一致。
- `valueType`：`/^(DOUBLE|INT|BOOL|ENUM)$/` —— 与 `@Pattern("DOUBLE|INT|BOOL|ENUM")` 及 `VALUE_TYPE_VALUES` 一致。
- `min <= max`：自定义 validator（diff 行 378–388）—— 与后端 `validateRange` 一致。
- `samplePeriod` 1–86400、`deadband` 可空 —— 与 DTO 一致。
- 编辑锁 `pointCode`：模板 `:disabled="!!form.id"`（diff 行 274）；`showModal(row)` 回填 `id` 与 `pointCode` 同值（diff 行 396–413）；与后端「携带同值 no-op 不拦、不一致拒绝」口径一致，单测覆盖（diff 行 1108–1116）。
- `status` 不进提交：表单默认与回填字段集均无 `status`（diff 行 336–352、397–413），payload 用 `{ ...form }` 并删除 `id`（新增时）；单测断言 payload 无 `status`（diff 行 1082–1105）。与后端 `IotPointUpdateForm` 注释「status 不进编辑集」一致。
- 关于 payload 携带空值字段：`formDefault` 中 `minValue/maxValue/samplePeriod/deadband/channelId/boundInstanceId/deptId` 为 `undefined`，`unit/protocolAddr/remark` 为空串。后端 `updatePoint` 走 MyBatis-Plus `updateById`，配 `@TableField(UPDATE_STRATEGY=NOT_NULL)` 语义只更新非 null 字段，空串单位会被 `resolveUnit` 按「空白=未携带」放行，不会清空库内原值；未发现 mass assignment 面（白名单实体构建）。此为实现选择，不构成缺陷。
- 单位自由文本 vs brief「后端强制单位字典校验」：diff 行 291–293 提示「单位须取自单位字典（后端校验，禁手填）」，brief §3 与实施报告 §5 已明确裁决：无数据源时允许文本输入、以后端 iot_unit 校验为准。属已裁决项，非缺陷。

### 3.3 权限指令覆盖（重点 3）

diff 行 509–565（`index.vue`），依据 `frontend/src/plugins/privilege-plugin.ts`、`frontend/src/directives/privilege.ts`（`mounted` 钩子无权限时移除节点）：

| 操作 | 权限位 | diff 行 | 覆盖 |
| --- | --- | --- | --- |
| 查询 / 重置 / 导出 | `iot:point:list` | 509 / 513 / 532 | 有 |
| 新建 / 导入向导跳转 | `iot:point:add` | 525 / 536 | 有 |
| 批量启用 / 停用 / 行编辑 / 行启用停用 | `iot:point:edit` | 529 / 530 / 562 / 563 / 564 | 有 |
| 批量删除 / 行删除 | `iot:point:remove` | 531 / 565 | 有 |
| 详情 | `iot:point:query` | 561 | 有 |
| 列表数据可见性 | `iot:point:list` | 509 | 有 |

- 与 SQL 种子（`20261122_iot_point_domain.sql` 行 115–119：4053–4057 = list/query/add/edit/remove）一致。4079/4080 为 §3.1 的 import/export 权限位，但 controller 实际对导入复用 `iot:point:add`、导出复用 `iot:point:list`（`IotPointController` 行 175/193/222/244），brief 也明确「export 复用 iot:point:list」。前端按钮权限与后端守卫一致，未发现越权可见。
- `v-privilege` 只做 UI 隐藏，后端 `@SaCheckPermission` + 数据权限裁剪为真正防线；本 diff 未绕过后端权限。

### 3.4 defineExpose 覆盖父级 ref 调用（重点 4）

- `point-form-modal.vue` diff 行 452：`defineExpose({ showModal, onSubmit })`；父组件 diff 行 586 `<PointFormModal ref="pointFormModalRef" @reloadList="ajaxQuery" />`，diff 行 666–668 通过 `pointFormModalRef.value.showModal(rowData)` 调用 —— 覆盖。
- `point-detail-modal.vue` diff 行 249：`defineExpose({ showModal })`；父组件 diff 行 587 `<PointDetailModal ref="pointDetailModalRef" />`，diff 行 670–672 通过 `pointDetailModalRef.value.showModal(rowData.id)` 调用 —— 覆盖。
- 单测 diff 行 1014–1022、1063–1116、1122–1146 覆盖父 ref 调用链，BUG-0008 回归防护到位。

### 3.5 空态诚实（重点 5）

- 列表空态：diff 行 550 `<template #emptyText>暂无点位数据</template>`，`ajaxQuery` 取 `responseData.data?.records || []`（diff 行 656–657），无造假数据。
- 详情空态：diff 行 160–167 反查/曲线两个表格分别使用 `#emptyText`「暂无引用数据项」/「近 24h 暂无曲线数据（后端 fail-soft 空集，真实数据待采集入库）」；`references.value`/`curvePoints.value` 仅在 `Array.isArray(...)` 时赋值（diff 行 220、225、232），未伪造曲线点或引用记录。
- 后端事实：`IotPointDetailVO.curve24h` 与 `IotPointCurveVO.dataPoints` 本期均为 fail-soft 空集（`IotPointServiceImpl.getPointCurve24h` 注释与兜底、`queryCurve24h` 异常吞掉返回 `List.of()`），前端诚实展示与契约一致。
- `referenceCount`：diff 行 150–153 使用 `detail.referenceCount != null ? 值 : '-'`，与 `IotPointDetailVO.referenceCount`（Long）口径一致；分母未知时显示 `-` 而非伪造 0。

### 3.6 安全红线（重点 6）

- diff 全量检查：无硬编码凭据、appKey/secret、HMAC 计算、`/openapi/v1/**` 直连或自造代理。所有 API 均经 `/@/lib/axios` 管理面请求封装。
- 无 eval / innerHTML / 动态脚本注入；表格显示走 Vue 模板插值。

## 4 与 TASK 验收标准 / brief 接口的其他一致性核对

- 「IotPlaceholder 移除」：diff 行 459–471 删除 `<IotPlaceholder title="点位管理" task="TASK-0056 / TASK-0070" />`，替换为完整管理页；行 591 删除 `import IotPlaceholder`。满足 TASK 验收标准与 brief §必做。
- 列表过滤：diff 行 619–630 覆盖 `pointCode/pointName/dataType/valueType/status/channelId/deptId/boundInstanceId` 八项 + `pageNum/pageSize`，与 brief §列表要求一致。后端 `buildListWrapper`（`IotPointServiceImpl` 行 224–251）对八字段逐一 eq/like，前端传参名对齐。
- 分页：`a-pagination` diff 行 571–584，`v-model:current="queryForm.pageNum"` / `v-model:pageSize="queryForm.pageSize"`、`@change="ajaxQuery"`，`onSearch` 重置页码（diff 行 648–651）；与后端 `@RequestParam pageNum/pageSize` 一致。
- 启停与删除：
  - `changeStatus` payload `{ ids, status }`（diff 行 674–689）与 `IotPointStatusForm`（`ids` `NotEmpty`、`status` `NotNull`）一致；
  - 停用消息透出 `resp.data?.effectNote`（diff 行 677–683），与 `IotPointStatusChangeVO.effectNote` 一致；「变更已保存，重新发布后对接入生效」纪律沿用，未改发布逻辑；
  - 删除 `remove(ids)`（diff 行 739–748）与 `deleteRequest` 逗号拼接（重点 1）一致；二次确认文案（diff 行 715–737）说明绑定拒绝/编码不复用，与后端规则一致。
- 详情三读：diff 行 206–240 `Promise.allSettled` 并行拉 `detail/references/curve`，逐路独立捕获 `otSentry.captureError`，便于局部失败不整体空白；聚合 `curve24h` 优先于独立 curve 端点（diff 行 219–233），与 `IotPointDetailVO.curve24h` / `IotPointCurveVO.dataPoints` 两个事实源对齐。
- 导入向导：diff 行 536 仅 `<router-link to="/iot/import">` 跳转；SQL 确认菜单 4016 路由为 `/iot/import`，未重复实现导入向导。
- 枚举常量：diff 行 101–116 将点位 data_type 与 value_type（复用 `IOT_DATA_TYPE_OPTIONS`）登记到 `iot-consts.ts`，与后端 DTO / 导入模板说明一致；未发现同一值域多处硬编码冲突（其他模块 `device-component-zone.vue` 的 `DATA_TYPE_LABEL` 为展示文案，非本 diff 改动，不在评审范围）。
- 测试：diff 行 764–1147 新增 `point-api.spec.ts`（11 例）与 `point.spec.ts`（9 例），覆盖端点参数、父 ref、payload 白名单、空态与权限按钮；实施报告 §2 给出四门实测输出（`build:test` / `lint` / `typecheck` / `test` 均 exit 0，82 文件 / 697 例通过）。四门细节属验收输出核对，非本 diff 静态评审可复现项；diff 结构与报告声明一致。

## 5 无法判定项

- 无。本 diff 的所有必查点均可依据指定路径与后端事实源判定。

## 6 备注（非发现，供协调者参考）

- 本评审为静态 diff 评审，未复跑四门验收命令；「已通过」结论来自实施报告 §2 的原始输出记录，未独立验证其真实性。
- 设计选择（非缺陷）：详情弹窗 `a-descriptions` 使用 `:loading="detailLoading"`，ant-design-vue 4.2.5 的 `DescriptionsProps` 未声明该 prop；该属性会被透传为 HTML 属性而不产生骨架屏效果。实际加载态由 `detailLoading` 在 `showModal` 期间控制、结束后填充数据，`Promise.allSettled` 保证不会因单路失败卡死；不构成功能或数据缺陷，故不计 Minor。
- 设计选择（非缺陷）：表单错误分支 `otSentry.captureError(handleIotConflictError(error))` 与 iot 域其他表单范式（dict / channel / instance 等）一致，axios 拦截器已对非静默错误统一 `message.error`，用户可见；非本任务引入。
