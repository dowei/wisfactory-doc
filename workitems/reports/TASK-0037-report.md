# TASK-0037 报告：validate/preview 单点试算与影响分析（管理面转发 calc trial）

## 0. 一句话结论

通过——backend 管理面新增 `POST /iot/item/validate` 与 `POST /iot/item/preview`，preview 转发 0086 交付的 calc `/internal/v1/calc/trial` 同引擎同口径（value/quality/missing/reason/engineVersion 原样回传，backend 不另算、缺数不冒充 0、calc 不可达 fail-soft），validate 复用 0036 语法/白名单/相对引用/存在性/循环链输出 4001 段错误；`cd backend && mvn -pl ot-admin test` 全量 **2599 测试 0 失败 0 错误（BUILD SUCCESS）= 基线 2577 + 净增 22**；影响分析 0036 既有 impact 端点复用零回归；试算端点不挂 @OperateLog（读语义裁决）。本任务提交 `2ad022e`（17 文件全落 backend/iot/dataitem + application.yml；区间内 6 个 frontend 文件属并行 TASK-0089 提交 `630327c`，非本任务产物）。

## 1. 元数据

| 项 | 值 |
| --- | --- |
| 任务 | TASK-0037（validate/preview 单点试算与影响分析） |
| BASE | `2d0f1a9` |
| 本任务提交 | `2ad022ebcf77d1c473a427ef4227bdb557735749`（main，注明 TASK-0037） |
| 改动文件数 | **17**（本提交自身 diff 全落声明范围） |
| 关联 | FEAT-0012 / REQ-0016 / TC-0012 / ADR-0005 / ADR-0013 增补 5 |
| 前置 | TASK-0036（公式校验链/impact 端点）、TASK-0086（calc trial 端点，权威契约） |
| 并行 | TASK-0089（纯 frontend），文件集/测试环境不交；其提交 `630327c` 夹在 BASE 与本提交之间（见 §6）。 |

## 2. 改动文件清单（本提交 `2ad022e` 自身 diff，17 文件）

