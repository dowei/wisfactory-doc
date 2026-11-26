# 实施 brief：TASK-0047（手动补算接口与计算状态页数据源）

## 批次

- TASK-0047（标准任务 3 人天；**计算引擎道 calc/**——前置 0046✅；补算异步任务+calc_state 维护+状态页数据源，解锁 0063 独立 schema 与 0061 运维监控页）
- BASE：开工 HEAD（记录进报告）；只改 calc/**；不碰 backend/frontend/ingest/契约/workitems（报告除外）

## 必读

- TASK 详情 + FEAT-0015/REQ-0025/REQ-0023/TC-0015/ADR-0010（能力一期、管理面入口最小化）
- 上游：reports\TASK-0046-report.md（级联重算 MetricCascade/拓扑序/BACKFILL 留痕/指标事件生产侧）+ TASK-0045-report.md（聚合物化+窗口）+ TASK-0041-report.md（执行器）+ calc/src/** 现有 metric/event 代码
- 契约：详细设计 §1.9A（iot_calc_state/iot_calc_log 独立 schema）+ §3.4 内部接口（recalculate/status/task）
- ADR-0009（稳定序号）

## 接口与决策（逐条结论）

1. **内部三接口（calc 内网，非管理面）**：`POST /calc/v1/recalculate`（body：timeRange+fullKeys[] → 异步 taskId，按依赖图拓扑序重算，复用 0046 MetricCascade）；`GET /calc/v1/status?fullKey=`（水位/calc_status/滞后/error_count）；`GET /calc/v1/task/{taskId}`（进度+逐项结果）；**管理面入口最小化**（ADR-0010）——本任务只交付 calc 内网接口+只读数据源 VO，管理面接线归 0061 运维监控页（后续）
2. **异步任务与限流**：大范围重算异步（taskId 立即返回、后台线程执行）；**补算风暴不拖垮在线计算**（分区隔离：补算任务与在线执行器分线程池/有界队列）；连续错误阈值（env `CALC_ERROR_STOP_THRESHOLD` 默认 5）→ 该节点 calc_status=3 异常停算，**只影响子树**（沿依赖图向下游隔离，不波及兄弟分支）；失败逐项留 MANUAL 记录（trigger_type=MANUAL、is_recalc=1）
3. **calc_state 维护**：水位推进（按窗口完成顺序）、last_calc_time、calc_status（1 正常 2 滞后 3 异常停算）、error_count/last_error；**当前内存态 + 契约测试**（独立 schema 落库归 0063——本任务定义 CalcStateStore 接口+内存实现，0063 换 JDBC 实现零改动替换，照 ingest/calc 假实现范式）；TDengine 仍故障（TD-0053）——真机补验列条件
4. **状态页数据源**：只读 VO（节点/指标全键/水位/状态/最近错误/任务进度）供 0061 消费；不写前端、不挂管理面路由
5. **权限与审计**：内部接口（calc 内网无鉴权）——**管理面入口单独权限+审计归 0061**（本任务 javadoc 登记接线义务）
6. **不碰**：0063 落库（后续）、0061 页面（后续）、backend/frontend、契约冻结

### 测试口径（TDD）

Contract 测试（recalculate 异步 taskId 立返+进度查询+拓扑序重算/MANUAL 留痕/连续失败停算只影响子树/水位推进/CalcStateStore 内存语义）；既有 0041/0042/0045/0046 测试零回归；红转绿留痕。

## 验收命令

1. `cd calc && mvn test` → 0 失败 0 错误（244 基线+新增，Results 段贴回）
2. live 冒烟：calc 起（8095）→ recalculate 异步提交→taskId 立返→进度查询→status 断言→连续失败停算子树隔离→全停
3. 范围核对：git diff --name-only BASE..HEAD 全落 calc/**

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-report.md。五项齐全 + 0061/0063 接线义务登记 + 一句话结论。

## 契约

- no-subagents；只改 calc/**；提交只 add calc/**；不把 TASK 置「已完成」；不改 workitems（报告除外）；契约冻结发现问题报告并停。
