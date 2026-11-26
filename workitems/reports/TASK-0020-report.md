# 验收报告 · TASK-0020（心跳超时判定与 LWT/graceful 状态联动——管理面消费与展示）

> **中断恢复接手**：前一执行者完成实现并已提交（`36bcd1f`），在冒烟阶段被额度中断（无报告、冒烟未完成）。本执行者接手后：①`git show 36bcd1f --stat` 盘点实现面对照 brief 核对完整性——**17 文件全在声明范围、brief 七条决策全部落地，未发现实现缺陷，无需重实现、无需新提交**；②五条验收全部由本会话**完整重跑采集**（未引用中断会话的任何输出，其「11 个 ERROR 均为预期业务异常」的说法未采信、亦未复用）；③TASK 状态未改、workitems 库仅新增本报告文件。

## 0. 总体结论

**5 条验收全部通过**：全量回归 HEAD 1755 = 基线（实测 11e200c）1723 + 32 新增（31 本任务 + 1 前置修订轮），0 失败 0 错误 BUILD SUCCESS；升级脚本二次执行均 DONE 幂等 + offline_reason 列在位（varchar(16) NULL 带注释）；dev 冒烟全序列绿（三类事件注入→三态落库区分→禁用网关照落→筛选精确匹配→心跳回写回在线清因→onlineStatus=0 回写不擦除原因→fail-closed 负路径 6/6）；prod 档干净单轮真实登录 200 + gateway/list 200 + 全程 0 ERROR（另在首轮做了负路径探针与注入/筛选/回写全链路，均绿）；提交 17 文件全落声明范围（brief 字面命令多出的 3 个 frontend 文件系 BASE 快照过期，见 §2.5）。

---

## 1. 改动文件清单（`git show 36bcd1f --numstat`，17 文件，1229+/8-）

| # | 文件 | 类别 |
| --- | --- | --- |
| 1 | backend/ot-admin/.../iot/gateway/event/GatewayOfflineEvent.java | 事件契约（4 字段 record + 三工厂） |
| 2 | backend/ot-admin/.../gateway/event/GatewayOfflineReason.java | 枚举 TIMEOUT/LWT_ABNORMAL/GRACEFUL + fromName fail-closed 解析 |
| 3 | backend/ot-admin/.../gateway/event/GatewayOfflineEventListener.java | @EventListener 消费，缺字段 WARN 忽略，消费异常收敛 |
| 4 | backend/ot-admin/.../gateway/event/GatewayOfflineEventPublisher.java | 占位发布器（立即发布，无 afterCommit——偏离理由留痕于 javadoc） |
| 5 | backend/ot-admin/.../gateway/event/GatewayOfflineProperties.java | 配置键 offline-timeout-multiplier(默认3)/offline-check-interval-seconds(默认30)，@Value 代码默认值交付（0017 IP 白名单同先例，application.yml 不在范围声明内） |
| 6 | backend/ot-admin/.../gateway/controller/IotGatewayInternalController.java | 新增 POST /internal/v1/gateway/offline-event 桥接端点（键白名单 4 键/reason 未知名 400/认证先于载荷） |
| 7 | backend/ot-admin/.../gateway/dto/GatewayOfflineEventForm.java | 桥接 typed 载荷 |
| 8 | backend/ot-admin/.../gateway/entity/IotGateway.java | 增 offlineReason 字段（与上游差异登记指向升级脚本头注） |
| 9 | backend/ot-admin/.../gateway/service/IotGatewayService.java | 增 applyOfflineEvent/publishOfflineEvent 声明；writeHeartbeats javadoc 增清除动作说明 |
| 10 | backend/ot-admin/.../gateway/service/impl/IotGatewayServiceImpl.java | 消费落展示态（online_status=0+reason，未知网关 WARN 忽略）/心跳回写 onlineStatus=1 时 wrapper.set(offline_reason,NULL)/list 增 offlineReason 纯 eq 筛选 |
| 11 | backend/ot-admin/.../gateway/controller/IotGatewayController.java | list @Operation 文案更新（offlineReason 筛选/列） |
| 12-16 | backend/ot-admin/src/test/.../iot/gateway/{controller/IotGatewayControllerTest(+1),controller/IotGatewayInternalControllerTest(+7),event/GatewayOfflineEventListenerTest(新6),event/GatewayOfflineEventPublisherTest(新6),service/impl/IotGatewayServiceImplTest(+11)} | 测试 31 用例（三 reason 落库/心跳清因/筛选/缺字段 fail-closed/未知 gatewayId 忽略/禁用正交/桥接负路径） |
| 17 | resources/sql/upgrade/20261122_iot_gateway_offline.sql | offline_reason 列幂等加列（information_schema 判存，与上游差异头注登记，不加菜单行） |

