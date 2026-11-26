# TD-0053 遗留 4 第 5 项（d）：[QUOTA-WARN] 阈值触发真机形态补测报告

- 执行：TD-0053 d 项补测执行者（ZCode agent）
- 日期：2026-11-23 01:15 ~ 01:40（本地）
- 环境：backend dev 档（ot-admin 8098，用毕即停）；TDengine http://192.168.31.145:6041（经 env `OT_TDENGINE_BASE_URL` 注入，与上一轮同口径）；MySQL 8.0.42@127.0.0.1:3307（WSL wisfactory-mysql，sql-exec.sh 每语句单独调用）
- 前置状态：git 工作区干净（仅未跟踪 `.agents/`，TASK-0025/0026 在途改动已收尾，本轮 `mvn install -pl ot-core -DskipTests` + `dev.cmd build` 均 BUILD SUCCESS）；8098 启动前空闲；基线 MySQL：
  ```
  SELECT id, tenant_code, status, db_created, point_quota, storage_quota_gb FROM sys_tenant WHERE deleted = 0
  → 1  default  1  1  null  null
  SELECT id, job_name, invoke_target, cron_expression, status FROM sys_job WHERE deleted = 0
  → 3  租户配额存储统计  iotQuotaStatsServiceImpl.collectAndWarn  0 0/5 * * * ?  0   （status=0 正常）
  ```
- 约束遵守：未改任何代码；TDengine 全程零直连（用量取数全部由 backend 服务运行期只读 SELECT 完成，未碰写路径）；未 commit；workitems 库只写本报告。

---

## 1. 代码定位（步骤 1 结论）

告警链路全部在 **`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/`**：

| 要素 | 位置 | 口径 |
| --- | --- | --- |
| 字段所在表 | `sys_tenant.storage_quota_gb`（`int DEFAULT NULL`，`resources/sql/upgrade/20261121_sys_tenant.sql:42`）；实体 `entity/SysTenant.java:58` Integer | GB 整数；管理入口 `SysTenantController` 随新增/编辑接口下发 |
| 阈值比较 | `service/impl/IotQuotaStatsServiceImpl.java:83-96` `warnIfOverThreshold` | `quotaGb==null \|\| quotaGb<=0 → return`（**NULL/0/负 = 不限永不告警**）；`ratio<=0 \|\| ratio>1 → return`（越界=告警关闭）；除法形态 `(double) usageBytes / quotaBytes >= ratio` |
| ratio 配置 | `quota/QuotaProperties.java:22` 默认 0.8；`application.yml:63-64` `storage-warn-ratio: ${OT_QUOTA_STORAGE_WARN_RATIO:0.8}` | 有效范围 (0,1]；yml 自带 env 注入口 |
| 日志格式 | `IotQuotaStatsServiceImpl.java:102-104` `log.warn("[QUOTA-WARN] tenant={} type={} usage={} quota={}", ...)` | 固定可 grep，usage/quota 均字节 |
| 防风暴节流 | 同文件 `recordRoundOutcome`（:110-125）+ `QuotaProperties.failLogEveryNRounds=20` | 节流只作用于 **TDengine 不可用**的 fail-soft WARN；[QUOTA-WARN] 软限告警本身每轮比对、无节流（属预期告警非风暴源） |
| 调度 | sys_job id=3 `iotQuotaStatsServiceImpl.collectAndWarn`，cron `0 0/5 * * * ?` | 任务失败/成功均写 sys_job_log（status 0=失败 1=成功） |
| 用量源 | `tdengineAdmin.storageUsageBytes(tenantCode)` → `information_schema.ins_disk_usage` SUM×1024 | 本轮实测正常（见形态 2 usage 值解读） |
| PointQuotaCounter | `quota/PointQuotaCounter.java` | **点位硬限**计数源接口（TASK-0021 接线），与存储软限告警（IotQuotaStatsService）是两条独立链路，与本项无关 |

