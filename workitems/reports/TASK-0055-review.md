# TASK-0055 diff 评审报告（fresh reviewer）

- 评审时间：2026-11-21
- 输入：TASK-0055-review-brief.md（角色/依据）、TASK-0055-diff.txt（16 文件，完整读入）、TASK-0055-brief.md（接口契约与裁决）、TASK-0055.md（验收标准）、TASK-0055-report.md（交叉印证）、L1-execute.md §13（评审口径）
- 评审两样均覆盖：①与 TASK 验收标准 / brief 接口契约的一致性；②明显质量缺陷（错误处理、边界、安全、范式一致性、占位页可维护性）

---

## 一、契约一致性核对（依据 ①，逐项）

| 契约点 | 结果 | 证据 |
| --- | --- | --- |
| 菜单号段 4008-4016 连续 | 通过 | SQL 新增 9 行 4008-4016 连续，parent=4000，sort 2-10 与 TASK-0001 的 4001（sort=1）衔接，无跳号/复用 |
| menu_type=2、无按钮行 | 通过 | 9 行全部 menu_type=2、perms 全 NULL、无 menu_type=3 行；未破坏 SaPermissionRegistryConsistencyTest 固化的「注解 ⊆ sys_menu SQL 登记」不变量（验收 3 两侧集合仍相等，恰为 iot:tenant:* 六个） |
| module_id=4 | 通过 | 9 行全部 module_id=4 |
| 幂等口径 | 通过 | INSERT IGNORE + 显式 id，样式与 20261121_sys_tenant.sql 逐列一致（visible=1/status=1/cache_flag=0 等值沿用）；sys_role_menu 复跑 OK(0) 实证（role_id,menu_id）复合键幂等成立；语句前无 -- 注释行，兼容 sql-exec.sh 剥注释 + 行尾分号切分 |
| privilege 指令绑定 iot:tenant:* | 通过 | 新建=iot:tenant:add、编辑=iot:tenant:edit、停用/启用=iot:tenant:status、删除=iot:tenant:remove，与 SysTenantController 六个 @SaCheckPermission 对齐；list/query 两个权限位由菜单可达性承担、页面无对应操作按钮，无漏绑 |
| 权限后端下发、前端只显隐、无越权前端拦截 | 通过 | 全 diff 无路由守卫/403 拦截逻辑；router/iot.ts 为空数组，静态注册位为空恰好避免绕过菜单门禁；页面可达性 = sys_menu 下发（buildRoutes import.meta.glob，router/index.ts 209-213 行解析 iot/*/index 路径，与新增视图文件一一匹配） |

## 二、质量抽查（依据 ②）

- API 封装：tenant-api.ts 与 api/system/position-api.ts 范式一致（getRequest/putRequest/deleteRequest、注释风格、无空壳预写，符合 brief 决策 3）；六个方法路径与 SysTenantController 逐一对上（/tenant/list、/{id}、POST、PUT、/{id}/status、DELETE /{ids}）。
- 错误处理：axios 拦截器统一 toast + 页面 try/catch + otSentry.captureError，与 dict 页同构；表单弹窗校验失败 catch 只对 validate 拒绝生效，无双重报错。
- 安全：无 v-html/innerHTML，表格与 Modal.confirm content 均纯文本渲染，无 XSS 注入面；无越权前端拦截逻辑。
- 图标：9 个菜单图标（BookOutlined/SnippetsOutlined/ClusterOutlined/AimOutlined/GatewayOutlined/ApartmentOutlined/BarChartOutlined/CloudUploadOutlined/ImportOutlined）均在 @ant-design/icons-vue 4.2.5 中存在。
- 前端用法：a-modal `:open="visible"`、okType:'danger'、`:pagination="false"` + 独立 a-pagination、OtLoading/PAGE_SIZE_OPTIONS 均为现仓 system 页既定用法（antd-vue 4.2.5 支持）。
- 占位页可维护性：9 页共用 iot-placeholder.vue 单组件传 props，注释标明菜单挂载点与归属任务，符合「空态列表骨架」验收口径。
- 越界反查：协调者已确认全部落在 brief 声明范围（routers.ts 仅 +3 行 import/注释/展开）；本次评审复核 diff 内容无越界实改。
- 无法判定项：无。

## 三、发现清单

1. **Minor** — `frontend/src/router/iot.ts` + `frontend/src/router/routers.ts`（+3 行）
   静态路由段以空数组落地，是对 brief 决策 3 字面「路由注册进 iot 段」的已声明偏离（实施报告·偏离 2）；动态注册机制经浏览器验收成立且静态注册反而绕开菜单门禁，属合理纠偏，但需协调者按报告请求正式裁决并在 TASK 任务结果留痕。

2. **Minor** — `frontend/src/views/iot/tenant/index.vue`（deleteTenant / changeStatus）
   删除/启停后按当前 pageNum 直接重查，删空末页或过滤后页码超界时出现空页（未收敛 pageNum）；沿袭 dict 页既有范式，非本 diff 新引入的模式，建议 TASK-0056 收口完整租户页时一并处理。

3. **Minor** — `frontend/src/views/iot/tenant/components/tenant-form-modal.vue`（showModal）
   编辑态把整行 record（含 createTime/updateTime/deleted 等非表单字段）Object.assign 进表单并整体作为 PUT /tenant 载荷上行（tenantCode 虽禁输仍在载荷中，依赖后端「编辑不进更新集」兜底）；与 dict 弹窗同范式，建议 TASK-0056 收敛提交字段。

4. **Minor** — `frontend/src/views/iot/tenant/index.vue`（confirmDelete）
   删除二次确认文案未提示「逻辑删除不释放编码」（tenant_code 删后不可复用，该约束目前仅写在 tenant-api jsdoc 注释中）；完整租户页归 TASK-0056，建议届时补提示文案。

## 四、结论

- Critical 0 / Important 0 / Minor 4。
- 依据 L1-execute §13：无退回项；4 条 Minor 记 TASK `任务结果` 的 deferred（收尾进 LOG），不进修订轮。
- 遗留裁决提醒：实施报告·偏离 2（空静态路由段）待协调者确认；报告·遗留 1（dev 库测试夹具清理）与遗留 2（8098/8099 运行中的服务）需协调者跟进，均不属 diff 本身缺陷。