与 brief 范围声明对照：gateway 主代码 11 + iot 测试 5 + upgrade SQL 1 = 17/17 全在声明内；frontend/、ingest/、Redis 状态写入、dataperm、application.yml **零触碰**。

## 2. 验收命令与原始输出（五条，全部本会话实测）

### 2.1 验收 1：全量回归 —— 通过

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

HEAD（46d9391，含 36bcd1f）Results 段：

```
[INFO] Tests run: 1755, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
[INFO] Total time:  02:49 min
```

**基线数核对（关键澄清）**：恢复须知称「基线 1755，若 36bcd1f 含新增测试则总数应 >1755，以实测为准」。实测裁决：以 `git worktree add` 取基线提交 **11e200c**（BASE 2ff499b 树 + TASK-0056 修订轮，不含本任务）独立安装其 ot-core 并跑全量：

```
（11e200c worktree，2026-11-22 12:24）
[INFO] Tests run: 1723, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
[INFO] Total time:  02:49 min
```

**1723 = brief 声明的基线**。差值 1755-1723=32 个新增 @Test 方法，逐文件归账（`git show` 双提交 @Test 计数差）：

- GatewayOfflineEventListenerTest 新文件 +6、GatewayOfflineEventPublisherTest 新文件 +6
- IotGatewayInternalControllerTest +7（10→17）、IotGatewayServiceImplTest +11（27→38）、IotGatewayControllerTest +1（12→13）→ 本任务合计 **+31**
- IotCredentialServiceImplTest +1（24→25）＝前置提交 46d9391（TASK-0019 修订轮）→ +1

即 **协调者笔记中的「基线 1755」实为前一执行者实现后（已含本任务 31 个新用例）的实跑计数**，与本会话 HEAD 实测 1755 完全一致——「应 >1755」的前提基于过期基线；实测口径下 1755 = 1723 + 32，全部新增在跑。新用例确实执行的证据（surefire 报告，2026-11-22 12:16-12:17）：

```
GatewayOfflineEventListenerTest:      Tests run: 6,  Failures: 0, Errors: 0
GatewayOfflineEventPublisherTest:     Tests run: 6,  Failures: 0, Errors: 0
IotGatewayInternalControllerTest:     Tests run: 17, Failures: 0, Errors: 0
IotGatewayServiceImplTest:            Tests run: 38, Failures: 0, Errors: 0
IotGatewayControllerTest:             Tests run: 13, Failures: 0, Errors: 0
```

环境纪律：worktree 跑基线装入了旧 ot-core，跑毕已**重装 HEAD ot-core** 并以 jar 内字节校验 `VERSION_CONFLICT` 在位（防 0019 报告遗留 2 的并行 jar 覆盖类问题）；worktree 已删除。判定：**通过**（BUILD SUCCESS、0 失败 0 错误、新增 31 用例全在跑且全绿）。

### 2.2 验收 2：升级脚本二次幂等 + 列核对 —— 通过

`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_iot_gateway_offline.sql` 连续两次：

```
=== run1 ===
OK(0)
OK(0)
OK(0)
OK(0)
note
skip: offline_reason already exists
OK(0)
DONE
=== run2 ===
OK(0)
OK(0)
OK(0)
OK(0)
note
skip: offline_reason already exists
OK(0)
DONE
```

列核对（`SHOW COLUMNS ... LIKE 'offline_reason'` + information_schema）：

```
Field           Type         Null  Key  Default  Extra
offline_reason  varchar(16)  YES        null

COLUMN_NAME     COLUMN_TYPE  IS_NULLABLE  COLUMN_DEFAULT  COLUMN_COMMENT
offline_reason  varchar(16)  YES          null            离线原因 TIMEOUT/LWT_ABNORMAL/GRACEFUL（NULL=在线）
```

两次执行均 DONE、第二次走判存跳过分支不报错不重复加列；列型/注释与脚本裁决逐字一致。判定：**通过**。

### 2.3 验收 3：dev 冒烟 —— 通过

启动：8098 启动前无 LISTENING；`OT_GATEWAY_INTERNAL_TOKEN=t0020-internal-token-xyz mvn spring-boot:run -pl ot-admin`（日志 backend/target/task0020_dev.log）→ `Started OtAdminApplication in 14.315 seconds`，**启动段 ERROR=0**。冒烟脚本 backend/target/t0020_smoke.py 原始输出：

