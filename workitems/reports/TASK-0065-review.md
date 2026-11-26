# diff 评审：TASK-0065（操作审计接线机械覆盖收口）

- 评审对象：提交 `09357d1`（隔离区间 `04e6acd..09357d1`，**4 文件 / 365 insertions**）——diff 包 `TASK-0065-diff.txt`
- 评审口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」+ 评审 brief 七项重点
- 评审方法：fresh 全量读 diff 4 文件 + 必读决策/上游报告 + 工作区源码实读；**不采信未核实报告声明**——独立重跑 `audit-operatelog-coverage.py`（两次）、独立重跑守卫测试（3/3 绿）、独立重跑全量 `mvn test -pl ot-admin`（2373/0/0）；并用自写正则二次独立推导写端点差集（61/58/3，与脚本一致）；逐文件实读切面（OperateLogAspect）、注解定义、凭据控制器、3 个内部端点控制器、时间线接口与 NoOp 实现、0064 守卫范式、0052 评审口径。工作区 HEAD=`09357d1`，与 diff 一致。

---

## 一、评审重点逐项核对（brief 七项）

### 1. 差集正确性 —— 通过

- **独立复核**：`python resources/scripts/audit-operatelog-coverage.py` 实跑两次，输出一致——写端点总数 61、已标注 58、缺标注 3；管理面真实缺口 0。另以独立正则窗口法二次推导得 61 写方法 / 58 标注 / 3 缺（heartbeat、offlineEvent、ack），与脚本逐项一致。执行者报告 step②「56 已标注 + step④ 补 2 = 58」与终态自洽。
- **差集 5 项中 3 项 internal 排除**：3 项（`/internal/v1/gateway/heartbeat`、`/internal/v1/gateway/offline-event`、`/internal/v1/config/ack`）实读确为 `@Anonymous` + `InternalAuthChecker.check`（静态令牌 + IP 白名单），按 brief 决策 4 明文排除，且高频心跳落 sys_operate_log 会淹没审计面（源码注释与报告登记一致）——排除成立。
- **管理面真实缺口恰 2 项**：`IotDataDictController#importErrorReport`、`IotPointController#importErrorReport` 实为唯一两处未挂的管理面写端点，现已补；无遗漏（反射口径覆盖全部 21 个 @RestController，无继承、无方法级 `@RequestMapping(method=POST)` 漏扫——已逐一 grep 验证为空）。
- 说明：设计接口表全清单本身无法独立复核（设计 PDF 不在本工作区，find 零命中），但守卫测试为**代码全量反射扫描**、不依赖设计表，故「管理面写端点零缺失」结论可独立成立；设计侧端点缺位（validate/preview）见第 7 项。

### 2. 补线正确性 —— 通过

- 两处均加 `@OperateLog(title=…, businessType=BusinessType.IMPORT)`：方法与参数位置正确（`@Target(METHOD)`，挂在 controller 方法上）；形态与既有 import 端点逐字一致（`IotDataDictController.java:132`、`IotPointController.java:170` 同款）。
- 切面织入有效：`OperateLogAspect.java:61` `@Pointcut("@annotation(net.otlab.core.annotation.OperateLog)")` + `@AfterReturning/@AfterThrowing`，对新注解自动生效，本次切面零改动。
- saveParam/saveResponse 口径：两方法入参为导入回执（错误行清单，无凭据/敏感字段）、方法签名 `void` 写 xlsx 流——`@AfterReturning` 收到 result=null，`OperateLogAspect.java:141` 的 `result != null` 守卫使 response 不落库；无需仿 0052 M1 做敏感屏蔽（0052 涉及 secret，此处无）。补线后审计五要素（操作人/时间/url/参数/成败）留存充分。

### 3. 守卫测试有效性 —— 通过

- **全局断言真全局（范围内）**：`OperateLogCoverageGuardTest.java:66-113` 扫 `net.otlab.admin.module` 全部 `@RestController`，按 iot./openapi. 前缀收窄（21 控制器），按 POST/PUT/DELETE/PATCH + 类级 `@RequestMapping` 拼路径，排除 `/internal/v1/`、`/openapi/v1/`；阈值 `controllers>=18`、`audited>=50` 防扫描空转。实测独立复跑 3/3 绿。覆盖面核实：21 控制器全为独立类（无 `extends`）、无方法级 `@RequestMapping(method=...)`、无 `@RestController` 文件名异于 `*Controller.java`——反射口径无系统性盲区。
- **凭据四类断言真实有效**：`credentialFourActionsAudited`（:128-146）逐项读真实注解——write `businessType=INSERT` 且 `saveParam=false`、view `saveResponse=false`、revoke `UPDATE`、list 无注解；与 `IotCredentialController.java:68/79/88/64` 实态逐字吻合，非空泛口径。
- **TDD 红→绿**：红相 2 失败（全局差集 + 接线锁定）精确指向 2 缺口，与测试结构一致（credential 用例在红相应绿）；绿相 3/3。报告所述首轮 `TreeMap<Class>` 自陷已改 `LinkedHashMap`（:154），与 diff 一致。

### 4. 脚本质量 —— 通过

- `audit-operatelog-coverage.py` 可重复执行（ROOT 支持 `sys.argv[1]`），注释剥离保留字符串字面量（权限码不误删），注解窗口以 `}` 行截断，与独立正则推导结果一致。
- brief 点名的 0064 踩坑复核：三段/四段注解形态问题属 0064 权限码语境（本脚本读 Java 非 SQL，不适用）；INSERT...SELECT 形态同为 SQL 解析问题（不适用）。`OP` 正则要求 `@OperateLog(`——实读作用域内 60 处带括号、7 处「裸 @OperateLog」均在注释中，当前无误报/漏报。

