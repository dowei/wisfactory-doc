# 微改批报告：TASK-0077 + TASK-0078（BUG-0004 修复 + 读超时放宽）

- BASE：`2e4a1b9`（开工时主仓 main HEAD，`git rev-parse --short HEAD`）
- 提交：`7f30568`（TASK-0077）、`f195b58`（TASK-0078），main
- 执行者：微改批 subagent（no-subagents，全程单执行者）；日期 2026-11-23
- 输入：批次 brief / BUG-0004 / TC-0025 / 涉及源码

---

## 一、改动文件清单

`git diff --name-only 2e4a1b9..HEAD`（共 7 个文件，全部落 brief「改动范围」声明内）：

| # | 文件 | 任务 | 改动 |
| --- | --- | --- | --- |
| 1 | `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdmin.java` | 0077 | STABLE_DDL[3] 的 period 列名 → `` `period` ``（附 BUG-0004 一行注释） |
| 2 | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/support/OpenApiTimeseriesClient.java` | 0077 | queryMetrics 指标查询 `AND period = '` → `` AND `period` = ' ``（附一行注释） |
| 3 | `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/TdengineProperties.java` | 0078 | `readTimeoutMs` 默认 `5000` → `30000`，javadoc 登记现场实测依据 |
| 4 | `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdminTest.java` | 0077 | L122 契约逐字断言同步为反引号形态（原断言裸 period，不更新则修复后必红） |
| 5 | `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/tdengine/TdengineStableDdlContractTest.java` | 0077 | 新增：BUG-0004 契约测试（st_metric_value 反引号锁死 + 其余三张表逐字不扩散） |
| 6 | `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/support/OpenApiTimeseriesClientTest.java` | 0077 | 新增用例 `queryMetricsPeriodColumnBackticked`（fake taosAdapter 回录 SQL 断言反引号 + 禁裸列名） |
| 7 | `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/tdengine/TdenginePropertiesDefaultsTest.java` | 0078 | 新增：readTimeoutMs 默认 30000 断言 + 其余默认值不受影响 |

生产代码净改动 3 处（+29/-5 含测试）；无重构、无其他域波及。范围核对原始输出见 §二.6。

## 二、统一验收原始输出

### 0. 前置惯例

```
$ cd backend && mvn install -pl ot-core -DskipTests -q
ot-core exit: 0
```

### 1. 先红（对 BASE 旧代码跑新增测试）

命令：`mvn test -pl ot-admin -Dtest='TdengineStableDdlContractTest,TdenginePropertiesDefaultsTest,OpenApiTimeseriesClientTest'`

```
[ERROR] Tests run: 11, Failures: 3, Errors: 0, Skipped: 0
[ERROR]   TdenginePropertiesDefaultsTest.readTimeoutDefaultIs30s:24
        ==> expected: <30000> but was: <5000>
[ERROR]   TdengineStableDdlContractTest.stMetricValueDdlPeriodBackticked:84
        ...TAGS (tenant_id BIGINT, metric_code BINARY(96), instance_code BINARY(96), period BINARY(8))
        ==> expected: <true> but was: <false>
[ERROR]   OpenApiTimeseriesClientTest.queryMetricsPeriodColumnBackticked:193
        ...SELECT ts, value, yoy, mom FROM iot_default.st_metric_value WHERE tenant_id = 1
           AND metric_code = 'M1' AND period = 'DAY' AND ts >= 0 AND ts <= 100000 ORDER BY ts ASC LIMIT 20001
        ==> expected: <true> but was: <false>
[INFO] BUILD FAILURE
```

3 处失败恰好为三个新契约断言；三个测试类的其余用例（含 `otherThreeStablesUnchanged`、既有 I-2/I-4 用例）全过。

### 2. 后绿（修复后重跑）

命令：`mvn test -pl ot-admin -Dtest='TdengineStableDdlContractTest,TdenginePropertiesDefaultsTest,OpenApiTimeseriesClientTest,HttpTdengineAdminTest'`

```
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.tdengine.HttpTdengineAdminTest
[INFO] Tests run: 2,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.tdengine.TdenginePropertiesDefaultsTest
[INFO] Tests run: 2,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.tdengine.TdengineStableDdlContractTest
[INFO] Tests run: 7,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.openapi.support.OpenApiTimeseriesClientTest
[INFO] Tests run: 21, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

### 3. 全量回归（统一验收第 1 条）

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
[INFO] Results:
[INFO]
[INFO] Tests run: 2249, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
[INFO] Total time:  01:23 min
[INFO] Finished at: 2026-11-23T01:04:45+08:00
```

（ot-admin 测试为 standalone mock 形态，全程未依赖 MySQL/Redis/backend 8098。）

### 4. 全仓同契约点扫描（BUG 加严 §13 条款 1）

