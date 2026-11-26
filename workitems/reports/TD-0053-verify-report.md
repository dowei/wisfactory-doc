# TD-0053 TDengine 真实联调核验报告

- 执行：TD-0053 执行者（ZCode agent）
- 日期：2026-11-22 23:00 ~ 23:58（本地）
- 服务器：http://192.168.31.145:6041（taosAdapter REST，Basic root/taosdata；全程只用 REST，符合项目 REST-only 架构）
- **服务器版本：`SELECT SERVER_VERSION()` → `3.4.3.9.community`**（client_version() 同值）
- 参照代码：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdmin.java`、`TdengineAdmin.java`、`service/impl/IotQuotaStatsServiceImpl.java`、`quota/QuotaProperties.java`、`application.yml`（OT_TDENGINE_* 键族）
- 方法：curl 直发 REST；每项结论带原始输出；网络超时类异常重试一次再定性（B1 重试 2 次均为 code 24 → 稳定复现，非瞬时）

## 总表

| 项 | 内容 | 结论 |
| --- | --- | --- |
| a | taosAdapter Basic 认证与库上下文行为 | **通过**（与 HttpTdengineAdmin 实现假设逐条吻合） |
| b | ins_disk_usage 现场版本可用性 | **不通过（服务器侧 fd）**——视图存在、项目 SQL 解析通过，执行真实库取数稳定复现 code 24；待 fd 根治复测 |
| c | 建库语句真实解析（scratch 建库） | **不通过（服务器侧 fd）**——DDL 语法/语义被接受，但 CREATE 挂起 → 库卡 creating（918）且不可 DROP；待 fd 根治复测 |
| d | 吞吐用量源接线与 type=throughput 告警 | **部分完成**：吞吐源确认未接线（待 0029/0068）；存储 fail-soft 路径已真机运行时实测命中；[QUOTA-WARN] 阈值触发形态待 0068 |
| e | 库名 36 字符长度上限 | **通过**——实测上限 64 字符，项目最长派生名 36 远低于上限 |
| f | 192 字节标识符上限真机验证 | **通过**——超级表/表名上限恰 192 字节（192 过/193 拒），与 LOG-0011 契约假设吻合；列/TAG 名上限 64 字节 |
| g | TAGS 值含单引号转义 | **部分通过**——REST 层字面转义三种形态实测通过；TAG 值写读往返待 0068（scratch 库不可用所致） |
| h | iot_default 健康确认 | **通过**——完全恢复、秒回、vgroup ready；库空（0 超级表，历史卡死期建表未完成，需补建） |

---

## a) Basic 认证与库上下文行为 —— 通过

### A1 正确口令、无库上下文、需要库的语句
```
POST /rest/sql  SHOW STABLES
→ HTTP 200 {"code":9728,"desc":"database not specified","timing":308300}
```

### A2 错误口令
```
POST /rest/sql（-u root:wrongpass）SHOW DATABASES
→ HTTP 200 {"code":855,"desc":"Authentication failure","timing":4136000}
```

### A3 无认证头
```
POST /rest/sql（无 Authorization）SHOW DATABASES
→ HTTP 401 {"code":4357,"desc":"no auth info input","timing":17700}
```

### A4 库上下文替代 USE（HttpTdengineAdmin 核心假设）
```
POST /rest/sql/iot_default  SHOW STABLES
→ HTTP 200 {"code":0,"column_meta":[["stable_name","VARCHAR",192]],"data":[],"rows":0}
```

### A5 错误用户名
```
POST /rest/sql（-u foo:taosdata）
→ HTTP 200 {"code":849,"desc":"Invalid user","timing":3296200}
```

### A6 不存在的库上下文
```
POST /rest/sql/wisfactory_verify_nonexist  SHOW STABLES
→ HTTP 200 {"code":9728,"desc":"database not specified","timing":2710800}
```

### A7 text/plain Content-Type（项目同款请求头）
```
POST /rest/sql -H "Content-Type: text/plain" SHOW DATABASES
→ HTTP 200 code 0，8 行（information_schema/performance_schema/iot_default/log/test_td1..4）
```

### 与实现假设逐条比对
| HttpTdengineAdmin 假设 | 实测 | 结果 |
| --- | --- | --- |
| Basic `base64(user:pass)` 认证 | A1 通过、A2/A3/A5 三种失败形态齐 | 吻合 |
| 应答 JSON `{"code":0,...,"data":...}` | 全部请求均此形态（另有 column_meta/rows/timing 字段） | 吻合 |
| HTTP≥400 → 走 HTTP 分支异常 | A3 HTTP 401 + JSON 体 | 吻合（异常消息含 HTTP 码与 body 摘要） |
| code≠0 → 走错误码分支异常 | A2（HTTP 200 + code 855）等 | 吻合（异常消息含 code 与 desc） |
| REST 库上下文 `/rest/sql/<db>` 替代 USE | A4 code 0 / A1 无上下文 9728 | 吻合 |
| databaseExists 取 SHOW DATABASES 首列比对 | SHOW DATABASES column_meta 首列即 name | 吻合 |
| 读超时 5s 足够 | mnode 级操作毫秒级；但 CREATE DATABASE >30s（见 c） | 5s 超时会触发 → BusinessException → fail-soft 重试轮（口径正确，但现场建库会持续失败，见 c） |

行为怪癖（记录）：creating 态库作 URL 上下文 → 9750 "Database not specified"（f3 实测）；错误码 9728 同时用于语法错与 "database not specified"。

---

## b) ins_disk_usage 现场版本可用性 —— 不通过（服务器侧 fd 未根治）

版本：**3.4.3.9.community**（`SELECT SERVER_VERSION()` → `{"code":0,...,"data":[["3.4.3.9.community"]],"rows":1}`）。

`SHOW SERVER VERSION` 语句在本版本不存在：`{"code":9728,"desc":"syntax error near \"version\""}`（项目不使用该语句，无影响）。`SHOW iot_default.DISK_USAGE` 同样语法不存在：`{"code":9728,"desc":"syntax error near \"disk_usage\""}`（项目实际走 information_schema 视图，无影响）。

### B1 项目原句（对 iot_default）
```
SELECT SUM(wal_size + data1 + data2 + data3 + cache_rdb + table_meta + ss + raw_data) AS total_kb
  FROM information_schema.ins_disk_usage WHERE db_name = 'iot_default'
