# TASK-0070 实施报告 · 批次一（下发状态页 + 批量导入向导 + 凭据管理）

- 执行者：implementer（frontend 独立道，brief 唯一输入：`TASK-0070-brief.md`）
- BASE：`f195b58`；HEAD：`d02ac44`（main，提交注明 TASK-0070 batch1）
- 验收口径：三门（build:test / lint / typecheck）+ 联调冒烟（8098 开工探测**空闲**，全量走查）+ 改动范围核对

---

## 一、改动文件清单（8 文件，全部落在 brief 声明范围 frontend/src/**）

`git diff --name-only f195b58..HEAD` 实测恰 8 文件（=第五条验收命令原始输出），无越界；提交 `d02ac44` 只含下述文件（git add 逐文件指定，同期工作区 backend/ TASK-0015 在途改动零触碰）。

| 文件 | 新/改 | 内容 |
| --- | --- | --- |
| `frontend/src/api/iot/publish-api.ts` | 新 | /iot/publish 三读写：queryVersionList（GET /list，scopeType+scopeId 必传）、queryPublishLogList（GET /log/list，versionId 或三元组）、resendPublishLog（POST /resend）。**不提供 publish/rollback**（确认门待 TASK-0015，本批不做不建占位） |
| `frontend/src/api/iot/credential-api.ts` | 新 | /iot/channel/credential 四读写：list（channelId 必传）、write（写入/轮换）、view/{id}（明文唯一出口）、DELETE /{id}（吊销）。头注登记权限双轨（4046/4051/4052）与「永不返回密文」 |
| `frontend/src/api/iot/point-api.ts` | 新 | 点位**导入三接口**（0022）：template 下载 / multipart import（silentError 自渲染回执）/ error-report 回传下载。点位列表/详情等其余接口随点位管理页（4011）升级批补建，本文件只含导入向导所需 |
| `frontend/src/views/iot/publish/index.vue` | 改 | 4015 占位升级完整页：发布范围过滤（scopeType=1 网关下拉选 id / 2 计算归一 0，未选不发请求+提示语）+ 版本列表分页（version/快照哈希 copyable/发布状态/发布时间）+ 行操作「下发状态」打开抽屉 |
| `frontend/src/views/iot/publish/components/publish-log-drawer.vue` | 新 | 下发状态抽屉（挂版本上下文，versionId 定位）：ack_status 三态徽标（0 待确认橙/1 已生效绿/2 失败红）+ timedOut=true 加「已超时」红标 + errorMsg 红字 tooltip + ackTime；版本级 publishState 汇总徽标（0 下发中/1 全部生效/2 存在失败）；重发按钮（iot:publish:resend）确认框，**已生效行置灰 + tooltip「已生效无需重发」**；行集不分页（每版本目标数≤3）；刷新 |
| `frontend/src/views/iot/import/index.vue` | 改 | 4016 占位升级三步向导（a-steps）：①上传（模板下载+dragger，iot:point:add）→ ②回执明细（成败计数 4 tag + persisted 落库态 + **全部行逐行回执表（失败不中断）** + 错误行下载）→ ③结果汇总（统计 descriptions + 落库状态 + 重新导入） |
| `frontend/src/views/iot/channel/index.vue` | 改 | 通道页行操作新增「凭据」入口（v-privilege iot:channel:list——凭据列表接口同权限位；操作列 130→180）+ 挂接 ChannelCredentialDrawer + 头注补记 |
| `frontend/src/views/iot/channel/components/channel-credential-drawer.vue` | 新 | 凭据管理抽屉（挂通道行）：列表只展示类型/掩码/key_version/状态（**永不展示密文**）；查看明文（v-privilege iot:credential:view）二次确认含「审计留痕」提示 → 明文弹窗带警示横幅、关闭即弃（前端状态不驻留）；写入/轮换（iot:credential:edit）表单（四类型字典+明文 ≤1000）+ 同类型重复写入=轮换确认；吊销确认框（已吊销行按钮不渲染）；写操作 catch 统一接 handleIotConflictError |

