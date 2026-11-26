# TASK-0002 实施报告（sys_user/dept/role 补 tenant_id 与登录租户注入）

- 执行者：implementer（subagent）
- BASE：`04fc3a5`（主仓库根 HEAD）；提交：`3f3b3c7`（main，注明含中断恢复续做）
- 总体结论：**5 条验收全部通过**（1063 例单测全绿 / 升级脚本幂等 / dev 冒烟全序列 / prod 真实启动冒烟 / 改动范围零越界）。
- 执行说明：本任务于 2026-11-21 15:20 前后因执行中断一次，经协调者指示恢复续做；恢复后的盘点结论见「偏离计划之处」第 1 条。

---

## 0. 中断恢复盘点（恢复执行要求 1）

中断时点：实现主体已落地（工作区含未提交改动），全量 ot-admin 套件尚未跑。恢复后以 `git status/diff` 盘点，逐项对照 brief：

| brief 交付项 | 中断时状态 | 恢复后处置 |
| --- | --- | --- |
| 升级脚本 `20261121_tenant_id_columns.sql` | 已写完未执行 | 按验收 2 连跑 + 核对（通过） |
| ot-core：TenantContext / OtTenantLineHandler / MybatisPlusConfig | 已写完，ot-core 95/95 已绿 | 恢复后复跑确认 95/95 |
| ot-admin 实体 8 个 +tenantId、Mapper 旁路方法、服务层覆盖、登录链路、init-admin 三件套、application.yml | 已写完未验证 | 按 TDD 纪律不默认正确：5 个新测试类（42 例）复跑全绿，随后全量 1063 例全绿 |
| 中途发现的设计缺陷 | 中断前未暴露 | 恢复后推演冒烟时发现「租户管理员角色授了 iot:tenant:* 运营方权限」——先改测试（红）再修实现（绿），详见偏离 2 与根因链 R2 |
| 5 条验收命令 | 未执行 | 全部执行，原始输出见下 |

盘点结论：工作区改动与中断前我方工作完全一致，无他人改动混入；`.agents/`、`ingest/`、`resources/contract/`、`backend/scripts/sql-exec.sh` 为任务前已存在的未跟踪文件，未纳入本次提交。

---

## 1. 改动文件清单