→ HTTP 200 {"code":24,"desc":"Too many open files","timing":22463900}
```
**重试一次：`{"code":24,...,"timing":2714900}` —— 稳定复现，非瞬时。**

### B2 视图字段全貌（SELECT *）同错
```
SELECT * FROM information_schema.ins_disk_usage WHERE db_name = 'iot_default'
→ {"code":24,"desc":"Too many open files","timing":5314700}
```

### B4 作用域对照（mnode 级全部正常）
```
SELECT * FROM information_schema.ins_dnodes
→ code 0：单 dnode，endpoint 0ec3b485ac52:6030（容器），vnodes=11 / support_vnodes=21，
  status=ready，create_time 2026-11-22T05:32:50.575Z，reboot_time 2026-11-22T14:57:21.736Z（今日已重启过）
```

### B7 关键补充：WHERE 指向不存在的库 → 干净返回（fd 错误仅在真实库取数时触发）
```
... WHERE db_name = 'wisfactory_verify_nonexist'
→ {"code":0,"column_meta":[["total_kb","DOUBLE",8]],"data":[],"rows":0}
```

### 定性
- 视图 ins_disk_usage 在 3.4.3.9 存在且项目 SQL **解析通过**（错误是运行期 24，非 9728 语法错）——代码注释「版本不适配时表现为 SQLERROR」的预防成立。
- 执行需落到 vnode 存储统计 → **稳定 EMFILE（code 24）**：taosd 今日重启后仍如此 → fd 修复未达 vnode 文件路径（dnode endpoint 为容器 ID，ulimit 修复若只做在宿主/systemd 层则未进入容器进程；建议核对容器层 nofile 与 `/proc/<pidof taosd>/limits`）。
- 对业务的影响口径（与代码设计一致）：storageUsageBytes → BusinessException → 统计任务按不可用跳轮、WARN 节流（d 项已真机实测该路径）。**取数成功形态（含 NULL/空 data → 0 字节回填分支）待 fd 根治后复测。**

---

## c) 建库语句真实解析 —— 不通过（服务器侧 vnode 创建路径卡死）

### C1 项目同款 DDL 建库（附录 B 参数全写齐，与 HttpTdengineAdmin 逐字一致）
```
CREATE DATABASE IF NOT EXISTS wisfactory_verify_a KEEP 1825d DURATION 10d BUFFER 256
（POST /rest/sql，Content-Type: text/plain，--max-time 20）
→ HTTP_STATUS:000（20s 无响应，连接未回包）
```

### C2 建库落卡死态
```
SHOW CREATE DATABASE wisfactory_verify_a
→ {"code":918,"desc":"Database in creating status","timing":1169800}
SHOW DATABASES → 9 行，含 wisfactory_verify_a
```
60s 后、+3 分钟轮询（POLL1~3）均为 918；约 12 分钟后复测仍 918。**不自愈。**

### C12 资源最小化变体同样卡死（排除资源参数因素）
```
CREATE DATABASE IF NOT EXISTS wisfactory_verify_b VGROUPS 1 KEEP 1825d DURATION 10d BUFFER 256
→ 30s 无响应（HTTP 000）→ 918
CREATE DATABASE IF NOT EXISTS wisfactory_verify_c VGROUPS 1 REPLICA 1 WAL_LEVEL 0 WAL_RETENTION_PERIOD 0 KEEP 1825d DURATION 10d BUFFER 256
→ 30s 无响应（HTTP 000）→ 918
```

### C6~C10 卡死库不可 DROP（逃生路径不存在）
```
DROP DATABASE IF EXISTS wisfactory_verify_a → HTTP 200 code 0 affected_rows=0（库未删）
DROP DATABASE wisfactory_verify_a（无 IF EXISTS）→ {"code":918,"desc":"Database in creating status"}
DROP DATABASE IF EXISTS wisfactory_verify_a FORCE → code 0 affected_rows=0（库未删）
```
终态 SHOW DATABASES：a/b/c 三库仍在（见「清理证明」节）。与冻结期 iot_default 症状完全同型（当时靠 taosd 重启恢复）。

### 对照：iot_default 库参数正确（此前轮次创建成功过的库）
```
SHOW CREATE DATABASE iot_default
→ "CREATE DATABASE `iot_default` BUFFER 256 CACHESIZE 1 CACHEMODEL 'none' ... DURATION 10d ...
   ... KEEP 1825d,1825d,1825d ... PRECISION 'ms' REPLICA 1 ... VGROUPS 2 ..."
