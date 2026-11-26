# 评审 brief：TASK-0060 diff 评审（代码评审）

> 实现提交 bf84526（BASE=4dd8144，9 文件 frontend/**）——指标看板（卡片/同比环比/趋势叠加，展示端最后一块页面）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0060-diff.txt`（4dd8144..bf84526，9 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0060-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0060-report.md`（含管理面读口缺口登记）
- TASK 详情 + FEAT-0019/REQ-0029/TC-0019/ADR-0011（看板同源）
- 上游：reports\TASK-0045-report.md（st_metric_value 四元组 tag+4 列 ts/value/yoy/mom+完整度无物理列）+ TASK-0059-review.md（API 缺口先例+安全红线）+ 范式 frontend/src/views/display/**
- 契约：`resources/contract/tdengine-super-tables.md` 冻结 v1.0.0（st_metric_value 4 列）

## 越界反查（协调者已比对，评审者复核）

- 9 文件全落 `frontend/**`；区间仅一提交（无并行交错）；backend/calc/ingest/契约零触碰。

## 评审重点（除 §13 常规质量缺陷外）

1. **安全红线（首要）**：全 diff 扫描确认零 appKey/secret、零 HMAC/openapi 直连、不自造代理；管理面无指标值读口→空态+缺口登记（源码级核实 /iot/metric/** 只有定义读口、值仅在 HMAC /openapi/v1/metrics/query、完整度仅在 calc 内网）
2. **同源纪律**：卡片 value/yoy/mom 形态对齐 st_metric_value 冻结契约（4 列 ts/value/yoy/mom，四元组 tag），不现算、不造假；涨幅方向色集中登记租户可配演进位（ADR-0011）
3. **完整度标记**：80% 阈值与 0045 同口径；完整度无物理列（0045 由内存+/calc/v1/status 承载）——无管理面端点走空态登记，不静默、不造假
4. **趋势图**：ECharts 多周期叠加+去年同期真实行对比（yoy 列或同期 ts 行）+**降采样明示标注**（ADR-0010，同 0059 曲线纪律）；同期为空时不伪造
5. **上看板标记与过滤**：ADR-0011 演进位标记渲染；工厂维度过滤/周期分组；空态统一「指标数据接入中」；路由 /display/metrics 挂 OtLayout 静态 hideInMenu 范式
6. **测试有效性**：601 用例（591 基线+10）三门全绿；display-routes/metric-utils 新测试是否真锁关键约束（路由可达/yoy-mom 方向色/完整度阈值/空态），非纯快照；live 8098 缺席登记合理性
7. **零回归与范围**：0058/0059 既有 display 测试零回归；commit 仅 9 文件 frontend/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0060-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。**特别对第 1 点（安全红线）与第 3 点（完整度空态处置）给出明确判定**。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
