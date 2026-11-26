# TASK-0008 diff 评审报告（fresh reviewer）

- 评审对象：`reports/TASK-0008-diff.txt`（BASE `88caf08` → `c1b2615`，13 文件）
- 依据：L1-execute.md §13 diff 评审口径；评审 brief、实施 brief（DDL 逐字 / 接口契约 / 路径裁决）、TASK-0008 详情、实施报告、conventions.md CONV-01
- 结论：**Critical 0 / Important 1 / Minor 3**

## 一致性核对（通过项，简列证据）

- **DDL 逐字性**：`resources/sql/upgrade/20261121_iot_data_dict.sql` 的 CREATE TABLE 与实施 brief 附录 A 六逐字一致（仅按 brief 要求加 `IF NOT EXISTS`），`uk_tenant_data_code` / `idx_tenant_category` 在位。
- **路径迁移完整性**：`/tenant` → `/iot/tenant` 无漏改点——全仓 grep（backend main/test Java、frontend/src、resources/sql）旧扁平路径零残留；SysTenantController 注解与 javadoc、SysTenantService / SysTenantServiceImpl 注释、SysTenantControllerTest 全部 DisplayName 与请求路径、tenant-api.ts 六处常量均已迁移；报告冒烟 S2b 实证旧路径失效（业务码 404）。
- **4002 双道闸**：`insertDict` 活跃行预检（data_code 精确匹配 + 插件注入租户条件 + @TableLogic 过滤）+ `DuplicateKeyException` 兜底，均映射 4002，照 SysTenantServiceImpl 范式；禁用/逻辑删行占码场景由兜底覆盖。
- **编辑锁**：data_code / data_type 不一致或未携带一律 PARAM_ERROR 拒绝；白名单更新排除 status / tenant_id / createBy/createTime/updateBy/updateTime / deleted，服务单测逐字段断言。
- **DELETE=禁用语义**：`disableDict` 仅置 status=0，不走 deleteById/deleteByIds（单测显式 verify never），可重复禁用幂等；@OperateLog 记 UPDATE 与「不冒充删除」口径自洽（申报偏离，接受）。
- **菜单种子**：4018-4022 恰五行（menu_type=3、parent_id=4008、列布局与 20261121_sys_tenant.sql 4002 段既有范式一致），不含 iot:dict:import；sys_role_menu 授 role_id=1 五行；守卫测试 `SaPermissionRegistryConsistencyTest.collectRegistryPerms` 扫 resources/sql 全目录（含 upgrade/、排除 backup/archive），新脚本确在守卫覆盖内。
- **租户隔离**：insertDict 服务层会话覆盖 tenantId；Mapper 空 BaseMapper 无旁路；`OtTenantLineHandler` 以 iot_ 前缀规则白名单式覆盖（fail-closed），SELECT/UPDATE/DELETE 注入 WHERE、INSERT 补列；报告冒烟 S9/S10 四证据（list / GET / 跨租户禁用拒）。
- **CONV-01**：iot_data_dict 独立成域，未触 sys_dict 任何代码；控制器 / 实体 / SQL 三处注明两类字典不混用。
- **改动范围**：13 文件独立核对全部落 brief 声明内，无越界。

## 发现清单

### Important

1. **[Important] 新增路径 mass assignment：`IotDataDictServiceImpl.insertDict`（backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java，`this.save(dict)` 直落请求实体）**——POST 请求体可直接携带 `status`（无校验注解，0/2/99 原样落库）与 BaseEntity 的 `deleted`（@TableLogic，传 1 即落库），deleted=1 会造成全 API 不可见的幽灵行**永久占用 uk_tenant_data_code**（编码永不复用 + 无物理/逻辑删路径 = 该编码不可见地报废且无法经 API 排查），status 直传则违反实体 javadoc 自我声明的「写侧仅 DELETE 禁用语义可置 0」不变量；updateDict 的白名单纪律未延伸到 insert（MetaObjectHandler `strictInsertFill` 仅填空值，客户端携带的 createBy/createTime 亦原样落库，同一根因）。SysTenant 范式同样存在 status 直传，但本实体注释已立不变量且 brief 点名 mass assignment 为评审重点——应照 updateDict 方式白名单构建新增实体（new IotDataDict() + 五个业务字段 + tenantId 会话覆盖，status 强制 1、deleted 不携带）。

### Minor

1. **[Minor] 分页无上限：`IotDataDictController.list` + `MybatisPlusConfig`（backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java:38）**——PaginationInnerInterceptor 未配 maxLimit，pageSize=-1（MyBatis-Plus 语义为不加 LIMIT）或超大值可放大查询；沿 SysTenant 既有范式与既有插件配置，非本 diff 引入，记横切 deferred。
2. **[Minor] data_code 字符集未约束：`IotDataDict.dataCode`（backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/IotDataDict.java）**——仅 @NotBlank+@Size(64)，无大写+下划线 @Pattern；实施报告遗留 2 已如实申报，收紧前须先在 REQ/CONV 立规则，维持 deferred。
3. **[Minor] brief TDD 清单「租户隔离（A 建的 B 看不见——插件注入）」未落单测**——standalone 单测无法模拟 TenantContext，以 dev 冒烟 S9/S10 替代并已在报告未做项 4 申报；覆盖口径偏差成立但证据在冒烟层，记 deferred。

## 评级依据说明

- Important 1 进 L1-execute §12 退回路径（修复面小：insertDict 白名单构建 + 补一条「POST 携带 status/deleted 不落库」单测）。
- Minor 3 记 TASK `任务结果` deferred（收尾进 LOG），不进修订轮。
