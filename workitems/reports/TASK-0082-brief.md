# 实施 brief：TASK-0082（调用方管理页与发布回滚按钮——0070 批次三承接）

## 批次

- TASK-0082（本批共 1 个，微改任务直接转正已确认；前端道——API 全部已交付：0052 调用方四接口、0024 回滚接口）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `frontend/src/**`（调用方管理新页 + 发布页回滚按钮 + api 客户端）
  - 前端路由/菜单挂载配置
  - 不碰 backend/**、ingest/**、workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0082.md`
- 后端 API 契约源（开工先读 controller）：backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**（管理面 controller：POST /openapi/client、GET /openapi/client/list、GET /openapi/client/{id}、POST /openapi/client/rotate、PUT /openapi/client/status——明文仅一次协议/掩码形态/轮换并行窗 7 天/吊销立即 403）+ iot/publish/controller/IotPublishController.java（POST /iot/publish/rollback，4073 iot:publish:rollback）+ 菜单种子 20261123_iot_api_client_rotation.sql（4075-4078 挂 4000）
- 上游报告：`reports\TASK-0052-report.md`（接口行为/明文一次性协议/M3 前端规避点）+ `reports\TASK-0070-report-batch2.md`（范式基准）
- 范式基准：frontend/src/views/iot/** 与 api/iot/**（0056/0070 批一 二范式）+ **0052 评审 M3 规避**：编辑表单 ipWhitelist/expireTime 必须支持清空（后端 SaveForm 空值文案与实现不符——前端提交显式空值/空数组而非省略字段，具体形态以 0052 controller 实参为准）

## 接口与决策（逐条结论）

1. **调用方管理页**（菜单 4075 已有种子，路由照 0056 范式）：列表（appKey/掩码/scope JSON 摘要/rate_limit/status/轮换时间）→ 开通弹窗（服务端生成——响应含 appKey+secret 明文，**明文仅一次展示弹窗**+复制按钮+关闭即弃）→ 编辑（scope JSON 编辑器/限流/ipWhitelist/expireTime——**支持清空**，规避 0052 M3）→ 轮换按钮（确认框+新明文仅一次弹窗+并行窗 7 天提示——旧 secret 7 天内仍有效）→ 吊销/启用状态切换（确认框+「立即 403」提示）。openapi:client:* 四权限位（4075-4078）挂 v-privilege。
2. **回滚按钮**（发布页版本行操作列，4073）：回滚确认框（文案：回滚=以旧快照生成新版本并重新下发，历史不被改写）→ POST /iot/publish/rollback（scopeType/scopeId/targetVersion 取行数据）→ 成功后列表刷新（新版本行出现）。iot:publish:rollback 权限位挂 v-privilege。
3. **空态/分页/错误态**照 0056/0070 批一范式；不自造请求层/组件库。

### 测试口径

调用方页：开通明文一次性交互（关闭即弃断言）/掩码展示/轮换弹窗新明文仅一次+并行窗提示/吊销确认/空值清空提交断言（M3 规避）/权限位隐藏；回滚按钮：权限位/确认框文案/成功刷新。pnpm 三门 + live 走查（8098 探测，忙则跳过登记待补）。

## 验收命令

1. `pnpm build:test` / `pnpm lint` / `pnpm typecheck` → 全 0 error
2. live 走查（视 8098 空闲）：调用方页开通→列表掩码→轮换→吊销；发布页回滚按钮全链；全停
3. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落 frontend/src/** 与路由配置

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0082-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；**提交只 add frontend/**；提交注明 TASK-0082；不改 workitems 库（唯一例外：写报告）；不把 TASK 置「已完成」。
- 同框架同组件；不自造组件库/请求层。
