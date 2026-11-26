# TASK-0065 实施报告：操作审计接线与时间线分工核查（机械覆盖审计）

> **中断恢复重派——前一执行者零落地，本次从零执行。**

- BASE：`537366e`（开工时主仓 HEAD，已确认，即 TASK-0064 提交）
- 本任务提交：`09357d1`（main，注明 TASK-0065；4 文件 365 insertions，全落 brief 声明范围）
- 结论先行：**五条验收全部通过**。全量 `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → **Tests run: 2373 = 基线 2370（TASK-0064 实测）+ 净增 3（本守卫测试 3 用例），0 失败 0 错误，BUILD SUCCESS**；TDD 红→绿（红相 2 失败精确指向 2 个未接线端点）；机械差集比对**管理面写端点零缺失终态**（61 写端点，58 已标注，3 个内部令牌端点按 brief 决策 4 排除）；dev 冒烟 4/4 + sys_operate_log 行实查（admin / 参数 / 状态 / 凭据查看 response=NULL 无明文）；prod 档真实启动 20.8s + **ERROR=0** + liveness 200；本人提交 4 文件全落声明范围。
- 审计切面行为零改动（只挂 @OperateLog 注解）；未碰 iot_data_item_history / sys_operate_log schema / 前端 / calc / ot-core。

---

## 0. 机械覆盖审计四步执行实录（照 0064 差集法）

| 步 | 动作 | 结果 |
| --- | --- | --- |
| ① 提取设计写操作接口全清单 | pypdf 提取详细设计 §3.1/续（p21-23）+ §3.4/§3.5（p26）接口表；POST/PUT/DELETE 类为准 | 网关/通道/点位/模板/实例/数据项/指标/发布/字典/凭据/状态规则/调用方管理共 12 类；只读 GET 不审；内部令牌（/internal/v1/**）与对外数据（/openapi/v1/**）端点排除（brief 决策 4） |
| ② grep 现库各 controller 写端点 @OperateLog 标注集 | `resources/scripts/audit-operatelog-coverage.py`（纯机械，类级 @RequestMapping 前缀拼接 + 注解窗口识别，注释行剥除） | iot/openapi 共 21 控制器、61 个写端点（POST/PUT/DELETE/PATCH），56 个已挂 @OperateLog |
| ③ 差集 = 缺审计项 | 机械比对 | **恰 5 项缺标注**：`/iot/dict/import/error-report`、`/iot/point/import/error-report`（管理面真实缺口 2）；`/internal/v1/gateway/heartbeat`、`/internal/v1/gateway/offline-event`、`/internal/v1/config/ack`（内部令牌端点 3，brief 决策 4 明确排除——各有独立审计链：iot_api_call_log / 403 全量） |
| ④ 逐项补接线 | 两个 importErrorReport 端点各加一行 `@OperateLog(title=…, businessType=BusinessType.IMPORT)`（既有切面直接挂，不改切面本身） | 复跑脚本：**管理面真实缺口 = 0，零缺失 PASS**；只标注的 2 处各 +1 行 |

**设计接口表 → 现库端点映射说明**（沿用 0064 命名映射登记）：设计 `iot:config:publish/list/rollback` ≡ 在库 `iot:publish:*`（TASK-0024 既成命名）；设计 `/iot/item/validate`、`/iot/item/preview`（公式校验/试算，POST）在库无对应端点（归内部 `/internal/v1/calc/trial`，TASK 未建管理面转发端点——见 §5 遗留）。设计 `/iot/channel/credential` 三类写 + 一类明文查看在库 4 动作全量 @OperateLog（0019 交付实态，见 §2.3 凭据核查）。

**前后值口径**（brief 决策 2）：sys_operate_log 框架既有列（操作人/时间/请求参数/响应）——**未扩展 schema 加前后值列**（实查 schema 25 列无 before/after 类列，见 §2.3）；「前后值」业务语义归 iot_data_item_history 时间线（0039 未交付）——分工核查结论见 §5。

---

## 1. 改动文件清单（4 文件，全落 brief 声明范围）

| 文件 | 性质 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/IotDataDictController.java` | 修改（+1 行） | `importErrorReport` 补 `@OperateLog(title="数据字典", businessType=IMPORT)`——唯一的 main 改动，纯接线 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/controller/IotPointController.java` | 修改（+1 行） | `importErrorReport` 补 `@OperateLog(title="采集点位", businessType=IMPORT)`——同上 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/OperateLogCoverageGuardTest.java` | 新增（brief 允许 test/java/**） | 3 用例：①**全局机械差集**——扫描 iot/openapi 全部 @RestController 写端点 ⊆ @OperateLog 标注集（排除 /internal/v1、/openapi/v1，防回归）②两补线端点注解实态逐项锁定（title/businessType）③凭据四类全量审计锁定（write saveParam=false / view saveResponse=false / revoke / list 不挂） |
| `resources/scripts/audit-operatelog-coverage.py` | 新增（brief 允许比对脚本放 resources/scripts/） | 机械差集比对：全端点清单 + 缺标注差集 + 内部/管理面分类 + 零缺失终态裁决 |

**不碰**：OperateLogAspect 核心行为（零行改动）、iot_data_item_history（0039 未交付，只核查登记）、sys_operate_log schema、前端、ingest、calc、ot-core。冒烟脚本 `backend/target/t0065_smoke.py` 为 target 临时产物不入库（不属提交范围）。

---

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` —— 通过

```
$ mvn install -pl ot-core -DskipTests
core exit=0   (BUILD SUCCESS)

$ mvn test -pl ot-admin（全量，官方口径终码）
[INFO] Results:
[INFO] 
[INFO] Tests run: 2373, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:27 min
[INFO] Finished at: 2026-11-23T10:08:17+08:00
```

**基线核对**：2373 = 基线 2370（TASK-0064 报告实测全量）+ 净增 3（本守卫测试 3 用例，零改动既有测试）。全量日志内守卫测试独立报告：

```
Test set: net.otlab.admin.module.iot.config.OperateLogCoverageGuardTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.189 s
```

### 2.2 验收 2：TDD 红转绿 + 覆盖比对零缺失终态 —— 通过

**红相**（未接线时，`-Dtest=OperateLogCoverageGuardTest`，终态两次运行——首轮含 1 处测试自身缺陷 TreeMap<Class>，已修，非产品问题，见 §4）：

```
[INFO] Results:
[ERROR] Failures: 
[ERROR]   OperateLogCoverageGuardTest.allManagementWriteEndpointsAudited:111 
        以下管理面写端点缺 @OperateLog 接线：[POST IotDataDictController#importErrorReport /iot/dict/import/error-report,
        POST IotPointController#importErrorReport /iot/point/import/error-report] ==> expected: <[]> 
[ERROR]   OperateLogCoverageGuardTest.newlyWiredErrorReportEndpointsLocked:118->assertAnnotated:205 
        IotDataDictController#importErrorReport 缺 @OperateLog 接线 ==> expected: not <null>
[ERROR] Tests run: 3, Failures: 2, Errors: 0, Skipped: 0
[INFO] BUILD FAILURE
```

**绿相**（接线后）：

```
[INFO] Results:
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

**为什么现在对了**：两个 importErrorReport 端点各补一行 `@OperateLog(businessType=IMPORT)` 后，全局差集断言（写端点 ⊆ 标注集）与接线实态锁定断言的机械差集结果逐值归零——红相两失败精确对应两处缺失，补线后双向通过；既有切面 `@Pointcut("@annotation(OperateLog)")` 对新增注解位置自动生效，无需改切面。

**机械差集比对零缺失终态**（`python resources/scripts/audit-operatelog-coverage.py` 终态输出节选）：

```
== 缺 @OperateLog 的写端点（机械差集）==
POST  IotGatewayInternalController.java heartbeat  /internal/v1/gateway/heartbeat
POST  IotGatewayInternalController.java offlineEvent /internal/v1/gateway/offline-event
POST  IotPublishInternalController.java ack        /internal/v1/config/ack

写端点总数=61 已标注=58 缺标注=3
  其中内部令牌端点（brief 决策4 排除，各有独立审计链）=3  管理面真实缺口=0

零缺失终态裁决（管理面写端点，内部令牌端点除外）：零缺失 PASS

== 管理面真实缺口 ==

== 凭据 GET 明文查看端点（ADR-0006）==
audited=True flags=saveResponse=false perms=iot:credential:view path=/iot/channel/credential/view/{id}
```

### 2.3 验收 3：dev 冒烟 + sys_operate_log 行实查 —— 通过

启动前 8098 无 LISTENING（发现一个前序 dev 残留进程 PID 61996 已自然退出，port 空出）；`mvn package -DskipTests -pl ot-admin` 终码 jar 133.6MB；`Started OtAdminApplication in 20.592 seconds`（dev 档）。`python backend/target/t0065_smoke.py` 原始输出：

```
OK   | S1 admin login | HTTP 200 token=有
OK   | S2 POST /iot/dict/import/error-report (新补接线) | HTTP 200 ct=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=utf-8 bytes=3713
OK   | S3 POST /iot/point/import/error-report (新补接线) | HTTP 200 ct=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=utf-8 bytes=3715
OK   | S4 GET /iot/channel/credential/view/6 (凭据明文查看) | HTTP 200 code=200
TOTAL=4 OK=4 FAIL=0
SMOKE DONE
```

**sys_operate_log 行实查**（sql-exec.sh 每条语句单独一次调用）：

```
SELECT id,user_name,business_type,request_method,url,status,param,response FROM sys_operate_log
WHERE url LIKE '%error-report%' OR url LIKE '%credential/view%' ORDER BY id DESC LIMIT 6
id    user_name  business_type  request_method  url                                     status  param                                                                                                 response
5615  admin      7              GET             /iot/channel/credential/view/6          1       {}                                                                                                    null
5614  admin      5              POST            /iot/point/import/error-report          1       [{"totalCount":2,"successCount":1,"failCount":1,"persisted":false,"rows":[...BAD_POINT...]}]       null
5613  admin      5              POST            /iot/dict/import/error-report           1       [{"totalCount":2,"successCount":1,"failCount":1,"persisted":false,"rows":[...BAD_CODE...]}]        null
```

- **新补接线端点**：5613/5614 两行实落，`user_name=admin`（操作人）、`param` 完整留存请求体（请求参数）、`status=1`（成功）、`business_type=5`（IMPORT）——运维审计可见「谁、何时、调了哪个写接口、参数是什么、成败」。response 为 null 因返回 xlsx 二进制流（切面 JSON 序列化不适用），与框架既有导出端点同口径。
- **凭据四类全量审计（ADR-0006）实查**：0019 交付面 grep + 本轮单测 + 冒烟三证——写入/轮换（POST /iot/channel/credential，saveParam=false）/查看明文（GET /view/{id}，saveResponse=false）/吊销（DELETE /{id}）四动作均挂 @OperateLog（守卫测试逐项断言锁定）；凭据明文查看 5615 行 `response=null`（saveResponse=false 短路，0052 评审实证口径），param 仅 `{}`（url 承载 id，不落明文）。全库凭据审计行明文 grep 复核：

```
SELECT id,url FROM sys_operate_log WHERE url LIKE '%credential%' AND (param LIKE '%{v1}%' OR response LIKE '%{v1}%' OR response LIKE '%secret%')
id  url
(空) —— 无明文/密文泄漏
```

用毕即停：`taskkill /F /PID 63868` → `netstat :8098` 无 LISTENING。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn package -DskipTests -pl ot-admin`（终码 jar 133.6MB）。启动命令（沿 0052/0064 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

干净单轮（日志 /tmp/t0065_prod.log）：

```
2026-11-23T10:10:45.959+08:00 INFO o.s.boot.tomcat.TomcatWebServer : Tomcat started on port 8098 (http) with context path '/'
2026-11-23T10:10:45.984+08:00 INFO net.otlab.admin.OtAdminApplication  : Started OtAdminApplication in 20.808 seconds (process running for 21.37)
grep -ac " ERROR " /tmp/t0065_prod.log → 0
liveness GET / → 200（/actuator/info 200；/actuator/health=DOWN 为外部依赖健康检查项，非启动错误）
```

- prod 启动 20.8s、**ERROR=0**、liveness 200。接线改动随 jar 生效，无需额外迁移（注解编译入 jar，无 schema/配置变更）。
- 用毕即停：`taskkill /F /PID 64656` → `netstat :8098` 无 LISTENING。

### 2.5 验收 5：改动范围核对 —— 通过（含交错提交说明）

**本任务提交自身（09357d1，父提交 04e6acd）范围核对**：

```
$ git show --name-only --format="" 09357d1
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/IotDataDictController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/controller/IotPointController.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/OperateLogCoverageGuardTest.java
resources/scripts/audit-operatelog-coverage.py
（4 files changed, 365 insertions(+)）
```

逐一核对：2 main = brief 声明的 `module/**` 仅限补 @OperateLog（各 +1 行，diff 实证仅注解）；1 test = `test/java/**` 内新增守卫（既有测试零改动）；1 脚本 = brief 允许的 `resources/scripts/` 比对产物。

**`git diff --name-only 537366e..HEAD` 交错说明**：BASE=537366e → HEAD=09357d1 区间含另一道 agent 的 TASK-0040 提交 `04e6acd`（calc/ 目录 49 文件，与本任务无关——按契约「不碰 calc」未触碰、未纳入本提交）。本任务提交 `09357d1` 的直接父即 04e6acd，其自身 diff 恰为上述 4 文件。TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。

---

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 3 个内部令牌端点补 @OperateLog（心跳/离线事件/配置 ACK） | brief 决策 4 明文排除：`/internal/v1/**` 各有独立审计链（网关内部令牌 + iot_api_call_log + 0051 403 全量），心跳高频落 sys_operate_log 会淹没有效审计面（0017/0025 已登记口径）。机械差集按路径前缀归类，不视为管理面缺口 |
| 改 OperateLogAspect 切面核心（如补前后值列 / 响应序列化扩展） | brief 硬约束「审计切面行为不动（只挂注解接线）」；saveResponse 短路口径见 0052 评审实证——保持框架既有能力，不扩展 schema |
| 建 iot_data_item_history 时间线表 / 实现前后值落库 | 0039 未交付，brief 明文「不建，只做分工核查登记」——见 §5 |
| `/iot/item/validate`、`/iot/item/preview` 管理面端点接线 | 现库无对应管理面端点（设计 §3.1 声明，但实现走内部 `/internal/v1/calc/trial`）；无对象可挂注解——登记遗留 §5 |
| survey/wechat/system 域写端点审计接线 | 不属设计 §3.1/续 接口表与 brief 决策 4 高风险范围（各有独立审计约定），守卫测试扫描范围限定 iot/openapi 包 |

---

## 4. 偏离计划之处

1. **BASE→HEAD 交错提交**：brief 验收 5 写 `git diff --name-only <BASE>..HEAD`，实测区间含并行道 TASK-0040（04e6acd，calc/ 49 文件）——非本任务产物。改以「本提交 09357d1 自身 diff（父 04e6acd）」为范围核对口径（§2.5），并如实登记交错事实；零触碰 calc。
2. **TDD 首轮红相含测试自身缺陷**：守卫首版用 `TreeMap<Class<? extends Annotation>, String>` 触发 `ClassCastException`（Class 不实现 Comparable）；该失败与产品代码无关，改为 `LinkedHashMap` 后消除——最终红相精确收敛到 2 个未接线端点（§2.2 记录终态红相）。**为什么现在对了**：映射容器按插入序遍历即可，不依赖排序，Class 键合法。
3. **守卫扫描范围收窄**：首版按 `net.otlab.admin.module` 全包扫描，把 survey/wechat/system 域 32 个历史未审写端点纳入差集——这些域非设计 §3.1 接口表范围（brief 决策 4）。收窄为 iot/openapi 两包 + /internal、/openapi 路径排除——差集精确收敛到 2 个真实缺口。**为什么现在对了**：范围与设计高风险接口表、机械审计脚本三者口径一致。
4. **冒烟端点 S4 选择凭据查看而非第三个补线写端点**：补接线端点只有 2 个（dict/point error-report），已全部真调（S2/S3）；第 3 项取设计 hard 项「凭据查看明文」实查审计行无明文（brief 决策 3 要求「冒烟双证」）。**为什么现在对了**：S2/S3 覆盖「每条补线端点真调」+ 审计行实查，S4 覆盖凭据明文不落审计列，三类证据齐全。

---

## 5. 遗留问题

1. **时间线分工核查结论（brief 决策 2 / TASK 描述明文要求）**：
   - **边界清晰、不重复双写**：sys_operate_log = 「谁调了哪个接口 + 何时 + 成败 + 请求参数/响应」（接口级操作审计，框架切面统一挂注解）；iot_data_item_history = 「业务值前后变化」（数据项级值时间线，公式/绑定/阈值/汇总每次变更的前后值）。二者维度不同（接口调用 vs 业务值），本任务核查结论：**无需在 sys_operate_log 加前后值列、无需双写**——接线只在 controller 挂 @OperateLog，未触碰任何值时间线写入点。
   - **0039 未交付 = 时间线缺位登记**：iot_data_item_history 的交付任务 TASK-0039 截至本任务仍**未交付**（本仓无该表 DDL / 无写入链 / 无对应 task 闭环）。因此「前后值」业务语义目前**无落点**——操作日志侧已完整留痕（谁/何时/成败/参数），但值变化前后对比在时间线侧缺位。此为**已知状态，登记不阻塞**：0039 交付时按上述边界接入（值变更写时间线，不回写 sys_operate_log），本任务的接线不与其冲突。
2. **设计 validate/preview 管理面端点缺位**：§3.1 p22 `POST /iot/item/validate`、`/iot/item/preview` 现库无管理面端点（实现为内部 `/internal/v1/calc/trial`，管理面 preview 转发端点未建）。后续若建，按本任务口径需挂 @OperateLog（businessType=OTHER，试算不落库）+ 纳入守卫全局口径。
3. **内部端点审计链独立性复核**：3 个排除端点的独立审计链（iot_api_call_log / PermissionDeniedAuditListener 403 全量 / 网关心跳落库字段）本任务只做口径登记（brief 决策 4），未做全量行为回归——不属本任务范围。
4. **xlsx 下载端点 response 列留空**：两个补线端点返回二进制流，切面 JSON 序列化不适用（response=null）；请求参数/操作人/成败/url 均已留存，满足审计主诉求。若后续要求留存错误清单内容，需在切面增加流响应适配——属切面核心改动，本任务契约禁止，登记备查。

---

**一句话结论：通过**——五条验收全绿：全量 mvn 2373（0 失败 0 错误 BUILD SUCCESS，= 基线 2370 + 守卫 3）；TDD 红转绿（红相 2 失败精确指向 2 个未接线端点，补线后归零）+ 机械差集管理面写端点零缺失终态（61 写端点 / 58 标注 / 3 内部端点按决策 4 排除）；dev 冒烟 4/4 + sys_operate_log 行实查（admin + 参数 + 状态留存、凭据查看 response=NULL 无明文）；prod 启动 0 ERROR + liveness 200；本提交 09357d1 的 4 文件全落声明范围（BASE..HEAD 区间含并行道 TASK-0040 calc/ 交错，已登记）。