未动文件：router（0055 口径菜单动态注册，4015/4016/4013 组件路径已在种子）、views 非 iot 目录、backend/**、ingest/**、resources/**、workitems 库（本报告除外）。

---

## 二、验收命令原始输出

### 1. `cd frontend && pnpm install` → 跳过（说明）

node_modules 在位且 package.json 依赖未变（与 0056 基线一致），按 brief「如依赖未变则跳过并说明」跳过。

### 2. `cd frontend && pnpm build:test` → EXIT=0（最终态复跑；首轮亦 0 error）

```text
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. ...(基线既有提示)
vite v5.2.12 building for test...
transforming...
./icon.svg referenced in .../vue3-json-viewer/dist/index.css didn't resolve at build time, ...（基线既有 warning）
[plugin vite:esbuild] src/views/survey/components/FormAuthModal/panel.vue?...: This assignment will throw because "editors" is a constant（存量 survey 文件 esbuild 提示，非本任务文件）
...（chunk 产物清单中略）
(!) Some chunks are larger than 500 kB after minification. ...（chunk 大小 warning，基线既有可忽略）
✓ built in 54.09s
BUILD_EXIT=0
```

### 3. `cd frontend && pnpm lint` → EXIT=0，0 error / 9 warning（=开工基线）

```text
✖ 9 problems (0 errors, 9 warnings)
  0 errors and 9 warnings potentially fixable with the --fix option.
LINT_EXIT=0
```

9 条 warning 全部为 tests/unit/** 既有文件 prettier 格式告警（与 0056 报告基线逐文件相同）；本任务 iot 新增文件曾产生的 8 条 prettier warning 已 `eslint --fix` 清零，`npx eslint "src/views/iot/**" "src/api/iot/**"` 复核输出为空（0 问题）。

### 4. `cd frontend && pnpm typecheck` → EXIT=0

```text
$ vue-tsc --noEmit
TYPECHECK_EXIT=0
```

### 5. 改动范围核对：`git diff --name-only f195b58..HEAD` → 恰 8 文件全在声明内

```text
frontend/src/api/iot/credential-api.ts
frontend/src/api/iot/point-api.ts
frontend/src/api/iot/publish-api.ts
frontend/src/views/iot/channel/components/channel-credential-drawer.vue
frontend/src/views/iot/channel/index.vue
frontend/src/views/iot/import/index.vue
frontend/src/views/iot/publish/components/publish-log-drawer.vue
frontend/src/views/iot/publish/index.vue
```

判定：8/8 全落 `frontend/src/**`，无越界；backend/ 在途改动（TASK-0015 影响预览道，工作区可见）未触碰、未纳入提交。

### 6. 联调冒烟（brief 第 4 条）→ 8098 探测空闲，**全量走查通过**（非缺席登记）

开工探测：`curl 127.0.0.1:8098/auth/captchaImage` → 000 无监听、netstat 空 = 8098/8099 均空闲 → 按 brief 主选项执行全链冒烟：`mvn spring-boot:run -pl ot-admin`（8098，~30s 起）→ `pnpm dev`（8099，vite ready 3.3s）→ 真实登录页登录 admin（验证码 ZY7A 截图识读，未绕过）→ 切换模块「智慧工厂」→ 三页走查 → 全停。

**（a）下发状态页（4015 配置发布）**

| 走查项 | 结果 |
| --- | --- |
| 未选范围 | 查询按钮 disabled + 提示「请先选择发布范围」+ 表格空态「暂无数据」 ✓ |
| scope=网关 GW_T0020_A 查询 | 版本列表空态（共0条） ✓（**brief 命名的「下发状态空态」**） |
| scope=计算 查询 | 版本 v24 行：ID 24 / 版本号 1 / 快照哈希 copyable / 已发布绿标 / 2026-11-22 23:49:48 ✓ |
| 下发状态抽屉 | 配置版本 v1、发布范围「计算（租户级，scopeId=0）」、版本级汇总「全部生效」绿徽标（publishState=1） ✓ |
| 行展示 | 目标类型 计算 / 目标标识 calc / 回执状态「已生效」绿徽标 / 回执时间 2026-11-22 23:50:00 / 失败原因 - ✓ |
| 已生效行重发置灰 | 重发按钮 disabled=true + tooltip「已生效无需重发」 ✓（DOM 断言 disabled: [true]） |
| 截图证据 | 抽屉全图（含灰化重发）已存 artifacts |

**（b）凭据管理（4013 通道页 → 凭据抽屉）**

| 走查项 | 结果 |
| --- | --- |
| 入口 | 通道行（CH_T0019_M1，网关 GW_T0019_01 下拉必选）操作列「详情 凭据 编辑 删除」——「凭据」入口在位 ✓ |
| 掩码展示（**brief 命名的「凭据掩码」**） | MQTT_PASSWORD → `T19-****wxyz`（v2，已吊销）/ DEVICE_TOKEN → `T19-****7766`（v1，启用）——**全程无密文/明文出现在列表** ✓ |
| 已吊销行 | 「吊销」按钮不渲染（仅 status=1 行有）✓ |
| 查看明文二次确认 | 确认框文案含「本次查看属高危操作，将被审计留痕（记录谁在何时查看了哪个凭据）」 ✓ |
| 明文弹窗 | 标题「凭据明文（已审计留痕）」+ 警示横幅「明文仅限本次查看，请勿复制外传」+ 解密明文正确回显（T19-device-token-99887766） ✓ |

**（c）批量导入向导（4016）**

| 走查项 | 结果 |
| --- | --- |
| 步骤条 | 上传文件 → 回执明细 → 结果汇总 三步渲染 ✓ |
| 模板下载/上传 | 「下载导入模板」按钮（iot:point:add）+ dragger 收集文件（IAB 文件选择不支持，经 DataTransfer 注入合成 xlsx——3 个无效行，openpyxl 生成，整批必拒零落库风险） ✓ |
| 逐行回执（**brief 命名的「导入向导回执」**） | 整批拒绝 alert「导入被整批拒绝…逐行回执明细如下」+ 计数 tag（参与校验 3 / 校验通过 0 / 校验失败 3 / **整批未落库**）+ 全部行明细表（失败不中断）：行 2 INVALID_DATA_TYPE「数据种类非法: 9」、行 3 MISSING_REQUIRED「必填字段缺失: pointName」、行 4 INVALID_CODE「点位编码不合规: bad code!」——与后端 0022 回执语义逐字一致 ✓ |
| 错误行下载 | 「下载错误行清单」点击 → download 事件触发成功（POST /iot/point/import/error-report）✓ |
| 结果汇总 | 统计 descriptions + 落库状态「整批拒绝（任何行未入库）」红标 + 警示 alert + 重新导入按钮 ✓ |
| 截图证据 | 汇总步骤全图（含步骤条 1✓2✓3、拒绝 toast）已存 artifacts |

**（d）全停**：TaskStop 后子进程残留（java PID 2880 / node PID 56328），确认进程身份（本任务所起，8098 开工时无监听）后定点终止——netstat 复查 8098/8099 监听清空。未强杀任何他人进程。

**冒烟闭环的修复（走查发现 → 当场修复 → 三门复跑）**：首轮走查发现两个新抽屉组件漏写 `defineExpose`（`publish-log-drawer.vue`、`channel-credential-drawer.vue`），点击入口静默无响应——当场补 `defineExpose({ showModal })` 后抽屉全部正常打开。修复后三门（build:test EXIT=0 / lint EXIT=0,0e9w / typecheck EXIT=0）在**最终态**复跑全绿；该修复已含于提交 `d02ac44`。

---

## 三、未做项与原因

### 批次二登记清单（本批 brief 明示不做，不建占位代码）

| 未做项 | 归属 | 原因 |
| --- | --- | --- |
| 发布页影响预览 + 二次确认段（含发布/回滚按钮 4072/4073） | TASK-0015 交付后接入 | brief「本批不做」；api 层未提供 publish/rollback 封装，页面不建占位按钮 |
| 待升级清单页（逐台 diff/批量确认） | TASK-0016 | 后端未交付 |
| 调用方管理页（开通/scope/轮换） | TASK-0052 | 后端未交付 |
| 适配器管理页（注册心跳/状态） | TASK-0033+ | 后端未交付 |
| 指标重算发起与进度入口 | TASK-0047 | 后端未交付 |

### 本批内未做/未覆盖项

| 未做项 | 原因 |
| --- | --- |
| 快照取回入口（GET /iot/publish/snapshot，iot:publish:query 4071） | brief 三页组决策未含快照查看——版本列表仅展示哈希（copyable）；若协调者裁决需要「查看快照原文」入口，属 4015 页小增量，登记遗留 |
| point-api 完整 CRUD 封装 | 点位管理页（4011）升级不在本批三页组（点位 CRUD 后端 0021/0022/0023 已就绪，前端升级批次待排）——本文件只含导入向导三接口 |
| 重发按钮 ack=0/2 行的实际触发走查 | dev 库版本 v24 行 ack=1（无失败/待确认行可测）；重发会真实触发下发接缝 NotifySender 并重置回执，为不扰动 dev 库与并行道未实际 POST——确认框渲染与已生效置灰分支已验证 |
| 凭据写入/轮换与吊销的实际执行走查 | 会写 dev 库凭据（同通道同类型=轮换改既有夹具 key_version）——表单/确认框渲染已走查，写路径与 0056 弹窗范式同构，实际落库验证登记集成轮 |
| timedOut 标红的真实复现 | dev 库行 timedOut=false（超时窗内）；三态徽标与「已超时」附加红标为模板条件渲染，静态正确性由页面-API 映射表佐证 |
| TC-0018 全流程手工冒烟 | UAT 手工场景（步骤 2-5）依赖完整链路与角色夹具，本轮口径为三页走查；TC-0018 步骤 4「错误行回执呈现并可下载」已在本批向导冒烟中等价覆盖 |

---

## 四、偏离计划之处

1. **「凭据管理页」落地形态 = 通道页（4013）行操作「凭据」抽屉，非独立菜单页**：凭据无独立菜单行（sys_menu 仅 4051/4052 两按钮行且 parent_id=4013），列表接口 channelId 必传（按通道查）——独立页反而需要再造一个「通道选择器 + 空态」层。抽屉形态与 0056 的 instance-item-drawer 同范式，权限显隐（view/edit 按钮 v-privilege）不受影响。若协调者要求独立页面菜单，需后端补菜单种子（本批改动范围不含）。
2. **「下发状态页」落地形态 = 配置发布页（4015）版本列表 + 下发状态抽屉**：brief「按 versionId 或 scope 查询」——versionId 定位由版本行打开（行携带 id/scopeType/scopeId/version，即同时持有 scope 上下文，满足「挂发布/版本上下文」）；scope 三元组独立定位入口（无版本行直接查）未设——log/list 不分页且版本数有限，从版本列表进入覆盖全部场景。若需独立「下发状态」菜单页请裁决。
3. **导入向导对无回执形态的批级拒绝补报**：0056 dict-import-modal 对 silentError 后无 data 回执的拒绝（如配额 4003、超限 fail fast）静默无提示；本向导 catch 补 `message.error(body?.message)` 兜底（仅新增一行文案展示，非自造请求层）。dict-import-modal 本身未动（非本批范围）。
4. **凭据抽屉未 emit reloadList**：凭据操作只落 iot_channel_credential 行，通道列表数据不受影响——与 tenant-admin-modal（0056「无 emit 不影响列表」口径）一致。
5. **冒烟中走真实登录页**（验证码截图识读 ZY7A），未用 token 注入；通道/凭据/版本夹具全部复用 dev 库既有数据（0019/0025 冒烟遗留），本任务未新建业务夹具、未清退他人夹具。
6. **pnpm dev 起停**：8099 冒烟后连同 8098 一并全停（brief「全停」）；停服时发现 TaskStop 只杀父 shell，定点核实进程身份（8098 java / 8099 node，均为本任务所起）后终止。

---

## 五、遗留问题

### 页面-接口映射核对表（静态证据：前端调用 vs controller 注解逐条对齐）

**配置发布/下发状态页**（IotPublishController @RequestMapping("/iot/publish")）

| 前端调用（publish-api.ts） | 后端端点 | 权限位（sys_menu） | 对齐 |
| --- | --- | --- | --- |
| queryVersionList → GET /iot/publish/list（pageNum/pageSize/scopeType/scopeId） | @GetMapping("/list") Result\<Page\<IotConfigVersion\>\> | iot:publish:list 4070 | ✓ |
| queryPublishLogList → GET /iot/publish/log/list（versionId 或三元组） | @GetMapping("/log/list") Result\<PublishLogListVO\>（行不分页 + timedOut + publishState） | iot:publish:list 4070 复用 | ✓ |
| resendPublishLog → POST /iot/publish/resend（{versionId, targetId}） | @PostMapping("/resend") @Valid ResendForm（已生效 400） | iot:publish:resend 4074 | ✓ |
| （未消费）GET /snapshot | iot:publish:query 4071 | 预留 | ✓ 前端未封装（快照入口未做，见遗留 1） |
| （未提供）POST /publish、/rollback | iot:publish:publish 4072 / rollback 4073 | 预留 | ✓ 前端未封装（待 0015 确认门） |

**凭据管理抽屉**（IotCredentialController @RequestMapping("/iot/channel/credential")）

| 前端调用（credential-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| queryCredentialList → GET /iot/channel/credential/list?channelId | @GetMapping("/list") Result\<List\<CredentialItemVO\>\>（只含掩码） | iot:channel:list 4046 | ✓ |
| writeCredential → POST /iot/channel/credential（{channelId, credType, credential}） | @PostMapping @Valid CredentialWriteForm（SM4 落库/审计 saveParam=false） | iot:credential:edit 4051 | ✓ |
| viewCredential → GET /iot/channel/credential/view/{id} | @GetMapping("/view/{id}") Result\<CredentialViewVO\>（审计 saveResponse=false） | iot:credential:view 4052 | ✓（v-privilege 无权隐藏入口） |
| revokeCredential → DELETE /iot/channel/credential/{id} | @DeleteMapping("/{id}")（status 置禁用） | iot:credential:edit 4051 | ✓ |

**批量导入向导**（IotPointController 导入三接口，共用 iot:point:add）

| 前端调用（point-api.ts） | 后端端点 | 权限位 | 对齐 |
| --- | --- | --- | --- |
| downloadImportTemplate → GET /iot/point/import/template | @GetMapping("/import/template")（十三列+示例行+说明行） | iot:point:add 4053 | ✓ |
| importPoints → POST /iot/point/import（multipart，silentError） | @PostMapping("/import") @RequestParam("file") Result\<PointImportReceipt\>（拒绝 code=400 + data=回执） | iot:point:add 4053 | ✓ |
| downloadImportErrorReport → POST /iot/point/import/error-report（body=回执） | @PostMapping("/import/error-report")（失败行→xlsx，无状态） | iot:point:add 4053 | ✓ |

页面内跨域消费：publish/index.vue → gatewayApi.queryGatewayList（scope=1 网关下拉，pageSize 500 对齐 maxLimit）。

### 遗留问题清单

1. **快照查看入口未做**（iot:publish:query 4071 预留）：brief 三页组未含；需协调者裁决是否随批次二或 4015 页补增量。
2. **重发 ack=0/2 分支、凭据写入/轮换/吊销的实际落库走查未做**：交互与确认框渲染已验证，实际 POST 路径归集成轮（避免 dev 库扰动）；重点验证：重发后回执重置+NotifySender 触发、轮换 key_version 递增、吊销后内部拉取 403。
3. **timedOut 标红未真实复现**（dev 库无超窗行）：条件渲染静态佐证；集成轮可造逾窗行或等超时扫描置 2（两者 UI 均覆盖：timedOut 红标 / ack=2 红徽标）。
4. **lint 基线 9 条 prettier warning**（tests/unit/** 既有，与 0056 基线相同）：不属本任务改动范围，未修——后续工程化任务统一 lint:fix。
5. **独立菜单页形态裁决**（偏离项 1/2）：凭据抽屉挂 4013、下发状态挂 4015——如需独立页面需补菜单种子（backend 范围）。
6. **dev 库审计留痕增量**：冒烟在夹具凭据 id=2 上执行一次「查看明文」，sys_operate_log 新增一条留痕记录（功能预期行为，非脏数据）；其余走查全只读（导入用整批必拒文件，零落库）。
7. **workspace 未跟踪 `.agents/` 目录**：开工前已存在，未触碰未提交。

---

**结论：通过**（三门 0 error + 联调冒烟三页走查全过 + 改动范围 8/8 无越界；批次二待依赖交付后另行 brief）。