提交：`3f3b3c7 feat(tenant): sys_user/dept/role 系补 tenant_id、登录先定租户与多租户拦截器（TASK-0002，含中断恢复续做）`（33 files changed, 1944 insertions / 24 deletions）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-core/src/main/java/net/otlab/core/tenant/TenantContext.java` | 新增 | 会话租户工具：登录成功后写入的 `tenant_id`/`tenant_code` 唯一读取入口；未登录/登录中/旧会话/非Web 上下文一律返回 null |
| `backend/ot-core/src/main/java/net/otlab/core/tenant/OtTenantLineHandler.java` | 新增 | 白名单式（fail-closed）TenantLineHandler：六张 sys_ 表 + `iot_` 前缀注入，其余一律忽略；无租户上下文经 `ignoreTable` 短路跳过全部注入 |
| `backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java` | 修改 | 拦截器链最前加 `TenantLineInnerInterceptor`（先于分页，官方建议顺序） |
| `backend/ot-core/src/test/java/net/otlab/core/tenant/OtTenantLineHandlerTest.java` | 新增 | 9 例：白名单/iot_ 前缀/非白名单忽略/表名归一/取值/无会话/非Web/旧会话/列名 |
| `backend/ot-admin/.../system/entity/SysUser.java` 等 8 个实体 | 修改 | SysUser/SysDept/SysRole/SysDict/SysConfig/SysUserRole/SysRoleMenu/SysRoleDept 增 `tenantId` 字段 |
| `backend/ot-admin/.../system/mapper/SysUserMapper.java` | 修改 | 新增 `selectByTenantAndUsername` / `countByTenantAndUsername`，均 `@InterceptorIgnore(tenantLine="true")`（登录显式查询与 init-admin 精确预检，旁路理由见代码 javadoc） |
| `backend/ot-admin/.../system/mapper/SysRoleMapper.java` | 修改 | 新增 `selectByTenantAndCode` 旁路方法（init-admin 角色供给） |
| `backend/ot-admin/.../system/service/SysUserService.java` | 修改 | 新增 `selectUserByTenantAndUsername`、`insertTenantUser` 接口 |
| `backend/ot-admin/.../system/service/impl/SysUserServiceImpl.java` | 修改 | insertUser/updateUser/importUsers/updateUserRoles 会话租户覆盖；`insertTenantUser`（路径租户例外，DuplicateKey→4002）；`prepareUserForInsert` 抽取共用管线 |
| `backend/ot-admin/.../system/service/impl/SysDeptServiceImpl.java` | 修改 | insertDept/updateDept 会话租户覆盖 |
| `backend/ot-admin/.../system/service/impl/SysRoleServiceImpl.java` | 修改 | insertRole/updateRole 会话租户覆盖；sys_role_menu/sys_role_dept/sys_user_role 关联行从会话注入 |
| `backend/ot-admin/.../system/service/impl/SysDictServiceImpl.java` / `SysConfigServiceImpl.java` | 修改 | 写侧 `setTenantId(null)`（一期仅预留列，不启用租户级覆盖） |
| `backend/ot-admin/.../system/entity/dto/LoginDTO.java` | 修改 | 增可选 `tenantCode` |
| `backend/ot-admin/.../system/controller/LoginController.java` | 修改 | 登录先定租户（显式 tenantCode → `ot.tenant.default-code` 缺省）；禁用/不存在租户拒登录（400「租户不存在或已禁用」）；用户显式 (tenant_id, username) 查询；成功后会话写 `tenant_id`/`tenant_code`（login 与 login/test 同口径） |
| `backend/ot-admin/.../iot/entity/dto/TenantAdminCreateForm.java` | 新增 | init-admin 表单（username/nickname/password @NotBlank；password 前端 SM4 密文；无 tenantId 字段——强制路径租户） |
| `backend/ot-admin/.../iot/service/SysTenantService.java` | 修改 | 新增 `selectEnabledByCode`、`initTenantAdmin` |
| `backend/ot-admin/.../iot/service/impl/SysTenantServiceImpl.java` | 修改 | init-admin：租户存在且 status=1 校验、(tenant_id,username) 双道闸（旁路预检 + DB 兜底→4002）、自动供给租户管理员角色 `tenant_admin_<tenantId>`（排除租户管理子树）、绑定用户 |
| `backend/ot-admin/.../iot/controller/SysTenantController.java` | 修改 | 新增 `POST /tenant/{id}/init-admin`（`@SaCheckPermission("iot:tenant:admin")` + `@OperateLog`） |
| `backend/ot-admin/src/main/resources/application.yml` | 修改 | **仅新增一个键** `ot.tenant.default-code: ${OT_TENANT_DEFAULT_CODE:default}`（brief 裁决 4 授权） |
| `backend/ot-admin/src/test/.../iot/controller/SysTenantControllerTest.java` | 修改 | +4 例（init-admin 注解/路由/参数校验/拒绝透传） |
| `backend/ot-admin/src/test/.../iot/service/impl/SysTenantInitAdminTest.java` | 新增 | 6 例 |
| `backend/ot-admin/src/test/.../system/controller/LoginControllerTenantTest.java` | 新增 | 6 例 |
| `backend/ot-admin/src/test/.../system/service/impl/SysUserTenantInjectionTest.java` | 新增 | 7 例 |
| `backend/ot-admin/src/test/.../system/service/impl/SysDeptRoleTenantInjectionTest.java` | 新增 | 7 例 |
| `resources/sql/upgrade/20261121_tenant_id_columns.sql` | 新增 | 六表 `tenant_id bigint NOT NULL DEFAULT 1`；`uk_username`→`uk_tenant_username (tenant_id, username)`；sys_dict/sys_config 可空预留；菜单 4017（iot:tenant:admin）+ 超管授权；information_schema 判存幂等（MySQL 8 无 ADD COLUMN IF NOT EXISTS，用单行条件化存储过程，兼容 sql-exec.sh 行尾分号切块） |

TDD 顺序：测试先于实现（红灯证据见根因链 R1），恢复后按契约对全部既有改动复测，未默认任何中断前改动正确。

---

## 2. 验收命令原始输出

### 验收 1：`cd backend && mvn test -pl ot-admin`（最终态，实测最终文件）

```
[INFO] Results:
[INFO]
[INFO] Tests run: 1063, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:05 min
[INFO] Finished at: 2026-11-21T22:00:15+08:00
```

逐类汇总（surefire 全量 105 个测试类，全部 0 失败 0 错误；加粗为本次新增/扩展类）：

```
Tests run: 16, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 6.584 s -- in net.otlab.admin.module.iot.controller.SysTenantControllerTest        ← +4 init-admin
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.986 s -- in net.otlab.admin.module.iot.service.impl.SysTenantInitAdminTest        ← 新增
Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.114 s -- in net.otlab.admin.module.iot.service.impl.SysTenantServiceImplTest     ← 存量原样
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.009 s -- in net.otlab.admin.module.system.controller.LoginControllerTenantTest    ← 新增
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.211 s -- in net.otlab.admin.module.system.controller.LoginControllerTest          ← 存量原样
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.037 s -- in net.otlab.admin.module.system.service.impl.SysDeptRoleTenantInjectionTest ← 新增
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.611 s -- in net.otlab.admin.module.system.service.impl.SysUserTenantInjectionTest     ← 新增
Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.132 s -- in net.otlab.core.tenant.OtTenantLineHandlerTest（ot-core 模块，另跑）     ← 新增
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.410 s -- in net.otlab.admin.module.survey.controller.SurveyFileControllerTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.300 s -- in net.otlab.admin.module.survey.controller.SurveyFormAuthControllerTest
Tests run: 21, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.987 s -- in net.otlab.admin.module.survey.controller.SurveyFormControllerTest
Tests run: 21, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.897 s -- in net.otlab.admin.module.survey.controller.SurveyFormDataControllerTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.198 s -- in net.otlab.admin.module.survey.controller.SurveyFormEditLockControllerTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.206 s -- in net.otlab.admin.module.survey.controller.SurveyFormItemControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.084 s -- in net.otlab.admin.module.survey.controller.SurveyFormLogicControllerTest
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.338 s -- in net.otlab.admin.module.survey.controller.SurveyFormPublicControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.096 s -- in net.otlab.admin.module.survey.controller.SurveyFormPublicResultControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.081 s -- in net.otlab.admin.module.survey.controller.SurveyFormSettingControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.088 s -- in net.otlab.admin.module.survey.controller.SurveyFormThemeControllerTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.127 s -- in net.otlab.admin.module.survey.controller.SurveyReportControllerTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.560 s -- in net.otlab.admin.module.survey.controller.SurveySyncControllerTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.266 s -- in net.otlab.admin.module.survey.controller.SurveyTemplateCategoryControllerTest
Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.406 s -- in net.otlab.admin.module.survey.controller.SurveyTemplateControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.183 s -- in net.otlab.admin.module.survey.controller.SurveyThemeCategoryControllerTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.200 s -- in net.otlab.admin.module.survey.controller.SurveyThemeControllerTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.261 s -- in net.otlab.admin.module.survey.controller.SurveyWebhookControllerTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.247 s -- in net.otlab.admin.module.survey.controller.SurveyWxNotifyControllerTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.259 s -- in net.otlab.admin.module.survey.mapper.SurveyFormDataMapperSqlTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.214 s -- in net.otlab.admin.module.survey.service.impl.SurveyFileServiceImplTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.227 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormDataExportSplitTest
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.527 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormDataPhoneVerificationTest
Tests run: 28, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.299 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormDataServiceImplTest
Tests run: 15, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.541 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormEditLockServiceImplTest
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.128 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormItemServiceImplTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.072 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormLogicServiceImplTest
Tests run: 34, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.569 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormServiceImplTest
Tests run: 36, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.314 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormSettingServiceImplTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.081 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormThemeServiceImplTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.152 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormViewCountServiceImplTest
Tests run: 29, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.745 s -- in net.otlab.admin.module.survey.service.impl.SurveyRedpackServiceImplTest
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.071 s -- in net.otlab.admin.module.survey.service.impl.SurveyReportServiceImplTest
Tests run: 15, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.064 s -- in net.otlab.admin.module.survey.service.impl.SurveyReportStatsPlusTest
Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.130 s -- in net.otlab.admin.module.survey.service.impl.SurveyTemplateServiceImplTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.500 s -- in net.otlab.admin.module.survey.service.impl.SurveyWebhookServiceImplTest
Tests run: 19, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.726 s -- in net.otlab.admin.module.survey.service.impl.SurveyWxNotifyServiceImplTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.admin.module.survey.service.support.SurveyAnswerLabelResolverTest
Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.020 s -- in net.otlab.admin.module.survey.service.support.SurveyCrossAnalyzerTest
Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.027 s -- in net.otlab.admin.module.survey.service.support.SurveyDataFilterMatcherTest
Tests run: 14, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.047 s -- in net.otlab.admin.module.survey.service.support.SurveyExamScorerTest
Tests run: 26, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.066 s -- in net.otlab.admin.module.survey.service.support.SurveyExportColumnPlannerTest
Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.035 s -- in net.otlab.admin.module.survey.service.support.SurveyFormItemValidatorTest
Tests run: 16, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.027 s -- in net.otlab.admin.module.survey.service.support.SurveyMatrixAnalyzerTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.010 s -- in net.otlab.admin.module.survey.service.support.SurveyQuotaResolverTest
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.020 s -- in net.otlab.admin.module.survey.service.support.SurveyValidityTextResolverTest
Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.434 s -- in net.otlab.admin.module.survey.service.SurveyFormAuthServiceTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.255 s -- in net.otlab.admin.module.system.config.AccountStatusEnforcementTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.admin.module.system.config.AccountStatusGuardTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.145 s -- in net.otlab.admin.module.system.config.ActuatorAccessFilterTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.005 s -- in net.otlab.admin.module.system.config.ActuatorAccessPolicyTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.062 s -- in net.otlab.admin.module.system.config.SaAnnotationEnforcementTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.565 s -- in net.otlab.admin.module.system.config.SaPermissionRegistryConsistencyTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.130 s -- in net.otlab.admin.module.system.controller.LoginLogControllerTest
Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.618 s -- in net.otlab.admin.module.system.controller.MonitorActuatorControllerTest
Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.210 s -- in net.otlab.admin.module.system.controller.MonitorCacheControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.049 s -- in net.otlab.admin.module.system.controller.PhoneCodeControllerTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.210 s -- in net.otlab.admin.module.system.controller.PublicConfigControllerTest
Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.293 s -- in net.otlab.admin.module.system.controller.SysDeptControllerTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.133 s -- in net.otlab.admin.module.system.controller.SysDictDataControllerTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.108 s -- in net.otlab.admin.module.system.controller.SysFileControllerTest
Tests run: 22, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.534 s -- in net.otlab.admin.module.system.controller.SysMenuControllerTest
Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.284 s -- in net.otlab.admin.module.system.controller.SysModuleControllerTest
Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.378 s -- in net.otlab.admin.module.system.controller.SysRoleControllerTest
Tests run: 18, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 8.984 s -- in net.otlab.admin.module.system.controller.SysUserControllerTest
Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.170 s -- in net.otlab.admin.module.system.controller.SysUserUiConfigControllerTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.064 s -- in net.otlab.admin.module.system.listener.PermissionDeniedAuditListenerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.368 s -- in net.otlab.admin.module.system.runner.DataCryptoMigrateRunnerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.083 s -- in net.otlab.admin.module.system.service.impl.AliyunSmsServiceTest
Tests run: 22, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.629 s -- in net.otlab.admin.module.system.service.impl.CacheMonitorServiceImplTest
Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.127 s -- in net.otlab.admin.module.system.service.impl.DataScopeServiceImplTest
Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.362 s -- in net.otlab.admin.module.system.service.impl.DypnsSmsServiceTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.003 s -- in net.otlab.admin.module.system.service.impl.NoopSmsServiceTest
Tests run: 16, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.130 s -- in net.otlab.admin.module.system.service.impl.PhoneCodeServiceImplTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.104 s -- in net.otlab.admin.module.system.service.impl.SysDeptServiceImplTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.090 s -- in net.otlab.admin.module.system.service.impl.SysDictDataServiceImplTest
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.133 s -- in net.otlab.admin.module.system.service.impl.SysFileServiceImplTest
Tests run: 35, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.223 s -- in net.otlab.admin.module.system.service.impl.SysMenuServiceImplTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.014 s -- in net.otlab.admin.module.system.service.impl.SysUserCryptoQueryTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.085 s -- in net.otlab.admin.module.system.service.impl.SysUserServiceImplKickOutTest
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.108 s -- in net.otlab.admin.module.system.service.impl.SysUserUiConfigServiceImplTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.112 s -- in net.otlab.admin.module.system.service.impl.TencentSmsServiceTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.043 s -- in net.otlab.admin.module.system.storage.LocalStorageServiceTest
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.200 s -- in net.otlab.admin.module.system.storage.MinioStorageServiceTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.004 s -- in net.otlab.admin.module.system.storage.StorageAndSmsConditionalTest
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.004 s -- in net.otlab.admin.module.wechat.config.WxMpServiceConfigTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.159 s -- in net.otlab.admin.module.wechat.controller.WxMpMassSendControllerTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.848 s -- in net.otlab.admin.module.wechat.controller.WxMpPortalControllerTest
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.148 s -- in net.otlab.admin.module.wechat.controller.WxMpWebAuthControllerTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.130 s -- in net.otlab.admin.module.wechat.handler.WxNotifyBindHandlersTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in net.otlab.admin.module.wechat.service.impl.WechatAudiencePayloadColumnConsistencyTest
Tests run: 27, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.255 s -- in net.otlab.admin.module.wechat.service.impl.WxMpFanServiceImplTest
Tests run: 21, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.328 s -- in net.otlab.admin.module.wechat.service.impl.WxMpMassSendServiceImplTest
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.086 s -- in net.otlab.admin.module.wechat.service.impl.WxMpOAuthServiceImplTest
Tests run: 28, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.899 s -- in net.otlab.admin.module.wechat.service.impl.WxMpTemplateSendServiceImplTest
Tests run: 30, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.163 s -- in net.otlab.admin.module.wechat.util.JsonPayloadValidatorTest
Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.admin.module.wechat.util.WxMpTemplateFieldParserTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.core.config.JacksonConfigTest
```

判定：**通过**。1063 例 = 存量 1033（TASK-0001 修订轮基线）+ 新增 30（SysTenantControllerTest 16 含 +4 / SysTenantInitAdminTest 6 / LoginControllerTenantTest 6 / SysUserTenantInjectionTest 7 / SysDeptRoleTenantInjectionTest 7），存量无回归。ot-core 另测 95/95（含新增 OtTenantLineHandlerTest 9 例）。

### 验收 2：升级脚本二次运行幂等 + 表结构核对

第 1 次运行：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261121_tenant_id_columns.sql`（18 条语句：SET NAMES / addcol 过程建删 + 8 次 CALL / swapuk 过程建删 / 菜单与授权 2 条 INSERT）：

