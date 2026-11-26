# 代码评审报告 · TASK-0070 批次二（diff 评审）

- 评审者：code-reviewer（fresh，唯一输入：`TASK-0070-review-batch2-brief.md` 及其「必读」清单 + 工作区源码）
- 评审对象：`88dce89..e492973`（12 文件，全落 `frontend/src/**`）；评审时 HEAD=e492973，工作区与提交逐字一致（`git diff e492973 --name-only -- frontend/src/` 为空）
- 口径：L1-execute §13「diff 评审」；前端三门（build:test/lint/typecheck 0 error）与真登录全链冒烟（含 5002 三支路与内容漂移复现）由执行者实测（`TASK-0070-report-batch2.md`），评审者按 brief 不复跑构建，采信其原始输出
- 后端契约核对源（评审者实读）：`IotModelTemplateController.java`（impact-preview）、`IotTemplateUpgradeController.java`、`IotInstanceLockController.java`、`IotPublishController.java`（publish/snapshot）、`IotModelInstanceController.java`（lock 表单域）、`PublishForm.java`、`SnapshotVO.java`、`TemplateImpactPreviewVO.java`、`TemplateChangeEntryVO.java`、`TemplateUpgradeRowVO.java`、`TemplateUpgradeConfirmForm.java`、`IotInstanceLockForm.java`、`IotTemplateVersionServiceImpl.java`（previewImpact/hasPendingTier2Confirm）、`TemplateAwarePublishConfirmGate.java`、`IotPublishServiceImpl.java`（retrieveSnapshot）、ot-core `ResultCode.java`（5002）、sys_menu 种子 SQL（4025/4033/4034/4071/4072）

---

## 一、结论

**通过**（无 Critical / 无 Important；Minor 7 条见第三节，均不阻塞本批验收）。

批次二三件组（发布页影响预览+二次确认段 / 待升级清单新页 / 快照查看抽屉）与后端已交付接口的契约逐端点核对一致；previewToken 协议三支路（未预览直发 / 带 token 过期·漂移 / 重预览）交互与引导文案完整成立；diff 展示与 ADR-0008 三档政策文案逐条对照一致；组件/请求封装/权限交互与 0056、批一范式同构；顺手清理三项（R-2/R-3/R-6）按 brief 落地且未越界；执行者报告的 BASE 修正与 defineExpose 冒烟修复复核属实。

---

## 二、评审重点逐项核对（brief 7 项）

### 1. previewToken 协议交互完整性 —— 三支路全部实现，判定口径成立

- **无 T2 不带 token 的判定口径**（brief 点名追问）：前端依据预览响应 `TemplateImpactPreviewVO.diff[]` 的 `tier` 字段（`TemplateChangeEntryVO.tier`）——`hasTier2 = preview.diff.some(tier === 'T2')`（`publish-confirm-modal.vue:207`）。未预览（previewed=false）同样按无 token 处理（footer「直接发布（未预览）」），由后端确认门兜底拒绝。请求体侧 `...(confirmToken ? { confirmToken } : {})` 缺省时字段整个不传，与 `PublishForm.confirmToken` 可缺省契约一致（一档/三档/无变更放行、二档缺失拒绝）。
- **三支路**（`publish-confirm-modal.vue:309-340` doPublish）：
  - 未预览直发被 5002 拒 → 无 token 分支文案「存在待确认的第二档模板变更——请先选择关联模板获取影响预览，核对 diff 后二次确认发布」（:330-331）——引导先做预览 ✓；
  - 带 token 再次 5002（过期/漂移）→ 文案「确认凭据已过期或发布内容已漂移——请重新获取影响预览并再次确认」+ **预览态清空**（preview=null / previewed=false，:325-328）→ 关联模板保持选中，重预览一键可达 ✓；
  - 两条文案均锚定「5002 下发未确认」，与 ot-core `ResultCode.java:38 PUBLISH_NOT_CONFIRMED(5002, "下发未确认")` 逐字对齐 ✓。
