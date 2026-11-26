# 评审报告：TASK-0026 diff 评审（代码评审）

> 评审者：fresh 子代理（只依据 review brief 必读清单与工作区源码判断）。
> 评审对象：实现提交 **2e4a1b9**（BASE=d08ec02，单提交，21 文件 +2054/-41——`git diff --stat` 复核一致）。
> 评审口径：`L1-execute.md` §13「diff 评审」（与 TASK 验收标准 / brief 接口的一致性 + 明显质量缺陷）。
> 工作区状态：HEAD=2e4a1b9，工作区干净（仅未跟踪 `.agents/`），diff 包与 HEAD 实文一致。

---

## 一、越界反查复核

成立。`git diff --name-only d08ec02..HEAD` 复跑 = **21 文件**，全部落 brief「改动范围」声明内：publish 主代码 14（notify 4 + dto 5 + service 3 + controller 1 + config 1）+ test 4 + pom.xml 1 + application.yml 1；无 resources/sql/** 文件（与「无新 SQL」声明一致）。

## 二、评审重点逐条核对结论（brief 9 条全部核对）

1. **afterCommit 裁决落地正确性——成立。** `MqttNotifySender.java:93-99`：`isSynchronizationActive()` 守卫 + `registerSynchronization(afterCommit)`，回调内仅网络发送（载荷在调用点事务内组装捕获，回调零 DB 读取）；`sendQuietly` 全捕获不外抛，提交链路不受扰（「不吞」按 fail-soft 设计意图吞并 WARN 留痕——符合 REQ-0033 验收 2 契约，非缺陷）。事务必然激活：调用点 `recordDispatch` 自带 `@Transactional`（`IotPublishLogServiceImpl.java:92`）；无事务链路 resend 确无 `@Transactional`（:287）→ 立即发送，行已即时提交、无伪通知。异常路径不重复：每次 notify 至多注册一个回调。javadoc 双处登记存在（`NotifySender.java`「0025 评审 Minor ④ 取舍登记…裁决=a」段 + `MqttNotifySender` 类注释），与 0025 评审 Minor ④ 原文（「建议 0026 brief 显式承接…a) 发送移至 afterCommit」）承接成立，冒烟日志 `deferred to afterCommit` ×2 行为证。
2. **MqttNotifySender——成立。** topic 映射逐字：网关 `iot/{tenantCode}/{gatewayCode}/config`（:117）、接入/计算/未知类型防御 → 共享 `iot/config.changed`（:25,:123）。取数：gatewayCode=下发行 targetId（落行时取自 `iot_gateway.gateway_code`——`IotPublishLogServiceImpl.java:116-120`，与 brief「gatewayCode 读 iot_gateway」同源等值，随行下发保证下游按 targetId 对账一致）；tenantCode=tenant_id 反查 `sys_tenant.tenant_code`（resolveTenantCode，平台级表无租户注入）。载荷 §3.3 五键逐字（tenant/scopeType/scopeId/version/hash）+ `@JsonPropertyOrder` 钉序；QoS 1 / retain=false（:154）。fail-soft 全路径 WARN 不抛（buildPayload/serialize/topicOf/sendQuietly 四处）。懒建立+自动重连+句柄复用在 `PahoMqttPublisher`（成员持有、`synchronized obtainClient`、`automaticReconnect(true)`）；真 broker 下的实际表现属 TD-0056（见「四、无法判定」）。
3. **条件装配边界——成立。** `MqttNotifySender`/`PahoMqttPublisher` 均 `havingValue="true"`；`NoOpNotifySender` `havingValue="false"+matchIfMissing=true`——false/缺省路径 NoOp 恒空操作，调用点数量与位置不变（recordDispatch 逐行 + resend 一次），NoOp 行为与 0025 完全等价（仅日志参数改 `version.getId()`，同值）。`MqttNotifySenderTest.enabledSwitchIsWiringContract` 锁定注解契约。
4. **snapshot 端点——成立。** 缺省=各租户最大版本、指定版精确命中、不存在/无版本 400（`resolveInternalVersionRow`）；哈希核对复用 0024 路径（提取共享私有方法 `readAndVerifySnapshot`，:540——管理面 `retrieveSnapshot` 为纯提取重构，行为不变，0024 测试未动 + 冒烟 list/snapshot 复证）；出站凭据终检 :395（命中 → 500 + `[SNAPSHOT] outbound credential guard tripped` ERROR 告警，fail-closed，报告偏离 8 已登记裁决）；跨租户可选 tenant 参数（tenant_code→id 精确过滤、不存在 400）、缺省歧义 400 fail-closed（snapshot 与 changes 双路径同口径，网关 id 全局唯一/计算 scopeId=0 恒可撞统一收口）。
5. **changes 端点——成立。** 五段 diff 对齐键 point→pointCode、channel→channelCode、dataItem→fullKey、formula→行 id、metric→metricCode，code 缺失回退行 id，(type,key) 排序确定性输出；added/removed/changed 按对象对齐（JsonNode 结构化全等，键序无关数组有序——canonical 口径 javadoc 登记）；needFullSync 触发面=跨度严格大于 changes-max-span / 基线版本行不存在 / 中间快照缺失或解析失败 / 版本序列空洞（防御同口径），四路径都只给 latest 摘要（基线可得给基线→latest 累计 diff、不可得留空三段）；since≥latest 空数组。服务层 17 测试 + 冒烟 diff/空数组/基线缺失/跨度超限全实证。
6. **Jackson 3 修复的范围控制——成立。** 改动严格限定三处：`InternalSnapshotVO.snapshot` 字段类型（tools JsonNode）+ `OUT_MAPPER` + `parseSnapshotBody`；diff/组装/终检/通知载荷路径保持 com.fasterxml（与 0024 同源，纯服务端内部串化）；无全局序列化行为变化（application.yml 仅 publish 键族，无 jackson 键），管理面 `SnapshotVO.snapshotBody` String 原文契约不动，其他端点 JSON 契约不受影响。框架级发现（出站树形 DTO 必须用运行时 HTTP 栈同源节点）建议登记正式 TD——见发现 ⑦。
7. **pom/yml——成立。** pom 仅新增 paho 1.2.5 显式钉死（含注释）；yml 仅 `changes-max-span`×1 + `notify.*`×7（enabled/broker-url/username/password/client-id/keep-alive-seconds/timeout-seconds），env 占位符齐备，password 经环境变量注入（ADR-0006：凭据不入库不入快照不入载荷）。
8. **回滚链——成立。** 未新增回滚代码路径：rollback 本体不在 diff 中；`recordDispatch`/`resend` 仅 NotifySender 签名形变（versionId→版本行引用），回滚语义未动。冒烟步骤 6 实证：回滚=新版本 4 复用 v1 快照（hash/uri/体三方一致）+通知按行触发+两行待确认落行——「回滚=旧快照新版本重新下发」自动生效。
9. **TD-0056 登记——成立。** 报告 §五1 文案完整（四步验证清单：专属主题五键载荷/共享主题/kill broker fail-soft+自动重连/retain=false 复核，含归属建议）；「通知载荷 version=业务版本号非 versionId」口径在报告 §五2 + `NotifyPayload.version` javadoc（「注意非版本行 id」）双处留痕清晰。

## 三、发现清单

**无 Critical，无 Important。** 以下 Minor 9 条，按 §13 记 TASK `任务结果` deferred（收口进 LOG），不进修订轮：

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | `PahoMqttPublisher.java:43-45` + `MqttNotifySender.java:93-99` | afterCommit 回调在提交线程同步阻塞发送：broker 黑洞（端口 DROP 而非 REFUSE）时 connectionTimeout 默认 30s、网关 scope 两行串行 ×2，发布/回滚 HTTP 响应最坏拖长约 60s——fail-soft 契约（事务不回滚、结果不受扰）字面成立、常见故障（REFUSED）毫秒级失败，但运营健壮性有折扣；建议登记 TD（afterCommit 改异步执行器投递，或下调默认 timeout-seconds） |
| 2 | Minor | `PahoMqttPublisher.java:80-82` | connect 失败路径 `client = null` 丢弃已构造的 `MqttClient` 未调 `close()`——paho 失败路径通常自停线程，但 close 是文档化资源释放动作，反复连接失败场景留线程/持久化残留风险面（防御性 close 缺失） |
| 3 | Minor | `PublishProperties.java:99` vs `PahoMqttPublisher.java:76` | brokerUrl 为空分支的 javadoc 仍写「抛 INVALID_CLIENT」，实际抛 `REASON_CODE_CLIENT_EXCEPTION`（执行者报告修复 1 已解释常量在 paho 1.2.5 不存在并改名，但该 javadoc 未同步）——文档漂移 |
| 4 | Minor | `IotPublishServiceImpl.java:720-722` | 提取 `readAndVerifySnapshot` 时遗留一行被截断的重复 section 注释（「…——发布/回滚/」处截断）+ 空行 + 原完整注释，纯外观瑕疵 |
| 5 | Minor | `PahoMqttPublisher.java:60-70` vs `:92-103` | `MqttConnectOptions` 组装逻辑完全重复两份（内联分支 vs `connectOptions()`，各 12 行）——改其一漏其二的静默漂移面（DRY 债，与 0025 评审 Minor ② 同型） |
| 6 | Minor | `IotPublishServiceImpl.java:698-707`（readQuietly/parseQuietly） | changes 逐版 diff 读取中间快照不做哈希核对：文件被篡改但仍是合法 JSON 时摘要静默失真、不翻转 needFullSync——REQ-0033 拉取比对哈希契约下摘要仅建议性、无状态腐蚀，但与 0024「哈希核对=存储损坏探测」纪律不一致；低成本补核对可让损坏同样落 needFullSync 兜底 |
| 7 | Minor | 执行者报告 §四7（代码落点 `InternalSnapshotVO.java` + `IotPublishServiceImpl.java:126-134`） | Jackson 3 框架级发现（Spring Boot 4 HTTP 栈=tools.jackson；出站树形 DTO 的 JSON 树字段用 Jackson 2 节点会被按 POJO 序列化成属性散列——冒烟实测）仅留痕于报告，未登记正式 TD——这是后续新增出站树形 DTO 任务会反复踩的约束，建议补登记 TD（评估结论：值得登记，非 ADR 级——不涉架构裁决，属框架事实登记） |
| 8 | Minor | `IotPublishServiceImpl.java:395` vs `:361-384` | 出站凭据终检防线不对称：内部 snapshot 端点有 `checkOutboundCredential`，管理面 `/iot/publish/snapshot` 取回无出站终检——brief D2 仅要求内部端点、0024 既有行为未破坏，但存储篡改场景下管理面同样外泄凭据体，登记为一致性观察项 |
| 9 | Minor | `IotPublishServiceImpl.java:421-427`（changesSince 查询）/ `resolveInternalVersionRow` | 内部 snapshot/changes 查询不过滤 `publish_status=1`——当前流程版本行只会是 1（0024 类注释「2 为上游 DDL 预留不产生」）无实际影响，未来若引入草稿/未发布态会外泄未发布版本；防御性过滤观察项 |

## 四、无法判定项

1. **真 broker 下的连接生命周期**（懒建立/自动重连/句柄复用/paho 1.2.5 真实表现）——本地无 EMQX，按 brief D5 mock 口径交付；已登记 TD-0056，联调环境就绪后验证，本评审无法判定。
2. **全量测试 2244 通过**未由本评审独立重跑——以执行者报告验收 1 原始输出为准（diff 评审职责为一致性+质量核对，非复跑验收命令）。

## 五、结论

**通过。** brief 评审重点 9 条全部核对成立；无 Critical / Important，Minor 9 条均为一致性债/防御性观察项/文档漂移，不阻断收尾，按 L1-execute §13 记 TASK `任务结果` deferred（收口进 LOG）。0025 评审 Minor ④ 交接约束已按决策 a（afterCommit）正确承接并双处 javadoc 登记；Jackson 3 出站修复限定在出站快照组装处、无全局序列化行为外溢（建议补登记 TD，见发现 ⑦）。