扫描口径：全仓 Java 中 TDengine 面向 SQL（字符串常量）的裸 period 列名。先定位全部 TDengine 相关 Java 文件（`grep -rln "taos|TDengine|tdengine"`，45 个），对其中 9 个 SQL 生产文件逐一 grep（HttpTdengineAdmin / TdengineAdmin / TdengineProperties / OpenApiTimeseriesClient / PointCurveQueryClient / TdengineTagConditionRenderer / TenantDbProvisionServiceImpl / IotQuotaStatsServiceImpl / DataPermServiceImpl）：

**SQL 上下文命中恰好只有两处已知点**（修复前）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdmin.java:47:
    + " TAGS (tenant_id BIGINT, metric_code BINARY(96), instance_code BINARY(96), period BINARY(8))"
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/support/OpenApiTimeseriesClient.java:187:
    + " AND period = '" + period + "'"
```

其余 period 命中均非 TDengine SQL（列举待裁决项：**无**）：
- MySQL MyBatis `@Select`（`OpenApiMetricMapper.java:27` 查 `iot_metric` 表）——MySQL 无此保留字冲突，不属 TDengine 契约点；
- Java 标识符/方法参数/注释（IotMetric*、SnapshotMetric、calc_period、heartbeat_period、sample_period 等）；
- `@RateLimit(period = 60)` 注解参数（限流窗口秒数，与 SQL 无关）。

兜底侧写扫描：`st_metric_value|CREATE STABLE` 全仓（java/xml/sql/yml，含 resources/）仅命中 TdengineAdmin(javadoc)、HttpTdengineAdmin、OpenApiTimeseriesClient 及对应两个测试——无其他 TDengine DDL 藏匿点。

### 5. 真机回归（TC-0025，TDengine 192.168.31.145:6041 / 3.4.3.9.community / REST only / root:taosdata）

DDL 来源：**机械提取自修复后编译产物**——反射读取 `HttpTdengineAdmin.STABLE_DDL`（临时夹具 `target/ddlextract/ExtractDdl.java`，未入库），消除人工转写误差。提取结果（`STABLE_DDL[3]` 已是反引号形态，前三张逐字未动）：

```
STABLE_DDL[0] = CREATE STABLE IF NOT EXISTS st_point_num (ts TIMESTAMP, value DOUBLE, quality INT) TAGS (tenant_id BIGINT, point_code BINARY(96), kind BINARY(16), instance_key BINARY(96), dept_id BIGINT)
STABLE_DDL[1] = CREATE STABLE IF NOT EXISTS st_point_state (ts TIMESTAMP, value INT, quality INT) TAGS (tenant_id BIGINT, point_code BINARY(96), kind BINARY(16), instance_key BINARY(96), dept_id BIGINT)
STABLE_DDL[2] = CREATE STABLE IF NOT EXISTS st_item_value (ts TIMESTAMP, value DOUBLE, quality INT) TAGS (tenant_id BIGINT, full_key BINARY(180), instance_code BINARY(96), data_code BINARY(64), category BINARY(32), source_type INT)
STABLE_DDL[3] = CREATE STABLE IF NOT EXISTS st_metric_value (ts TIMESTAMP, value DOUBLE, yoy DOUBLE, mom DOUBLE) TAGS (tenant_id BIGINT, metric_code BINARY(96), instance_code BINARY(96), `period` BINARY(8))
```

scratch 库：`wisfactory_verify_p7`（brief 批次口径）。初始态 `SHOW DATABASES` = `information_schema / performance_schema / iot_default / log`（无 scratch，干净起点）；探活 `SELECT SERVER_VERSION()` → `3.4.3.9.community`。

**步骤 1/0 建库**（已知慢特性，`curl -m 60`，实测 15.2s）：

```
[01 CREATE DATABASE IF NOT EXISTS wisfactory_verify_p7]
{"code":0,...,"data":[[0]]}          real 0m15.228s
```

**步骤 2 复现锚点（裸 period 建表）→ 9728**：

```
{"code":9728,"desc":"syntax error near \"period binary(8))\"","timing":4089500}
```

**步骤 3 反引号建表 → code 0，表在位**：

```
{"code":0,...,"data":[[0]]}
[SHOW STABLES] → data: [["st_metric_value"]], rows: 1
```

**步骤 4 指标查询两形态**：

```
[4a 裸 period] → {"code":9728,"desc":"syntax error near \"period = 'DAY' and ts >= 0 ...\""}
[4b `period`]  → {"code":0,"column_meta":[["ts","TIMESTAMP",8],["value","DOUBLE",8],["yoy","DOUBLE",8],["mom","DOUBLE",8]],"data":[],"rows":0}
```

（4b 空表空集但 code 0 且 column_meta 四列正确 = 语法通过；行形态断言由单测承载。）

**步骤 5 修复回归：提取的四张 DDL 逐张执行 → 全部 code 0，四张在位**：

```
[st_point_num]      {"code":0,...}
[st_point_state]    {"code":0,...}
[st_item_value]     {"code":0,...}
[st_metric_value]   {"code":0,...}
[SHOW STABLES] → data: [["st_point_state"],["st_point_num"],["st_item_value"],["st_metric_value"]], rows: 4
```

**清理证明（终态）**：

```
[DROP STABLE ×4] 各 {"code":0,...}
[SHOW STABLES]   {"code":0,...,"data":[],"rows":0}
[DROP DATABASE]  {"code":0,...}
[SHOW DATABASES 终态] → [["information_schema"],["performance_schema"],["iot_default"],["log"]], rows: 4
```

终态与初始态一致，scratch 库零残留。

### 6. 改动范围核对（统一验收第 3 条）

```
$ git diff --name-only 2e4a1b9..HEAD
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdmin.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/TdengineProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/support/OpenApiTimeseriesClient.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/tdengine/HttpTdengineAdminTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/tdengine/TdenginePropertiesDefaultsTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/tdengine/TdengineStableDdlContractTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/support/OpenApiTimeseriesClientTest.java
```

7 个文件全落 brief 声明的 `tdengine/**` + `openapi/**` + `src/test/java/**` 三条之内；未碰其他域、未碰 SQL 升级脚本。工作区仅剩 BASE 时已存在的未跟踪目录 `.agents/`（本批未动）。

## 三、未做项与原因

| 未做项 | 原因 |
| --- | --- |
| 按调用类型拆分超时（建库长超时/常规查询短超时） | brief 明确「不做，超出本微改范围，如需另立任务」；已在 TdengineProperties javadoc 登记该边界 |
| TASK-0077/0078 置「已完成」、BUG-0004 填验证结果、TC-0025 状态流转 | 契约禁止执行者改 workitems 状态（报告除外），归协调者收尾 |
| workitems 库其他文件同步 | 同上，未触碰 |
| OpenApiTimeseriesClient 类 javadoc 中「period 标签」字样未加反引号标注 | 纯文档描述非 SQL 契约点；守「唯一最小偏离形态」不扩散改动 |

## 四、偏离计划之处

1. **openapi 文件实际路径**：brief 改动范围写 `net/otlab/admin/module/iot/openapi/**`，实际 `OpenApiTimeseriesClient` 位于 `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/support/`。brief 任务描述明确点名该文件为修复目标，按文件名执行；路径前缀与仓库实际结构存在出入，特此登记（无越界——文件即声明目标本身）。
2. **scratch 库名**：TC-0025 前置条件文本写 `wisfactory_verify_period`，批次 brief 统一口径为 `wisfactory_verify_p7`，按 brief 执行。
3. **契约测试的既有断言更新**：`HttpTdengineAdminTest` L122 原有「逐字附录 B」断言含裸 period，修复后必然转红，随 0077 同步为反引号形态（属测试文件范围，非生产改动）。
4. **DDL 提取方式**：brief 要求「从修复后代码提取」，采用反射读私有常量的临时夹具（`target/` 下，未提交）替代人工转写，更强地保证提取忠实性；无生产行为影响。

## 五、遗留问题

1. **契约文档同步**：根因提到「上游附录 DDL 沿用裸列名」——若冻结契约文档（附录 B）文本仍为裸 period，需文档侧同步为反引号形态（docs 域不在本批范围，未核查，待裁决/另立）。
2. **现场已补建表与代码形态一致性复核**：现场手工补建的 st_metric_value 已按反引号形态（TD-0053-verify-report2），与本次修复形态一致；最终一致性归 TASK-0068 真机联调复核。
3. **TDengine 3.4.3.9 保留字全集未逐一排查**：本次按契约点只处理 period；四表 DDL 与指标查询真机全 code 0 已证明当前 SQL 面无其他保留字冲突，但后续版本升级若再暴露新保留字，建议另立扫描任务（可复用本批 TdengineStableDdlContractTest 形态）。

---

## 结论

**通过**——两处裸 period 在真机 3.4.3.9 复现 9728（步骤 2/4a），修复后从代码机械提取的四张超级表 DDL 真机逐张 code 0 且四表在位、清理归零（终态 SHOW DATABASES 与初始一致），指标查询反引号形态 code 0；契约测试先红后绿留痕，全仓扫描证明裸 period 命中仅两处已修；全量 `mvn test -pl ot-admin` 2249 例 0 失败 0 错误 BUILD SUCCESS；`readTimeoutMs` 默认 30000 断言绿（红时 5000 留痕），改动 7 文件全落 brief 声明范围。
