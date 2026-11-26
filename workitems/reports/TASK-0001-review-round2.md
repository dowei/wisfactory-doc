# TASK-0001 修订轮复评报告（fresh reviewer，scoped）

- 复评对象：修订提交 `98c155e`（2 文件，74 insertions / 1 deletion）
- 复评范围：仅判定原评审 Important 1 是否修复、修复 diff 有无新引入缺陷；原 4 条 Minor 已由协调者记 deferred，不在本轮判定范围
- 复评方式：读原评审报告 / fixdiff / 实施报告修订轮一节，另对仓库现状做了独立核验（源码逐字比对、实体字段全集核对、MP 更新策略与 MetaObjectHandler 配置核对、本地独立复跑两个租户测试类）

## 结论：Important 已修复

原发现 1（PUT /tenant 可绕过 `iot:tenant:status` 改写 status 且不受 0/1 校验）已消除。判定依据四层，全部独立核验通过：

1. **代码层（任何情况下不得改动 status）**：HEAD=98c155e 的 `updateTenant`（`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java:82-92`）与 fixdiff 逐字一致——不再 `updateById(请求实体)`，改为白名单新建 `upd` 实体。status 从未被拷贝进 `upd`；实体所有字段均为包装类型（无原生类型默认值漏进 SET）；仓库无全局 update-strategy 覆盖（application.yml:133-148 仅 id-type/logic-delete 配置，SysTenant 无 `@TableField(updateStrategy)`），MP 默认 NOT_NULL 策略下 null 字段不进 UPDATE SET——编辑路径**物理上不含 status 列**，与请求携带何种 status（0/1/5/任意）无关，且严格强于「回写库内同值」（无并发启停覆盖窗口），符合协调者「status 以库内值为准（忽略请求字段）」的裁决口径。
2. **唯一启停入口不变**：status 的唯一写路径仍是 `updateTenantStatus`（同文件 97-109 行，`iot:tenant:status` + 0/1 校验，未改动），既有 0/1 校验用例原样保留。
3. **测试层**：新增 `testUpdateTenant_RequestStatusNeverWritten` 对捕获的更新实体断言 status 为 null（覆盖合法 0 与非法 5 两请求）；实施报告红灯输出（`expected: <null> but was: <0>`，对修订前实现失败）证明该测试真实复现缺陷而非恒真断言。本复评独立复跑 `mvn test -pl ot-admin -Dtest='SysTenantServiceImplTest,SysTenantControllerTest'`：13/13 + 12/12 全绿，与报告 R2/R3 一致。
4. **端到端佐证**：实施报告 R4 dev 冒烟——PUT 携带 status=0 与非法 status=5 均回读 status=1，同时 tenantName/pointQuota 正常更新、updateTime 前移（证明编辑路径本身未被破坏）；专用启停接口 PUT /tenant/4/status 仍生效。

## 新发现：无

修复 diff 核对未发现新引入缺陷：

- **白名单完整性**：实体字段全集 = SysTenant 十字段（id/tenantCode/tenantName/status/pointQuota/storageQuotaGb/expireTime/contactName/contactPhone/remark）+ BaseEntity 五审计字段（createTime/createBy/updateTime/updateBy/deleted）。白名单取 id（行键，取自 `existing` 而非请求，客户端亦无法借此改写主键位次）、tenantCode（经不可改校验后与库内同值，no-op）、其余 7 个可编辑业务字段——**无遗漏、无夹带**，恰好排除 status 与全部审计/逻辑删字段。
- **事务边界**：`@Transactional(rollbackFor=Exception.class)` 未动；白名单构建位于同方法同事务内的存在性/编码校验之后；`@TableLogic` 的 `deleted=0` WHERE 语义不受影响。
- **审计语义**：`upd` 的 updateBy/updateTime 为 null → `strictUpdateFill`（`backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java:66-67`）按当前登录者回填，审计反而更正确；审计五字段 mass assignment 副产物修复有专测（`testUpdateTenant_AuditFieldsNotUpdatable`），且协调者裁决已明确允许。
- **测试断言未放松**：既有 11 例 service 用例 + 12 例 controller 用例逐一对现行文件核对，断言原样未改；两处 status 相关既有用例（仅 0/1、0/1 均放行且不夹带业务字段）保留完整。
- **原 Minor 未恶化**：list/remove/updateTenantStatus 三个入口与修复前逐字相同（pageSize 无上限、DELETE 解析、默认租户护栏均维持 deferred 原状）；Minor 2 的 PUT 路径被顺带修复，POST 路径未动、deferred 口径不变。

附注（非发现，仅供记录）：编辑请求无法将 remark/expireTime 等可空字段清空为 null——此为 MP updateById NOT_NULL 策系的既有行为，修复前后一致，非本轮引入；`upd` 每次回写 tenant_code 同值属注释中明示的 no-op 设计，无功能影响。

## 一句判定理由

编辑路径的白名单更新使 status 列在物理上不可能进入 UPDATE SET（代码、单测红灯转绿、冒烟三重证据一致，且本复评独立复跑全绿），启停语义唯一入口仍是有独立权限点与 0/1 校验的专用接口，修复本身未引入白名单遗漏、事务边界或断言放松等新缺陷。
