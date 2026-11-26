# 验证 brief：TASK-0044（指标定义纳入发布快照与查询接口——交付核实）

> **重要定性**：开工核查发现本任务两个交付物**已由前序任务完整实现**——①指标纳入发布快照：0024 ConfigSnapshotAssembler.java:196-209 已按 Schema v1.0.1 装载 iot_metric；②管理面查询接口：0043 IotMetricController 六端点（list/detail/CRUD/changeStatus）+ 六权限码已在位；③对外 metrics/query：0051 OpenApiController:84 已在位。**本批次=真机验收核实，零源码改动**——若验证发现真实缺口才写报告等裁决，不自行实现。

## 批次

- 验证任务（非实现；BASE=开工 HEAD；纯验证零代码改动）
- 范围：不改任何源码；只允许验证脚本（临时文件不入库）+ 报告

## 必读（只给路径）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0044.md`（验收标准：指标变更未发布不生效；列表按数据范围过滤）
- 前序交付实证（开工已核）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/ConfigSnapshotAssembler.java:196-209`（metrics 装载）+ `iot/metric/controller/IotMetricController.java:67-132`（六端点+权限码）+ `openapi/controller/OpenApiController.java:84`（metrics/query）
- 上游报告：reports\TASK-0043-report.md（指标 CRUD）+ reports\TASK-0024-report.md（快照 Schema v1.0.1）+ reports\TASK-0051-report.md（对外六接口）
- 关联：FEAT-0014/REQ-0024/TC-0014/ADR-0011（看板同源）

## 验证项（逐项实测，原始输出留证）

1. **快照纳入实证**：dev 起 backend → 现有计算域快照（iot_config_version scope_type=2）GET /iot/publish/snapshot → 快照体含 metrics 数组且字段与 iot_metric 表一致（比对至少一条指标 fullKey/aggType/period）
2. **"保存≠生效"实证**：改一条 iot_metric（如 period HOUR→DAY，用 SQL 或管理 API）**不发布** → 再拉最新快照 → 指标仍为旧值；发布后 → 快照指标变新值；测完还原数据
3. **管理面列表数据权限**：普通角色（无 iot:metric:list）直调 GET /iot/metric/list → 403；admin 200；有权限角色按部门数据范围过滤（若有现成 smoke 角色沿用，否则记录环境限制）
4. **对外 metrics/query 同源**：0051 签名的 test_client 真调 /openapi/v1/metrics/query → 返回值与管理面同源（同 iot_metric 数据）

## 硬约束

- no-subagents；**零源码改动**；验证失败先定位配置 vs 实现缺口——实现缺口写报告**不自行修**，等裁决（可能需立微改 TASK）
- 测试造的数据测完还原（sys_tenant/iot_metric 等共享表）
- 8098 启动前探测、用毕即停；MySQL/Redis 宿主 phpStudy（8.0.12@3307/3.0.504@6379）——不要碰 wsl docker 容器；sql-exec.sh 每条语句单独一次调用
- 报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0044-report.md`（逐项原始输出 + 4 条验证结论 + 一句话：0044 可否直接 close 或需补实现）

## 报告结构

五项：改动文件清单（应为零）/ 每条验证命令原始输出 / 未验证项与原因 / 偏离计划之处 / 遗留问题。
