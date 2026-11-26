# TASK-0043 实施报告：iot_metric 指标 CRUD 与口径标注

执行者：implementer（no-subagents 单体会话）
开工时间：2026-11-22 16:50 前后；报告完成：2026-11-22 17:45 前后
BASE：9e73d3e（开工时主仓 HEAD，开工实测记录；feat(iot) TASK-0036）
本任务提交：fb574cf（main，feat(iot) TASK-0043）
**五条验收全部通过**（细节与原始输出见 §2；结论先行：全量回归 1979 = 开工基线实测 1923 + 本批净增 56，0 失败 0 错误 BUILD SUCCESS（终码复跑确认）；升级脚本二次幂等两次 DONE + 表结构与附录 A p33 逐字一致 + 菜单 4058-4062 五行核对一致；dev 冒烟全序列绿（比值型完整/缺分母拒/口径未标注三态拒/班次拒/批量生成含入参覆盖与整批拒绝/裁剪场景含创建侧防线/启停落库/period+status 筛选/删除归档不复用）+ 启停事件日志留痕 2 条；prod 档真实启动 8.86s + 真实登录 200 + metric/list 200 全程 0 ERROR；本任务提交 17 文件全落声明范围）。

---

## 1. 改动文件清单（本任务提交 fb574cf，17 文件）

### 1.1 主码（12，metric 域新包 `iot/metric/**`）

| 文件 | 性质 |
| --- | --- |
| `resources/sql/upgrade/20261122_iot_metric_domain.sql` | 新增——iot_metric DDL（附录 A p33 逐字，无上游差异登记）+ sys_menu 4058-4062 五行（parent 4014）+ 超管授权；CREATE IF NOT EXISTS + INSERT IGNORE 幂等 |
| `backend/.../iot/metric/entity/IotMetric.java` | 新增——实体（占位登记：上看板标记 TD-0039 / src_type 级联 TD-0027 逐字 DDL 无对应列，不落列不校验） |
| `backend/.../iot/metric/mapper/IotMetricMapper.java` | 新增 |
| `backend/.../iot/metric/mapper/IotMetricCodeMapper.java` / `IotMetricCodeRow.java` | 新增——uk_tenant_metric_code 双道闸第一道旁路预检（含已删行、旁路多租户插件显式 (tenant_id, metric_code)） |
| `backend/.../iot/metric/dto/IotMetricCreateForm.java` / `IotMetricUpdateForm.java` / `IotMetricStatusForm.java` / `IotMetricGenerateForm.java` | 新增——四表单（创建/修改/批量启停/批量生成清单） |
| `backend/.../iot/metric/event/MetricStatusChangedEvent.java` | 新增——指标启停领域事件 record（metricId/tenantId/metricCode/newStatus/period/aggType） |
| `backend/.../iot/metric/service/IotMetricService.java` / `impl/IotMetricServiceImpl.java` | 新增——六方法服务（口径强制标注/比值型齐备与同实例存在启用 4001/唯一双道闸 4002/calendar 锁 1/编码不入字典/批量生成整批事务/裁剪与写路径防线/启停事件） |
| `backend/.../iot/metric/controller/IotMetricController.java` | 新增——七端点（list/detail/add/generate/edit/changeStatus/remove），权限码 4058-4062 语义集（generate 共用 add、changeStatus 共用 edit） |

### 1.2 测试（5）

| 文件 | 性质 |
| --- | --- |
| `backend/test/.../iot/metric/service/impl/IotMetricServiceImplTest.java` | 新增 43 用例 |
| `backend/test/.../iot/metric/config/IotMetricMenuPermissionGuardTest.java` | 新增 3 用例（SQL 4058-4062 ↔ 控制器注解双向核对，照 point 守卫范式） |
| `backend/test/.../iot/redline/MetricRedlineTest.java` | 新增 10 用例（授权例外——红线套件增补指标域：列表双条件裁剪/NULL 行 fail-closed/详情 403/scope=4 空集 create_by 不适用/1=0 fail-closed/写越权三原语 403+可见回归/创建侧防线） |
| `backend/test/.../iot/redline/RedlineFixture.java` | 修改——夹具扩展：指标行种子 + Metric Mapper/CodeMapper 桩 + 指标分页条件模拟器（dept_id IN / instance_id IN / FAIL_CLOSED 求值）+ 真实 IotMetricServiceImpl 装配 + itemMapper.selectByIds 桩 + 事件总线类型过滤（领域事件不分发给缓存失效监听器） |

