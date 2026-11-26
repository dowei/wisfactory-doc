# TASK-0070 实施报告 · 批次二（发布页影响预览段 + 待升级清单页 + 快照查看入口）

- 执行者：implementer（frontend 独立道，brief 唯一输入：`TASK-0070-brief-batch2.md`）
- 开工 BASE：`082d694`（`git rev-parse --short HEAD`）；实现提交 = **e492973**（12 文件，注明 TASK-0070 batch2）
- **净足迹基准修正（并行道事实）**：BASE 记录后、本批提交前，TASK-0029 两提交（cfed417 / 88dce89）落入 main——净足迹核对以 `git diff --name-only 88dce89..e492973` 为准（12 文件）；`082d694..HEAD` 会混入 0029 的 ingest 文件（非本任务改动，全程未触碰）
- 验收口径：三门（build:test / lint / typecheck）+ 联调冒烟（8098 开工探测**空闲**，全量走查）+ 改动范围核对

---

## 一、改动文件清单（12 文件，全部落在 brief 声明范围 frontend/src/**）

`git diff --name-only 88dce89..e492973` 实测恰 12 文件（=第五条验收命令原始输出），无越界；提交 `e492973` 逐文件显式 add（未用 git add -A，工作区 ingest/** 0029 在途改动与 `.agents/` 零触碰）。

| 文件 | 新/改 | 内容 |
| --- | --- | --- |
| `frontend/src/api/iot/upgrade-api.ts` | 新 | 0016 三接口组一个封装：queryUpgradeList（GET /iot/template/upgrade/list，templateId 必传）、confirmUpgrade（POST /iot/template/upgrade/confirm，{instanceIds,targetVersion}）、setInstanceLock（PUT /iot/instance/lock）。头注登记跨命名空间（清单/确认归模板域子资源控制器、锁定归实例域控制器）与「视线（iot:template:query 4025）与动作（iot:instance:edit 4034）分离」 |
| `frontend/src/api/iot/publish-api.ts` | 改 | 扩两读写：publishConfig（POST /iot/publish/publish，{scopeType,scopeId,confirmToken?}，**silentError** 由发布弹窗自渲染错误引导）、querySnapshot（GET /iot/publish/snapshot）。头注改写：发布已接入（0015 确认门 confirmToken 语义）；**回滚 4073 不提供**（批次二三件组决策未含，登记后续批次） |
| `frontend/src/api/iot/template-api.ts` | 改 | 扩 queryImpactPreview（GET /iot/template/impact-preview，权限复用 iot:template:query 4025） |
| `frontend/src/views/iot/publish/components/publish-confirm-modal.vue` | 新 | 发布确认弹窗（挂 scopeType+scopeId 上下文）：关联模板下拉（pageSize 500 对齐 maxLimit）→ 获取影响预览（未选模板 disabled+提示）→ 预览结果段（affectedInstanceCount/lockedInstanceCount **分列**+锁定提示、档位摘要、模板当前版本、本次发布版本（计算域）、凭据 TTL、diff 表逐条目档位徽标 T1 随发布生效/T2 强制确认/T3 逐台迁移+「不随本次发布」附加标、三档政策 alert——ADR-0008 文案）→ 二次确认（Modal.confirm 含影响面与档位摘要）→ 携 previewToken 发布；**无 T2 diff 直接发布不带 token**；5002 三支路引导（未预览→「存在待确认的第二档模板变更，请先获取影响预览」；带 token 过期/漂移→「确认凭据已过期或发布内容已漂移，请重新获取影响预览」+**预览态清空**）；发布成功 emit published → 父页刷新版本列表 |
| `frontend/src/views/iot/publish/components/snapshot-drawer.vue` | 新 | 快照抽屉（挂版本行上下文，行携 scopeType/scopeId/version/snapshotUri）：GET /iot/publish/snapshot（iot:publish:query 4071）→ 信封展示（配置版本/发布范围/**Schema 版本解析自快照体顶层 schemaVersion**/快照文件（版本行 snapshotUri）/**快照哈希+复制按钮**）+ 哈希核对提示 alert + 快照体**折叠树（vue3-json-viewer 既有依赖，expanded=false 折叠）/原文 pre 切换**（radio 组）+ 解析失败回落原文 |
| `frontend/src/views/iot/publish/index.vue` | 改 | 查询组新增「发布」按钮（v-privilege iot:publish:publish 4072，danger，未选范围 disabled）→ 打开发布确认弹窗（scopeLabel 计算：网关下拉 label / 计算租户级）；版本行操作列新增「查看快照」（v-privilege iot:publish:query 4071，操作列 110→170）；查询按钮 disabled 前置收紧为 scopeReady（scopeType+（scopeType=1 时网关）必选，与 ajaxQuery 卫语句对齐）；头注登记发布/快照接入与回滚未含 |
| `frontend/src/views/iot/upgrade/index.vue` | 新 | 待升级清单页：模板下拉（必传前置卫语句，未选不发请求+提示）+ 行集表（instanceId/instanceCode/instanceName/**已应用→目标**/locked 徽标/**tierSummary 分段徽标**/操作列）+ 行展开 diff 表（逐条目：对象类型/业务键/变更形态徽标/字段/旧值/新值/档位徽标，T3 行内警示 alert「逐台确认迁移…每台迁移结果展示待迁移任务交付后增强」）+ 行 selection 批量确认（**整批原子**提示）+ 逐台/批量确认 dialog（T3 台数提示；targetVersion=行集模板当前版本）+ **锁定开关行操作**（锁定/解锁确认框，语义文案：锁定不自动跟随二档变更/解锁回归跟随面）+ 空态区分（未选模板「请先选择模板」/已选「暂无待升级实例（全部已对齐）」）；确认/锁定后两态均重查（4002 提示刷新与行消失两态统一） |
| `frontend/src/views/iot/template/index.vue` | 改 | 行操作新增「待升级清单」入口（v-privilege iot:template:query 4025——清单查询权限位）→ router.push /iot/upgrade?templateId=预选；操作列 170→240；头注登记 |
| `frontend/src/router/iot.ts` | 改 | 预留空位首次启用：named export `iotUpgradeListRouter`（/iot/upgrade，hideInMenu，meta.title 待升级清单），javadoc 登记无后端菜单行缘由（0016 复用既有权限位不新增菜单种子） |
| `frontend/src/router/index.ts` | 改 | buildRouteList 挂载 `iotUpgradeListRouter`（照 surveyRecycleRouter 同款：无菜单项页面无条件挂布局下，可达性由权限点承担） |
| `frontend/src/views/iot/import/index.vue` | 改 | 顺手清理 R-2：`accept=".xlsx,.xls"` → **".xlsx"**（后端解析器仅支持 xlsx，.xls 上传只会得到解析失败通用错误） |
| `frontend/src/views/iot/import/index.vue` | 改 | 顺手清理 R-6：「下一步：开始导入」按钮挂 `v-privilege="'iot:point:add'"`（无权限角色不再得到不可用空壳页） |
| `frontend/src/views/iot/channel/components/channel-credential-drawer.vue` | 改 | 顺手清理 R-3：对「已吊销」行的同类型重复写入，确认文案补「并将该已吊销凭据复活启用（status 置回启用）」（rotate 复活语义，启用行文案不变） |