```
KEEP 1825d / DURATION 10d / BUFFER 256 与附录 B 一致 → 建库参数语义一旦建成即正确。

### 定性
- DDL 的解析/语义校验链完整（e 项探针证明名称校验、REPLICA 校验等均先于 vnode 创建）。
- **vnode 创建（文件创建）路径在当前服务器 fd 状态下必然挂起** → 建库不可用 → `数据库不存在 → 建库 → 四张超级表 → 写入` 全链阻塞。
- 项目口径下现场表现：REST 5s 读超时 → BusinessException → 扫描任务 WARN → 下轮重试 → 服务端反复卡 creating（且无法从 REST 清理）——**运维必须先修 fd 再联调**，否则每轮重试留垃圾库。
- 完整「建库 → SHOW CREATE 核对 → SHOW DATABASES 判存 → DROP」闭环实测待 fd 根治后以 scratch 库重做。

---

## d) 吞吐用量源接线与 type=throughput 告警 —— 部分完成

### 代码结论（数据源确认）
- **吞吐源未接线**：`IotQuotaStatsServiceImpl` 类注释明确「吞吐（type=throughput）同型告警：用量源归接入侧（ingest），随其交付接线」；`warnQuota(...)` 为私有方法，仅 `warnIfOverThreshold` 以 type="storage" 调用。→ **type=throughput 告警实测：待 0029（ingest 交付）/0068**。
- 存储源 = `tdengineAdmin.storageUsageBytes(tenantCode)` → `information_schema.ins_disk_usage`（B1 SQL，KB×1024）。
- 调度：sys_job 播种（`resources/sql/upgrade/20261121_tenant_db_task.sql`）：`iotQuotaStatsServiceImpl.collectAndWarn` cron `0 0/5 * * * ?`；`tenantDbProvisionServiceImpl.scanPendingDatabases` cron `0 * * * * ?`；bean 白名单契约测试在位。

### 运行时实测（backend dev 档，8098 纪律全程遵守）
前置说明：工作区 ot-admin 因 TASK-0025 半成品改动**编译失败**（`PublishProperties.java` javadoc 内 `ack-*/target-*` 的 `*/` 提前终止注释 → 一串「非法字符/\uff0c」编译错）——本次未改任何代码，改用 21:14 的既有构建产物 `ot-admin/target/ot-admin-1.0.0.jar` 起服；git status 显示本轮未提交改动全部在 publish/ 域，tdengine/quota/sys_job 子系统与该 jar 一致。

启动证据：
```
23:28:31.141 Tomcat initialized with port 8098 (http)
23:28:49.417 SysJobServiceImpl - 定时任务[租户库建库扫描]已注册调度: 0 * * * * ?
23:28:49.419 SysJobServiceImpl - 定时任务[租户配额存储统计]已注册调度: 0 0/5 * * * ?
```
（env 注入 OT_TDENGINE_BASE_URL=http://192.168.31.145:6041 生效：错误来自远端 code 24 而非 localhost 连接拒绝。）

23:30:00 配额轮（第一轮）——**fail-soft WARN 形态真机命中**：
```
2026-11-22 23:30:00.134 [ot-job-2] WARN  n.o.a.m.iot.service.impl.IotQuotaStatsServiceImpl -
  TDengine 存储统计不可用，本轮跳过（连续失败第 1 轮，仅首轮与每 20 轮告警防风暴）：
  失败租户 1/1，最后失败 tenant=default 异常摘要：存储用量 iot_default失败：TDengine 错误码 24（Too many open files）