**触发条件总结**：`status=1` 租户 + `storage_quota_gb ≥ 1` + `ot.quota.storage-warn-ratio ∈ (0,1]` + `usage/quotaBytes ≥ ratio` → `[QUOTA-WARN] tenant=<code> type=storage usage=<bytes> quota=<bytes>`，只告警不停写（ADR-0011）。

## 2. 触发方案设计（为什么不是 storage_quota_gb=0）

任务草案提示「storage_quota_gb=0」，但按代码口径 **0 恰好 = 不限（永不告警）**；int 列最小可行值为 **1**。而 usage 仅约 190KB（fd 根治后空库元数据占用），对 1GB@0.8 需 859MB 用量——TDengine 写路径被禁（用量只读），usage 做不大。唯一可行触发路径是 yml 自带配置口 `ot.quota.storage-warn-ratio`（有效范围 (0,1]），以启动参数注入 **0.0001**：

- 这是**配置注入非代码改动**（dev.env.example 文档化的 APP_ARGS 用法：`--ot.quota.storage-warn-ratio=0.0001`），随进程停止即消失；
- 数据侧仍按任务原意设 `storage_quota_gb=1`（最小可行值），还原置 NULL；
- 解除形态设计为**不重启只还原 quota**——ratio 保持 0.0001 不变，隔离单变量证明「告警消失由 quota 还原驱动」。

## 3. 三形态原始日志（步骤 2/3/4 实测）

日志文件：`backend/logs/ot-admin-dev.log`（dev.ps1 单级轮转，01:25 轮在 `ot-admin-dev.prev.log`）。配额轮 cron `0 0/5 * * * ?`。

### 形态 1 · 基线轮（01:25:00，quota=NULL，默认 ratio 0.8）——静默只统计不告警

本轮日志窗口（01:24~01:25）经 `grep -v DEBUG` 后 **0 行 WARN/INFO**；ot-job-4 执行配额任务：取租户（Total: 1）→ REST 取用量（成功）→ quota=null 直接 return → 任务成功收尾：

```
2026-11-23 01:25:00.025 [ot-job-4] DEBUG n.o.a.module.iot.mapper.SysTenantMapper.selectList - ==> Parameters: 1(Integer)
2026-11-23 01:25:00.030 [ot-job-4] DEBUG n.o.a.module.iot.mapper.SysTenantMapper.selectList - <==      Total: 1
（231ms 后，无任何 WARN/INFO）
sys_job_log：job_id=3 租户配额存储统计 status=1 cost_time=231  create_time=2026-11-23 01:25:00
```

### 形态 2 · 阈值触发轮（01:30:00，quota=1 + ratio=0.0001）——[QUOTA-WARN] 命中

数据前置：`UPDATE sys_tenant SET storage_quota_gb = 1 WHERE tenant_code = 'default' AND deleted = 0` → `OK(1)`，复核 `storage_quota_gb=1`。停服重启（PID 57072），java 进程命令行取证（`Get-CimInstance Win32_Process`）：

```
"C:\Users\dowil\.jdks\corretto-21.0.12\bin\java.exe" -Xms512m -Xmx1024m -Dfile.encoding=UTF-8 -jar
  "D:\workspace\WisFactory\backend\ot-admin\target\ot-admin-1.0.0.jar" --ot.quota.storage-warn-ratio=0.0001
```

**[QUOTA-WARN] 原始日志行（全轮唯一，全程计数 = 1）**：

```
2026-11-23 01:30:00.241 [ot-job-3] WARN  n.o.a.m.iot.service.impl.IotQuotaStatsServiceImpl - [QUOTA-WARN] tenant=default type=storage usage=194560 quota=1073741824
```

逐字段解读：
- `tenant=default` —— 租户标识正确；
- `type=storage` —— warnQuota 固定传参；
- `usage=194560` 字节 = **190KB×1024**，与第二轮报告 b 项「iot_default 空库 190KB（2 vgroup×95KB SUM）」**完全一致**——storageUsageBytes 取数链路真机正常的同源佐证；
- `quota=1073741824` = 1×1024³——GB→字节换算正确；
- 数学：194560/1073741824 ≈ **0.000181 ≥ 0.0001** → 触发（默认 0.8 阈值下该值远低于阈值不触发，反推自洽）；
- 告警不停轮：sys_job_log `job_id=3 status=1 cost_time=233`（01:30:00）——warn-and-continue，任务正常收尾。

