# TASK-0061 diff 评审（首评）

- 评审角色：独立代码评审（fresh / no-subagents / 只评审不修改）
- 评审对象：`60c4568` 单提交隔离 diff 包（`TASK-0061-diff.txt`，25 文件，BASE=`640c9bd`，区间夹入 0084 的 `3e09775` 已按提交隔离）
- 口径：L1-execute §13「diff 评审」；标准任务首评，1 退回 + 1 复评上限
- 结论：**退回执行者一轮**——发现 1 项 Important；其余按 Minor 记 deferred。无 Critical。

---

## 一句话总评

四类指标看板与只读端点的空态/租户隔离/安全红线/DDL 静态契约/测试与范围登记均按 D1–D5 真实落地（在线率 N/A、跨租户 403、Redis 锁释放、DDL 未执行逐行切分均核实无误），但 D2 巡检任务的核心清单产出（离线清单 / CALC_ERR / 完整度）是「按类型空集落库」的桩实现，三类聚合纯函数全仓无生产调用，真机每次运行只稳定落「待核验」占位 + 空集，未满足 REQ-0031 验收 2，构成 Important 退回项。

---

## 发现清单

### [Important] I-1 · 巡检任务未真正产出三类清单，只落空集/占位，REQ-0031 验收 2 未满足

- **出处**：`InspectTaskRunner.java` 的 `safeCalcErr()`（`return List.of()`）、`safeCompleteness()`（`return List.of()`）、`inspectTenant()` 中 `onlineSnap = List.of()`；`InspectPayloads.java` 的 `groupCalcErrors()`/`approximateLowCompleteness()` 在全仓 `src/main/java` 仅定义、无生产调用（仅 `InspectAggregatorTest` 单测调用）。
- **说明**：报告第五节把「无会话租户 → fail-soft 空集」登记为边界，但本任务新增了 `tenant_id` 循环（`activeTenants()` 查 `sys_tenant`）却仍以「无会话租户」为由让 CALC_ERR / 完整度永远为空；`CalcReadService` 确有 `requireSessionTenant()` 会话门禁，但 0063 已提供 `DataPermService.resolveClientScope(clientId, tenantId, scopeJson)` 显式租户重载能力，且巡检**根本未注入 `CalcReadService`/`DataPermService`**（构造器只有 `store` + `jdbc`），失败原因是设计上没接数据源，不是外部 REST 故障——与 brief D2「TDengine REST 故障才允许占位」的授权条件不同。离线清单同样无 `statusread`/`iot:st` Redis 消费者，第 4 类 `TENANT_DB_MISMATCH` 则是无条件写死 `待核验`（无 REST 探测尝试）。看板三 tab 因此在真机长期空/占位，TC-0020 步骤 3「三类清单产出」不成立。
- **判定**：验收 2 + brief D2.1/D2.2/D2.3 未满足，按 §13 Important → 退回一轮。
- **建议修复方向**：在 runner 注入 `CalcReadService`/`DataPermService` 并用 `resolveClientScope` 或会话上下文桥接显式租户，调用 `groupCalcErrors`/`approximateLowCompleteness` 真实产出 CALC_ERR / 完整度；离线清单接 statusread current 的 `iot:st` Redis 聚合计数（禁逐点扫 TDengine）；`TENANT_DB_MISMATCH` 增加真实 REST 探测，仅探测失败时落 `待核验`。若确需延期，应在报告中如实标为「未实现而非故障占位」，由协调者裁决返工归属。

### [Minor] M-1 · 看板计算成功率对受限角色恒为 N/A，且未按 fullKey 实例段裁剪

- **出处**：`InspectReadServiceImpl.safeCalcState()`：`if (scope.allTenant()) return rows; return List.of();`；`collectCalcLogs()` 使用 `calcReadService.log(fullKey, ...)` 依赖内部会话裁剪，`dashboard()` 对 fullKey 无显式校验。
- **说明**：req 目标角色含租户管理员；0063 `CalcReadServiceImpl.visible()` 本可按 fullKey 第 3 段实例码 + `instanceIds` 正确裁剪，但此处对所有非 allTenant（含租户管理员）直接返回空集，成功率卡永久 N/A（安全方向 fail-closed 无泄露，但可用性受损）。属保守口径而非越权，记 deferred；后续可用 `CalcStateVO.getFullKey()` 实例段解析 + `DataPermService` 可见集裁剪替代一刀切空集。

