# 实施 brief：TASK-0095 指标管理页补齐

## 批次

- TASK-0095（本批共 1 个）
- 验收粒度：一个 subagent 跑一次本批的整体验收。
- BASE：449d7eb3——派发前在代码仓 `git rev-parse --short HEAD` 的结果（协调者填写；取哈希必须在代码仓目录里执行，别拿文档库的 HEAD 顶替）。
- 改动范围：仅以下前端文件/目录：
  - `frontend/src/api/iot/metric-api.ts`（新）
  - `frontend/src/views/iot/metric/index.vue`（改，替换 IotPlaceholder）
  - `frontend/src/views/iot/metric/components/metric-form-modal.vue`（新，可按需拆 detail modal）
  - `frontend/tests/unit/api/iot/metric-api.spec.ts`（新）
  - `frontend/tests/unit/views/iot/metric.spec.ts`（新）
  - 如需复用常量/字典查询，只允许改 `frontend/src/components/iot/iot-consts.ts` 和指标相关测试文件；不得改 backend/calc/ingest/contract。

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0095.md`
- 关联对象：`features/FEAT-0018.md`、`features/FEAT-0014.md`、`requirements/REQ-0024.md`、`testcases/TC-0014.md`
- 后端接口事实源：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/controller/IotMetricController.java`
- 后端 DTO：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/dto/` 下 `IotMetricCreateForm.java`、`IotMetricUpdateForm.java`、`IotMetricStatusForm.java`、`IotMetricGenerateForm.java`
- 前端范式：`frontend/src/views/iot/channel/index.vue`、`frontend/src/views/iot/dict/index.vue`、`frontend/src/api/iot/channel-api.ts`

## 接口与决策

- `metricApi` 必须覆盖后端已有六/七端点：
  - `queryList(params)` → GET `/iot/metric/list`（pageNum/pageSize/metricCode/metricName/instanceId/period/status）
  - `getById(id)` / `detail(id)` → GET `/iot/metric/{id}`
  - `add(form)` → POST `/iot/metric`
  - `generate(instanceId, form)` → POST `/iot/metric/generate?instanceId=`（注意 instanceId 是 query 参数，form 是 body）
  - `update(form)` → PUT `/iot/metric`
  - `changeStatus(form)` → PUT `/iot/metric/changeStatus`
  - `remove(ids)` → DELETE `/iot/metric/{ids}`（路径参数逗号拼接）
  - 从 `/@/lib/axios` 按项目既有范式导入 `getRequest/postRequest/putRequest/deleteRequest`；不要新增依赖
- 页面必须替换 `IotPlaceholder`，不再显示“完整功能由 TASK-0056 / TASK-0070 交付”。
- 列表：分页 + 过滤 metricCode/metricName/instanceId/period/status；数据权限由后端裁剪，前端不造假数据。
- 新增/编辑：metricCode 创建后不可改；表单校验与后端 DTO 注解一致——aggType 1-5、period HOUR/DAY/MONTH/YEAR、calendarType 锁 1；比值 aggType=5 时 ratioNumItemId/ratioDenItemId 必填；非比值型 ratio 字段置空；instanceId 必填即指标绑定设备语义。
- 批量生成：入口传 instanceId，按后端 `IotMetricGenerateForm` 提交；如前端缺少指标模板载体，留“按清单批量生成”的输入方式并在报告登记，不伪造模板。
- 权限：列表/查询 `iot:metric:list`/`iot:metric:query`、新增和批量生成 `iot:metric:add`、编辑启停 `iot:metric:edit`、删除 `iot:metric:remove`，使用项目既有 `v-privilege` 指令。
- 安全：不碰 appKey/secret/HMAC，不走 `/openapi/v1/**`；只走管理面 `/api` 登录态。
- 不改后端；后端缺接口时停并报告，不自造接口或 mock 数据。

## 歧义裁决

- 本任务只做指标页；点位页归 TASK-0097，不要碰 `frontend/src/views/iot/point/**`、不要改 point-api。
- `calendarType` 后端锁 1（自然），前端表单隐藏或只读锁定 1，不提供班次 2 选项。
- 启停走 changeStatus，不在编辑表单里改 status。
- 批量生成走 `POST /iot/metric/generate?instanceId=`，不是 `/iot/metric/add`。
- 如果新增弹窗组件，必须正确 `defineExpose`，避免 BUG-0008 复发；父组件通过 ref 调用时要有测试覆盖。
- 空态诚实：无数据显示空表/空详情，不伪造指标。

## 验收命令

- 命令：`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`
- 期望：四门 exit 0；lint 0 error；test 以派发时基线为零回归（执行前先记录 `pnpm test` 或报告基线；新增测试计入后总数更高）；改动文件范围仅前端文件清单内；页面不再出现 `IotPlaceholder`/“完整功能由 TASK-0056 / TASK-0070 交付”。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0095-report.md`。必须包含五项：

1. 改动文件清单
2. 每条验收命令的原始输出（不要只写「通过」）
3. 未做项与原因
4. 偏离计划之处
5. 遗留问题

## 契约

- **no-subagents**：你不得再派 subagent。
- 只改本批 TASK 范围内的文件；不得把 `状态` 置为「已完成」，那由协调者收尾。
- 不手写索引行；不得改本批之外的对象。
- 测试或验收失败先定位**根因**再修——禁止为了让输出匹配而改测试、改期望或打补丁。
- 每个修复要能说明**为什么现在对了**：附复现测试（先红后绿）或推理链，写进报告。

## 收尾回写

- 在 TASK 详情里勾选 `计划子任务`、填 `修改内容` / `任务结果`（含实测输出）。
- LOG 与 git 哈希由协调者跑 `close-task.sh` 生成。