```
OK(0)   ← SET NAMES、DROP/CREATE ot_tn_addcol
OK(0) ×8 ← 8 次 CALL：逐表判存 → 列缺失 → ALTER 执行（首次加列，DDL 受影响行数 0）
OK(0)   ← DROP ot_tn_addcol
OK(0)   ← DROP/CREATE ot_tn_swapuk
OK(0)   ← CALL ot_tn_swapuk（uk_username 存在 → DROP；uk_tenant_username 缺失 → ADD）
OK(0)   ← DROP ot_tn_swapuk
OK(1)   ← INSERT IGNORE sys_menu 4017（首插 1 行）
OK(1)   ← INSERT IGNORE sys_role_menu (1,4017)（首插 1 行）
DONE
```

第 2 次（幂等）与第 3 次（程序化复验）输出逐行一致：

```
OK(0)
OK(0)
OK(0)
OK(1) ×8   ← 8 次 CALL：列已存在 → IF 跳过 ALTER；CALL 的 OK(n) 取过程内最后一条
OK(0)        SELECT..INTO 的受影响行数（=1），非重复执行 DDL（无 SQLERROR 为准）
OK(0)
OK(0)
OK(1)      ← CALL ot_tn_swapuk 同上（索引已存在，跳过）
OK(0)
OK(0)
OK(0)      ← INSERT IGNORE 两行已存在 → OK(0)
DONE       ← exit 0，无 SQLERROR
```

核对 1：`SHOW CREATE TABLE sys_user`（uk_tenant_username 在位、uk_username 已删、tenant_id NOT NULL DEFAULT 1）：

```
sys_user	CREATE TABLE `sys_user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '用户ID',
  ...
  `tenant_id` bigint(20) NOT NULL DEFAULT '1' COMMENT '租户ID（登录态/调用上下文注入，禁前端传参；详细设计 §0.3）',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_username` (`tenant_id`,`username`),
  KEY `idx_phone_index` (`phone_index`),
  KEY `idx_email_index` (`email_index`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='用户表'
```

（`SHOW INDEX FROM sys_user` 同步核对：索引清单 = PRIMARY / uk_tenant_username(2 列) / idx_phone_index / idx_email_index，uk_username 已不存在。）

核对 2：`SELECT tenant_id, COUNT(*) FROM sys_user GROUP BY tenant_id`（存量行回填 default 租户）：

```
tenant_id	COUNT(*)
1	3
```

核对 3：菜单 4017 与授权：

```
SELECT id, menu_name, perms FROM sys_menu WHERE id = 4017;
id	menu_name	perms
4017	初始化租户管理员	iot:tenant:admin
SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id = 4017;
role_id	menu_id
1	4017
```

判定：**通过**（两次 DONE 无 SQLERROR；uk_tenant_username 在位；存量 3 用户全部 tenant_id=1；4017/授权在位）。

### 验收 3：dev 冒烟序列（启动：`cd backend && mvn spring-boot:run -pl ot-admin` 后台，就绪判据 `GET /auth/captchaImage` HTTP 200）

启动前端口探测：8098 无 LISTENING（netstat 查证）。就绪：`READY after ~20s (captchaImage HTTP 200)`。密码说明：init-admin 的 password 为前端同口径 SM4 密文（jshell + hutool 5.8.25 预计算 `Passw0rd123` 的密文）；operator 登录走 dev 档 `/auth/login/test`（明文），与生产口 /auth/login 共用同一租户解析与登录链路（LoginControllerTenantTest 已分别覆盖两条路径）。