```
23:35:00 第二轮——**防风暴节流生效**（WARN 计数仍为 1，静默跳过）：
```
23:35:00.015 [ot-job-4] SELECT ... FROM sys_tenant WHERE deleted = 0 AND (status = ?)   ← 轮询正常进行
sys_job_log：job_id=3 status=1 cost_time=38（status 语义 0=失败 1=成功 → 任务调用成功，fail-soft 未炸穿）
```

建库扫描（每分钟）全程安全：
```
sys_tenant 仅 1 行：id=1 tenant_code=default status=1 db_created=1 storage_quota_gb=NULL
扫描轮 pending 查询 Total: 0 → 全程日志 grep "CREATE DATABASE|建库 iot" 计数 = 0（零 CREATE 调用，零服务器污染）
```

[QUOTA-WARN] 超阈值行未触发——两个原因且均为设计使然：default 租户 `storage_quota_gb=NULL`（未设配额=不限、永不告警，ADR-0011 口径）；且 ins_disk_usage 取数本身失败（b 项）。**[QUOTA-WARN] 阈值触发的真机形态待 0068（前置：fd 根治 + 租户设 quota）。**

---

## e) 库名 36 字符长度上限实测 —— 通过（上限实测 64，项目契约安全）

方法：`CREATE DATABASE <名> REPLICA 3` 在单 dnode 上必被语义拒绝（code 820 "Out of dnodes"），不触发 vnode 创建、零副作用；名称校验先于该检查（200 字符名对照报 9751 证明校验顺序）。探针名均以 `wisfactory_verify_` 前缀构造精确长度。

```
e0 短名基线 wisfactory_verify_e00        → {"code":820,"desc":"Out of dnodes"}（名称通过）
e1 36 字符 wisfactory_verify_e36_abcdefghijklm   → {"code":820,...}（名称通过）
e2 37 字符 wisfactory_verify_e37_abcdefghijklmn  → {"code":820,...}（名称通过）
e3 200 字符标定 → {"code":9751,"desc":"Invalid identifier name: wisfactory_verify_cal_0123...","timing":219900}
e4 37 字符 + 伪参数 XYZFOO 1 → {"code":9728,"desc":"syntax error near \"xyzfoo 1\""}（名称通过，语法错在后）
e5 64 字符 wisfactory_verify_n64_aaaa...(64)     → {"code":820,...}（名称通过）
e6 65 字符 wisfactory_verify_n65_aaaa...(65)     → {"code":9751,"desc":"Invalid identifier name: wisfactory_verify_n65_..."}
```

**结论：库名上限 = 64 字符（64 过 / 65 拒，9751）。** 佐证：SHOW DATABASES column_meta `name VARCHAR(64)`、SHOW CREATE DATABASE `Database VARCHAR(65)`。
项目契约：`iot_<tenant_code>`，tenant_code ≤32 → 派生库名最长 **36 字符 ≤ 64**，安全余量充足。TdengineAdmin.databaseName 的 ≤32 校验继续成立（其目的是注入面收窄，不是服务器上限对齐）。
注：36/37 字符名「完整建库成功」形态当前被 c 项 vnode 故障阻断，待 fd 修复后可用 scratch 库复测——不影响本项上限结论。

---

## f) 192 字节标识符上限真机验证（LOG-0011 登记）—— 通过（上限恰 192，与契约假设吻合）

iot_default 现状为空库（0 超级表，见 h），无现存超级表可 SHOW CREATE TABLE 对照契约 DDL；改用**零变异只读探针**：`SHOW CREATE STABLE/SHOW CREATE TABLE iot_default.<名>`（库内无任何表，任何名字必 9731 不存在——纯元数据查找，无创建无变更）。探针名精确构造长度（ASCII，1 字符 = 1 字节）：

```
f6 基线 短名 SHOW CREATE STABLE iot_default.wisfactory_verify_ok
   → {"code":9731,"desc":"Table does not exist"}（名称校验通过）
