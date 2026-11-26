# 实施 brief：TASK-0020（心跳超时判定与 LWT/graceful 状态联动——管理面消费与展示）

## 批次

- TASK-0020（本批共 1 个，标准任务 2 人天；backend 串行道——0017 回写接口的下游）
- BASE：`2ff499b`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/**`（离线事件消费/LWT 区分/列表筛选扩展）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_gateway_offline.sql`（如需状态区分列——见裁决 3）
  - 不碰 frontend/、ingest/、Redis 状态写入（ADR-0010 口径归计算引擎）、dataperm

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0020.md`
- 关联对象：同目录 `features\FEAT-0006.md`、`requirements\REQ-0009.md`（验收 2/4/5：心跳正常在线/超阈值离线可筛/禁用拒接入/LWT 与 graceful 可区分）、`requirements\REQ-0011.md`、`testcases\TC-0006.md`
- 决策：`decisions\ADR-0009.md`（TD-0029 LWT 事件链——事件由计算引擎侧判定落状态，本任务只消费管理面展示）、`decisions\ADR-0010.md`（TD-0017/0038 相关口径）
- 既有实现：`iot/gateway/**`（0017：回写接口五字段组/回写白名单/list 含 online_status 列）——本任务是其下游
- 口径权威：详细设计 V1.1 §3.1 内部回写段与 REQ-0009 验收 5（LWT 异常断开与正常下线 graceful 可区分）

## 接口与决策（逐条结论，不再讨论）

1. **职责边界（任务正文硬要求）**：本任务**不执行超时判定、不直写状态 Redis**（判定与状态落 Redis 归计算引擎，ADR-0010 口径）——管理面消费的是**事件**：离线事件（心跳停止超阈值）与 LWT/graceful 事件由接入/计算引擎侧发出，本任务实现「事件的消费端」：Spring 应用事件监听（与 0006 DataPermCacheEvictEvent 同范式——事件类定义在本任务，发布方为未来接入/计算引擎接线，**占位发布器 + 单测直接注入事件验证消费逻辑**）。
2. **事件契约（本任务定义，登记进网关域 javadoc）**：`GatewayOfflineEvent`（gatewayId/tenantId/offlineReason 枚举 TIMEOUT/LWT_ABNORMAL/GRACEFUL/occurredAt）；消费端按 reason 更新管理面展示态。
3. **状态区分（REQ-0009 验收 5）**：iot_gateway 现有 online_status 仅 0/1——**补列裁决**：增量脚本加 `offline_reason varchar(16) DEFAULT NULL COMMENT '离线原因 TIMEOUT/LWT_ABNORMAL/GRACEFUL（NULL=在线）'`（「与上游差异」头注登记——上游 DDL 无此列，验收 5 要求可区分）；消费端按事件写 online_status=0 + offline_reason；心跳回写（0017 接口）成功时清 offline_reason=NULL 置在线——**0017 回写白名单需扩 offline_reason 清除动作**（gateway 域最小改动，授权）。
4. **超时阈值配置**：`ot.gateway.offline-timeout-multiplier`（默认 3——3 个心跳周期，REQ-0009 口径「如 3 个心跳周期」）+ `ot.gateway.offline-check-interval-seconds`——**判定器仍不在本任务**（归计算引擎/接入侧），本期只落配置键与消费端（判定器接线登记遗留）。管理面「超阈值转离线且可在列表筛出」的验收=离线事件注入后的筛选正确性。
5. **列表筛选扩展**：`GET /iot/gateway/list` 增加 offlineReason 过滤参数与响应字段（照 0017 既有 list 范式）；「最近心跳」列已存在（0011 last_heartbeat_time）。
6. **测试口径（TDD）**：事件消费三 reason 各自落库（TIMEOUT/LWT_ABNORMAL/GRACEFUL → online_status=0 + 对应 reason）；心跳回写清 reason 置在线；list 筛选 offlineReason；事件缺字段 fail-closed（未知 gatewayId 忽略 WARN）；禁用网关的事件仍落展示态（禁用与离线正交）。
7. 冒烟（dev）：注入三类事件 → list/DB 核对三态区分 → 模拟心跳回写 → 回在线 → 筛选参数生效。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1723 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对 offline_reason 列在位
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0020-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0020）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
