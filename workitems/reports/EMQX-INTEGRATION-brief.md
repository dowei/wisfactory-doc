# 实施 brief：EMQX 真联调验证（TD-0056 / TD-0057 清账——纯验证不改业务代码）

## 批次

- 真联调批次（**非工作项实现**——验证已交付的 MQTT 链路在真 broker 上可用；TD-0056 backend 发送 + TD-0057 ingest 消费 + calc 订阅缝观察）
- BASE：开工时主仓 HEAD（`git rev-parse --short HEAD` 记录进验证报告）
- 改动范围（越界即违约）：
  - **零业务代码改动**——只允许环境配置（application-dev.yml 的 broker URL/开关、env 注入）与验证脚本（临时文件放 /tmp 或 reports/）
  - 不碰 backend/ingest/calc/frontend 的源码逻辑；不改 workitems（报告除外）

## 环境事实（协调者 2026-11-23 实测）

- **EMQX**（192.168.31.145）：1883/18083/8083/8084/8883 全开；Dashboard `emqx is running`；**匿名 allow**（认证列表空）+ **授权 file 规则放行 iot/#**——裸 MQTT CONNECT CONNACK `20 02 00 00` accepted、SUBACK QoS1 成功、pub/sub 往返收到业务主题消息（实证）
- MySQL/Redis 已迁宿主 phpStudy（8.0.12@3307 / Redis 3.0.504@6379，实测正常）
- TDengine 192.168.31.145:6041（root/taosdata）fd 根治可用（CREATE DATABASE 20-25s 慢特性）
- frontend 8099 / backend 8098 / ingest 8090 / calc 8095——启动前探测、用毕即停
- **TD-0058 注意**：Spring Boot 4 = Jackson 3（tools.jackson）——如需构造 stub 响应用 Jackson 3 API

## 验证项（逐项实测，原始输出留证）

### A. 连通性与 broker 基本行为（已由协调者验证，你复审一次即可）
- 裸 MQTT CONNECT/SUB/PUB 往返（可复用 reports 里的 python 裸协议脚本思路，或 paho 客户端若可用）

### B. backend → EMQX 真发（TD-0056）
1. 配置 backend dev 的 `ot.publish.notify.enabled=true` + broker-url `tcp://192.168.31.145:1883`（env 优先，不落源码）
2. backend dev 起（8098）→ **外部 MQTT 客户端订阅** `iot/config.changed` 与 `iot/{tenantCode}/{gatewayCode}/config`
3. 真调 `POST /iot/publish/publish`（scope_type=2 计算域 + scope_type=1 网关域各一次）→ 订阅端断言：
   - 共享主题 `iot/config.changed` 载荷五键逐字（tenant/scopeType/scopeId/version/hash），hash 与版本行 snapshot_hash 一致
   - 网关专属主题 `iot/{tenantCode}/{gatewayCode}/config` 收到对应网关通知（QoS1）
4. fail-soft 旁证：broker 不可达时发布仍 200（既有测试已覆盖，本轮真机只做正向）

### C. ingest ← EMQX 真收（TD-0057）
1. ingest 配真 broker（env `INGEST_MQTT_BROKER=tcp://192.168.31.145:1883`），ingestd 起（8090）
2. 真 MQTT pub 一条 §3.2 上行 payload 到 `iot/group_a/GW_TEST/data`（合法 batchId/points，复用 ingest 契约测试 fixture）
3. 断言全链：TDengine 真库写入（SELECT st_point_num 对应子表值）+ Redis 点位缓存键（iot:pt:group_a:*，用 Python RESP 或 phpStudy redis-cli）+ 新值事件（Redis Pub/Sub 订阅 iot:newvalue:group_a 收到）
4. 未知编码批 → 死信不落库（日志断言）
5. **注意**：Go 侧在途任务已完成（0029 收官），真写路径可用；TDengine 库名按 DBName() 派生（不要手拼）

### D. calc 订阅缝观察（TASK-0040 遗留）
- calc 当前为定时对账口径（config.changed 订阅缝占位）——本轮只验证「backend 发的 config.changed 真消息格式正确」，calc 消费不做（占位说明保留 TD 登记）

## 硬约束
- no-subagents；零源码改动（只 env/配置）；验证脚本临时文件不入库
- 所有服务用毕即停（8098/8090/8095/8099 释放）；scratch 数据（若有）测完清理
- 失败先定位是配置问题还是实现缺陷——实现缺陷写报告**不自行修**，等裁决
- 验证报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\EMQX-INTEGRATION-report.md`（逐项原始输出+TD-0056/0057 可否清账结论）

## 报告结构
五项：改动文件清单（应为零或仅配置）/ 每条验证命令原始输出 / 未验证项与原因 / 偏离计划之处 / 遗留问题。末尾一句话结论：TD-0056/0057 可清账 / 需补测（哪条）。写完即结束。
