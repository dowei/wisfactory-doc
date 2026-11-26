# 实施 brief：TASK-0037（validate/preview 单点试算与影响分析）

> 标准任务（backend 管理面试算端点 + calc trial 转发客户端）。BASE=2d0f1a9。一个执行者做完整、统一测一次。前置 0036✅、0086✅（calc trial 端点已交付，解开原先死锁）。与 TASK-0089（纯 frontend）文件集/测试环境不交，可并行。

## 批次

- TASK-0037（本批 1 个）
- 验收粒度：一个 subagent 跑一次整体验收（backend mvn test）
- BASE：`2d0f1a9`
- 改动范围（文件/目录级，越界反查用）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/**`（validate/preview 端点 + 试算服务 + calc trial 客户端 + DTO）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/**`
  - `backend/ot-admin/src/main/resources/application.yml`（可选：calc 基址配置项，若加需说明 prod 档）
  - **不得触碰**：frontend/**、calc/**（trial 端点已由 0086 交付，只读调用不改）、ingest/**、resources/contract/**（冻结）、既有 iot 业务模块（inspect/alarm/displayread/calcread 等）、影响分析现有 impact 端点行为

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0037.md`（前置已改为 0036 + 0086）
- 关联对象：`features/FEAT-0012.md`、`requirements/REQ-0016.md`、`testcases/TC-0012.md`、`decisions/ADR-0005.md`、`ADR-0013 增补 5`
- **calc trial 端点契约（0086 交付，权威）**：`calc/src/main/java/net/otlab/calc/trial/TrialRequest.java` / `TrialResponse.java` / `calc/web/TrialController.java`
  - `POST http://{calcBase}/internal/v1/calc/trial`
  - Request：`{formula, variables:Map<key, number|{value,quality,ts}>, atTime?, window?:{start,end,stepMs}}`
  - Response：`{value:number|null, quality:int, missing:[key], reason, engineVersion, curve?:[{ts,value,quality,missing,reason}]}`
- 影响分析（0036 已交付，复用不改）：`IotDataItemController.java:142-146` `GET /iot/item/{itemId}/impact`（引用图反查；指标半边空集占位是 0043 范围，本任务不改）
- 设计端点缺位登记：`reports/TASK-0065-report.md:21/195/214`（设计 §3.1 的 POST /iot/item/validate、/iot/item/preview 管理面端点现库缺位）
- 公式校验可复用：`backend/.../iot/dataitem/service/support/FormulaFunctionWhitelist.java` + `RelativeRefExpander.java`（0036 已有语法/函数白名单）；calc 侧 `FormulaExpressionValidator` 只存在于 calc 进程，管理面先做粗校验再转发
- 配置范式：`application.yml` 既有 ot.* 段；HTTP 客户端可用 Spring `RestClient`/`HttpClient`（项目 Java 21）

## 接口与决策（逐条结论，无选项）

### D1 · 管理面端点
- `POST /iot/item/validate`（请求 {instanceId, itemId?, formula}）→ 不落库；语法校验（函数白名单 + 相对引用展开 {SELF.X} + 引用存在性 + 循环依赖检测），返回 `{valid:boolean, errors:[...]}`；错误码 4001 段（与保存拒存同口径）。管理面粗校验后**可转发 calc trial 做权威校验**（trial 的语法/白名单拒绝 400 映射为 invalid）。
- `POST /iot/item/preview`（请求 {instanceId, itemId?, formula, atTime?, variables?}）→ 不落库；用实时值/指定时刻值代入求值（DELTA/INTEGRAL 取一窗历史），**转发 calc trial 同口径**，缺数明示不用 0。
- 权限 `@SaCheckPermission("iot:item:query")`（与 impact/effective-params 同口径 4037）；**不挂 @OperateLog**——试算不落库无业务副作用，按读处理（与 0061/0062 读不审一致；TASK-0065 §5 曾建议 "若建则挂 @OperateLog"，本任务裁决为读语义，报告说明差异理由，避免试算刷审计）。
- 影响分析：0036 的 `GET /iot/item/{itemId}/impact` 已交付，本任务**不重做**，仅在报告确认可用（影响清单含上级项；指标半边空集占位属 0043 范围，不扩张）。

### D2 · calc trial 转发客户端（新增，可配 + fail-soft）
- 新增 backend→calc HTTP 客户端：`ot.calc.base-url` 配置（默认 `http://127.0.0.1:8095`，prod 档需 env 注入实际地址；与 TDengine `OT_TDENGINE_BASE_URL` 同范式）。
- 调 `POST {base}/internal/v1/calc/trial`，请求/响应 DTO 与 calc TrialRequest/Response **逐字对齐**（ camelCase、毫秒）；超时/连接失败/非 2xx/响应不可解析 → fail-soft 返回明确"试算服务不可用"错误（**不伪造试算结果、不静默返回 0/空**），报告登记。
- 同口径：管理面 preview 结果直接来自 trial 响应（value/quality/missing/reason/engineVersion 原样回传），**不在 backend 另算**（ADR 定版：试算与真算同引擎）。

### D3 · 缺数与变量取数（管理面职责）
- preview 的 variables 由管理面提供：优先用请求体显式 variables；未提供时按 fullKey 从数据项定义/状态读口（0050）取当前值——取不到的标识符**不填入** variables（让 trial 报缺数 missing），**绝不填 0 蒙混**（REQ-0016 验收 4）。DELTA/INTEGRAL 的历史窗由调用方传 window/history（0086 trial 支持 window，历史取数超出本任务范围则诚实缺数）。
- 影响分析的数据权限：validate/preview 按 instanceId 走 requireDataPerm（与 0036 写路径同口径，无权 403 不泄露）。

### D4 · 测试口径（TDD 红转绿）
- validate：合法公式 valid=true；函数白名单外/循环依赖/畸形 token → valid=false + 4001 错误；相对引用 {SELF.X} 展开正确。
- preview：转发 trial 成功 → value/quality/missing/engineVersion 原样回传（mock calc HTTP 服务断言请求体逐字）；缺数（trial 返回 missing）→ 前端可见缺数提示不冒充 0；calc 不可达 → fail-soft 明确错误不假结果。
- 影响分析：确认 0036 impact 端点零回归（上级项清单）。
- 既有 2577 基线零回归。

## 验收命令（必须原样贴回输出）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（含新增 validate/preview/转发客户端契约测试 + 既有 2577 基线零回归），贴 Results 段
2. 范围核对：`git diff --name-only 2d0f1a9..HEAD` 全落改动范围声明；frontend/calc/ingest/contract/既有 iot 业务模块零触碰

## 报告

报告写到：`reports\TASK-0037-report.md`。五项齐全 + 一句话结论 + TDD 红转绿 + 同口径证据（preview 结果直接来自 trial）+ 缺数不冒充 0 + fail-soft 证据 + 不挂 @OperateLog 的裁决理由。

## 契约

- no-subagents；只改本批改动范围；不置 TASK 已完成；不改 workitems 索引。
- 试算不落库、同引擎同口径（转发 calc trial 不另算）；缺数明示不用 0；calc 不可达 fail-soft 不造假；数据权限 requireDataPerm 403；不挂 @OperateLog（读语义，报告说明）。
- 不硬编码凭据/文档库路径；测试失败先定位根因，禁止为匹配输出改测试/改期望。
