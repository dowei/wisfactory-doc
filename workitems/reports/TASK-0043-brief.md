# 实施 brief：TASK-0043（iot_metric 指标 CRUD 与口径标注）

## 批次

- TASK-0043（本批共 1 个，标准任务 3 人天；backend 串行道）
- BASE：以开工时主仓 HEAD 为准（记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/**`（指标域新包）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_metric_domain.sql`（新增）
  - 不碰 frontend/、ingest/、dataitem 域（只读）、dataperm

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0043.md`
- 关联对象：同目录 `features\FEAT-0014.md`、`requirements\REQ-0024.md`（验收：口径未标注不可保存/比值型分子分母齐备/批量生成正确）、`testcases\TC-0014.md`
- 决策：`decisions\ADR-0009.md`（TD-0027 级联+src_type、TD-0039 上看板标记）；`decisions\ADR-0011.md`（TD-0028 新实例自动生成走发布链路）；`decisions\ADR-0014.md`（指标编码不入业务字典）
- 表结构权威：附录 A p33 iot_metric DDL（brief 已提取贴下）；接口行 §3.1 p21-22 指标段（自行提取）
- 数据权限：ADR-0016 裁决 3/6——指标列表 dept_id/instance_id 双重过滤（素材文件 reports\TASK-0005-inputs.md §一表格）
- 既有范式：iot/instance（0012 创建/裁剪/防线）、iot/template（0011 批量保存）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_metric_domain.sql，逐字 + 幂等 + 头注）

```sql
CREATE TABLE `iot_metric` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '指标 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `metric_code` varchar(96) NOT NULL COMMENT '指标编码（租户内唯一）',
  `metric_name` varchar(100) NOT NULL COMMENT '指标名称',
  `instance_id` bigint NOT NULL COMMENT '归属实例 ID',
  `source_item_id` bigint DEFAULT NULL COMMENT '来源数据项 ID（比值型可空）',
  `agg_type` tinyint NOT NULL COMMENT '聚合 1 平均 2 累计 3 最大 4 最小 5 比值',
  `period` varchar(8) NOT NULL COMMENT '周期 HOUR/DAY/MONTH/YEAR',
  `calendar_type` tinyint DEFAULT '1' COMMENT '日历口径 1 自然 2 班次（预留）',
  `ratio_num_item_id` bigint DEFAULT NULL COMMENT '比值分子数据项 ID',
  `ratio_den_item_id` bigint DEFAULT NULL COMMENT '比值分母数据项 ID',
  `yoy_flag` tinyint DEFAULT '0' COMMENT '是否算同比 0 否 1 是',
  `mom_flag` tinyint DEFAULT '0' COMMENT '是否算环比 0 否 1 是',
  `dept_id` bigint DEFAULT NULL COMMENT '绑定部门 ID',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_metric_code` (`tenant_id`,`metric_code`),
  KEY `idx_tenant_instance` (`tenant_id`,`instance_id`),
  KEY `idx_period_status` (`period`,`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='指标定义表';
```

菜单种子 **4058 起**连续（4057 已占）：iot:metric:list/query/add/edit/remove 五行挂 4014 指标管理页下，幂等 + 超管授权。

### 代码结构（/iot/metric 段，语义照 REQ-0024/§3.1 指标段）

- CRUD：list（分页+过滤 metricCode/metricName/instanceId/period/status + 数据权限裁剪——dept_id IN 部门集 OR instance_id ∈ 可见集，ADR-0016 裁决 6 同型；create_by 分支不适用资产类——照 0021 口径）、detail、add（metric_code 租户内唯一双道闸 4002 + 字符集 `[A-Za-z0-9_]`；**口径强制标注**——agg_type/period 必填且 period ∈ HOUR/DAY/MONTH/YEAR；比值型（agg_type=5）分子分母数据项 id 齐备且同实例/同租户存在启用——REQ-0024 验收 2；非比值型忽略 ratio_* 字段置 NULL；calendar_type 锁 1 自然（班次预留 2 拒绝创建——REQ-0024 一期口径）；instance_id 归属校验；dept_id 校验）、edit（metric_code 不可改；白名单）、**停用**（status 置 0=停调度结果保留——REQ-0024 口径，任务正文；删除：remove 软删+编码归档不复用）、启停事件占位（MetricStatusChangedEvent 只发布——调度消费方归 FEAT-0013/0045，照 0013 范式）。
- **批量生成（按模板）**：`POST /iot/metric/generate?instanceId=`（iot:metric:add）——**新实例自动生成走发布链路（ADR-0011 TD-0028）**：本期实现「按实例已绑定模板的指标清单生成」——模板指标定义存储在模板域（0011 未建 iot_metric_template？检查：模板域只有数据项模板——**指标模板载体未定义**。裁决：本期批量生成接口实现为「按请求传入指标清单批量生成」（同 0011 批量保存范式），「模板自动生成」留待指标模板载体定义（REQ-0024 后续裁决/0024 发布链），报告登记占位口径。生成校验同 add（逐项口径标注/比值约束/唯一性），整批事务。
- **编码不入业务字典**（ADR-0014）：metric_code 无字典校验。
- **上看板标记**（TD-0039）：本期 DDL 无该列——占位登记（REQ-0029 上看板标记随展示端任务落，报告遗留）。
- 数据权限裁剪接线 + 写路径防线（update/remove/status 照 0021 修订轮范式——resolve+可见判定，指标可见=dept_id IN 部门集 OR instance_id ∈ 可见集）+ 4002 双道闸 + `@Transactional` 照范式；tenant_id 会话覆盖。

### 测试口径（TDD）

CRUD 逐个（唯一 4002/改码拒/口径未标注拒/比值型分子分母齐备约束（缺一拒）/非比值型 ratio 置 NULL/calendar_type=2 拒/启停停调度结果保留语义（status 落库断言，调度侧归 FEAT-0013）/编码不入字典）/批量生成（逐项校验整批事务/唯一性）/裁剪（列表越权/detail 403/写路径防线红转绿）——红线套件增补指标域用例（授权例外）。
冒烟（dev）：admin 建指标（比值型完整/缺分母拒/口径未标注拒）→ 批量生成 → 裁剪场景（受限用户）→ 启停 → list 筛选 period/status。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线以开工实测为准 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对表结构 + 4058-4062 五行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + metric/list 200
5. 改动范围核对：`git diff --name-only <开工 BASE>..HEAD` 全落在声明内（redline 增补授权例外）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0043-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0043）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
