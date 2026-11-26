# TASK-0063 报告：iot_calc_state 计算状态独立 schema 与只读账号

- 执行者：TASK-0063 执行者（no-subagents；calc/** + resources/sql/upgrade SQL + backend/ot-admin 只读端点）
- **BASE（开工 HEAD）= `ea34d02`**（`git rev-parse --short HEAD`，2026-11-24 main）；**交付提交 = `fb8f66d`**（注明 TASK-0063，18 文件全落声明范围，`git add` 逐路径、绝不 `git add -A`）
- 执行日期：2026-11-24
- 结论先行：**验收 3 条全部通过**——① `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` 2457 测试 0F0E0Skip BUILD SUCCESS（含新增 calc 只读 8 项 + 全量守卫零回归）；`cd calc && mvn test` 288 测试 0F0E3Skip BUILD SUCCESS（275 基线 + 13 新增）；② DDL 脚本静态结构检查全过（幂等/schema/两表字段索引/账号分离/无审计四件套）+ 应用步骤已写明；③ 假 DataSource 契约测试（H2 真 JDBC + fail-soft）通过，dev 未起服务（DDL 未在用户库执行，按契约改走假 DataSource）。
- **DDL 未执行声明**：**本任务未在任何用户 MySQL（宿主 phpStudy 8.0.12@3307 等）执行本 DDL 或任何写库语句**；脚本只产出由用户自行按报告步骤应用。契约测试全部走 H2 内存库 / 假 DataSource，不连宿主库。
- 并行事实：并行 0053 修订轮提交 `e9cc251`（TASK-0053，openapi/subscribe 7 文件）夹在 BASE 与交付提交之间，与本任务零文件交集；本任务未碰其源码/报告。8098 全程未启动（dev 冒烟按契约改假 DataSource 契约测试，无端口占用）；契约文件零触碰；workitems 库仅本报告变动。

---

## 一、改动文件清单（提交 `fb8f66d`，18 项）

### 1.1 calc/** — CalcStateStore JDBC 实现（0047 注入缝替换，消费侧零改动）（8）

| 文件 | 职责 |
| --- | --- |
| `calc/src/main/java/net/otlab/calc/state/JdbcCalcStateStore.java`（新） | CalcStateStore JDBC 实现：JdbcTemplate + `iot_calc_state` 表（§1.9A 8 字段），upsert 先 UPDATE 再 INSERT（跨 MySQL/H2 通用，唯一键 `uk_tenant_full_key` 覆盖不追加）；get/byTenant/all 全部 SELECT；连接/表/权限异常 **fail-soft WARN 降级**（get→null、列表→空集、upsert 空操作），不阻塞在线计算。 |
| `calc/src/main/java/net/otlab/calc/CalcEngineApplication.java`（改） | `calcStateStore(CalcProperties)` Bean：默认 InMemoryCalcStateStore（零回归）；`CALC_STATE_JDBC_ENABLED=true` + URL 时改装配 JdbcCalcStateStore（计算引擎**读写账号**，URL/账号/口令 env 注入不硬编码；缺 URL fail-fast）。消费侧（管理器/状态机/VO/控制器）零改动。 |
| `calc/src/main/java/net/otlab/calc/config/CalcProperties.java`（改） | 新增 4 项 env：`calcStateJdbcEnabled`（默认 false）/ `calcStateJdbcUrl` / `calcStateJdbcUser` / `calcStateJdbcPassword` + getter/setter。 |
| `calc/src/main/resources/application.yml`（改） | 4 项 env 映射：`CALC_STATE_JDBC_ENABLED/URL/USER/PASSWORD`。 |
| `calc/pom.xml`（改） | 增 `spring-boot-starter-jdbc`（JdbcTemplate）、`mysql-connector-j`（runtime）、`h2`（test，JDBC 契约测试，不连宿主库）。 |
| `calc/src/test/java/net/otlab/calc/state/IotCalcSchemaDdlContractTest.java`（新） | DDL 脚本静态结构契约 5 项：幂等/schema/ADR-0009 Q1 头注/state 全字段+无审计四件套/log 全字段+三索引/账号分离+口令占位。只读文件，不连库。 |
| `calc/src/test/java/net/otlab/calc/state/JdbcCalcStateStoreContractTest.java`（新） | 7 项：`@TestFactory` 同套契约跑内存实现 + **真 JDBC 实现（H2 内存库模拟 iot_calc_state DDL）**（upsert/get/租户隔离/同 uk 覆盖/水位状态错误往返/byTenant）+ 1 项连接失败 fail-soft（不抛异常、WARN 降级）。 |
| `calc/src/test/java/net/otlab/calc/config/CalcPropertiesTest.java`（改） | +1 项 TASK-0063 JDBC 默认关闭/空 URL/空账号零回归断言。 |

### 1.2 resources/sql/upgrade — 独立 schema DDL（只产出，不执行）（1）

| 文件 | 职责 |
| --- | --- |
| `resources/sql/upgrade/20261124_iot_calc_schema.sql`（新） | ①`CREATE SCHEMA IF NOT EXISTS iot_calc`；②`iot_calc_state`（§1.9A 8 列 + uk_tenant_full_key + update_time ON UPDATE，**不套审计四件套**）；③`iot_calc_log`（§1.9A 14 列 + idx_task/idx_key_period/idx_trigger_time）；④⑤ 账号分离：`iot_calc_rw`（rw 账号 SELECT/INSERT/UPDATE）+ `iot_calc_ro`（仅 SELECT），information_schema 判存幂等 CREATE USER，口令 `${IOT_CALC_RW_PASSWORD}` / `${IOT_CALC_RO_PASSWORD}` 占位由部署注入；头注登记 ADR-0009 Q1 定版依据 + 不执行声明 + 执行后核对语句。 |

### 1.3 backend/ot-admin/** — 管理系统只读端点（9）

| 文件 | 职责 |
| --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/config/CalcReadDataSourceConfig.java`（新） | 管理系统专用只读 DataSource + JdbcTemplate Bean `calcReadJdbcTemplate`（iot_calc_ro 账号，env `OT_CALC_READ_JDBC_*`）；**独立于 @Primary 动态数据源（业务库 master）**，不写穿；未配置 URL 时装「取连接即抛未配置」占位源，只读查询 fail-soft 空集、不阻塞启动、不依赖 test-scope 驱动、不连宿主库。 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/CalcStateVO.java`（新） | 计算状态只读视图（§1.9A 字段 + INT→文本翻译 statusText，ADR-0009 TD-0005）。 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/CalcLogVO.java`（新） | 计算日志只读视图（§1.9A 全字段，is_recalc→Boolean）。 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/CalcReadService.java`（新） | 只读服务接口：state(fullKey) / log(fullKey,from,to)。 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/impl/CalcReadServiceImpl.java`（新） | 只读实现：**全 SELECT 零写口**；会话租户行过滤、fullKey 过滤、INT→文本、is_recalc 映射、时间窗检索；数据权限照实例域范式（fullKey 第 3 段 instanceCode → instanceId → VisibleScope 可见集，越权不返回）；iot_calc 不可读 fail-soft 空集 WARN；核心读构造可直击单测（deps null 跳过权限）。 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/controller/IotCalcReadController.java`（新） | `GET /api/v1/iot/calc/state?fullKey=` + `GET /api/v1/iot/calc/log?fullKey=&from=&to=` 两只读端点；权限沿用已注册管理面只读位 `iot:instance:query`（0064 无 calc 位，理由见 §四）；读操作不挂 @OperateLog（读不审，0079 范式）；无写口（状态页只读，重算走 0061）。 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/calcread/controller/IotCalcReadControllerTest.java`（新） | 3 项：参数透传/空态、坏区间 403/400 传递、两 GET 权限注解锁定 + 零写口机械断言。 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/calcread/impl/CalcReadServiceImplTest.java`（新） | 5 项：H2 建 iot_calc_state/log 同结构表，会话租户行过滤/fullKey 过滤/INT→文本、空态 fail-soft、时间窗 + is_recalc 映射、坏参拒绝、instanceCode 提取。 |
| `backend/ot-admin/pom.xml`（改） | +`spring-boot-starter-jdbc`（JdbcTemplate）、`h2`（test）。 |

> 范围核对：`git show --name-only --format="" fb8f66d` = 18 项全落 `calc/**` + `resources/sql/upgrade/20261124_iot_calc_schema.sql` + `backend/ot-admin/pom.xml` + `backend/ot-admin/src/**/calcread/**`；越界 0。BASE..HEAD 区间多出的 `openapi/subscribe/**` 7 项属并行 0053 修订轮 `e9cc251`，与本任务零文件交集（评审按 `fb8f66d` 判断归属）。

---

## 二、验收命令原始输出

### 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 0 失败 0 错误

ot-core 安装：`CORE_INSTALL_EXIT=0`。ot-admin 全量终验 Results 段（CR 尾已去）：

```
[INFO] Running net.otlab.admin.module.iot.calcread.controller.IotCalcReadControllerTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.541 s -- in net.otlab.admin.module.iot.calcread.controller.IotCalcReadControllerTest
[INFO] Running net.otlab.admin.module.iot.calcread.impl.CalcReadServiceImplTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.565 s -- in net.otlab.admin.module.iot.calcread.impl.CalcReadServiceImplTest
...（其余既有类逐类 0F0E，含 IotPermissionsCompleteGuardTest / OperateLogCoverageGuardTest 零回归）...
[INFO] Results:
[INFO]
[INFO] Tests run: 2457, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
EXIT=0
```

### 验收 1（续）：`cd calc && mvn test` → 0 失败 0 错误

终验（交付提交 `fb8f66d` 对应源码）Results 段：

```
[INFO] Running net.otlab.calc.state.IotCalcSchemaDdlContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.020 s -- in net.otlab.calc.state.IotCalcSchemaDdlContractTest
[INFO] Running net.otlab.calc.state.JdbcCalcStateStoreContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 6.255 s -- in net.otlab.calc.state.JdbcCalcStateStoreContractTest
...（其余既有类逐类 0F0E，275 基线零回归）...
[INFO] Results:
[INFO]
[INFO] Tests run: 288, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  16.543 s
EXIT=0
```

275 基线逐类零回归：0047 修订轮终验 275 类计数逐字一致（ack 8 / app 1 / config 12〔本任务 +1〕/ event 9 / executor 4+5+10+4+2+1skip+2 / graph 7+17+23 / metric cascade 2+late 6+agg 8+companion 6+sched 4+window 7+writer 5 / operator 6+5+4 / runtime 11 / snapshot 13+9+20 / status 8+2skip+8+5 / storage 5+7 / state 6+4+4+5 / task 7+3 / web 1）；3 Skipped 仍为 0041 env 门控 1 + 0049 env 门控 2。新增 13 = DDL 静态 5 + JDBC 契约 7 + 配置默认 1（288 = 275 + 13）。

### 验收 2：DDL 脚本静态检查（只产出不执行）

```
=== DDL 静态检查（仅读文件，不连任何数据库） ===
-- 文件存在 -- EXISTS: resources/sql/upgrade/20261124_iot_calc_schema.sql
-- schema 幂等 -- 1
-- 表幂等 -- 2
-- state 无审计四件套（state 建表块内） --
  create_time: absent(OK)   create_by: absent(OK)   update_by: absent(OK)
  `deleted`: absent(OK)     `version`: absent(OK)
  state 字段: ['tenant_id','full_key','last_calc_time','watermark','calc_status','error_count','last_error','update_time']
  uk_tenant_full_key: True
-- 账号分离 -- GRANT SELECT, INSERT, UPDATE ON `iot_calc`.* TO 'iot_calc_rw'@'%';
              GRANT SELECT ON `iot_calc`.* TO 'iot_calc_ro'@'%';
-- 口令占位 -- 3（${IOT_CALC_RW_PASSWORD}/${IOT_CALC_RO_PASSWORD}）
-- ADR-0009 Q1 头注 -- 1
DDL_NOT_EXECUTED=true
```

**DDL 应用步骤（交付用户按环境自行执行 —— Agent 未执行）**：

1. 部署方注入口令环境变量：`IOT_CALC_RW_PASSWORD`（计算引擎读写）、`IOT_CALC_RO_PASSWORD`（管理系统只读），替换脚本中两个占位符（或由部署工具注入）。
2. 按 `backend/scripts/sql-exec.sh` **每条语句单独一次调用**执行 `resources/sql/upgrade/20261124_iot_calc_schema.sql`（phpStudy MySQL 8.0.12@3307，root 连接串由脚本外注入）。
3. 执行后核对：`SELECT SCHEMA_NAME FROM information_schema.SCHEMATA WHERE SCHEMA_NAME='iot_calc';`（应 1 行）；`SHOW TABLES FROM iot_calc;`（应 iot_calc_state / iot_calc_log）；`SHOW GRANTS FOR 'iot_calc_rw'@'%'`（SELECT/INSERT/UPDATE ON iot_calc.*）；`SHOW GRANTS FOR 'iot_calc_ro'@'%'`（仅 SELECT ON iot_calc.*）。
4. 应用后启用 calc 侧 `CALC_STATE_JDBC_ENABLED=true` + `CALC_STATE_JDBC_URL=jdbc:mysql://host:3307/iot_calc` + `CALC_STATE_JDBC_USER=iot_calc_rw` + `CALC_STATE_JDBC_PASSWORD=<注入>`；管理系统侧 `OT_CALC_READ_JDBC_URL=jdbc:mysql://host:3307/iot_calc` + `OT_CALC_READ_JDBC_USER=iot_calc_ro` + `OT_CALC_READ_JDBC_PASSWORD=<注入>`。
5. 真机复验：起 calc（8095 探测 FREE 后）、`recalculate` 提交→`status` 断言水位/calc_status→JdbcCalcStateStore 契约对真库复跑 upsert/租户隔离/字段映射；管理系统 `GET /api/v1/iot/calc/state|log` 只读探针；用毕即停、端口回 FREE。

### 验收 3：假 DataSource 契约测试（DDL 未在用户库执行 → 不做起库 dev 冒烟）

```
$ mvn -q -Dtest=JdbcCalcStateStoreContractTest test   # calc（H2 内存库，不连宿主）
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0
06:52:09 WARN [CALC-STATE-JDBC] get 降级（tenant=1, fullKey=T.A）——在线计算不阻塞: Failed to obtain JDBC Connection
06:52:10 WARN [CALC-STATE-JDBC] all 降级 ... / byTenant 降级 ... / upsert 降级 ... ——在线计算不阻塞
EXIT=0

$ mvn -q -Dtest=IotCalcReadControllerTest,CalcReadServiceImplTest test -pl ot-admin  # H2 内存库
[INFO] Tests run: 3 / 5, Failures: 0, Errors: 0, Skipped: 0
EXIT=0
```

> dev 冒烟未起 8098：DDL 由用户自行应用（本任务未执行），真库探针条件未满足；按 brief「否则假 DataSource 契约测试」口径执行，8098 全程未启动、无端口占用。

### TDD 红转绿留痕（每项附「为什么现在对了」）

1. **DDL 静态契约 RED→GREEN**：先写 `IotCalcSchemaDdlContractTest` → `mvn test` `[ERROR] Tests run: 5, Failures: 5`（5 项全报「缺 DDL 脚本」）；产脚本后 4 项绿、1 项 `idempotent` 红——原断言用 `CREATE USER 后首字符` 启发式误判包在 `information_schema` 判存动态 SQL 中的 `CREATE USER` 为裸建。**修**：断言改为「逐行扫描裸 `CREATE USER` + 必须有 `information_schema.USER_PRIVILEGES` 判存」——为什么现在对了：账号幂等真实机制是 `information_schema` 判存 + 预处理动态 SQL，测真机制而非字符巧合，脚本保持 MySQL 跨版本幂等；GREEN 5/5。
2. **JDBC 契约 GREEN 实证**：`@TestFactory` 同套契约在内存实现 + `JdbcCalcStateStore`（H2 真 JDBC，建同结构表）各跑 3 条 + fail-soft 1 条 = 7 项全绿——为什么现在对了：0047 注入缝 `CalcStateStore` 接口的真 JDBC 实现通过 upsert/get/租户隔离/uk 覆盖/字段往返全部断言，消费侧一行未改；连接失败 WARN 降级不抛异常，brief「fail-soft 不阻塞在线计算」成立。
3. **backend 只读端点 RED→GREEN（3 处真实修复）**：① 首轮编译 `[ERROR] 需要 class/interface`（`CalcReadDataSourceConfig` 多余右花括号，早期编辑残留）→ 删除；② `@SaCheckPermission.value()` 断言 `expected <iot:instance:query> but was <[iot:instance:query]>` → 改 `value()[0]`——为什么现在对了：sa-token 注解 value 是数组，锁定单元素语义；③ `SaTokenContextException: 上下文尚未初始化` → 照既有 `SysTenantInitAdminTest` 用 `SaTokenContextMockUtil.setMockContext()/clearContext()`（为什么： sa-token 单测需显式 mock 上下文，非代码缺陷）；④ H2 `BIGINT to TIMESTAMP` 转换失败 → 测试插入改 `new java.sql.Timestamp(ms)`（为什么：真实 DDL 列是 datetime，测试夹具须用时间类型，服务读映射正确）。修完 8/8 绿。
4. **环境类问题（非代码回归）**：并行 java JVM 锁 `target/classes` 导致首次 `mvn clean` 失败、后续全量跑出 454 `NoClassDefFound`（target 被部分清空）→ 重跑 `mvn clean compile -pl ot-admin`（694 类恢复）后全量 2457 绿；本项属构建目录状态，已恢复，不阻断验收。

---

## 三、未做项与原因

1. **DDL 未在用户库执行（契约硬约束）**：本任务只产出 `20261124_iot_calc_schema.sql`，**未在宿主 phpStudy 8.0.12@3307 或任何库执行该 DDL / 任何写库语句**；契约测试全部 H2/假 DataSource。应用步骤见 §二验收 2。真机验证条件：用户按步骤应用 DDL + 注入口令 + 配置 env 后，按 §二验收 2 步骤 5 复验。
2. **真库 JDBC 冒烟（依赖 DDL 已应用）**：未做。原因同上；已用 H2 真 JDBC 契约测试 + fail-soft 测试覆盖同套语义（upsert/租户隔离/字段映射/连接失败降级），真机列条件在 §二写明。
3. **管理面重算/写口（0061）**：本端点只读，状态页操作走重算接口归 0061 管理面转发，本任务不触发任何 calc 写（TASK 技术方案 + brief 决策 3）。
4. **iot_calc_log 物理写（calc 侧）**：0047 交付 CalcLogStore 内存留痕 + 0046 task 层 MANUAL/BACKFILL 缝；本任务 DDL 建表 + 管理读口就绪，calc 侧物理追加写归演进位（0047 §三-2 已登记），本任务未改 calc 内部计算语义。
5. **dev 8098 冒烟**：未起（DDL 未应用、真库探针条件未满足）；按 brief 用假 DataSource 契约测试替代，8098 探测/启动均不涉及。
6. **iot:calc:query 专用权限位**：0064 高危清单口径无该位（重算为内部令牌接口无 perms 位），未新建种子（避免越界改 0064 已有种子与守卫测试）；沿用已注册 `iot:instance:query`，详见 §四。
7. **TASK 状态**：按契约未把 TASK-0063 置「已完成」，保持「进行中」待评审；未改 workitems 库（唯一例外：本报告）。

---

## 四、偏离计划之处

1. **权限码沿用 `iot:instance:query` 而非 `iot:calc:query`**：brief 要求「复用 iot:calc:query 或最贴近既有权限码，若无则沿用管理面只读权限」；核查 0064 差集终态无 `iot:calc:*` 种子（重算 = 内部令牌接口无 perms 位），新建位需改 0064 种子 + 守卫测试（越出本任务 backend/ot-admin/src/main 声明范围且触碰他人交付面）。**取向**：沿用已注册的实例只读位 `iot:instance:query`（4031），与 fullKey 实例维度数据权限裁剪同一口径；javadoc 登记后续 0061 建 calc 位时可平移注解、业务零改。守卫测试全量通过（注解 ⊆ 种子集），零 403 窗口。
2. **只读 DataSource 用独立 JdbcTemplate Bean 而非 DynamicDataSource 路由**：项目有 `@Primary` 动态数据源（业务库 master），若只读端点走它会用业务账号、不满足「独立只读账号」。**取向**：`CalcReadDataSourceConfig` 自建独立 DataSource（iot_calc_ro env 注入）+ JdbcTemplate Bean，不挂 `@DataSource` 注解、不进动态路由——为什么现在对了：物理账号隔离 + 代码 SELECT-only 双保险，不写穿业务库，全量 Spring 上下文测试零回归。
3. **未配置只读源时装配「取连接即抛」占位源而非 H2**：H2 是 test-scope，生产引用会 ClassNotFound。**取向**：占位源任何 getConnection 抛「未配置」，服务层 fail-soft 空集——为什么现在对了：不依赖测试驱动、不连宿主库、不阻塞启动，与 calc 侧 JDBC 关闭时内存零回归同口径；测试用 H2 单独注入 JdbcTemplate 直击核心读。
4. **upsert 采用 UPDATE-then-INSERT 而非 ON DUPLICATE KEY UPDATE**：MySQL 与 H2 对该方言支持差异。**取向**：先 UPDATE，影响行数 0 再 INSERT，唯一键 `uk_tenant_full_key` 保证并发下覆盖语义——为什么现在对了：跨库通用、契约测试在同 uk 覆盖场景断言 `all().size()==1`（不追加行），真库复验同断言可直接复用。
5. **DDL 账号 CREATE 用 information_schema 判存 + 预处理动态 SQL**：MySQL 8.0 无统一 `CREATE USER IF NOT EXISTS` 跨版本口径。**取向**：`SET @exists` 判存 + `PREPARE/EXECUTE` 动态建号——为什么现在对了：脚本幂等可重跑，静态契约断言真幂等机制；口令占位不硬编码。
6. **dev 冒烟改假 DataSource 契约测试**：brief 验收 3 二选一（DDL 已应用→真库探针，否则→假 DataSource）；选后者，未起 8098——为什么现在对了：DDL 未执行是硬约束，起服务也无可查真表；H2 真 JDBC + fail-soft 契约已覆盖同语义。

---

## 五、遗留问题

1. **真机 DDL 应用 + 真库复验（用户侧动作）**：按 §二验收 2 步骤 1-5 由用户执行；本任务不执行、未验证真 MySQL 表/账号实态。
2. **calc 侧 iot_calc_log 物理写（演进位）**：DDL + 读口就绪，追加写归后续任务（0047 §三-2 登记）。
3. **管理面重算写口 / 专用 calc 权限位（0061）**：本任务只读端点就绪，重算转发 + `iot:calc:*` 种子归 0061；本端点注解届时平移。
4. **F-04/F-05（0047 修订轮 deferred Minor）**：拒绝计数翻倍 / topoOrder 环回退 fail-open 未在本任务范围（calc 内部语义），仍 deferred。
5. **多实例一致性**：JdbcCalcStateStore 每次调用独立连接（无本地缓存），多实例下由 MySQL 唯一键保证 uk 覆盖；CalcTaskManager 进程内任务队列为单实例口径（0047 已登记演进位），本任务未改。
6. **并行提交区间**：`ea34d02..fb8f66d` 含 0053 修订轮 `e9cc251`（7 项 openapi/subscribe），与本任务零交集；评审按 `fb8f66d` 判断归属。
7. **TASK 状态**：保持「进行中」待评审，未置「已完成」；workitems 库仅本报告变动。

---

## 一句话结论

**通过**——验收 1（backend 全量 2457 测试 0F0E0Skip BUILD SUCCESS，含 calc 只读 8 项 + IotPermissionsCompleteGuard/OperateLogCoverage 零回归；calc 288 测试 0F0E3Skip BUILD SUCCESS，275 基线零回归 + 13 新增，H2 真 JDBC 契约 + fail-soft WARN 实证）、验收 2（DDL 静态检查全过：幂等/schema/两表字段索引/账号分离/无审计四件套；应用步骤与 DDL 未执行声明已写明）、验收 3（假 DataSource 契约测试通过；DDL 未在用户库执行故未起 8098 真库冒烟，按 brief 二选一口径合规）三条全部以原始输出为证通过；交付提交 `fb8f66d` 18 文件全落声明范围、越界 0；TASK 保持「进行中」待评审。

---
---

# 修订轮（2026-11-24）：diff 评审退回 1 Important + 1 Minor

- 触发：`TASK-0063-review.md` 判定不通过——**I-1（Important）** DDL 建号段落 `PREPARE/EXECUTE/DEALLOCATE` 三条写在同一行，与脚本声明的执行通道 `sql-exec.sh`（按 `;\n` 切分）不兼容，只读探针实测 `SQLERROR near 'PREPARE...'`（`:64`/`:73`）；**m-1（Minor）** `CalcReadServiceImpl` javadoc 称单段 fullKey「仅同租户可见」但 `visible()` 对无实例段直接 `false` 全裁，与 state 侧真实单段 metricCode 场景不符且该路径无端到端测试。
- 修订范围 = 原范围子集（`resources/sql/upgrade/20261124_iot_calc_schema.sql` + `backend/ot-admin/src/**/calcread/**`）；**calc 侧零改动**。修订 BASE = `fb8f66d`（被评审提交）；**修订提交 = `640c9bd`**（注明 TASK-0063 修订轮，4 文件，全落声明范围、越界 0）。
- 结论先行：**I-1 真修复 + m-1 顺手修**——TDD 红转绿（DDL 单行守卫对旧脚本必红→修脚本转绿；数据权限端到端 2 项新测锁 fail-closed 行为）；sql-exec.sh 真实通道只读探针实证切分正确；全量 **backend 2461 测试 0F0E0Skip**（2457 基线 + 4 新增）、**calc 288 测试 0F0E3Skip**（基线不变）BUILD SUCCESS；DDL 仍未在用户库执行。

## 修订轮一、改动文件清单（提交 `640c9bd`，4 项）

| 文件 | 修订内容 |
| --- | --- |
| `resources/sql/upgrade/20261124_iot_calc_schema.sql` | I-1：`:64`/`:73` 的 `PREPARE rw_stmt...; EXECUTE...; DEALLOCATE...;`（及 ro 同形）由单行拆为各占一行；头注第 5 条补「按 `;\n` 切分、每条语句必须独占一行」说明。全文核对：无非注释行含同行多语句、所有语句行以 `;` 结尾。 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/calcread/impl/IotCalcDdlSplitContractTest.java`（新） | I-1 守卫 2 项：复刻 `sql-exec.sh` 的 `sql.split(";\\s*\\n")` + 剥注释逻辑，断言切分后任一段不含残余分号（覆盖脚本内**所有**语句，不漏同类）；建号段落 PREPARE/EXECUTE/DEALLOCATE 六语句各独占一行（F-1 锚点）。 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/calcread/impl/CalcReadServiceImpl.java` | m-1：javadoc「数据权限」段订正——如实说明单段 fullKey（state 侧 metricCode）无实例段时**仅 allTenant 可见、受限角色 fail-closed 裁掉**（原写「仅同租户可见」与实现不符）；**不改裁剪逻辑**（方向正确）。 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/calcread/impl/CalcReadDataPermContractTest.java`（新） | m-1 端到端 2 项：走真实 `visible()` 路径（mock DataPermService + OpenApiInstanceMapper，非 deps=null 捷径）——受限角色单段 fullKey 与越权实例行被裁、仅可见实例 I01 行通过；allTenant 角色单段 fullKey 可见（3 行全通过）。 |

## 修订轮二、TDD 红→绿证据（每项附「为什么现在对了」）

**I-1 RED（对旧脚本必红）**：

```
$ mvn -q -Dtest=IotCalcDdlSplitContractTest test -pl ot-admin   # 修订 BASE fb8f66d 的旧脚本
[ERROR] Tests run: 2, Failures: 2, Errors: 0, Skipped: 0
everyStatementIsOnItsOwnLine: 以下切分段仍含同行分号:
  [PREPARE rw_stmt FROM @rw_create; EXECUTE rw_stmt; DEALLOCATE PREPARE rw_stmt,
   PREPARE ro_stmt FROM @ro_create; EXECUTE ro_stmt; DEALLOCATE PREPARE ro_stmt]
accountStatementsEachOnOwnLine: 建号语句须独占一行: EXECUTE rw_stmt → expected <1> but was <0>
EXIT=1
```

- 为什么红：旧脚本 `:64`/`:73` 按 `sql-exec.sh` 切分后整行是一段，`;\\s*\\n` 只在行尾分号处切，同行内 `;` 全部粘在一条语句里——正是评审 F-1 的可应用性缺陷。

**I-1 GREEN**：

```
$ 拆行后 mvn -q -Dtest=IotCalcDdlSplitContractTest test -pl ot-admin
EXIT=0   # 2/2 通过
$ grep 静态核对（非注释行同行多语句）→ 空；行尾无 ; 的非注释行 → NONE
```

- 为什么现在对了：PREPARE/EXECUTE/DEALLOCATE 各自以 `;\n` 结尾，`sql-exec.sh` 切分后每段恰一条语句，JDBC `Statement.execute` 不再收到多条语句字符串；守卫复刻真实切分算法并对全脚本断言，任何同类「同行多语句」回归都会被捕获。

**m-1 GREEN（端到端锁定 fail-closed 行为）**：

```
$ mvn -Dtest=CalcReadDataPermContractTest test -pl ot-admin
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- in ...CalcReadDataPermContractTest
[INFO] BUILD SUCCESS
restrictedRoleClipsSingleSegmentAndOutOfScopeInstance → 行集=[T|M.B|I01|day]（单段 M.A 裁掉、越权 I99 裁掉）
allTenantRoleSeesSingleSegmentFullKey → 行集 3 行（含单段 M.A）
```

- 为什么现在对了：测试经 `new CalcReadServiceImpl(jdbc, instanceMapper, dataPermService)` 走真实 `visible()`（区别于原核心读测试的 deps=null 捷径），断言与订正后 javadoc 完全一致——受限角色单段 fullKey fail-closed（不泄露）、allTenant 可见；裁剪逻辑未变，安全方向不动，仅文档与代码对齐并补证据。

## 修订轮三、sql-exec.sh 真实通道只读探针（切分实证，无 DDL/无对象创建）

```
$ bash backend/scripts/sql-exec.sh "$(cat /tmp/t0063-probe/split_probe.sql)" localhost 3307 root root ot_wisfactory
（探针与修订后建号段落同结构：SET @exists=(information_schema 判存);
  SET @create=IF(...); PREPARE probe_stmt FROM @create; EXECUTE probe_stmt;
  DEALLOCATE PREPARE probe_stmt; SELECT 'SPLIT_PROBE_DONE'; 每行一条语句）
OK(0)
OK(0)
OK(0)
probe_ok
1
OK(0)
marker
SPLIT_PROBE_DONE
DONE
PROBE_EXIT=0
```

- 探针全部为只读：`SELECT ... information_schema` + `SELECT` 常量，**未执行任何 DDL、未创建/修改任何对象**，宿主用户库未被写入。对照评审 F-1 的单行形态 `SQLERROR near 'PREPARE...'`，拆行形态 `OK/OK/OK/1/OK/DONE` 实证 `sql-exec.sh` 切分正确。

## 修订轮四、全量验收命令原始输出

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[INFO] Running net.otlab.admin.module.iot.calcread.impl.CalcReadDataPermContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- in ...CalcReadDataPermContractTest
[INFO] Running net.otlab.admin.module.iot.calcread.impl.IotCalcDdlSplitContractTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotCalcDdlSplitContractTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotPermissionsCompleteGuardTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0 -- in ...OperateLogCoverageGuardTest
[INFO] Results:
[INFO] Tests run: 2461, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
EXIT=0

$ cd calc && mvn test
[INFO] Running net.otlab.calc.state.IotCalcSchemaDdlContractTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotCalcSchemaDdlContractTest
[INFO] Running net.otlab.calc.state.JdbcCalcStateStoreContractTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- in ...JdbcCalcStateStoreContractTest
[INFO] Results:
[INFO] Tests run: 288, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
[INFO] Total time:  17.694 s
EXIT=0
```

**基线零回归**：backend 2461 = 原 2457 基线 + 4 新增（DDL 切分守卫 2 + 数据权限端到端 2），既有 2457 类逐类 0F0E、守卫全绿；calc 288 与修订前逐类一致（**calc 侧本轮零改动**），3 Skip 仍为 0041/0049 env 门控。

**范围核对**：`git show --name-only 640c9bd` = 4 文件（SQL 1 + backend calcread main 1 + test 2），全落声明范围、越界 0；`git diff fb8f66d..640c9bd` 同 4 项；DDL 未执行声明仍成立（只读探针无任何写库）；TASK 保持「进行中」。

## 修订轮五、未做项与遗留（增量）

1. **DDL 仍未在用户库执行**（契约硬约束）：修订仅改脚本文本，未执行；只读探针全 SELECT。真机验证条件同报告 §二验收 2 步骤（用户按**修订后**脚本应用：建号语句已拆行，`sql-exec.sh` 可完整跑通到 GRANT/FLUSH）。
2. **m-1 裁剪逻辑未改**：按 review 指令「方向 fail-closed 正确，不改裁剪逻辑」——受限角色单段 fullKey 状态页空集是安全保守口径；如需受限角色按实例看单段 metricCode 状态，属口径变更（需统一 state fullKey 形态或调整 visible 策略），另开任务，本轮仅订正文案 + 补证据。
3. **`JdbcTemplate` 具名 Bean 演进注意项**（评审 §三-1 附带）：当前无无限定注入点，不构成现状问题，登记备查。
4. **其余评审重点（2/3/4/5/6）零改动**，按 review「其余重点通过，不动」执行。

## 修订轮一句话结论

**通过（修订轮）**——I-1 真修复：DDL 建号六语句拆为各占一行，`IotCalcDdlSplitContractTest` 对旧脚本必红（2F）、修后转绿（0F），扫描全脚本无同行多语句；`sql-exec.sh` 真实通道只读探针 `OK/OK/OK/1/OK/DONE`（对照旧单行 `SQLERROR near 'PREPARE...'`）实证切分正确，DDL 仍未在用户库执行；m-1 顺手修：javadoc 订正为「单段 fullKey 仅 allTenant 可见/受限角色 fail-closed」，新增端到端 2 项锁定真实 `visible()` 路径行为、裁剪逻辑零改；全量 backend 2461 / calc 288 测试 0F0E、基线零回归，提交 `640c9bd` 4 文件全落范围、越界 0；TASK 保持「进行中」，待协调者 scoped 复评（只看 I-1/m-1 修复 diff 与证据）。
