# TASK-0063 diff 评审报告（代码评审）

- 评审者：fresh 代码评审子代理（no-subagents；只评审、只写本报告，未改动任何源码 / workitems）
- 口径：L1-execute §13「diff 评审」；评审对象提交 `fb8f66d`（BASE=`ea34d02`，隔离 `e9cc251`；18 文件）
- 证据来源：brief 指定文件（diff 包 / 实施 brief / 报告 / TASK-0063 / FEAT-0020 / REQ-0022 / REQ-0023 / TC-0020 / ADR-0009 / TASK-0047-review-round2 / 详细设计 §1.9/§1.9A/§1.10 权威 PDF）+ 工作区 `D:\workspace\WisFactory` 源码
- 独立核验动作：
  - `git show --name-only fb8f66d` = 18 文件全落声明范围（calc 8 + SQL 1 + backend/ot-admin 9），越界 0；`ea34d02..fb8f66d` 夹入的 `e9cc251`（0053，openapi/subscribe 7 文件）与本提交零文件交集。
  - `grep -rn "iot:calc:" resources/sql backend --include=*.sql` → 无 `iot:calc:*` 种子；`IotInstanceMenuPermissionGuardTest.java:52` 确认 `iot:instance:query` = 4031 已注册；TASK-0064 收尾确认「重算为内部令牌接口无可注册对象」。
  - surefire XML 正则聚合：`ot-admin` 234 报告 **2457/0F/0E/0Skip**，`calc` 43 报告 **288/0F/0E/3Skip**，与报告 Raw 段一致。
  - 详细设计 PDF 用 pypdf 独立提取英文 Table DDL（v1.1 第 33/36 页）与 §1.9/§1.9A 字段表（第 16/17 页），逐字比对 DDL 脚本（中文文本层 pdftotext 乱码，但**字段名/索引/约束的拉丁文本可提取**，本次不受 0047 评审「PDF 不可读」的无法判定限制）。
  - **只读探针**：`sql-exec.sh "SELECT SCHEMA_NAME ... WHERE SCHEMA_NAME='iot_calc'" ... ""` 只返回表头、`DONE`，无 `iot_calc` 行 → 用户库确无该 schema，DDL 未执行属实（未执行任何写库语句）。
  - **只读执行形态探针**（用于发现项 F-1）：单行 `SET @s=...; PREPARE ...; EXECUTE ...; DEALLOCATE ...;` → `SQLERROR ... near 'PREPARE ...'`；同逻辑拆成每行一条 → `OK(0)/OK(0)/1/OK(0)/DONE`。

---

## 一、六项评审重点逐条结论

### 重点 1 · DDL 脚本正确性（权威逐字）—— **基本通过，附 1 项 Important（见发现 F-1）**

以详细设计 v1.1 §1.9（第 16 页）/§1.9A（第 17 页）字段表与附录 A DDL（第 33/36 页）逐字核对 `resources/sql/upgrade/20261124_iot_calc_schema.sql`：

| 核对项 | 权威契约 | 脚本实态 | 判定 |
| --- | --- | --- | --- |
| 独立 schema | §1.9「放独立 schema（如 iot_calc）」 | `CREATE SCHEMA IF NOT EXISTS \`iot_calc\`` | ✓ |
| iot_calc_state 字段（§1.9） | id / tenant_id / full_key / last_calc_time / watermark / calc_status / error_count / last_error / update_time（9 列） | 9 列一一对应，类型一致（varchar(180)/datetime/tinyint/int/varchar(1000)） | ✓ |
| state 唯一键 | `uk_tenant_full_key (tenant_id, full_key)` | 同名同列 | ✓ |
| state 无审计四件套 | §1.9「本表不套审计四件套，仅留 update_time」 | state 建表块内无 `create_time/create_by/update_by/deleted/version`，仅 `update_time ON UPDATE CURRENT_TIMESTAMP` | ✓（与重点 4 一致） |
| iot_calc_log 字段（§1.9A） | 14 列（id…duration_ms，含 `input_summary json`、`is_recalc tinyint`） | 14 列一一对应，含 `json` 类型与三索引 | ✓ |
| log 三索引 | `idx_task` / `idx_key_period` / `idx_trigger_time` | 三个索引列名/列序逐字一致 | ✓ |
| 账号分离 | §1.9「计算引擎独占写、管理系统只读；独立账号」 | `iot_calc_rw`（SELECT/INSERT/UPDATE）与 `iot_calc_ro`（仅 SELECT）分账号 GRANT | ✓ |
| 口令无硬编码 | brief 契约 | `${IOT_CALC_RW_PASSWORD}` / `${IOT_CALC_RO_PASSWORD}` 占位 + 头注「由部署注入」；无 `IDENTIFIED BY 'root'` 等真值 | ✓ |
| 幂等 | brief 契约 | schema/表 `IF NOT EXISTS`；账号 `information_schema.USER_PRIVILEGES` 判存 + 预处理动态 SQL | ✓（静态断言机制真实） |
| ADR-0009 Q1 头注 | Q1 定版 A | 头注第 3 行「ADR-0009 Q1 定版 A（同实例分 schema + 独立账号…）」 | ✓ |
| 脚本未在用户库执行 | 硬约束 | 只读探针无 `iot_calc` schema；报告「DDL 未执行声明」+ H2 契约佐证 | ✓ |

