# TASK-0027 评审报告：MQTT 消费与 HTTP 上报统一流水线（diff 评审）

- 评审者：代码评审子代理（fresh，仅依据评审 brief「必读」清单与工作区源码判断）
- 评审对象：提交 `e1b6266`（BASE=d02ac44；`e1b6266^`=b3d1e0c，diff 包按 `e1b6266^..e1b6266` 隔离，23 文件）
- 评审日期：2026-11-23
- 评审口径：L1-execute §13「diff 评审」+ 评审 brief 评审重点 9 条

## 一、评审者独立复核记录（非转抄执行者报告）

- **越界反查**：`git diff --name-only e1b6266^..e1b6266` 复跑 = 23 文件，逐条核对全部落 `ingest/**`，非 ingest 文件数 = 0；无既有测试文件被改动。越界 = 0（与协调者比对结论一致）。
- **验收 1 复跑**（评审者本机）：`go vet ./...` EXIT=0；`go test ./... -run 'Contract' -count=1 -v` 计数 = **`--- PASS` 135 行 / `--- FAIL` 0 行**——与执行者报告 §2.1 完全一致；Race 命名测试 `-count=2` 复跑 pipeline/mqttsource 全 ok。
- **backend 端点参照核对**（IotPublishInternalController.java / InternalSnapshotVO.java / SnapshotStorage.java / CredentialPullVO.java 实读）：
  - 端点 query 形态 `scopeType/scopeId/version(可选)/tenant(可选)` 与 `snapshot/client.go` 请求构造逐字一致；内部令牌头 `X-Internal-Token` 一致。
  - `InternalSnapshotVO.snapshot` 类型为 Jackson 3（tools.jackson）`JsonNode` —— 出站体确为**再序列化形态**（非存储原文），证实执行者 §4.2 发现：下游从响应体不可重算 `snapshotHash = SHA-256(原始存储字节)`（SnapshotStorage.sha256Hex + 取回侧 readAndVerify 哈希核对 500 实读确认）。
  - `CredentialPullVO` 实读无 `gatewayStatus` 字段（TASK-0079 未交付）——决策 5「状态未知」路径的现网前提成立。
- **契约核对**：统一消息模型 §3.2 四行主题矩阵（data/status/config + LWT 与 status 同主题、QoS 1、graceful 标记键名未冻结）与 mqttsource 订阅/路由逐字对齐；快照 Schema §3.3「响应含 version、snapshotHash 与快照体 / 通知只作触发信号 / 失败继续用旧版」与 client.go+reconciler.go 对齐；超级表契约库名 `iot_<tenant_code>` 与 `_5f` 转义为 0073 既有 BUG-0002 纪律（tdengine.go 实读，本 diff 未触碰）。
- **超表 INSERT 形态**（`iot_group_5fa.F01_5fWS01_5fEQ001_5fM01`）：`_5f` 为既有 escapeIdentifier 单射转义（`_`→_5f），非缺陷。

## 二、发现清单

