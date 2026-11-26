# 评审 brief：TASK-0087 diff 评审（代码评审）

> 实现提交 578753f（BASE=33aef05，区间干净单提交，10 文件）——管理面数据项与指标值四只读端点（补 FEAT-0019 展示端读口缺口）。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0087-diff.txt`（578753f 单提交，10 文件，1239 行）
- 实施 brief + 报告：reports\TASK-0087-brief.md / TASK-0087-report.md（含 D1-D4 落地 + 完整度可达性评估 + 复用策略 + TDD 红转绿）
- TASK 详情：`tasks/TASK-0087.md`（前置 0054，关联 FEAT-0019/REQ-0028/0029）
- 缺口来源：reports\TASK-0059-review.md D-4 + TASK-0060-review.md D-3 + `frontend/src/api/display/display-api.ts` 头注/指标段注释（前端消费契约）
- **被改的共享核心（重点核实不破坏）**：`backend/.../openapi/service/OpenApiQueryService.java` + `impl/OpenApiQueryServiceImpl.java`（提取共享查询核心；HMAC 六接口仍走 requireIdentity/resolveClientScope）
- 复用基线：`.../openapi/support/OpenApiTimeseriesClient.java`（fail-soft 空集/白名单/行数上限/最新一条口径）
- 数据权限范式：`.../iot/dataperm/DataPermService.java`（resolve(userId) 会话版）、`IotStatusReadController.java`（0050 登录态裁剪）
- 完整度源评估：`calc/.../web/MetricObserveController.java:38`（/calc/v1/metric 8095 内网无鉴权观测面，非生产数据源）

## 越界反查

- 提交 10 文件：新建 iot/displayread 包（Service/VO/Controller/Impl）+ 修改 openapi service 接口/实现 2 文件（共享核心提取）+ 3 测试；区间 33aef05..HEAD 干净单提交；frontend/calc/ingest/resources/contract/HMAC 鉴权逻辑（OpenApiAuthFilter/requireIdentity）零触碰。

## 评审重点

1. **共享核心提取不破坏 HMAC（头号重点）**：`OpenApiQueryService`/`Impl` 的改动是否仅做查询能力共享（realtime/history/metrics 核心），HMAC 六接口行为完全不变——跑既有 HMAC 契约测试零回归；新管理面与 HMAC 的**唯一差异是身份→scope 翻译**（HMAC resolveClientScope vs 管理面 resolve(userId)），差异是否被约束在薄适配层、没有混进查询核心。核实 `OpenApiQuerySharedCoreTest` 是否真锁"同参数两路径结果一致"。
2. **登录态数据权限（D2）**：四端点走登录态 @SaCheckPermission("iot:instance:query") + 读不审（无 @OperateLog）；数据权限用 `DataPermService.resolve(SecurityUtils.getUserId())` 会话版（不是 resolveClientScope）；scope 外集合空集/单资源 403、跨租户隔离、403 文案不泄露资源标识；**不引入 appKey/secret、不复用 requireIdentity、不写 /openapi/v1/**。
3. **完整度诚实空集（D1.4）**：执行者评估"不可达 → 诚实空集"是否属实且合理——calc /calc/v1/metric 是 8095 内网无鉴权观测面、完整度仅内存视图无物理列、ot-admin 无 calc 基址/客户端；响应 `available=false` + reason + 空 rows，不伪造 0/1、不静默判完整；不得为了"出数"去连内网观测面或造假。
4. **四端点契约（D1）**：realtime（批量≤200 fullKey→v/q/ts）、item/history（fullKey+interval/agg+时间窗）、metric/values（metricCodes+period+窗→value/yoy/mom/ts 同源不现算）、metric/completeness；输入校验（超 200/空/非法编码不炸）；继承 fail-soft 空集/白名单/行数上限/最新一条口径。
5. **测试有效性与零回归**：backend 2576 用例 0F0E（2559 基线 + 17 新增）；TDD 红转绿真实、非恒真；HMAC 六接口既有测试零回归（重点 1 的证据）；既有 2559 零回归。
6. **范围**：10 文件全落声明范围（iot/displayread + openapi service 共享提取 + 测试）、越界 0、区间干净。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0087-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 1 点（共享提取不破坏 HMAC）与第 3 点（完整度诚实空集）给出明确判定**。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
