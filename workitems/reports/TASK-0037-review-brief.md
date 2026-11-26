# 评审 brief：TASK-0037 diff 评审（代码评审）

> 实现提交 2ad022e（BASE=2d0f1a9；区间夹入并行 0089 的 630327c frontend 提交，已用单提交隔离，17 文件）——validate/preview 试算端点 + calc trial 转发客户端。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0037-diff.txt`（2ad022e 单提交隔离，17 文件，1483 行）
- 实施 brief + 报告：reports\TASK-0037-brief.md / TASK-0037-report.md（含 D1-D4 落地 + TDD 红转绿 + 不挂 @OperateLog 裁决理由 + §5.4/§7 偏离登记）
- TASK 详情：`tasks/TASK-0037.md`（前置 0036 + 0086）
- 关联对象：FEAT-0012/REQ-0016/TC-0012 + ADR-0005 + ADR-0013 增补 5
- **calc trial 契约（0086 交付，权威，只读调用）**：`calc/.../trial/TrialRequest.java`/`TrialResponse.java`/`web/TrialController.java`
- 影响分析（0036 已交付，复用不改）：`IotDataItemController.java:142-146` GET /iot/item/{itemId}/impact
- 审计守卫（本任务有连带改动，重点）：`backend/.../iot/config/OperateLogCoverageGuardTest.java`（声明范围外，加了两条精确路径豁免）+ `reports/TASK-0065-report.md:214`（曾建议 validate/preview 挂 @OperateLog，本 brief 裁决为读语义不挂）

## 越界反查（本任务有 1 个声明范围外连带，重点核实）

- 提交 17 文件：16 全落声明范围（backend iot/dataitem 主代码 + DTO + 客户端 + 测试 + application.yml）；**1 个声明范围外**：`backend/.../iot/config/OperateLogCoverageGuardTest.java`（M，加两条精确路径豁免 /iot/item/validate、/iot/item/preview）。执行者称是为满足"试算不挂 @OperateLog"硬约束又保持守卫绿的最小改动。**必须独立核实**：豁免是否精确到只有这两条 POST 路径、未放宽其他端点覆盖、守卫仍机械有效（新增其他 POST 无审计仍会被抓）。区间夹入的 0089 frontend 已隔离。

## 评审重点

1. **同口径（核心）**：preview 是否直接转发 calc `/internal/v1/calc/trial` 并原样回传 value/quality/missing/reason/engineVersion（backend 不另算）；请求/响应 DTO 与 calc TrialRequest/Response 逐字对齐（camelCase/毫秒）；测试是否用 mock calc HTTP 服务断言请求体逐字、响应原样回传（非恒真）。
2. **缺数不冒充 0 + fail-soft**：preview 缺数（trial 返回 missing）→ 明确缺数提示、value 不为 0；calc 不可达/超时/非2xx/解析失败 → fail-soft 返回"试算服务不可用"明确错误，**不伪造试算结果、不静默返回 0/空**；variables 取不到时不填 0 让 trial 报缺数。
3. **validate 校验链**：函数白名单+相对引用 {SELF.X} 展开+引用存在性+循环依赖，错误码 4001 段；管理面粗校验 + 可转发 trial 权威校验。
4. **端点与权限**：POST /iot/item/validate + /iot/item/preview；@SaCheckPermission("iot:item:query") + requireDataPerm 无权 403（不泄露实例标识）；**不挂 @OperateLog**（试算读语义，报告说明与 TASK-0065 建议的差异理由）；影响分析 0036 impact 端点零回归。
5. **审计守卫连带（越界反查重点）**：见上，精确路径豁免最小性 + 守卫仍有效。
6. **测试有效性与零回归**：backend 2599 用例 0F0E（2577 基线 + 22 新增）；TDD 红转绿真实；既有 2577 零回归；范围（含 1 连带）判定。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0037-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 1 点同口径、第 2 点缺数/fail-soft、越界反查的守卫豁免最小性给出明确判定**。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
