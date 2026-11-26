# TD-0053 复测报告（第二轮：fd 根治后遗留 4 复测 + default 租户库补建）

- 执行：TD-0053 复测执行者（ZCode agent，第二轮）
- 日期：2026-11-23（本地）
- 服务器：http://192.168.31.145:6041（taosAdapter REST，Basic root/taosdata；全程只用 REST）
- 服务器版本：`SELECT SERVER_VERSION()` → `3.4.3.9.community`（`HTTP:200 CURL_TIME:0.068507s`）
- 前置状态（协调者 2026-11-23 实测，本轮基线复核一致）：容器已按 `--ulimit nofile=1048576` 重建 + 数据全清空；本轮开场 `SHOW DATABASES` 仅 3 系统库：
  ```
  {"code":0,"column_meta":[["name","VARCHAR",64]],"data":[["information_schema"],["performance_schema"],["log"]],"rows":3}
  HTTP:200 CURL_TIME:0.092275s
  ```
  上一轮遗留 1（a/b/c 三卡死库）随清空消失，遗留 2（fd）由本轮实测正面证明根治（见 b/c/e/f 各项）。
- 参照代码：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdmin.java`（createDatabase SQL、STABLE_DDL、STORAGE_USAGE_SQL 逐字提取）、`TdengineAdmin.java`（databaseName = `iot_<tenant_code>`）
- 方法：curl 直发 REST；建库类命令一律 `-m 60`；每项结论带原始输出与耗时。本轮未改任何代码、未起 backend、未动 workitems 库（仅写本报告）、未 commit。

## 总表

| 项 | 内容 | 结论 |
| --- | --- | --- |
| 任务 A | iot_default 补建 + 四张超级表 | **完成（带一处登记偏差）**——库参数逐项吻合；3 张逐字通过，st_metric_value 逐字 DDL 被服务器拒（**新发现 N1：`period` 保留字**），以唯一最小偏离形态（`` `period` ``）补建到位，4 张在位 |
| b | ins_disk_usage 项目真实 SQL 取数 | **通过**——真实库毫秒级返回（fd 根治正面证据）；空库=190/188 KB、写入后=217 KB、不存在库 rows 0 → data.isEmpty→0 分支 |
| c | 建库→核对→判存→DROP 闭环 | **通过**——scratch wisfactory_verify_c 全链 16.32s/1.60s，上一轮卡死完全解除 |
| e | 36 字符库名「建成功」侧 | **通过**——建库成功（35.08s，在建库慢已知特性内）+ SHOW CREATE 核对 + 判存在位 + DROP |
| f | 192 字节超级表名「建成功」侧 | **通过**——CREATE code 0（252ms）+ SHOW CREATE 往返 + SHOW STABLES 列出 |
| g | TAGS 单引号三形态写入+回读往返 | **通过**——三形态写入均 affected_rows=1，回读三行全部 `it's`，三形态过滤 COUNT 均=3 |
| d | [QUOTA-WARN] 阈值触发真机形态 | **顺延**——需起 backend dev，backend 工作副本被 TASK-0026 在途占用（串行排除线），待 0026 收尾后补测 |

---

## 任务 A：default 租户库补建（TenantDbProvisionService 判存自愈口径的手工等价执行）

判存：开场 SHOW DATABASES 无 iot_default（见基线）→ 需补建。执行前 SHOW iot_default.STABLES 因库不存在无从谈起，直接按 createDatabase 手工等价三步走。

### A-1 建库（项目 createDatabase SQL 逐字：`CREATE DATABASE IF NOT EXISTS <db> KEEP 1825d DURATION 10d BUFFER 256`，db=iot_default）
```
POST /rest/sql  CREATE DATABASE IF NOT EXISTS iot_default KEEP 1825d DURATION 10d BUFFER 256
→ {"code":0,"column_meta":[["affected_rows","INT",4]],"data":[[0]],"rows":1}
  HTTP:200 CURL_TIME:12.978525s
```