净增用例 = **+56**（43+3+10，RedlineFixture 修改不增用例数）。

---

## 2. 验收命令原始输出

### 2.1 验收 1：全量回归 —— 通过

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[WARNING] Tests run: 1979, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

基线以开工实测为准（brief 口径）：BASE=9e73d3e 实测：

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[WARNING] Tests run: 1923, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

1979 = 1923 + 56（净增，明细见 §1.2），0 失败 0 错误（3 skipped 为红线二-中台 scope 占位——TASK-0051 接线后启用，基线同值）。**终码复跑说明**：全量首轮 1979 通过后，dev 冒烟 S7a 暴露一处主码缺陷（表单 `instanceId` @NotNull 拦截批量生成清单项——§3 偏离 3），修复后按纪律对提交版终码复跑全量，以上 1979 输出即终码复跑结果。

红转绿记录（新增测试首跑红 3 处，逐一「为什么现在对了」）：
1. `testGenerate_BatchSuccess` 红（「来源引用不存在: 数据项 300」）——根因：`validateAndBuild` 以 `Set<Long> prefetchedItemIds` 承载批内预载、全命中时返回空 Map 供引用校验查找，空 Map 必然判「引用不存在」。修复：预载改传 `Map<Long, IotDataItem>` 并新增 `resolveReferenceItems`（预载命中复用 + 缺失补载合并）——现在对了：批内第二遍预载与逐项构建共用同一份数据集，两条路径看到的引用行一致。
2. `MetricRedlineTest.writePath_changeStatus_invisibleForbidden` 红（ClassCastException）——根因：夹具直连事件总线把一切事件强转 `DataPermCacheEvictEvent`，指标可见回归真实发布 `MetricStatusChangedEvent` 即炸。修复：总线按类型分发（缓存失效事件进监听器、其余领域事件放行不分发）——现在对了：总线模拟的是 Spring 事件总线「监听器只消费自己的事件类型」语义，与点位/指标事件只发布不建监听的范式一致。
3. `MetricRedlineTest.createPath_generate_invisibleInstanceForbidden` 红（期望 403 得 400）——根因：`generateMetrics` 表单内容校验（清单空）先于创建侧数据权限防线执行，违反「存在性 → 数据权限 → 业务校验」顺序（0012/0021 修订轮范式）。修复：resolveInstance + requireCreatePerm 前移——现在对了：写原语先过数据权限闸再碰表单内容，与 update/changeStatus/deleteMetrics 的防线顺序一致。

### 2.2 验收 2：升级脚本二次幂等 + 表结构/菜单核对 —— 通过

```
$ bash sql-exec.sh resources/sql/upgrade/20261122_iot_metric_domain.sql   # 第一次
OK(0)          # SET NAMES
OK(0)          # CREATE TABLE IF NOT EXISTS（建表，返回受影响 0）
OK(5)          # sys_menu 4058-4062 五行插入
OK(5)          # sys_role_menu 超管五行授权
DONE
$ bash sql-exec.sh resources/sql/upgrade/20261122_iot_metric_domain.sql   # 第二次
OK(0)
OK(0)
OK(0)
OK(0)
DONE           # 两次 DONE；第二次全部 OK(0) 不报错不重复插入
```

表结构与菜单核对（三条核对语句原始输出）：

```
$ bash sql-exec.sh "SHOW CREATE TABLE iot_metric; ..."   （完整输出）
Table	Create Table
iot_metric	CREATE TABLE `iot_metric` (
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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='指标定义表'
id	menu_name	menu_type	perms
4058	指标查询	3	iot:metric:list
4059	指标详情	3	iot:metric:query
4060	指标新增	3	iot:metric:add
4061	指标修改	3	iot:metric:edit
4062	指标删除	3	iot:metric:remove
role_id	menu_id
1	4058
1	4059
1	4060
1	4061
1	4062
DONE
```

逐字核对：与 brief 逐字 DDL（pypdf 自附录 A p33 复核一致，字形规范化不改语义）列序/列型/注释/索引完全一致（`COLLATE=utf8mb4_0900_ai_ci` 为 MySQL 8 默认排序规则随库继承，脚本未显式指定 COLLATE——与点位/通道域脚本同口径）；菜单五行与 4058-4062 语义一一对应（4057 确为 TASK-0021 点位域占用，4014 为既有指标管理页，号段连续取号无冲突）。