未动文件：views 非 iot 目录、backend/**、ingest/**、resources/**、workitems 库（本报告除外）；新组件 defineExpose 与 0056/批一抽屉范式一致。

---

## 二、验收命令原始输出

### 1. `cd frontend && pnpm build:test` → EXIT=0（最终态复跑；首轮亦 0 error）

```text
$ vite build  --base=/admin/ --mode test
...（chunk 产物清单中略）
(!) Some chunks are larger than 500 kB after minification. ...（chunk 大小 warning，基线既有可忽略）
✓ built in 1m 6s
BUILD_EXIT=0
```

### 2. `cd frontend && pnpm lint` → EXIT=0，0 error / 9 warning（=开工基线）

```text
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the --fix option.
LINT_EXIT=0
```

9 条 warning 全部为 tests/unit/** 既有 prettier 格式告警（与 0056/批一报告基线逐文件相同）。本批新增文件曾产生 8 条 prettier warning，`eslint --fix` 清零；`npx eslint "src/views/iot/**" "src/api/iot/**" "src/router/**"` 复核输出为空（0 问题）。

### 3. `cd frontend && pnpm typecheck` → EXIT=0

```text
$ vue-tsc --noEmit
TYPECHECK_EXIT=0
```

### 4. 改动范围核对：`git diff --name-only 88dce89..e492973` → 恰 12 文件全在声明内

```text
frontend/src/api/iot/publish-api.ts
frontend/src/api/iot/template-api.ts
frontend/src/api/iot/upgrade-api.ts
frontend/src/router/index.ts
frontend/src/router/iot.ts
frontend/src/views/iot/channel/components/channel-credential-drawer.vue
frontend/src/views/iot/import/index.vue
frontend/src/views/iot/publish/components/publish-confirm-modal.vue
frontend/src/views/iot/publish/components/snapshot-drawer.vue
frontend/src/views/iot/publish/index.vue
frontend/src/views/iot/template/index.vue
frontend/src/views/iot/upgrade/index.vue
```

判定：12/12 全落 `frontend/src/**`（含 brief「前端路由/菜单挂载配置」授权的 router 两文件），无越界；并行道 TASK-0029 提交（cfed417/88dce89，ingest/**）未触碰、未纳入提交。

### 5. 联调冒烟（brief 第 2 条）→ 8098 探测空闲，**全量走查通过**（非缺席登记）

开工探测：`curl 127.0.0.1:8098/auth/captchaImage` → 000/EXIT=7 无监听、netstat 无 LISTENING；WSL mysql 3307 / redis 6379 在位 → `mvn spring-boot:run -pl ot-admin`（8098，Started OtAdminApplication in 11.868s）→ `pnpm dev`（8099）→ 真实登录页登录 admin（验证码 3BTS 截图识读，未绕过）→ 切换模块「智慧工厂」→ 三件组走查 → 全停。

**（a）发布页影响预览 + 二次确认段（4015，brief 命名的「发布页预览→5002→带 token 发布全链」）**

| 走查项 | 结果 |
| --- | --- |
| 未选范围 | 查询/发布双 disabled + 提示「请先选择发布范围」+ 表格空态 ✓ |
| scope=计算 查询 | 版本列表 3 行（v3/v2/v1 0015 冒烟遗留），行操作「下发状态 查看快照」在位 ✓ |
| 发布弹窗结构 | 标题「发布配置（影响预览 + 二次确认）」+ 保存≠生效/ADR-0008 提示横幅 + 发布范围「计算（租户级，scopeId=0）」+ 关联模板下拉 + 「影响预览需先选择关联模板」+ 未选模板时「获取影响预览」disabled ✓ |
| **直接发布（未预览）→ 5002** | 模板 10 造 T2 变更（公式 /1000→/10000，模板自动升版 4→5 summary=T1:1,T2:1）后点「直接发布（未预览）」→ 弹窗内红字 alert「**发布未通过确认门（5002 下发未确认）**：存在待确认的第二档模板变更——请先选择关联模板获取影响预览，核对 diff 后二次确认发布」；后端日志 `[CONFIRM-GATE] publish rejected (token missing) tenant=1 scope=2-0 targetVersion=4` ✓（silentError 自渲染，无双重 toast） |
| 影响预览段 | 选模板 SMOKE0015_TPL → 获取影响预览 → 受影响实例数 3 / **锁定实例数 1（不随发布，进待升级清单逐台确认）分列** + 档位摘要 T1:2,T2:1 + 模板当前版本 v5 + 本次发布版本（计算域）v4 + 凭据有效期 300 秒 + diff 三条目（dataName T1 随发布生效 / defaultFormula /100→/10000 **T2 强制确认** / status 1→- T1——0015 已登记的一次性历史噪声条目）+ T2 警示横幅「随发布生效但强制二次确认」+ 页脚按钮变「确认发布（已预览，携确认凭据）」 ✓ |
| 二次确认 + 携 token 发布 | Modal.confirm「存在第二档变更（T1:2,T2:1）：受影响实例 3 台（未锁定跟随）、锁定实例 1 台（不随发布，进待升级清单）。确认发布计算范围 v4 吗？」→ 确认 → 「发布成功：已生成配置版本 v4」→ 版本列表刷新出现 v4 行（id 32，已发布，04:40:09）；后端日志 `[CONFIRM-GATE] tier2 publish confirmed tenant=1 scope=2-0 targetVersion=4` ✓ |
| **内容漂移（再次 5002）→ 重新预览** | 预览（v6 锚定 token）→ 探针 PUT /iot/template/item 再改公式（/100000→/1000000，模板 6→7 T2:1）→ 弹窗内点「确认发布」→ alert「**发布未通过确认门（5002 下发未确认）：确认凭据已过期或发布内容已漂移——请重新获取影响预览并再次确认**」+ **预览结果段清空、页脚回「直接发布（未预览）」**；后端日志 `publish rejected (content drifted since preview) … expectedHash=e9cf721d… storedHash=d9670fe8…` → 重新获取影响预览（模板当前版本 v7）→ 确认发布 → 「发布成功：已生成配置版本 v5」；后端 `tier2 publish confirmed … targetVersion=5` ✓ |

**（b）待升级清单页（/iot/upgrade，brief 命名的「行/diff 展开/单台+批量确认」）**

| 走查项 | 结果 |
| --- | --- |
| 入口 + 预选 | 模板管理页行操作「待升级清单」（iot:template:query 显隐）→ URL `/#/iot/upgrade?templateId=10`，模板下拉预选 SMOKE0015_TPL（SMOKE0015 template），标签栏出现「待升级清单」页签 ✓ |
| 行集 | 20 EQ1A / 21 EQ1B / 22 EQ1C，「v2 → v4」格式 + locked 徽标（22 **已锁定**橙标，操作列「解锁」；未锁定行「锁定」）+ tierSummary 徽标（T1:2 T2:1）+ 批量确认升级（0）disabled ✓ |
| diff 展开 | EQ1A 行展开 → 逐条目表（模板数据项 ELEC_CONS dataName 修改 T1 / LOAD_RATE defaultFormula 修改 T2 / 模板头 status 噪声 T1），档位徽标与旧值/新值 code 样式 ✓ |
| **锁定开关（PUT /iot/instance/lock）** | EQ1C「解锁」→ 确认框「解除后回归跟随面（二档变更随发布自动生效）」→ 行变未锁定/操作列变「锁定」；回锁 → 确认框「锁定后不自动跟随二档变更，保留在待升级清单逐台确认」→ 行变已锁定；后端日志 `[INSTANCE-LOCK] instanceId=22 locked 1 -> 0` → `0 -> 1` 往返净零 ✓ |
| **单台确认** | EQ1A「确认升级」→ 确认框「从模板版本 v2 确认升级到 v4 吗？…锁定实例同样经此通道升级」→ 成功 → 行消失（对齐不再出现）+「确认升级成功：1 台实例已对齐模板当前版本 v4」；后端 `[INSTANCE-UPGRADE] confirmed instanceId=20 … applied 2 -> 4` ✓ |
| T3 语义提示 | 删 LOAD_RATE 数据项（真实逻辑删）→ 模板 7→8 **T3:1**（`[TEMPLATE-VERSION] … dataItem[LOAD_RATE] REMOVED (T3)`）→ 清单行变「v2 → v8 / T1:2 T3:1」→ 展开 EQ1B：行内警示「**该实例存在第三档变更（结构破坏类）：逐台确认迁移，永不自动生效；确认升级推进版本簿记，每台迁移结果展示待迁移任务交付后增强**」+ diff 行「LOAD_RATE 删除 T3 逐台迁移」红徽标 ✓ |
| **批量确认** | 全选 3 台（20/21/22）→ 批量确认升级（3）启用 → 确认框「选中的 3 台实例…到模板当前版本 v8…整批原子：任一实例校验失败整批拒绝不半成功（冲突 4002 请刷新重试）。**其中 3 台存在第三档变更（逐台迁移语义，同上）**」→ 批量确认 → 「确认升级成功：3 台实例已对齐模板当前版本 v8」→ **清单空态**「暂无待升级实例（引用实例的模板版本已全部对齐模板当前版本）」共 0 条；后端逐台留痕 applied 4→8 / 2→8 / 2→8（22 保持 locked=1） ✓ |

**（c）快照查看入口（brief 命名的「快照抽屉哈希展示」）**

| 走查项 | 结果 |
| --- | --- |
| 入口 | 版本行操作「查看快照」（iot:publish:query 4071）打开「配置快照」抽屉 ✓ |
| 信封展示 | 配置版本 v3 / 发布范围 计算（租户级，scopeId=0）/ **Schema 版本 1.0.1**（解析自快照体顶层）/ 快照文件 1/2-0-3.json（版本行 snapshotUri）/ **快照哈希（SHA-256）64 位全显 + 复制按钮** + 哈希核对提示 alert「服务端取回时已做 SHA-256 哈希核对（与落库哈希不一致将报错告警——存储损坏探测）」 ✓ |
| 折叠树 / 原文切换 | 折叠树（默认）：JsonViewer 折叠展示 dataItems[19 项]/formulas[10 项]/metrics[]/schemaVersion/scopeId/scopeType/tenantId/version；切「原文」→ JSON 原文完整回显（{"schemaVersion":"1.0.1","tenantId":1,…}）✓ |

**（d）冒烟闭环修复（走查发现 → 当场修复 → 三门复跑）**：首轮点击「发布」报 `Uncaught TypeError: publishConfirmModalRef.value.showModal is not a function`——新弹窗组件漏写 `defineExpose({ showModal })`（与批一两抽屉同型问题），当场补上后弹窗/预览/发布全链正常；修复后三门（build:test EXIT=0 / lint EXIT=0,0e9w / typecheck EXIT=0）在**最终态**复跑全绿；该修复已含于提交 `e492973`。

**（e）全停**：TaskStop 后子进程残留（java PID 55956 / node PID 44336），核实进程身份（corretto-21 java / pnpm node，均为本任务所起——8098/8099 开工时无监听）后定点终止——netstat 复查 8098/8099 LISTENING 清空。未强杀任何他人进程。

---

## 三、未做项与原因

### 批次三登记清单（本批 brief 明示不做，不建占位代码）

| 未做项 | 归属 | 原因 |
| --- | --- | --- |
| 调用方管理页（开通/scope/轮换） | TASK-0052 | API 未交付 |
| 适配器管理页（注册/心跳/状态） | TASK-0033+ | API 未交付 |
| 指标重算发起与进度入口 | TASK-0047 | API 未交付 |
| 回滚按钮（iot:publish:rollback 4073） | 0070 后续批次 | API 已交付（0024）但**批次二三件组决策未含回滚交互**——publish-api 头注登记「本封装不提供」，页面不建占位按钮 |
| 每台迁移结果展示（T3 逐台迁移结果） | TASK-0081 | brief 明示「留待 0081 交付后增强，本批先承接确认动作」——清单页 T3 提示文案已锚定该边界 |

### 本批内未做/未覆盖项

| 未做项 | 原因 |
| --- | --- |
| 发布弹窗 diff 内 T3「不随本次发布」附加标的 live 走查 | 冒烟序列中 T3 删除发生在发布链验证之后（发布时 diff 仅含 T1/T2 条目）；T3 徽标+附加标与待升级清单展开行为**同一模板块**（清单侧已 live 验证），发布侧标注静态正确性佐证 |
| token TTL 自然过期（300s 等待）复现 | 过期与内容漂移同归 5002 同一处理分支（文案「已过期或发布内容已漂移」），**漂移支路已实测**（expectedHash≠storedHash 实证）；TTL 等待无增量信息 |
| 网关 scope（scopeType=1）发布链走查 | 确认凭据恒锚定计算域（0015 偏离 5：网关域快照仅点位/通道，gate 对网关 scope 恒放行）→ 网关发布无 token 分支；网关下拉/版本列表批一已走查 |
| 无权限角色的入口隐藏/403 走查 | admin 全权限会话；拦截一律后端（REQ-0034 口径），显隐由 v-privilege 逐按钮挂权限位（4071/4072/4025/4034）静态核对 |
| 发布/确认并发冲突（5001/4002）的 live 复现 | 后端报文与全局 toast 兜底（非 silentError 路径除 publish 外均为既有全局拦截器），与 0056/批一范式同构；前端无冲突特判需求 |

---

## 四、偏离计划之处

1. **「待升级清单页」落地形态 = 静态路由 /iot/upgrade（hideInMenu）+ 模板页行操作入口，非独立菜单页**：0016 明文「不新增菜单种子」（清单查询复用 iot:template:query 4025 / 确认与锁定复用 iot:instance:edit 4034）→ 无 sys_menu 行可挂——照 0055 在 `router/iot.ts` 预留的「无后端菜单项 iot 页面静态注册位」+ `surveyRecycleRouter`（buildRouteList 挂布局下）范式落地；页内自带模板选择器可独立直达，实测标签栏以 meta.title「待升级清单」正常呈现页签。若协调者要求独立菜单行需后端补菜单种子（backend 范围，本批不做）。
2. **BASE 修正（并行道事实，非越界）**：开工 BASE=082d694 记录在案；TASK-0029 提交（cfed417/88dce89）在本批提交前落入 main——净足迹核对基准相应取 88dce89..e492973（12 文件）；082d694..HEAD 中出现的 ingest 文件全部属 0029，与本任务无关（未 stage/commit 过任何非 frontend 文件）。
3. **冒烟闭环修复**：publish-confirm-modal 漏 defineExpose（与批一两抽屉同型）——冒烟首击即暴露、当场修复、三门最终态复跑全绿，修复含于本批提交（批一先例同口径）。
4. **漂移支路的内容变更经探针直调后端制造**：UI 弹窗遮挡下无法经页面在「预览后」触发漂移，用 dev /auth/login/test 登录 + curl PUT /iot/template/item 直改（0015/0016 探针同口径）——漂移为 0015 复合哈希语义的真实触发，**前端 5002 处理（alert 文案/预览态重置/重预览引导）为 UI 实测**。
5. **T3 场景经删除模板 10 的 LOAD_RATE 数据项制造**（真实逻辑删，删除即三档语义）：模板 10/实例 20-22 为 0015 冒烟遗留夹具；删除后模板 10 数据项仅剩 ELEC_CONS，确认升级将 20/21/22 applied 全推进到 v8——dev 库遗留登记见遗留 2。
6. **查询按钮 disabled 前置收紧**：批一版本列表查询按钮仅判 scopeType，本批顺手对齐为 scopeReady（scopeType=1 时网关必选才启用）——与 ajaxQuery 既有卫语句一致，发布按钮复用同一前置；纯显示性微调（原来网关未选时按钮可点但点了不出请求）。

---

## 五、遗留问题

### 页面-接口映射核对表（静态证据：前端调用 vs controller 注解逐条对齐）

**发布页（IotPublishController @RequestMapping("/iot/publish") + IotModelTemplateController）**

| 前端调用 | 后端端点 | 权限位（sys_menu） | 对齐 |
| --- | --- | --- | --- |
| publishApi.publishConfig → POST /iot/publish/publish（{scopeType,scopeId,confirmToken?}，silentError） | @PostMapping("/publish") @Valid PublishForm（confirmToken 0015 扩展；缺失/过期/漂移 5002） | iot:publish:publish 4072 | ✓ |
| publishApi.querySnapshot → GET /iot/publish/snapshot（scopeType/scopeId 必传，version=行版本） | @GetMapping("/snapshot") Result\<SnapshotVO\>（服务端哈希核对） | iot:publish:query 4071 | ✓ |
| templateApi.queryImpactPreview → GET /iot/template/impact-preview?templateId | @GetMapping("/impact-preview") Result\<TemplateImpactPreviewVO\>（previewToken/TTL/diff/计数分列） | iot:template:query 4025 | ✓ |
| （未提供）POST /iot/publish/rollback | iot:publish:rollback 4073 | 预留 | ✓ 前端未封装（批次二三件组未含，登记批次三） |

**待升级清单页（IotTemplateUpgradeController + IotInstanceLockController）**

| 前端调用（upgrade-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryUpgradeList → GET /iot/template/upgrade/list（templateId/pageNum/pageSize） | @GetMapping("/list") Result\<Page\<TemplateUpgradeRowVO\>\>（行=applied<current，diff 带档位，instanceCode 增量字段） | iot:template:query 4025 | ✓ |
| confirmUpgrade → POST /iot/template/upgrade/confirm（{instanceIds[],targetVersion}） | @PostMapping("/confirm") @Valid TemplateUpgradeConfirmForm（targetVersion 必须等于模板当前版本；乐观锁/重复提交 4002 整批拒绝） | iot:instance:edit 4034 | ✓（页面确认/锁定按钮均挂此权限） |
| setInstanceLock → PUT /iot/instance/lock（{instanceId,locked}） | @PutMapping("/lock") @Valid IotInstanceLockForm（0 解除/1 锁定） | iot:instance:edit 4034 | ✓ |

页面内跨域消费：publish-confirm-modal → templateApi.queryTemplateList（pageSize 500 对齐 maxLimit）；upgrade/index.vue → 同款模板下拉。

### 遗留问题清单

1. **批一 Minor 未清项（R-1/R-4/R-5/R-7）仍在**：brief 只授权 R-2/R-3/R-6；R-7 同款「scopeType=1 显示原始 scopeId（网关 id）」在快照抽屉发布范围沿用（与下发状态抽屉同信封限制）——可读性增强登记可选。
2. **dev 库冒烟遗留数据**：模板 10 升至 v8（数据项仅剩 ELEC_CONS，LOAD_RATE 已逻辑删）、实例 20/21/22 applied 全=v8（22 保持 locked=1——0015/0016 冒烟延续状态）、配置版本 v4（id 32）/v5 两行、快照文件、版本历史行——照 0015/0016 冒烟遗留同口径隔离未清理（DB 行引用一致性优先）。
3. **sys_operate_log 审计增量**：本批冒烟产生留痕——发布 4 次（2 次失败 5002 如实记 status=0 + 2 次成功）、锁定往返 2 条、确认升级 4 条（单台 1 + 批量 3 逐台）、数据项删除/修改若干——均为 @OperateLog 预期行为（0016 STEP10 同口径），非脏数据。
4. **`.agents/` 未跟踪目录**：开工前已存在，未触碰未提交；ingest/** 在途改动（0029）已被其自身提交收口，本批全程未触碰。
5. **待升级清单页的菜单形态裁决**（偏离 1）：静态路由 + 模板页入口已可用；如需侧栏菜单行需后端补种子（本批改动范围不含）。
6. **快照抽屉哈希核对失败分支（存储损坏 500）未复现**：dev 存储健康，后端取回核对不一致报 500 走全局拦截器——条件渲染无需前端特判，登记观察。

---

**结论：通过**（三门 0 error：build:test/lint/typecheck EXIT=0、lint 9 warning=既有基线；联调冒烟三件组全链走查通过——发布页 5002 未预览引导→影响预览（计数分列+档位徽标+TTL）→二次确认携 token 发布 200（v4）→内容漂移再次 5002（expectedHash≠storedHash 实证）→预览态重置→重新预览→发布 200（v5），待升级清单行集/diff 展开/锁定开关往返/单台+批量确认（含 T3 逐台迁移语义提示）→空态，快照抽屉哈希复制/Schema 版本/折叠树与原文切换；冒烟闭环修复 1 处（defineExpose，批一同型）已复跑三门背书；改动范围 12/12 全落 frontend/src/** 无越界）。
