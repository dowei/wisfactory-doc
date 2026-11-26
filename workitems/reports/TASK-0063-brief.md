# 实施 brief：TASK-0063（iot_calc_state 计算状态独立 schema 与只读账号）

## 批次

- TASK-0063（标准任务 2 人天；前置 0047✅；CalcStateStore 从内存换 JDBC 实现（0047 注入缝零改动替换）+ 独立 schema DDL + 管理系统只读端点，解锁 FEAT-0020 计算成功率数据源）
- BASE：开工 HEAD（记录进报告）
- 改动范围（越界即违约）：
  - `calc/**`（CalcStateStore JDBC 实现——0047 注入缝替换）
  - `resources/sql/upgrade/20261124_iot_calc_schema.sql`（新增独立 schema+表+只读账号 DDL——**只产出脚本，不在用户库执行**）
  - `backend/ot-admin/src/main/java/**`（管理系统只读端点最小接线——仅状态读口）
  - 对应测试；不碰 frontend/ingest/契约/workitems（报告除外）

## 必读

- TASK 详情 + FEAT-0020/REQ-0023/REQ-0022/TC-0020 + ADR-0009（Q1 定版 A：独立 schema 同实例分 schema+独立账号）
- 上游：reports\TASK-0047-report.md + TASK-0047-review-round2.md（**CalcStateStore @Bean 注入缝+@TestFactory 同套契约跑内存/JDBC 替身已证实 0063 消费侧零改动**）+ calc/src/** 现有 state 代码
- 契约：详细设计 §1.9A（iot_calc_state 字段：full_key/last_calc_time/watermark/calc_status/error_count/last_error；iot_calc_log 全字段）+ §1.10 独立 schema
- 平台：MySQL 宿主 phpStudy 8.0.12@3307（root/root）——**只读探针可用，DDL 不在用户库执行**（脚本交付给用户自行应用）

## 接口与决策（逐条结论）

1. **DDL 脚本**（resources/sql/upgrade/20261124_iot_calc_schema.sql）：CREATE SCHEMA IF NOT EXISTS `iot_calc`；CREATE TABLE iot_calc_state（按 §1.9A 字段规范，**不套审计四件套**——TASK 明文）；CREATE TABLE iot_calc_log（§1.9A 全字段+索引）；CREATE USER/GRANT 只读账号（计算引擎读写账号+管理系统只读账号分离——账号名/密码用占位符+注释说明由部署注入，**不硬编码真实口令**）；头注登记 ADR-0009 Q1 定版依据。脚本幂等（IF NOT EXISTS + information_schema 判存）；**agent 不在用户库执行该 DDL**，只在报告写明应用步骤
2. **CalcStateStore JDBC 实现**：实现 0047 定义的 CalcStateStore 接口（@Bean 替换内存实现——消费侧零改动），通过 env 注入 JDBC URL/账号（只读账号仅用于管理系统侧，**计算引擎用读写账号**）；连接失败 fail-soft WARN、不阻塞在线计算；@TestFactory 同套契约跑 JDBC 替身（0047 已留测试范式）——**真库验证因 DDL 未应用到用户库，用 H2/内存 SQL 或假 DataSource 契约测试，真机列条件**
3. **管理系统只读端点**：`GET /api/v1/iot/calc/state?fullKey=`（复用 iot:calc:query 权限或最贴近既有权限码，查 0064 高危权限清单口径——若无则沿用管理面只读权限）+ `GET /api/v1/iot/calc/log?fullKey=&from=&to=`——只读取 iot_calc schema（只读账号），**状态页只读操作走重算接口**（TASK 技术方案：不通过本端点触发写）；数据权限裁剪照实例域范式
4. **审计豁免**：iot_calc_state 不套审计四件套（TASK 明文）；管理系统对 calc 状态的**只读查询**不走 @OperateLog（0079 范式：读操作不审）
5. **不碰**：0061 运维监控页（后续消费本端点）、calc 内部计算语义（0047 已交付）、契约冻结

### 测试口径（TDD）

DDL 脚本结构断言（schema/表/账号分离/幂等/无审计列）；CalcStateStore JDBC 契约（同 @TestFactory 跑内存与 JDBC 替身——0047 范式）；管理系统只读端点（有数据/空态/越权 403/只读账号不可写验证）；既有 0047/0050 测试零回归；红转绿留痕。

## 验收命令

环境：MySQL 宿主 phpStudy（只读探针可用）；**DDL 不在用户库执行**；8098 启动前探测、用毕即停；sql-exec.sh 每条语句单独一次调用。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 0 失败 0 错误（Results 段贴回）；`cd calc && mvn test` → 0 失败（275 基线+新增）
2. DDL 脚本静态检查（幂等/schema/账号分离/无审计列）+ 报告写明应用步骤
3. dev 冒烟（可选，若 DDL 由用户已应用则真库探针；否则假 DataSource 契约测试）

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0063-report.md。五项齐全 + **DDL 未执行声明**+一句话结论。

## 契约

- no-subagents；只改范围（calc/** + SQL 脚本 + backend 只读端点）；不把 TASK 置「已完成」；不改 workitems（报告除外）；不在用户库执行 DDL。
