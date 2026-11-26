# 评审 brief：TASK-0053 diff 评审（代码评审）

> 实现提交 a230f91（BASE=ebd443d，隔离区间剔除 0047 calc 提交 f19edc1；22 文件 backend openapi/**）——推送订阅消费侧（闭合 FEAT-0017）。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-diff.txt`（ebd443d..a230f91 限 backend/，22 文件）
- 实施 brief + 报告：reports\TASK-0053-brief.md / TASK-0053-report.md（含字段名定版登记+EMQX/Kafka 补验条件）
- TASK 详情 + FEAT-0017/REQ-0028/TC-0017/ADR-0010（同窗口重发）
- **权威契约**：`resources/contract/unified-message-model.md` v1.1.0「3.5」推送行——数据项订阅 4 字段（fullKey/ts/value/quality）+ 指标事件 9 扩展字段（metric_code/instance_key/period/window_start/window_end/value/yoy/mom/quality）+ §修订说明第 7 条"字段名待 0053 定版"
- 上游：reports\TASK-0046-report.md（事件生产侧：9 字段、**幂等按全键+窗口非 hash**）+ TASK-0051-report.md（HMAC+审计）+ TASK-0054-report.md + TASK-0029-report.md

## 越界反查

- 22 文件全落 `backend/ot-admin` openapi 模块+测试；区间内 0047 calc 提交已隔离；calc/frontend/ingest/契约零触碰（冻结契约未改实证）。

## 评审重点

1. **字段名定版（本任务契约义务）**：camelCase/毫秒通过 VO javadoc+测试正式定版闭合"待 0053 定版"项——**冻结契约本体未改**（CONV-03 升版走裁决）是否成立；定版口径与 0046 生产侧 javadoc 登记一致
2. **数据项订阅**：4 字段逐字（fullKey/ts/value/quality，camelCase）；订阅时 scope 校验（appKey 授权含目标实例，否则拒绝订阅）；HMAC 链走 0051 既有不改
3. **指标事件消费**：9 扩展字段齐备（含 quality）、MQTT+Kafka 双通道（后者 env 默认关）；**幂等严格按"指标全键+窗口"非 hash**（hash 含值随 0046 修正重算变化——用 hash 幂等会重复消费，这是关键）；同窗重发以最新值为准（live 100→150 实证）；无订阅者不报错
4. **租户隔离**：主题含 tenant + 载荷 tenant 双校验，租户 B 载荷丢弃+WARN（live 实证）——与 0046 生产侧主题隔离形成双保险；scope 三态与 0051 一致
5. **不写业务表**：订阅事件写环形缓冲（触发信号），**以 0045 st_metric_value 物化表为准**不重复写业务表（ADR-0010）——核实无重复落库路径
6. **审计与订阅管理**：订阅管理走 0051 HMAC+@OperateLog（0079/0052 范式）；调用审计 iot_api_call_log 403 类全量不改
7. **环境替代证据**：EMQX 1883 退化→本地真协议夹具 broker 完成 pub/sub、生产侧 env 可切；Kafka 无 broker env 默认关；**充分性判定**（与 0046 同口径：协议层真验证+生产栈真实，恢复后免改码复验）
8. **测试有效性**：2445 测试 0F（基线+22）；4 处 TDD 红转绿（含 live 发现 @Configuration CGLIB 构造修复 proxyBeanMethods=false+public 构造）留痕真实；live 全序列与断言对应；0051/0054 零回归
9. **零回归与范围**：commit 仅 22 文件 backend openapi/**；越界 0

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 3 点（幂等非 hash）与第 7 点（环境替代证据充分性）给出明确判定**。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
