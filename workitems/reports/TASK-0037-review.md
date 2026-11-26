# TASK-0037 代码评审报告（diff 评审 · 标准首评）

## 结论

**通过**。无 C/I（阻断 / 重要）发现；记 1 条 Minor（deferred，审计守卫硬化建议，不影响当前正确性）。在标准首评「1 退回 + 1 复评」上限内，本轮无需退回执行者。

评审对象：`2ad022e` 单提交隔离的 17 文件 diff（BASE=`2d0f1a9`；区间夹入的 TASK-0089 提交 `630327c` frontend 已隔离）。经 `git diff-tree --no-commit-id --name-only -r 2ad022e` 独立核实：本提交自身仅含 17 个 backend 文件，零 frontend / calc / ingest / contract 文件，范围口径与报告一致。

评审方法：回到源码与权威契约独立核实，不采信报告自述；对三项头号重点（同口径、缺数/fail-soft、守卫豁免最小性）逐项给出明确判定；独立重跑全量测试复现数据。

---

## 发现清单

| 级别 | 出处 | 一句话说明 |
| --- | --- | --- |
| Minor（deferred） | `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/OperateLogCoverageGuardTest.java:69-70`（`EXCLUDED_EXACT_PATHS` 声明）与 `:190-199`（`isExcluded`） | 豁免键只取路径做 `Set.contains`、未绑定 HTTP 动词，未来若有人在同一路径再挂一个未审计的 PUT/DELETE/PATCH 写端点会被误豁免；当前无任何此类端点，两条 POST 豁免与「新增其他 POST 无审计仍被抓」的硬要求均已满足，故仅作硬化建议（改 `verb+path` 复合键），不构成本轮退回。 |

除上项外未发现其它问题。下列「登记偏离」均为 brief 明文许可或已在实施报告充分说明，不计为发现。

---

## 头号重点判定

### ① 同口径 — 判定：通过

- **preview 直接转发 calc trial、backend 不另算**：`IotTrialServiceImpl.preview`（`.../service/impl/IotTrialServiceImpl.java:540-570`）展开相对引用后调 `calcTrialClient.trial(request)`，该客户端 `CalcTrialClient.java:732` 以 `POST {base}/internal/v1/calc/trial` 发出；返回字段由 `new IotItemPreviewVO(response.value(), response.quality(), response.missing(), response.reason(), response.engineVersion(), response.curve())` 逐字段直接赋值，无任何算术 / 质量码重算。对新代码 grep `ArithmeticEvaluator|OperatorEvaluator|DeltaOperator|evaluate` 在 backend dataitem 主代码零命中，确认无第二套求值器。
- **DTO 逐字对齐权威契约**：逐字段比对 `CalcTrialRequest`/`CalcTrialResponse` 与 0086 权威 `calc/.../trial/TrialRequest.java`/`TrialResponse.java`——`formula/variables/history/atTime/window` 五字段与 `TrialWindow(long,long,long)`、`VariableSample(double,int,long)` 完全一致；响应七字段 `value(Double)/quality(int)/missing/reason/curve/engineVersion/rejected` 与 `CurvePoint(long ts,Double value,int quality,List missing,String reason)` 完全一致；camelCase、毫秒（`Long atTime` / `long ts`）一致。缺省构造语义（null→`Map.of()`/`List.of()`/`""`）亦与 calc record 一致。
- **mock calc HTTP 服务非恒真断言**：`CalcTrialClientTest.java:1399-1429` 用 JDK `HttpServer` 起假 calc，捕获真实请求体后断言路径、`Content-Type`、`formula`、嵌套 sample 的 value/quality/ts（毫秒）、`atTime` 及 `actual.size()==5` 字段集，并断言响应逐字段回传——是真实 HTTP 往返 + 逐字断言，非同值赋值恒真。服务层 `IotTrialServiceImplTest.previewPassesThroughTrial` 以 ArgumentCaptor 断言公式/变量/回传值，两层证据闭合。

### ② 缺数不冒充 0 + fail-soft — 判定：通过

- **缺数显式、value 不为 0**：`CalcTrialResponse`/`IotItemPreviewVO` 保留 `value=null + quality=6 + missing` 原样；`CalcTrialClientTest.passesThroughMissingVerbatim`（`:1432-1442`）与 `IotTrialServiceImplTest.previewMissingNeverZero`（`:1231-1246`）均断言 `assertNull(value)`、`quality=6`、`missing=["B"]`、`reason` 含「不以 0 代入」。
- **variables 取不到不填 0**：`IotTrialServiceImpl.resolveVariables`（`:573-600`）对 `sample==null || v==null` 一律 `continue` 不放入；`previewFetchesVariablesFromRealtimeRead` 断言实时读口只返回 X 时，发给 trial 的 variables `size()==1` 仅含 X、Y 不填 0。显式 variables 非 null 即优先（空 Map 为显式空集，由 trial 报缺数）。
- **calc 不可达 fail-soft 不造假**：`CalcTrialClient.trial`（`:731-779`）对序列化失败、URI 非法、连接/超时、中断、非 2xx（400 除外）、应答不可解析全部抛 `BusinessException(500, "试算服务不可用：…")`，`unavailable()` 仅记 WARN、不返回任何默认结果。测试覆盖不可达/500/不可解析/400 四类且 400 单独抛 `CalcTrialRejectedException` 并由服务层映射 4001，拒绝与不可用严格区分。**未发现伪造结果、静默 0 / 空返回路径。**

