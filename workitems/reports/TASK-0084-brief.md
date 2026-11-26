# 实施 brief：TASK-0084（新值事件到 MQTT 数据项订阅主题的桥接生产者）

## 批次

- TASK-0084（微改任务直接已确认；ingest 道——前置 0029✅；闭合 TASK-0053 评审 I-2 计划缺陷，收官 FEAT-0017）
- BASE：开工 HEAD（记录进报告）；只改 `ingest/**`；不碰 backend/frontend/calc/契约/workitems（报告除外）

## 必读

- TASK 详情 + FEAT-0017/REQ-0028/TC-0017
- 契约：`resources/contract/unified-message-model.md` v1.1.0「3.5」推送行——数据项订阅主题 `iot/open/{tenant_code}/{instance_code}/item`、消息体统一消息模型 **4 字段 fullKey/ts/value/quality**（camelCase/毫秒，0053 已定版）
- 上游：reports\TASK-0029-report.md（Redis `iot:newvalue:{tenant}` 发射侧，payload {tenant,pointCode,ts,value,quality}）+ reports\TASK-0053-report.md（消费侧订阅 iot/open，4 字段读口）+ ingest/internal/event/** 现有代码（Pub/Sub 客户端/Publisher 接口）+ ingest/internal/mqttsource 或 mqtt 客户端（如有）

## 接口与决策（逐条结论）

1. **桥接组件**：订阅 Redis `iot:newvalue:{tenant}`（复用 0029 event 包 RESP2 客户端，不新增依赖）→ 将 {tenant,pointCode,ts,value,quality} 映射为统一消息模型 4 字段 {fullKey,ts,value,quality} → MQTT publish 到 `iot/open/{tenant}/{instance}/item`（Paho 客户端 env broker 可切，与 0053 订阅端同主题）
2. **pointCode→fullKey/instance 映射**：按 0036 既定口径（pointCode 即 fullKey 核心，字段纪律"现场只认 pointCode 不认 id"不改）——从 pointCode 解析 instanceCode（pointCode 形如 `F01.WS01.EQ001.M01` → instance 取设备段，具体以 0036/0034 映射函数为准，开工先读 ingest/internal/tdengine 的 childTable/fullKey 映射复用）；映射失败→丢弃+WARN 不发布（不造假主题）
3. **opt-in**：env `INGEST_ITEM_BRIDGE_ENABLED` 默认 **false**（0053 订阅端未就绪时不无效发布）；开启后独立 goroutine 订阅+发布，ctx 取消退出（0041 ValueStore/StatusEventSink 生命周期范式）
4. **租户隔离**：主题含 tenant、payload 不含 tenant（契约 4 字段）——tenant 只走主题段（0053 消费侧按主题+appKey scope 过滤，已实证租户 B 隔离）；桥接不注入额外 tenant 字段进 payload
5. **无订阅者不报错**：MQTT QoS1 fire-and-forget；Redis 订阅断线重连指数退避（0027 fail-soft 范式）；桥接失败只 WARN 不影响 ingest 主流水线（验令牌→查快照→死信→质量→写库）
6. **不碰**：ingest 写库/校验/快照主链、0053 消费侧、backend、契约文件

### 测试口径（TDD）

Contract 测试（newvalue payload→4 字段映射精确/dts 原样/quality 透传/pointCode→fullKey·instance 映射正确/映射失败丢弃/主题逐字/opt-in 默认关/租户主题隔离/无订阅者不报错/ctx 取消退出）；既有契约测试零回归；红转绿留痕。

## 验收命令

环境：Redis 宿主 phpStudy 3.0.504@6379；EMQX 1883 退化（本地真协议夹具 broker 验证，生产侧 env 可切）；TDengine REST 故障本批不涉；端口纪律。

1. `cd ingest && go build ./... && go vet ./... && go test ./...` → 0 失败 0 错误（基线零回归+新增，Results 段贴回）
2. live 冒烟：本地夹具 broker + 真 Redis → 开启桥接 → PUBLISH 一条 newvalue → MQTT 订阅 iot/open/+/+/item 真收 4 字段消息（主题 tenant/instance 正确）→ 映射失败样本丢弃 → 关闭开关验证无发布 → 全停
3. 范围核对：git diff --name-only BASE..HEAD 全落 ingest/**

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0084-report.md。五项齐全 + 一句话结论。

## 契约

- no-subagents；提交只 add ingest/**；不把 TASK 置「已完成」；不改 workitems（报告除外）；契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