- **silentError 自渲染成立**：`publishConfig` 请求级 `silentError: true`（publish-api.ts）→ `axios.ts:104-108` 仅跳过全局 toast、仍 reject AxiosResponse（body 在 `e.data`）→ `e.data.code === 5002` 判定成立、错误经弹窗内 alert 渲染，无双重提示；其他错误回落 `body.message` 兜底。
- **带 token 发布的二次确认**：`hasTier2 && previewed` → `Modal.confirm`（影响面计数+档位摘要+目标版本核对，:287-302）→ onOk 携 `preview.previewToken` 发布 ✓；与确认门校验链（token 非空→凭据存在→复合哈希一致→token 匹配，`TemplateAwarePublishConfirmGate.java:83-123`）对应的三个失败情形全部收敛到 5002 两个前端分支（缺失=未预览分支、过期/漂移/不匹配=带 token 分支）——支路映射完备。
- **越界核对**：网关 scope（scopeType=1）发布经确认门恒放行（gate :74-77，网关域快照不含模板内容），前端携 token 发布无害；执行者报告遗留 3 已登记该边界，与代码事实相符。

### 2. diff 展示与 ADR-0008 一致性 —— 逐条对照成立

| ADR-0008 政策 | 前端文案（文件:行） | 判定 |
| --- | --- | --- |
| T1 纯新增/展示类，随发布自动生效 | 徽标「T1 随发布生效」（publish-confirm-modal.vue:123；upgrade/index.vue:345-347）；无 T2 时 alert「……均为第一档（纯新增/展示类，不改变任何已有值的含义）：随发布自动生效，直接发布即可」（:82-88）——「不改变任何已有值的含义」与 ADR 档位定义逐字同源 | ✓ |
| T2 随发布生效 + 强制影响预览+二次确认；锁定不跟随、进待升级清单 | 徽标「T2 强制确认」；alert「随发布生效但强制二次确认」（:68-74）；受影响/锁定实例**分列**（affectedInstanceCount / lockedInstanceCount 两个 descriptions-item，:56-60，锁定附注「不随发布，进待升级清单逐台确认」）；锁定开关文案「锁定后该实例不自动跟随二档变更（语义变更），保留在待升级清单逐台确认后再升级」（upgrade/index.vue:447-451） | ✓ |
| T3 永不自动、逐台确认迁移，每台出 diff 与迁移结果 | 徽标「T3 逐台迁移」+ **「不随本次发布」附加标**（:125-127，brief 点名要求项）；发布侧「逐台迁移，不随本次发布——本页发布不含这些变更」双 alert（:75-95）；清单侧 T3 行内警示「永不自动生效；确认升级推进版本簿记，每台迁移结果展示待迁移任务交付后增强」（upgrade/index.vue:108-110、381）——迁移结果留待 0081 的边界与批 brief 预定口径一致 | ✓ |
| FORBIDDEN（改 data_code 直接禁止） | `v-else` 红标「{{tier}} 禁止」（publish-confirm-modal.vue:126；upgrade/index.vue:354） | ✓ |
| 档位摘要（tierSummary） | 发布弹窗「档位摘要」（空变更显「（无变更）」，:61）；清单行 tierSummary 按 `,` 分段着色徽标（upgrade/index.vue 模板段）——与后端 `TemplateTierRuleTable.formatSummary`「T1:1,T2:1」格式吻合 | ✓ |

### 3. 待升级清单页 —— 行集/确认/锁定与 0016 契约逐字段对齐

