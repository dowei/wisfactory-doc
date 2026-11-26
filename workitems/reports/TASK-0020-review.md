# 评审报告 · TASK-0020 diff 评审（fresh reviewer）

- 评审对象：提交 `36bcd1f`（17 文件，diff 包 TASK-0020-diff.txt，1534 行完整读入）
- 评审依据：实施 brief 七条决策 + L1-execute §13 两样（brief 契约一致性 / 明显质量缺陷）+ ADR-0009 / ADR-0010
- 结论：**Critical 0 / Important 0 / Minor 4**。brief 契约逐条核对全部落地，无越界、无违约；4 条 Minor 均不进修订轮（记 TASK 任务结果 deferred）。

## 一、brief 契约一致性核对（逐条，全部通过）

1. **职责边界**：diff 内零 Redis 写入、零超时判定逻辑（无阈值比较/无扫描循环/无状态裁决）——`GatewayOfflineProperties` 只落键与缺省，`GatewayOfflineEventPublisher` 只发事件，`applyOfflineEvent` 只写管理面展示态（MySQL）。ADR-0010 TD-0049 口径守住。
2. **事件契约**：`GatewayOfflineEvent` record 四字段（gatewayId/tenantId/offlineReason/occurredAt）+ 三工厂；`GatewayOfflineReason` 三值 TIMEOUT/LWT_ABNORMAL/GRACEFUL 与 brief 决策 2 逐字一致；契约登记在网关域 javadoc（事件类头 + 控制器 + 服务接口）。
3. **补列与「与上游差异」**：`20261122_iot_gateway_offline.sql` 头注完整登记（值域/写入方/清除方/接口对应/幂等口径），实体 javadoc 与 @Schema 双向指向脚本；列型 varchar(16) NULL 带注释与裁决逐字一致；information_schema 判存幂等（报告实测两次 DONE）。
4. **0017 回写白名单扩清除动作的最小性**：五字段白名单未动、`GatewayHeartbeatItem` 未加字段——仅 `onlineStatus==1` 时追加一条 `wrapper.set(offline_reason, null)`（MP 实体 null 不进 SET，wrapper 强制是唯一正确路径）；`onlineStatus=0` 不触碰 reason（有专项测试断言 sqlSet 不含 offline_reason）。最小性成立。
5. **list 筛选**：`selectGatewayList` 增 offlineReason 纯 eq（照 0017 status/onlineStatus 同款范式）；响应字段经实体序列化带出（报告 prod 实测 offlineReason present）；controller @Operation 文案同步。
6. **fail-closed**：缺字段监听器 WARN 忽略 + 服务层第二道（BusinessException）；未知 gatewayId WARN 忽略返回 false；reason 未知名桥接层 400 不静默映射（`fromName` 大小写敏感精确匹配，有测试）。
7. **禁用与离线正交**：`applyOfflineEvent` 不查/不改 status，更新集仅两字段；禁用网关照落有专项测试。
8. **范围**：17 文件全落声明内（frontend/ingest/Redis/dataperm/application.yml 零触碰）；0041-0045 守卫（GatewayReferenceGuard 等）零改动，回归 1755 全绿佐证。

## 二、质量缺陷核对（重点项全部通过）

- **监听器异常不炸主流程**：监听器与发布器双层 try/catch 收敛 WARN；日志只记 e.getMessage() 与 0006 DataPermCacheEvict 范式逐字一致（已对照源码核验）。
- **事件消费幂等性**：消费为状态覆盖写（online_status=0 + reason，last-write-wins），重复事件天然幂等，无累积副作用。
- **SQL 参数化**：全部 MyBatis-Plus wrapper 参数绑定 + 枚举名落库，无字符串拼接；SQL 脚本 PREPARE 静态串、sql-exec.sh 行尾分号纪律在位。
- **`onlineStatus == 1` 拆箱安全**：控制器条目解析层强制「onlineStatus 必填数值 0/1」（越界值/缺失均 400），服务层不可能收到 null——已对源码核验。
- **CRUD 不开放写入**：`IotGatewayCreateForm` / `IotGatewayUpdateForm` 无 offlineReason 字段（已 grep 核验）；网关查询（含已删除行排除）与 0017 回写同款 list+eq 范式，语义一致。
- **认证先于载荷**：`internalAuthChecker.check` 在解析前调用，负路径 401 有测试与冒烟证据。
- **publishOfflineEvent 无 @Transactional**：刻意为之且 javadoc 留痕（避免桥接事务吞并消费事务的 UnexpectedRollbackException 纠缠），推理正确；发布器不做 afterCommit 的偏离亦有理由留痕（消费写独立展示态、无 0006 的读后一致性危害），成立。

## 三、发现清单（4 Minor，0 Critical / 0 Important）

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | `backend/ot-admin/.../gateway/controller/IotGatewayInternalController.java`（parseOfflineEventForm 复用 parseHeartbeatTime，L150；文案 L255/L259） | 桥接端点 occurredAt 格式错误时 400 文案误指「lastHeartbeatTime 须为 yyyy-MM-dd HH:mm:ss 或毫秒时间戳」——复用心跳时间解析器未换文案，误导接入侧排障；fail-closed 本身成立，冒烟未覆盖该负路径。 |
| 2 | Minor | 同上文件（parseOfflineEventForm 越界键分支，L126） | 越界键文案复用「回写字段越界（不支持的键: …）」——「回写」是心跳回写术语，在离线事件端点语境下措辞错位（冒烟 S3e 输出即此文案）；仅文案问题，白名单拦截本身正确。 |
| 3 | Minor | `backend/ot-admin/.../gateway/service/impl/IotGatewayServiceImpl.java`（applyOfflineEvent 与 writeHeartbeats 清因分支） | 心跳回写清因与离线事件消费对同一行并发提交为 last-writer-wins，无 occurredAt 与 last_heartbeat_time 的顺序防护——并发离线事件可把刚回在线的网关再次置离线；当前发布方为占位/桥接（同步无队列）窗口极小，展示态可重建且下次心跳自愈（javadoc 已登记兜底），判定器真实接线时需复核事件顺序语义（登记遗留性质）。 |
| 4 | Minor | 同上文件（selectGatewayList offlineReason 过滤分支） | list 的 offlineReason 过滤值域不校验——`offlineReason=FOO`/`timeout`（大小写不符）静默返回空列表而非 400，与桥接端点的 fail-closed 400 风格不一致；符合 brief「照 0017 纯 eq 范式」口径故不算违约，冒烟 S6b 亦按此口径验收，登记备查防误用。 |

## 四、评审口径说明

- 本评审针对 diff 本身（提交 36bcd1f）；报告实测数字（1755=1723+32、幂等 DONE、冒烟/prod 序列）与 diff 内容交叉印证无矛盾。
- Minor 按 L1-execute §13 不进修订轮，记 TASK `任务结果` deferred（收尾进 LOG）。
- 发现 1/2 同根因（桥接端点复用心跳回写辅助未换文案），若后续统一修缮可合并处理；发现 3 建议随判定器接线任务一并裁决。