f7 192 字节名 → {"code":9731,"desc":"Table does not exist"}        ← 192 通过
f8 193 字节名 → {"code":9751,"desc":"Invalid identifier name: wisfactory_verify_s193_aaa..."} ← 193 拒
f9 200 字节名 → {"code":9751,...}
f10 SHOW CREATE TABLE 192 字节 → {"code":9731,...}（表名 192 通过）
f11 SHOW CREATE TABLE 193 字节 → {"code":9751,...}（表名 193 拒）
```

**超级表名/表名上限 = 恰 192 字节**（SHOW STABLES/SHOW TABLES column_meta `VARCHAR(192)` 同构佐证）——与 LOG-0011/TASK-0074 遗留 3 的「192 字节标识符上限」契约假设完全吻合，报错形态：code 9751 "Invalid identifier name: <名>"。

列名/TAG 名上限（同法，只读 SELECT 于 log.adapter_status，LIMIT 0）：
```
f14 列名恰 192 字节 → {"code":9751,...}（被拒！列名上限比表名严）
f16 基线 40 字节不存在列 → {"code":9730,"desc":"Invalid column name: wisfactory_verify_no_such_col"}（合法长度形态）
f17 列名恰 64 字节 → {"code":9730,...}（64 通过校验）
f18 列名恰 65 字节 → {"code":9751,...}（65 拒）
f19 TAG 名 64 字节（WHERE 探针）→ {"code":9730,...}（通过）
f20 TAG 名 65 字节 → {"code":9751,...}（拒）
```
**列名/TAG 名上限 = 64 字节。**

契约比对：四张超级表 DDL（HttpTdengineAdmin.STABLE_DDL）表名 ≤15 字符、列名 ≤7、TAG 名 ≤13，BINARY(96/180/64/32/16/8) 是**值宽度**不是标识符长度——全部远低于上限。**上限不构成契约风险。**
「边界值建表成功」形态待 fd 修复后 scratch 库复测（当前 CREATE STABLE 路径随建库一并不可用；卡死库上下文一律 9750）。

---

## g) TAGS 值含单引号转义（LOG-0011 遗留 3，0029 真执行器预探）—— 部分通过

REST 层字面转义（SELECT 字面量经完整 REST 路径往返）：
```
g1 成对转义   SELECT 'it''s' AS v → {"code":0,...,"data":[["it's"]],"rows":1}
g2 反斜杠     SELECT 'it\'s' AS v → {"code":0,...,"data":[["it's"]],"rows":1}
g3 双引号包裹 SELECT "it's"  AS v → {"code":0,...,"data":[["it's"]],"rows":1}
```
**三种形态在 3.4.3.9 REST 层均正确产出含单引号字面量。**

TAG 列真实写入 + 回读（scratch 超级表 INSERT ... USING ... TAGS('it''s') + tag 过滤 SELECT）需可用 scratch 库——被 c 项服务器 vnode 故障阻断（三个 scratch 库全部卡 creating、其上下文不可用）→ **待 0068 复测**。对 0029 执行器的预判：SQL 标准成对 `''` 与反斜杠 `\'` 皆可靠，执行器按其一实现并在联调时以本项用例回归。