- **行集字段**：instanceId/instanceCode/instanceName/appliedTemplateVersion/currentVersion/locked/tierSummary/diff 与 `TemplateUpgradeRowVO.java:26-48` 逐字段一致；「已应用 → 目标」列 = applied → current 组合渲染 ✓；请求 `templateId/pageNum/pageSize` 与 `IotTemplateUpgradeController.list` 三 @RequestParam 对齐，templateId 未选不发请求（卫语句对齐后端 required）✓。
- **批量确认组装**：`{instanceIds: rows.map(instanceId), targetVersion}` 与 `TemplateUpgradeConfirmForm`（@NotEmpty instanceIds / @NotNull targetVersion）一致；targetVersion=首行 `currentVersion`（upgrade/index.vue:271）——行集按单模板查询、行间 currentVersion 恒等（后端「必须等于模板当前版本」校验兜底），取值口径成立；整批原子/冲突 4002 文案与后端语义逐字一致 ✓。
- **确认后行消失/空态**：成功与失败两态统一重查（:434-438 finally ajaxQuery）；空态两区分（未选模板「请先选择模板」/已选「暂无待升级实例（引用实例的模板版本已全部对齐模板当前版本）」，:268）——冒烟已实证空态呈现 ✓。
- **锁定开关**：`PUT /iot/instance/lock {instanceId, locked: 0|1}` 与 `IotInstanceLockForm`（0 解除/1 锁定）一致；入口在行操作列（brief 预定位置）✓；锁定/解锁确认文案与 ADR-0008 二档政策一致（锁定不自动跟随、解锁回归跟随面）✓。0012「locked=1 编辑受限」口径未在文案体现——见 Minor F4。
- **权限分离**：清单查询/重置/查询按钮挂 `iot:template:query`（与后端 @SaCheckPermission 一致），确认升级/锁定挂 `iot:instance:edit`（与 `IotTemplateUpgradeController.confirm`、`IotInstanceLockController.setLock` 注解一致）；模板页入口按钮同挂 4025——「视线与动作分离」落地成立 ✓。v-privilege 按 perms 字符串匹配（`privilege.ts:23`），种子 SQL 实证 4025/4033 权限串存在 ✓。

### 4. 快照抽屉 —— 契约与「原始字节按原样处理」成立

- **信封与哈希**：配置版本/发布范围/快照文件（版本行 snapshotUri）/快照哈希（`a-typography-paragraph copyable` 复制按钮）——SnapshotVO（scopeType/scopeId/version/snapshotHash/snapshotBody）与 `IotConfigVersion`（行携 snapshotUri）实读一致；「快照文件」取自版本行而非快照 VO（SnapshotVO 无 snapshotUri 字段，前端取数来源正确）✓。4071 权限位：入口按钮与抽屉内刷新按钮均挂 `iot:publish:query`，种子 SQL 4071=iot:publish:query ✓。
- **Schema 版本**：解析自快照体顶层 `schemaVersion`（信封 VO 无该字段）——与 SnapshotVO 结构实读一致 ✓。
- **原始字节透传后的体形态**（brief 点名追问）：管理面 `IotPublishServiceImpl.retrieveSnapshot:379-381` 出站 snapshotBody = `new String(存储原始字节, UTF_8)`（与内部拉取共用 readAndVerifySnapshot，TASK-0080 RawValue 修复口径覆盖管理面）；前端**原文视图 `{{ snapshot.snapshotBody }}` 逐字渲染**（不 JSON.parse、不重序列化，数字形态如 5.000000 原样保留），折叠树视图 JSON.parse 仅作展示、不回传任何请求——「按原样处理」成立 ✓；解析失败回落原文视图（:104-107）✓。
- **折叠树/原文切换**：`vue3-json-viewer` 为既有依赖（package.json 2.3.1；样式经 `plugin/admin-ui/index.ts:56` 全局注册，与 operate-log-detail-modal 同款）✓。

### 5. 同框架同组件 —— 与 0056/批一范式一致

- 请求全部走既有 `/@/lib/axios`（getRequest/postRequest/putRequest + silentError 请求级扩展）；OtLoading（写路径 show/hide try-finally，与 channel/gateway 页同款）/ otSentry / PAGE_SIZE_OPTIONS / v-privilege / ot-margin-* 全为既有约定。
- 新弹窗与抽屉 `defineExpose({ showModal })` 均在位（publish-confirm-modal.vue:342-344、snapshot-drawer.vue:148-150）——执行者冒烟修复（弹窗漏 expose）复核属实；父页 ref 调用签名匹配（publish/index.vue:272/278/287）；本批仅这两个新组件需要对外暴露，**无第四处遗漏**。
- a-pagination 属性组合（v-model:current/pageSize + @change + show-total + PAGE_SIZE_OPTIONS）与 gateway 页逐属性同款；模板下拉 pageSize 500 对齐 maxLimit 与 batch1/批一既有同款；写操作 catch 统一 `otSentry.captureError(handleIotConflictError(e))` 与 0056 各 modal 逐字同款（组件登记不触发口径未变，冲突提示由全局拦截器 toast 兜底）。
- 路由形态：`router/iot.ts` named export + `router/index.ts buildRouteList` 挂载（照 surveyRecycleRouter 范式，hideInMenu 无菜单项页面）——与 0016「不新增菜单种子」决策一致；页面可达性由权限点承担（REQ-0034 口径）✓。
- 行选择批量模式与 tenant/template 页既有 `:row-selection` + 全选确认范式同构（且确认内容计数取当前页交集，较既有页更精确）。