```
=== S1) admin login (tenantCode 缺省 -> 走 default)   【含 tenantCode 缺省回归】
{"code":200,"data":{"nickname":"管理员","token":"51a99e9b-5715-4157-9de1-26f9803e0d93","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795269717809}

=== S2) POST /tenant 建租户 A (code=tenanta)
{"code":200,"data":null,"message":"操作成功","timestamp":1795269718825}
=== S2b) POST /tenant 建租户 B (code=tenantb)
{"code":200,"data":null,"message":"操作成功","timestamp":1795269719264}
=== S2c) GET /tenant/list 定位 A/B 的 id
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":6,"tenantCode":"tenantb",...,"status":1,...},{"id":5,"tenantCode":"tenanta",...,"status":1,...},{"id":1,"tenantCode":"default",...}],"size":20,"total":3},...}

=== S3) POST /tenant/5/init-admin 建租户 A 管理员 operator
{"code":200,"data":4,"message":"操作成功","timestamp":1795269732061}
=== S3b) POST /tenant/6/init-admin 建租户 B 管理员同名 operator
{"code":200,"data":5,"message":"操作成功","timestamp":1795269732981}   ← 同名两租户并存（A=4，B=5）
=== S3c) 重复 init-admin（同租户同名）-> 4002
{"code":4002,"data":null,"message":"用户名已存在: operator","timestamp":1795269733147}

=== S4) operatorA 登录 (tenantCode=tenanta)
{"code":200,"data":{"nickname":"Operator A","token":"cf1e75c7-d6b0-4dae-b4a3-fdf6a207c99c","tokenTimeout":86400,"userId":4,"username":"operator"},"message":"操作成功","timestamp":1795269733405}
=== S4b) operatorB 登录 (tenantCode=tenantb)
{"code":200,"data":{"nickname":"Operator B","token":"516a0266-126d-4c7f-a7bf-22950f7435d6","tokenTimeout":86400,"userId":5,"username":"operator"},"message":"操作成功","timestamp":1795269733914}

=== S5) operatorA GET /user/list —— 仅见租户 A 的 operator（userId=4 / tenantId=5），不见 B 的（5）与 admin
code= 200 total= 1
[(4, 'operator', 'Operator A', 5)]
=== S5b) operatorB GET /user/list —— 仅见租户 B 的 operator（userId=5 / tenantId=6）
code= 200 total= 1
[(5, 'operator', 'Operator B', 6)]
=== S5c) admin GET /user/list 对照 —— 会话租户 1，仅见 tenant_id=1 的 3 个存量用户
code= 200 total= 3
[(1, 'admin', 1), (3, 'iot_view_t0055', 1), (2, 'ry', 1)]

=== S6) operatorA 直调 POST /tenant/6/init-admin（他租户运营方端点）
{"code":403,"data":null,"message":"无权访问该资源","timestamp":1795269751699}
HTTP_STATUS:200
（HTTP 信封 200、业务码 403：NotPermissionException 由 GlobalExceptionHandler 映射进 Result.code，与框架既有约定一致；403 即按业务码判定）

=== S7) admin 禁用租户 A：PUT /tenant/5/status {"status":0}
{"code":200,"data":null,"message":"操作成功","timestamp":1795269751844}
=== S7b) GET /tenant/5 回读
{"code":200,"data":{"id":5,"tenantCode":"tenanta","tenantName":"Tenant A Ltd","status":0,...},...}

=== S8) operatorA 再登录（租户 A 已禁用）→ 拒
{"code":400,"data":null,"message":"租户不存在或已禁用","timestamp":1795269752095}
=== S8b) operatorB 登录回归（租户 B 未受影响）
code= 200 userId= 5
```

DB 层佐证（冒烟后）：

```
SELECT id, tenant_code, status FROM sys_tenant;
1 default 1 / 2 acme 0 / 4 revtmp 0 / 5 tenanta 0（S7 禁用）/ 6 tenantb 1
SELECT id, username, tenant_id FROM sys_user;
1 admin 1 / 2 ry 1 / 3 iot_view_t0055 1 / 4 operator 5 / 5 operator 6
SELECT id, role_code, tenant_id FROM sys_role WHERE role_code LIKE 'tenant_admin%';
4 tenant_admin_5 5 / 5 tenant_admin_6 6
SELECT user_id, role_id, tenant_id FROM sys_user_role;
(1,1,1)/(2,2,1)/(3,3,1)/(4,4,5)/(5,5,6)
角色菜单隔离佐证：role_id=4（租户 A 管理员角色）授权行 146 条、其中 iot:tenant:* 权限 0 条
```

判定：**通过**（brief 冒烟序列全步符合；TC-0001 步骤 2 与「禁用登录拒」在本任务落地）。

### 验收 4：prod 档真实启动冒烟（BUG-0001 已修，按原命令口径）

打包：`cd backend && mvn package -pl ot-admin -DskipTests` → BUILD SUCCESS，`ot-admin/target/ot-admin-1.0.0.jar`（132,545,395 字节；-DskipTests 因全量测试已在验收 1 覆盖，见偏离 6）。

启动（原命令 + TASK-0001 已确认的 prod fail-fast 必需三密钥环境变量）：

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

启动就绪与 ERROR 核对：

```
READY after ~15s (captchaImage HTTP 200)
final probe: 200
grep -c "ERROR" /tmp/prod_boot.log → 0
2026-11-21T22:05:29.026+08:00  INFO 20820 --- [ot-framework] [main] net.otlab.admin.OtAdminApplication : Started OtAdminApplication in 18.448 seconds (process running for 19.229)
```

prod 真实登录（`POST /auth/login`：图形验证码经 Redis 键 `captcha:code:{uuid}` 预置 JSON 字符串 `"AB12"`，密码为 hutool SM4-ECB 加密 admin123 后 hex 再 base64；不传 tenantCode → 走 default）：

```
=== P1) prod POST /auth/login
{"code":200,"data":{"nickname":"管理员","token":"9f28bcb1-e53b-41b3-aade-535521959739","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795269958900}

=== P2) prod GET /tenant/list（带 token）
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":6,"tenantCode":"tenantb",...},{"id":5,"tenantCode":"tenanta",...,"status":0,...},{"id":1,"tenantCode":"default",...}],"size":10,"total":3},"message":"操作成功","timestamp":1795269959298}
HTTP_STATUS:200

=== P3) prod GET /user/list（会话租户隔离佐证：仅 tenant_id=1 的 3 用户）
code= 200 total= 3
[(1, 'admin', 1), (3, 'iot_view_t0055', 1), (2, 'ry', 1)]
```

判定：**通过**（打包 + prod 启动 0 ERROR + 真实登录 200 + list 200；且会话租户隔离在 prod 同样生效）。

### 验收 5：改动范围核对