**Critical：0 条。Important：0 条。Minor：12 条。**

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | `ingest/internal/pipeline/dedup_ttl.go:96-107`（diff 行号约 2461-2486） | 「已过期视为首见」分支做 `t.live++` 而键仍在 `seen`（map 尺寸不变），与同处注释「live 计数不变」自相矛盾——经分析该分支在单调时钟下不可达（惰性出队按 FIFO 头部弹出全部过期节点后，seen 中不可能残留过期项；time.Time.Sub 走单调钟读数），属潜伏死代码；若未来时钟源被换成纯墙钟实现，live 将虚增 → 过早 FIFO 淘汰窗口内活键 → 幂等窗口被缩短可致重复落库。建议改注释或在该分支补偿 live 计数 |
| 2 | Minor | `ingest/internal/mqttsource/mqttsource.go:197-206`（Run 订阅失败分支） | Connect 成功但 subscribeAll 持续失败时，重试间隔恒为 BackoffInitial（默认 500ms）不指数递增、无封顶也无计数器（仅日志）——broker ACL 配错类故障会形成无限 500ms 定频 Connect+Subscribe+Close 循环；brief 要求的指数退避只覆盖了「连接不可达」分支。HTTP 路径不受扰，故障面窄，故 Minor |
| 3 | Minor | `ingest/cmd/ingestd/main.go:279-287` + `pipeline.go:258` | 优雅停机只排空 HTTP 在途请求（srv.Shutdown），pcancel 后 `Pipeline.Run` 立即退出、攒批缓冲中未写出行随退出丢弃——pipeline.go 注释自述「真实部署由 0027 收口」而本批（即 0027）未收口；writer=假执行器阶段零实损，0029 接真写后成为真实丢失窗口（≤BatchSize 行）。报告「SIGTERM 排空」表述对 HTTP 在途成立、对攒批缓冲不成立 |
| 4 | Minor | `ingest/cmd/ingestd/main.go:124-157`（loadConfig） | 畸形 `INGEST_SNAPSHOT_SCOPE`（parseScope 报错）、`INGEST_GATEWAY_MAP` 残缺条目（段数≠3/通道号非整数）、非法 `INGEST_SNAPSHOT_INTERVAL/INGEST_DEDUP_TTL` 均被静默忽略、无告警日志——配置打错字时只会得到泛化日志（如「对账未装配」），排障成本高 |
| 5 | Minor | `ingest/go.mod:5-9` | `github.com/eclipse/paho.mqtt.golang` 被 `mqttsource/paho.go` 直接 import 却标 `// indirect`——go.mod 未跑 `go mod tidy`（tidy 会重分类），版本钉 v1.5.1 本身有效，功能无损，属卫生问题 |
| 6 | Minor | `ingest/README.md:15`、`ingest/cmd/ingestd/main.go:54` | 两处组件描述用「真客户端（哈希校验）」缩略语，未限定为「哈希锚点形态校验+原样承载」口径；client.go/reconciler.go 包注释与契约测试是诚实且精确的（见三.4），报告 §4.2/§5.2 亦已澄清——仅文档措辞与实现语义有轻微落差，建议同步措辞 |
| 7 | Minor | `ingest/internal/gwtoken/puller_gatewaystatus_contract_test.go:88`（TestContractCredentialPullGatewayStatusDoesNotBreakFailClosed） | 假后端 handler 内残留无效果语句 `_, _ = json.Marshal(any(map[string]any{}))`（结果被丢弃）——死代码噪声，删之 |
| 8 | Minor | `ingest/internal/mqttsource/mqttsource.go:224-240`（route） | status/config 出口（StatusSink/ConfigSink）在 broker 回调内同步调用——默认实现仅计数+日志无影响，但后续若接慢实现（如 online_status 落库）会经 paho 顺序分发对 data 路由形成队头阻塞；建议在 sink 接口注释固化「必须非阻塞或自带缓冲」约束 |
| 9 | Minor | `ingest/internal/gwtoken/gwtoken.go:189-192`（Validate） | gatewayStatus 落在 {0,1} 之外的值（如未来后端扩枚举 2=已删除）一律按启用对待（「仅 0 是禁用」已注释）——符合决策 5 非 fail-closed 姿态，但 0079 交付时建议随字段定版把「未知枚举值」裁决为 deny，避免静默放行面 |
| 10 | Minor | `ingest/internal/mqttsource/mqttsource.go:174-188`（route KindData）+ 报告 §4.3 | MQTT data 路租户取主题槽位、无任何与 GatewayMap/令牌侧解析的交叉校验，完全信任 EMQX ACL（包注释与报告已如实登记此信任边界）——静态映射 mock 阶段网关向他人租户主题发布即按他人租户进流水线（单 scope 缓存下 pointCode 跨租户重名会错配配置）。属契约自身架构（EMQX 只做接入认证路由），但 TD-0057 真联调必须实证 ACL 拦截有效；可选防御：路由侧加「槽位租户 vs 静态解析租户」一致性守卫 |
| 11 | Minor | `ingest/internal/gwtoken/gwtoken.go:201-231`（credential TTL 缓存） | 禁用网关 deny 读的是 TTL 缓存（默认 30s）里的凭据——网关被禁用后最长 30s 内 auth/mqtt 仍可能 allow（缓存语义继承自 0076，凭据吊销同窗口）——建议在 0079/0068 联调检查单中加一条「禁用后 ≤TTL 内收敛 deny」的预期口径 |
| 12 | Minor | `ingest/internal/mqttsource/paho.go:45-73`（Connect/Close） | Connect 超时路径（WaitTimeout 7s 到期）丢弃 paho 客户端但 paho 内部拨号可能仍在途，孤儿客户端若随后以同一 ClientID（`ingestd-<pid>` 固定）连上会与监督循环新连接互踢产生会话抖动；正常情形 SetConnectTimeout(5s) < 7s 兜住，且 PahoPort 全量真 broker 验证已登记 TD-0057——挂 TD-0057 验证清单即可 |