### 6. 顺手清理三项 —— 按 brief 落地且未越界

| 项 | 出处（workspace 行号） | 判定 |
| --- | --- | --- |
| R-2 accept 收窄 | `import/index.vue:40` `accept=".xlsx"`（与页面提示文案「.xlsx」一致） | ✓ |
| R-3 吊销轮换文案 | `channel-credential-drawer.vue:223-227` 启用行文案不变、已吊销行补「并将该已吊销凭据复活启用（status 置回启用）」——方向与后端 rotate 复活语义一致 | ✓ |
| R-6 导入下一步挂权限位 | `import/index.vue:53` `v-privilege="'iot:point:add'"`（与 dragger 同位） | ✓ |

两文件 diff 仅含上述改动，未夹带 R-1/R-4/R-5/R-7（brief 未授权，批一 Minor 基线维持）——未越界 ✓。

### 7. 三态/清理/分页

- **三态**：清单页 loading（tableLoading）+ 空态两区分 + 错误全局 toast+otSentry；发布弹窗 previewing 按钮 loading + publishError 自渲染 alert + 无变更展示（「（无变更）」+空 diff 表）；快照抽屉空态（a-empty，快照体为空）+ 错误全局 toast——加载态缺失见 Minor F5。
- **卸载清理**：无新增定时器/全局监听器；两抽屉/弹窗 close() 全量复位（preview/previewed/publishError/versionRow/snapshot/bodyView/templateId）；Modal.confirm onOk 均 resolve promise。
- **分页**：清单页 v-model:current/pageSize + @change 与既有页一致；发布弹窗 diff 表与清单 diff 展开表 `:pagination="false"`（条目数受模板变更规模天然约束 + y 滚动）——与 0056/0018 存量口径同构，不另立发现。

### 越界反查（复核）

`git diff --name-only 88dce89..e492973` = 12 文件，全部 `frontend/src/**`（含 brief「前端路由/菜单挂载配置」授权的 router 两文件），无越界——与协调者比对一致；BASE 修正（082d694→88dce89 混入 TASK-0029 ingest 提交）经 git log 实证属实（cfed417/88dce89 为 0029 两提交，介于开工记录与本批提交之间）。

---

## 三、发现清单

无 Critical、无 Important。以下 7 条 Minor：