脚本**内容**正确；但账号建号段落的**可应用性**存在 Important 问题（F-1）：`PREPARE/EXECUTE/DEALLOCATE` 三条语句写在同一行，与脚本自身声明的执行通道 `sql-exec.sh`（按 `;\n` 切分）不兼容。

### 重点 2 · CalcStateStore JDBC 替换 —— **通过**

- 接口实现：`JdbcCalcStateStore implements CalcStateStore`（0047 接口 4 方法 get/upsert/byTenant/all 全实现），字段映射逐字 §1.9（tenant_id/full_key/last_calc_time/watermark/calc_status/error_count/last_error/update_time）。
- **0047 消费侧零改动核实**：`git show --name-only fb8f66d` 中**不含** `CalcTaskManager.java`/`CalcStateMachine.java`/`CalcStatusController.java`/`CalcState.java`/`InMemoryCalcStateStore.java`；仅改 `CalcEngineApplication.calcStateStore(CalcProperties)` 的 `@Bean`（`:235`）与 `CalcProperties`/`application.yml` env 映射。`CalcTaskManager`/`CalcStateMachine` 均依赖接口类型（0047 round2 已确认），消费侧确实零改动。
- env 切换：`CALC_STATE_JDBC_ENABLED=true` + URL 非空 → `JdbcCalcStateStore`，否则内存实现（默认 false 零回归）；缺 URL fail-fast（`IllegalStateException`），设计合理。
- **@TestFactory 非恒真**：`JdbcCalcStateStoreContractTest.sameContractRunsOnRealJdbcImplementation()` 的 factories 同时含 `InMemoryCalcStateStore` 与 `new JdbcCalcStateStore(h2Jdbc())`（真 H2 JDBC、建同结构表含 `uk_tenant_full_key`），每实现各跑 upsert/get、同 uk 覆盖、字段往返 3 条真实断言；非 `assertInstanceOf` 恒真。upsert 采用 UPDATE-then-INSERT（`:67-79`），同 uk 覆盖断言 `all().size()==1`，跨 MySQL/H2 方言通用。
- fail-soft：`connectionFailureIsFailSoftAndNeverBlocksOnlineCalc` 用 `jdbc:h2:tcp://127.0.0.1:1` 必失败源，断言 get→null / all,byTenant→空集 / upsert `assertDoesNotThrow`，与 javadoc「不阻塞在线计算」一致。
- 读写账号语义：calc 侧 `CALC_STATE_JDBC_USER=iot_calc_rw`（读写），管理系统 `OT_CALC_READ_JDBC_USER=iot_calc_ro`（只读），与 DDL GRANT 对齐。

### 重点 3 · 管理系统只读端点 —— **通过（含明确判定，见第三节）**

