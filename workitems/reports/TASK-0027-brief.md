# 实施 brief：TASK-0027（MQTT 消费与 HTTP 上报统一流水线——Go 道真集成第一环）

## 批次

- TASK-0027（本批共 1 个，标准任务 7 人天；**Go 道 `ingest/**` 纯 Go**——原含的 backend credential 触点已拆出为微改 TASK-0079（backend 道排队，ot-admin 单 Maven 模块整编译不可与 0015 并行，拆出才解锁双道并行））
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `ingest/**`（全部：消费源/幂等/快照真客户端/ingestd 接线）
  - 不碰 backend/**、frontend/**、workitems（报告除外）、契约文件（CONV-03 冻结，发现问题报告并停）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0027.md`
- Go 道交接包（协议+环境事实）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\GO-LANE-HANDOFF.md`
- ingest 现状自述：`D:\workspace\WisFactory\ingest\README.md`（假实现零改动替换原则/处理顺序冻结口径/并发模型起步口径）
- 关联对象：workitems 库 `features\FEAT-0009.md`、`requirements\REQ-0013.md`、`testcases\TC-0009.md`
- 上游交付：`reports\TASK-0076-report.md`（gwtoken/httpapi/authmqtt/register 缺口 I3）+ `reports\TASK-0076-brief.md`
- 契约三份（唯一权威）：`D:\workspace\WisFactory\resources\contract\`（统一消息模型 v1.1.0 / config-snapshot-schema v1.0.1 / 超级表 v1.0.0）
- 快照端点契约（真客户端对接对象）：`reports\TASK-0026-report.md`（GET /internal/v1/config/snapshot 响应形态+内部令牌）+ backend 源码 iot/publish/controller/IotPublishInternalController.java
- 上游代码（开工先读）：ingest/internal/{pipeline,message,snapshot,cache,httpapi,gwtoken,quality,tdengine} + cmd/ingestd/main.go（占位）

## 接口与决策（逐条结论，不再讨论）

1. **ingestd 接线**（cmd/ingestd/main.go 从占位转正）：装配 mux+httpapi+pipeline+consumer；配置走环境变量——`INGEST_HTTP_ADDR`（默认 `:8090`，**端口纪律：不占 8098/8099**）、`INGEST_BACKEND_BASE_URL`、`INGEST_INTERNAL_TOKEN`、`INGEST_MQTT_BROKER/USERNAME/PASSWORD`、`INGEST_REDIS_ADDR`（0029 才用，先留键位）。优雅停机（SIGTERM 排空）。
2. **MQTT 消费者**：`eclipse/paho.mqtt.golang`（go.mod 显式钉版本）实现 pipeline Consumer 接口（零改动替换假消费源）；订阅矩阵 §3.2 逐字：`iot/{tenantCode}/{gatewayCode}/data|status` 上行 + `iot/{tenantCode}/{gatewayCode}/config` 下行，LWT 遗嘱消息与 status 同主题——按主题路由成对应信封进流水线；QoS 1。**无 EMQX 环境 → mock 口径**：broker 侧抽象薄接口（照 MqttPublisherPort 范式），契约测试用假 broker 断言「MQTT 信封与 HTTP body 同构进同一流水线、两路结果一致」；真 broker 验证登记 **TD-0057**（挂 0068，文案给 4 步验证清单）。broker 不可达 fail-soft（指数退避重连、不阻断 HTTP 上报路径）。
3. **幂等真实现**：替换内存假去重——(pointCode, ts) 单条幂等 + batchId 整批幂等；**单实例形态用进程内有界 TTL 去重**（javadoc 登记多实例演进位=Redis 化）；原子性沿用 0076 修订轮 MarkIfSeen+per-key 串行化范式（BUG-0002 红线）；并发去重红线测试保留并扩到新实现。
4. **快照真客户端**：snapshot 拉取抽象替换为真 HTTP 客户端（GET /internal/v1/config/snapshot，内部令牌鉴权头；响应 version/snapshotHash/snapshot 体——哈希校验 + 原子替换复用 0073 缓存）；单测 httptest 假端点 + **验收冒烟打真 backend**（哈希核对）。config.changed 通知（backend 下发）本期不消费——拉取对账兜底已覆盖，通知消费归后续（TD 登记随 TD-0057）。
5. **gateway status 实校验（Go 侧先行，后端触点拆出）**：0076 authmqtt I3 缺口的 Go 侧承接——gwtoken/puller 消费 credential/pull 响应的 `gatewayStatus` 字段（**字段名即契约，后端由微改 TASK-0079 交付**，本批后端尚未带此字段：puller 对字段缺失/空值按「状态未知」处理并登记，不 fail-closed 不放行扩大）→ authmqtt 回调实校验（禁用网关 deny）；红线测试用 **fake puller**（返回禁用状态 → allow 即失败）——不依赖真实后端字段到位；真联调核验归 TASK-0068/后续 smoke（0079 交付后）。
6. **0028/0029 边界**：质量判定细节/纠偏豁免/死信计数存储/攒批真写/Redis 点位缓存/新值事件归 0028/0029——本任务 writer 保持记录式假执行器、cache 保持内存假实现（接口零改动替换原则，**不改 0028/0029 要换的实现**）；补传数据原样过流水线（纠偏豁免归 0028）。
7. 错误码纪律：契约码沿用 message 包（空 points 4001、单消息 ≤500 等），不自造。
8. Git 纪律：提交只 add `ingest/**` 与 credential 触点文件（**绝不 `git add -A`**，主仓同期有其他道提交）；提交注明 TASK-0027。

### 测试口径（TDD）

既有 Contract 全套保持全绿（零改动替换原则的直接证据）；新增：MQTT 消费者路由矩阵（四主题→信封类型）、两路上报一致性（同一 batchId/payload 经假 broker 与 HTTP 入口流水线结果逐字段一致）、幂等（单条重复/整批重复/并发红线）、authmqtt 禁用网关 deny 红线、快照真客户端哈希校验+原子替换、ingestd 装配冒烟。红转绿留痕。

## 验收命令（整体验收）

环境事实（协调者 2026-11-23 实测）：WSL docker wisfactory-mysql（8.0.42@3307 root/root ot_wisfactory）/wisfactory-redis（7.2.4；**Go 测试用 db1+**，backend 用 db0）；TDengine 已根治可用但**本任务 writer 为假执行器不碰 TDengine**；8098 归 backend 冒烟、8090 归 ingestd——启动前探测、用毕即停；WSL2 空闲关停→长间隙前保活循环。

1. `cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（含既有 Contract 全套）
2. live 冒烟（每步原始输出贴回）：backend dev 起（8098；**启动前探测，被占则等待重试配保活**——backend 道可能有并行任务）→ ingestd 起（8090）→ 快照真拉取（iot_default scope，哈希核对）→ /ingest/v1/report 两批上报（合法批落流水线 + 未知编码批死信不落库）→ authmqtt 禁用网关 deny（fake puller）→ 全停
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内（纯 ingest/**）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0027-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题（含 TD-0057 登记文案）。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件；不改冻结契约。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0027）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