---

## h) iot_default 健康确认 —— 通过（从 creating 卡死态完全恢复；库为空）

```
SHOW iot_default.STABLES
→ {"code":0,"column_meta":[["stable_name","VARCHAR",192]],"data":[],"rows":0}   （毫秒级返回）
SHOW CREATE DATABASE iot_default → code 0，完整参数串（见 c 节）——BUFFER 256 / DURATION 10d / KEEP 1825d 与附录 B 一致
SELECT COUNT(*) FROM information_schema.ins_stables WHERE db_name='iot_default' → code 0，data [[0]]（0 张）
SHOW iot_default.VGROUPS
→ code 0：vgroup_id 3、4，db_name=iot_default，tables=0，v1_dnode=1，v1_status=leader，
  "6/6"，is_ready=true，cacheload=0（两行同构）
SHOW iot_default.TABLES → 0 行
```

- 异常：**无**。卡死态完全解除（metadata 秒回、vgroup ready）。
- 事实记录（非异常）：库为空（0 超级表 0 表）——冻结期「卡 creating」时建表未完成。default 租户 `db_created=1`（扫描任务自愈已认），但四张超级表（st_point_num/st_point_state/st_item_value/st_metric_value）**尚未建**。`createDatabase` 为幂等 IF NOT EXISTS，0068 联调时对该租户重触发即可补齐——无需手工干预、无需删库。
- 只读观察遵守：未对 iot_default/test_td1..4/log 做任何写操作（log 库仅只读 SELECT/DESCRIBE）。

---

## 服务器版本

- TDengine Server/Client：**3.4.3.9.community**（`SELECT SERVER_VERSION()` / `SELECT CLIENT_VERSION()` 同值）
- 部署形态：单 dnode 容器（endpoint `0ec3b485ac52:6030`），vnodes 11 / support 21，dnode create 2026-11-22T05:32:50Z，reboot 2026-11-22T14:57:21Z（fd 修复伴随的重启）
- 本次核验触达过的错误码全集：0（成功）、24（Too many open files）、820（Out of dnodes）、849（Invalid user）、855（Authentication failure）、918（Database in creating status）、4357（no auth info input，HTTP 401）、9728（syntax error / database not specified）、9730（Invalid column name）、9731（Table does not exist）、9750（Database not specified）、9751（Invalid identifier name）

## 清理证明与遗留

### 已遵守的约束
- 仅创建 `wisfactory_verify_*` 命名对象；未 DROP/ALTER 任何现存库表；未改任何代码文件；未动 workitems 库（仅写本报告）；backend 用毕即停（23:28 起 → 23:39 止，`dev.cmd stop` → PID 52196 已停止，8098 复测 Connection refused 已释放）；WSL wisfactory-mysql/wisfactory-redis 容器原样未动；未 commit。