- 独立只读源：`CalcReadDataSourceConfig` 自建 `DriverManagerDataSource` 并暴露 `@Bean("calcReadJdbcTemplate")`；服务层 `@Qualifier("calcReadJdbcTemplate")` 注入（`:384`）。未配置 URL 时装配「取连接即抛」占位源、fail-soft 空集、不依赖 test-scope H2、不连宿主库。
- **不进 @Primary 核实**：`@Primary` 仅存于 `ot-core`/`DynamicDataSourceConfig.java:33` 的业务库 master 动态源；`CalcReadDataSourceConfig` 全文无 `@Primary`，且 `DriverManagerDataSource` 是方法内局部变量、**未注册为 Spring Bean**，物理上不存在第二个 DataSource Bean 竞争主源；JdbcTemplate 以具名 Bean + `@Qualifier` 消费。全仓 `grep JdbcTemplate` 在本模块外无无限定注入点。**无主数据源污染，判定成立。**
- 数据权限裁剪：`CalcReadServiceImpl.visible()`（`:123-136`）对可解析实例的 fullKey 走 `selectByCodesForTenant` → `VisibleScope.instanceIds()` 命中，越权不返回；`allTenant` 直过。详见发现 F-2（路径未测 + 文档/实现不一致，fail-closed 无泄露）。
- 读不审（0079）：两 GET 无 `@OperateLog`；`OperateLogCoverageGuardTest.writeVerb()`（`:153-165`）只认 POST/PUT/DELETE/PATCH，GET 不在覆盖集，范式一致。
- 零写口：控制器仅 `/state`、`/log` 两个 `@GetMapping`（`:47/:54`），测试机械断言 `getMappings==2 && writeMappings==0`；状态页写操作归 0061。
- 权限码沿用 + javadoc 0061 平移登记：见第三节专项判定。

### 重点 4 · 不套审计四件套 —— **通过**

`iot_calc_state` 建表块无 `create_time/create_by/update_by/deleted/version`（DDL 第 21-29 行 + 契约测试 `stateColumnsAndNoAuditQuartet()` 以建表块为界隔离断言），仅保留 §1.9 明文允许的 `update_time`。与 TASK-0063「iot_calc_state 不套审计四件套」技术方案明文一致。

### 重点 5 · 测试有效性 —— **通过**

- 独立 surefire 聚合：backend `ot-admin` 2457/0/0/0、calc 288/0/0/3（3 Skip 为 0041/0049 既有 env 门控），与报告 Raw 段一致；新增 backend 8（IotCalcReadControllerTest 3 + CalcReadServiceImplTest 5）+ calc 13（DDL 静态 5 + JDBC 契约 7 + 配置默认 1）。
- DDL 静态检查：`IotCalcSchemaDdlContractTest` 5 项覆盖幂等/schema+ADR 头注/state 全字段+无审计四件套/log 全字段+三索引/账号分离+口令占位；机制断言真机制（逐行扫裸 `CREATE USER` + `information_schema.USER_PRIVILEGES` 判存），非字符巧合。
- TDD 红转绿留痕 4 处（DDL 断言机制修正、JDBC 契约 GREEN、backend 3 处真实修复、并行 JVM 锁 `target/classes` 环境类恢复）在报告 §二逐条登记，理由可复核；“并行锁致 454 `NoClassDefFound` → `mvn clean compile` 恢复 694 类”属构建目录状态，非代码回归。

### 重点 6 · 零回归与范围 —— **通过**

- 0047/0050/0053 既有测试零回归：275 calc 基线逐类计数一致（报告 §二验收 1 续），backend 全量守卫 `IotPermissionsCompleteGuardTest`/`OperateLogCoverageGuardTest` 全绿。
- commit 仅 18 文件（已独立 `git show` 核对），全落声明范围；`frontend/`/`ingest/`/冻结契约零触碰；H2 真 JDBC 契约（非真机，DDL 未应用的列条件已在报告登记为「真机复验」条件）。
- 越界反查：`e9cc251`（0053）已隔离；DDL 未执行（只读探针佐证）。

---

## 二、发现清单

| 严重级 | 出处（绝对路径:行） | 一句话说明 |
| --- | --- | --- |
| **Important** | `D:\workspace\WisFactory\resources\sql\upgrade\20261124_iot_calc_schema.sql:64` 与 `:73` | 建号段落把 `PREPARE …; EXECUTE …; DEALLOCATE …;` 三条语句写在同一行，脚本自身声明的执行通道 `sql-exec.sh` 只按 `;\n` 切分，实测该行报 `SQLERROR near 'PREPARE …'`，账号建号失败、DDL 只建出 schema+两表，需改为每条一行（既有 `20261122`/`20261123` 脚本即此范式）。 |
| **Minor** | `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\calcread\impl\CalcReadServiceImpl.java:37` 对比 `:128-129` | javadoc 称「旧形态单段 metricCode 仅同租户可见」，但 `visible()` 对解析不出实例段的 fullKey 直接 `return false` 把行全部裁掉；而实际写入 `iot_calc_state` 的 fullKey 来自 `item.fullKey()`（=metricCode，单段，见 `CalcEngineApplication.java:288` / `MetricRuntime.java:250`），因此非 allTenant 角色在状态端点可能看不到任何行——方向 fail-closed 不泄露，但与文档不符且该裁剪路径无端到端测试（核心读测试用 `new CalcReadServiceImpl(jdbc)` 跳过数据权限）。 |

