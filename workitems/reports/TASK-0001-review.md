# TASK-0001 diff 评审报告（fresh reviewer）

- 评审对象：`TASK-0001-diff.txt`（BASE 35d22f6 → HEAD 8d95fb2，9 文件，越界反查已由协调者完成）
- 依据：`L1-execute.md` §13 diff 评审口径 + 实施 brief 接口契约 + TASK-0001 验收标准
- 评审输入：diff 包（已完整读）、TASK-0001-brief.md、tasks/TASK-0001.md、TASK-0001-report.md；为验证 diff 假设另读入 BaseEntity / ResultCode / SysDictController（brief 指定模板）/ MybatisPlusConfig（MetaObjectHandler）/ GlobalExceptionHandler / SqlLikeUtils 的现行源码

## 契约一致性核对（§13 轴 1，全部通过）

| 契约点 | 结论 |
| --- | --- |
| DDL 逐字性 | 通过——20261121_sys_tenant.sql 建表段与 brief 附录 A DDL 逐列逐键一致，仅按 brief 授权加 `IF NOT EXISTS` |
| 4002 语义 | 通过——`DUPLICATE_CODE(4002, "编码重复")` 为唯一追加枚举，全仓无码冲突；语义取通用「编码重复」；活跃行预检 + `DuplicateKeyException` 兜底双道闸均映射 4002（冒烟步骤 2/9 实证） |
| tenant_code 不可改 | 通过——updateTenant 先查库，请求编码缺失或不一致即 PARAM_ERROR 拒绝且不触 update |
| 逻辑删后编码不复用 | 通过——BaseEntity.deleted 确有 `@TableLogic`；兜底闸覆盖已删行占码；冒烟步骤 9 与 DB 佐证一致 |
| 权限码 iot:tenant:* | 通过——六接口逐一对应 list/query/add/edit/remove/status；存量守卫 SaPermissionRegistryConsistencyTest 强制 SQL 登记 |
| 菜单种子号段与幂等 | 通过——sys_menu 4000-4007（4000 目录/4001 页面/4002-4007 按钮）、module_id=4、sys_role_menu 授 role 1，全 INSERT IGNORE，验收 2 两次连跑 DONE |
| 默认租户种子 | 通过——`INSERT IGNORE INTO sys_tenant (id,tenant_code,tenant_name,status) VALUES (1,'default','默认租户',1)` 与 brief 逐字一致 |
| 其余 brief 条款 | 通过——写方法均 `@Transactional(rollbackFor=Exception.class)`；方法命名符合 selectXxxList/insertXxx/updateXxx/deleteXxxById 模式；TDD 覆盖面（新建成功/双场景 4002/改码被拒/启停仅 0/1/配额写入/路由/权限注解/参数校验）与 brief 要求吻合 |

## 发现清单（§13 轴 2 + 轴 1 偏差）

### Important

1. **编辑接口可静默改写 status，且不受 0/1 校验，绕过「启停独立权限点」**
   - 出处：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java` updateTenant（约 67-76 行，`return this.updateById(tenant)`）；对应 Controller `PUT /tenant`（`SysTenantController.java` edit）。
   - 说明：brief 裁决 5 明确「启停独立接口与独立权限点」，且配额字段是仅列名的随行字段；但 PUT /tenant 请求体携带任意非空 `status` 会经 `updateById` 直写 DB——持有 `iot:tenant:edit` 而无 `iot:tenant:status` 的调用方可启停租户，且 `status=2/-1/99` 等越界值不经 updateTenantStatus 的 0/1 校验即落库（DDL 语义 status 仅 0/1）。建议 updateTenant 剥离/校验 status 后再 update。

### Minor

2. **POST/PUT 接受客户端 JSON 里的 id 与审计五字段（mass assignment）**
   - 出处：`SysTenantController.java` add/edit（约 56-71 行，`@RequestBody SysTenant` 全量反序列化）+ `SysTenantServiceImpl.java` insertTenant/updateTenant。
   - 说明：MetaObjectHandler 为 strict fill（仅补 null），客户端自带的 createBy/createTime/updateBy/updateTime 会原样进 INSERT/UPDATE SET；POST 带 id 可显式指定位次，其主键冲突还会被 `DuplicateKeyException` 兜底误报成 4002「租户编码已存在」。系 SysDictController 模板既有范式、需已鉴权操作者，记 deferred。

3. **分页 pageSize 无上限**
   - 出处：`SysTenantController.java` list（约 38-46 行，`new Page<>(pageNum, pageSize)` 未设 cap；分页拦截器亦未配 maxLimit）。
   - 说明：pageSize 可传任意大值一次拉全表；与模板一致属全库既有形态，评审口径点名分页上限，记 deferred。

4. **DELETE /tenant/{ids} 对非数字/空段抛 NumberFormatException → 500「系统繁忙」**
   - 出处：`SysTenantController.java` remove（约 82-92 行，`Long::parseLong` 未预校验，`"1,,2"`、`"abc"`、空串均触雷）。
   - 说明：有 GlobalExceptionHandler 兜底不泄栈，但报错语义失真；与模板一致，记 deferred。

5. **内置默认租户（id=1, default）无删除/禁用护栏**
   - 出处：`SysTenantServiceImpl.java` updateTenantStatus / deleteTenantByIds（均无 id=1 拦截）。
   - 说明：持 iot:tenant:remove/status 者可逻辑删或禁用 TASK-0002 登录注入所依赖的种子行，且按「编码不复用」纪律删除后不可恢复；brief 未要求护栏，属边界设计缺口，记 deferred。

## 结论

- 无 Critical；Important 1 条（发现 1）按 §13 进退回路径；Minor 4 条收尾记 TASK `任务结果` deferred。
- 契约七点全部通过；实施报告五项验收输出与 diff 交叉印证无矛盾（报告所载偏离均为环境/存量缺陷，不在本 diff 范围内）。