命令：`git diff --name-only 04fc3a5..HEAD`（主仓库根；HEAD=3f3b3c7）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/SysTenantController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/dto/TenantAdminCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/SysTenantService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/controller/LoginController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysConfig.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysDept.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysDict.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysRole.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysRoleDept.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysRoleMenu.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysUser.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/SysUserRole.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/entity/dto/LoginDTO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/mapper/SysRoleMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/mapper/SysUserMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/SysUserService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/impl/SysConfigServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/impl/SysDeptServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/impl/SysDictServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/impl/SysRoleServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/impl/SysUserServiceImpl.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/SysTenantControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantInitAdminTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/controller/LoginControllerTenantTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/service/impl/SysDeptRoleTenantInjectionTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/system/service/impl/SysUserTenantInjectionTest.java
backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java
backend/ot-core/src/main/java/net/otlab/core/tenant/OtTenantLineHandler.java
backend/ot-core/src/main/java/net/otlab/core/tenant/TenantContext.java
backend/ot-core/src/test/java/net/otlab/core/tenant/OtTenantLineHandlerTest.java
resources/sql/upgrade/20261121_tenant_id_columns.sql
```

33 个文件全部落在 brief「改动范围」清单内：ot-core `net.otlab.core/**`（新增 tenant 包 + MybatisPlusConfig 最小改动）、system 模块（实体/登录链路/服务层）、iot 模块（仅 SysTenantController 追加 init-admin 及其 service/dto）、`src/test/java/**`、`resources/sql/upgrade/20261121_tenant_id_columns.sql`（新增）；application.yml 仅新增 `ot.tenant.default-code` 一个键（brief 裁决 4 授权）。未碰 frontend/、ingest/、application-prod.yml。**通过**。

---

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 登录页租户编码字段 UI | 前端归 TASK-0056（brief 裁决 2 明确「前端字段归 TASK-0056，本任务只交付 API」） |
| `iot_` 前缀业务表的注入实测 | 当前库尚无 iot_ 表（43 表均为框架表）；`iot_` 规则属「未来任务交付时自动纳入」，handler 逻辑已由单测以假表名（iot_point/iot_data_item）覆盖，真实表交付时随其任务验收 |
| TC-0001 步骤 3-8（点位配额强控/存储吞吐软限/建库触发/到期告警/私有化包全功能冒烟） | FEAT 级联调用例，依赖 TASK-0003/0004/0007 等交付；本任务仅落地其步骤 2（登录隔离）与「禁用登录拒」，brief 已裁决 |
| `uk_role_code` 改 (tenant_id, role_code) | brief 表结构增量仅改 sys_user 唯一约束；sys_role 编码保持全局唯一，租户管理员角色以 `tenant_admin_<tenantId>` 派生编码满足（见遗留 4） |
| 会话内跨租户预检的「框架内通用绕过能力」 | 仅在登录/init-admin 两处确需旁路，各以 `@InterceptorIgnore(tenantLine)` 显式登记在 Mapper 方法上，未提供通用旁口（fail-closed 纪律） |

## 4. 偏离计划之处

1. **中断恢复盘点**（本次执行的既定要求）：见报告第 0 节。恢复后未默认任何中断前改动正确——5 个新测试类 42 例与全量 1063 例全部复跑验证；中断前已完成的 ot-core 95/95 亦复跑确认。
2. **设计缺陷修复（恢复后发现，先红后绿）**：初版 init-admin 给租户管理员角色授予**全量**菜单，其中含 `iot:tenant:*` 运营方权限——推演 brief 冒烟「operatorA 直调 init-admin → 403」时发现该缺陷（运营方端点会被租户管理员调通，403 防护失效）。修复：租户管理员角色授予除「租户管理页（sys_menu 4001）+ 全部 `iot:tenant:*` 按钮行」外的全量菜单。红：`SysTenantInitAdminTest.testInitAdmin_SuccessProvisionsRoleAndUser` FAILURE（授权集含 4001/4002/4017）；绿：6/6。DB 佐证：role_id=4 授权 146 行、iot:tenant:* 为 0 行。dev 冒烟 S6 实测 403。
3. **init-admin 自动供给租户管理员角色**（超出 brief 字面「复用现有用户创建服务」的最小表述，但为 brief 冒烟与 REQ-0001 验收 1 的必要条件）：brief 冒烟要求 operator 登录后 user/list 返回本租户数据，而框架权限完全由角色派生（详细设计 §0.4「租户管理员用角色表达，不扩 user_type 枚举」）——不供给角色则 operator 无任何权限，user/list 必 403，冒烟与 REQ 验收 1 均无法成立。落地口径：每租户一个角色 `tenant_admin_<tenantId>`（uk_role_code 全局唯一的派生规避），data_scope=1（租户内全部，多租户插件已按租户收敛），授予除租户管理子树外全量菜单；幂等（存在即复用 + 并发撞码兜底）。如需收窄租户管理员菜单集，改 `ensureTenantAdminRole` 排除规则即可。
4. **application.yml 新增键**：`ot.tenant.default-code: ${OT_TENANT_DEFAULT_CODE:default}`——brief 裁决 4 预授权的一个键；采用 env 可注入形态以支撑「私有化包注入默认租户后用户无感」。
5. **sys_dict/sys_config 写侧置空 tenant_id**：brief 裁决 3「一期不过滤不启用覆盖」的落地面——除不进拦截器白名单外，写侧一律平台级（防止请求体 mass assignment 预先写入租户级覆盖值，污染二期启用）。
6. **验收 4 的两处命令形态偏离**（均沿 TASK-0001 已确认先例）：a) 打包用 `-DskipTests`（全量测试已由验收 1 在同一代码状态覆盖，无代码变更介于其间）；b) 启动命令补 `SM4_KEY`/`CRYPTO_DATA_KEY`/`CRYPTO_INDEX_KEY`（prod 档 fail-fast 必需，TASK-0001 报告偏离 2 同口径，取 application.yml 的 dev 默认值，非新密钥）。
7. **构建顺序**：`mvn test -pl ot-admin` 前需 `mvn install -pl ot-core`（单模块构建引用 .m2 中的 ot-core；TASK-0001 报告同款问题）。
8. **进程纪律**：dev/prod 实例均先探测端口再启动；两实例完成后均停止（mvn 后台任务被停止时其 java 子进程曾残留监听 8098——该进程为本任务自起实例，已按纪律 taskkill 并复核端口清零）。
9. **git commit 执行**（`3f3b3c7`，main，注明含中断恢复）：验收 5 的范围核对命令比较两个提交点，不提交则命令恒为空；提交内容严格限于上列 33 个文件。

## 5. 遗留问题

1. **存量旧会话窗口**：升级部署前已签发的 Sa-Token 会话没有 `tenant_id`，`TenantContext` 按无租户上下文处理——旧会话在自然过期（24h）或重新登录前查询不过滤（等价于升级前行为，非新增暴露面）。上线建议：部署后统一清 Sa-Token 会话（Redis）或强制重登；若要代码层兜底，可在 AccountStatusGuard 复核时同步校验会话租户与库内 tenant_id 一致（建议后续任务）。
2. **租户管理员菜单集口径**：当前为「除租户管理子树外全量」。REQ/FEAT 未定义租户管理员菜单白名单；若后续定版收窄，需改 `ensureTenantAdminRole` 的排除规则，并对存量租户的 `tenant_admin_*` 角色做一次数据订正。
3. **`uk_role_code` 全局唯一未动**：租户角色编码须全局唯一。`tenant_admin_<tenantId>` 派生规则依赖此约束；二期若放开 (tenant_id, role_code) 联合唯一属表结构增量任务（涉及 01_base 与存量库）。
4. **多租户插件覆盖面依赖 SQL 可解析性**：TenantLineInnerInterceptor 经 JSqlParser 改写 SQL；本项目 MP lambda 语句全覆盖，存量 XML mapper 已核实无 sys_user/dept/role 引用。后续任务新增 XML SQL 引用白名单表时插件同样生效（白名单方向 fail-closed），但复杂 SQL（存储过程/多级子查询）的注入覆盖以 MP 官方能力为界，届时应在新任务验收中实测。
5. **冒烟数据留痕**：租户 tenanta（status=0，S7 后未恢复）/tenantb、operator×2（tenant 5/6）、角色 tenant_admin_5/6 及 146×2 授权行、菜单 4017/授权——均为预期交付数据；重复验收可直接重跑（脚本与流程幂等；tenanta 需先手工置回 status=1 或换新编码）。
6. **登录口文案**：租户「不存在」与「已禁用」对外统一为「租户不存在或已禁用」（brief 裁决 2 原文口径，防租户编码枚举）；如需区分提示需协调者重新裁决。

---

## 附：根因修复链（契约要求「为什么现在对了」）

| # | 现象（红） | 根因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- | --- |
| R1 | ot-core 测试编译失败：找不到 `OtTenantLineHandler`；ot-admin 测试编译失败：`iot.entity.dto` 包不存在、`selectUserByTenantAndUsername`/`insertTenantUser`/`initTenantAdmin`/`selectEnabledByCode` 等符号缺失 | TDD 红灯预期：实现未写 | 实现主代码 | 红→绿符合 TDD 流程；两段红灯原始输出见执行记录 |
| R2 | `SysTenantInitAdminTest.testInitAdmin_SuccessProvisionsRoleAndUser` FAILURE（冒烟推演发现的设计缺陷：租户管理员角色含 iot:tenant:* 运营方权限，S6 的 403 防护会失效） | 初版 `ensureTenantAdminRole` 授予全量菜单未排除运营方子树 | 授予集排除「租户管理页 4001 + iot:tenant:* 按钮行」（按 id + 权限前缀双规则，未来新增 iot:tenant:* 行自动排除） | 测试断言授权集=[1001,1002]（不含 4001/4002/4017）转绿；DB 实测 role_id=4 的 iot:tenant:* 授权 0 行；冒烟 S6 实测 403 |
| R3 | `OtTenantLineHandlerTest.testLoggedInWithoutTenantInSession` expected null but was 5 | sa-token 内存 Dao 在同 JVM 共享：前序用例给 loginId=1 的会话写了 tenant_id=5；`StpUtil.logout()` 无 token 上下文时是 no-op 删不掉既有会话 | 测试改 `StpUtil.logout(1L)` 按 loginId 注销后再登录 | 会话确为新净会话，tenant_id 为空 → null；实现逻辑本身无缺陷（测试隔离问题） |
| R4 | 4 个新测试类 tearDown 抛 `SaTokenContext 上下文尚未初始化` | `@AfterEach` 里先 `clearContext()` 再 `StpUtil.logout()`（logout 需要上下文）；及一个用例循环内已自行清理、@AfterEach 再 logout | 统一改为先 logout 后 clearContext，logout 包 try/catch（上下文已清时忽略） | Sa-Token mock 上下文的生命周期顺序得到保证；断言逻辑未放松 |
| R5 | MP 3.5.17 下 `getTenantId()` 返回 null 会生成 `tenant_id = NULL`（永假）而非跳过——登录路径显式查询会被打穿（javap 反编译 buildTableExpression/processInsert 证实：无 null 分支，唯一跳过路径是 ignoreTable） | 「无会话→跳过注入」不能靠 getTenantId 返回 null 实现 | `ignoreTable` 先判白名单、再判 `TenantContext.getTenantId()==null` 短路；getTenantId 在可达路径上恒非 null | 登录/匿名/旧会话/非Web 四类上下文全表忽略；单测（无会话/旧会话/非Web 三例）+ 冒烟 S4/S5（登录后隔离生效）双向证实 |
| R6 | 跨租户场景的两类查询在会话态下被插件追加会话租户条件而必然失真：a)「已登录重登另一租户」的用户查询；b) init-admin 的 (tenant_id,username) 预检与租户管理员角色查询（会话租户 1 ≠ 路径租户 B） | 白名单表会话态查询一律注入会话租户（fail-closed 的固有语义） | 三个 Mapper 方法 `@InterceptorIgnore(tenantLine="true")` 显式旁路 + 显式 (tenant_id, …) 条件；旁路仅限这三处且 javadoc 写明理由 | 旁路是「显式条件替代隐式注入」而非关闭过滤：登录/预检/角色查询条件本身完整包含租户维；单测（A/B 各自命中、同名并存预检 0 计数）+ 冒烟 S3/S3b/S4 证实 |

## 契约自查

- 未派 subagent；未把 TASK 置「已完成」；未改 workitems 库其他文件（仅本报告）。
- 改动 33 文件全部在 brief 范围声明内（验收 5 原始输出）；frontend/、ingest/、application-prod.yml 未碰；application.yml 仅增授权的一个键。
- 全量 1063 例 + ot-core 95 例 0 失败 0 错误；5 条验收命令全部执行且原始输出贴回；TDD 先红后绿（R1/R2），测试与期望未被反向修改。
- 自起实例（dev/prod）均已停止，8098 端口复核 0 监听。

---

## 附 2：重复派工独立复核附录（第二次接手会话，2026-11-21 22:07–22:27）

**派工快照滞后说明**：本报告正文（§0–§5、根因链、契约自查）由并行会话的恢复执行者完成——提交 `3f3b3c7`（22:06:43）、报告落盘（22:10:43）。协调者随后的重复派工快照仍为「实施到一半、无报告无提交」，导致第二次独立接手。第二次接手经盘点确认实现已完整提交后，**未重复实现、未改任何代码**，做的是全量独立复核；结论与正文完全一致，未发现需修正的缺陷。

复核内容与结果（独立重跑，原始输出要点）：

1. **验收 1 复跑**（先 `mvn install -pl ot-core -DskipTests`，与正文偏离 7 同款前置）：`Tests run: 1063, Failures: 0, Errors: 0, Skipped: 0`，BUILD SUCCESS（22:11:27，与正文 22:00:15 的运行相互独立）；另跑 `mvn test -pl ot-core` → `Tests run: 95, Failures: 0, Errors: 0, Skipped: 0`。
2. **验收 2 复跑**：升级脚本两次运行输出逐行一致、`DONE`、无 SQLERROR；`SHOW CREATE TABLE sys_user` 证实 `uk_tenant_username(tenant_id,username)` 在位、`uk_username` 已删、`tenant_id NOT NULL DEFAULT 1`；`GROUP BY tenant_id` → 租户 1 为 3 行（存量回填）、租户 5/6 各 1 行（正文冒烟残料）；菜单 4017 恰 1 行、perms=`iot:tenant:admin`、`sys_role_menu (1,4017)` 在位；8 表 tenant_id 列空/非空属性与 brief 裁决 1 一致。正文对「CALL 的 OK(n) 恒定上报」的解释与第二次观察吻合。
3. **验收 3 复跑**（全新租户 tna=7 / tnb=8，与正文 S1–S8 的 tenanta/tenantb 完全独立）：启动前 8098 无监听 → `Started OtAdminApplication in 12.841 seconds`、启动段 ERROR=0 → admin 缺省租户登录 200 → 建 tna/tnb → init-admin 同名 operator 各建成功（userId 6/7——与既有租户 5/6 的 operator 构成四租户同名并存，均不串号）→ operatorA/B 以 tenantCode=tna/tnb 登录（昵称 Operator A/B 证实显式 (tenant_id,username) 命中各自行）→ user/list 各仅 1 行（tenantId 7/8）→ operatorA 直调 `POST /tenant/8/init-admin` → `{"code":403,"message":"无权访问该资源"}` → admin 禁用 tna → operatorA 再登录 `{"code":400,"message":"租户不存在或已禁用"}` → 不存在编码同文案。403 的库内留痕：`sys_operate_log id=52`（user_id=6，fail_reason=缺少权限：iot:tenant:admin）。收尾按纪律停止自起 dev 实例（mvn 终止后确认残留 java 子进程 PID 28116 并终止），8098 复测无监听。
4. **验收 4 复跑**：`mvn clean package -DskipTests` BUILD SUCCESS（46.9 s）；prod 启动（正文同款原命令 + 三密钥）`Started OtAdminApplication in 18.85 seconds`、`profile "prod"`、启动段 ERROR=0；预置验证码（Redis 键 `captcha:code:task0002a`=`"AB12"`）+ SM4 密文真实登录 200；`GET /tenant/list` HTTP 200、total=5（含 tna status=0）；`/auth/login/test` 在 prod 被拒（500 测试登录接口仅限开发环境使用）。收尾停止自起 prod 实例，8098 无监听。
5. **验收 5 复核**：`git diff --name-only 04fc3a5..HEAD` 输出与正文 §验收 5 的 33 文件清单逐字一致。

对正文的勘误与备注（正文其余内容维持原样）：

- 正文 §0 表格与 §4-1 中「5 个新测试类 **42 例**」计数有误，应为 **30 例**（ot-admin 新增测试方法数：16 含 +4 / 6 / 6 / 7 / 7；§2 逐类统计与「1063 = 1033 + 30」口径正确）。
- 第二次复核新增的租户 tna（id=7，冒烟末已禁用）/tnb（id=8）、operator×2、角色 tenant_admin_7/8、sys_operate_log id 44–53、Redis 验证码键与 token 记录（自然过期）为复核留痕，性质同正文遗留 5。
- 第二次接手未产生新 git 提交（零代码改动，交付物即 `3f3b3c7`）；复核用临时日志已清理，未产出仓库文件。

---

# 修订轮（验收退回一轮：评审 Critical 0 / Important 1 / Minor 8，详见 TASK-0002-review.md）

- 起因：评审 **I-1**——`ensureTenantAdminRole` 自动供给的租户管理员角色超授平台级权限：排除集（4001 + `iot:tenant:*` 前缀）覆盖不到「系统监控 110 子树」（`monitor:online:list` 枚举全平台在线用户 / `monitor:online:forceLogout` 强踢任意用户 / `monitor:cache:remove` 删任意 Redis 键 / `monitor:job:*` / `monitor:actuator:view`）与「日志管理 140 子树」（`system:operateLog:*` / `system:loginLog:*` 平台日志）——违背 REQ-0001 验收 1「该管理员只见本租户数据」。首轮 R2 只断言了 `iot:tenant:*` 为 0 行，未覆盖 monitor/日志侧。
- 协调者裁决（照做）：排除集扩展为「租户管理子树（4001 及其后代）+ 系统监控子树（110 及其后代）+ 日志管理子树（140 及其后代）+ `iot:tenant:*` 前缀」，按菜单树祖先遍历判定（不写死后代 id 清单，消解评审 M-5）；dev 库 tenant_admin_5/6/7/8 四角色存量授权订正；测试补 monitor:*/日志类 = 0 行断言（先红后绿）。
- 修订轮提交：`2d20dd5 fix(iot): 租户管理员角色排除系统监控/日志管理子树，按菜单树祖先遍历判定（TASK-0002 修订轮 / 评审 I-1）`（main 分支）。范围核对 `git show --stat 2d20dd5`：**仅 2 个文件**——`SysTenantServiceImpl.java`（83 行区段改动）+ `SysTenantInitAdminTest.java`（+97/-14），符合「改动限 SysTenantServiceImpl（及必要的测试类）」契约。（附注：`3f3b3c7..2d20dd5` 区间另含协调者侧工件提交 `470d7f5`——ingest 骨架/契约冻结稿/sql-exec 工具，非本任务改动。）