### F-1（Important）细节与证据

- **复现**（只读探针，未执行任何 DDL/写库）：
  - 单行形态（同 DDL 第 64 行）：`SET @s='SELECT 1 AS x'; PREPARE st FROM @s; EXECUTE st; DEALLOCATE PREPARE st;` → `SQLERROR: You have an error in your SQL syntax ... near 'PREPARE st FROM @s; EXECUTE st; DEALLOCATE PREPARE st' at line 1`。
  - 拆行形态：`SET @s='SELECT 1 AS x';\nPREPARE st2 FROM @s;\nEXECUTE st2;\nDEALLOCATE PREPARE st2;` → `OK(0)/OK(0)/1/OK(0)/DONE`。
- **根因**：`backend/scripts/sql-exec.sh` 以 `sql.split(";\\s*\\n")` 切分，只有「分号+换行」才是分隔符；同行的 `;` 被当作一条语句内容交给 JDBC `Statement.execute()`，MySQL 拒绝一条字符串里的多语句。
- **影响**：报告 §二验收 2 步骤 2 与脚本头注第 8 行均指定 `sql-exec.sh` 为执行通道；按该通道跑完整文件时，schema 与两表建成、但到 ④（rw 账号）即中止，`iot_calc_rw`/`iot_calc_ro` 未创建，后续 env 配置（步骤 4）因无账号不可用。属可一步修复（把第 64、73 行拆成各 3 行）但会阻断交付物的核心用途。
- **不是**脚本内容错误：账号名/GRANT/判存/占位均正确；仅语句分行格式不符合执行工具。

### F-2（Minor）细节与证据

- 写入侧：`CalcTaskManager.run()` 将请求 `fullKeys[]` 原样作为 `item.fullKey()`，经 `CalcEngineApplication.java:288` 写入 state；`MetricRuntime.recomputeWindow()`（`:250`）以 `d.metricCode().equals(fullKey)` 匹配，证明 state 的 fullKey 是**单段 metricCode**。
- 读取侧：`instanceCodeOf()` 按 `\\|` 切分取 `parts[2]`；单段 metricCode → `null` → `visible()` 返回 `false`（`:128-129`），所有 state 行被裁掉。对照 `iot_calc_log` 写入用四段 `tenant|metric|instance|period`（`MetricRuntime.java:198/219/258`），log 端点裁剪正常。
- 安全方向：`WHERE tenant_id=?` 已保证同租户，多裁不泄露，非越权漏洞；但 tenant-admin（scope=1 → `allTenant=true`）不受影响，仅部门/实例受限角色在状态页可能空集，与 REQ-0023「按 fullKey/实例查水位」的可观测目标有差距。建议统一 state 的 fullKey 口径或在 `visible()` 对无实例段 fullKey 按 javadoc 放行（同租户），并补一条真实 fullKey 的数据权限测试。

---

## 三、专项判定：重点 3「iot_calc_ro 不进 @Primary + 权限码沿用」—— **明确通过，无 C/I**

就 brief 特别要求的两个子点分别给出明确判定：

1. **iot_calc_ro 不进 @Primary / 无主数据源污染 —— 判定：通过（成立）**。
   - `@Primary` 在全仓仅见 `ot-core/config/datasource/DynamicDataSourceConfig.java:33`（业务库 master 动态源）；`CalcReadDataSourceConfig.java` 无 `@Primary`、无 `DataSource` Bean（其 `DriverManagerDataSource` 是方法内局部变量），不存在被自动注入为主源或被动态路由接管的可能。
   - 消费：`@Bean("calcReadJdbcTemplate")` + `@Qualifier("calcReadJdbcTemplate")`（`CalcReadServiceImpl.java:384`）；全模块外无无限定 `JdbcTemplate` 注入点；全量 Spring 上下文测试零回归。物理账号隔离（iot_calc_ro 仅 SELECT）+ 代码 SELECT-only 双保险成立，**无写穿业务库、无主源污染**。
   - 附带说明：因新增了一个具名 `JdbcTemplate` Bean，若未来出现无 `@Qualifier` 的 `JdbcTemplate` 注入会与自动配置的 `jdbcTemplate` 二义；当前无此消费点，不构成现状问题，登记为演进注意项。