### 5. 时间线分工结论 —— 通过

- 结论自洽：「sys_operate_log=谁/何时/成败/参数」vs「iot_data_item_history=业务值前后变化」，与 REQ-0034 核心规则、FEAT-0021 业务规则、TASK 描述、ADR-0006 一致；无双写——隔离 diff 4 文件零触碰 `iot_data_item_history`（实读仅 `IotDataItemHistoryService` 接口 + `NoOpIotDataItemHistoryService` 占位，TASK-0039 未交付），结论与源码一致。

### 6. 零回归 —— 通过（评审者独立复跑）

- 本评审独立执行 `mvn test -pl ot-admin` → **Tests run: 2373, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**，与报告声明完全一致（=0064 基线 2370 + 守卫净增 3）。
- 既有审计测试（0052 `OpenApiClientAuditNoPlaintextTest`、0064 `IotPermissionsCompleteGuardTest`、0019 `IotCredentialAuditNoPlaintextTest`、0051 `PermissionDeniedAuditListenerTest`）均在 2373 全量集合内且全绿，未受影响。

### 7. 执行者登记项 —— 通过（登记充分，不需额外动作）

- **validate/preview 管理面端点缺位**：实读全仓 controller 无 `/iot/item/validate`、`/iot/item/preview`（calc trial 为内部令牌链），确属「无对象可挂」。新增管理面端点属新业务能力、超出本任务「仅补 @OperateLog」范围；登记备查即可，无需本轮动作。
- **xlsx 下载 response 留空**：方法为 `void`，result=null 致 response 列 null（报告归因「切面 JSON 序列化不适用」方向对、机制表述略偏，见下方 M3）；审计主诉求（操作人/参数/成败/url）齐备，属框架既有能力边界，登记充分。

---

## 越界反查复核 —— 与协调者结论一致

- 隔离提交 `09357d1`（直接父 `04e6acd`）`git show --name-only` 恰 4 文件：2 controller 各 +1 行注解、1 守卫测试、1 比对脚本，全落 brief 声明范围。
- OperateLogAspect 核心、`sys_operate_log` schema、`iot_data_item_history`、前端、calc、ot-core 零触碰（隔离 diff 实证）。
- BASE=`537366e`..HEAD=`09357d1` 区间含并行道 `04e6acd`（TASK-0040 calc/ 交错），评审以隔离区间为准，与 brief 一致。

---

## 二、发现清单

无 Critical、无 Important。以下 Minor 不阻塞收尾（§13：记 deferred，进收尾 LOG，不进修订轮）：

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| M1 | Minor | `D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0065-report.md:18, :102, :192`；对照源码 `IotGatewayInternalController.java:93-99`、`IotPublishInternalController.java:63-71`、`StaticTokenInternalAuthChecker.java:57-83`、`PermissionDeniedAuditListener.java:41-46`、`GlobalExceptionHandler.java:133-137` | 报告称 3 个排除内部端点「各有自己的审计链（iot_api_call_log / 403 全量）」缺乏源码支撑：`iot_api_call_log` 仅由 `/openapi/v1/**` 机器验签链（OpenApiAuthFilter）写入；内部端点认证失败抛 `BusinessException`，不发布 `PermissionDeniedEvent`，故不走 403 审计监听器。排除本身仍由 brief 决策 4 + 高频淹没口径成立，仅支撑论证措辞不准，建议报告订正。 |
| M2 | Minor | `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/OperateLogCoverageGuardTest.java:88-110, :152-165` | 「全局」断言依赖 `getDeclaredMethods()` + 阈值 `controllers>=18/audited>=50`，若未来出现 `*Endpoint.java` 命名、方法级 `@RequestMapping(method=POST)` 或继承方法会静默漏扫——当前三种情形均已 grep 验证为空、覆盖真实充分，属未来健壮性提示非现存缺陷。 |
| M3 | Minor | `TASK-0065-report.md:136, :216`；对照 `OperateLogAspect.java:141` | report 将 xlsx 端点 `response=null` 归因于「切面 JSON 序列化不适用」，实际机制是 `void` 方法使 `result=null` 触发 `result != null` 短路（未进入序列化分支）；结论（response 留空、审计主字段留存）正确，建议订正因果表述。 |

---

## 三、结论

**通过**。

- 七项评审重点逐项核实：差集 61/58/3 经脚本与独立正则双证一致，3 项 internal 排除成立、管理面真实缺口恰 2 项且无遗漏；补线注解形态/位置/切面织入/敏感口径与既有 import 端点一致；守卫测试为范围内真全局反射扫描且凭据四类断言真实、TDD 红→绿证据自洽；脚本可重复且无误报；时间线分工与源码一致无双写；全量回归由评审者独立复跑 2373/0/0 BUILD SUCCESS。
- 越界反查一致：隔离提交 4 文件全落声明范围，切面核心/schema/前端/calc/ot-core 零触碰。
- 发现 3 条 Minor（M1-M3），无 Critical / Important——按 §13 记 deferred 收尾进 LOG，不进修订轮；其中 M1/M3 为报告论证/因果措辞订正，M2 为守卫测试未来健壮性提示，均不影响交付正确性。
