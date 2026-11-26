# 评审 brief：TASK-0058 批次二 diff 评审（代码评审）

> 实现提交 8660ee0d（BASE=ba0a908，区间干净单提交，14 文件）——工厂状态数聚合读口 + 总览/工厂卡片接真数据。FEAT-0019 硬验收 2 最后缺口。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0058-batch2-diff.txt`（8660ee0d 单提交，14 文件，1463 行）
- 实施 brief + 报告：reports\TASK-0058-batch2-brief.md / TASK-0058-batch2-report.md（含 D1-D4 落地 + TDD 红转绿 + 空态诚实说明）
- TASK 详情：`tasks/TASK-0058.md`（批次一已交付，本批为批次二）
- 关联对象：FEAT-0019/REQ-0029（**验收 2：卡片运行/异常/离线数与状态判定结果一致**）/TC-0019 + ADR-0011（上看板标记）
- 状态判定源（权威，只读复用）：`backend/.../iot/statusread/controller/IotStatusReadController.java` + `StatusCurrentVO.java` + `StatusReadServiceImpl.java`（0050，currentForClient 取 statusCode）
- 指标值源（只读复用）：0087 `/iot/metric/values` + `frontend/.../display/components/metric-card.vue`（复用不改）

## 越界反查

- 提交 14 文件全落声明范围（backend iot/statusread 6 + frontend display 8）；device/metrics/workshop 页面、backend 其他模块、calc/ingest/contract、前端纯函数零触碰；区间 ba0a908..HEAD 干净单提交。

## 评审重点

1. **状态数一致性（D1，REQ-0029 验收 2，头号重点）**：factory-summary 是否从数据权限裁剪后的实例树取工厂子树 nodeType=4 设备，逐个**复用 0050 StatusReadService 的 statusCode 取法**（不复制状态判定逻辑）；RUNNING→running、FAULT→fault、OFFLINE→offline、其余→unknown（UNKNOWN 不冒充正常）；测试是否用 mock 各设备 statusCode 断言分类计数精确、子树遍历正确。
2. **fail-soft 不造假**：单设备状态读失败按 unknown（不炸全厂）、整厂异常返回全 0 + "状态数据接入中"、前端 0 是真 0 显示"0"不是空态；核实无伪造状态数路径。
3. **空态诚实（D2）**：能耗构成/关键数据项/上看板标记无数据源（后端 mark 占位未落列、无能耗端点、非硬验收）→ 保持诚实空态并登记，不伪造；工厂页指标看板接 0087 metric/values 同源展示（复用 metric-card、不现算、null 不冒充 0）。
4. **端点与权限**：GET /api/v1/iot/status/factory-summary；requireDataPerm 403 通用文案不泄露工厂标识、tenant 会话注入、iot:instance:query、读不审（与 status current 一致）。
5. **测试有效性与零回归**：backend 2613 用例 0F0E（2599 基线+14）、frontend 四门 0 错误、649 用例（631+18）全绿；TDD 红转绿真实；既有 display + statusread 零回归。
6. **范围**：14 文件全落声明范围、越界 0、区间干净。

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0058-batch2-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。特别对第 1 点状态数一致性（与 0050 同源、UNKNOWN 不冒充正常）、第 2 点 fail-soft 不造假、第 3 点空态诚实给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
