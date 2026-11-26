# TASK-0010 diff 评审报告（fresh reviewer）

- 评审对象：单提交 `9a4c6ef`（4 文件，+437 −0），BASE `9045f38`
- 评审依据：review brief 两口径（①与 brief 契约一致性；②明显质量缺陷——边界 / SQL 注入面 / 接口 javadoc 契约完整性（0011 接线依据）/ 测试断言真实性）+ `L1-execute.md` §13
- 评审方法：完整读 diff 包（4 文件逐行）、实施 brief、TASK 详情、实施报告、既有 `IotDataDictServiceImpl` 口径基准；并独立核实——diff 包与 `git show 9a4c6ef` 逐字节一致、提交父哈希=BASE、`git diff --name-only` 4 文件全落声明范围、`BaseEntity` 确有 `@TableLogic`（deleted=未登记口径成立）、新测试类 17 例 `@Test` 与报告「17 例」一致、1260=1243+17 闭环、测试口径与既有 `IotDataDictServiceImplTest` 同款（TableInfoHelper/openMocks/ReflectionTestUtils 逐行同构）。

## 一、与 brief 契约一致性（逐条核对，全部一致）

| brief 裁决 | diff 落地 | 结论 |
| --- | --- | --- |
| 1 接口形态 | `Map<String, DictCodeStatus> validateForTemplateRef(Long tenantId, Collection<String> dataCodes)` 逐字采纳；三态 OK/NOT_REGISTERED/DISABLED；deleted 行经 `@TableLogic`（`BaseEntity` 已证实）按未登记口径；一次 IN 预检去重回填、不逐个 count | 一致 |
| 2 文案区分 + 纯查询 | `MSG_NOT_REGISTERED="数据编码未在字典登记: "`、`MSG_DISABLED="数据编码已停用: "` 与 brief 逐字一致 + `messageFor` 拼接；实现无 throw、无 `@Transactional`、无写路径（测试 `never insert/update/delete` 锁定） | 一致 |
| 3 租户显式双保险 | 显式 `tenant_id` eq 进查询条件 + 插件同值注入；「插件注入致条件重复」的运行时实测不可达（本任务无端点无会话链路），报告未做项 4 / 遗留 1 已作语义分析并说明，留 0011 联测实测——brief 明文「以实测为准，报告里说明」已满足 | 一致（说明到位） |
| 4 测试口径 | 空集合空 Map 不查库、三态各断言、批内混合（3 态齐 + times(1) selectList）、deleted=NOT_REGISTERED、跨租户不可见以实现为准（wrapper 断言例：sqlSegment 含 tenant_id/data_code/IN 且参数值=入参 2L）、文案常量逐字 + 枚举存在性 | 一致 |
| 5 不接线 | 无端点/无菜单/无冒烟/未碰模板域；TASK 详情状态仍「进行中」未被实施者回写 | 一致 |

## 二、发现清单（Critical 0 / Important 0 / Minor 3）

1. **Minor** / 出处：`IotDataDictService.java` 新增 javadoc（对照 `IotDataDictServiceImpl.java` 校验器实现注释）/ 接口 javadoc 未载明返回 Map 的保序保证——实现用 `LinkedHashMap` 且实现注释明确预期「调用方可按模板数据项顺序取首个失败项报错」（0011 将依赖的用法），但该契约既不在 0011 接线的唯一契约面（接口 javadoc）也无测试锁定，换 HashMap 实现将静默失去顺序而不构成违约；建议接线前补一行 javadoc 或由 0011 自行按入参序遍历规避。
2. **Minor** / 出处：`IotDataDictServiceImpl.java#validateForTemplateRef`（IN 条件构建）/ 入参集合无单批上限（对照同文件 `importDicts` 的 5000 行上限），超大集合将生成巨型 IN 子句，接口 javadoc 亦未声明上限口径；当前唯一调用方 0011（单模板数据项数）天然受限且结果 fail-closed，风险低，建议 0011 接线时补调用侧约束。
3. **Minor** / 出处：`IotDataDictServiceImplValidateForTemplateRefTest.java` `testValidateForTemplateRef_DeletedRowIsNotRegistered` / `testValidateForTemplateRef_CrossTenantCodeNotVisible` / 两例均以「mock 查不到该行」模拟 DB 侧行为，与「未登记」用例断言同构、无独立回归力：`@TableLogic` 真实过滤未经端到端验证（跨租户的显式条件已由 wrapper 断言例间接覆盖）；测试注释已如实披露模拟口径，且 brief 裁决 5 本就不含集成/冒烟，口径所限——建议随 0011 / TC-0003 联测闭环（与实施报告遗留 1/2 汇合）。

## 三、质量缺陷面核查（无 Critical / Important 级发现）

- **边界**：null/空集合→空 Map 不查库；tenantId null→全判 NOT_REGISTERED 不查库；批内 null/空串→NOT_REGISTERED 且不进 IN；库行 status null→DISABLED（fail-closed）；status 非 1 值→DISABLED；重复编码去重回填单条结果；返回 Map 键与入参一致（含 null 键映射，javadoc 有载）。空集合/超大集合外无漏防边界。
- **SQL 注入面**：`LambdaQueryWrapper` 全参数化，无字符串拼接进 SQL；测试断言参数值绑定（tenant=2L）。
- **javadoc 契约完整性（0011 接线依据）**：签名语义、三态含义、纯查询 + 4002 映射归调用方、deleted/已停用口径、一次 IN + 去重、显式租户双保险、fail-closed 边界、null/空集合返回空 Map、文案常量与 `messageFor` 用法均已载明；仅缺发现 1 的保序声明。
- **测试断言真实性**：17 例中三态映射、去重、fail-closed 早退、wrapper 条件、写路径为零均为真实断言（非恒真）；TDD 红灯为编译失败（无断言放松）；deleted/跨租户两例的模拟局限如实披露（见发现 3）。

## 四、结论

- 与 brief 契约逐条一致，越界反查独立复核通过（4 文件全落声明范围，diff 包与提交逐字节一致），验收数字闭环（1260=1243+17，报告原始输出在案）。
- **Critical 0 / Important 0 / Minor 3**：按 `L1-execute.md` §13，Minor 不进修订轮，记 TASK `任务结果` deferred（收尾进 LOG）；3 条 Minor 均建议在 TASK-0011 接线时一并消化（发现 1/2 为接线契约与调用侧约束，发现 3 与实施报告遗留 1/2 同源）。