### [Minor] M-2 · history 分页先 `LIMIT page*size` 再内存 `subList`，跨页会重复/漏数据

- **出处**：`InspectReadServiceImpl.history()` 调用 `store.history(tenantId, type, fromMs, toMs, page * size)`，随后 `rows.subList((page-1)*size, ...)`。
- **说明**：SQL 每次只取前 `page*size` 行，page 2 取前 100 行再切 [50,100)，page 3 同样需 ≥150 行才正确；当总行数 < page*size 时页面截断，且无 `OFFSET`，多页浏览结果重叠/缺失。当前前端固定 `pageNum=1`（`index.vue` loadHistory），线上未暴露，记 deferred；建议 store 接 `OFFSET/limit` 或改游标分页。

### [Minor] M-3 · 报告「0 ingest 引用」措辞与 diff 包有 2 处 ingest 注释提及不一致

- **出处**：`TASK-0061-report.md` 第五节「0 ingest 引用」；diff 包 `InspectScheduler.java`/`InspectTaskScheduler.java` 头注各有「与 ingest（fail-soft）同口径」字样。
- **说明**：仅为注释措辞，0 ingest/**文件**改动与提交隔离（25 文件、0084 的 `3e09775` 单独提交）均已核实无误，不构成越界，记 deferred 消除歧义即可。

### [Minor] M-4 · 菜单种子缺租户隔离，`iot:instance:query` 权限对运营方/租户管理员边界未细化

- **出处**：`20261124_iot_inspect.sql` 的 sys_menu 171 种子 `perms='iot:instance:query'`，sys_role_menu 仅超管授权；brief D4/D5 明确沿用该码。
- **说明**：实现与 brief 一致（不新建权限位），但 REQ-0031 角色口径「运营方可看健康用量、业务明细仅租户管理员」在单一权限码下无法区分——同码用户既能看四类指标也能看 history/latest 明细。属计划/权限模型层缺口，非本任务实现错误，记 deferred 由后续权限设计处理。

---

## 逐条评审重点核对（均独立核实源码/契约原文）

1. **四类指标口径（D1）**：
   - 在线率：`InspectPayloads.onlineRateText(0,0)=(3,0)="N/A"`、`dashboard()` 无快照→N/A、`DashboardVO.onlineRate` 默认 `"N/A"`、前端 `helper.ts`/`iot-inspect-helper.spec.ts` 同口径。**判定：N/A 非 100%，成立**。
   - 计算成功率：`calcSuccessRateText(空)="N/A"`，基于 0063 calc state→fullKey→log，按 `resultStatus==1` 成功计数（与 iot_calc_log DDL「1 成功 2 失败」一致），无日志 N/A、不造 100%。
   - 通道健康取 `iot_channel.status`（既有 iot_channel_domain DDL 核实），指标产出取 `iot_metric`，均无物理 completeness 列时 `metricBasis` 标注近似口径。D1 通过。
2. **巡检任务（D2）**：
   - `InspectTaskScheduler` 使用 `@Qualifier("rawStringRedisTemplate")` + `setIfAbsent(key,token,ttl)`（SETNX），TTL=周期×2；`releaseLock()` `get` 比对 token 一致才 `delete`（防误删），Redis 不可用 try/catch WARN fail-soft 跳过、`@Scheduled(fixedDelayString/initialDelayString)` 复用 `PublishSchedulingConfig` 的 `@EnableScheduling`（源码核实该类仅此一个 config，未重复建）。**Redis 重入锁正确释放：成立**。
   - 三类清单/比对：`TENANT_DB_MISMATCH` 诚实标注 `result=待核验` + 原因「TDengine REST 当前故障」+ `expectedDb=iot_<code>`，不误导为一致/不一致、不伪造 0/100%（brief D2.4 允许，成立）；CALC_ERR/完整度/离线为**空集占位**——其诚实性（空数组 + `item_count=0` + 报告登记）OK，但构成 I-1 功能缺口。
   - 巡检只写 `iot_inspect_result`（store.save）+ 归档删除，全仓 grep 确认无业务表写操作。
3. **落库 90 天归档（D3，ADR-0014）**：`iot_inspect_result` 7 字段 + `idx_tenant_time_type` 复合索引，无 create_by/update_by/update_time/deleted/version（DDL 契约 test 锁定）；归档 `DELETE WHERE inspect_time < ?`（Java 侧算 cutoff，方言无关），参数化、幂等、retention 可配 90；DDL 全仓仅被 `IotInspectDdlSplitContractTest` 静态读取，**未在 phpStudy MySQL 执行**（grep 无执行脚本引用），`sql-exec.sh` 切分 `;\s*\n` 与契约测试逐行切分口径一致（3 段、0 同行多语句）。通过。
4. **只读端点与租户隔离（D4）**：三 GET 仅 `@GetMapping`，零写映射、零 `@OperateLog`（反射断言）；`requireTenantAccess()` 从 `TenantContext.getTenantId()` 取会话 tenant，请求参数带不同 tenant→403（code 403 核实于 `ResultCode`），无租户上下文→403；history 受限 scope 无可见实例→空集 fail-closed；枚举在 `TenantContext`/`SecurityUtils`/`DataPermService.resolve` 均核实签名存在。**跨租户 403 / tenant 会话强制注入：成立**。但 `safeCalcState()` 对受限角色一刀切空集见 M-1。
5. **前端监控页（D5）**：四类指标卡 + 四 tab + 历史回看齐备；空态 `a-empty`/N/A（无点位 N/A、无数据 a-empty），在线率前端不现算、不把 N/A 换成 100%；api 仅 `getRequest('/api/v1/iot/inspect/**')`，**不持 appKey/secret、不直连 openapi、不自造代理**（与 display-api.ts 头注同红线，grep 无违规命中），菜单 id=171 避开 168-170（01_base.sql 核实 168/169/170 已用）、挂 110、组件路径/权限码正确。通过。
6. **测试有效性与零回归**：surefire 原始报告核实 2482 用例 0F0E（21 新增 inspect 用例逐类 4/5/3/3/1/5 全绿）；前端 `build:test`/`lint`/`typecheck` 脚本名与 package.json 一致，9 warning 为存量、本任务 4 文件 0 warning（报告声明）；全量 diff 无 `System.out`/`console.log`/`debugger`/`TODO/FIXME` 调试残留；0063/0050/0045/既有测试零改动、contract 冻结目录零触碰。通过（报告实测输出与此一致）。
7. **范围与登记**：`git show --name-only 60c4568` 恰 25 文件全在 brief 声明范围；`git diff 640c9bd..HEAD` 30 文件 = 25 + 0084 的 5 ingest 文件，后者在单独提交 `3e09775`（父为 640c9bd）、`60c4568` 父为 `3e09775`，diff 包 0 ingest 文件引用（2 处仅注释措辞，见 M-3）；HEAD 漂移登记属实。通过。

---

## 越界反查

`git diff --name-only 640c9bd..HEAD` 中声明集外仅 5 个 ingest 文件，全部归属并行提交 `3e09775`（TASK-0084 修订），与 0061 提交 `60c4568` 无交集（其父即 `3e09775`），diff 包无 ingest/**文件改动。`resources/contract/**`、`iot/publish|calcread|statusread|metric` 既有代码、framework upstreams、`01_base.sql` 均零改动。**结论：无越界。**

---

## 契约遵守

- fresh / no-subagents / 只评审不修改（除本报告外未改任何文件、未推进状态、未派子代理）：已遵守。
- 对特别关注点的明确判定：
  - **在线率 N/A 非 100%：通过**（后端纯函数 + 看板默认 + 前端 helper + 双端测试四重一致）。
  - **跨租户 403 / tenant 会话注入：通过**（`requireTenantAccess` 会话取值 + 参数不一致 403 + 无上下文 403，code=403 源码核实；但注意这是 read 端，巡检任务侧因 I-1 未真正使用租户数据）。
  - **Redis SETNX 重入锁正确释放：通过**（setIfAbsent + token 比对 DEL + fail-soft）。
  - **DDL 只产出未执行、逐行切分：通过**。
  - **前端不碰 appKey/HMAC：通过**。
  - **三类清单空集占位对看板展示的影响**：空集/待核验本身诚实标注（不算 0/100%、不误导为「一致」），但作为巡检任务的**唯一产出**未满足验收 2 → 计入 I-1 退回项，而非仅登记。

---

## 处置建议

按 §13：I-1 为 Important → 退回执行者一轮修复（仅需在巡检域补真实取数/聚合，改 `InspectTaskRunner`，必要时加显式租户重载契约测试）；复评 scoped 只看 I-1 修复 diff + 原发现。M-1～M-4 记 TASK `任务结果` 的 deferred，随收尾进 LOG，不进修订轮。本轮评 1 Important / 0 Critical / 4 Minor。
