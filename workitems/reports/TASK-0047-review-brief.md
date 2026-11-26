# 评审 brief：TASK-0047 diff 评审（代码评审）

> 实现提交 f19edc1（BASE=ebd443d，20 文件 calc/**）——手动补算接口与计算状态页数据源（解锁 0063 独立 schema/0061 运维监控页）。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-diff.txt`（ebd443d..f19edc1 限 calc/，20 文件）
- 实施 brief + 报告：reports\TASK-0047-brief.md / TASK-0047-report.md（含 5 处 TDD 红转绿留痕+接线义务）
- TASK 详情 + FEAT-0015/REQ-0025/REQ-0023/TC-0015/ADR-0010（管理面入口最小化）
- 上游：reports\TASK-0046-report.md（MetricCascade 拓扑序）+ calc/src/** 现有 metric/graph/event 代码
- 契约：详细设计 §1.9A（iot_calc_state/log 独立 schema）+ §3.4 内部接口 + ADR-0009

## 越界反查

- 20 文件全落 calc/**；区间仅一提交；并行 0053 backend 改动已隔离；backend/frontend/ingest/契约零触碰。

## 评审重点

1. **异步补算正确性**：CalcTaskManager 独立线程池 2 workers+有界队列 64（超限拒绝不拖垮在线计算——限流语义正确）；recalculate taskId 立返（live 64ms/27ms）；task/{id} 进度可查；0040 拓扑序执行（复用而非另写排序）；逐项 MANUAL 留痕缝（trigger_type=MANUAL/is_recalc=1）
2. **停算保护**：连续错误阈值（env 可配）→ calc_status=3；**子树隔离实证**——坏根停算后下游子树隔离、兄弟分支不受影响（live 实证）；第三轮停算跳过；水位单调推进不回退
3. **CalcStateStore 缝**：接口+内存实现——0063 换 JDBC 零改动替换是否真成立（接口边界清晰、无内存特有语义泄漏）；状态字段（last_calc_time/watermark/calc_status/error_count/last_error）与 §1.9A 逐字一致
4. **管理面入口最小化（ADR-0010）**：内部接口无鉴权（calc 内网）；只加 /calc/v1/status 只读 VO 段，**不挂管理面路由**——0061 的权限/审计/页面接线义务 javadoc 是否清晰
5. **测试有效性**：265 测试 0F（244 基线+21）；5 处红转绿（编译期/2 真实断言/Spring 撞映射/端口争用/live 拓扑偏序）留痕真实；live fail=0 全序列与断言对应
6. **零回归与范围**：0041/0042/0045/0046 既有测试零回归；commit 仅 20 文件 calc/**；TDengine 真机补验（TD-0053）列条件不阻断

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0047-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 3 点（CalcStateStore 缝 0063 可替换性）给出明确判定**。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