## RR1. 改动内容

| 文件 | 改动 |
| --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java` | 排除判定重写：`isOperatorOnlyMenu`（单行 id/前缀判断）→ `collectOperatorOnlyMenuIds` + `isUnderOperatorSubtree`——逐菜单沿 parent_id **向上判祖先**是否落在 `OPERATOR_ONLY_SUBTREE_ROOTS = {4001, 110, 140}` 三棵运营方专属子树，外加 `iot:tenant:*` 前缀兜底行；fail-closed 口径：父链断裂（父行缺失且非顶层 0）与父链成环的脏数据行「无法证明未落运营方子树」一律排除（宁少授不误授）。不写死后代 id——110/140/4001 子树内未来新增页自动被排除（消解 M-5） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantInitAdminTest.java` | 成功用例夹具补齐系统监控（110/111/112/113/150/168/170/130/132）与日志管理（140/141/142/144/145）子树菜单行，授权集断言维持 `[1001,1002]`（monitor:*/平台日志全部排除）；新增 1 例孤儿行 fail-closed 用例（parent 指向不存在行 → 不授予） |

## RR2. TDD 红转绿记录（先红）

红灯 1（先改测试断言，对修订前实现跑 `mvn test -pl ot-admin -Dtest='SysTenantInitAdminTest'`）：

