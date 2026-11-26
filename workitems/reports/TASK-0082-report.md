# 实施报告：TASK-0082（调用方管理页与发布回滚按钮——0070 批次三承接）

- 日期：2026-11-23
- 执行者：implementer（frontend 独立道，brief 唯一输入：`TASK-0082-brief.md`）
- BASE：`1494e1e`（开工时主仓 HEAD，即 TASK-0052 提交）；本任务提交 = **7771181**（main，注明 TASK-0082，6 文件）
- 验收口径：三门（build:test / lint / typecheck）+ live 走查（8098 开工探测**空闲**，全量走查）+ 改动范围核对
- 结论先行：**全部验收通过**。三门全 0 error；live 走查（真实登录 admin、验证码识读、未绕过）覆盖调用方页开通（明文仅一次弹窗+复制+关闭即弃断言）/列表掩码/轮换（并行窗文案双提示+新明文仅一次）/吊销（立即 403 文案）/编辑 M3 规避（清空支持+显式空值提交审计实证）/发布页回滚（确认框 ADR-0008 文案+行数据载荷+新版本 v6 快照哈希与 v3 完全一致实证「历史不被改写」）；提交 6 文件全落声明范围。

---

## 一、改动文件清单（6 文件，`git diff --name-only 1494e1e..7771181` 实测恰 6，全部落 `frontend/src/**`）

