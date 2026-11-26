# 实施 brief：TASK-0058 批次二（总览/工厂卡片接真数据——状态数聚合 + 核心指标）

> 标准任务（backend 聚合读口 + frontend 接真值），续做 TASK-0058（批次一骨架+空态已交付评审通过，本批为批次二）。BASE=ba0a908。一个执行者做完整、统一测一次。这是 FEAT-0019 硬验收 2（卡片运行/异常/离线数与状态判定一致）的唯一缺口。

## 批次

- TASK-0058 批次二（续做，TASK 已是「进行中」，无需重新认领）
- 验收粒度：一个 subagent 跑一次整体验收（backend mvn test + frontend 四门）
- BASE：`ba0a908`
- 改动范围（文件/目录级，越界反查用）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusread/**`（新增工厂状态数聚合读口 + VO/DTO/service）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusread/**`
  - `frontend/src/views/display/overview/index.vue`、`factory/index.vue`、`components/display-factory-card.vue`、`components/display-slot-card.vue`（可复用现有槽位组件，按需）
  - `frontend/src/api/display/display-api.ts`（加 1-2 封装，只增不改既有签名）
  - `frontend/tests/unit/display/**`
  - **不得触碰**：frontend 其他页面（device/metrics 已交付，零触碰）、backend 其他模块（calcread/displayread/openapi/dataitem/inspect/alarm 等）、calc/ingest/resources/contract（冻结）、component 纯函数（metric-utils 等）

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks/TASK-0058.md`（任务结果段批次一 + 批次二待办）
- 关联对象：`features/FEAT-0019.md`、`requirements/REQ-0029.md`（**验收 2：卡片运行/异常/离线数与状态判定结果一致**）、`testcases/TC-0019.md`、`decisions/ADR-0011.md`（上看板标记）
- 批次一报告/评审：`reports/TASK-0058-report-batch1.md` / `TASK-0058-review-batch1.md`（骨架+空态已交付；3 Minor deferred，其中 M-2 入口形态裁决"归批次二或独立入口，不返工"）
- 状态判定数据源（权威）：`backend/.../iot/statusread/controller/IotStatusReadController.java`（GET /api/v1/iot/status/current?instanceId → StatusCurrentVO{statusCode,statusName,color,seq,quality}）、`StatusCurrentVO.java`、`StatusReadServiceImpl.java`（读 iot:st Redis）
- 状态编码值域：`StatusCurrentVO:30` RUNNING/FAULT/UNKNOWN… + 状态字典（IotStatusDict，seq 排序）
- 实例树：`GET /iot/instance/tree`（0012，数据权限裁剪）+ IotInstanceTreeVO{id,parentId,nodeType,children,mark}；nodeType 1工厂 2车间 3产线 4设备 5组件
- 指标数据源（复用不重造）：`GET /iot/metric/list`（0045）+ `GET /iot/metric/values`（0087 管理面读口，本批可复用于工厂核心指标）
- 前端空态槽位：`frontend/src/views/display/components/display-slot-card.vue`（批次一的"运行/异常/离线/关键数据项/核心指标"占位）

## 接口与决策（逐条结论，无选项）

### D1 · backend 工厂状态数聚合读口（硬验收 2）
- 新增 `GET /api/v1/iot/status/factory-summary?factoryInstanceId=` → 返回该工厂（含子树）设备的 statusCode 分类计数 `{running:int, fault:int, offline:int, unknown:int, total:int}`。
- 实现口径：从实例树取 factoryInstanceId 子树下 nodeType=4（设备）的实例 → 对每个设备读现有 status current（StatusReadService 的同租户口径，复用其取 statusCode 的方法，不复制状态判定逻辑） → 按 statusCode 聚合计数。statusCode→分类映射：RUNNING→running；FAULT/故障类→fault；OFFLINE/离线类→offline；其余（UNKNOWN/停机等）→unknown（**不把 UNKNOWN 算进任何"正常"类**）。
- **fail-soft**：单个设备状态读失败按 unknown 计（不炸全厂），整厂聚合异常返回全 0 + 明确"状态数据接入中"，**不伪造状态数**。
- 数据权限：factoryInstanceId 走 requireDataPerm（无权 403，不泄露工厂标识），tenant 会话注入；权限码沿用 iot:instance:query；**读不审**（0079 范式，与 status current 一致）。
- 状态数必须与"设备状态判定结果一致"（REQ-0029 验收 2）——测试用 mock 各设备 statusCode 断言分类计数精确。

### D2 · frontend 总览/工厂卡片接真数据
- 总览页（overview/index.vue + display-factory-card.vue）：每个工厂卡片加载时调 factory-summary 填"运行/异常/离线"三槽位真值；原 display-slot-card 占位换为数值显示（保留 0=真 0 时显示"0"，不是"接入中"；无数据/fail-soft 时才"接入中"）。
- 工厂页（factory/index.vue）：工厂级指标看板复用 0087 metric/values（按工厂维度），车间列表保留节点真数据（0012 树），**能耗构成（按车间占比）保持"接入中"空态**——无专用端点/无数据源，诚实空态不造假（REQ-0029 业务能力有描述但非硬验收，报告登记）。
- "关键数据项"槽位：无上看板标记数据源（后端 mark 占位未落列，ADR-0011/0044 登记）——**保持空态**，报告登记；不伪造"哪些是关键"。
- "上看板标记"（node.mark）仍可选字段、后端未交付前恒缺省——卡片放大/置顶逻辑保留但不造标记。

### D3 · 车间页设备列表状态色块
- 若车间/产线页有设备列表状态色块空位，复用 status current（按设备）填色块（与 0059 设备详情页状态区同源口径）；无数据走 UNKNOWN 灰显，不造假。若批次一车间页已用实例树 status 字段展示，则保持现状、本批不改——评审按源码判定。

### D4 · 测试口径（TDD 红转绿）
- backend：factory-summary 分类计数精确（RUNNING/FAULT/OFFLINE/UNKNOWN 各设备 mock → 计数）、子树遍历正确、单设备失败按 unknown、整厂 fail-soft 全 0、无权 403、tenant 隔离。
- frontend：卡片三槽位真值渲染（0 显示"0"不是空态）、fail-soft 空态、能耗/关键数据项保持空态诚实、安全红线（零 appKey/HMAC）。
- 既有 display 单测（0058 批次一 583 + 0059/0060/0089 的 631 基线）零回归。

## 验收命令（必须原样贴回输出）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（含 factory-summary 聚合契约测试 + 既有 2599 基线零回归），贴 Results
2. `cd frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test` → 0 错误 0 失败（含卡片真值/空态测试 + 既有 631 零回归），贴原始输出
3. 范围核对：`git diff --name-only ba0a908..HEAD` 全落改动范围声明；device/metrics 页面/其他模块/冻结契约零触碰

## 报告

报告写到：`reports\TASK-0058-batch2-report.md`（批次一报告已单独存在，本批独立报告）。五项齐全 + 一句话结论 + TDD 红转绿 + 状态数一致性证据 + 能耗/关键数据项空态诚实说明 + 上看板标记未落地登记。

## 契约

- no-subagents；只改本批改动范围；TASK 已是进行中，不置已完成；不改 workitems 索引。
- 状态数必须与设备状态判定一致（REQ-0029 验收 2）、与 0050 同源不另判；不造假数据（无源空态、0 是真 0）；能耗构成/关键数据项/上看板标记无数据源保持诚实空态并登记。
- 数据权限 requireDataPerm 403、tenant 会话注入、读不审；前端安全红线不碰 appKey/HMAC。
- 测试失败先定位根因，禁止为匹配输出改测试/改期望。
