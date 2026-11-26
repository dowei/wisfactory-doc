# TASK-0001 实施报告（sys_tenant 租户表与租户管理接口）

- 执行者：implementer（subagent）
- 完成时间：2026-11-21 12:30 前后
- BASE：`35d22f6`，HEAD（提交后）：`8d95fb2`（main 分支）
- 总体结论：**验收 5 条中 4 条完全通过；验收 4 通过但有前提**——打包 jar 自带的 `application-prod.yml` 存在先于本任务的存量缺陷（顶层 `ot:` 键重复，YAML 解析失败），本批无权修改该文件，prod 冒烟改用「外置修正配置」完成（真实 jar + prod 档 + 真库真 Redis，启动无 ERROR、真实登录成功、`GET /tenant/list` 200）。详见验收 4 与遗留问题。

---

## 1. 改动文件清单

提交：`8d95fb2 feat(iot): sys_tenant 租户表与租户管理接口（TASK-0001）`（仅含下列 9 个文件，921 insertions / 1 deletion）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/SysTenant.java` | 新增 | 实体，继承 BaseEntity（审计五字段自动有），`@TableName("sys_tenant")`、`@TableId(type = IdType.AUTO)` |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/mapper/SysTenantMapper.java` | 新增 | `@Mapper`，继承 BaseMapper，空接口 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/SysTenantService.java` | 新增 | 接口继承 IService<SysTenant>，方法命名 selectTenantList/insertTenant/updateTenant/updateTenantStatus/deleteTenantByIds |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java` | 新增 | 重复码双道闸（活跃行预检 + DuplicateKeyException 兜底→4002）；编辑改码拒绝；启停仅 0/1；写方法 `@Transactional(rollbackFor = Exception.class)` |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/SysTenantController.java` | 新增 | `@RequestMapping("/tenant")` 扁平路径，`@SaCheckPermission("iot:tenant:*")` 六接口，写操作 `@OperateLog`，pageNum/pageSize 分页 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImplTest.java` | 新增 | service 单测 11 例 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/SysTenantControllerTest.java` | 新增 | controller 单测 12 例（standalone MockMvc + Mockito） |
| `backend/ot-core/src/main/java/net/otlab/core/result/ResultCode.java` | 修改 | **仅追加** `DUPLICATE_CODE(4002, "编码重复")`（含错误码分段注释） |
| `resources/sql/upgrade/20261121_sys_tenant.sql` | 新增 | 建表（DDL 逐字 + IF NOT EXISTS）+ sys_module id=4 + sys_menu 4000-4007 + 超管授权 + 默认租户 default，全幂等 |

TDD 执行顺序：先写两个测试文件并跑出编译失败（红），再实现主代码转绿；详见验收 1。

---

## 2. 验收命令原始输出

### 验收 1：单测 + 存量回归

命令：`cd backend && mvn test -pl ot-admin`

最终态原始输出（脚本定稿后完整复跑一次，确保测试的是最终文件）：

```
[INFO] Results:
[INFO]
[INFO] Tests run: 1031, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  45.451 s
```

逐类汇总（surefire 全量 101 个测试类，全部 0 失败 0 错误；新增两类加粗）：

```
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.949 s -- in net.otlab.admin.module.iot.controller.SysTenantControllerTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.405 s -- in net.otlab.admin.module.iot.service.impl.SysTenantServiceImplTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.376 s -- in net.otlab.admin.module.survey.controller.SurveyFileControllerTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.316 s -- in net.otlab.admin.module.survey.controller.SurveyFormAuthControllerTest
[INFO] Tests run: 21, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.961 s -- in net.otlab.admin.module.survey.controller.SurveyFormControllerTest
[INFO] Tests run: 21, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.873 s -- in net.otlab.admin.module.survey.controller.SurveyFormDataControllerTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.197 s -- in net.otlab.admin.module.survey.controller.SurveyFormEditLockControllerTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.206 s -- in net.otlab.admin.module.survey.controller.SurveyFormItemControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.082 s -- in net.otlab.admin.module.survey.controller.SurveyFormLogicControllerTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.300 s -- in net.otlab.admin.module.survey.controller.SurveyFormPublicControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.109 s -- in net.otlab.admin.module.survey.controller.SurveyFormPublicResultControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.095 s -- in net.otlab.admin.module.survey.controller.SurveyFormSettingControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.096 s -- in net.otlab.admin.module.survey.controller.SurveyFormThemeControllerTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.137 s -- in net.otlab.admin.module.survey.controller.SurveyReportControllerTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.415 s -- in net.otlab.admin.module.survey.controller.SurveySyncControllerTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.245 s -- in net.otlab.admin.module.survey.controller.SurveyTemplateCategoryControllerTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.369 s -- in net.otlab.admin.module.survey.controller.SurveyTemplateControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.204 s -- in net.otlab.admin.module.survey.controller.SurveyThemeCategoryControllerTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.210 s -- in net.otlab.admin.module.survey.controller.SurveyThemeControllerTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.218 s -- in net.otlab.admin.module.survey.controller.SurveyWebhookControllerTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.220 s -- in net.otlab.admin.module.survey.controller.SurveyWxNotifyControllerTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.276 s -- in net.otlab.admin.module.survey.mapper.SurveyFormDataMapperSqlTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.203 s -- in net.otlab.admin.module.survey.service.impl.SurveyFileServiceImplTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.157 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormDataExportSplitTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.493 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormDataPhoneVerificationTest
[INFO] Tests run: 28, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.283 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormDataServiceImplTest
[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.499 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormEditLockServiceImplTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.125 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormItemServiceImplTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.071 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormLogicServiceImplTest
[INFO] Tests run: 34, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.532 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormServiceImplTest
[INFO] Tests run: 36, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.291 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormSettingServiceImplTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.073 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormThemeServiceImplTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.146 s -- in net.otlab.admin.module.survey.service.impl.SurveyFormViewCountServiceImplTest
[INFO] Tests run: 29, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.758 s -- in net.otlab.admin.module.survey.service.impl.SurveyRedpackServiceImplTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.070 s -- in net.otlab.admin.module.survey.service.impl.SurveyReportServiceImplTest
[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.062 s -- in net.otlab.admin.module.survey.service.impl.SurveyReportStatsPlusTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.237 s -- in net.otlab.admin.module.survey.service.impl.SurveyTemplateServiceImplTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.501 s -- in net.otlab.admin.module.survey.service.impl.SurveyWebhookServiceImplTest
[INFO] Tests run: 19, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.706 s -- in net.otlab.admin.module.survey.service.impl.SurveyWxNotifyServiceImplTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.admin.module.survey.service.support.SurveyAnswerLabelResolverTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.020 s -- in net.otlab.admin.module.survey.service.support.SurveyCrossAnalyzerTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.027 s -- in net.otlab.admin.module.survey.service.support.SurveyDataFilterMatcherTest
[INFO] Tests run: 14, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.047 s -- in net.otlab.admin.module.survey.service.support.SurveyExamScorerTest
[INFO] Tests run: 26, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.066 s -- in net.otlab.admin.module.survey.service.support.SurveyExportColumnPlannerTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.035 s -- in net.otlab.admin.module.survey.service.support.SurveyFormItemValidatorTest
[INFO] Tests run: 16, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.032 s -- in net.otlab.admin.module.survey.service.support.SurveyMatrixAnalyzerTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.009 s -- in net.otlab.admin.module.survey.service.support.SurveyQuotaResolverTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.019 s -- in net.otlab.admin.module.survey.service.support.SurveyValidityTextResolverTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.447 s -- in net.otlab.admin.module.survey.service.SurveyFormAuthServiceTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.273 s -- in net.otlab.admin.module.system.config.AccountStatusEnforcementTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.013 s -- in net.otlab.admin.module.system.config.AccountStatusGuardTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.152 s -- in net.otlab.admin.module.system.config.ActuatorAccessFilterTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.005 s -- in net.otlab.admin.module.system.config.ActuatorAccessPolicyTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.062 s -- in net.otlab.admin.module.system.config.SaAnnotationEnforcementTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.565 s -- in net.otlab.admin.module.system.config.SaPermissionRegistryConsistencyTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.207 s -- in net.otlab.admin.module.system.controller.LoginControllerTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.125 s -- in net.otlab.admin.module.system.controller.LoginLogControllerTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.616 s -- in net.otlab.admin.module.system.controller.MonitorActuatorControllerTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.200 s -- in net.otlab.admin.module.system.controller.MonitorCacheControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.049 s -- in net.otlab.admin.module.system.controller.PhoneCodeControllerTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.210 s -- in net.otlab.admin.module.system.controller.PublicConfigControllerTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.293 s -- in net.otlab.admin.module.system.controller.SysDeptControllerTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.133 s -- in net.otlab.admin.module.system.controller.SysDictDataControllerTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.108 s -- in net.otlab.admin.module.system.controller.SysFileControllerTest
[INFO] Tests run: 22, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.534 s -- in net.otlab.admin.module.system.controller.SysMenuControllerTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.284 s -- in net.otlab.admin.module.system.controller.SysModuleControllerTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.378 s -- in net.otlab.admin.module.system.controller.SysRoleControllerTest
[INFO] Tests run: 18, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 8.984 s -- in net.otlab.admin.module.system.controller.SysUserControllerTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.170 s -- in net.otlab.admin.module.system.controller.SysUserUiConfigControllerTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.064 s -- in net.otlab.admin.module.system.listener.PermissionDeniedAuditListenerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.368 s -- in net.otlab.admin.module.system.runner.DataCryptoMigrateRunnerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.083 s -- in net.otlab.admin.module.system.service.impl.AliyunSmsServiceTest
[INFO] Tests run: 22, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.629 s -- in net.otlab.admin.module.system.service.impl.CacheMonitorServiceImplTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.127 s -- in net.otlab.admin.module.system.service.impl.DataScopeServiceImplTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.362 s -- in net.otlab.admin.module.system.service.impl.DypnsSmsServiceTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.003 s -- in net.otlab.admin.module.system.service.impl.NoopSmsServiceTest
[INFO] Tests run: 16, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.130 s -- in net.otlab.admin.module.system.service.impl.PhoneCodeServiceImplTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.104 s -- in net.otlab.admin.module.system.service.impl.SysDeptServiceImplTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.090 s -- in net.otlab.admin.module.system.service.impl.SysDictDataServiceImplTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.133 s -- in net.otlab.admin.module.system.service.impl.SysFileServiceImplTest
[INFO] Tests run: 35, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.223 s -- in net.otlab.admin.module.system.service.impl.SysMenuServiceImplTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.014 s -- in net.otlab.admin.module.system.service.impl.SysUserCryptoQueryTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.085 s -- in net.otlab.admin.module.system.service.impl.SysUserServiceImplKickOutTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.108 s -- in net.otlab.admin.module.system.service.impl.SysUserUiConfigServiceImplTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.112 s -- in net.otlab.admin.module.system.service.impl.TencentSmsServiceTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.043 s -- in net.otlab.admin.module.system.storage.LocalStorageServiceTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.200 s -- in net.otlab.admin.module.system.storage.MinioStorageServiceTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.004 s -- in net.otlab.admin.module.system.storage.StorageAndSmsConditionalTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.004 s -- in net.otlab.admin.module.wechat.config.WxMpServiceConfigTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.159 s -- in net.otlab.admin.module.wechat.controller.WxMpMassSendControllerTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.848 s -- in net.otlab.admin.module.wechat.controller.WxMpPortalControllerTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.148 s -- in net.otlab.admin.module.wechat.controller.WxMpWebAuthControllerTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.130 s -- in net.otlab.admin.module.wechat.handler.WxNotifyBindHandlersTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in net.otlab.admin.module.wechat.service.impl.WechatAudiencePayloadColumnConsistencyTest
[INFO] Tests run: 27, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.255 s -- in net.otlab.admin.module.wechat.service.impl.WxMpFanServiceImplTest
[INFO] Tests run: 21, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.328 s -- in net.otlab.admin.module.wechat.service.impl.WxMpMassSendServiceImplTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.086 s -- in net.otlab.admin.module.wechat.service.impl.WxMpOAuthServiceImplTest
[INFO] Tests run: 28, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.899 s -- in net.otlab.admin.module.wechat.service.impl.WxMpTemplateSendServiceImplTest
[INFO] Tests run: 30, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.163 s -- in net.otlab.admin.module.wechat.util.JsonPayloadValidatorTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.006 s -- in net.otlab.admin.module.wechat.util.WxMpTemplateFieldParserTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.016 s -- in net.otlab.core.config.JacksonConfigTest
[INFO] Tests run: 1031, Failures: 0, Errors: 0, Skipped: 0
```

判定：**通过**。1031 例 = 存量 1008 + 新增 23（SysTenantController 12 + SysTenantServiceImpl 11），存量无回归。

TDD 先红后绿记录：

1. 红灯 1（实现缺失）：只跑 `mvn test -pl ot-admin -Dtest='SysTenant*'`，编译失败（主代码未写）——
   ```
   [ERROR] /D:/.../iot/controller/SysTenantControllerTest.java:[56,13] 找不到符号
   [ERROR]   符号:   类 SysTenantController
   [ERROR] /D:/.../iot/service/impl/SysTenantServiceImplTest.java:[7,41] 程序包net.otlab.admin.module.iot.entity不存在
   ```
2. 红灯 2（全量守卫拦截）：主代码落地、升级脚本未写时全量跑，1031 例中 1 失败——
   ```
   [ERROR] Failures:
   [ERROR]   SaPermissionRegistryConsistencyTest.everySaCheckPermissionIsRegisteredInSysMenuSql:93
     以下 @SaCheckPermission 权限码未在 resources/sql 的 sys_menu INSERT 中登记
     （新加注解必须同步登记菜单行，参见 BUG-0041/BUG-0046）：
     [iot:tenant:add, iot:tenant:edit, iot:tenant:list, iot:tenant:query, iot:tenant:remove, iot:tenant:status]
   ==> expected: <true> but was: <false>
   [INFO] Tests run: 1031, Failures: 1, Errors: 0, Skipped: 0
   [INFO] BUILD FAILURE
   ```
   根因与修复：该守卫扫描 `@SaCheckPermission` 并要求权限码出现在 resources/sql 的 sys_menu INSERT 中（含 upgrade/ 子目录，`collectRegistryPerms()` 递归走整棵 resources/sql 树、排除 backup/archive）。修复 = 写出升级脚本登记 4002-4007 六个权限行（本来就是本任务交付物），未改任何测试。写脚本后复跑全量转绿。
3. 中间修复 1（测试自身 API 误设，非实现缺陷）：`SaCheckPermission.value()` 在 Sa-Token 1.45 返回 `String[]`，断言按 String 写导致 1 例失败：
   ```
   [ERROR] SysTenantControllerTest.testPermissionAnnotations:88 expected: <iot:tenant:list> but was: <[iot:tenant:list]>
   ```
   修正断言为 `value()[0]`（注解取值本身正确、六权限串逐一断言未放松），复跑 12/12 通过。为什么现在对了：注解声明的数组首元素即权限串，`value()[0]` 与实现 `@SaCheckPermission("iot:tenant:list")` 严格相等。
4. 中间修复 2（ot-core 依赖版本）：`-pl ot-admin` 单模块编译引用 .m2 中的旧 ot-core（无 DUPLICATE_CODE）报「找不到符号」；先 `mvn install -pl ot-core` 安装新枚举后转绿。此为构建顺序问题，不是代码缺陷。

### 验收 2：升级脚本二次运行幂等

命令 1：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261121_sys_tenant.sql`（第 1 次）

```
OK(0)
OK(0)
OK(0)
OK(8)
OK(1)
DONE
```

（5 条语句：CREATE TABLE IF NOT EXISTS 已存在→0；sys_module id=4 已存在→0；sys_menu 8 行已存在→0；sys_role_menu 首次插入 8 行；默认租户首次插入 1 行）

命令 1（第 2 次）：

```
OK(0)
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

两次均 `DONE`，无 SQLERROR。**通过**。

核对命令 2：`bash backend/scripts/sql-exec.sh "SELECT id,tenant_code,tenant_name,status FROM sys_tenant;"`

```
id	tenant_code	tenant_name	status
1	default	默认租户	1
DONE
```

含 id=1 default 行。**通过**。

核对命令 3：`SHOW CREATE TABLE sys_tenant`（节选，uk_tenant_code 存在）：

```
sys_tenant	CREATE TABLE `sys_tenant` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '租户 ID',
  `tenant_code` varchar(32) NOT NULL COMMENT '租户编码（全局唯一，创建后不可改）',
  ...
  UNIQUE KEY `uk_tenant_code` (`tenant_code`),
  KEY `idx_status` (`status`)
)
```

补充核对（菜单/授权/逻辑删行占码）：

```
SELECT id,menu_name,perms FROM sys_menu WHERE id BETWEEN 4000 AND 4007;
id	menu_name	perms
4000	智慧工厂	null
4001	租户管理	null
4002	租户查询	iot:tenant:list
4003	租户详情	iot:tenant:query
4004	租户新增	iot:tenant:add
4005	租户修改	iot:tenant:edit
4006	租户删除	iot:tenant:remove
4007	租户启停	iot:tenant:status
SELECT role_id,menu_id FROM sys_role_menu WHERE menu_id=4000;
role_id	menu_id
1	4000
```

### 验收 3：dev 档接口冒烟

启动：`cd backend && mvn spring-boot:run -pl ot-admin`（后台）。就绪判据：`http://localhost:8098/auth/captchaImage` 返回 HTTP 200（约 12s 就绪）。

登录：dev 档用 `POST /auth/login/test`（LoginController 提供的免验证码/免 SM4 测试登录，prod 档会拒绝）：

```
=== POST /auth/login/test {"username":"admin","password":"admin123"}
{"code":200,"data":{"nickname":"管理员","token":"17e4eb5e-f031-499e-bd8a-80bed56de50c","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795234119162}
```

后续请求头：`Authorization: Bearer <token>`（`sa-token.token-prefix: Bearer`，`token-name: Authorization`）。

CRUD 序列原始输出（9 步全过）：

```
=== 1) POST /tenant 新建 acme (pointQuota=100)
{"code":200,"data":null,"message":"操作成功","timestamp":1795234145266}
HTTP_STATUS:200

=== 2) POST /tenant 同码重建
{"code":4002,"data":null,"message":"租户编码已存在: acme","timestamp":1795234145346}
HTTP_STATUS:200

=== 3) GET /tenant/list?tenantCode=acme   （含 acme，total=1）
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":2,"tenantCode":"acme","tenantName":"Acme Group","status":1,"pointQuota":100,"storageQuotaGb":20,"expireTime":null,"contactName":"Zhang San","contactPhone":"13800000000","remark":"smoke test tenant","createBy":1,"createTime":"2026-11-21 12:09:05","deleted":0,"updateBy":1,"updateTime":"2026-11-21 12:09:05"}],"size":10,"total":1},"message":"操作成功","timestamp":1795234149338}
HTTP_STATUS:200

=== 4) GET /tenant/{id}（id=2）字段齐全（编码/名称/状态/双配额/联系人/电话/备注/审计五字段）
{"code":200,"data":{"id":2,"tenantCode":"acme","tenantName":"Acme Group","status":1,"pointQuota":100,"storageQuotaGb":20,"expireTime":null,"contactName":"Zhang San","contactPhone":"13800000000","remark":"smoke test tenant","createBy":1,"createTime":"2026-11-21 12:09:05","deleted":0,"updateBy":1,"updateTime":"2026-11-21 12:09:05"},"message":"操作成功","timestamp":1795234149506}
HTTP_STATUS:200

=== 5) PUT /tenant 改 tenantCode -> 被拒（400）
{"code":400,"data":null,"message":"租户编码不允许修改","timestamp":1795234156380}
HTTP_STATUS:200

=== 6) PUT /tenant 改 pointQuota=200 -> 200
{"code":200,"data":null,"message":"操作成功","timestamp":1795234156489}
HTTP_STATUS:200
=== 6b) 回读 pointQuota 一致（pointQuota:200）
{"code":200,"data":{"id":2,"tenantCode":"acme","tenantName":"Acme Group","status":1,"pointQuota":200,"storageQuotaGb":20,...},"message":"操作成功","timestamp":1795234156562}

=== 7) PUT /tenant/2/status {"status":0} -> 200
{"code":200,"data":null,"message":"操作成功","timestamp":1795234156659}
HTTP_STATUS:200
=== 7b) 回读 status=0
{"code":200,"data":{"id":2,...,"status":0,"pointQuota":200,...},"message":"操作成功","timestamp":1795234156732}

=== 8) DELETE /tenant/2 -> 200
{"code":200,"data":null,"message":"操作成功","timestamp":1795234162187}
HTTP_STATUS:200

=== 9) DELETE 后同码重建 -> 4002（编码不复用）
{"code":4002,"data":null,"message":"租户编码已存在: acme","timestamp":1795234162283}
HTTP_STATUS:200
```

DB 层佐证（步骤 9 后，逻辑删行仍占用编码）：

```
SELECT id, tenant_code, tenant_name, status, deleted FROM sys_tenant WHERE tenant_code='acme';
id	tenant_code	tenant_name	status	deleted
2	acme	Acme Group	0	1
```

判定：**通过**（1/2/3/4/6/7/8 均 200；2/9 返回 body code=4002；5 返回 body code=400）。

### 验收 4：prod 档真实启动冒烟

**第 1 次尝试（按 brief 原命令）失败——存量配置缺陷，非本批改动所致：**

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root REDIS_HOST=localhost java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod

12:10:18.579 [main] ERROR org.springframework.boot.SpringApplication -- Application run failed
while constructing a mapping
 in 'reader', line 1, column 1:
    mybatis-plus:
    ^
found duplicate key ot
 in 'reader', line 125, column 1:
    ot:
    ^
```

根因（先于本任务存在，文件不在本批改动范围）：`backend/ot-admin/src/main/resources/application-prod.yml` 顶层键 `ot:` 出现两次（94 行 ot.crypto/desensitize 段、125 行 ot.redisson/ip/mail 段），Spring Boot 4 的 snakeyaml 严格模式对重复键直接解析失败 → **该仓库当前 prod 档任何实例都无法启动**（与本次交付的代码无关；`git log` 确认该文件最后改动是 BASE 之前的 `4b91219`）。此外 brief 原命令还缺 prod fail-fast 必需的 `SM4_KEY` / `CRYPTO_DATA_KEY` / `CRYPTO_INDEX_KEY` 三个环境变量（application-prod.yml 中无默认值，缺失即启动失败）。

**第 2 次尝试（外置修正配置完成冒烟意图）**：从打包 jar 中解出 `application.yml` 与 `application-prod.yml`（即 Maven 过滤后的真实打包内容），仅将重复的 `ot:` 两块合并为一块（合并后含 crypto/desensitize/ip/mail/redisson 五个子键，Python yaml 校验通过），放入仓库外的临时目录，经 `--spring.config.location` 指向该目录启动。仓库内任何文件未因此改动。

启动命令（实际）：

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod --spring.config.location=file:<临时目录>/
```

启动原始输出（关键行）：

```
12:14:26.354+08:00  INFO 6084 --- [main] net.otlab.admin.OtAdminApplication : Started OtAdminApplication in 18.511 seconds (process running for 19.257)
```

启动阶段 ERROR 行数：**0**（用 grep -c "ERROR" 核对）。8098 约 12s 就绪。

prod 真实登录（`POST /auth/login`，走图形验证码 + SM4 密文全流程；验证码经 Redis 键 `captcha:code:{uuid}` 预置，密码为 hutool SM4-ECB 加密 admin123 后 hex 再 base64，与前端口径一致）：

```
{"code":200,"data":{"nickname":"管理员","token":"0112faf4-7035-48e6-b3bf-e5364c9d27e1","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795234496881}
HTTP_STATUS:200
```

带 token `GET /tenant/list`（一次 200，返回 default 租户种子行）：

```
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":1,"tenantCode":"default","tenantName":"默认租户","status":1,"pointQuota":null,"storageQuotaGb":null,"expireTime":null,"contactName":null,"contactPhone":null,"remark":null,"createBy":null,"createTime":"2026-11-21 12:04:13","deleted":0,"updateBy":null,"updateTime":"2026-11-21 12:04:13"}],"size":10,"total":1},"message":"操作成功","timestamp":1795234500443}
HTTP_STATUS:200
```

判定：**通过（附前提）**——启动日志无 ERROR、端口就绪、真实登录成功、`GET /tenant/list` 200 均满足；但这是「外置修正配置」下的结论。打包配置自身的重复键缺陷必须在后续任务修复（见遗留问题 1），修复后应按 brief 原命令复跑一次 prod 冒烟。

说明：冒烟过程中产生的 2 条 ERROR 日志（12:14:42 系统异常/操作日志写入失败）来自我第一次故意用错误的验证码值登录（Redis 值格式不合规被 Jackson 反序列化拒绝），属预期内的负向尝试，与启动无关、与租户代码无关。

### 验收 5：改动范围核对

命令：`git diff --name-only 35d22f6..HEAD`（在主仓库根；HEAD=8d95fb2）

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/SysTenantController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/SysTenant.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/mapper/SysTenantMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/SysTenantService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/SysTenantControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImplTest.java
backend/ot-core/src/main/java/net/otlab/core/result/ResultCode.java
resources/sql/upgrade/20261121_sys_tenant.sql
```

9 个文件全部落在 brief「改动范围」清单内，无越界。**通过**。

---

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| TASK 实施步骤 3「联调管理页」 | brief 歧义裁决 1：不在本任务，前端页归 TASK-0056；本任务交付可用接口即视为该步骤在本侧完成 |
| 「禁用租户后登录/上报/对外 API 全拒」端到端验证 | brief 歧义裁决 2：归 TASK-0002 与接入/对外各任务；本任务只交付 status 字段与启停接口。TC-0001 是 FEAT 级联调用例，不作为本任务验收门 |
| 修复 `application-prod.yml` 重复 `ot:` 键 | 文件在 brief 改动范围之外（「application*.yml 均不在本批」「越界即违约」），无权修改；以报告 + 遗留问题呈报，待授权 |
| 修改 brief 命令之外的环境变量 | 未额外改任何 brief 未授权内容；prod 启动仅补了 fail-fast 必需的环境变量值（见偏离 2） |

## 4. 偏离计划之处

1. **数据库环境修复（无仓库文件改动，仅 SQL 数据操作）**。brief 环境事实称「库已由 01/02/03 初始化（43 表）」，实际库存在三处初始化残缺，阻塞验收 2/3，已按 01_base.sql 等权威脚本逐字补种：
   - `sys_menu` 表整表缺失（01_base 建表 + 82 行核心菜单 + 54 行超管授权 + 02 的 30 行问卷菜单/30 行授权 + 03 的 24 行微信菜单/4 行授权全部缺失，`sys_role_menu` 为 0 行）——从 01/02/03 脚本对应段落原样提取执行补齐（补齐后 sys_menu=136 行、sys_role_menu=88 行）；
   - `sys_user` 缺 admin（id=1）、`sys_role` 缺超级管理员（id=1）、`sys_user_role` 缺 (1,1)、`sys_dept` 缺 id=1——按 01_base 种子行逐字补种（否则 admin/admin123 无法登录，验收 3 无法进行）。
   这些是「环境恢复到 brief 声称的基线」，不是业务改动；涉及行全部来自权威脚本原文。
2. **验收 4 的 prod 启动命令偏离 brief 原文**：a) 补了 prod fail-fast 必需的 `SM4_KEY=otlab_260829____`、`CRYPTO_DATA_KEY=otlab-data260930`、`CRYPTO_INDEX_KEY=otlab-idx-260930`（取 application.yml 的 dev 默认值，非密钥入库）；b) 因打包的 application-prod.yml 有重复键缺陷（见遗留 1），用 `--spring.config.location` 指向仓库外修正后的同名配置目录。prod 冒烟的核心意图（真实 jar、prod 档、真库真 Redis、无 ERROR 启动、真实登录+鉴权请求 200）已全部达成。
3. **执行了 git commit**（`8d95fb2`，main）：验收 5 的命令比较的是 `35d22f6..HEAD` 两个提交点，不提交则该命令恒为空输出、无法起到「对比改动范围」的作用；提交内容严格限于 9 个范围内文件。
4. **SQL 脚本排版纪律**：为兼容 `backend/scripts/sql-exec.sh`（按「行尾分号」切块、整块跳过以 `--` 开头的块），升级脚本的可执行语句区不得挂 `--` 注释行，说明文字集中在脚本头/尾；脚本头已写明该约定。首次运行曾因语句前挂注释行被吞（仅 3 条语句生效），修正后连跑两次全部语句生效（见验收 2 输出）。
5. 冒烟测试请求体使用了 ASCII 值（如 "Acme Group"）：Git Bash 下 curl 直接内联中文会导致非 UTF-8 字节使后端 JSON 解析失败（第一次尝试即报「请求参数格式错误」），与被测代码无关，改用 ASCII 规避。

## 5. 遗留问题

1. **`backend/ot-admin/src/main/resources/application-prod.yml` 顶层键 `ot:` 重复（94/125 行），prod 档无法启动**——先于本任务存在的存量缺陷（BASE 35d22f6 即如此），本批无权修改。修复方案（一行结构修复：把 125 行的 `ot:` 删除、其 redisson/ip/mail 子块并入 94 行的 `ot:` 块）已在外置配置中验证可行。**建议尽快授权修复并在修复后按 brief 原命令复跑验收 4。**
2. brief 的 prod 启动命令缺 `SM4_KEY` / `CRYPTO_DATA_KEY` / `CRYPTO_INDEX_KEY`（prod fail-fast 必需），照抄执行必然启动失败；建议协调者在后续任务 brief 中补全。
3. 环境初始化脚本在 43 表之外还应产出 sys_menu/菜单/授权/admin 用户等数据，当前库是部分初始化状态（本报告偏离 1 已补齐）。建议核查当时的初始化执行通道为何吞掉部分语句（疑似与 sql-exec.sh 跳过 `--` 开头块的行为同源：01_base.sql 中部分语句紧跟注释行，若用它执行会被整块跳过），必要时为初始化脚本补充同样的「语句前不挂注释行」纪律或在脚本头注明适用执行器。
4. 冒烟产生的业务数据：租户 acme（已逻辑删除、编码不复用占用中）、sys_module id=4、sys_menu 4000-4007、sys_role_menu (1,4000..4007)、默认租户 default——均为预期交付数据；如需干净环境重放验收，直接重复执行升级脚本即可（幂等）。
5. `@SaCheckPermission` 的 standalone 单测只能断言注解取值，运行时鉴权链路（Sa-Token 拦截器）由 dev/prod 冒烟的 401/鉴权行为侧面覆盖；iot 前端页面接入后的权限回显归 TASK-0056。

## 附：根因修复链（契约要求「为什么现在对了」）

| 失败 | 根因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- |
| SysTenant 测试编译失败 | 实现未写（TDD 红灯预期） | 实现主代码 | 红→绿符合 TDD 流程 |
| `找不到符号 DUPLICATE_CODE` | 单模块构建引用 .m2 旧 ot-core | `mvn install -pl ot-core` 后重跑 | 构建顺序问题，非代码缺陷 |
| `testPermissionAnnotations` expected `<iot:tenant:list>` but was `<[iot:tenant:list]>` | 断言按 String 写，而 Sa-Token 1.45 的 `value()` 返回 `String[]` | 断言改 `value()[0]` | 注解数组首元素即权限串，与实现声明的串严格相等，断言语义未放松 |
| 全量 `SaPermissionRegistryConsistencyTest` 红 | `iot:tenant:*` 未在 SQL 登记（升级脚本当时还没写） | 写出升级脚本登记 4002-4007 | 守卫按语句解析 resources/sql 全树（含 upgrade/），sys_menu INSERT 中的六权限码被收集，断言通过；未改任何测试 |
| 升级脚本首跑仅 3 条语句生效、后续 SQLERROR sys_menu 不存在 | sql-exec.sh 按「行尾分号」切块且整块跳过 `--` 开头的块，脚本中语句前挂了 `--` 注释行 | 语句区与注释完全分离（重写脚本），并同步修复了当时缺失 sys_menu 的环境 | 修正后连跑两次输出 5 条语句 + DONE，幂等且数据齐全（见验收 2） |
| prod 启动 `found duplicate key ot` | application-prod.yml 顶层 `ot:` 重复（存量缺陷，越界文件） | 外置修正配置完成冒烟；仓库文件未动 | 合并后 `ot:` 单键、yaml 校验通过、prod 启动无 ERROR 且接口 200（见验收 4） |

---

# 修订轮（验收退回一轮）

- 起因：评审报告 Important 1——`PUT /tenant` 编辑接口可经 mass assignment 静默改写 `status`（绕过 `iot:tenant:status` 权限点，且不受 0/1 校验）。
- 协调者裁决（照做）：编辑接口不承载启停语义——service 层更新时对 status 强制以库内值为准（忽略请求中的 status 字段），不选择报错方案；修复走字段白名单构建更新，并把 id/createBy/createTime/updateBy 等审计字段一并排除在可更新集外（顺带消解评审 Minor 2 的审计字段 mass assignment，属修复副产物）。
- 修订轮提交：`98c155e fix(iot): 租户编辑接口字段白名单构建更新，status 与审计字段不进更新集（TASK-0001 修订轮）`（main 分支）。

## R1. 改动文件清单增补

仅 2 个文件，均在 brief 原 9 文件范围内（`git show --stat 98c155e`：2 files changed, 74 insertions, 1 deletion）：

| 文件 | 改动 |
| --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java` | `updateTenant` 不再 `updateById(请求实体)`，改为字段白名单构建更新实体：status / createBy / createTime / updateBy / updateTime / deleted 一律不进更新集 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImplTest.java` | 新增 2 例（红转绿）：`testUpdateTenant_RequestStatusNeverWritten`（验收要求的红转绿测试，status=0 与非法 5 均不落库）、`testUpdateTenant_AuditFieldsNotUpdatable`（审计字段副产物佐证） |

实现（`updateTenant` 现行为）：校验租户存在 + 编码不可改后，新建 `upd` 实体，仅拷贝 `id`（行键）、`tenantCode`（经不可改校验后与库内同值，回写为 no-op）、`tenantName`、`pointQuota`、`storageQuotaGb`、`expireTime`、`contactName`、`contactPhone`、`remark`；`status` 与全部审计/逻辑删字段不拷贝，`this.updateById(upd)`。

## R2. TDD 红转绿记录

红灯（先写测试，对着修订前实现跑）：`cd backend && mvn test -pl ot-admin -Dtest='SysTenantServiceImplTest'`

```
[ERROR] net.otlab.admin.module.iot.service.impl.SysTenantServiceImplTest.testUpdateTenant_AuditFieldsNotUpdatable -- Time elapsed: 0.007 s <<< FAILURE!
org.opentest4j.AssertionFailedError: createBy 不得经编辑请求改写 ==> expected: <null> but was: <999>
[ERROR] net.otlab.admin.module.iot.service.impl.SysTenantServiceImplTest.testUpdateTenant_RequestStatusNeverWritten -- Time elapsed: 0.007 s <<< FAILURE!
org.opentest4j.AssertionFailedError: 请求携带的 status（0 或非法 5）不得进入更新实体，库内状态以库内值为准 ==> expected: <null> but was: <0>
[INFO] Results:
[ERROR] Failures:
[ERROR]   SysTenantServiceImplTest.testUpdateTenant_AuditFieldsNotUpdatable:221 createBy 不得经编辑请求改写 ==> expected: <null> but was: <999>
[ERROR]   SysTenantServiceImplTest.testUpdateTenant_RequestStatusNeverWritten:195 请求携带的 status（0 或非法 5）不得进入更新实体，库内状态以库内值为准 ==> expected: <null> but was: <0>
[INFO]
[ERROR] Tests run: 13, Failures: 2, Errors: 0, Skipped: 0
[INFO] BUILD FAILURE
[INFO] Total time:  20.099 s
```

红灯证实评审发现：请求携带的 status=0 与客户端 createBy=999 原样进入更新实体（旧实现 `updateById(tenant)` 直传请求实体）。未改任何既有测试与期望即出现红灯，证明测的是缺陷本身。

绿灯（实现白名单后，仅跑两个租户测试类）：`mvn test -pl ot-admin -Dtest='SysTenantServiceImplTest,SysTenantControllerTest'`

```
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.322 s -- in net.otlab.admin.module.iot.controller.SysTenantControllerTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.389 s -- in net.otlab.admin.module.iot.service.impl.SysTenantServiceImplTest
[INFO] Tests run: 25, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

（输出中的 3 条 GlobalExceptionHandler ERROR 日志是负向用例的预期业务异常日志。既有 11 例 service 测试与 12 例 controller 测试全部原样通过——未改任何存量断言。）

## R3. 补充验收 1：全量复跑

命令：`cd backend && mvn test -pl ot-admin`

完整 Results 段原始输出：

```
[INFO] Results:
[INFO]
[INFO] Tests run: 1033, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  45.597 s
[INFO] Finished at: 2026-11-21T12:32:23+08:00
```

新增两类（本轮 13 例 = 原 11 例 + 修订轮 2 例）：

```
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.259 s -- in net.otlab.admin.module.iot.controller.SysTenantControllerTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.353 s -- in net.otlab.admin.module.iot.service.impl.SysTenantServiceImplTest
```

判定：**通过**。1033 例（上轮 1031 + 本轮 2）0 失败 0 错误，存量无回归。

## R4. 补充验收 2：dev 档冒烟复验（服务重启后）

端口 8098 起始无监听（`netstat` 查证 LISTEN 为空），前轮 dev 实例早已停止；用修订后代码重新执行 `cd backend && mvn spring-boot:run -pl ot-admin`（后台）——就绪判据 `GET /auth/captchaImage` HTTP 200：

```
READY after ~12s (captchaImage HTTP 200)
final probe: 200
```

登录（dev 档测试登录）：

```
=== POST /auth/login/test {"username":"admin","password":"admin123"}
{"code":200,"data":{"nickname":"管理员","token":"b6bc98bc-a661-43c2-aa16-2022fef79cf1","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795235597315}
```

（原 id=2 的 acme 已逻辑删除且编码不复用，按裁决改为自行设计序列：新建临时租户 revtmp。其 id=4 而非 3，是因为上轮冒烟步骤 9 的同码重建 INSERT 撞 uk_tenant_code 后事务回滚，InnoDB 自增号不随回滚退回、id=3 已被消耗——属预期现象，不影响断言。）

```
=== 1) POST /tenant 新建临时租户 revtmp (pointQuota=100, status=1)
{"code":200,"data":null,"message":"操作成功","timestamp":1795235602345}
HTTP_STATUS:200
=== 2) GET /tenant/list?tenantCode=revtmp 拿 id 并确认初始 status=1
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":4,"tenantCode":"revtmp","tenantName":"Rev Round Temp","status":1,"pointQuota":100,"storageQuotaGb":20,"expireTime":null,"contactName":"Rev Tester","contactPhone":"13800000001","remark":"TASK-0001 revision round smoke","createBy":1,"createTime":"2026-11-21 12:33:22","deleted":0,"updateBy":1,"updateTime":"2026-11-21 12:33:22"}],"size":10,"total":1},"message":"操作成功","timestamp":1795235602484}
HTTP_STATUS:200
```

核心复验 1——编辑携带 status=0 改不动（同时证明编辑路径本身正常：tenantName/pointQuota 正常更新、updateTime 变为 12:33:31，唯独 status 未动）：

```
=== 3) PUT /tenant 携带 {"id":4, 正常字段..., "status":0} —— 编辑不得改写启停
{"code":200,"data":null,"message":"操作成功","timestamp":1795235611215}
HTTP_STATUS:200
=== 3b) GET /tenant/4 回读：status 应仍为 1（改不动），tenantName/pointQuota 应已更新
{"code":200,"data":{"id":4,"tenantCode":"revtmp","tenantName":"Rev Round Temp Edited","status":1,"pointQuota":200,"storageQuotaGb":20,"expireTime":null,"contactName":"Rev Tester","contactPhone":"13800000001","remark":"edit with status=0","createBy":1,"createTime":"2026-11-21 12:33:22","deleted":0,"updateBy":1,"updateTime":"2026-11-21 12:33:31"},"message":"操作成功","timestamp":1795235611280}
HTTP_STATUS:200
```

核心复验 2——编辑携带非法 status=5 同样忽略不落库（与单测镜像）：

```
=== 4) PUT /tenant 携带非法 "status":5 —— 同样忽略不落库
{"code":200,"data":null,"message":"操作成功","timestamp":1795235619210}
HTTP_STATUS:200
=== 4b) GET /tenant/4 回读 status 仍为 1
{"code":200,"data":{"id":4,"tenantCode":"revtmp","tenantName":"Rev Round Temp Edited","status":1,"pointQuota":200,...,"remark":"edit with invalid status=5",...,"updateTime":"2026-11-21 12:33:39"},"message":"操作成功","timestamp":1795235619272}
```

核心复验 3——专用启停接口仍有效：

```
=== 5) PUT /tenant/4/status {"status":0} —— 专用启停接口仍有效
{"code":200,"data":null,"message":"操作成功","timestamp":1795235619348}
HTTP_STATUS:200
=== 5b) GET /tenant/4 回读 status=0
{"code":200,"data":{"id":4,"tenantCode":"revtmp","tenantName":"Rev Round Temp Edited","status":0,"pointQuota":200,...},"message":"操作成功","timestamp":1795235619422}
```

清理（临时租户删除，逻辑删）：

```
=== 6) DELETE /tenant/4 清理临时租户
{"code":200,"data":null,"message":"操作成功","timestamp":1795235623705}
HTTP_STATUS:200
=== 6b) GET /tenant/list?tenantCode=revtmp 确认已删（total=0）
{"code":200,"data":{"current":1,"pages":0,"records":[],"size":10,"total":0},"message":"操作成功","timestamp":1795235623789}
```

判定：**通过**。冒烟后停止 dev 实例；`revtmp` 编码按「删除后编码不复用」纪律占用中（与上轮 acme 同口径，属预期业务数据）。

## R5. 偏离与根因说明（为什么现在对了）

**缺陷根因**：旧 `updateTenant` 在校验编码后直接 `this.updateById(tenant)` 把整个请求实体交给 MyBatis-Plus——MP 默认按「非空字段进 SET」生成 UPDATE，`SysTenant.status` 是实体上的普通字段，无人拦截；MetaObjectHandler 只负责审计四字段的 strict fill（仅补 null，不覆盖已有值）。于是：持有 `iot:tenant:edit` 而无 `iot:tenant:status` 的调用方在编辑请求里携带任意非空 status 即可直写 DB（绕过权限点，且不受专用接口的 0/1 校验）；同理客户端自带的 createBy/createTime/updateBy/updateTime 也原样进 UPDATE SET（strict fill 见非空不覆盖）。

**为什么现在对了**：更新不再使用请求实体，而是按白名单新建 `upd` 实体后 `updateById(upd)`——SET 列完全由白名单决定：

1. `status` 不在白名单 → UPDATE SET 根本不含 status 列 → 库内值天然保持（即裁决的「强制以库内值为准、忽略请求 status、不报错」的最彻底实现：编辑接口对 status 完全无语义，连「写回同值」都不发生；副产物是并发启停不会被编辑请求覆盖）。专用启停语义唯一入口仍是 `PUT /tenant/{id}/status`（`iot:tenant:status` + 0/1 校验）。
2. `createBy/createTime/updateBy/updateTime/deleted` 不在白名单 → 客户端审计字段 mass assignment 消解；`updateBy/updateTime` 由 MetaObjectHandler strictUpdateFill 按当前登录者回填，审计语义反而更正确。
3. `id` 仅作 updateById 的行键（@TableId 不进 SET），客户端无法借编辑改写主键位次。
4. `tenantCode` 经不可改校验后与库内同值，回写为 no-op——保留它使既有单测断言（`saved.getTenantCode()=="acme"`）无需改动且语义不变；DB 效果与「不回写编码列」完全等价。

**偏离说明（仅两条，均非行为偏离）**：

1. 裁决原文「service 层更新时对 status 强制以库内值为准」存在两种实现读法（回写库内同值 / 不触 status 列）。取「不触 status 列」：满足同一验收口径（携带 0/5 均改不动）且严格更优（不发生任何 status 写入、无并发覆盖窗口）。单测断言按此实现写为「更新实体 status 为 null」。
2. 评审 Minor 2 的 POST（新增）路径 mass assignment（客户端可带 id 指定位次、审计四字段）本轮**未**修：裁决明确只覆盖「可更新集」（UPDATE 路径），维持评审 deferred 口径不变；若后续要收口，属独立小改。

**契约自查**：未派 subagent；未改 TASK-0001 状态与 workitems 库其他文件；改动仅原 9 文件中的 2 个 + 无新增文件，application*.yml 未碰；追加 1 个 commit（`98c155e`，main，注明 TASK-0001 修订轮）；红灯先于实现出现、测试与期望未被反向修改。