| # | 级别 | 出处 | 说明 |
| --- | --- | --- | --- |
| F-1 | Minor | `frontend/src/views/iot/publish/components/publish-confirm-modal.vue:26-34` | 「关联模板」下拉切换/清空后不重置预览态（无 @change 复位 previewed/preview），后续确认发布段与二次确认框仍展示上一模板的 diff/计数/凭据（hasTier2 亦按旧 diff 计算）——发布动作本身仍被后端确认门兜底（token 与租户内容哈希绑定，行为正确），风险限于展示混淆；建议切模板即清预览态 |
| F-2 | Minor | `frontend/src/views/iot/publish/components/publish-confirm-modal.vue:207` + `backend/…/gate/TemplateAwarePublishConfirmGate.java:79` | 「无 T2 不带 token」判定是**单模板**维度（所预览模板的 diff[].tier），而后端确认门判档是**租户全域**（hasPendingTier2Confirm 遍历租户全部 T2 版本行）——租户内其他模板存在待确认 T2 时，无 token 直发被 5002 拒，引导文案可恢复但未指明 T2 在哪个模板（重预览同模板会循环 5002）；口径系批 brief 预定（「无 T2 不带 token 直发」），属设计层不对称登记，非实现偏差 |
| F-3 | Minor | `frontend/src/views/iot/publish/components/publish-confirm-modal.vue:75-81` | 「变更**均为**第三档」alert 挂在 `v-else-if="hasTier3"`（无 T2 即触发），diff 为 T1+T3 混合时「均为」失准（应为「存在第三档变更」）；同屏独立的 T3 error alert 文案正确可对冲，发布行为不受影响 |
| F-4 | Minor | `frontend/src/views/iot/upgrade/index.vue:443-459` | 锁定确认文案只讲「不自动跟随二档变更」，未提及 0012 既有口径「锁定后实例编辑仅允许 status」的连带后果（api 注释与后端 javadoc 均登记了该语义）——锁定影响面提示不完整 |
| F-5 | Minor | `frontend/src/views/iot/publish/components/snapshot-drawer.vue:17-19,58` | 快照取回中无加载指示（tableLoading 仅用于抑制空态，descriptions 全显「-」）；哈希位在未取回/取回失败时渲染空 content 的 copyable 段落（复制图标悬空可点）——纯展示性瑕疵 |
| F-6 | Minor | `frontend/src/api/iot/upgrade-api.ts:6`、`frontend/src/views/iot/upgrade/index.vue:10`、`frontend/src/router/iot.ts:18` | 注释将 iot:instance:edit 的 sys_menu id 写作 4034，种子 SQL 实证 4033=iot:instance:edit / 4034=iot:instance:remove（源头为 TASK-0016 报告 :254，前端沿用）——v-privilege 按 perms 字符串匹配、功能无影响，纯注释失准，执行者报告第五节映射表同注记一并修正 |
| F-7 | Minor | `frontend/src/views/iot/publish/components/publish-confirm-modal.vue:63,68-74,137` | 网关 scope（scopeType=1）发布时弹窗仍呈现计算域语义文案——「本次发布版本（计算域）」、T2 alert「随发布生效但强制二次确认」、footer「确认发布（已预览，携确认凭据）」（网关域快照仅点位/通道、确认门恒放行）：预览信息本身有效，但文案暗示 T2 变更随本次网关发布生效，属语义误导（gate 层已兜底，发布行为正确） |

---

## 四、对执行者报告的复核

1. **BASE 修正说明**：属实——`git log` 实证 cfed417/88dce89 为 TASK-0029 两提交且位于开工记录 082d694 之后、本批 e492973 之前；净足迹以 88dce89..e492973 计恰 12 文件，与本评审独立比对一致。
2. **defineExpose 冒烟修复**：最终态两新组件 expose 均在位（第二节 5 已核）——修复完整，无第四处遗漏。
3. **偏离登记 1-6**：逐条与代码事实相符——静态路由 /iot/upgrade 形态与 0016「不新增菜单种子」吻合；查询按钮 disabled 收紧（scopeReady）与 ajaxQuery 卫语句一致且发布按钮复用同前置，属纯显示性微调；漂移制造经探针直调后端、前端 5002 处理为 UI 实测的口径合理。
4. **冒烟遗留登记**：T3 附加标 live 走查缺口（同一模板块清单侧已实证）、TTL 自然过期未复现（与漂移同分支且漂移已实证）、网关 scope 发布链未走查（gate 恒放行有代码背书）——均为如实登记的验证边界，静态审查结论不冲突。
5. **页面-接口映射核对表**：逐行核对属实；唯 iot:instance:edit 行 id 注记 4034 失准（见 F-6）。
6. **三门构建/冒烟**：按 brief 评审者不复跑、不重放，采信报告原始输出（build:test/lint/typecheck EXIT=0 + 三件组全链走查记录，含 5002 双支路与内容漂移的 expectedHash≠storedHash 实证）。

## 五、无法判定项

无。评审重点 1-7 均可依据 brief「必读」+ 工作区源码 + 后端契约源（controller/VO/Form/gate 实现/种子 SQL）判定；唯一未独立复核的是执行者实测的构建输出与冒烟截图（按 brief 口径采信，非存疑）。
