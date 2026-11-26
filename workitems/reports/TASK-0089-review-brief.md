# 评审 brief：TASK-0089 diff 评审（代码评审）

> 实现提交 9459e34（BASE=2d0f1a9，单提交隔离，6 文件纯 frontend）——展示端接入管理面值读口（FEAT-0019 收尾）。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0089-diff.txt`（9459e34 单提交，6 文件，744 行）
- 实施 brief + 报告：reports\TASK-0089-brief.md / TASK-0089-report.md（含 D1-D4 落地 + TDD 红转绿 16 红→20 绿 + 空态/安全红线证据）
- TASK 详情：`tasks/TASK-0089.md`（前置 0087，关联 FEAT-0019/REQ-0029/0030）
- 关联对象：FEAT-0019/REQ-0029/REQ-0030/TC-0019 + 0059/0060 原评审缺口登记
- 0087 端点契约（权威）：`backend/.../displayread/controller/IotDisplayReadController.java`（四端点）+ `MetricValueVO`/`MetricCompletenessVO`（响应字段）
- 前置约定（brief D 纪律）：组件与纯函数零改动（metric-card/trend/metric-utils/device-utils 等），display-api.ts 只增不改既有签名

## 越界反查

- 提交 9459e34 恰 6 文件（display-api.ts + device-realtime-zone.vue + metrics/index.vue + 3 新测试），全落声明范围；组件/纯函数/backend/calc/ingest/contract 零触碰（报告称逐一 git diff --quiet 核实）；并行 0037 的 backend 暂存变更未进本提交（提交路径受限）。

## 评审重点

1. **安全红线（头号重点）**：四封装只走管理面登录态 getRequest；全 diff 零 appKey/secret、零 HMAC/createHmac 直连、零 /openapi/v1/** 直连、零自造代理（含注释/文案措辞不混淆）；display-api.ts 既有封装签名未改（diff 无删除既有行）。
2. **空态诚实（REQ 不造假）**：realtime 的 v=null/无该项/读口失败 → "数据接入中"空态不造假；指标值 null 伴随值不冒充 0、同源不现算；**完整度 available=false 时 completenessMap 保持空 + readGap 告警（"接入中"），不静默判完整、不显示伪造百分比/角标**。
3. **端点契约对齐 0087**：四封装 URL/参数/响应字段逐字对齐（realtime Map<fullKey,{v,q,ts}>、metric values value/yoy/mom/ts、completeness available/reason/rows），asArray/空对象容错。
4. **组件/纯函数零改动**：metric-card.vue/metric-trend.vue/metric-utils.ts/trendDisplay/completenessDisplay/buildMetricTrend/device-utils.ts 等零改动；只换 load 数据来源与 readGap 判定——git diff 确认。
5. **测试有效性与零回归**：pnpm build:test/lint/typecheck/test 四门 0 错误；630 用例（610 基线+20 新增）零回归；新增测试真锁 URL/参数/空态/安全扫描非快照；偶发 menu-cache-frame 超时（无关既存、隔离通过）按环境抖动登记合理、未改测试掩盖。
6. **范围**：6 文件全落声明范围、越界 0。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0089-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。特别对第 1 点安全红线、第 2 点完整度 available=false 空态给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