2. **权限码沿用 `iot:instance:query` —— 判定：合理、合规，通过**。
   - 事实链：brief 决策 3 要求「复用 `iot:calc:query` 或最贴近既有权限码，若无则沿用管理面只读权限」；`TASK-0064.md:29` 收尾明示「重算为内部令牌接口无可注册对象」，全仓 SQL `grep iot:calc:` 为空 → **确无 `iot:calc:*` 位**；`iot:instance:query` 为已注册种子 4031（`IotInstanceMenuPermissionGuardTest.java:52`，`20261122_iot_instance_columns.sql`），且实例域守卫只扫 `IotModelInstanceController` 单类，新端点复用不改其一一对应。
   - 合理性：取「最贴近既有只读位」且与 fullKey 实例维度数据权限裁剪同口径，避免为建一个新位而越界改 0064 种子及其守卫测试（超出本任务 backend/ot-admin/src/main 声明范围）；全局守卫「注解 ⊆ 种子集」仍成立（2457 测试含守卫全绿），无 403 窗口扩大到无权限角色。
   - 可演进性：`IotCalcReadController` javadoc（`:29-30`）已登记「后续 0061 建 calc 位时注解平移、业务零改」，平移点清晰。
   - 结论：**沿用决策正确，不构成发现**；`/state` 与 `/log` 两 GET 均挂该位（`:48/:55`），测试 `bothEndpointsAreReadonlyAndGuarded` 机械锁定。

---

## 四、无法判定

1. **真 MySQL 实库 DDL 应用态**：只读探针确认用户库当前无 `iot_calc` schema，但本评审未执行（也不应执行）DDL；建表/建号在 phpStudy 8.0.12@3307 上的最终语法实态（F-1 修复后）需在用户按步骤应用后复验。
2. **多实例并发 upsert 的唯一键竞态**：`JdbcCalcStateStore` 无本地缓存/事务，两实例同时 UPDATE 0 行→INSERT 时后者会因唯一键冲突被 fail-soft 吞掉；报告 §五-5 已登记为演进位，本评审未做并发压测，判「无法判定（现口径不阻断）」。
3. **数据权限裁剪的端到端 403 行为**：核心读测试以 `new CalcReadServiceImpl(jdbc)` 跳过数据权限（deps=null），未见对 `visible()` 真路径的越权/可见断言；F-2 依源码静态推导 + fullKey 写入链确认，未起真实 sa-token + DataPermService 联调，判「静态判定，运行时未独立验证」。
4. **TDD 历史 RED 留痕**：4 处红转绿依报告原始输出，未回放历史提交断言，判「报告自证充分、未独立复现」。

---

## 五、结论

**不通过（退回修复 1 项 Important 后复评）**——六项评审重点中，重点 2（JDBC 替换 + 0047 消费侧零改动）、重点 3（不进 @Primary + 权限码沿用，专项判定明确通过）、重点 4/5/6 均通过；重点 1 的 DDL **脚本内容**逐字对齐详细设计 §1.9A/§1.10（全字段/三索引/唯一键/无审计四件套/账号分离/口令占位/幂等/ADR 头注全过）、DDL 未执行属实，但**建号段落的语句分行格式与脚本自身声明的执行通道 `sql-exec.sh` 不兼容**（F-1，只读探针实测报错），会在用户按报告步骤应用时于建号处中止、留下无账号的半套 schema，须修复后复评。另登记 1 项 Minor（F-2 数据权限裁剪文档/实现不一致 + 未测，fail-closed 不泄露）。无 Critical。

交付提交 `fb8f66d` 18 文件全落声明范围、越界 0，区间内 0053 修订已隔离，frontend/ingest/契约零触碰；backend 2457/calc 288 测试 0F 已独立复核。本评审未修改任何源码或 workitems（仅写本报告）。
