# 实施 brief：微改批 TASK-0077 + TASK-0078（两任务一个执行者，统一测一次）

## 批次

- 微改批（N=2 → 同一 subagent，全部做完统一测一次；用户已批准）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/tdengine/**`（0078：TdengineProperties 默认值；0077 若 STABLE_DDL 常量在此）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/openapi/**`（0077：OpenApiTimeseriesClient 指标查询 SQL 一处）
  - `backend/ot-admin/src/test/java/**`（对应测试）
  - 不碰其他域、不碰 SQL 升级脚本、不碰 workitems（报告除外）

## 任务一：TASK-0077（BUG-0004 修复）

- 背景：TDengine 3.4.3.9 将 period 列入保留字——`HttpTdengineAdmin.STABLE_DDL[3]`（st_metric_value 建表）与 `OpenApiTimeseriesClient` L187 附近指标查询的裸 period 列名被拒（9728）。证据：`reports\TD-0053-verify-report2.md`（现场已以反引号最小偏离手工补建 st_metric_value）。
- 修复口径：两处裸 period 列名加反引号（`` `period` ``）——**唯一最小偏离形态**，与现场已补建形态一致；不做其他重构。
- **BUG 修复类加严（§13）**：
  1. **全仓同契约点扫描**：grep 全仓 TDengine 面向 SQL（Java 字符串常量）中的裸 period 列名（`\bperiod\b` 不带反引号的 SELECT/DDL 上下文），扫描结果贴报告——除两处已知点外若还有命中，一并列出待裁决（不许自行扩散修复范围）
  2. **回归测试先红后绿**：新增契约测试断言 STABLE_DDL[3] 含 `` `period` ``（反引号形态）+ 指标查询 SQL 含反引号——先对旧代码跑红、修复后绿，留痕
- 真机回归（TC-0025 步骤 5）：从修复后代码提取四张超级表 DDL → REST 对 scratch 库（wisfactory_verify_p7）逐张执行 → 全部 code 0 → SHOW STABLES 四张在位 → DROP 清理。TDengine http://192.168.31.145:6041（3.4.3.9，root/taosdata），REST only。

## 任务二：TASK-0078（读超时放宽）

- 背景：现场实测 CREATE DATABASE 稳定 20-25s（TD-0053，用户裁决接受），`TdengineProperties.readTimeoutMs` 默认 5000 必撞客户端超时——每租户建库走一轮假报错+60s 重试。
- 修复口径：默认值 `5000` → `30000` 一行改动 + javadoc 登记现场实测依据（CREATE DATABASE 20-25s，TD-0053-verify-report2）+ 新增默认值断言测试（防无意识回退）。
- 不做：按调用类型拆分超时（超出本微改范围，如需另立任务）。

## 统一验收（两任务全做完测一次）

环境事实（协调者 2026-11-23 实测）：WSL docker wisfactory-mysql（8.0.42@3307）/wisfactory-redis 正常；8098 本批不用（不起 backend）；sql-exec.sh 每条语句单独一次调用；TDengine REST 正常（建库 ~20-25s 属已知慢特性，curl 用 `-m 60`）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 真机回归段（0077）：四张超级表 DDL 对 scratch 库全 code 0 + 清理证明（SHOW DATABASES 终态）
3. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0077-0078-report.md`（一批一报告）。五项：改动文件清单 / 验收命令原始输出（含先红后绿与扫描证据）/ 未做项与原因 / 偏离计划之处 / 遗留问题。末尾一句话结论。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」（协调者收尾）；不改 workitems 库其他文件。
- 允许 git commit（main，注明 TASK-0077/0078）。
