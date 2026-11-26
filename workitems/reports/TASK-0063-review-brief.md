# 评审 brief：TASK-0063 diff 评审（代码评审）

> 实现提交 fb8f66d（BASE=ea34d02，隔离区间剔除 0053 修订 e9cc251；18 文件）——iot_calc 独立 schema DDL + CalcStateStore JDBC 实现 + 管理系统只读端点。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0063-diff.txt`（e9cc251..fb8f66d，18 文件）
- 实施 brief + 报告：reports\TASK-0063-brief.md / TASK-0063-report.md（含 DDL 未执行声明+权限码判断点）
- TASK 详情 + FEAT-0020/REQ-0023/REQ-0022/TC-0020 + ADR-0009（Q1 定版 A）
- 上游：reports\TASK-0047-review-round2.md（CalcStateStore @Bean 注入缝+@TestFactory 证实可替换）+ calc/src/** state 代码
- 契约：详细设计 §1.9A（iot_calc_state/log 字段）+ §1.10 独立 schema

## 越界反查

- 18 文件全落声明范围（calc state/JDBC+backend calcread 只读包+resources/sql DDL+测试）；区间内 0053 修订（e9cc251）已隔离；frontend/ingest/冻结契约零触碰；**DDL 未执行**（报告声明+H2 契约佐证）。

## 评审重点

1. **DDL 脚本正确性（权威逐字）**：CREATE SCHEMA iot_calc + iot_calc_state 全字段按 §1.9A（full_key/last_calc_time/watermark/calc_status/error_count/last_error）+ uk + **无审计四件套**（TASK 明文）；iot_calc_log 全字段+三索引；iot_calc_rw/iot_calc_ro 账号分离、口令占位（无硬编码真实口令）、幂等判存、ADR-0009 Q1 头注——逐字核对；**脚本未在用户库执行**实证
2. **CalcStateStore JDBC 替换**：JdbcCalcStateStore 实现 0047 接口——**0047 消费侧零改动**核实（CalcEngineApplication @Bean env 切换）；@TestFactory 同套契约跑内存+H2 JDBC（非恒真）；连接失败 fail-soft WARN 不阻塞在线计算；读写账号语义（calc 写/管理读）
3. **管理系统只读端点**：GET /api/v1/iot/calc/state|log——独立 iot_calc_ro DataSource（**不进 @Primary 动态源**——核实无主数据源污染）；数据权限裁剪；**读不审**（0079 范式）；零写口（无 POST/PUT）；权限码沿用 iot:instance:query（0064 无 iot:calc:* 位）——判断合理性、javadoc 0061 平移登记
4. **不套审计四件套**：iot_calc_state 表无 create_by/update_by 等列——与 TASK 明文一致
5. **测试有效性**：backend 2457/calc 288 测试 0F（基线零回归+新增）；DDL 静态检查全过；4 处 TDD 红转绿留痕（含并行 java 锁 target 环境类恢复）
6. **零回归与范围**：0047/0050/0053 既有测试零回归；commit 仅 18 文件；H2 真 JDBC 契约（非真机——DDL 未应用故列补验条件）

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0063-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 3 点（iot_calc_ro 不进 @Primary 主数据源 + 权限码沿用）给出明确判定**。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
