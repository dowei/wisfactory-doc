# TASK-0043 diff 评审报告（fresh reviewer）

- 评审对象：fb574cf（TASK-0043 单提交，17 文件约 3370 行；BASE 9e73d3e）
- 评审依据：TASK-0043-review-brief.md（两样：①brief 契约一致性 ②明显质量缺陷）+ L1-execute.md §13 + 实施 brief + TASK 详情 + 实施报告 + ADR-0009/0011/0014/0016 + REQ-0024
- 必读均完整读入：diff 包 3370 行逐行读完（未抽查）；验证性反查了 Base 体 @TableLogic、DataPermConditionAssembler、VisibleScope、SqlLikeUtils.escape、ResultCode、IotPointCodeMapper / IotPointServiceImpl 既有范式、commit 与 17 文件清单实况
- 结论：**Critical 0 / Important 0 / Minor 10**。无退回项；Minor 记 deferred。

## 一、契约一致性核对（依据①，逐项）

| 契约点 | 判定 |
| --- | --- |
| DDL 与附录 A p33 逐字 | ✅ 脚本 CREATE TABLE 与 brief 逐字 DDL 列序/列型/注释/默认值/三索引逐列一致；`COLLATE=utf8mb4_0900_ai_ci` 为库级继承、脚本未显式写（与点位域同口径，报告已登记）；CREATE IF NOT EXISTS + INSERT IGNORE 显式 id 幂等成立 |
| 菜单 4058-4062 五行 + 超管授权 | ✅ parent 4014、menu_type=3、perms 五码一一对应；sys_role_menu role_id=1 五行；与守卫测试 EXPECTED_MENU_ROWS 双向锁定 |
| 口径强制标注（agg_type/period 合法枚举） | ✅ 表单 @NotNull/@NotBlank/@Pattern + 服务层 Set 值域双保险（resolveAggType/resolvePeriod），创建/批量/编辑生效组合三路径全覆盖；冒烟 S4 三态拒实证 |
| 比值型（agg_type=5）分子分母齐备且同实例存在启用 | ✅ 齐备性（缺一 400）在 validateAndBuild 与 generateMetrics 第一遍双收敛；存在/同实例/启用三查在 validateReferences（4001），单条/批量/编辑生效组合均走到；同租户由多租户插件注入保证 |
| 非比值型 ratio_* 置 NULL | ✅ 创建侧忽略请求值置 NULL（实体 null + DB DEFAULT NULL）；编辑侧切非比值型经 UpdateWrapper 显式清 NULL（规避 MP null 跳过语义），红/单测均有断言 |
| calendar_type 锁 1 | ✅ 未携带落 1、携带 2 拒、值域外拒（resolveCalendarType）；编辑生效组合同口径 |
| metric_code 唯一双道闸 + 字符集 + 不入字典 | ✅ 含已删行旁路预检（IotMetricCodeMapper 显式 (tenant_id, metric_code)、#{} 参数化，照 IotPointCodeMapper 范式）+ DuplicateKey 兜底，双 4002 文案区分；[A-Za-z0-9_] 表单 @Pattern + 服务层双保险；全链路无字典 Mapper 依赖（ADR-0014） |
| 批量生成整批事务逐项校验 | ✅ @Transactional(rollbackFor=Exception.class)；两遍校验（字段级 + 批内一次 IN 库级）；任一失败抛出即整批回滚；DuplicateKey 兜底 4002「整批未入库」；单批 ≤500；instanceId 以 query 入参覆盖（0011 口径） |
| 模板自动生成占位口径 | ✅ 请求清单批量而非模板载体——控制器/服务/DTO/SQL 头注/报告五处登记占位，指向 REQ-0024 后续裁决/0024 发布链 |
| 裁剪 + 写路径防线（0021 修订轮范式） | ✅ list 接 pointCondition（dept_id IN OR instance_id IN，NULL 不特判 fail-closed、空集 1=0、allTenant 直过；create_by 不适用资产类照 0021）；detail 403 单对象形态（裁决 2）；update/changeStatus/remove resolve+可见判定、整批 fail-closed、403 文案不泄露标识（红测 10 用例覆盖，含 NULL 行 fail-closed/1=0/scope=4 空集固化） |
| 启停事件只发布 | ✅ MetricStatusChangedEvent 六字段 payload（含调度所需 period/aggType），只发布不建监听、add/generate 不发（ADR-0011 TD-0028 口径留痕），照 0013/0021 范式；冒烟 S9 两条留痕实证 |
| 上看板占位登记 | ✅ TD-0039/src_type 级联（TD-0027）两处占位在 SQL 头注 + 实体 javadoc + 报告 §4 三层登记，不落列不校验 |
| SQL 参数化 | ✅ 旁路 Mapper 全 #{}（含 foreach）；like 经 SqlLikeUtils.escape；wrapper.apply 条件为服务端 VisibleScope 数字渲染，无用户输入拼接 |

越界反查：17 文件全落 brief 声明范围（metric 域 12 + test 4 = brief 授权的「红线套件增补指标域用例」例外 + 升级脚本 1）；未触 frontend/ingest/dataitem/dataperm 组件本体。