### 形态 3 · 解除轮（01:35:00，quota=NULL 还原，ratio=0.0001 保持不变）——告警消失

数据还原：`UPDATE sys_tenant SET storage_quota_gb = NULL WHERE tenant_code = 'default' AND deleted = 0` → `OK(1)`，复核 `storage_quota_gb=null`。**不重启**（进程与 ratio 不变）。01:35 轮日志窗口 `grep -v DEBUG` 后 **0 行 WARN/INFO**，任务正常收尾：

```
sys_job_log：job_id=3 租户配额存储统计 status=1 cost_time=174  create_time=2026-11-23 01:35:00
```

唯一变量（quota 还原）驱动告警消失，解除形态成立。

### 背景注记（非本项形态，如实留痕）

第一次启动用 dev 默认 `OT_TDENGINE_BASE_URL`（localhost:6041，与本环境真实服务器不符），01:20 轮复刻了上一轮已登记的 fail-soft 形态（顺带证明该路径依旧工作、防风暴文案口径不变）：

```
2026-11-23 01:20:00.090 [ot-job-4] WARN n.o.a.m.iot.service.impl.IotQuotaStatsServiceImpl -
  TDengine 存储统计不可用，本轮跳过（连续失败第 1 轮，仅首轮与每 20 轮告警防风暴）：
  失败租户 1/1，最后失败 tenant=default 异常摘要：存储用量 iot_default失败：TDengine REST 不可达 http://localhost:6041（java.net.ConnectException）
```

经 env 注入 `OT_TDENGINE_BASE_URL=http://192.168.31.145:6041` 重启后即消除（进程重启连续失败计数清零，与代码「恢复即清零」口径一致）。

## 4. 还原证明

1. **MySQL 写操作全集（仅 2 条，净效果 = 0）**：
   ```
   UPDATE sys_tenant SET storage_quota_gb = 1    WHERE tenant_code='default' AND deleted=0  → OK(1)
   UPDATE sys_tenant SET storage_quota_gb = NULL WHERE tenant_code='default' AND deleted=0  → OK(1)
   ```
2. **sys_tenant 终态**（与开场基线逐列一致）：
   ```
   SELECT id, tenant_code, status, db_created, point_quota, storage_quota_gb FROM sys_tenant WHERE deleted = 0
   → 1  default  1  1  null  null
   ```
3. **sys_job 未动**（3 行原样，status 均 0）：
   ```
   → 1 微信粉丝对账同步 / 2 租户库建库扫描 / 3 租户配额存储统计（cron/status 与基线一致）
   ```
4. **ratio 注入为进程级**：`--ot.quota.storage-warn-ratio=0.0001` 仅存在于 java 命令行，backend 已停（`dev.cmd stop` → PID 57072 已停止），该配置随进程消失；后续启动恢复默认 0.8。
5. **8098 已释放**：停服后 `netstat -ano | grep :8098` 无输出。
6. sys_job_log 新增行为任务运行正常产物（job_id=3 共 3 行本轮：01:25/01:30/01:35，均 status=1），非人工改动。

## 5. 一句话结论

**d 项通过**：[QUOTA-WARN] 阈值触发的真机形态（内容/字段/租户标识/字节换算/告警不停轮）与解除形态（quota 还原即静默、NULL=不限口径）均按代码口径实测命中，原始日志行与还原证据齐备；至此 TD-0053 遗留 4 五项（b/c/e/f/g + d）全部闭环，**TD-0053 可全面闭合（挂 TASK-0068 联调口径下）**——吞吐 type=throughput 告警数据源接线归 0029/0068 的既有登记口径不变；另登记运维事实：dev 默认 `OT_TDENGINE_BASE_URL` 指向 localhost，本机联调须以 env 注入真实服务器地址。