```
[ERROR] Tests run: 6, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 3.851 s <<< FAILURE! -- in net.otlab.admin.module.iot.service.impl.SysTenantInitAdminTest
[ERROR] net.otlab.admin.module.iot.service.impl.SysTenantInitAdminTest.testInitAdmin_SuccessProvisionsRoleAndUser -- Time elapsed: 0.074 s <<< FAILURE!
org.mockito.exceptions.verification.TooManyActualInvocations:
roleMenuMapper.insert(
    <Capturing argument: SysRoleMenu>
);
Wanted 2 times:
-> at net.otlab.admin.module.iot.service.impl.SysTenantInitAdminTest.testInitAdmin_SuccessProvisionsRoleAndUser(SysTenantInitAdminTest.java:300)
But was 14 times:
-> at net.otlab.admin.module.iot.service.impl.SysTenantServiceImpl.ensureTenantAdminRole(SysTenantServiceImpl.java:255)
```

（期望 2 行 + 4001/4002/4017 + monitor/日志夹具 9 行 = 14——红灯精确落在 I-1 超授本身。）

红灯 2（首次实现取「自根向下遍历」：夹具中 132/145 的父行 130/144 不在菜单集、遍历到不了，仍授予 4 行）：

```
[ERROR] Tests run: 6, Failures: 1 ... TooManyActualInvocations: Wanted 2 times ... But was 4 times
```

按裁决「按菜单树祖先遍历判定」把实现改为**沿 parent_id 向上判祖先**（孤儿/断链/成环 fail-closed 排除），夹具补齐 130/144 中间行使建模与真实树一致，并新增孤儿行专测。绿灯：

```
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.service.impl.SysTenantInitAdminTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

## RR3. 补充验收 1：全量复跑

命令：`mvn install -pl ot-core`（前置，同前）+ `cd backend && mvn test -pl ot-admin`

```
[INFO] Results:
[INFO]
[INFO] Tests run: 1064, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  52.373 s
[INFO] Finished at: 2026-11-21T22:49:46+08:00
```

1064 = 上轮 1063 + 修订轮新增孤儿行 fail-closed 用例 1。存量无回归。

## RR4. 补充验收 3：dev 库四角色存量订正（一次性 SQL，非幂等口径）

先枚举：被排除菜单集 34 个（4001 子树 8 + 110 子树 19 + 140 子树 7，含 iot:tenant:* 前缀行），四角色待删行各 26：

```
WITH RECURSIVE excluded AS (SELECT id FROM sys_menu WHERE id IN (4001, 110, 140) OR perms LIKE 'iot:tenant:%'
  UNION DISTINCT SELECT m.id FROM sys_menu m JOIN excluded e ON m.parent_id = e.id)
SELECT rm.role_id, r.role_code, COUNT(*) AS rows_to_delete FROM sys_role_menu rm
  JOIN sys_role r ON r.id = rm.role_id JOIN excluded x ON rm.menu_id = x.id
  WHERE r.role_code REGEXP '^tenant_admin_[0-9]+$' GROUP BY rm.role_id, r.role_code ORDER BY rm.role_id;

