# 实施 brief：TASK-0053（推送订阅与指标产出事件——数据服务侧订阅消费）

## 批次

- TASK-0053（标准任务 3 人天；**backend 数据服务道**——前置 0045✅/0042✅/0029✅；**事件生产侧已由 0046 交付**（MQTT iot/{tenant}/metric/out + Kafka，9 扩展字段含 quality），本任务只做订阅消费侧）
- BASE：开工 HEAD（记录进报告）；只改 backend/ot-admin 的 openapi（数据服务）模块+测试；不碰 calc/frontend/ingest/契约/workitems（报告除外）

## 必读

- TASK 详情 + FEAT-0017/REQ-0028/TC-0017/ADR-0010（修正后同窗口重发）
- **权威契约**：`resources/contract/unified-message-model.md` v1.1.0「3.5」章推送行——数据项订阅主题 `iot/open/{tenant_code}/{instance_code}/item`（消息体统一消息模型 4 字段 fullKey/ts/value/quality）+ 指标产出事件 Kafka `iot.metric.result.{tenant}` / MQTT `iot/{tenant}/metric/out`（9 扩展字段：metric_code/instance_key/period/window_start/window_end/value/yoy/mom/quality）；§修订说明第 7 条字段名风格"待 TASK-0053 定版"——**本任务正式定版 camelCase/毫秒**（与 0046 生产侧 javadoc 登记一致）
- 上游：reports\TASK-0046-report.md（**事件生产侧交付**：载荷 9 字段+幂等口径"按指标全键+窗口，**非 hash**"）+ TASK-0051-report.md（对外鉴权+调用审计链）+ TASK-0054-report.md（dict/sync 增量口径）+ TASK-0029-report.md（新值事件发射侧 iot:newvalue:{tenant}）
- 平台：EMQX 192.168.31.145:1883（当前退化 MQTT 握手无 CONNACK——用本地真协议夹具 broker 验证，生产侧 addr env 可切，恢复后免改码复验）；Kafka 无 broker（env 开关默认关）

## 接口与决策（逐条结论）

1. **数据项变更订阅**：订阅 `iot/open/{tenant_code}/{instance_code}/item`（Paho MQTT 客户端，env broker 可切），事件源=0029 新值事件（经数据服务转发或桥接——按 0046/0029 已落地的事件链对接，不自造新事件）；**订阅时校验 scope**（appKey 授权范围含目标实例，否则拒绝订阅/断开）；消息体统一消息模型 4 字段逐字（fullKey/ts/value/quality）——不改契约
2. **指标产出事件消费**：订阅 MQTT `iot/{tenant}/metric/out` + Kafka `iot.metric.result.{tenant}`（后者 env 默认关）——**幂等口径=指标全键+窗口（非 hash**，hash 含值随 0046 修正重算变化）；载荷定版 camelCase/毫秒（§修订说明第 7 条的"待定版"由本任务闭合）；中台以订阅代替高频轮询
3. **事件落地/转发**：订阅到的事件写入数据服务可读形态（内存环形缓冲+指标结果侧读口对接 0045 st_metric_value 物化——**以物化表为准，事件仅作触发信号**，不重复写业务表 ADR-0010）；"无订阅者不报错"（生产侧已保证，消费侧重连/退订幂等）
4. **租户隔离**：主题含 tenant + 载荷含 tenant 双重校验——**租户 B 收不到租户 A 事件**（生产侧 0046 已保证主题隔离，消费侧再验租户字段一致，不一致丢弃+WARN）；scope 裁剪与 0051 对外 scope 三态一致
5. **对外载荷与审计**：订阅管理接口（开通/关闭订阅）走 0051 appKey+HMAC 链+scope 校验；订阅动作走 @OperateLog 审计（0079/0052 范式）；调用审计 iot_api_call_log 403 类全量（0051 既有，不改）
6. **不碰**：calc 生产侧（0046 已交付）、0047 补算、0060 看板、前端、契约文件（字段名定版通过 javadoc+VO 实现，不改冻结文件——CONV-03 升版走裁决）

### 测试口径（TDD）

Contract 测试（数据项订阅 4 字段逐字+scope 校验/指标事件 9 字段 camelCase 定版+全键窗口幂等/修正重发同窗口幂等不重复/租户 B 隔离/无订阅者不报错/Kafka env 默认关）；既有 0051/0054 测试零回归；红转绿留痕。

## 验收命令

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 0 失败 0 错误（Results 段贴回）
2. live 冒烟：本地夹具 broker 起→数据服务订阅→真发数据项事件+指标事件（含修正重发）→断言 4 字段/9 字段+幂等+租户隔离→全停
3. prod 档真实启动回归 → 0 ERROR
4. 范围核对：git diff --name-only BASE..HEAD 全落 backend openapi 模块+测试

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-report.md。五项齐全 + 字段名定版登记（0053 闭合契约待定版项）+ EMQX/Kafka 环境补验条件 + 一句话结论。

## 契约

- no-subagents；只改 backend openapi 模块；提交只 add backend 对应文件；不把 TASK 置「已完成」；不改 workitems（报告除外）；契约冻结发现问题报告并停。
