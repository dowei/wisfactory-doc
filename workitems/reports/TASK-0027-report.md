# TASK-0027 报告：MQTT 消费与 HTTP 上报统一流水线（Go 道真集成第一环）

- 执行者：TASK-0027 执行（brief 驱动，纯 Go 批次：只改 ingest/**）
- BASE：`d02ac44`（开工时主仓 main HEAD；同期含 0070 batch1 前端提交，与本次无关）
- 交付提交：`e1b6266`（唯一提交，23 文件全部 ingest/**，注明 TASK-0027）
- 执行日期：2026-11-23
- 验收结论：**3/3 验收命令全部通过**（原始输出见 §2；live 冒烟含一处 live 实证驱动的语义修正，红转绿留痕见 §4.2）

## 1. 改动文件清单（23 个，全部落 ingest/**；`git show --name-only e1b6266` 核对非 ingest 文件数=0）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `ingest/cmd/ingestd/main.go` | 重写 | 占位转正（决策 1）：装配 mux+httpapi+pipeline+mqttsource.Consumer+Reconciler 对账；环境变量配置；优雅停机（SIGTERM→HTTP 排空→流水线/消费源/对账取消→broker 连接释放）；纯 HTTP 形态（MQTT 未配置）走 StartFlusher |
| `ingest/cmd/ingestd/main_test.go` | 新增 | 装配冒烟 3 项（配置默认值/环境变量装配/全组件真装配启动→healthz→fail-closed 401→优雅停机→端口释放） |
| `ingest/internal/mqttsource/mqttsource.go` | 新增 | MQTT 消费源（决策 2）：§3.2 主题矩阵路由（data→统一消息信封进流水线〔实现 pipeline.Consumer 零改动替换假消费源〕、status→状态信封〔LWT graceful 区分〕、config→通知信封〔本期不消费〕）；Port 薄接口；监督循环指数退避 fail-soft；有界队列背压（满则阻塞不丢弃）；畸形主题守卫计数 |
| `ingest/internal/mqttsource/paho.go` | 新增 | paho.mqtt.golang v1.5.1 适配（go.mod 显式钉版本）：初连快速失败（重连节奏归监督循环）、AutoReconnect+ResumeSubs+CleanSession=false、QoS 1 订阅、连接丢失回调 |
| `ingest/internal/mqttsource/mqttsource_contract_test.go` | 新增 | 契约测试 3 项（SplitTopic 主题矩阵逐字、订阅矩阵 `iot/+/+/data|status|config`、四主题→信封类型路由矩阵〔data 进消费源/status×3 含 LWT 与 graceful/config 通知〕） |
| `ingest/internal/mqttsource/mqttsource_failsoft_contract_test.go` | 新增 | 契约测试 3 项（broker 不可达指数退避 50→100→200ms 实测断言、断线重连重订阅+ctx 停机释放、有界队列背压零丢弃） |
| `ingest/internal/mqttsource/twopath_contract_test.go` | 新增 | 两路一致性 2 项（镜像双流水线逐字段一致：计数/3 条 INSERT SQL 逐字节/缓存 v-q-ts/死信；同一流水线跨通道幂等：broker 投递后 HTTP 重报同 batchId→duplicate=true、落库恰一次） |
| `ingest/internal/pipeline/dedup_ttl.go` | 新增 | 幂等真实现（决策 3）：TTLBatchDeduper/TTLPointDeduper——进程内有界 TTL 表（单锁原子 MarkIfSeen 保持 0076 修订轮 C1 语义；容量 FIFO 淘汰+TTL 惰性过期回收双保险；代次隔离防误删；多实例演进位=Redis 化 javadoc 登记） |
| `ingest/internal/pipeline/dedup_ttl_contract_test.go` | 新增 | 契约测试 6 项（原子语义/窗口内重报/TTL 过期重放/点级键粒度/容量 FIFO 淘汰/过期回收/默认值） |
| `ingest/internal/pipeline/pipeline_submit_ttl_race_contract_test.go` | 新增 | 并发红线扩到 TTL 真实现 3 项（30 轮×12 goroutine 同 batchId 恰一处理、同 (pointCode,ts) 跨批恰一行写出、串行单条+整批+跨批幂等） |
| `ingest/internal/snapshot/client.go` | 新增 | 快照真客户端（决策 4）：InternalSnapshotClient——GET /internal/v1/config/snapshot + X-Internal-Token（+tenant 消歧可选参数），Result<InternalSnapshotVO> 信封解析、哈希锚点形态校验（64 位小写 hex）、Schema 校验、信封/体 version 同值校验；§4.2 live 修正后哈希=对账锚点口径 |
| `ingest/internal/snapshot/reconciler.go` | 新增 | 拉取对账循环（决策 4：通知本期不消费、拉取兜底）：启动即拉、周期对账、同版本哈希漂移探测（fail-soft 拒收保持旧版）、版本哈希稳定跳过、失败保持旧版计数告警 |
| `ingest/internal/snapshot/client_contract_test.go` | 新增 | 契约测试 9 项（请求形态含内部令牌头、version 缺省+tenant 参数、哈希锚点非重算口径、锚点形态校验、version 不同值拒收、非 200/业务码失败、非法快照体拒收、不可达有限超时、喂 0073 缓存原子替换） |
| `ingest/internal/snapshot/reconciler_contract_test.go` | 新增 | 契约测试 6 项（新版本替换、失败保持旧版+恢复、同版本跳过、同版本哈希漂移拒收+恢复收敛、启动即拉、ctx 取消退出） |
| `ingest/internal/gwtoken/puller.go` | 修改 | 决策 5：pull 响应解析扩 `gatewayStatus *int`（字段名即契约，TASK-0079 交付；nil=缺失/null=状态未知）→ Credential 原样承载 |
| `ingest/internal/gwtoken/gwtoken.go` | 修改 | 决策 5：GatewayIdentity 扩 `GatewayStatusKnown/GatewayDisabled`（0076 修订轮 I3 扩展位转正）；GatewayStatusEnabled/Disabled 常量；Validate 从 Credential 承载（仅显式 0=禁用）；包注释同步 |
| `ingest/internal/gwtoken/gwtoken_gatewaystatus_contract_test.go` | 新增 | 契约测试 3 项（禁用承载、未知不 fail-closed 不放行扩大、启用区分） |
| `ingest/internal/gwtoken/puller_gatewaystatus_contract_test.go` | 新增 | 契约测试 3 项（gatewayStatus 0/1 解析、缺失/null=未知非错误、既有 fail-closed 纪律不被破坏） |
| `ingest/internal/httpapi/authmqtt.go` | 修改 | 决策 5：回调实校验——`GatewayStatusKnown && GatewayDisabled → deny`（原因进服务端日志）；状态未知保持 0076 行为；包/方法注释同步（I3 缺口 Go 侧闭合） |
| `ingest/internal/httpapi/authmqtt_gatewaystatus_contract_test.go` | 新增 | 红线测试 3 项（fake puller 禁用→deny〔allow 即失败〕、启用不误伤、未知〔fake+真实链路双口径〕保持 allow） |
| `ingest/README.md` | 修改 | 目录描述同步（ingestd 已接线、mqttsource/gwtoken/httpapi 新目录、snapshot 真客户端与对账、pipeline 幂等 TTL） |
| `ingest/go.mod` | 修改 | 新增依赖：github.com/eclipse/paho.mqtt.golang **v1.5.1**（显式钉版本，决策 2）+ 传递依赖 gorilla/websocket v1.5.3、golang.org/x/net v0.44.0、golang.org/x/sync v0.17.0 |
| `ingest/go.sum` | 新增 | 模块校验和（随 paho 引入） |

依赖纪律：新增第三方依赖 1（paho，brief 决策 2 授权钉版本）。新增契约测试 **41 项**（全部 `Contract` 命名）+ ingestd 装配冒烟 3 项（非契约命名）；**0073/0076 既有 94 项契约测试零改动全绿**（零改动替换原则的直接证据）。

## 2. 验收命令原始输出

### 2.1 验收 1：`cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（EXIT=0）

```text
ACCEPT-1 BUILD+VET EXIT=0          （build/vet 无任何输出，双零退出）

ok  	net.otlab/ingest/cmd/ingestd	2.636s
ok  	net.otlab/ingest/internal/cache	0.846s
ok  	net.otlab/ingest/internal/gwtoken	2.027s
ok  	net.otlab/ingest/internal/httpapi	1.885s
ok  	net.otlab/ingest/internal/message	1.007s
ok  	net.otlab/ingest/internal/mqttsource	1.876s
ok  	net.otlab/ingest/internal/pipeline	2.164s
ok  	net.otlab/ingest/internal/quality	0.614s
ok  	net.otlab/ingest/internal/snapshot	2.265s
ok  	net.otlab/ingest/internal/tdengine	2.496s
ACCEPT-2 TEST EXIT=0
```

契约符合性全套（`go test ./... -run 'Contract' -v`）：**`--- PASS` 135 行、`--- FAIL` 0 行**（0073/0076 既有 94 + 本次新增 41）。分包计数：pipeline 31、snapshot 34、httpapi 18、gwtoken 16、mqttsource 8、message/quality/cache/tdengine 28（既有）。

并发红线稳定性补跑（TTL 真实现扩展 + 0076 既有 Memory 版）：

```text
go test ./internal/pipeline/ ./internal/mqttsource/ -run 'Race' -count=3
ok  	net.otlab/ingest/internal/pipeline	1.738s
go test ./internal/pipeline/ -run 'Race' -count=10（中途复跑）
ok  	net.otlab/ingest/internal/pipeline	1.033s
```

### 2.2 验收 2：live 冒烟全序列（每步原始输出）

**环境处置（偏离 1，详见 §4.1）**：探测 8098/8090 全空闲、WSL wisfactory-mysql/redis 在跑（协调者 2026-11-23 实测一致）；backend dev 实例未运行且本任务禁跑 mvn（TASK-0015 道在途）——以**既有构建产物** `backend/ot-admin/target/ot-admin-1.0.0.jar`（2026-11-23 01:15 构建，含 0026 交付代码）`java -jar` 起 dev 实例（零编译零 mvn、对 backend 源码零写入），OT_GATEWAY_INTERNAL_TOKEN=tok-0027-smoke，启动日志 `Started OtAdminApplication in 11.229 seconds`、启动段 `grep -c " ERROR "`=**0**；WSL 保活循环全程未复发空闲关停。

**步骤 0：建联调数据（backend API，dev /auth/login/test 登录）**

```text
POST /auth/login/test {"username":"admin","password":"admin123","tenantCode":"default"}
→ {"code":200,...token":"e795242e-96fc-4bd2-8e69-1e144ffbb338"...}

POST /iot/channel/credential {"channelId":21,"credType":"HTTP_SECRET","credential":"gw-0027-secret-material"}
→ {"code":200}   （通道 21=0026 冒烟遗留 SMOKE0026_CH1；凭据为 0026/0019 链路首次真联调所需）
POST /iot/channel/credential {"channelId":21,"credType":"MQTT_PASSWORD","credential":"gw-0027-mqtt-material"}
→ {"code":200}

GET /internal/v1/credential/pull?channelId=21&credType=HTTP_SECRET（内部令牌）
→ {"code":200,"data":{"channelId":21,"ciphertext":"{v1}P5+Wy8nbOBgZNBebrcn4mA==:JOpiCNM1jv068aG7XYaRiIZUmpYaE1DliHIIJ6frrS0=","credType":"HTTP_SECRET","keyVersion":"1"}}
—— 响应**无 gatewayStatus 字段**（TASK-0079 交付前现状，正是决策 5「状态未知」路径的真实形态）

POST /iot/publish/publish {"scopeType":1,"scopeId":9111}
→ {"code":200,"data":{"id":31,...,"version":5,"snapshotHash":"415c24f63d45fd7b410949d72156854a0012a0a14172da5e85185200738da473","snapshotUri":"1/1-9111-5.json","publishStatus":1,...}}
—— v4 快照文件为 0026 收口清理残留缺口（500 快照文件缺失），发布 v5 重建可用快照；确认门一档放行
GET /internal/v1/config/snapshot?scopeType=1&scopeId=9111&tenant=default → 200 version=5，
  快照体含点位 SMOKE0026_P1（id=12021, dataType=1, channelId=21, status=1）、通道 SMOKE0026_CH1
```

**步骤 1：ingestd 起（8090）→ 快照真拉取（哈希核对）**

装配 env：`INGEST_HTTP_ADDR=:8090`、`INGEST_BACKEND_BASE_URL=http://127.0.0.1:8098`、`INGEST_INTERNAL_TOKEN=tok-0027-smoke`、`INGEST_GATEWAY_MAP=SMOKE0026_GW:21:default`、`INGEST_MQTT_BROKER=tcp://127.0.0.1:18883`（无 EMQX——同时作为 fail-soft 实测注入）、`INGEST_SNAPSHOT_SCOPE=1:9111`、`INGEST_SNAPSHOT_TENANT=default`、`INGEST_SNAPSHOT_INTERVAL=10s`、`INGEST_DEDUP_TTL=1h`、`INGEST_REDIS_ADDR=127.0.0.1:6379`（键位预留）。

```text
2026/11/23 02:41:14 ingestd: MQTT 消费源已装配 broker=tcp://127.0.0.1:18883
2026/11/23 02:41:14 ingestd: 快照对账已装配 scope=1:9111 tenant="default" interval=10s
2026/11/23 02:41:14 ingestd: HTTP 接入面已启动 [::]:8090（report/auth-mqtt/healthz）
2026/11/23 02:41:14 snapshot: 对账快照已替换 version=5 hash=415c24f63d45fd7b410949d72156854a0012a0a14172da5e85185200738da473
curl http://127.0.0.1:8090/healthz → ok
```

**哈希核对**：ingestd 承载的 snapshotHash `415c24f6...` 与 backend 版本行（发布响应）**逐字一致**，且与原始快照文件 `sha256sum backend/ot-admin/data/snapshots/1/1-9111-5.json` → `415c24f63d45fd7b410949d72156854a0012a0a14172da5e85185200738da473` **三方一致**（锚点核对口径——修正过程见 §4.2）；其后 ~3 分钟 × 10s 对账零重复替换（同版本哈希稳定跳过，替换行仅 1 条）。

**步骤 2：/ingest/v1/report 两批上报（合法批落流水线 + 未知编码批死信不落库）**

```text
=== 批1（合法点位 SMOKE0026_P1，X-Gateway-Token=credential/pull 密文）===
{"accepted":1,"rejected":0,"rejections":[],"duplicate":false}

=== 批2（未知编码 SMOKE_X99_UNKNOWN）===
{"accepted":0,"rejected":1,"rejections":[{"pointCode":"SMOKE_X99_UNKNOWN","ts":1795372701000,"reason":"unknown_code"}],"duplicate":false}

=== 批1重报（同 batchId 幂等）===
{"accepted":1,"rejected":0,"rejections":[],"duplicate":true}
```

批1 accepted=1=令牌链真联调通过（HTTP_SECRET 密文比对）+ 合法点位过流水线受理；批2 未知编码死信、不落库（reason=unknown_code）；重报 duplicate=true（QoS1 整批幂等）。writer 为记录式假执行器（决策 6——0029 换真写），落库形态由契约测试锁定（全限定 INSERT）。

**步骤 3：authmqtt 禁用网关 deny（fake puller 红线）+ 真链路 allow/deny**

```text
=== 禁用网关 deny 红线（fake puller，契约测试——不依赖 0079 字段到位）===
go test ./internal/httpapi/ -run 'TestContractMQTTAuthDenyOnDisabledGatewayRedLine' -count=1 -v
--- PASS: TestContractMQTTAuthDenyOnDisabledGatewayRedLine (0.00s)
ok  	net.otlab/ingest/internal/httpapi	1.437s

=== auth/mqtt 正确网关令牌（MQTT_PASSWORD 密文；gatewayStatus 缺失=状态未知→0076 行为）===
{"result":"allow","topics":{"publish":["iot/default/SMOKE0026_GW/data","iot/default/SMOKE0026_GW/status"],"subscribe":["iot/default/SMOKE0026_GW/config"]}}

=== auth/mqtt 错误令牌 → deny ===
{"result":"deny"}

=== auth/mqtt 内部令牌错误 → 401+deny ===
{"result":"deny"} [http 401]
```

**步骤 4：broker 不可达 fail-soft（指数退避、不阻断 HTTP 上报路径）——ingestd.log 实测**

```text
02:41:14 ... 500ms 后重试——指数退避 ... dial tcp 127.0.0.1:18883: connectex: No connection could be made...
02:41:14 ... 1s 后重试——指数退避 ...
02:41:15 ... 2s 后重试——指数退避 ...
02:41:17 ... 4s 后重试——指数退避 ...
02:41:21 ... 8s 后重试——指数退避 ...
02:41:29 ... 16s 后重试——指数退避 ...
02:41:45 ... 30s 后重试——指数退避 ...（封顶）
—— 共 7 次退避重试期间，步骤 2/3 的 report 与 auth/mqtt 全部正常应答（HTTP 路径零受扰）
```

**步骤 5：全停**

```text
taskkill /PID 17112（ingestd）/ taskkill /PID 55312（backend）→ 均「成功: 已终止」
netstat 8090/8098 LISTENING → 0（端口归零，用毕即停）
```

（注：live 优雅停机受 Windows 控制台限制——taskkill 为硬杀、无 SIGTERM 投递面；优雅停机路径（HTTP 排空→流水线取消→broker 连接释放）由 `TestIngestdAssemblySmoke` 单元级实证，见 §4.6。）

**步骤 6：backend 全程日志核对**：`grep -c " ERROR "` = 7，逐条核对全部为冒烟探针的**预期业务错误**（v4 快照文件缺失 2 条〔0026 清理残留，发布 v5 前的探测〕、无令牌探测 401 一条、凭据创建前 pull 403 两条、MQTT_PASSWORD 凭据创建前 auth 探测 deny 两条）——最终流程（凭据就绪后）零新增错误。

### 2.3 验收 3：改动范围核对

交付提交 `git show --name-only e1b6266`：23 文件全部 `ingest/**`，非 ingest 文件数 = **0**；`git status --porcelain -- ingest/` = 空（工作区干净）。`git diff --name-only d02ac44..HEAD` 中另有的 29 个非 ingest 文件属并行的 TASK-0015 道提交（fa064cd/83b99c3/b3d1e0c 等，本任务未触碰——见 §5.7）。

## 3. 未做项与原因（brief 明确归后/边界外，非缺口）

| 未做项 | 归属 | 原因 |
| --- | --- | --- |
| EMQX 真 broker 联调（PahoPort 真链路、QoS1 会话、LWT 真遗嘱） | TD-0057（挂 0068） | 本地无 EMQX（GO-LANE-HANDOFF 实测）；mock 口径：Port 薄接口 + 假 broker 契约测试锁行为，4 步验证清单文案见 §5.1 |
| config.changed 通知消费（通知触发即时拉取） | 后续任务（TD 登记随 TD-0057） | brief 决策 4 明示本期不消费——拉取对账兜底已覆盖（live 步骤 1 的 10s 对账即兜底实证）；通知信封路由/解析已就位（ConfigSink 挂点） |
| writer 真 TDengine 写（REST 执行器接线）、Redis 点位缓存真实现、新值事件 | 0028/0029 | brief 决策 6：接口零改动替换原则，不改 0028/0029 要换的实现——writer=记录式假执行器、cache=内存假实现原样保留 |
| 质量判定细节（量程二次判定）、纠偏豁免、死信计数存储 | 0028 | 同上；补传数据原样过流水线（PassthroughJudge 直通） |
| 网关→通道解析真实现（校验令牌归属通道/网关注册表） | 0020 | 0076 mock 口径沿用：INGEST_GATEWAY_MAP 静态映射（env 配置化），接口（GatewayChannelResolver）留位零改动替换 |
| credential/pull 的 gatewayStatus 字段（后端微改） | TASK-0079 | 本批后端尚未带字段（live 实测响应无此键）——Go 侧消费+状态未知处理+红线测试已就位（决策 5），真联调核验归 0068/0079 交付后 |
| `-race` 并发检测 | （环境限制） | 本机无 gcc（cgo 缺失）——同 0076 口径；并发安全由竞态回归测试行为断言覆盖（Memory 版 2 项 + TTL 版 2 项，30 轮×12 goroutine） |
| 多租户/多网关快照分片（跨租户快照隔离完整演示，TC-0009 步骤 8） | 演进位（§5.6 登记） | 冻结流水线 SnapshotLookup 按 pointCode 单缓存查点；本批按单 scope 起步形态交付（INGEST_SNAPSHOT_SCOPE 单值），分片需流水线租户维度扩展——非本批 brief 交付面 |

## 4. 偏离计划之处

### 4.1 backend dev 以既有 jar 起动（非 mvn）

brief 口径「backend dev 起（8098）」与任务指令「不许跑 mvn/编译 backend」在 8098 空闲时冲突。处置：以 2026-11-23 01:15 既有构建产物 `java -jar` 起 dev 实例——零编译、零源码写入、零测试执行，对 TASK-0015 道零干扰；实例仅在冒烟段存活（起前探测空闲、用毕即停、端口归零确认）。曾向用户发起处置询问未获答复，按约束优先原则自主裁决。

### 4.2 哈希校验语义修正（live 实证驱动的红转绿，本报告最重要偏离）

初版 InternalSnapshotClient 按「SHA-256(响应快照体字节) == snapshotHash」实现重算校验。**live 首拉即红**：

```text
snapshot: 对账拉取失败（fail-soft 保持旧版，下个窗口重试）: snapshot: 快照哈希核对失败
version=5 expected=415c24f6...（backend 版本行）actual=c2c4eb7e...（响应体重算）
```

根因：后端 `snapshotHash = SHA-256(原始存储字节)`（CANONICAL_MAPPER 序列化，`minValue:5.000000`）；而内部端点出站快照体是 Jackson 3 **再序列化形态**（`InternalSnapshotVO.snapshot` 经 JsonNode 往返，`5.000000→5.0`）——两者不逐字节一致，下游从响应体**不可重算**该哈希（原文件字节 sha256=415c24f6 实证比对）。fail-soft 链路（保持旧版+窗口重试+HTTP 路径不受扰）在 live 中即被此缺陷实测验证，行为正确。

修正：哈希校验收敛为**对账锚点口径**（§3.3「拉新快照、校验哈希」的可实现语义）——客户端校验锚点形态（64 位小写 hex，非法拒收）+ 原样承载；同版本哈希稳定性归 Reconciler（漂移=存储被篡改/端点异常探测，fail-soft 拒收）；跨进程比对（通知 hash vs 响应 snapshotHash）挂点留给通知消费。为什么现在对了：客户端能校验的是「端点声明的锚是否可对账」，而非「再序列化体的重算哈希」——锚点三方一致（响应 snapshotHash = 版本行 = 存储文件字节）由 live 步骤 1 实证；快照体完整性由 ParseSnapshot Schema 校验 + 后端取回侧哈希核对（readAndVerifySnapshot 500）承担。契约留白（§3.3 未定义哈希字节域）登记见 §5.2，按 CONV-03 纪律不改契约、报告待裁决。

### 4.3 订阅矩阵以 `+` 通配承载 {tenant}/{gateway} 槽位

平台侧消费者订阅 `iot/+/+/data|status|config`（矩阵主题形态 §3.2 逐字，槽位通配）；per-gateway 主题级隔离由 EMQX ACL（auth/mqtt 回调主题范围）执行——矩阵逐字性保持，消费面按平台规模通配（路由从主题槽位解析 tenant/gateway，租户=认证授予主题矩阵=令牌侧产物，满足处理顺序第 1 步）。

### 4.4 status/config 信封路由到独立出口（不进数据流水线 processOne）

「按主题路由成对应信封进流水线」的落地解读：data 信封→统一数据流水线（processOne 五步冻结顺序）；status→StatusSink（默认计数+日志，LWT 告警语义挂点——online_status 存储归联调）；config→ConfigSink（本期不消费，计数+日志）。理由：status/config 载荷非统一消息数据信封（status 契约未冻结字段级 Schema、config 为 §3.3 五键通知），走 processOne 会被信封纪律整批拒绝为 invalid_envelope——违背 §3.2 对两主题的语义。出口为接口挂点，后续接线零改动替换。

### 4.5 ingestd 配置键位补充

决策 1 六键之外新增装配必需键位：`INGEST_GATEWAY_MAP`（静态解析 mock 口径）、`INGEST_SNAPSHOT_SCOPE/INGEST_SNAPSHOT_TENANT/INGEST_SNAPSHOT_INTERVAL`（对账范围与周期）、`INGEST_DEDUP_TTL`（幂等窗口）、`INGEST_MQTT_CLIENT_ID`——全部 ingest/** 内，javadoc 逐键登记用途与归属。

### 4.6 live 优雅停机证据形态

Windows 控制台进程无 SIGTERM 投递面（taskkill=硬杀）；优雅停机（HTTP Shutdown 排空→pctx 取消→消费源退出释放 broker 连接→run 返回）由 `TestIngestdAssemblySmoke` 全链路实证（ctx 取消→run 返回→Port.Close 断言）；live 段按「用毕即停+端口归零」口径收口。

## 5. 遗留问题

### 5.1 [TD-0057 登记文案]

**[TD-0057] TASK-0027 MQTT 真联调缺口**：MQTT 消费源已按 mock 口径交付（`mqttsource.Port` 薄接口隔离 paho.mqtt.golang v1.5.1；主题矩阵路由/四主题→信封类型/两路上报一致性/fail-soft 指数退避/有界背压/断线重订阅由 8 个假 broker 契约测试锁定），`PahoPort` **未经真实 broker 验证**（本地无 EMQX）。登记动作：部署/联调环境补 `wisfactory-emqx` 容器后，置 `INGEST_MQTT_BROKER=tcp://<emqx>:1883`（账号经 INGEST_MQTT_USERNAME/PASSWORD 注入；EMQX 侧需配 HTTP 认证回调指向 `/ingest/v1/auth/mqtt` + 内部令牌头），执行 4 步验证清单：
① **路由矩阵真链路**：网关按回调授予主题发 data/status（QoS 1）→ ingestd 日志 DataRouted/StatusRouted 递增、合法点位进流水线落库、未知编码死信计数；LWT（异常断开，无 graceful 标记）与正常下线（graceful:true）经状态出口区分接收。
② **config.changed 下行**：管理系统发布网关 scope → 订阅 `iot/<tenantCode>/<gatewayCode>/config` 收到五键通知（本期消费=计数+日志；通知消费接线后验证 hash 触发即时拉取+对账替换）。
③ **fail-soft 反向验证**：kill broker 后恢复 → ingestd 指数退避重连日志收敛（500ms→30s 封顶）、期间 HTTP 上报路径全程可用、恢复后自动重连重订阅（ResumeSubs/CleanSession=false 会话与重投保持）。
④ **QoS1 至少一次端到端**：broker 重投同 batchId → 整批幂等 duplicate=true 恰一落库；(pointCode, ts) 跨批重投 → 点级幂等恰一行（TTL 幂等窗口 1h 内）。
登记人：TASK-0027 执行者；建议归属：TASK-0068 接入联调（或环境就绪后的修订轮）。

### 5.2 [契约留白发现——待协调者裁决，本批未改契约]

§3.3「响应含 version、snapshotHash 与快照体……拉新快照、校验哈希」未定义 snapshotHash 的**字节域**：实测为原始存储字节的 SHA-256，而出站快照体为 Jackson 再序列化形态（BigDecimal `5.000000`→DoubleNode `5.0`），下游不可重算——「校验哈希」在当前端点形态下只能落为对账锚点比较（本批语义）。候选修复（归协调者/CONV-03 升版裁决，Go 侧不动）：a) 内部端点以原文 String 回传快照体（管理面 SnapshotVO 同口径）；b) 契约定义「规范哈希」（对再序列化后的确定形态约定字节域）；c) 维持对账口径并在契约注明哈希字节域=存储原文。修复前下游完整性依赖：Schema 校验 + 同版本哈希稳定性 + 后端取回侧核对（已有）。

### 5.3 [gatewayStatus 边界]

TASK-0079 交付前：真实 credential/pull 响应无 gatewayStatus（live 实测）→ 状态未知路径=0076 行为（网关级禁用会 allow）；0079 交付后行为自动切换（显式 0→deny，红线测试已锁）。真联调核验（0079 字段到位后禁用网关端到端 deny）归 TASK-0068/后续 smoke。

### 5.4 [冒烟数据留存]

DB 留存：网关 9111/通道 21（0026 遗留）+ 本次新增 HTTP_SECRET/MQTT_PASSWORD 凭据各 1 行（channelId=21，key_version=1）+ 快照版本 5 行（id=31）；快照文件 `backend/ot-admin/data/snapshots/1/1-9111-5.json` **保留未删**（gitignore 覆盖）——v5 为当前最新版，删除将复现 v4 式「文件缺失」缺口。凭据明文未入任何提交/报告。

### 5.5 [口径登记]

- TTL 幂等窗口默认 1h（INGEST_DEDUP_TTL 可配）：必须 ≥ 网关 QoS1 重投窗口；容量默认 65536，超限 FIFO 淘汰最早标记项（ADR-0014 口径）——超窗口/超容量的重投按新键处理（javadoc 已登记）。
- 状态载荷 graceful 标记键名 `graceful`（契约原文「须发 graceful 标记区分」但键名未冻结）——真联调时若网关实际键名不同，只改 parseStatus 宽松解析一处。
- ingestd 纯 HTTP 形态（INGEST_MQTT_BROKER 空）合法存在：idleConsumer+StartFlusher，MQTT 未配置不构成启动失败。

### 5.6 [多租户快照分片演进位]

本批单 scope 起步形态（INGEST_SNAPSHOT_SCOPE 单值→单 Cache）：多网关/多租户需按 scope 分片 Cache + 流水线租户维度查点（SnapshotLookup 接口扩展或 Inbound 携带 scope）——冻结接口的扩展面归协调者裁决后接线（0076 已预登记「0027 定版」，本批以单 scope 落地并把扩展面显式登记）。

### 5.7 [环境与并行]

- 本机无 gcc：`-race` 未跑（同 0076 登记）；建议有 gcc 环境对 pipeline（TTL 去重双入口并发面）/mqttsource 补一轮 `go test -race`。
- 并行 TASK-0015 道工作区改动/提交（backend/**）全程未触碰、未编译、未测试；本任务 backend 交互仅 live 冒烟段（既有 jar 起停 + HTTP API 数据操作）。
- WSL 保活循环全程未复发空闲关停；8090/8098 起停均探测确认。
- go.mod 传递依赖（gorilla/websocket、x/net、x/sync）随 paho 进入，非本任务直接引用。

## 可追溯性

- **git 提交**：`e1b6266`（main，仅 ingest/** 23 文件，注明 TASK-0027）
- **修改文件**：§1 清单 23 项（8 修改 + 15 新增）；非 ingest 文件数 = 0
- **验收**：§2 三条命令原始输出全部通过；契约测试 135/135 PASS（既有 94 零改动）

## 六、一句话结论

**通过**——验收 1（build/vet 双零退出 + go test 全包 ok + 契约 135 PASS/0 FAIL + Race -count=3/-10 稳定）、验收 2（live 冒烟全序列：快照真拉取 version=5 哈希三方一致〔对账锚点口径，live 实证驱动的 §4.2 修正红转绿〕、report 合法批 accepted=1/未知编码批 rejected=1 reason=unknown_code 不落库/重报 duplicate=true、authmqtt 真链路 allow+主题范围与 deny、禁用网关 deny 红线 fake puller PASS、broker 不可达指数退避 500ms→30s 封顶 7 次期间 HTTP 路径零受扰、全停端口归零）、验收 3（交付提交 23 文件全落 ingest/**、越界 0）三条全部以原始输出为证通过；偏离均为 live 实证驱动的语义修正与口径登记（§4），遗留仅 TD-0057 真联调、契约留白 §5.2 待裁决与演进位登记，无阻断缺陷。