```
=== S1) admin dev login -> 200 token= 870a62c2...
=== S2) 建两网关（主冒烟 + 禁用态网关） ===
S2a POST gateway GW_T0020_01 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2b POST gateway GW_T0020_DIS -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2c ids: GW_T0020_01 = 14 | GW_T0020_DIS = 15
=== S3) 内部桥接端点负路径（fail-closed） ===
S3a no token -> 401 -> HTTP 200 code 401 | 内部令牌无效  [期望401] OK
S3b wrong token -> 401 -> HTTP 200 code 401 | 内部令牌无效  [期望401] OK
S3c 未知 reason -> 400 -> HTTP 200 code 400 | 离线事件 offlineReason 未知名（仅允许 TIMEOUT/LWT_ABNORMAL/GRACEFUL）: NOT_A_REASON  [期望400] OK
S3d 未知 gatewayCode -> 400 -> HTTP 200 code 400 | 网关编码不存在或已删除: GW_NOT_EXISTS  [期望400] OK
S3e 键越界 extraKey -> 400 -> HTTP 200 code 400 | 回写字段越界（不支持的键: [hacker]）——仅允许 [tenantCode, offlineReason, gatewayCode, occurredAt]  [期望400] OK
S3f 缺 reason -> 400 -> HTTP 200 code 400 | 离线事件缺少 offlineReason（仅允许 TIMEOUT/LWT_ABNORMAL/GRACEFUL）  [期望400] OK
=== S4) 注入三类事件（TIMEOUT/LWT_ABNORMAL/GRACEFUL），主网关轮转三态 ===
S4 inject TIMEOUT -> HTTP 200 code 200 | 操作成功  [期望200] OK
S4 inject LWT_ABNORMAL -> HTTP 200 code 200 | 操作成功  [期望200] OK
S4 inject GRACEFUL -> HTTP 200 code 200 | 操作成功  [期望200] OK
S4 final row after 3 injections: {"id": 14, "gatewayCode": "GW_T0020_01", "onlineStatus": 0, "offlineReason": "GRACEFUL"}
S4 断言: onlineStatus=0 且 offlineReason=GRACEFUL（最后注入者） OK
=== S5) 三态同时呈现：主网关 GRACEFUL，另注入 TIMEOUT 到禁用网关（禁用与离线正交照落） ===
S5a PUT 禁用网关 status=0 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5b 注入 LWT_ABNORMAL 到禁用网关 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5c 禁用网关行: {"id": 15, "gatewayCode": "GW_T0020_DIS", "status": 0, "onlineStatus": 0, "offlineReason": "LWT_ABNORMAL"}
S5c 断言: status=0（禁用保持）且 onlineStatus=0 + offlineReason=LWT_ABNORMAL 照落 OK
=== S6) list 筛选 offlineReason（首轮脚本断言见偏离 3 说明） ===
S6 filter TIMEOUT -> total 0 codes [] OK
S6 filter LWT_ABNORMAL -> total 2 codes ['GW_T0020_B', 'GW_T0020_DIS']（GW_T0020_B 为中断会话 11:57:04 遗留行，非本会话产物——见下）
S6 filter GRACEFUL -> total 3 codes ['GW_T0020_C', 'GW_T0020_D', 'GW_T0020_01']（C/D 同为遗留行）
=== S7) 心跳回写回在线清因（onlineStatus=1 -> offline_reason=NULL） ===
S7a heartbeat onlineStatus=1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S7b 回写后主网关行: {"gatewayCode": "GW_T0020_01", "onlineStatus": 1, "offlineReason": null, "lastHeartbeatTime": "2026-11-22 12:30:00"}
S7b 断言: onlineStatus=1 且 offlineReason=None OK
=== S8) onlineStatus=0 回写不触碰 offline_reason（事件原因不被心跳擦除） ===
S8a 注入 TIMEOUT -> HTTP 200 ... OK
S8b heartbeat onlineStatus=0 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S8c 行: {"onlineStatus": 0, "offlineReason": "TIMEOUT"}
S8c 断言: 仍 offline=TIMEOUT（未被 onlineStatus=0 回写擦除） OK
=== S9) GRACEFUL 收尾 ===
S9 注入 GRACEFUL -> HTTP 200 ... OK
```

S6 首轮脚本严格断言（codes==[期望]）被判 403 风格的「MISMATCH」——**根因是冒烟脚本断言过严而非实现缺陷**：库中另有中断会话 11:57:04 留下的 GW_T0020_A/B/C/D 四行（A 已被其心跳回在线 reason=NULL、B=LWT_ABNORMAL、C/D=GRACEFUL，DB create_time=11:57:04 与本会话 12:28:02 可分账）。修正断言复跑（t0020 S6b）：