role_id	role_code	rows_to_delete
4	tenant_admin_5	26
5	tenant_admin_6	26
6	tenant_admin_7	26
7	tenant_admin_8	26
```

（删除前总授权 584 = 4×146。）

订正 DELETE（CTE 多表删除一次执行）：

```
WITH RECURSIVE excluded AS (SELECT id FROM sys_menu WHERE id IN (4001, 110, 140) OR perms LIKE 'iot:tenant:%'
  UNION DISTINCT SELECT m.id FROM sys_menu m JOIN excluded e ON m.parent_id = e.id)
DELETE rm FROM sys_role_menu rm
  JOIN sys_role r ON r.id = rm.role_id JOIN excluded x ON rm.menu_id = x.id
  WHERE r.role_code REGEXP '^tenant_admin_[0-9]+$';

OK(104)      ← 4×26 精确命中
DONE
```

订正后核对（每角色 146→120；被排除子树残留 0；被排除类权限串 0）：

```
SELECT rm.role_id, r.role_code, COUNT(*) AS remaining FROM sys_role_menu rm
  JOIN sys_role r ON r.id = rm.role_id WHERE r.role_code REGEXP '^tenant_admin_[0-9]+$' GROUP BY rm.role_id, r.role_code;
role_id	role_code	remaining          ← 146-26=120
4	tenant_admin_5	120
5	tenant_admin_6	120
6	tenant_admin_7	120
7	tenant_admin_8	120

WITH RECURSIVE excluded AS (...) SELECT COUNT(*) AS remaining_excluded_rows FROM sys_role_menu rm
  JOIN sys_role r ON r.id = rm.role_id JOIN excluded x ON rm.menu_id = x.id
  WHERE r.role_code REGEXP '^tenant_admin_[0-9]+$';
remaining_excluded_rows
0

SELECT DISTINCT m.perms FROM sys_role_menu rm JOIN sys_menu m ON m.id = rm.menu_id JOIN sys_role r ON r.id = rm.role_id
  WHERE r.role_code REGEXP '^tenant_admin_[0-9]+$' AND (m.perms LIKE 'monitor:%' OR m.perms LIKE 'system:operateLog%'
  OR m.perms LIKE 'system:loginLog%' OR m.perms LIKE 'iot:tenant:%');
（空结果——0 行）
```

## RR5. 补充验收 2：dev 冒烟复核（走修订后授予路径）

启动：端口 8098 起始无监听（netstat 探证）→ `mvn spring-boot:run -pl ot-admin` 后台 → `READY after ~15s (captchaImage HTTP 200)`。复核租户用**全新租户 tenantc**（id=9）以强制走「新建角色 + 授予」路径（复用旧租户会命中角色复用分支，测不到新排除逻辑）：

```
=== RS1) admin 登录（tenantCode 缺省）
{"code":200,"data":{"nickname":"管理员","token":"740ad2c1-680b-4707-aa71-9e684d2c23e6","tokenTimeout":86399,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795272688347}

=== RS2) POST /tenant 建复核租户 tenantc
{"code":200,"data":null,"message":"操作成功","timestamp":1795272689731}   ← id=9

=== RS3) POST /tenant/9/init-admin 建复核管理员 operator（修订后授予路径）
{"code":200,"data":8,"message":"操作成功","timestamp":1795272691681}

=== RS4) operator 登录（tenantCode=tenantc）
{"code":200,"data":{"nickname":"Operator C","token":"4112dd2c-625a-4911-8100-b0da442b254a","tokenTimeout":86400,"userId":8,"username":"operator"},"message":"操作成功","timestamp":1795272692047}

=== RS5) GET /user/info（operatorC 权限集/菜单响应）→ 接口响应原始判定
code = 200
permissions total = 92
BAD perms = 无                       ← monitor:* / system:operateLog* / system:loginLog* / iot:tenant:* 权限串 0 个
iot:* 权限 = 无
非 iot 权限样例(前12) = ['survey:data:add', 'survey:data:edit', 'survey:data:list', 'survey:data:remove',
                        'survey:form:add', 'survey:form:edit', 'survey:form:list', 'survey:form:query',
                        'survey:form:remove', 'survey:redpack:list', 'survey:redpack:send', 'survey:report:list']
menu rows total = 37
BAD menu rows = 无                   ← 菜单树 37 行中 110/140/4001 两子树 + 租户管理子树行 0 个
menu ids = [1, 2, 3, 4, 5, 22, 27, 32, 33, 38, 43, 48, 53, 57, 100, 120, 2000, 2001, 2003, 2007, 2009, 2010,
            2013, 2017, 2018, 2020, 2022, 4000, 4008, 4009, 4010, 4011, 4012, 4013, 4014, 4015, 4016]

=== RS6) 未误伤对照：operatorC GET /user/list 仍 200 且仅本租户
code = 200 total = 1
[(8, 'operator', 9)]
```

DB 佐证（RS7）——含新角色在内全部 tenant_admin_* 角色被排除子树授权 0 行、每角色 120 行：

```
WITH RECURSIVE excluded AS (...) SELECT r.role_code, COUNT(*) AS excluded_rows FROM sys_role_menu rm
  JOIN sys_role r ON r.id = rm.role_id JOIN excluded x ON rm.menu_id = x.id
  WHERE r.role_code REGEXP '^tenant_admin_[0-9]+$' GROUP BY r.role_code;
（空结果——0 行）

SELECT r.role_code, COUNT(*) AS remaining FROM sys_role_menu rm
  JOIN sys_role r ON r.id = rm.role_id WHERE r.role_code REGEXP '^tenant_admin_[0-9]+$' GROUP BY r.role_code;
role_code	remaining
tenant_admin_5	120
tenant_admin_6	120
tenant_admin_7	120
tenant_admin_8	120
tenant_admin_9	120
```

判定：**补充验收三项全部通过**（全量 1064 绿 / 接口响应与菜单树无 monitor 与日志类权限 / 四角色订正后 DB 核对 0 残留、新角色构造即 0）。

## RR6. 修订轮根因与「为什么现在对了」

| 失败 | 根因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- |
| 红灯 14 行授权 | 排除集只覆盖 4001 + 前缀，110/140 两子树 19+7 行全部落入授予集（评审 I-1） | 排除集扩展为三棵运营方专属子树 + 前缀兜底 | 单测授权集=[1001,1002] 断言绿；DB 订正后四角色 monitor/日志授权 0 行；冒烟 RS5 权限集/菜单响应双 0 |
| 中间红灯 4 行授权 | 首版实现「自根向下遍历」对父行缺失的孤儿链（132←130 缺、145←144 缺）遍历不到，仍授予 | 按裁决改「沿 parent_id 向上判祖先」：命中根即排除；断链/成环 fail-closed 排除；夹具补齐中间行并新增孤儿行专测 | 向上遍历对任何父链可达树根的形态都正确，未来子树内新增页自动排除（M-5 消解）；孤儿/断链行真实库不存在，但一旦出现（父行被删的脏数据）按 fail-closed 宁少授不误授监控/日志权限 |

修订轮契约自查：仅改 `SysTenantServiceImpl` + 其测试类（`git show --stat 2d20dd5`：2 文件，166 insertions/14 deletions）；一个 commit 注明 TASK-0002 修订轮；报告仅追加本节；未派 subagent、未置 TASK 状态、未碰 workitems 库其他文件；冒烟自起实例已停止（mvn 终止后残留 java 子进程 PID 18060 已终止，8098 复核 0 监听）。评审 Minor 8 条（M-1~M-8）按评审结论记 deferred，不进本修订轮。
