# 评审报告：TASK-0058 批次一 diff 评审（代码评审）

- 评审角色：fresh 代码评审子代理（口径：`.agents/protocol/L1-execute.md` §13「diff 评审」）
- 输入：diff 包 `TASK-0058-diff-batch1.txt`（BASE=5758d5f，提交=9aed9a4，13 文件）+ 批次 brief + 执行者报告 + TASK-0058 + FEAT-0019/REQ-0029/TC-0019/ADR-0011 + 范式基准（TASK-0055 报告、`api/iot/**`、`views/iot/**`、`router/index.ts` 既有挂载）
- 契约：fresh（仅依据必读与工作区源码）、no-subagents、只评审不修改；三门已由执行者实测，评审者不复跑构建
- 评审方式：逐条核对评审重点 1–7 + 常规质量缺陷（错误处理/边界/安全）+ 越界反查

## 越界反查（复核结论：无越界）

`git diff --name-only 5758d5f..HEAD` 输出 13 文件，与 brief 声明集、报告清单三者完全一致，全部落 `frontend/src/**` 与 `frontend/tests/**`。无 `backend/**`、`ingest/**`、`calc/**`、`workitems` 触碰。

工作区另有未提交的 `backend/.../openapi/**` 改动（TASK-0054 并行道）与未跟踪的 `.agents/`，均**不在**提交 9aed9a4 内（该提交 `git add` 只含 frontend/），与本批文件集不相交。越界项：无。

## 评审重点逐条核对