| 文件 | 新/改 | 内容 |
| --- | --- | --- |
| `frontend/src/api/iot/apiclient-api.ts` | 新 | OpenApiClientController 五接口封装：queryClientList（GET /openapi/client/list）、queryClientDetail（GET /{id}）、saveClient（POST——开通/编辑同端点）、rotateClientSecret（POST /rotate）、updateClientStatus（PUT /status）。头注登记明文仅一次协议（SecretVO 唯一明文出口、弹窗关闭即弃）、轮换并行窗、吊销立即生效、M3 规避口径 |
| `frontend/src/views/iot/apiclient/index.vue` | 新 | 调用方管理页（sys_menu 4075 `/iot/apiclient`，页面行承载 openapi:client:list）：查询组（appKey 模糊/status 精确）+「开通调用方」按钮（v-privilege openapi:client:add 4076）+列表（appKey/调用方名称/**secret 掩码**/**scope JSON 摘要**（解析 instanceIds/subtree：未授权=红标「默认拒绝」/实例×N（含子树）/格式异常橙标）/限流（0=不限）/状态/**最近轮换时间**（空=从未轮换））+行操作 编辑/轮换密钥（4077）/吊销-启用（4078，按行状态条件切换）——全部 v-privilege |
| `frontend/src/views/iot/apiclient/components/apiclient-form-modal.vue` | 新 | 开通/编辑表单弹窗：校验与 OpenApiClientSaveForm 注解对齐（appKey 必填 ≤64 编辑禁用不提交/名称 ≤100/scope JSON 前置校验非空合法对象/rateLimit ≥0/ipWhitelist ≤500/expireTime/remark ≤500）；**M3 规避：编辑提交显式携带全部可编辑字段**——ipWhitelist 清空提交 `''`、expireTime 清空提交 `null` 而非省略字段（载荷构造恒含全部键）；scope 空白串=显式清空（后端三态契约）；开通成功 emit('created', SecretVO) 交父页打开一次性明文弹窗（本组件不持有明文） |
| `frontend/src/views/iot/apiclient/components/apiclient-secret-modal.vue` | 新 | **secret 明文一次性展示弹窗**（REQ-0035 验收 1 前端半边）：appKey+明文各挂复制按钮（a-typography-paragraph copyable 既有范式）+后端 notice 告警条+轮换场景附加并行窗 info 条（旧 secret 默认 7 天内有效过窗失效）+「我已妥善保存，关闭（关闭后无法再次查看）」单按钮；**关闭即弃**（组件状态清空，不在前端驻留） |
| `frontend/src/api/iot/publish-api.ts` | 改 | 扩 rollbackConfig（POST /iot/publish/rollback，{scopeType,scopeId,targetVersion}）；头注改写：「回滚本封装不提供」→「TASK-0082 接入，回滚=以 targetVersion 旧快照生成新版本并重新下发，历史版本不被改写」 |
| `frontend/src/views/iot/publish/index.vue` | 改 | 版本行操作列新增「回滚」（v-privilege iot:publish:rollback 4073，danger）；确认框文案（ADR-0008 回滚语义）：「回滚=以该版本旧快照生成新版本并重新下发，历史版本不被改写（新版本快照复用旧快照文件）」→ POST /iot/publish/rollback（**scopeType/scopeId/targetVersion 取行数据**，与查询上下文解耦）→ 成功 toast + 刷新版本列表；操作列 170→230；头注登记回滚接入与「版本行只由发布/回滚链路写入（TD-0034）」 |

**未动 router 两文件**（brief 授权范围含「前端路由/菜单挂载配置」，实测不需要——见偏离 1）；未动 backend/**、ingest/**、resources/**、workitems 库（报告除外）。

## 二、验收命令原始输出

### 1. `cd frontend && pnpm build:test` → EXIT=0

```text
$ vite build  --base=/admin/ --mode test
(!) Some chunks are larger than 500 kB after minification. ...（chunk 大小 warning，基线既有可忽略）
✓ built in 56.00s
BUILD_EXIT=0
```

### 2. `cd frontend && pnpm lint` → EXIT=0，0 error / 9 warning（=开工基线）

```text
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the `--fix` option.
LINT_EXIT=0
```

9 条 warning 全部为 tests/unit/** 既有 prettier 告警（与 0056/0070 报告基线一致）。本批新文件曾产生 8 条 prettier warning，`npx eslint --fix`（仅限本批 4 个新/改文件路径）清零；`npx eslint "src/views/iot/**" "src/api/iot/**"` 复核输出为空（EXIT=0）。

### 3. `cd frontend && pnpm typecheck` → EXIT=0

```text
$ vue-tsc --noEmit
TYPECHECK_EXIT=0
```

### 4. 改动范围核对：`git diff --name-only 1494e1e..7771181` → 恰 6 文件全在声明内

```text
frontend/src/api/iot/apiclient-api.ts
frontend/src/api/iot/publish-api.ts
frontend/src/views/iot/apiclient/components/apiclient-form-modal.vue
frontend/src/views/iot/apiclient/components/apiclient-secret-modal.vue
frontend/src/views/iot/apiclient/index.vue
frontend/src/views/iot/publish/index.vue
```

判定：6/6 全落 `frontend/src/**` 无越界；提交逐文件显式 add（未用 git add -A，工作区 `.agents/` 未跟踪目录零触碰）。

### 5. live 走查（8098 开工探测空闲，全量走查通过）——非缺席登记

开工探测：`curl 127.0.0.1:8098/auth/captchaImage` → 000 无监听；WSL mysql 3307 / redis 6379 在位 → 复核 jar 落后于 HEAD 提交后重建（`mvn package -DskipTests -pl ot-admin` BUILD SUCCESS，javap 核对 OpenApiClientController 五方法在包内）→ `dev.cmd start`（8098）+ `pnpm dev`（8099）→ 真实登录页登录 admin（验证码 ENME 截图识读，未绕过）→ 切换「智慧工厂」模块 → 走查 → 全停。

**（a）调用方管理页（4075，brief 命名的「开通→列表掩码→轮换→吊销」）**

| 走查项 | 结果 |
| --- | --- |
| 菜单与页面挂载 | 侧栏出现「调用方管理」（4075 动态注册），`/#/iot/apiclient` 直达正常，标签页「调用方管理」页签 ✓ |
| 列表掩码/状态/摘要 | 既有两行（0052 冒烟遗留）：mid_client_0052 掩码 `EAK7****tbK4`、test_client `test****`；scope JSON 摘要「实例×1（含子树）」；限流 100/8 次/分；已吊销红标/启用绿标；吊销行操作列正确切为「启用」、启用行「吊销」（条件渲染）✓ |
| **开通→明文仅一次** | 开通弹窗含 info 条「开通成功后 secret 明文仅展示一次（…任何接口不再返回）」→ 提交 smoke_0082_client（scope={"instanceIds":[2],"subtree":true}）→ **一次性弹窗**：appKey+43 字符明文 `bx3ZJRVR3F3WXu8_4uxY28NTL0QyIOmOuLeJaOh7i5s` 各带复制按钮+notice 告警条+「我已妥善保存，关闭（关闭后无法再次查看）」；**列表同步刷新**出新行 id=5 且掩码 `bx3Z****7i5s`（前四后四与明文吻合——明文不出列表）；点关闭后 DOM 快照断言「弹窗关闭+明文零残留」（关闭即弃）✓ |
| **轮换** | 确认框：「…轮换后新 secret 明文仅展示一次（须立即复制保存）；旧 secret 在并行窗口（默认 7 天）内仍然有效，过窗自动失效。」→ 确认 → 新明文弹窗（新明文 `hKNfGiiq…BoTiQ` **与旧明文不同**）+ **轮换专属并行窗 info 条**「轮换并行窗：旧 secret 在并行窗口（默认 7 天）内仍然有效，窗口过窗后自动失效；请用新 secret 替换调用方侧配置」+ toast「轮换成功：新 secret 已生效（旧 secret 并行窗内仍有效）」；关闭后列表掩码更新为 `hKNf****oTiQ`、最近轮换时间从「从未轮换」变 `2026-11-22 22:15:53`、明文零残留 ✓ |
| **吊销** | 确认框：「吊销立即生效：该调用方后续请求将立即返回 403（无缓存空窗）。非物理删除，行保留审计。」→ 确认 → toast「吊销成功：该调用方后续请求已立即 403」+ 行状态变「已吊销」、操作列变「启用」✓ |
| **编辑（M3 规避）** | 编辑弹窗 appKey **禁用**（创建后不可改）+info 条「编辑不触达 appKey/secret/状态」→ 先保存 ipWhitelist="10.0.0.1,10.0.0.2"+expireTime=2027-01-01（DB 核对两值落库 ✓）→ 再**清空两字段**（IP 清空输入、到期时间经清除图标置空——UI 清空成立）→ 保存 → toast「修改成功」→ **sys_operate_log param 实证**：id=5598 `{"id":5,…,"ipWhitelist":"",…}`（**显式空串**，且与 5597 的非空值对比成立）、5598 无 expireTime 键（后端审计序列化 NON_NULL——载荷构造恒含 `expireTime: form.expireTime \|\| null`，代码级恒键）；DB 复核 ip_whitelist/expire_time 保持旧值=**后端 trimToNull 吞掉显式空值**（即 0052 M3 登记的「SaveForm 空值文案与实现不符」后端侧现状，前端规避面已完成）✓ |
| 明文全链不泄露 | 列表/编辑弹窗全程仅掩码；明文仅出现在一次性弹窗存活期内 ✓ |

**（b）发布页回滚按钮（4015 版本行操作列，brief 命名的「回滚确认框→POST→成功刷新」）**

| 走查项 | 结果 |
| --- | --- |
| 未选范围守卫 | 查询/发布双 disabled + 空表「暂无数据」（既有前置未被破坏）✓ |
| 版本列表 | scope=计算 查询出 v5/v4/v3/v2（0070 批二冒烟遗留），每行操作列「下发状态 查看快照 **回滚**」在位 ✓ |
| **回滚确认框** | 点 v3 行「回滚」→「确定要回滚到版本 v3 吗？**回滚=以该版本旧快照生成新版本并重新下发，历史版本不被改写（新版本快照复用旧快照文件）**。」（ADR-0008 文案）✓ |
| **载荷取行数据** | 首次确认（v3 快照文件缺，见偏离 3）后端审计实证 `{"scopeType":2,"scopeId":0,"targetVersion":3}`——三元组取自版本行数据 ✓ |
| **错误态** | 目标快照缺失 500 → 全局拦截器报错（sys_operate_log id=5599 status=0 如实留痕），前端无崩溃可继续操作 ✓ |
| **回滚成功全链** | 环境数据对齐后重试 → toast「回滚成功：已以 v3 快照生成新版本并重新下发」→ **列表刷新出现新版本行 v6（id=34），其快照哈希 `b89df2060ed1a…c22bd0` 与 v3 行完全一致**（新版本行 snapshot_uri/snapshot_hash 复用旧快照文件——DB 核对 v3/v6 的 snapshot_uri 同为 `1/2-0-3.json`），v3/v4/v5 历史行零改动；操作日志 id=5600 status=1（成功回滚审计）✓ |

**（c）权限位**：admin 全权限会话（与 0070 批二同口径不做无权角色 live）；v-privilege 逐按钮静态核对——页面按钮 add 4076、编辑/轮换 edit 4077、吊销/启用 remove 4078、页面行 list 4075（菜单已实证可达）、回滚 rollback 4073，与 sys_menu 种子/控制器 @SaCheckPermission 一一对应。

**（d）全停**：`dev.cmd stop`（PID=2740 已停止）+ 前端 dev 任务终止 → netstat 复查 8098/8099 无 LISTENING；残留进程核查仅剩他方 workbuddy node（非本任务所起，未触碰）。

## 三、未做项与原因

| 项 | 原因 |
| --- | --- |
| 路由文件改动（router/iot.ts、router/index.ts） | brief 改动范围声明含路由配置授权，实测**不需要**：菜单 4075（path=/iot/apiclient，component=iot/apiclient/index）后端种子已在位，走 buildRoutes `import.meta.glob` 动态注册（与 4015 发布页同链路），live 实证页面可达——不建无菜单行的静态路由（0055 范式静态位仅用于无后端菜单项页面） |
| 无权限角色的按钮隐藏/403 live 走查 | admin 全权限会话；拦截一律后端（REQ-0034 口径），v-privilege 逐按钮静态核对（0070 批二同口径） |
| appKey 撞键 4002 live 复现 | 后端单源逻辑（0052 单测+冒烟已覆盖 4002 文案），前端走全局拦截器无特判需求；不再重复构造 |
| 启用（status=1）live 点击 | 吊销→启用按钮条件渲染已 live 实证（已吊销行显示「启用」）；启用动作与吊销同端点对称（PUT /status），未重复点击 |
| scope 授权树形选择器（树内勾选生成 JSON） | 0052 报告 §3 已登记归前端后续增强，TASK-0082 brief 未列——本页 scope 为 JSON 文本编辑+前置合法性校验（brief「scope JSON 编辑器」文字形态落地） |

## 四、偏离计划之处

1. **路由零改动**（见未做项 1）：brief 把「前端路由/菜单挂载配置」列入改动范围，实测 4075 菜单行走纯动态注册即可达，未改 router 两文件——属「授权范围内做了更少的事」，登记供收尾复核。
2. **M3 规避的边界实况**（重要登记）：前端已完整规避（编辑表单两字段可清空+提交显式空值不省略键+audit param 实证 `ipWhitelist":""`），但 **OpenApiClientServiceImpl 编辑分支对 ipWhitelist/expireTime 用 `StrUtil.trimToNull`/null 判空——显式空值到达后端仍被吞掉（DB 不清空）**，即 0052 M3 所指「SaveForm 空值文案与实现不符」的后端实现现状未修（backend 范围，本任务不可触）。前端侧规避完成后，「清空 ip_whitelist/expire_time 真正落库」需后端把这两字段的三态改为「空串=清空」（scope 已是三态可清空，无需改）。已登记遗留 1。
3. **回滚首次失败属环境数据错位（非前端缺陷）**：dev.ps1（jar，cwd=backend）的快照存储 `backend/data/snapshots/1/` 缺 scope 2-0 文件——0070 批二用 `mvn spring-boot:run`（cwd=backend/ot-admin）产生的 v2-v5 快照文件落在 `backend/ot-admin/data/snapshots/1/`。处置：把 4 个 `2-0-*.json` 复制到 server cwd 存储（纯 dev 库夹具对齐动作，非代码改动、非主仓文件）。前端在失败轮的行为（确认框→行数据载荷→500 全局报错→可继续操作）本身就是错误态 live 证据（操作日志 5599 status=0 留痕）。
4. **表单弹窗明文传递形态**：开通/轮换响应 SecretVO 由页面 index 持有并传给一次性弹窗（brief「响应含 appKey+secret 明文，明文仅一次展示弹窗+复制按钮+关闭即弃」按此落地）；弹窗关闭即置 null，不写 localStorage、不进任何持久状态。
5. **walkthrough 操作面备注（非代码偏离）**：本环境 IAB 对 antd 菜单/按钮的 Playwright locator click 频繁 actionability 超时，走查改走 dom_cua 节点点击/坐标点击完成；所有断言仍以页面 DOM 快照与 DB/操作日志为准，不影响证据有效性。浏览器截图两次卡死由重开读态绕过（未影响走查）。

## 五、遗留问题

1. **后端 M3 残留（本任务前端规避已闭环，后端面待修）**：编辑接口 ipWhitelist/expireTime 显式空值被 `trimToNull`/null 判空吞掉——「清空授权范围」以外的两个字段无法经管理面清到 DB（DB 实证：清空保存后 `ip_whitelist=10.0.0.1,10.0.0.2`、`expire_time=2027-01-01 00:00:00` 保持原值）。建议后续后端小任务把编辑分支两字段改三态（scope 同款）；前端载荷已按显式空值发送，后端修复后无需再动前端。
2. **dev 库冒烟遗留数据**：调用方 smoke_0082_client（id=5，当前已吊销态、ip_whitelist/expire_time 非空——恰为 M3 实证现场）；iot_config_version 新增 v6（id=34，快照复用 1/2-0-3.json）；sys_operate_log 新增调用方管理 5 行（5594-5598）+配置发布 2 行（5599 失败/5600 成功，均如实留痕）；backend/data/snapshots/1/ 补入 4 个 2-0-*.json（夹具对齐）。照 0015/0052/0070 冒烟遗留同口径隔离未清理，如需清理由收尾方处置。
3. **无权限角色显隐的 live 断言缺席**：同 0070 批二登记口径（admin 全权限），v-privilege 五权限位静态核对通过；后续如有低权角色测试账号可补 live。
4. **`.agents/` 未跟踪目录**：开工前已存在，全程未触碰未提交。
5. **一次性明文弹窗的复制按钮在自动化环境未点击验证**（copyable 为 antd 既有范式，0070 快照抽屉同款已在批二 live 验证复制可用）；人工使用面无差异。

**页面-接口映射核对表（静态证据：前端调用 vs controller 注解逐条对齐）**

| 前端调用 | 后端端点 | 权限位（sys_menu） | 对齐 |
| --- | --- | --- | --- |
| apiclientApi.queryClientList → GET /openapi/client/list（pageNum/pageSize/appKey/status） | @GetMapping("/list") Result\<Page\<OpenApiClientVO\>\>（只回掩码/轮换时间） | openapi:client:list 4075（页面行承载） | ✓ |
| apiclientApi.saveClient → POST /openapi/client（无 id=开通/带 id=编辑，载荷恒含全部可编辑键） | @PostMapping Result<?>（OR(add,edit)+分支精查；开通响应 SecretVO 明文仅一次；编辑白名单字段） | openapi:client:add 4076 / openapi:client:edit 4077 | ✓ |
| apiclientApi.rotateClientSecret → POST /openapi/client/rotate（{id}） | @PostMapping("/rotate") Result\<OpenApiClientSecretVO\>（新明文仅一次+并行窗） | openapi:client:edit 4077 | ✓ |
| apiclientApi.updateClientStatus → PUT /openapi/client/status（{id,status 0\|1}） | @PutMapping("/status")（吊销立即 403） | openapi:client:remove 4078 | ✓ |
| apiclientApi.queryClientDetail → GET /openapi/client/{id} | @GetMapping("/{id}")（本页未消费——列表行已含全部展示字段，封装保留给详情扩展） | openapi:client:list 4075 | ✓（未消费不建空壳调用） |
| publishApi.rollbackConfig → POST /iot/publish/rollback（{scopeType,scopeId,targetVersion} 取行数据） | @PostMapping("/rollback") Result\<IotConfigVersion\>（回滚=新版本行复用旧快照；目标版本不存在 400） | iot:publish:rollback 4073 | ✓ |

---

## 六、一句话结论

**通过**——三门 0 error（build:test EXIT=0 / lint 0e9w=基线 / typecheck EXIT=0）、live 走查全链绿（开通明文仅一次弹窗+关闭即弃断言、列表掩码、轮换并行窗双提示+掩码/轮换时间更新、吊销「立即 403」确认、M3 编辑清空支持+显式空值提交 sys_operate_log 5598 实证、回滚确认框 ADR-0008 文案+行数据载荷+**v6 新版本快照哈希与 v3 完全一致的历史不改写实证**）、提交 7771181 的 6 文件全落 frontend/src/** 无越界；唯一遗留为后端侧 M3 字段清空实现（前端规避已完成并登记，非本任务范围）。