### A-2 SHOW CREATE DATABASE 核对（与超级表契约附录 B 参数比对）
```
SHOW CREATE DATABASE iot_default
→ {"code":0,...,"data":[["iot_default","CREATE DATABASE `iot_default` BUFFER 256 CACHESIZE 1 CACHEMODEL 'none'
   ... DURATION 10d WAL_FSYNC_PERIOD 3000 MAXROWS 4096 MINROWS 100 STT_TRIGGER 2 KEEP 1825d,1825d,1825d
   ... PRECISION 'ms' REPLICA 1 WAL_LEVEL 1 VGROUPS 2 ... WAL_RETENTION_PERIOD 3600 ..."]],"rows":1}
  HTTP:200 CURL_TIME:0.063159s
```
**KEEP 1825d / DURATION 10d / BUFFER 256 与附录 B 逐项吻合**（其余为服务器默认值展开）。

### A-3 四张超级表（STABLE_DDL 逐字，经 REST 库上下文 /rest/sql/iot_default，幂等 IF NOT EXISTS）
执行前 `SHOW iot_default.STABLES` → `rows:0`（空库确认，0.068592s）。

```
st_point_num    → {"code":0,...} HTTP:200 CURL_TIME:0.235267s   （逐字通过）
st_point_state  → {"code":0,...} HTTP:200 CURL_TIME:0.231593s   （逐字通过）
st_item_value   → {"code":0,...} HTTP:200 CURL_TIME:0.423218s   （逐字通过）
st_metric_value → {"code":9728,"desc":"syntax error near \"period binary(8))\"","timing":2702700}
                  HTTP:200 CURL_TIME:0.014395s                  （逐字被拒）
```

### 新发现 N1（重要）：`period` 是 3.4.3.9 的保留字，裸用作列/TAG 名必被拒

- iot_default 上逐字 DDL 重试一次，稳定复现：`{"code":9728,"desc":"syntax error near \"period binary(8))\"","timing":1610700}`（CURL_TIME 0.008341s）。
- scratch 库内对照探针（同型最小 DDL）：
  ```
  裸名      CREATE STABLE ... TAGS (period BINARY(8))    → {"code":9728,"desc":"syntax error near \"period binary(8))\""} （0.011456s）
  反引号    CREATE STABLE ... TAGS (`period` BINARY(8))  → {"code":0,...}（0.268320s）
  SHOW CREATE STABLE → ... TAGS (`period` VARCHAR(8)) ...（物理 TAG 名即 period；BINARY 服务器回显为 VARCHAR 同义别名，形状不变）
  ```
- **代码影响面（两处，均为裸 `period`）**：
  1. `HttpTdengineAdmin.STABLE_DDL[3]`（st_metric_value 建表 TAG 名）→ 当前代码对**任何新租户**建库全链必在第 4 步失败 → fail-soft 重试轮每轮卡同处（库与前三张幂等跳过，第 4 张永不过）。
  2. `OpenApiTimeseriesClient` 指标序列查询 `... AND period = '<period>'`（该文件 L187）→ 同源必 9728。
- **缓解现状**：st_metric_value 本期无写入方（OpenApiTimeseriesClient 类注释：计算引擎未交付，st_item_value/st_metric_value 本期无写入）→ **不阻塞 0029 点位写路径**（st_point_num/st_point_state 逐字通过）；但**指标链路联调前必须修**——修法即两处裸 `period` 改反引号 `` `period` ``（本轮不改代码，登记待办）。

### A-3 补建收尾（唯一偏离，登记）：st_metric_value 以反引号形态补建
```
CREATE STABLE IF NOT EXISTS st_metric_value (ts TIMESTAMP, value DOUBLE, yoy DOUBLE, mom DOUBLE)
  TAGS (tenant_id BIGINT, metric_code BINARY(96), instance_code BINARY(96), `period` BINARY(8))
→ {"code":0,...} HTTP:200 CURL_TIME:0.216139s
```
除 `` `period` `` 加反引号外与契约 DDL 逐字一致；列名/TAG 名/类型/顺序不变，物理 TAG 名为 `period`。