## 三、按 brief 评审重点逐条结论

1. **MQTT 消费者**——符合。订阅矩阵 `iot/+/+/data|status|config` 与 §3.2 三类主题形态逐字（`+` 通配槽位已在报告 §4.3 登记）；SplitTopic 四段守卫（层数/kind/空槽位）+ 畸形计数；LWT 与 status 同主题（无需独立订阅，心跳/LWT/graceful 三形态路由测试锁定）；QoS 1 落在 PahoPort.Subscribe 逐字；Port 薄接口隔离 paho；`pipeline.Consumer` 接口零改动（pipeline.go 不在 diff 中），mqttsource.Consumer 以 `Messages() <-chan Inbound` 零改动接入。
2. **两路上报一致性**——真实锁住，非「各自能跑」。`TestContractMQTTAndHTTPReportIsomorphicSameResult` 用真接线双镜像流水线（假 broker→真 mqttsource.Consumer→Run vs 真 ReportHandler→PipelineSink→Submit），逐字段比对：全部流水线计数、3 条 INSERT SQL 逐字节、点位缓存 v/q/ts、死信计数；`TestContractMQTTAndHTTPSamePipelineCrossPathIdempotent` 在同一条流水线上 broker 投递后 HTTP 重报同 batchId → duplicate=true、落库恰 3 条单元组 INSERT（跨通道幂等）。两路均收敛于 processOne 五步冻结顺序（Inbound{Tenant, Payload} 同构）。
3. **幂等真实现**——达标。ttlTable 的 MarkIfSeen 首见判定+标记单锁一次完成（BUG-0002 范式保持）；(pointCode,ts) 单条（键粒度与假实现同构）+ batchId 整批双入口；内存界成立：live ≤ cap（FIFO 淘汰）+ TTL 惰性回收分摊 O(1)，order 队列有界（窗口内重报不入队、达 cap 后每新标记弹出恰一节点，队列长 ≈ cap）；并发红线扩到 TTL 实现（30 轮×12 goroutine 恰一处理）评审者复跑通过；多实例=Redis 化演进位在包注释与 main.go javadoc 双登记。唯一瑕疵为发现 1（不可达分支的 live++/注释矛盾）。
4. **快照真客户端与哈希口径（关键发现评估）**——**口径正确、未过度声明、未虚构校验能力**。评审者独立证据链：InternalSnapshotVO.snapshot=Jackson3 JsonNode（再序列化实锤）+ SnapshotStorage 哈希=存储字节 SHA-256 → 重算不可行成立，执行者 live 首拉红（expected≠actual）与缺陷机理吻合。实现侧：锚点形态校验（64 位小写 hex，非法拒收）+ 原样承载 + 信封/体 version 同值 + Schema 校验 + Reconciler 同版本哈希漂移探测（fail-soft 保持旧版）；`TestContractInternalSnapshotClientHashIsAnchorNotRecomputed` 显式锁「hash≠sha256(body) 亦接受」——这是对能力边界的诚实展示而非虚构。仍敞开的完整性威胁（与 §5.2 登记一致）：出站链路在途篡改快照体（哈希不变）Go 侧不可探测（Schema 校验只挡结构性非法、漂移探测只挡哈希变化）——即 backend→ingest 网络段无密码学完整性核验，靠内网静态令牌+IP 白名单+后端取回侧核对兜底，TASK-0080（原始字节透传）闭合。结论：在此契约现状（§3.3 未定义哈希字节域）下对账锚点口径是唯一可实现语义，防御充分性如实登记、无过度声明；唯 README/main.go 措辞缩略（发现 6）。
5. **gatewayStatus**——语义明确、红线真实。nil=缺失/显式 null=状态未知（Known/Disabled 双标志），仅 `Known && Disabled` deny 且置于令牌校验之后、主题范围签发之前（deny 不带 topics、原因进服务端日志）；状态未知不 fail-closed 不放行扩大（保持 0076 allow），fake+真实 HTTP 链路双口径测试锁定；禁用网关 deny 红线用 fake puller（allow 即测试失败）真实有效；pull 403 fail-closed 纪律未被扩展破坏（有专测）。残留口径见发现 9/11。
6. **ingestd 接线**——符合。决策 1 六键齐备 + 补充键位 javadoc 逐键登记（§4.5）；默认 `:8090`（装配冒烟断言，8098/8099 未被占用）；优雅停机顺序（HTTP Shutdown 排空 → pcancel → 消费源退出释放 broker 连接 → Port.Close 断言）由 `TestIngestdAssemblySmoke` 实证——缓冲排空缺口见发现 3；纯 HTTP 形态（broker 空）合法存在（idleConsumer+StartFlusher）。
7. **fail-soft**——达标。指数退避 500ms→30s 封顶：契约测试行为断言（50→100→200ms 递增）+ live 日志 7 次退避实测；HTTP 路径结构性独立（consumer/对账各自 goroutine，report 链路无 MQTT 依赖）；无 goroutine 泄漏（lost 信号缓冲 1、sleepCtx 定时器 Stop、route 阻塞发送带 ctx 逃逸分支、Connect 失败不启 paho 自动重连后台任务）。缺口仅发现 2（订阅失败分支定频）。
8. **零改动替换原则**——成立。23 文件全落 ingest/**；既有测试文件零改动（git diff 文件清单核对）；被改的三个 0076 生产文件（gwtoken.go/puller.go/authmqtt.go）恰为决策 5 授权的 gatewayStatus 承载面；pipeline/tdengine/cache/message/quality 接口文件未在 diff 中；**评审者复跑契约 135 PASS / 0 FAIL、vet 双零退出**——「既有 94 项零改动全绿」的直接证据成立。
9. **并发模型**——符合 README 冻结口径。单消费循环（Run 单 goroutine）+ 有界队列（QueueDepth 1024，route 满则阻塞背压、零丢弃有专测）+ 单 flusher；攒批缓冲有界丢最旧（0073 既有）；Race 探针测试为行为原子性断言（30×12 恰一），`-race` 缺 gcc 未跑已在报告 §5.7 如实登记（评审者同机复现同限制）。

## 四、结论

**通过**——无 Critical / Important 发现；12 条 Minor 均不阻断验收（发现 1 为不可达潜伏分支、发现 2/3/12 属边界故障模式或 0029/TD-0057 前置收口面、其余为卫生与登记口径问题）。执行者报告的九项重点声明经评审者独立复核（含契约测试复跑、backend 端点与三份契约实读）全部成立；两处关键偏离（§4.2 哈希对账锚点口径、§4.3 `+` 通配订阅）判定为正确且如实登记，未虚构校验能力。建议：Minor 1/5/7 可随 TASK-0080 或下一修订轮顺带收口；发现 2/3/8/10/11 的裁决面归 TASK-0029/TD-0057/0079 对应任务，不构成本任务返工项。