```
filter TIMEOUT -> total 0 []
  断言: 全部行 offlineReason== TIMEOUT 且 onlineStatus==0 -> OK
filter LWT_ABNORMAL -> total 2 [{"gatewayCode": "GW_T0020_B", ...}, {"gatewayCode": "GW_T0020_DIS", ...}]
  断言: 全部行 offlineReason== LWT_ABNORMAL 且 onlineStatus==0 -> OK
filter GRACEFUL -> total 3 [{"gatewayCode": "GW_T0020_C", ...}, {"gatewayCode": "GW_T0020_D", ...}, {"gatewayCode": "GW_T0020_01", ...}]
  断言: 全部行 offlineReason== GRACEFUL 且 onlineStatus==0 -> OK
filter NOT_A_REASON -> total 0 code 200 OK
```

即筛选为「按列值精确 eq 匹配」，结果集与库内实际行完全一致（含中断会话遗留行也被正确筛出）。DB 核对（sql-exec.sh）：

```
id  gateway_code  online_status  offline_reason  status  create_time
10  GW_T0020_A    1              null            1       2026-11-22 11:57:04   ←中断会话遗留
11  GW_T0020_B    0              LWT_ABNORMAL    1       2026-11-22 11:57:04   ←遗留
12  GW_T0020_C    0              GRACEFUL        1       2026-11-22 11:57:04   ←遗留
13  GW_T0020_D    0              GRACEFUL        0       2026-11-22 11:57:04   ←遗留（禁用）
14  GW_T0020_01   0              GRACEFUL        1       2026-11-22 12:28:02   ←本会话（S9 后终态）
15  GW_T0020_DIS  0              LWT_ABNORMAL    0       2026-11-22 12:28:03   ←本会话（禁用正交）
```

应用日志：全程 6 条 ERROR 均为 GlobalExceptionHandler 记录的**预期业务拒绝**（S3 的 2×401 内部令牌无效 + 4×400 未知名/未知网关/键越界/缺 reason），无应用缺陷、无 GATEWAY-OFFLINE WARN。用毕即停：taskkill → 8098 复核无 LISTENING。判定：**通过**（brief 测试口径七项序列全绿）。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（ot-admin-1.0.0.jar，42.5s）。启动命令（沿 0017/0019 确认命令 + 三密钥 + 本任务内部令牌 env）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
OT_GATEWAY_INTERNAL_TOKEN=t0020-internal-token-xyz \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

**干净单轮（第 2 轮，日志 task0020_prod2.log，Redis 预置 captcha:code:t0020q="AB21" EX 600 + SM4 admin123 密文）**：

```
Started OtAdminApplication in 16.581 seconds (process running for 17.161)
=== prod2 ERROR count ===
0
LOGIN HTTP 200 code 200 message 操作成功
LOGIN token ad99bfd6...
LIST /iot/gateway/list HTTP 200 code 200 total 10
LIST cols offlineReason present: True
```

**首轮（日志 task0020_prod.log）**除启动外额外实测了本任务端点全链路（此轮含 1 次故意的无令牌负路径探针，故日志有 1 条预期业务 ERROR——为交付字面「全程 0 ERROR」另起干净第 2 轮，0019 同先例）：

```
Started OtAdminApplication in 20.494 seconds
LOGIN HTTP 200 code 200 token fdee71fb...
LIST /iot/gateway/list HTTP 200 code 200 total 10（10 行全部含 offlineReason 字段；遗留行三态如实展示）
INJECT no-token HTTP 200 code 401 [expect 401] OK
INJECT TIMEOUT HTTP 200 code 200 [expect 200] OK
FILTER offlineReason=TIMEOUT HTTP 200 code 200 total 1 codes ['GW_T0020_01'] OK
HEARTBEAT back-online HTTP 200 code 200 data 1 [expect 200/1] OK
AFTER-HEARTBEAT row: {"gatewayCode": "GW_T0020_01", "onlineStatus": 1, "offlineReason": null, "lastHeartbeatTime": "2026-11-22 12:31:00"}
断言 回在线清因: OK
```

用毕即停：两轮均 taskkill → 8098 复核无 LISTENING。判定：**通过**（启动+全程 0 ERROR、真实登录 200、gateway/list 200；prod 档离线事件注入/筛选/回写清因全链路可用）。

### 2.5 验收 5：改动范围核对 —— 通过（brief 字面命令含 BASE 快照过期差集，与 0019 报告 §2.5 同现象）