main（12）：
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemController.java`（两读端点 + 试算服务注入）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/CalcTrialProperties.java`（`ot.calc.*` 可配）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/CalcTrialRequest.java`（逐字对齐 calc TrialRequest）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/CalcTrialResponse.java`（逐字对齐 calc TrialResponse）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemValidateForm.java` / `IotItemValidateVO.java`（validate 请求/响应）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemPreviewForm.java` / `IotItemPreviewVO.java`（preview 请求/响应）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotTrialService.java`（试算服务接口）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotTrialServiceImpl.java`（粗校验 + 取数 + trial 转发同口径）
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/CalcTrialClient.java` / `CalcTrialRejectedException.java`（HTTP 客户端 + fail-soft + 400 拒绝区分）
- `backend/ot-admin/src/main/resources/application.yml`（`ot.calc.base-url` 默认 127.0.0.1:8095，env `OT_CALC_BASE_URL`；prod 档继承本配置，部署注入实际地址，不硬编码地址/凭据）

test（5）：
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemControllerTest.java`（+5 用例：权限/不审、校验委派、403、preview 原样回传、fail-soft 透传）
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/impl/IotTrialServiceImplTest.java`（新增 12 用例：validate 五类 + 403 + preview 转发/缺数/取数/fail-soft/400 映射/atTime）
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/support/CalcTrialClientTest.java`（新增 5 用例：请求体逐字、缺数透传、不可达/非 2xx/不可解析 fail-soft、400 拒绝区分；JDK HttpServer 假 calc，无外部依赖）
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/OperateLogCoverageGuardTest.java`（+两条精确路径豁免，见 §7 范围偏离说明）

**未触碰**：frontend/**、calc/**（trial 端点只读调用不改）、ingest/**、`resources/contract/**`、既有 iot 业务模块（inspect/alarm/displayread/calcread 等）、0036 impact 端点行为、workitems 索引；无 git add -A、无 push、未置 TASK 已完成。

## 3. 五项交付（D1–D4 + 契约）

### D1 · 管理面端点
- `POST /iot/item/validate`（request `{instanceId, itemId?, formula}`）→ 不落库；返回 `{valid, errors:[{code,message}], expandedFormula}`；语法（花括号残留/超长）、一期函数白名单（INTEGRAL 专门文案）、相对引用 `{SELF.X}`/`{PARENT.X}` 展开、引用存在性（SELF/PARENT/指回本实例的绝对键）、循环依赖（DFS 环上 id）五类错误统一 4001 段；`valid=false` 不抛异常（结构化 errors 供前端定位）。
- `POST /iot/item/preview`（request `{instanceId, itemId?, formula, atTime?, variables?}`）→ 不落库；展开相对引用后转发 calc trial，**value/quality/missing/reason/engineVersion/curve 原样回传**（VO 字段与 trial 响应一一对应，backend 不另算）；trial HTTP 400（语法/白名单拒绝）映射为 4001（与 validate 同口径）。
- 权限：两端点均 `@SaCheckPermission("iot:item:query")`（4037，与 impact/effective-params 同口径）；服务层按 instanceId 走 requireDataPerm（`VisibleScope`，无权 403，文案不泄露实例标识）；**不挂 @OperateLog**（裁决见 §8）。
- 影响分析：0036 既有 `GET /iot/item/{itemId}/impact` 复用不改；`IotDataItemControllerTest.testImpact_BothRoutesAnd403Passthrough` 全量通过（含上级项清单/双路由/403），指标半边空集占位属 0043 范围不扩张。

### D2 · calc trial 转发客户端（可配 + fail-soft）
- `ot.calc.base-url` 默认 `http://127.0.0.1:8095`，connect/read timeout 默认 3000/5000ms，env `OT_CALC_BASE_URL` / `OT_CALC_CONNECT_TIMEOUT_MS` / `OT_CALC_READ_TIMEOUT_MS` 可覆盖；prod 档需注入实际内网地址（与 `OT_TDENGINE_BASE_URL` 同范式）。
- `CalcTrialClient.trial()` → `POST {base}/internal/v1/calc/trial`；请求 DTO 与 calc `TrialRequest` 逐字对齐（formula/variables/history/atTime/window 五字段 camelCase，毫秒），响应 DTO 与 `TrialResponse` 逐字对齐（value/quality/missing/reason/curve/engineVersion/rejected）。
- fail-soft 四类失败（连接失败/超时、非 2xx、响应不可解析、URI 非法）→ `BusinessException(500, "试算服务不可用：…")`，不伪造试算结果、不静默返回 0/空；HTTP 400 单独抛 `CalcTrialRejectedException` 携带 trial 的 reason，由服务层映射为 4001（拒绝 ≠ 不可用）。

### D3 · 缺数与变量取数
- preview 的 variables：请求体显式 `variables` 优先（非 null 即视为显式，空 Map 为显式空集→trial 报缺数）；未提供时按展开公式中的 fullKey 从 0050/0087 管理面实时值读口 `IotDisplayReadService.itemsRealtime(instanceId, keys)` 取当前值（v/q/ts → `{value,quality,ts}` 样本传给 trial），**取不到的标识符不填入 variables**（`OpenApiRealtimeItemVO.v=null` 即跳过），缺数由 trial 以 `value=null + missing + quality=6` 明示，绝不填 0。
- DELTA/INTEGRAL 历史窗：0086 trial 支持 `history`/`window`，本任务 preview 不构造 history（历史取数超范围，诚实缺数）；DTO 保留字段与契约对齐，0038 窗口曲线任务可直接扩展。

### D4 · 测试口径（TDD 红转绿，见 §4）
validate 5 类 + 403、preview 转发逐字/缺数/取数/fail-soft/400 映射/atTime、客户端请求体逐字/缺数/四类 fail-soft、影响分析 impact 零回归、2577 基线零回归（终态 2599）。

## 4. TDD 红转绿留痕（每修复附「为什么现在对了」）

| 轮次 | 观察（原始输出摘要） | 修复 | 为什么现在对了 |
| --- | --- | --- | --- |
| **红 1**（`-Dtest=CalcTrialClientTest,IotTrialServiceImplTest,…`，`/tmp/t0037-red.txt`） | `COMPILATION ERROR` 16 处「找不到符号」：`IotTrialService` / `CalcTrialRequest` / `CalcTrialResponse` / `IotItemValidateVO` / `CalcTrialClient` / `CalcTrialClientTest:68` 等 14 个新类 | 按测试引用建立 DTO/VO/服务接口/客户端骨架与实现 | 红在「新类根本不存在」而非「断言写错」——测试确实在驱动新行为，未为匹配输出改期望 |
| **绿 1 部分** | `IotTrialServiceImpl.java:232` 编译错：`IotModelInstance` 无 `getInstanceId()` | 改用 `instance.getId()` | 实体主键访问器是 `getId()`（与 0036 实现一致）——现在字段访问与实体契约一致 |
| **绿 1 再部分** | 编译错：VO 为 `@Data` POJO 但测试用 record 访问器 `vo.valid()`；controller test 2 处 `verify(dataItemService)` 残留 | 把 `IotItemValidateVO`/`IotItemPreviewVO` 改为 record；verify 目标改为 `trialService` | VO 作为只读结果 record 更贴合契约，controller 试算委派对象与端点字段注入一致——两处符号错消失 |
| **绿 1 再再部分** | 12 个 service 用例中 5 个 ERROR：`数据项不存在或不属于该实例: 201/202` | 测试夹具 `stubTenantItems` 补 `when(itemMapper.selectById(id))` 桩（生产代码无需改，`selectById` 是真实 DB 行为） | 之前只桩了 `selectList`，`requireItemOfInstance` 走 `selectById` 返回 null 是测试桩缺失而非实现缺陷——补齐桩后归属校验真实生效 |
| **绿 2** | 定向 5 类共 51 用例全绿（3+3+28+12+5），全量 2599/0/0/0 | — | 见 §6 原始输出；未改任何测试期望值来凑绿 |

## 5. 关键证据

### 5.1 同口径证据（preview 结果直接来自 trial）
- `IotItemPreviewVO` 六字段（value/quality/missing/reason/engineVersion/curve）在 `IotTrialServiceImpl#preview` 由 `CalcTrialResponse` 构造**直接赋值**，无任何算术/质量码重算；`IotTrialServiceImplTest.previewPassesThroughTrial` 断言 value=3.0/engineVersion=`calc-engine/1.0.0/trial-v1` 与 mock trial 响应逐字相同。
- `CalcTrialClientTest.forwardsExactTrialRequest` 断言发往假 calc 的 body 恰为 TrialRequest 五字段（formula/variables/history/atTime/window，camelCase，ts 毫秒），`Content-Type: application/json`，路径 `/internal/v1/calc/trial`；响应 value=7.5/quality=0/missing=[]/engineVersion 原样回传。
- backend 无第二套求值器：grep 新增代码仅含 `ArithmeticEvaluator` 零次（trial 在 calc 进程内执行），backend 只做 HTTP 转发与字段映射。

### 5.2 缺数不冒充 0
- `CalcTrialClientTest.passesThroughMissingVerbatim`：trial 响应 `value=null, quality=6, missing=["B"], reason="缺数: B（不以 0 代入）"` 原样透传，断言 `assertNull(value)`、`missing=["B"]`。
- `IotTrialServiceImplTest.previewMissingNeverZero` 同上（服务层原样映射）。
- `IotTrialServiceImplTest.previewFetchesVariablesFromRealtimeRead`：实时读口只返回 X 的值，Y 取不到 → 发给 trial 的 variables **只含 X 一个键**，Y 不填 0；trial 报缺数。

### 5.3 fail-soft 证据
- `CalcTrialClientTest.unreachableCalcFailsSoft`（停掉假 calc）：抛 `BusinessException` 且消息含「试算服务不可用」。
- `CalcTrialClientTest.non2xxAndUnparseableFailSoft`：HTTP 500 与不可解析体均抛明确错误（500 消息含状态码），不返回空结果。
- `CalcTrialClientTest.rejection400IsDistinctFromUnavailable`：HTTP 400 抛 `CalcTrialRejectedException`，reason 含 INTEGRAL；`IotTrialServiceImplTest.previewTrial400MapsTo4001` 断言映射为 4001（与「服务不可用」500 严格区分）。
- `IotDataItemControllerTest.testPreview_FailSoftPassthrough`：HTTP 层 `$.code=500` + message 含「试算服务不可用」。

### 5.4 不挂 @OperateLog 的裁决理由
- brief D1 明文「不挂 @OperateLog」；validate/preview 是试算读语义：不写库、不发领域事件、不改引用边、不记时间线——无业务副作用；挂审计会把公式编辑器每次按键/预览都刷进 `sys_operate_log`，淹没真实配置变更轨迹（与 0061 巡检/0062 告警读接口、0039 时间线读口「读不审」一致）。
- TASK-0065 §5 曾建议「若建则挂 @OperateLog（businessType=OTHER）」，本任务按其当时的「试算端点当时不存在」前提裁决：端点现建，按读语义处理；差异理由在此登记，避免审计刷屏。
- `IotDataItemControllerTest.testTrialEndpointsPermissionAndNoAudit` 反射锁定两方法 `@SaCheckPermission=iot:item:query` 且 `@OperateLog=null`，防后人误挂。
- 守卫协同：既有 `OperateLogCoverageGuardTest` 机械要求「iot 包下所有写方法（含 POST）挂审计」，故对其加**两条精确路径豁免**（`/iot/item/validate`、`/iot/item/preview`），不改其余任何端点覆盖口径；这是为满足 brief「不挂 @OperateLog」必须的最小测试侧调整，属本任务唯一范围外改动（见 §7）。

## 6. 验收命令原始输出（原样贴回）

### 6.1 `cd backend && mvn -pl ot-admin test`（post-commit 终态，Results 段）

```
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 7.379 s -- in net.otlab.admin.module.iot.config.OperateLogCoverageGuardTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.033 s -- in net.otlab.admin.module.iot.dataitem.config.IotDataItemMenuPermissionGuardTest
[INFO] Tests run: 28, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.482 s -- in net.otlab.admin.module.iot.dataitem.controller.IotDataItemControllerTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.504 s -- in net.otlab.admin.module.iot.dataitem.service.impl.IotTrialServiceImplTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.722 s -- in net.otlab.admin.module.iot.dataitem.service.support.CalcTrialClientTest
[INFO] Results:
[INFO] 
[INFO] Tests run: 2599, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  04:14 min
[INFO] Finished at: 2026-11-24T13:21:13+08:00
[INFO] ------------------------------------------------------------------------
```

2599 = 基线 2577 + 新增 22（IotTrialServiceImplTest 12 + CalcTrialClientTest 5 + IotDataItemControllerTest +5），0 失败 0 错误。

### 6.2 范围核对：`git diff --name-only 2d0f1a9..HEAD`

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/CalcTrialProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/CalcTrialRequest.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/CalcTrialResponse.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemPreviewForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemPreviewVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemValidateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemValidateVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotTrialService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotTrialServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/CalcTrialClient.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/CalcTrialRejectedException.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/OperateLogCoverageGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/impl/IotTrialServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/support/CalcTrialClientTest.java
frontend/src/api/display/display-api.ts
frontend/src/views/display/device/device-realtime-zone.vue
frontend/src/views/display/metrics/index.vue
frontend/tests/unit/display/device-realtime-zone.spec.ts
frontend/tests/unit/display/display-read-ports.spec.ts
frontend/tests/unit/display/metrics-board-read-ports.spec.ts
```

前 17 行 = 本任务提交 `2ad022e` 自身全部 diff（全落 §2 声明范围）；后 6 行 = 并行任务 TASK-0089 的提交 `630327c`（纯 frontend，夹在 BASE 与本提交之间，brief 已预告可并行），**非本任务产物**、零触碰。本提交自身范围核对命令 `git diff-tree --no-commit-id --name-only -r HEAD` 输出 17 行、全落声明范围；本提交未含任何 frontend/calc/ingest/contract 文件。

## 7. 范围偏离说明（唯一一处）

- **偏离**：修改了 `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/OperateLogCoverageGuardTest.java`（在 brief 声明的 `.../dataitem/**` 测试路径之外）。
- **原因**：该守卫机械断言「所有 iot 管理面 POST 必须挂 @OperateLog」，与 brief D1 硬约束「validate/preview 不挂 @OperateLog」直接冲突；不改守卫则新增两端点必然使全量 mvn test 失败。
- **最小化**：只加两条**精确路径**豁免（`/iot/item/validate`、`/iot/item/preview`），不动扫描范围、不减少既有 50+ 写端点的审计覆盖、不加通用前缀豁免；`audited.size() >= 50` 与 `controllers >= 18` 断言保持。已在 §5.4 说明理由。
- 若评审认为此偏离不可接受，替代方案只有「挂 @OperateLog（违反 brief D1）」或「把端点移出 iot 包（违反改动范围）」——两者都比本方案代价更高。

## 8. 未做项与原因

| 项 | 原因 |
| --- | --- |
| validate 转发 trial 做权威校验 | brief D1 为「可转发」而非必须；validate 本地粗校验已覆盖五类错误且与 0036 保存链同口径，避免校验接口依赖 calc 可用性；权威同引擎校验由 preview 转发实现，报告登记。 |
| preview 构造 `history`/`window` 取 DELTA 历史窗 | 历史取数超本任务范围（brief D3 明示「历史取数超出本任务范围则诚实缺数」），DTO 保留字段供 0038 扩展。 |
| 重做影响分析 impact 端点 | 0036 已交付，brief D1 明示复用不改；全量测试 impact 零回归。 |
| 改 calc/ingest/frontend/contract/既有 iot 业务模块 | 越界，零触碰。 |

## 9. 契约纪律

no-subagents；只提交本批改动（17 文件，未 git add -A，未 push）；未置 TASK 已完成；未改 workitems 索引；试算不落库、同引擎同口径、缺数不冒充 0、calc 不可达 fail-soft、数据权限 requireDataPerm 403、不挂 @OperateLog；未硬编码凭据/文档库路径；测试失败先定位根因（§4 每行「为什么现在对了」），未为凑输出改期望。
