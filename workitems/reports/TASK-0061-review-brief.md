# 评审 brief：TASK-0061 diff 评审（代码评审）

> 实现提交 60c4568（BASE=640c9bd，区间夹入的 0084 修订 3e09775 已按路径隔离）——运维监控页：backend 新建 iot/inspect 巡检域 + frontend monitor/iot-inspect 监控页。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0061-diff.txt`（60c4568 单提交隔离，25 文件，0 ingest 引用）
- 实施 brief + 报告：reports\TASK-0061-brief.md / TASK-0061-report.md（含 D1-D5 逐条落地 + 5 轮 TDD 红转绿 + 验收原始输出）
- TASK 详情 + FEAT-0020/REQ-0031/TC-0020 + ADR-0014（TD-0047 巡检落库 90 天）
- 复用数据源（只读，本任务不改）：iot/calcread（0063 GET /calc/state|log）、iot/statusread（0050）、iot/metric（0045）；前端 display-api.ts 安全红线头注
- 调度范式：iot/publish/config/PublishSchedulingConfig.java（@EnableScheduling）+ notify/PublishAckTimeoutScanner.java（@Scheduled 范式）
- 事件源范式：iot/gateway/event/GatewayOfflineEvent.java + GatewayOfflineEventListener.java（0062 告警会复用本任务边界，评审留意边界清晰）

## 越界反查

- 提交 60c4568 恰 25 文件全落 brief 声明范围（backend iot/inspect 14 主 + 7 测试、frontend monitor/iot-inspect + api/monitor + tests、resources/sql/upgrade/20261124_iot_inspect.sql）；区间内 0084 修订 3e09775（5 文件 ingest/**）已按提交隔离、diff 包 0 ingest 引用；publish/calcread/statusread/metric 既有模块、contract 冻结目录零触碰。

## 评审重点

1. **四类指标口径（D1，REQ-0031 验收 1/6）**：通道健康取既有 channel 状态；点位在线率走**巡检聚合快照**、禁止逐点扫 TDengine，新租户/无点位显示 **N/A 非 100%**；计算成功率基于 0063 calc state/log 派生、无数据不造 100%；指标产出基于 metric list + 完整度（无字段则用 calc state 近似并注明口径）。核实每个指标的数据源真实、空态诚实。
2. **巡检任务（D2）**：Spring @Scheduled 复用 @EnableScheduling（不重复建 config）；**Redis SETNX 重入保护**（拿到锁才跑、finally 比对 value 释放防误删、Redis 不可用 fail-soft 跳过不阻断）；三类清单（离线/CALC_ERR/完整度低于阈值）+ 标签租户库名比对齐全；TENANT_DB_MISMATCH 因 TDengine REST 故障产出「待核验」占位（brief 允许）；巡检只写结果表不改业务数据。
3. **落库 90 天归档（D3，ADR-0014）**：`iot_inspect_result` 字段与索引合理、不套审计四件套（系统级无操作员）；归档按 `now - 90 day` 条件删除幂等、retention 可配；**DDL 只产出未执行**（报告声明 + IotInspectDdlSplitContractTest 逐行切分兼容 sql-exec.sh，无同行多语句）。
4. **只读端点与租户隔离（D4，REQ-0031 验收 5）**：三 GET（dashboard/history/latest）；tenant 从会话强制注入、禁止请求参数指定 tenant；跨租户明细 403 / fail-closed 空集；权限码沿用 iot:instance:query；读不审（0079 范式）；零写口（无 POST/PUT 暴露前端）。
5. **前端监控页（D5）**：四类指标卡 + 四 tab 清单 + 历史回看；空态 a-empty / N/A 非 100%；api 只调管理面端点、**不持 appKey/secret、不直连 openapi HMAC、不自造代理**；菜单种子编号避开已用 168-170、挂 110 系统监控下。
6. **测试有效性与零回归**：backend 2482 用例 0F0E（含 21 新增）、frontend build:test + lint（0 error）+ typecheck（0 TS error）+ pnpm test 606 passed；5 轮 TDD 红转绿留痕真实、临时调试日志已清理；0063/0050/0045 只读调用零回归；既有测试零改动。
7. **范围与登记**：25 文件全落声明范围；0084 并行提交已隔离；报告登记 HEAD 漂移（BASE 640c9bd → 3e09775 非本任务所为）属实。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0061-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。特别对第 1 点（在线率 N/A 非 100%）与第 4 点（跨租户 403/tenant 会话注入）给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
