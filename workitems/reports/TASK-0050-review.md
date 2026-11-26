# 代码评审报告：TASK-0050（状态展示读口——数据服务统一状态查询）

- 评审对象：实现提交 `e49fea8`（BASE=`331f860`，单提交 15 文件 backend/**）
- 评审口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」
- 评审角色：fresh 评审者（仅依据 brief 所指文件与工作区源码，未向协调者追问，未派 subagent）
- 评审日期：2026-11-24

## 结论

**通过。** 发现清单中 **Critical 0 / Important 0 / Minor 2**（均为 deferred，不进修订轮）。按 §13「整体无 Critical/Important 即明确给出通过结论」，本任务 diff 评审通过。

对评审重点 3（TDengine 真机读回未验真的证据充分性）的明确判定：**证据充分，判定通过**——该结论是「按 brief 授权口径足以收尾」的判定，**不是**「真机读回已验真」的宣称；真机读回仍为未验真，列为环境恢复后条件（详见下文 §3）。

---

## 一、越界反查复核（协调者已比对，评审者独立复核）

- `git log --oneline 331f860..e49fea8` → 仅一笔 `e49fea8`，无混入 0042 calc 改动。
- `git diff --name-only 331f860..e49fea8` → 恰 15 文件：statusread 新模块 7 + openapi 4 + 测试 4。
- 越界计数（声明集外文件）= 0；`^(calc|frontend|ingest)/` 命中 = 0。
- 与 brief 改动范围声明一致，越界反查 **PASS**。

---

## 二、评审重点逐条核对（7 条）

### 1. 不现算纪律（ADR-0004）——PASS

- `StatusReadServiceImpl.current(...)` 经 `readRedis()` 用 `@Qualifier("rawStringRedisTemplate")` 直读 `STATE_KEY_PREFIX + tenantCode + ":" + instanceCode` = `iot:st:default:F01.WS01.EQ001`，与 0049 写侧 `iot:st:{tenant}:{instance}`（tenant=tenantCode、instance=instanceCode）形态一致；raw 模板无 key-prefix，规避 BUG-1110 同款读空。
- 仅解析 Redis JSON 的 `statusCode/seq/quality/ts`，**无任何从原始点位重新求值的逻辑**；calc 硬编码的 `statusName/color` 被丢弃后重新翻译——这正是不现算纪律。
- 无键 / 解析异常 / code=UNKNOWN 三处均归一为 `UNKNOWN` + `seq=null` + `quality=3`；管理面 `current()`/历史与对外 `currentForClient()` 共用同一核心 `current(...)`，两面口径一致（dev 冒烟 S5/S6 实证两面同载荷）。
- 源码核对：`D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\statusread\impl\StatusReadServiceImpl.java`（readRedis / 核心 current）。

### 2. 动态翻译——PASS（附 Minor-2）

- 翻译单一来源为 `StatusTranslationMapper`：`iot_status_dict` by code / by seq + `iot_status_rule` by code 兜底，三条 SQL 均 `@InterceptorIgnore(tenantLine)` + 显式 `tenant_id`，每次查询实时读表，无 `@Cacheable`/无 Ehcache/无进程内字典缓存。
- 全仓 grep：statusread/openapi 主码中除 `UNKNOWN` 哨兵编码与 `QUALITY_UNKNOWN=3` 外，**无硬编码状态名/颜色字典**；回退为「以编码自身作名称、颜色 null」，无臆造颜色。0049 F-06（Redis 硬编码 statusName/color）反例已规避，且 `currentReadsRedisAndTranslatesFromDict` 明确断言不采纳 calc 硬编码的 `RUNNING/#00FF00`。
- 随字典改名改色即时跟随有双重实证：`currentTranslationFollowsDictChange` 单测（第二次查表返回新值即跟随）+ dev 冒烟 S2（真实 UPDATE 字典 `运行/#00FF00` → `运转中/#1E90FF` 即刻跟随，对应 TC-0016 步骤 6 / REQ-0007 验收 4）。
- 唯一观察点见发现清单 Minor-2（历史 UNKNOWN 归一未查字典，与 current 路径在「UNKNOWN 被登记进字典」假设下不一致）。

### 3. 历史时间线 SQL + 真机读回证据充分性——PASS（本条为 brief 特别要求明确判定）

**SQL 形态（源码 + 契约测试均核实）**

- `OpenApiTimeseriesClient.queryStatusHistory`：`fullKey = instanceCode + ".STATUS"` → `full_key = 'F01.WS01.EQ001.STATUS'`，**无 `full:` 前缀**，与 0049 修订轮 F-02 锁定的写侧规范形态一致；`tenant_id = <显式>`、`ts >= start AND ts <= end` 下推、`ORDER BY ts ASC`、库名 `iot_{tenantCode}`（经 `TdengineAdmin.databaseName` 唯一入口）、`SELECT ts, value, quality`。
- 编码经 `SAFE_CODE=[A-Za-z0-9_.]+` 白名单校验后拼接，tenant/时间为数值，无新增注入面。
- 假 taosAdapter 契约测试 `StatusHistorySqlContractTest`：内置 JDK `HttpServer`，**用真实 `OpenApiTimeseriesClient`（非 mock）发真实 REST 请求并捕获真实发出的 SELECT**，逐项断言库名 / `full_key` 规范形态 / `!contains("full:")` / tenant / 时间范围 / `ORDER BY ts ASC`。这是对 0049 F-02 教训的直接机械锁死，不是自证式 mock。评审核对源码生成 SQL 与测试断言一致。
- value=seq → 状态翻译路径由 `historyTranslatesSeqViaDict` / `historyUnknownSentinel` 单测覆盖（-1/quality=3 → 对外 seq=null/UNKNOWN，哨兵不暴露）；fail-soft 空集由 dev 冒烟 S7（真调 200/`data=[]`）实证。

**【明确判定】真机读回未验真的证据充分性**

- **判定：证据充分，通过（足以按 brief 口径收尾）。**
- 判定依据：
  1. brief（决策 3）与评审 brief 第 3 点**明文授权**「TDengine REST 故障期以代码路径 + 假适配器 SQL 形态测试闭合，真机读回列为环境恢复后交付条件，不阻塞代码收尾（与 0049 同口径）」。执行者严格照此口径执行，无超范围宣称。
  2. 替代证据强度到位：真实客户端 + 假适配器的 SQL 全形态精确断言（直接对住 F-02），加 value→seq→字典翻译单测、fail-soft 实证。
  3. 执行者**明确标记未验真**、登记 TD-0053、给出可执行补验条件（报告 §3）；与 0049 修订轮 F-01 裁决「应答解析单测 + 精确断言 + 真机补验条件登记 = 闭合到本轮可做到的上限」同口径。
- 判定边界（必须同时说明）：本判定只确认「SQL 形态正确 + 代码路径正确 + 翻译映射正确」，**不等于**「真机读回已验真」；真机 `SELECT` 读回的真实数据一致性仍为**未验真**，须环境恢复后按报告 §3 条件补跑。该缺口是 brief 授权的环境条件，不构成质量缺陷、不阻塞收尾。

### 4. 对外 scope / 错误段——PASS

- `OpenApiQueryServiceImpl.statusCurrent/statusHistory`：`instanceId==null` → `paramError` → `ResultCode.PARAM_ERROR=400`（源码核对 `ResultCode.PARAM_ERROR(400,...)`）；`requireInstanceVisible` → `BusinessException(OpenApiCodes.SCOPE_DENIED=6003)`（源码核对常量值 6003）。
- 复用 0051 既有码，无新增错误码；无权限 6002 / 越 scope 6003 / 参数 400 与 0051 §0 映射表一致（参数类对外仍走 400，与评审 brief 第 4 点「参数错 400」一致；实施 brief 中「6004」为举例措辞，以评审 brief 口径为准）。
- HMAC 鉴权链未改：`OpenApiController` 新端点沿用类级 `@Anonymous` + `requireIdentity`（过滤器属性，缺位 fail-closed 401），过滤器/策略/审计/限流零改动。dev 冒烟 S4（scope 外 HTTP 200/code 6003）、S8（参数 400）、prod 签名探针 200/6003 均实证。

### 5. 数据权限——PASS

- 管理面：`current()/history()` 先 `requireSessionTenant()`（TenantContext 缺位 403），再 `requireVisibleBySession()`（`dataPermService.resolve(userId)` → `VisibleScope`，非 allTenant 且目标不在可见集 → 403 不泄露存在性）；实例编码解析走 `selectByIdsForTenant(tenantId, ...)` 显式租户，跨租户实例取不到编码 → UNKNOWN，无越权读取。控制器单测 `currentForbiddenPropagates` 锁定 403 直达。
- 对外面：复用 0051 机器路径 `resolveClientScope(clientId, tenantId, scopeJson)`（`resolveClientScope` 显式租户重载已核对：tenantId=null/scope 空均 fail-closed `denied()`），scope 外 6003。
- 未发现越权读取他人实例状态路径。

### 6. 测试有效性——PASS（评审未独立复跑全量，依据报告原始输出 + 源码核对一致性）

- 4 新测试类与报告计数一致：`StatusReadServiceImplTest` 9 + `StatusHistorySqlContractTest` 1 + `IotStatusReadControllerTest` 5 + `OpenApiStatusQueryTest` 5 = 20；报告 Results 段 2423/0/0/0 = 基线 2403 + 20，类级 0051（31）/0048（13）数值与既有报告一致。
- live 10/10（P0/P1 + S1–S8）与断言逐条对应：动态翻译改名改色、UNKNOWN 两面一致、scope 6003、参数 400、历史空集、SQL 契约均有对应单测/实证。
- TDD 红转绿留痕真实：编译红（生产类不存在）→ 构造不匹配红（接口/impl 方法 split，测试改持 impl 类型 + 补 Mockito matcher import）→ 绿，修复理由与源码（3 参 impl 构造 + 6 参 Spring 构造）自洽。
- 说明：评审以「报告原始输出 + diff 源码核对」为据，未独立重跑 mvn（§13 未要求复跑，且环境有状态服务）。

### 7. 零回归与范围——PASS

- 见 §一：单提交、15 文件、全落声明范围、calc/frontend/ingest 零触碰。
- 0051/0048 既有测试逐类零回归（报告 Results 段 + 构造向后兼容：8 参构造保留、setter 注入避免改 0051 测试）。
- 工作区当前 `git status` 仅 `?? .agents/`、`?? .tmp-t0044v/`，无本任务提交文件残留；并发 0042 calc 改动未混入 `e49fea8`。

---

## 三、发现清单

| # | 严重级 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| M-1 | Minor | `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\statusread\impl\StatusReadServiceImpl.java`（`history()`，取 `page.rows()` 后未消费 `page.truncated()`）；`OpenApiTimeseriesClient.queryStatusHistory`（`LIMIT FETCH_LIMIT=20001`，`toPointPage` 计算 truncated） | 状态历史两读口返回 `List<VO>`，丢弃 `OpenApiPage.truncated()` 且不设 `X-Truncated`，与 0051 I-4「对外不允许静默丢数据」纪律不一致；超 20000 状态变化点时静默截断。状态变位为低频事件，实际影响极低，建议 deferred（后续让 status 端点承载截断标记或改分页契约）。 |
| M-2 | Minor | `StatusReadServiceImpl.java` history 的 UNKNOWN 分支（`vo.setStatusCode("UNKNOWN"); vo.setStatusName("UNKNOWN");`，未走 `applyTranslation`、color 留 null） | current 路径在 UNKNOWN 已登记字典时会翻译为字典名/色（`currentUnknownTranslatesFromDictIfRegistered` 已断言），历史路径却硬置 `"UNKNOWN"`/null；「UNKNOWN 登记进字典」假设下两面展示不一致。现有字典未登记 UNKNOWN（0048 烟测仅 RUNNING/STOPPED/ALARM/FAULT），影响极低，建议 deferred。 |

两条 Minor 均按 §13 记入 TASK `任务结果` deferred（收尾进 LOG），不进修订轮。

---

## 四、§13 常规质量缺陷核对（错误处理 / 边界 / 安全）

- 错误处理：Redis 读异常、脏 JSON 均 fail-soft UNKNOWN，不炸展示；TDengine 失败 fail-soft 空页；状态读口缺位 `requireStatusReader()` fail-closed 抛 500，不静默造假 UNKNOWN。合理。
- 边界：from≥to / 负数 / null 在管理面控制器与核心 history、对外服务三处校验；instanceCode 空返回空集；UNKNOWN `-1` 哨兵与 `quality=3` 双向归一。合理。
- 安全：full_key 白名单 + 显式 tenant_id + 显式租户实例解析；字典 Mapper 旁路插件配显式租户条件（机器路径无会话）；管理面数据权限裁剪与对外 scope 裁剪均在读之前。未发现新增注入面或越权面。
- 未发现 Critical / Important 级质量缺陷。

---

## 五、评审限制说明

- 本评审未独立执行 `mvn test` / dev/prod 冒烟（评审职责为 diff + 报告一致性核对；实测输出由执行者报告提供，已逐项核对内部自洽与源码一致）。
- 基线 2403 的逐任务累加来源（0054/0058/0081 等前序并行提交）未逐一回溯，超出本次 fresh 评审的输入范围；0051（31）/0048（13）类级零回归数值与既有报告一致。
- TDengine 真机读回状态：**未验真**（环境阻断，TD-0053 登记）——本评审判定其「证据充分、可收尾」，但明确不将其表述为已验真。

---

## 六、一句话结论

**通过**——15 文件单提交全落声明范围、calc/frontend/ingest 零触碰；不现算直读、动态翻译（代码+dev 真改字典双重实证）、历史 SQL 形态（假 taosAdapter 真实捕获并精确锁死无 `full:` 前缀）、对外 6003/400 复用既有码、两面数据权限/UNKNOWN 一致，均核实无误；两条 Minor（M-1 截断标记未透传、M-2 历史 UNKNOWN 未查字典）记 deferred 不阻塞。评审重点 3 明确判定：**TDengine 真机读回未验真的替代证据充分，按 brief 授权口径通过，真机补验仍列为环境恢复后条件。**