### ③ 越界反查：审计守卫豁免最小性 — 判定：通过（安全相关，未只看绿灯）

连带改动仅 `OperateLogCoverageGuardTest.java` 一处（+11 行），独立核实结论：

1. **豁免精确到两条 POST 路径**：新增 `EXCLUDED_EXACT_PATHS` 为 `HashSet` 恰含 `"/iot/item/validate"`、`"/iot/item/preview"` 两个完整路径字面量；`isExcluded` 用 `contains(path)` 精确相等，非前缀、非正则。逐一核对全仓 class 级 `@RequestMapping` 与方法级映射，`IotDataItemController` 无类级前缀（方法持全路径），且无任何其它控制器能经 `basePath+firstMappingPath` 拼出这两条路径——豁免与两条新增端点一一对应。
2. **未放宽其他端点覆盖**：`AUDIT_PACKAGE_PREFIXES`、`EXCLUDED_PATH_PREFIXES` 未动；`controllers>=18`、`audited.size()>=50`、`assertEquals(empty, missing, …)` 三条断言原样保留，扫描范围与既有 50+ 写端点审计覆盖不减。
3. **守卫仍机械有效**：`isExcluded` 仅在 `missing` 判定前短路；任何其它未挂 `@OperateLog` 的新 POST（路径不等于这两条）仍进入 `missing` 使测试失败。独立全量运行中该守卫 3 用例全绿，未因豁免而失去拦截能力。

结论：该越界改动确为满足 brief D1「不挂 @OperateLog」硬约束、同时保持守卫绿的最小测试侧调整，范围外影响为零。上表 Minor 仅针对「动词未纳入豁免键」的理论硬化空间，不改变本判定。

---

## 其余重点判定

- **validate 校验链**：`validate`（`:415-535`）复用 0036 的 `FormulaFunctionWhitelist`/`RelativeRefExpander`/`FormulaCycleDetector`，覆盖函数白名单、`{SELF.X}/{PARENT.X}` 展开、残留花括号终检、引用存在性、循环依赖五类，错误统一收 `4001` 段 `errors` 不抛异常；管理面粗校验 + preview 转发 trial 权威校验的组合成立。
- **端点与权限**：两端点均 `POST`、`@SaCheckPermission("iot:item:query")`、服务层按 instanceId `requireDataPerm`（无权 403，文案「无权访问该实例（数据权限不命中）」不泄露实例标识）；`testTrialEndpointsPermissionAndNoAudit` 反射锁定权限码与 `@OperateLog=null`。不挂 @OperateLog 系 brief D1 明示裁决，与 TASK-0065 §5 早期建议的差异已在实施报告 §5.4 逐条登记，接受。0036 impact 端点零回归（全量测试中 `testImpact_BothRoutesAnd403Passthrough` 通过）。
- **TDD 与零回归**：报告 §4 的红相为「新类不存在」编译错（非改期望凑绿），留痕可信；本轮独立重跑全量验证通过（见下）。

### 登记偏离（非发现）

| 项 | 性质 |
| --- | --- |
| validate 未转发 trial 权威校验 | brief D1 为「可转发」（非必须），实施报告 §8 登记理由（避免校验接口依赖 calc 可用性；权威同引擎校验由 preview 承担），许可范围。 |
| 端点不挂 @OperateLog | brief D1 硬约束，报告 §5.4 说明与 TASK-0065 §5 的差异理由，许可范围。 |
| 修改 `OperateLogCoverageGuardTest.java` | 唯一声明范围外连带，最小性已在上文独立核实，报告 §7 登记，接受。 |
| 未构造 history/window（DELTA 历史窗） | brief D3 明示历史取数超范围则诚实缺数，DTO 保留字段供 0038，许可范围。 |

---

## 独立验证证据

本评审自行执行 `cd backend && mvn -pl ot-admin test`（未采信报告输出）：

```
[INFO] Tests run: 3,  Failures: 0, Errors: 0, Skipped: 0 -- ...OperateLogCoverageGuardTest
[INFO] Tests run: 28, Failures: 0, Errors: 0, Skipped: 0 -- ...IotDataItemControllerTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0 -- ...IotTrialServiceImplTest
[INFO] Tests run: 5,  Failures: 0, Errors: 0, Skipped: 0 -- ...CalcTrialClientTest
[INFO] Results:
[INFO] Tests run: 2599, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

- 复现报告数值：2599 = 基线 2577 + 净增 22（service 12 + client 5 + controller 5）；0 失败 0 错误 0 跳过。
- 范围核对：`git log --oneline` 与 `git diff-tree` 确认 2ad022e 与 0089 的 630327c 提交相互独立，本提交 17 文件全落 backend/iot/dataitem 与 application.yml 声明范围。

---

**最终结论：通过**——三项头号重点（同口径、缺数/fail-soft、守卫豁免最小性）均给出明确通过判定，无 C/I 需要退回；唯一 Minor（守卫豁免键未含动词）记为 deferred 硬化建议。
