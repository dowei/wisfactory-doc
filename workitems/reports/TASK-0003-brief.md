# 实施 brief：TASK-0003（租户开通触发 TDengine 建库与配额强控）

## 批次

- TASK-0003（本批共 1 个，标准任务；backend 串行道——前序 TASK-0002 须已收口，文件集与 0002 相交点见改动范围）
- BASE：`17e09b1`（主仓 HEAD，派发时回填）
- 改动范围：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/**`（TDengine 管理抽象 + 配额 + 统计任务）
  - `backend/ot-core/src/main/java/net/otlab/core/result/ResultCode.java`（仅追加枚举）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`
  - `resources/sql/upgrade/20261121_tenant_db_task.sql`（新增）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增 `ot.tdengine.*` 与 `ot.quota.*` 配置键）
  - 不碰 frontend/、ingest/、0002 已交付的登录链路与拦截器（复用不修改）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0003.md`
- 关联对象：同目录 `features\FEAT-0001.md`、`requirements\REQ-0001.md`、`testcases\TC-0001.md`
- 决策：`decisions\ADR-0011.md`（TD-0035 配额分档/TD-0041 吞吐软限）、`decisions\ADR-0002.md`（标签双保险，建库参数无关）
- 超级表契约（建库参数上下文）：`resources\contract\tdengine-super-tables.md` v1.0.0（KEEP 1825d 等建库参数见其附录 B 章）
- 前批交付：`SysTenantController/SysTenantServiceImpl`（开通流程挂点）、`backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/SysJobService.java`（框架定时任务能力）

## 人类裁决与本批实现口径（2026-11-21 人类裁决：mock 验收先做）

**TDengine 本地不可用（6041/6030 不通）→ 本任务对 TDengine 依赖项采用「接口抽象 + mock 单测验收」，真实建库/统计验证登记 TD-0053 挂到 TASK-0068 联调闭合。** 铁律 3 的落实方式：mock 验收证明的是**触发与编排逻辑**（谁在何时以什么参数调了什么），真实库行为归联调。

## 接口与决策（逐条结论，不再讨论）

1. **TDengine 管理抽象（fail-soft 边界）**：新增 `net.otlab.admin.module.iot.tdengine.TdengineAdmin` 接口（`void createDatabase(String tenantCode)`、`long storageUsageBytes(String tenantCode)`、`boolean databaseExists(String tenantCode)`）+ REST 实现 `HttpTdengineAdmin`（taosAdapter REST，base-url 配置 `ot.tdengine.base-url`，默认 `http://localhost:6041`；库名一律 `iot_<tenant_code>` 只由 tenant_code 派生，禁外部传参——REQ-0001 核心规则；建库语句按超级表契约附录 B 参数：KEEP 1825d 等全部写齐）。连接失败/超时抛业务异常由调用方捕获计数，**不重试堆栈炸穿主流程**。
2. **建库触发异步化+重试（配套裁决，理由：本地无 TDengine 不阻塞开通主流程；TDengine 故障不应阻断租户主数据落库——数据丢失不可逆、库可补建）**：
   - sys_tenant 加列 `db_created tinyint DEFAULT 0`（upgrade 脚本，幂等）；默认租户 default 置 1（私有化约定初始库由部署脚本建）
   - 开通（新建租户 status=1）事务提交后置 db_created=0 入待建；**定时扫描任务**（框架 sys_job 注册，建议 60s 间隔可配）扫 status=1 且 db_created=0 的租户 → 调 TdengineAdmin.createDatabase → 成功置 1；失败记 WARN 日志（含 tenant_code 与异常摘要）留待下轮，连续失败不升级不阻断
   - 事件式即时触发（开通后立刻尝试一次）可加但不替代扫描兜底
3. **点位配额强控**：SysTenantService 新增 `void checkPointQuota(Long tenantId)`——point_quota 空/0=不限直接过；达上限抛 `BusinessException(ResultCode.QUOTA_EXCEEDED.getCode(), "点位配额不足（上限 N）")`；`ResultCode` 追加 `QUOTA_EXCEEDED(4003, "配额不足")`（4001-4099 参数与校验段，语义全局——后续导出条数等配额复用）。**调用方接线归 TASK-0021（点位 CRUD），本任务只交付校验器与单测**（mock 计数验证 100/100 拒、101 提示明确）。
4. **存储/吞吐统计与软限告警**：定时任务（框架 sys_job，建议 5min 间隔可配）逐启用租户调 `storageUsageBytes` → 与 `storage_quota_gb`/阈值配置（`ot.quota.storage-warn-ratio` 默认 0.8）比对 → 超阈值发 WARN 告警日志（格式固定可 grep：`[QUOTA-WARN] tenant=<code> type=storage usage=<x> quota=<y>`；吞吐同型 type=throughput）——**只告警不停写**（TD-0035/0041 定版）；TDengine 不可用时本轮跳过并 WARN 一次（防日志风暴：连续失败只在首次与每 N 轮记一次）。
5. **测试口径（mock 验收）**：TdengineAdmin 全部 mock——建库参数断言（库名/KEEP）、开通置待建标志、扫描任务成功置 1/失败留 0+WARN、配额校验边界（空/0/未达/达限）、统计任务超阈值告警与不可用静默。TDD 先红后绿。

## 验收命令（整体验收）

环境事实：MySQL 127.0.0.1:3307 root/root 库 ot_wisfactory；Redis localhost:6379；8098 唯一端口（启动前探测，被占等待，自己起的完成后必须停）。SQL 走 `bash backend/scripts/sql-exec.sh`（多条语句每条一行行尾分号）。

1. `cd backend && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（含 TASK-0002 后基线 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对 `SHOW CREATE TABLE sys_tenant`（db_created 列在位）
3. dev 冒烟（真实验证触发与重试路径——本地无 TDengine 恰好验证 fail-soft）：admin 登录 → 建租户 E → 回读 db_created=0 → 等/触发扫描轮 → 日志出现建库尝试 WARN（含 iot_e）且 db_created 仍 0、开通主流程未受影响（租户 E status=1 可查）→ 手动将 ot.tdengine.base-url 指向不可达地址与可达假地址各验一轮失败路径 → 全程原始输出贴回
4. prod 档真实启动回归（BUG-0001 已修原命令）→ Started 无 ERROR + list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0003-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0003）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