### 2.3 验收 3：dev 冒烟 —— 通过

启动：dev.ps1（8098，dev 档）→ 冒烟脚本 `backend/target/t0043_smoke.py` 原始输出（完整贴回）：

```
=== S1) 双账号 dev login（admin / 受限 smoke_perm_a_t0006） === S1a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S1b -> HTTP 200 code 200 | 操作成功  [期望200] OK
=== S2) admin 建比值型指标（分子分母齐备同实例启用）-> 成功，回读四要素/比值字段/日历 1 ===
S2a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2 断言: 比值型 id=4 落库 agg=5 num=9 den=10 period=DAY calendar=1 yoy=1 mom=1 status=1 OK
=== S3) admin 建比值缺分母 -> 拒（REQ-0024 验收 2——缺一拒，未落库） ===
S3a -> HTTP 200 code 400 | 比值型指标必须同时提供分子与分母数据项（ratioNumItemId/ratioDenItemId 缺一不可）  [期望400] OK
S3 断言: 缺分母 400 拒且未落库 OK
=== S4) admin 口径未标注拒（agg_type 缺失 / period 缺失 / period=WEEK 非法——未标注不可保存） ===
S4a agg_type 缺失 -> HTTP 200 code 400 | 聚合方式不能为空（口径强制标注——未标注不可保存）  [期望400] OK
S4b period 缺失 -> HTTP 200 code 400 | 统计周期不能为空（口径强制标注——未标注不可保存）  [期望400] OK
S4c period=WEEK -> HTTP 200 code 400 | 统计周期仅允许 HOUR/DAY/MONTH/YEAR  [期望400] OK
S4 断言: 口径未标注三态 400 拒且均未落库 OK
=== S5) admin 班次日历 calendar_type=2 -> 拒（REQ-0024 一期不做班次口径，锁 1） ===
S5a -> HTTP 200 code 400 | 班次日历口径一期不开放（REQ-0024 一期不做班次口径聚合，calendar_type 锁 1 自然日历）  [期望400] OK
S5 断言: 班次口径拒创建（calendar_type 锁 1） OK
=== S6) admin 建聚合型指标（HOUR，instance 3）-> 成功 ===
S6a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S6 断言: 聚合型 id=5 落库 agg=1 period=HOUR OK
=== S6b) admin 建不可见实例指标（instance 4 = F01.WS02 dept 2，供受限用户裁剪场景） ===
S6b -> HTTP 200 code 200 | 操作成功  [期望200] OK
S6b 断言: admin 对 dept 2 实例建指标成功（scope=1 全租户） OK
=== S7) 批量生成（instanceId=12，按请求清单——同 0011 批量保存范式） ===
S7a 两项清单 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S7 断言: 批量生成 2 条落库 instance=12（项内 instanceId=99 被入参覆盖） OK
S7b 编码占用重放 -> HTTP 200 code 4002 | 指标编码已存在: GEN_T43_M1  [期望4002] OK
S7c 坏项整批拒绝 -> HTTP 200 code 400 | 比值型指标必须同时提供分子与分母数据项（ratioNumItemId/ratioDenItemId 缺一不可）  [期望400] OK
S7 断言: 占用 4002 / 坏项 400 整批拒绝且未落库任何行 OK
=== S8) 裁剪场景（受限用户——dept 4 车间口径） ===
S8a restricted metric list -> HTTP 200 code 200 | 操作成功  [期望200] OK
S8a 断言: restricted 可见（dept/instance 双分支）且 instance 4 指标不返回 -> ['GEN_T43_M1', 'GEN_T43_M2', 'T43_SMOKE_AGG_H', 'T43_SMOKE_RATIO']
S8b detail 越权 -> HTTP 200 code 403 | 无权访问该指标（数据权限不命中）  [期望403] OK
S8b 断言: 受限用户查不可见指标详情 -> 403（数据权限不命中） OK
S8c edit 越权 -> HTTP 200 code 403 | 无权操作该指标（数据权限不命中）  [期望403] OK
S8c 断言: 受限用户直改不可见指标 -> 403 且未生效 OK
S8d generate 越权 -> HTTP 200 code 403 | 无权在该实例下创建指标（数据权限不命中）  [期望403] OK
S8d 断言: 受限用户向不可见实例批量生成 -> 403（创建侧防线，照 0012 I-1 同型） OK
S8e generate 可见实例放行 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S8e 断言: 受限用户向可见实例批量生成 -> 200（防线不误伤） OK
S8f admin 对照面 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S8f 断言: admin 可见全部冒烟指标（scope=1 对照面） OK
=== S9) 启停（停用停调度结果保留——status 落库断言；事件日志留痕见报告 grep） ===
S9a disable -> HTTP 200 code 200 | 操作成功  [期望200] OK
S9b enable -> HTTP 200 code 200 | 操作成功  [期望200] OK
S9 断言: changeStatus 置 0/置 1 落库 status 断言 OK（事件留痕 grep 见报告）
=== S10) list 筛选 period/status ===
S10a period=HOUR -> HTTP 200 code 200 | 操作成功  [期望200] OK
S10b period=DAY&status=1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S10 断言: period=HOUR 单值命中；period=DAY&status=1 组合筛选（5 行） OK
=== S11) 收尾：删除冒烟指标（软删）+ 编码归档不复用演示 ===
S11a 批量删除 [4, 7, 8, 9, 6, 5] -> HTTP 200 code 200 | 操作成功  [期望200] OK
S11b 同编码重存 -> HTTP 200 code 4002 | 指标编码已被删除记录占用，不可复用: T43_SMOKE_RATIO  [期望4002] OK
S11 断言: 软删后列表不返回；同编码重存 4002 归档不复用 OK
SMOKE DONE
```