brief 原命令口径 `git diff --name-only <BASE>..HEAD` 实测（以 36bcd1f 为任务 HEAD、2ff499b 为 BASE）20 文件：

```
backend/ot-admin/.../gateway/**（11 主代码） + src/test/.../gateway/**（5 测试） + resources/sql/upgrade/20261122_iot_gateway_offline.sql
frontend/src/views/iot/dict/components/dict-import-modal.vue      ← 非 36bcd1f 产物
frontend/src/views/iot/instance/components/instance-item-drawer.vue ← 非 36bcd1f 产物
frontend/src/views/iot/instance/components/instance-params-modal.vue ← 非 36bcd1f 产物
```

3 个 frontend 文件系中间提交 **11e200c（TASK-0056 修订轮：弹窗刷新事件名错位清查）** 的改动（`git show 11e200c --name-only` 三文件逐一吻合），非本任务产物；brief 的 BASE 快照在 0056 修订轮入库后过期。**任务自有提交 36bcd1f 的 diff 恰好 17 文件，100% 落在声明范围**（§1 清单）；frontend/、ingest/、dataperm、Redis 状态写入零触碰；workitems 库仅新增本报告；TASK 状态未改；无新 commit。判定：**通过**。

## 3. 未做项与原因

1. **超时判定器本体**（brief 决策 4 明文）：判定归接入/计算引擎侧（ADR-0010 TD-0049），本任务只落配置键（ot.gateway.offline-timeout-multiplier=3 / offline-check-interval-seconds=30，@Value 代码默认值交付）与消费端——判定器接线登记遗留，非本执行者新增缺项。
2. **事件多实例跨节点传播**：事件为进程内 Spring 应用事件（与 0006 DataPermCacheEvictEvent 同口径），集群形态归后续任务裁决（事件 javadoc 已登记）。
3. 无其他缺项：brief 七条决策（事件契约/枚举/补列+清因/配置键/筛选/测试口径/冒烟序列）逐条核对均有实现与证据。

## 4. 偏离计划之处

1. **基线裁决性补跑（非 brief 命令）**：为解决恢复须知「基线 1755 vs 应 >1755」的矛盾，以临时 worktree 在 11e200c 实跑全量（1723）定基线，随后重装 HEAD ot-core 并字节校验。结论：协调者笔记的 1755 是前一执行者实现后的计数（已含本任务新用例），实测 1755 与「1723+32」逐数吻合。
2. **prod 多跑一轮干净轮**：首轮按验收需要做了负路径探针（无令牌 401），日志留下 1 条预期业务 ERROR；为交付字面「全程 0 ERROR」按 0019 先例重启单脚本单登录的干净第 2 轮（0 ERROR）。
3. **dev 冒烟 S6 首轮脚本断言过严**：未预期库中存在中断会话遗留行（GW_T0020_A/B/C/D），其被筛选正确命中被判 MISMATCH；修正断言复跑全绿。**这是脚本缺陷不是实现缺陷**，两段原始输出均保留于 §2.3。
4. 采信口径：本报告全部证据为本会话重跑采集，未引用中断会话的任何输出（其 11-ERROR 说法未采信亦无需采信——本会话实测 dev 6 条/prod 1 条+0 条，均逐条归因为预期业务拒绝）。

## 5. 遗留问题

1. **判定器接线**（brief 已登记的既有遗留）：配置键已落位待消费；归接入/计算引擎侧任务。
2. **dev 库冒烟留痕（是否清理由协调者裁决）**：本会话新增 iot_gateway 两行——GW_T0020_01（id=14，终态：在线、reason=NULL、last_heartbeat_time=2026-11-22 12:31:00）、GW_T0020_DIS（id=15，禁用、offline=LWT_ABNORMAL，未删）；另有中断会话遗留 GW_T0020_A/B/C/D（id=10-13）与 0019 报告遗留 5 所列各行，处置口径一并归协调者。
3. **本机环境注意（非本任务产物，建议后续任务知悉）**：①本机 WSL 空闲即整机关停（.wslconfig 含 autoMemoryReclaim/mirrored 等实验项），mysql/redis 容器（host 网络）随之重启，恢复期约 1 分钟——每个触 WSL 的步骤前需探测 3307 就绪，长会话冒烟期间建议 WSL 保活进程；②存在前一会话遗留 worktree `D:\workspace\WisFactory-base-check`（36bcd1f detached），未触碰；③会话期间工作区出现非本任务的他方未跟踪目录（frontend/src/components/iot/formula/、frontend/tests/unit/components/），保持原样未动。

——报告完（中断恢复接手，五条验收证据全部为本会话实测）。