### A-4 终态证明
```
SHOW iot_default.STABLES
→ {"code":0,"column_meta":[["stable_name","VARCHAR",192]],
   "data":[["st_item_value"],["st_point_num"],["st_metric_value"],["st_point_state"]],"rows":4}
  HTTP:200 CURL_TIME:0.078179s

SHOW CREATE STABLE iot_default.st_metric_value
→ "CREATE STABLE `st_metric_value` (`ts` TIMESTAMP, `value` DOUBLE, `yoy` DOUBLE, `mom` DOUBLE)
    TAGS (`tenant_id` BIGINT, `metric_code` VARCHAR(96), `instance_code` VARCHAR(96), `period` VARCHAR(8))
    SECURITY_LEVEL 0 SECURE_DELETE 0"   （HTTP:200 CURL_TIME:0.064987s）

SHOW iot_default.VGROUPS → vgroup_id 9、10，均 leader / is_ready=true / tables=0（0.079066s）
```

自愈口径核对：default 租户 `db_created=1` 已认 + 库存在 → scanPendingDatabases 不会再触发 createDatabase（TdengineAdmin.databaseExists 判存语义），手工补建的库与 4 张超级表即服务运行期的稳态；st_metric_value 的反引号偏离不与运行期代码冲突（本期无写入方）。

---

## 任务 B：遗留 4 复测

### b) ins_disk_usage 项目真实 SQL —— 通过（fd 根治的正面证据）

SQL 逐字（HttpTdengineAdmin.STORAGE_USAGE_SQL，`String.format` 后形态，经 /rest/sql 无库上下文，与实现一致）：
```
SELECT SUM(wal_size + data1 + data2 + data3 + cache_rdb + table_meta + ss + raw_data) AS total_kb
  FROM information_schema.ins_disk_usage WHERE db_name = '<db>'
```

| 测点 | 原始输出 | 耗时 | Java 侧映射 |
| --- | --- | --- | --- |
| iot_default（4 超级表、0 行数据） | `{"code":0,...,"data":[[190]],"rows":1}` | 0.139237s | 190×1024=194560 字节（表元数据占用，非 0 属正确） |
| wisfactory_verify_c（2 超级表、0 行数据） | `{"code":0,...,"data":[[188]],"rows":1}` | 0.143449s | 188×1024 字节 |
| 同库写入 60 个时间戳后（63 行） | `{"code":0,...,"data":[[217]],"rows":1}` | 0.185470s | 217×1024 字节——**有数分支增量可见** |
| WHERE 指向不存在库 | `{"code":0,...,"data":[],"rows":0}` | 0.092092s | data.isEmpty → 返 0 分支（代码 L88-90） |

- 明细对照（`SELECT db_name, wal_size, cache_rdb, table_meta ...`）：视图按 vgroup 出行——log 1 行、iot_default 2 行、wisfactory_verify_c 2 行；写入前后 iot_default wal 1/1、cache_rdb 6/6、table_meta 88/88 → SUM 跨 vgroup 行求和正确（190=2×95）；scratch 写入后 wal 1→3、table_meta 88→100 → 188→217。SUM 的列内跨 vgroup 聚合口径与 Java 侧 `SUM(...) WHERE db_name=` 一致。
- **NULL→0 分支**：视图各列在真实库上均出数值（空库也有 table_meta/cache_rdb 元数据占用），NULL 形态无法在真实视图上自然构造；代码防御分支（first.get(0).isNull() → 0）保留，data 空形态已由不存在库实测（rows 0 → 0）。
- **与上一轮对比**：同一条 SQL 上一轮稳定 code 24（EMFILE），本轮毫秒级返回——**fd 根治在 vnode 存储统计路径得到正面证明**。

### c) 建库→SHOW CREATE 核对→判存→DROP 闭环（scratch 库 wisfactory_verify_c）—— 通过