启停事件日志留痕核对（事件无消费方，发布即留痕——`logs/ot-admin-dev.log`）：

```
$ grep -c "指标启停事件已发布" backend/logs/ot-admin-dev.log
2
$ grep "指标启停事件已发布" backend/logs/ot-admin-dev.log   （完整，UTF-8 原文）
2026-11-22 17:30:00.111 ... 指标启停事件已发布（停用停调度结果保留，调度消费方 FEAT-0013/0045）: metricId=4, tenantId=1, metricCode=T43_SMOKE_RATIO, newStatus=0, period=DAY, aggType=5
2026-11-22 17:30:00.239 ... 指标启停事件已发布（停用停调度结果保留，调度消费方 FEAT-0013/0045）: metricId=4, tenantId=1, metricCode=T43_SMOKE_RATIO, newStatus=1, period=DAY, aggType=5
```

2 条 = S9a 置 0（newStatus=0）+ S9b 置 1（newStatus=1）——置 0/置 1 两条触发路径均留痕，payload 携带调度消费所需的周期/聚合口径。用毕即停：dev.ps1 stop → PID 已停 → `netstat :8098 LISTENING` 复核无输出。

冒烟场景种子（dev 库内操作，不入库不进提交）：受限账号 smoke_perm_a_t0006（user 6，dept 4，role 6 data_scope=2）补授 iot:metric:list/query/add/edit/remove 五菜单（INSERT IGNORE sys_role_menu，OK(5)）；比值分子分母复用 dev 库既有数据项（instance 12 的 iot_data_item 9/10/11/12，enabled=1）；裁剪场景锚点：instance 3（dept 4，dept 分支可见）、instance 12（dept NULL、父 8 dept 4 子树，instance 分支可见）、instance 4（F01.WS02 dept 2，双分支皆不命中=不可见）。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn package -pl ot-admin -am -DskipTests`（终码，含 §3 偏离 3 修复）→ ot-admin-1.0.0.jar。启动命令（沿 0017/0019/0020/0021 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

**干净单轮（日志 /tmp/task0043/prod_clean.log，Redis 预置 captcha:code:t0043p="AB43" JSON 字符串 EX 600 + SM4 admin123 密文）**：

```
Started OtAdminApplication in 8.859 seconds (process running for 9.34)
=== ERROR count ===
0
LOGIN HTTP 200 code 200 message 操作成功
LOGIN token 1910072f...
LIST /iot/metric/list HTTP 200 code 200 total 0
METRIC-LIST rows: []
LIST /iot/point/list HTTP 200 code 200 total 2
PROD PROBE DONE
```

真实登录 200 + metric/list 200（total 0——冒烟指标已软删、列表过滤逻辑删行，正确行为）+ point/list 200（既有数据对照面）；启动段与全程 **ERROR=0**（探针后再核一次仍为 0）。单轮即干净（Redis 验证码值按 0021 教训预置为 JSON 字符串形态）。用毕即停：taskkill PID 25036 → `netstat :8098 LISTENING` 复核无输出，Redis 验证码键已被登录消费（DEL → 0）。

### 2.5 验收 5：改动范围核对 —— 通过

`git diff --name-only 9e73d3e..HEAD`（本任务提交 fb574cf 相对其父提交；17 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/controller/IotMetricController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/dto/IotMetricCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/dto/IotMetricGenerateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/dto/IotMetricStatusForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/dto/IotMetricUpdateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/entity/IotMetric.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/event/MetricStatusChangedEvent.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/mapper/IotMetricCodeMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/mapper/IotMetricCodeRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/mapper/IotMetricMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/service/IotMetricService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/service/impl/IotMetricServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/metric/config/IotMetricMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/metric/service/impl/IotMetricServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/MetricRedlineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/RedlineFixture.java
resources/sql/upgrade/20261122_iot_metric_domain.sql
```

