# 实施 brief：TASK-0060（指标看板——卡片/同比环比/趋势叠加）

## 批次

- TASK-0060（本批共 1 个，标准任务 3 人天；前端道——前置 0058 批一✅，指标数据源 0045✅ 已就位（st_metric_value 写入方），展示端指标看板）
- BASE：开工 HEAD（git rev-parse --short HEAD 记录进报告）
- 改动范围（越界即违约）：frontend/src/**（展示端指标看板页+api 客户端+组件）、路由配置；不碰 backend/calc/ingest/契约/workitems（报告除外）

## 必读

- TASK 详情 + FEAT-0019/REQ-0029/TC-0019/ADR-0011（看板同源）
- 上游：reports\TASK-0058-report-batch1.md（展示端骨架）+ TASK-0045-report.md（st_metric_value 四元组 tag+4 列 ts/value/yoy/mom+完整度由内存/状态视图承载）+ TASK-0044-report.md（指标查询同源实证：metrics/query 曾空集因无写入方，0045 已补）+ TASK-0059-review.md（管理面 API 缺口先例与安全红线）
- 范式：frontend/src/views/display/**（0058 批一+0059 设备详情页）

## 接口与决策

1. **路由与页面**：/display/metrics（展示端独立视图区，hideInMenu 静态路由范式同 0058/0059）——指标卡片网格（按指标分组/工厂维度过滤）+ 趋势图区
2. **指标卡片**：本期值 + 同比（yoy）+ 环比（mom）涨跌带方向颜色——**数据与指标结果表同数**（0045 st_metric_value：value/yoy/mom 真实列，不现算 ADR-0011）；涨跌颜色租户可配留二期（ADR-0011）——本期固定方向色（涨绿跌红或按产品口径）并 javadoc 登记演进位
3. **完整度标记**：低于阈值（80%，与 0045 阈值同口径）带明显标记——**0045 完整度无物理列**（由 /calc/v1/status.metric 承载）——本批优先从管理面指标查询接口取完整度（若有），无对应端点则走空态并登记缺口（同 0059 API 缺口处置纪律），**不静默、不造假**
4. **趋势图**：多周期叠加（小时/日/月/年）+ 去年同期对比（st_metric_value 同 ts 去年同期行或 yoy 列）；**降采样明示标注**（ADR-0010，同 0059 曲线纪律）
5. **数据源与安全红线**：优先管理面登录态端点（metrics/query 管理面形态需 grep 确认——**禁止前端持有 appKey/secret、禁止直连 openapi/v1 HMAC**）；管理面无指标值读口→空态+缺口登记（0045 写入方就位但读口是否管理面化待核），不自造代理
6. **TDengine 环境**：REST 6041 故障（TD-0053）——真机数据空态 fail-soft，前端空态验证，真机读回列环境恢复后条件
7. **数据纪律**：不造假数据、无数据槽位「指标数据接入中」空态；卡片值与结果表同数（同源断言）

## 验收命令

1. pnpm build:test / lint / typecheck 全 0 error；pnpm test 全绿
2. live 走查（8098 探测，忙则跳过登记）：展示端指标看板页→卡片/趋势/完整度标记/空态→全停
3. 范围核对：git diff --name-only BASE..HEAD 全落 frontend/**

## 报告

报告写到 D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0060-report.md。五项齐全 + 管理面读口缺口登记（若有）+ 一句话结论。

## 契约

- no-subagents；提交只 add frontend/**；不把 TASK 置「已完成」；不改 workitems（报告除外）；同框架同组件；不造假数据。