实施报告红转绿 3 处记录齐全（预载空 Map 引用误判 / 夹具事件总线强转 / 防线顺序），修复 rationale 成立；偏离 1-6 与 diff 实况核对一致。

## 二、非编号观察（不属缺陷，请协调者知悉）

1. **创建侧防线超出 brief 字面枚举**（add/generate 目标实例须 ∈ 可见集 403）：brief 写路径防线字面仅 update/remove/status——本批照 0012 修订轮 I-1 同型加创建侧防线并红测覆盖（S8d/S8e 实证不误伤），安全正向加严；实施报告偏离 2 已请求确认，评审无异议，建议收录为范式。
2. **编辑挪挂不校验目标实例可见性**（updateMetric 仅 existence/same-tenant）：与 0021 点位编辑 boundInstanceId 同口径（同租户资产重挂锚点、调用者自身可见面不扩大），范式一致，非本批新引入。

## 三、发现清单

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | `IotMetricServiceImpl.generateMetrics` 第一遍批内判重 | 清单项 metricCode=null 走「批内重复」分支误报 4002「指标编码批内重复: null」（应为 400 编码必填）——产线由表单 @NotBlank 先拦、仅服务层防御路径可达，错误码/文案分类错误 |
| 2 | Minor | `IotMetricServiceImpl.generateMetrics`（seenCodes + DuplicateKey 兜底文案） | 表排序规则 utf8mb4_0900_ai_ci 不区分大小写，批内同码大小写变体（如 A/a）双双通过预检与批内判重后第二笔 insert 撞物理唯一索引，报「并发冲突，请重试」而重试永不成功——结果安全（整批回滚），仅文案口径失真 |
| 3 | Minor | `IotMetricServiceImpl.changeMetricStatus` / `deleteMetrics` 缺失 id 计算 | 入参含重复 id（如 [5,5]）时触发「整批拒绝」分支但 missing 为空，文案「指标不存在或已删除: []」误导（入参未去重）；无安全影响 |
| 4 | Minor | `IotMetricServiceImpl.changeMetricStatus` 更新循环 + `publishStatusChangedEvent` | 逐行 updateById 返回值未校验：listByIds 与更新之间行被并发删除则该行静默跳过而启停事件照发（事件可能引用已删指标）——窗口极窄、注释已声明消费方幂等承接，建议校验返回值或发布前重读 |
| 5 | Minor | `IotMetricServiceImpl.updateMetric` 白名单构建 | 「null 不进更新集」语义下 sourceItemId/deptId 一经设置只能改值不能清空——brief 未要求清空语义，登记功能限制 |
| 6 | Minor | `IotMetricServiceImpl.buildListWrapper`（createByColumn 传 null）+ `MetricRedlineTest.scope4Only_listEmptyEvenForOwnRows` | create_by 分支不适用（资产类照 0021 口径）与 ADR-0016 裁决 3 明文「指标列入 scope=4 按 create_by 过滤的配置类清单」存在成文张力——brief 裁决在先、红测固化，实现契约一致，但 ADR 成文与实现口径的分歧需上游对齐（0021 点位域同款既有张力） |
| 7 | Minor | `IotMetricServiceImpl.publishStatusChangedEvent`（changeMetricStatus 事务内调用） | 事件在主链事务提交前发布——事件 javadoc 已明示并提示未来消费方用 @TransactionalEventListener，本期无监听方、照 0013/0021 范式，前瞻性登记（事件事务边界） |
| 8 | Minor | `IotMetricServiceImpl.buildListWrapper`（query.getDeptId() 分支） | list 过滤在 brief 枚举（metricCode/metricName/instanceId/period/status）外额外支持 deptId eq——与权限条件 AND 叠加不破裁剪，无害枚举外扩展，登记备查 |
| 9 | Minor | TASK-0043-report.md §1.2 标题 | 「测试（5）」实列 4 个测试文件（12 主码 java + 4 测试 + 1 SQL = 17 方与 §2.5 自洽）——标题计数笔误；净增用例 56（43+3+10）与 diff 实况核对无误 |
| 10 | Minor | `IotMetricServiceImplTest.testCreate_CodeTooLong_And_NoDictValidation` | 「编码不入字典」用例为弱断言——以夹具不装配字典 Mapper 的结构性事实承载、无行为级断言，测试名承诺强于实际验证内容；真实保证是服务依赖清单无 SysDictMapper（ADR-0014 形态成立） |

## 四、判定

- Critical 0 / Important 0 / Minor 10——无退回项，不走修订轮；Minor 10 条记 TASK `任务结果` deferred（收尾进 LOG）。
- brief 五项硬验收的 diff 侧证据齐备：逐字 DDL、口径标注三态拒、比值型齐备+同实例启用 4001、非比值置 NULL（含编辑显式清）、calendar 锁 1、双道闸 4002+字符集+不入字典、整批事务批量生成、占位口径登记、裁剪 fail-closed+写路径防线（含红测授权例外）、启停只发布、上看板/级联占位、菜单守卫。