全落声明范围（metric/** 新包 12 文件 + test iot/** 4 文件——其中 redline 两文件为 brief 明文授权例外「红线套件增补指标域用例」+ 升级脚本 1 文件）。未触碰 frontend/、ingest/、dataitem 域（只读）、dataperm 组件本体、workitems 库（本报告除外）、TASK 详情。

---

## 3. 偏离计划之处

1. **pypdf 提取对齐**：§3.1 p22 指标段实际 3 行接口（list / CRUD / changeStatus）——brief 枚举本批交付七端点（list/detail/add/generate/edit/changeStatus/remove）：detail 照 0021 点位域形态补详情端点（brief 五权限码含 query，点位域同款）；generate 为 brief 裁决新增（§3.1 无此行，权限复用 iot:metric:add——生成是新增的批量形态）；changeStatus 为 §3.1 独立行（权限 iot:metric:edit 与 §3.1 权限列一致）。
2. **创建侧数据权限防线超出 brief 字面枚举**：brief 写路径防线枚举「update/remove/status 照 0021 修订轮范式」，本批另对 add/generate 加「目标实例须 ∈ 可见集」防线（403「无权在该实例下创建指标」）——照 0012 修订轮 I-1「子级创建的父实例必须 ∈ 可见集」同型：指标创建锚点为必填 instance_id，向不可见实例写指标是同类写原语穿透 TC-0002 红线路径；dev 冒烟 S8d/S8e 实证（受限用户向不可见实例 403、向可见实例 200 不误伤）。登记请协调者知悉，如认越界可修订轮回退。
3. **冒烟 S7a 暴露并修复一处主码缺陷**：创建表单 `instanceId` 原挂 @NotNull（bean 校验）——批量生成清单项按 brief 裁决「instanceId 以 query 入参覆盖项内携带值」可不携带，bean 校验先行拦截导致 generate 恒 400。修复：@NotNull 移除、单点新增的必填性下沉服务层 `resolveInstance`（400「归属实例不能为空（统计对象范围必填）」文案不变，单测 testCreate_InstanceValidation 语义不变）——「为什么现在对了」：单点新增必填性校验仍在（服务层双保险），批量生成项由 query 入参统一覆盖（0011 templateId「入参为准」同款口径）。终码已复跑全量（§2.1）。
4. **generateMetrics 防线顺序**：目标实例 resolve+创建侧防线前移至表单内容校验之前（「存在性 → 数据权限 → 业务校验」范式顺序，红线用例 3 的修复，见 §2.1 红转绿 3）。
5. **冒烟轮次修正**：首轮冒烟 S7a 因偏离 3 缺陷中止（此前的 S2-S6b 产生 3 行冒烟指标），修复后以 SQL DELETE 清理 3 行残留重跑全序列（终轮全绿输出已贴回）；脚本修正为 UTF-8 输出落盘（首轮 Git Bash 控制台中文乱码为终端解码问题、非应用缺陷，落盘文件读取正常）。
6. **批量生成单批上限 500**：brief 未规定上限，照 0011 批量保存范式（ITEM_BATCH_MAX=500）落 `GENERATE_BATCH_MAX=500` 并入守卫用例——范式对齐项，非偏离。

## 4. 未做项与原因

1. **「模板自动生成」**：brief 裁决占位口径——模板域只有数据项模板、指标模板载体未定义，本期批量生成实现为「按请求传入指标清单批量生成」（同 0011 批量保存范式，整批事务逐项校验）；「按实例已绑定模板的指标清单自动生成」留待指标模板载体定义（REQ-0024 后续裁决 / 0024 发布链）。
2. **上看板标记（TD-0039/ADR-0011）**：逐字 DDL 无该列——占位登记（REQ-0029 上看板标记随展示端任务落，brief 明示「报告遗留」）。
3. **src_type 级联标记（TD-0027/ADR-0009）**：逐字 DDL 无该列——本期来源仅数据项（source_item_id），上游指标级联随月年逐级聚合链（FEAT-0012/REQ-0025）与上游 DDL 修订落；TASK 详情「src_type 级联标记」以本占位口径承载（实体/脚本头注均登记）。
4. **比值型分母 0/无数据的「无值留痕不写 0/无穷大」（TC-0014 步骤 4）**：计算侧语义，归 FEAT-0012/REQ-0025——本批交付定义侧齐备性约束（分子分母齐备且同实例存在启用），REQ-0024 验收 2 的算法断言随计算引擎交付验证。
5. **指标引用边进数据项依赖图（TASK-0036 impact API 的指标关联占位）**：brief 改动范围明示「不碰 dataitem 域（只读）」，iot_data_item_ref 不含指标边——0036 报告登记的该占位延续，指标引用边随计算引擎依赖图任务落。
6. **口径文字界面展示（REQ-0024 验收 3 的「或界面明确展示口径文字」分支）**：归前端（FEAT-0014 页面任务），后端已按「未标注不可保存」承载应用层强制。

## 5. 遗留问题

1. **乐观锁并发编辑提示（TC-0014 步骤 6）**：iot_metric 逐字 DDL 无 version 列，框架层乐观锁无承载——与其他 iot 域（模板/实例/点位）同款现状（REQ-0024/FEAT-0014「并发=乐观锁+留痕」的一致性口径随上游 DDL 修订统一处理）；操作留痕由 @OperateLog 框架日志承载。
2. **启停事件消费方**：MetricStatusChangedEvent 只发布不建监听（照 0013 范式）——停用停调度/启用恢复调度的真实消费归 FEAT-0013/TASK-0045，端到端调度验证归联调；本批 S9 落库断言 + 日志留痕已实证发布路径。
3. **停用「重新发布后对接入生效」明示**：点位域 0023 有 effectNote 明示（配置发布链联动）；指标域 changeStatus 未挂发布链提示（本期配置发布归 0024）——调度语义以「停用=停调度结果保留」为准（REQ-0024 口径），发布联动提示随 0024 交付时对齐。
4. **dev 库冒烟残留**（不影响仓库与提交）：role 6 的 4058-4062 五菜单授权、smoke_perm_a_t0006 密码哈希与 admin 同值（0021 轮既有）、iot_metric 逻辑删行 6 条（T43_SMOKE_*/GEN_T43_*，编码归档占用——同码不可复用为既有纪律）、INSTANCE 12 的数据项 9-12 被引用关系（无写路径改动）。后续任务冒烟可复用，不需要时手工清理。
5. **指标列表 dept 分支的可见集缓存失效挂点**：指标 dept_id 由本服务维护、无改绑事件源——与点位域同款窗口口径（TTL 内改绑不即时失效，TASK-0006 缓存设计覆盖），登记待 0006 评审确认（0021 报告遗留 5 同款）。

## 6. 结论

五条验收全部通过；TASK-0043 brief 全部交付项完成（DDL+实体、七端点（list/detail/add/generate/edit/changeStatus/remove）、口径强制标注（未标注不可保存）、比值型分子分母齐备约束 + 同实例存在启用 4001、非比值型 ratio_* 置 NULL、calendar_type 锁 1（班次 2 拒）、编码不入字典（ADR-0014）、启停停调度结果保留语义 + MetricStatusChangedEvent 占位发布、批量生成按请求清单整批事务（「模板自动生成」占位登记）、数据权限裁剪（dept_id IN 部门集 OR instance_id IN 可见集，create_by 分支不适用资产类）+ 写路径防线（update/changeStatus/remove 照 0021 修订轮范式 + 创建侧防线照 0012 I-1 同型）、红线套件增补指标域用例、菜单 4058-4062、单测/冒烟/prod 回归）。TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。