1. **数据纪律（首要）——通过**。全 diff 扫描未发现假数据/硬编码假数字。所有数据槽位（`display-slot-card.vue`）仅渲染 `<a-skeleton>` + 「实时数据接入中」空态；工厂/工厂页/车间页卡片内的运行/异常/离线、关键数据项、核心指标、能耗、状态色块、实时值均为空态槽位。`mark` 为可选字段，无标记走普通卡片（见 M-1 关于「放大」的措辞差距，但不涉及假数据）。未交付页（0059 设备详情/0060 指标看板）无入口、无死链——路由单测显式断言无 `/device/` 路径。
2. **模型树真接入——通过**。`display-api.ts` 只封装已交付的 `GET /iot/instance/tree`（已在后端 `IotModelInstanceController.tree()` 核实存在）；`DisplayTreeNode` 字段与后端 `IotInstanceTreeVO`（id/parentId/ancestors/instanceCode/instanceName/nodeType/templateId/appliedTemplateVersion/deptId/sort/status/locked/children）逐一对齐，`mark` 为预留可选字段。`queryInstanceTree()` 从拦截器 resolve 的响应体取 `res.data`（与既有 `instance-api.ts` 消费方式一致）。`use-display-tree` 不二次过滤权限，错误态/空态明确。注：所谓 lazy load 实为「一次拉全树 + 前端纯函数按需派生子节点」（见 M-3），与当前唯一可用端点相符。
3. **三级页面与下钻——通过（偏离已登记且合理）**。`/display`、`/display/factory/:factoryId`、`/display/workshop/:factoryId/:workshopId` 三路由已接线（`buildRouteList` 内 `push(...displayRouters)`，与 `iotUpgradeListRouter` 同范式，挂 OtLayout 下）。下钻点击数 0→1→2，设备详情第 3 次由 0059 承接、本批不写入口。静态 `hideInMenu` 路由范式与既有 survey/*、iot/upgrade 一致。执行者登记的「未走 sys_menu 动态菜单」偏离：brief 明示「独立路由、与管理端并存不混用」，且 `hideInMenu` 是测试硬约束，采用静态路由合理（但无应用内入口，见 M-2）。
4. **上看板标记（ADR-0011）——基本通过，一处措辞/实现差距（M-1）**。`mark` 可选字段在 `DisplayTreeNode` 与 slot-card props 中预留；渲染层有徽标、`--marked` 视觉强调、父级按 mark 排序 + `order:-1` 置顶；无标记时无徽标/无 class/保持后端 sort，降级正确。差距：注释与测试名宣称的「放大」在 CSS 中未实现（`--marked` 仅加左边框+阴影，无尺寸放大）——因 mark 恒缺省当前不可见。
5. **同框架同组件——通过**。沿用 OtLayout（路由挂载）、OtLoading（`ot-loading/index.ts` 的 store show/hide）、antd v4 的 `a-card`/`a-skeleton`/`a-empty`/`a-alert`、请求层 `getRequest`（`/@/lib/axios`）。图标 `@ant-design/icons-vue/{Right,Left,Reload}Outlined` 子路径导入已在 node_modules 与既有用法核实可用。未自造 UI 库/请求层。
6. **测试有效性——通过（3 文件共 10 用例）**。routes（3）：路径/命名、下钻≤3 且无设备死链、需登录+hideInMenu+不混用 `/iot/`——锁定真实约束。slot-card（2）：空态无假数字、mark 徽标/class 渲染——锁定数据纪律与标记逻辑。tree（5）：`childrenOf` 按 sort、`childrenOf(父)` 直接子/lazy 语义、`findNode(999)/childrenOf(999)` 空（前端不补无权节点）、`descendantsOfType` 跨层级、mark 透传。测试为约束断言而非纯渲染快照。
7. **卸载清理/内存泄漏——通过**。composable 仅用 `ref`，无定时器/监听器/订阅，`OtLoading.show/hide` 在 try/finally 配对，无泄漏面。lint 9 warning 经核为存量未改测试文件（edit-lock-api/setting-drawer-persist/axios/redpack-*），新增文件静态检查无新增 warning 迹象；「本批 0 新增 warning」未独立复跑 lint 核实（按 brief 不复跑，标**无法判定**，但无静态反证）。

## 常规质量核对（§13）

- **错误处理/边界**：`queryInstanceTree` 空/非数组返回 `[]`；`factoryId`/`workshopId` 用 `Number.isFinite` 守卫，非法 id → 空态而非报错；加载失败 error 态独立展示；禁用节点（status===0）不下钻。✓
- **安全/权限**：前端完全依赖后端数据权限裁剪，`findNode`/`descendantsOfType` 仅基于已裁剪树操作；直调无权限工厂 id 时 `findNode` 返回 null → 「不存在或无权访问」空态，不带 id 再发请求，无越权数据泄露。✓
- **路由守卫**：display 三路由无 `meta.public`，走 `beforeEach` token 守卫。✓
- **路由注册**：绝对子路径挂在 path='/' 的 OtLayout 父路由下，与 `/iot/**` 无冲突。✓
- **UI 细节（Minor）**：禁用卡片仍有 `cursor:pointer` 与 `@click`，点击静默无反馈（不下钻本身正确）。

## 发现清单

| 编号 | 严重级 | 出处（文件:行） | 说明 |
| --- | --- | --- | --- |
| M-1 | Minor | `frontend/src/views/display/components/display-slot-card.vue:42-45`、`frontend/src/views/display/components/display-factory-card.vue:52-55`；测试名 `frontend/tests/unit/display/display-slot-card.spec.ts:26` | brief 评审点 4 要求标记驱动「放大/置顶」，但 `--marked` 类只加左边框+阴影、无尺寸放大；测试名「放大 Class」名不副实。因 `mark` 接口未交付、恒缺省，当前无可见错误；属批次二接入真值前的声明/实现差距，建议批次二补尺寸放大或将措辞改为「视觉强调」。 |
| M-2 | Minor | `frontend/src/router/display.ts:1-31`、`frontend/src/router/index.ts:242-244` | 三路由均 `hideInMenu` 且无 sys_menu 种子，全仓无指向 `/display` 的菜单或页面链接，展示端仅能靠手输 URL 直达。与 brief 的静态路由/hideInMenu 设计一致、非缺陷，但请协调者确认展示端入口形态（URL 直连/大屏壳/独立菜单）是否为预期。 |
| M-3 | Minor | `frontend/src/api/display/display-api.ts:56-64`、`frontend/src/views/display/composables/use-display-tree.ts:373-389` | brief 的「按需 lazy load」实为「一次性拉全树 + 前端纯函数派生子节点」，非网络级分段加载。与当前唯一可用端点（仅全树查询）相符、实现正确，仅语义留痕；若后续树规模增大需服务端分段加载再议。 |

> 无法判定项：lint「本批 0 新增 warning」与三门 exit 0 依 brief 不复跑，仅依据执行者报告与静态检查，未独立实测。

## 结论

**通过**——无 Critical、无 Important。越界反查干净（13 文件全落 frontend/）；数据纪律、模型树真接入、三级下钻≤3、无死链、同框架同组件、测试有效性、卸载清理均满足 brief 与 TASK/REQ/FEAT/ADR 约束。3 条 Minor（M-1 标记「放大」未落地、M-2 展示端无应用内入口、M-3 lazy load 为前端派生语义）均不阻塞本批骨架验收：M-1 随批次二 mark 真值接入处理或改措辞，M-2 请协调者确认入口形态，M-3 仅留痕。按 §13，Minor 记 TASK `任务结果` deferred（收尾进 LOG），不进修订轮。
