# diff 评审报告：TASK-0048（iot_status_rule 配置接口与校验 + 状态编码稳定字典）

- 日期：2026-11-22
- 评审者：fresh reviewer（不共享协调者上下文；no-subagents；只评审不修改）
- 评审对象：单提交 `d74d56f`（BASE `fb574cf`），22 文件约 2980 行（diff 包 TASK-0048-diff.txt 完整读入）
- 依据：L1-execute §13；实施 brief（DDL 逐字/TD-0013 落表/优先级物理拒绝/上限 10/Expander 同口径/直绑映射/只增不删纪律）；ADR-0009 TD-0013、TD-0014
- 结论：**Critical 0 / Important 0 / Minor 4**——通过，无退回项；Minor 按规程记 TASK `任务结果` 的 deferred。

## 一、契约一致性核对（重点项逐条）

| 重点项 | 核对结果 |
| --- | --- |
| 规则表 p35 逐字 | ✅ 以 `docs/factory-data-platform-detail-design-v1.1.pdf` 附录 A p35 实际提取复核：14 列名称/类型/顺序、`uk_template_priority (template_id,priority)`、`KEY idx_tenant_template`、ENGINE/CHARSET 全部一致；上游无 `config_version` 列（报告口径属实）；`binding_map json` 为 brief 裁决补列且脚本头注登记「与上游差异」 |
| 状态字典表与 TD-0013 语义对齐 | ✅ 与 brief 逐字 DDL 一致（report SHOW CREATE 逐字段核对）；seq 服务层 max+1 且 `selectMaxSeq` 显式含禁用行（禁用不复用）；`uk_tenant_status_code` + `uk_tenant_seq` 双物理唯一且均不含 deleted；接口面仅 list/add/disable（守卫测试锁端点数=3，无删除/无名称编辑/无恢复）；disable 只置 status 不触碰 seq |
| priority uk 双道闸 4002 | ✅ 第一道含已删行旁路预检（`@InterceptorIgnore(tenantLine)` + 显式 tenant_id，与 uk 作用域精确一致，0021/0043 范式）+ 第二道 DuplicateKey 兜底；文案区分「已存在」/「已被删除记录占用」 |
| 上限 10 条 | ✅ `selectActiveCountByTemplate`（deleted=0 含停用行）≥10 拒，400 文案「状态规则上限 10 条」；「活跃」口径保守并在头注登记（偏离 4） |
| condition_expr Expander 同口径 + 存在性 4001 | ✅ `ConditionExprValidator` 的 RELATIVE_REF/BRACE_REF 与 `RelativeRefExpander` 逐字节同形（已对照源码核验）；SELF/PARENT 大小写不敏感、编码大小写保留、绝对引用剥壳、残留花括号终检（0013 修订轮 I-1 同口径）；SELF 存在性核对 iot_data_item_template（tenant 插件注入同租户条件，范围正确）；PARENT/绝对引用收集不核存在性=0036 一期口径，已登记 |
| status_code 未登记拒 | ✅ 4001「请先在状态字典中创建」；已禁用 4001 拒（禁用一次性）；新增与修改携带时同口径 |
| 直绑映射 binding_map json 列与校验 | ✅ ruleId（偏离 2，必要补全已登记）+ pointId 校验存在 4001 + 批内枚举值互撞拒 + statusCode 逐项校验存在且启用（**禁用编码不可入映射**，与 brief「存在启用」一致）；整份 JSON 幂等覆盖；白名单不夹带其他字段；Jackson 序列化失败显式 400 |
| 菜单 4063-4069 七行一一对应 | ✅ 脚本七行恰=规则四码+字典三码，两个控制器权限码并集⊆七值、各自精确相等（守卫测试六用例锁定）；种子构成偏离（弃 query/增 statusdict:list）系 brief 种子段与字典段并读的唯一自洽解，已在脚本头注+守卫 javadoc+报告偏离 1 登记 |
| 写路径模板 resolve 防线 | ✅ create/list/update/binding-map/remove 五路径全部先 resolve 模板存在同租户；deleteRules fail-closed 整批拒绝不半删 |
| SQL 参数化 / mass assignment | ✅ 全部 `#{}` 参数化、like 经 SqlLikeUtils.escape；实体白名单构建，seq/status/tenantId 服务层定、deleted 不可注入 |
| 0056 占位按钮对应 | ✅ frontend `views/iot/template/index.vue` 状态规则入口占位挂模板页（=4009），与七权限行 parent_id=4009 对应；占位文案待本域前端接线（TASK-0048 范围不碰 frontend，无错位） |

## 二、发现清单

| 级别 | 出处 | 说明 |
| --- | --- | --- |
| Minor | `IotStatusRule` 实体 javadoc / 升级脚本头注（业务纪律段） | 判定链语义契约未随配置面声明：enabled=0 的停用规则是否被判定链跳过、存量规则/直绑映射引用已禁用编码时的求值行为，配置侧均未落文字（仅「保存≠生效」登记）——须在 0024 发布链/0049 判定引擎承接时明确，避免下游各自解释 |
| Minor | `IotStatusDictServiceImpl.createDict` DuplicateKey 兜底 catch 块 | 兜底文案混流：uk_tenant_status_code 与 uk_tenant_seq 两类冲突共用同一 4002 文案「状态编码或稳定序号并发冲突，请重试」——并发窗口漏过预检的真重复编码按提示重试会确定性再败，且用户无法区分两种根因（兜底机制本身存在且已测，仅文案/可诊断性问题） |
| Minor | `IotStatusRuleUpdateForm` / `IotStatusRuleServiceImpl.updateRule`、`createRule` | 修改路径字段校验不对称：statusName 无 @NotBlank 且 validateCommonTexts 只查长度——PUT 可把状态名称改成空白串（DDL NOT NULL 拦不住空串）；另 createRule 对 statusCode 无服务层限长（超长未登记码报 4001 而非 400），与字典域 validateStatusCode 的「@Size 之外服务层双保险」口径不对称 |
| Minor | `IotStatusDictServiceImplTest.testList_Filters` | 测试断言过弱：`sql.toUpperCase().contains("STATUS")` 因 status_code 列名恒含 "STATUS" 而永真——status eq 过滤条件若被误删该断言仍绿（首个 `sql.contains("status =")` 分支有效，宜收敛为唯一断言） |

## 三、复核确认（非缺陷，留痕）

1. **seq max+1 并发竞态兜底**（brief 重点）：两用户同租户并发新增撞 uk_tenant_seq → DuplicateKey → 4002「请重试」，物理唯一保序号不漂移——兜底在位且有专项测试（testCreate_DuplicateKeyFallback），与 0011/0021/0043 双道闸范式同款。
2. **禁用一次性幂等重放**：重复 disable 再执行一次 UPDATE（触碰 update_time）——语义 no-op、无状态翻转风险，可接受。
3. **菜单 sort 7-13**：4009 之下此前无按钮子行（4010-4016 为 4000 下页级行、4058-4062 挂 4014），无排序冲突；号段 4063-4069 起点自洽（4062 已占属实）。
4. **dev 库冒烟残留**（字典 4 行 + 规则 10 逻辑删行）：只增不删纪律的正常业务数据，报告遗留 1 已登记，不影响仓库。
5. **五项验收输出**与 diff 相互印证：2036=1979+57 用例数与四个测试文件规模吻合；两表 SHOW CREATE 与脚本 DDL 一致。