```
1) 建库（项目 DDL 逐字，-m 60）
   CREATE DATABASE IF NOT EXISTS wisfactory_verify_c KEEP 1825d DURATION 10d BUFFER 256
   → {"code":0,...,"data":[[0]],"rows":1}  HTTP:200 CURL_TIME:16.318422s
2) SHOW CREATE DATABASE wisfactory_verify_c
   → code 0，参数串与 iot_default 同构：BUFFER 256 / DURATION 10d / KEEP 1825d,1825d,1825d /
     PRECISION 'ms' / REPLICA 1 / VGROUPS 2（0.059108s）
3) 判存（databaseExists 口径：SHOW DATABASES 首列比对）
   → data 含 "wisfactory_verify_c"（rows 5，0.069316s）
4) 测毕 DROP
   DROP DATABASE IF EXISTS wisfactory_verify_c
   → {"code":0,...} HTTP:200 CURL_TIME:1.598993s（真实删除，见清理证明）
```
**上一轮 C1~C12 全链卡死形态完全解除**：CREATE 秒级~十几秒回包（非 HTTP 000 挂死）、无 918 creating 态、DROP 即时生效。

### e) 36 字符库名「建成功」侧 —— 通过

```
名：wisfactory_verify_e36_abcdefghijklmn（bash ${#NAME}=36，上一轮仅测过名称校验拒绝侧与 820 语义拒绝侧）
CREATE DATABASE IF NOT EXISTS <名> KEEP 1825d DURATION 10d BUFFER 256（-m 60）
→ {"code":0,...,"data":[[0]],"rows":1}  HTTP:200 CURL_TIME:35.077415s
SHOW CREATE DATABASE <名> → code 0，BUFFER 256 / DURATION 10d / KEEP 1825d,... 可见（应答完整）
SHOW DATABASES → data 含该 36 字符名（0.060514s）
DROP DATABASE IF EXISTS <名> → code 0，1.522099s
```
上一轮结论「上限 64、项目最长派生名 36 安全余量充足」补齐成功侧证据：36 字符名可完整建库/判存/删除。本轮三次 CREATE DATABASE 耗时 12.98s/16.32s/35.08s，方差大但在「建库慢」已知特性带内，`-m 60` 均足够。

### f) 192 字节超级表名「建成功」侧 —— 通过

```
名：wisfactory_verify_s192_ + 'a'×169（bash 验证 ${#NAME}=192）
CREATE STABLE IF NOT EXISTS <名> (ts TIMESTAMP, value DOUBLE) TAGS (tenant_id BIGINT)
  （POST /rest/sql/wisfactory_verify_c）
→ {"code":0,...} HTTP:200 CURL_TIME:0.252431s
SHOW CREATE STABLE wisfactory_verify_c.<名> → code 0，应答完整回显 192 字节名与建表语句（显示侧截断，往返成功）
SHOW wisfactory_verify_c.STABLES → data 含该 192 字节名（0.063314s）
```
与上一轮 f6~f11 拒绝侧（192 过/193 拒）合璧：**192 字节标识符上限的双侧证据齐**，LOG-0011 契约假设（表名 ≤192 字节）完全成立。测毕随 scratch 库 DROP 一并清理。

### g) TAGS 值含单引号写入+回读往返 —— 通过（三形态全绿）

scratch 超级表：`CREATE STABLE IF NOT EXISTS st_quote_test (ts TIMESTAMP, value DOUBLE) TAGS (tenant_id BIGINT, note BINARY(64))`（code 0，0.279605s）。

```
g1 成对转义   INSERT INTO t_quote_a USING st_quote_test TAGS(1, 'it''s') VALUES(...)  → code 0 affected_rows=1（0.067536s）
g2 反斜杠     INSERT INTO t_quote_b USING st_quote_test TAGS(1, 'it\'s') VALUES(...)  → code 0 affected_rows=1（0.058433s）
g3 双引号包裹 INSERT INTO t_quote_c USING st_quote_test TAGS(1, "it's") VALUES(...)  → code 0 affected_rows=1（0.067392s）

回读  SELECT tbname, note FROM st_quote_test ORDER BY tbname
→ {"code":0,...,"data":[["t_quote_a","it's"],["t_quote_b","it's"],["t_quote_c","it's"]],"rows":1×3}（0.092522s）
  ——三种 SQL 形态落库后全部归一为 it's（单引号完整保留，无丢字符/无残留转义符）

过滤  WHERE note = 'it''s'  → count=3（0.070421s）
      WHERE note = 'it\'s'  → count=3（0.072386s）
      WHERE note = "it's"   → count=3（0.073410s）
      ——写入形态与过滤形态可任意组合，值等价
```
执行器注记（诚实留痕）：g3 首次执行因本机 bash 引号写法中断（SQL 未发出，服务器零影响），修正引号后重发成功——上表即重发结果。