### scratch 建库终态（未达成清理，如实留痕）
```
DROP DATABASE IF EXISTS wisfactory_verify_a/b/c 各一次（收尾轮）→ 均 code 0 affected_rows=0
SHOW DATABASES 终态（11 行）：information_schema / performance_schema / iot_default /
  wisfactory_verify_c / log / wisfactory_verify_b / test_td3 / test_td2 / test_td4 /
  wisfactory_verify_a / test_td1
```
**遗留 1（运维动作项）**：`wisfactory_verify_a / wisfactory_verify_b / wisfactory_verify_c` 三库卡 creating 态（918），REST 层 DROP（含 FORCE）全部无效——与冻结期 iot_default 同型。**taosd 容器层 fd 根治并重启后**，三库将同 iot_default 一样恢复可操作，届时执行即可清理：
```
DROP DATABASE IF EXISTS wisfactory_verify_a;
DROP DATABASE IF EXISTS wisfactory_verify_b;
DROP DATABASE IF EXISTS wisfactory_verify_c;
-- 然后 SHOW DATABASES 证明三库消失（预期回到 8 库基线）
```

**遗留 2（运维动作项，根因）**：fd 问题未根治——mnode 级操作全部正常，vnode 文件级操作（ins_disk_usage 取数、CREATE DATABASE 的 vnode 创建）稳定 EMFILE code 24，且 taosd 今日已重启过一次仍如此。dnode endpoint 为容器 ID → 疑似 ulimit 修复未达容器进程。建议：在容器层确认并设置 nofile（`docker exec <tdengine容器> cat /proc/$(pidof taosd)/limits`），或以 `--ulimit nofile=<soft>/<hard>` / compose `ulimits` 重拉容器；重启后先复测 B1 SQL（应返回数值行）与 C1 建库（应秒回）再进行 0068 联调。

**遗留 3（开发工作区阻塞项）**：ot-admin 当前编译失败——TASK-0025 半成品 `PublishProperties.java` javadoc 第 8 行 `ack-*/target-*` 中 `*/` 提前终止注释（编译器报「非法字符 \uff0c」串）。TASK-0025 完成或先修注释（如改为 `ack-* / target-*`）前，任何 backend 真机联调只能复用旧 jar。本次未改代码，留给 TASK-0025。

**遗留 4（复测清单，fd 根治后）**：
1. b：ins_disk_usage 对真实库取数成功形态（含空库/NULL → 0 字节回填分支）；
2. c：完整「项目同款 DDL 建库 → SHOW CREATE DATABASE 核对 → SHOW DATABASES 判存 → DROP 清理」闭环（scratch 库）；
3. e/f：36 字符库名与 192 字节表名「建成功」形态（上限数值已定案，补成功侧证据）；
4. g：TAGS 含单引号值的写入 + 回读往返；
5. d：[QUOTA-WARN] 阈值触发真机形态（给 default 租户设临时 storage_quota_gb 后触发一轮）。

**遗留 5（登记口径）**：吞吐 type=throughput 告警数据源归 0029 ingest 交付接线——本次代码确认未接线，接口形态（复用 `warnQuota`）已就位。

## 一句话总结

**TD-0053 不能闭合、保持挂 TASK-0068**：接口行为类核验（a 认证与库上下文、e 库名上限 64、f 标识符上限 192/64、g 字面转义、h 恢复健康）全部通过且与代码假设吻合，但服务器 vnode 文件路径 fd 耗尽在 taosd 重启后仍未根治（ins_disk_usage 取数 code 24、CREATE DATABASE 挂起卡 creating 且不可 DROP），b/c/d 的取数与建库实测不通过——**0029 写路径验收闸门未解锁**，解锁前置 = 容器层根治 fd 并重启 → 遗留 4 复测清单通过 → 0068 联调（含 iot_default 四张超级表幂等补建）。
