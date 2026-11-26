# 评审 brief：TASK-0087 scoped 复评 round2（修订轮）

> 隔离修订 diff：578753f..2d0f1a9，2 文件（OpenApiQueryServiceImpl + allTenant 契约测试）。首评发现 I-1（Important，allTenant scope 判定缺失），评审 agent 因频率限制中断未写完整报告，发现由协调者独立核实。本轮为唯一复评。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」；scoped 复评只覆盖 I-1 修复 + 回归确认）

## 必读（只给路径，不粘正文）

- **本轮隔离 diff**：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0087-diff-round2.txt`（2 文件，209 行）
- TASK 详情：`tasks\TASK-0087.md`（任务结果段含协调者裁决与 I-1 证据）
- 退回 brief：`reports\TASK-0087-revision-brief.md`（I-1 修复要求）
- 修订报告段：`reports\TASK-0087-report.md`（追加的「修订轮」段）
- 首评 brief + 主体报告：`reports\TASK-0087-brief.md` / `TASK-0087-report.md` 主体段
- 权威证据：`backend/.../openapi/service/impl/OpenApiQueryServiceImpl.java`（:118 正确范式 `!allTenant() && instanceIds().isEmpty()`；:182/:275/:285/:395 为本轮修复点）、`backend/.../iot/dataperm/VisibleScope.java:36-38`（allTenant() 的 instanceIds 恒空集）

## 越界反查

- 修订 2d0f1a9 恰 2 文件（OpenApiQueryServiceImpl + IotDisplayReadAllTenantContractTest）；区间 578753f..HEAD 干净；frontend/calc/ingest/resources/contract/HMAC 鉴权零触碰。

## 复评重点

1. **I-1（Important）真修复**：`:182 itemsRealtime`、`:285 metricsQuery`、`:395 requireInstanceVisible` 三处 scope 判定是否改为 allTenant 感知（allTenant=true 视为可见、跳过 instanceIds 集合检查），与 `:118` 范式一致；`:395` 是否保留 `instanceId == null` 始终拒绝。**附带修复 `:275`（metricsQuery 装载 instanceCode 回显集合）是否必要且正确**——执行者称不修则 allTenant 下 instanceCode 仍空、指标读口不闭合，请独立核实这个判断。
2. **HMAC 零回归**：`resolveScope`/HMAC 身份翻译路径、`:329 points/history` 未改；HMAC 六接口既有测试全绿（2577 全量含既有）。
3. **测试有效性（红转绿）**：`IotDisplayReadAllTenantContractTest` 是否走真实共享核心 + mock `DataPermService.resolve(userId)→VisibleScope.allTenant(false)`，端到端断言管理面 realtime/metric values **非空**；在修复前代码上必红（realtime 空集，`Failures:1` BUILD FAILURE）、修后绿（1/0/0）。非恒真、非 mock 捷径。
4. **回归与范围**：backend 2577 用例 0F0E（2576 基线+1），既有 scope 外/denied/跨租户/HMAC 六接口/守卫测试零回归；范围仅 2 文件、越界 0。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0087-review-round2.md`。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。对 I-1 是否闭合、:275 附带修复是否必要正确、是否有新 C/I 给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