**TAG 列含单引号值的「写入 + 回读 + 过滤」完整往返打通**，与上一轮 g1~g3 字面层证据合璧。0029/后续执行器可任选 `''` 成对或 `\'` 反斜杠实现，联调以本用例回归。

### d) [QUOTA-WARN] 阈值触发真机形态 —— 顺延（登记）

需要起 backend dev 档给 default 租户设临时 storage_quota_gb 后触发一轮；backend 工作副本当前被 TASK-0026 在途占用（串行排除线），本轮不起 backend。**待 0026 收尾后补测**。不阻塞 0029 写路径闸门（告警形态与写路径无依赖）。

---

## 清理证明

```
DROP DATABASE IF EXISTS wisfactory_verify_c                        → code 0（1.598993s）
DROP DATABASE IF EXISTS wisfactory_verify_e36_abcdefghijklmn       → code 0（1.522099s）
终态 SHOW DATABASES
→ {"code":0,"column_meta":[["name","VARCHAR",64]],
   "data":[["information_schema"],["performance_schema"],["iot_default"],["log"]],"rows":4}
  HTTP:200 CURL_TIME:0.060146s
```
**终态 = 基线 3 系统库 + iot_default（本轮唯一新增）**。所有 scratch 对象（wisfactory_verify_c 库及其内 st_period_probe / 192 字节超级表 / st_quote_test 三张、t_quote_a/b/c 三张子表、103 行数据；wisfactory_verify_e36_abcdefghijklmn 库）全部清除。iot_default 内仅 4 张契约超级表、0 行数据（vgroups tables=0）。未 DROP/ALTER 任何本轮新建对象以外的库表；/tmp/td53_insert.sql 为本机临时文件（非仓库文件）。

## 遗留

1. **d 项顺延**：[QUOTA-WARN] 阈值触发真机形态——待 TASK-0026 收尾后起 backend dev 补测（给 default 租户设临时 storage_quota_gb 触发一轮）。本轮遗留仅此一项。
2. **新发现 N1 待修（代码侧，两处一处小修）**：`period` 为 3.4.3.9 保留字——`HttpTdengineAdmin.STABLE_DDL[3]` 与 `OpenApiTimeseriesClient` 指标查询的裸 `period` 需改反引号 `` `period` ``。不修则：新租户 createDatabase 永远卡在第 4 张表（fail-soft 重试轮）、指标序列查询必 9728。**不阻塞 0029 点位写路径**（本期 st_metric_value 无写入方、点位两表逐字通过），**必须在指标链路/计算引擎联调（0068 口径）前落地**。iot_default 内 st_metric_value 已按反引号形态手工就位，default 租户不受此修复阻塞。
3. （登记口径不变）吞吐 type=throughput 告警数据源归 0029 ingest 交付接线；d 项补测时一并覆盖。
4. 上一轮遗留 1（三卡死库）/遗留 2（fd）/遗留 3（ot-admin 编译失败，TASK-0025 域）：分别随数据清空、容器 ulimit 重建、任务移交 TASK-0026 而闭环/移出本轮范围。

## 一句话总结

**TD-0053 可以闭合（挂 0068 的联调口径下）**：遗留 4 中 b/c/e/f/g 全部通过且带原始输出（fd 根治在存储统计、建库、DROP 全链正面证实），d 项为告警形态补测顺延（不阻塞）；唯一新增为 `period` 保留字小修登记（不阻塞 0029 写路径）；iot_default + 四张超级表已按判存自愈口径手工就位——**0029 写路径验收闸门解锁**（点位写路径的前置：fd 根治 ✓、scratch 建库闭环 ✓、iot_default 就绪 ✓、TAGS 往返 ✓）。
