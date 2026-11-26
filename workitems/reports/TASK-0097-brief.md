# 实施 brief：TASK-0097 点位管理页补齐

## 批次

- TASK-0097（本批共 1 个）
- 验收粒度：一个 subagent 跑一次本批的整体验收。
- BASE：4e4c1461——验收时对比改动范围用（取哈希必须在代码仓目录里执行，别拿文档库的 HEAD 顶替）。
- 改动范围：仅以下前端文件/目录：
  - `frontend/src/api/iot/point-api.ts`（改）
  - `frontend/src/views/iot/point/index.vue`（改，替换 IotPlaceholder）
  - `frontend/src/views/iot/point/components/point-form-modal.vue`（新，可按需拆 detail modal）
  - `frontend/src/views/iot/point/components/point-detail-modal.vue`（新）
  - `frontend/tests/unit/api/iot/point-api.spec.ts`（新）
  - `frontend/tests/unit/views/iot/point.spec.ts`（新）
  - 如需复用常量/字典查询，只允许改 `frontend/src/components/iot/iot-consts.ts` 和已有点位相关测试文件；不得改 backend/calc/ingest/contract。

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0097.md`
- 关联对象：`features/FEAT-0018.md`、`requirements/REQ-0034.md`、`testcases/TC-0001.md`
- 后端接口事实源：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/controller/IotPointController.java`
- 后端 DTO：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/IotPointCreateForm.java`、`IotPointUpdateForm.java`、`IotPointStatusForm.java`、`IotPointDetailVO.java`
- 前端范式：`frontend/src/views/iot/dict/index.vue`、`frontend/src/views/iot/channel/index.vue`、`frontend/src/api/iot/point-api.ts`

## 接口与决策

- `pointApi` 必须覆盖后端已有点位端点：
  - `queryList(params)` → GET `/iot/point/list`（已有，保留）
  - `getById(id)` / `detail(id)` → GET `/iot/point/{id}`
  - `references(id)` → GET `/iot/point/{id}/references`
  - `curve(id)` → GET `/iot/point/{id}/curve`
  - `add(form)` → POST `/iot/point`
  - `update(form)` → PUT `/iot/point`
  - `changeStatus(form)` → PUT `/iot/point/changeStatus`
  - `remove(ids)` → DELETE `/iot/point/{ids}`（路径参数逗号拼接）
  - `export(params)` → GET `/iot/point/export`，下载 xlsx；如框架下载封装不支持 GET 下载，允许只暴露 API 包装并在页面用现有下载工具，须在报告说明
  - 导入三方法与 `queryList` 保留，导入向导独立菜单 4016 不重复实现，本页只保留跳转链接
  - `postRequest/putRequest/deleteRequest/getDownload` 等从 `/@/lib/axios` 按项目既有范式导入；不要新增依赖
- 页面必须替换 `IotPlaceholder`，不再显示“完整功能由 TASK-0056 / TASK-0070 交付”。
- 列表：分页 + 过滤 `pointCode/pointName/dataType/valueType/status/channelId/deptId/boundInstanceId`；过滤项以接口 DTO 为准，无数据源时允许用原生输入或既有字典封装，但空态必须诚实，不造假数据。
- 新增/编辑：pointCode 创建后不可改；表单校验与后端 DTO 一致；启停走 changeStatus，不在编辑表单里改 status。
- 详情：复用后端聚合详情 `IotPointDetailVO`（含 referenceCount/curve24h 等事实口径）；可展示反查 `references`，曲线按后端 fail-soft 空集诚实提示。
- 权限：列表/查询 `iot:point:list`/`iot:point:query`、新增 `iot:point:add`、编辑启停 `iot:point:edit`、删除 `iot:point:remove`，使用项目既有 `v-privilege` 指令；没有权限的按钮隐藏。
- 安全：不碰 appKey/secret/HMAC，不走 `/openapi/v1/**`；只走管理面 `/api` 登录态。
- 不改后端；后端缺接口时停并报告，不自造接口或 mock 数据。

## 歧义裁决

- 本任务只做点位页；指标页归 TASK-0095，不要碰 `frontend/src/views/iot/metric/**`、不要新建 metric-api。
- 导入向导已有完整菜单（4016 / `views/iot/import/index.vue`），本页仅放跳转，不重复实现导入步骤。
- 配置文件变更“保存≠生效，需发布后生效”的纪律不因本任务改变；如页面提示，沿用既有文案。
- 曲线/反查数据为空时展示诚实空态，不得伪造曲线点或引用记录。
- 如果使用新增弹窗组件，必须正确 `defineExpose`，避免 BUG-0008 复发；父组件通过 ref 调用时要有测试覆盖。

## 验收命令

- 命令：`cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`
- 期望：四门 exit 0；lint 0 error；test 至少 677 例基线零回归（新增测试计入后总数更高）；改动文件范围仅前端文件清单内；页面不再出现 `IotPlaceholder`/“完整功能由 TASK-0056 / TASK-0070 交付”。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0097-report.md`。必须包含五项：

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
