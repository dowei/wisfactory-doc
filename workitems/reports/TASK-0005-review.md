# 评审报告：TASK-0005 diff 评审（fresh reviewer）

- 评审对象：提交 `384d58f`（TASK-0005 单提交，24 文件约 2387 行，已隔离并行提交 cff0642；越界反查与报告声明一致——全部落在 brief 声明范围 `dataperm/**` + application.yml 单键内）
- 评审角色：代码评审（跨系统共用权限组件，从严）
- 依据：ADR-0016 七项裁决（逐条对照）、`L1-execute.md` §13 diff 评审口径、TASK-0005-brief / TASK-0005 详情 / TASK-0005-inputs（§4 全文 + 五种 scope 语义）/ TASK-0005-report（交叉印证）
- 方法：完整读入 diff 包（2387 行逐行）+ 全部必读文件；并到代码仓交叉验证了 `OtTenantLineHandler`（租户白名单）、`MybatisPlusConfig`（@MapperScan 模式与插件注册顺序）、`DataScopeEnum`（1-5 值）、框架 `DataScopeServiceImpl`（语义对齐）、`TenantContext`（无上下文行为）
- 结论汇总：**Critical 0 / Important 2 / Minor 6**。ADR-0016 七项裁决全部按口径落实（见下表）；Important 2 条属权限放大与纵深防御缺口，按 §13 进退回路径；Minor 6 条记 deferred。

## 一、ADR-0016 七项裁决一致性核对（逐条）

| 裁决 | 结论 | 证据 |
| --- | --- | --- |
| 1 子树上限截断 + WARN | ✅ 一致 | `DataPermProperties.subtreeMax=5000`、application.yml 单键 `ot.dataperm.subtree-max`（`${OT_DATAPERM_SUBTREE_MAX:5000}`）；`expandSubtree` 按 ancestors 路径序（深度升序，同深按 id）截断保近端——祖先深度严格小于后代，截断前缀对祖先闭包，正确；WARN 逐字符 `[DATAPERM] subtree truncated user=<id> kept=<n> cap=<n>` 有测试锁定（`DataPermResolveSubtreeTest`） |
| 2 SCOPE_EMPTY / SCOPE_FORBIDDEN + isDetailAccess | ✅ 一致 | 常量以 `DenyStrategy` 枚举交付；`isDetailAccess` 尾段词表判定、未知形态 fail-closed 判 403；本任务未接线（符合 brief）；启发式局限见 Minor-2 |
| 3 仅本人适用集 | ✅ 一致 | 配置类 `configCondition` create_by 分支（含无锚点表只承载 create_by 语义，brief 有言在先）；时序 `timeseriesCondition` 对 meOnly 返回 `1 = 0`；system 系列表未触碰 |
| 4 多角色并集与 scope=4 合并 | ✅ 一致 | 部门集合仅 scope∈{2,3,5} 产出；scope=4 仅置 `createByOnly`；配置类公式 `(instance_id IN 并集) OR create_by=me` 精确匹配裁决原文；时序无贡献；scope=4 独占时点位列表按 create_by（裁决 3 适用集，报告偏离 7 留痕） |
| 5 中台 scope JSON 三态 | ⚠️ 一致但含 Important-1 | null/空白/畸形/空数组=默认拒绝（含畸形 JSON WARN + fail-closed 测试）；subtree=false 精确；subtree=true 复用同一 expandSubtree（含截断）；allTenant/createByOnly 恒 false——但 `exactScope` 从实例行推导 deptIds，产生授权放大，见 Important-1 |
| 6 点位双条件 NULL 不特判 | ✅ 一致 | Assembler/Renderer 均只渲染 IN 比较，测试断言输出不含任何 NULL 分支；锚点求解对 dept_id=NULL 不命中 |
| 7 TDengine 标签名 | ✅ 一致 | 常量 `TAG_INSTANCE_KEY`/`TAG_DEPT_ID`；测试禁旧称 instance_code/dept；单引号翻倍转义有测试 |

其他契约核对：VisibleScope 签名与 brief 定版形态一致（record + 防御性拷贝）；VisibleScopeCache 接口 + NoOp + resolve「先缓存后计算」接缝就位；自包含设计无泄漏（import 仅 jackson/lombok/本包 mapper/DataScopeEnum/Spring，不依赖 iot 业务服务、不调框架 DataScopeService）；@MapperScan 模式覆盖 `dataperm.mapper` 子包（运行期可扫描）；SQL 注入面干净（6 个 @Select 全参数化，片段渲染值仅 Long 与转义字符串，列名为调用方常量契约）。

## 二、发现清单

### Important（2 条，按 §13 进退回路径）

1. **Important · 中台 scope 的部门集合从实例行推导，点位类条件下产生超出 scope 声明的授权放大**
   - 出处：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermServiceImpl.java` 的 `exactScope(...)`（deptIds = 可见实例行的 dept_id 并集），并被 `DataPermClientScopeTest` 两条用例（deptIds={10,11} / {10,20,30} 断言）固化为预期行为。
   - 说明：ADR-0016 裁决 5 只授予中台「实例集合」，未定义中台的部门集合；`pointCondition`/`renderPointCondition` 渲染 `dept_id IN (部门集) OR bound_instance_id IN (实例集)` 后，实例分支已覆盖全部被授权点位，而推导出的 dept 分支会额外放行「与被授权实例同部门、但 scope 外的其他实例」的点位行/TDengine 子表（如 scope 只含某车间一台设备时，同部门兄弟设备的原始点位数据可见）。对用户路径无此问题（用户 deptIds 本就是角色授权本体），属中台路径特有放大。建议：中台 deptIds 恒为空集（点位条件仅实例分支，fail-closed 方向），或将该放大语义经 ADR 增补留痕后再保留现行为。

2. **Important · 组件对「无租户上下文」调用无防御性 fail-closed 守卫，最坏失效模式为静默跨租户全量**
   - 出处：`DataPermServiceImpl.resolve / resolveClientScope` 入口（无 TenantContext 检查）；行为链经 `OtTenantLineHandler.ignoreTable`（无上下文时对所有表跳过注入）+ `DataPermInstanceMapper.selectAllForPerm` / `DataPermDeptMapper.selectIdAncestors`（退化为无租户谓词的全表扫描）证实可达。
   - 说明：无租户上下文时（异步线程 / 接线方漏设上下文），组件会静默产出跨租户的 VisibleScope 且无任何告警，下游业务查询同线程同样无租户谓词——这是本组件设计上要防的最严重失效模式，目前仅靠「调用方必须在有租户上下文的会话内调用」的 javadoc 前提与报告遗留 3 的接线核对点兜底。组件入口加 3 行守卫（`TenantContext.getTenantId()==null → denied()+WARN`）不违背 brief「消费上下文不重复实现」的边界（守卫不是重复实现隔离），且与组件自身 fail-closed 哲学一致；建议加守卫，或以 ADR 留痕说明为何不守。

### Minor（6 条，记 TASK 任务结果 deferred，不进修订轮）

1. **Minor · 中台路径截断 WARN 沿用 `user=<id>` 承载 clientId**——出处：`DataPermServiceImpl.expandSubtree`（subjectId=clientId）与 `DataPermClientScopeTest` 断言 `user=5`。日志主体歧义：grep 无法区分用户 5 与调用方 5；格式为裁决 1 硬约束，改字段需 ADR 增补，至少应在 TASK-0006/0051 接线文档注明。
2. **Minor · isDetailAccess 尾段启发式存在误分类面**——出处：`DataPermAccessStrategy.COLLECTION_TAIL_KEYWORDS` + `tailSegment`。嵌套集合路由（如 `/point/history/{code}`，尾段是编码）与无尾词 REST 集合（`/instance`）都会落入 403 档；两档均为拒绝方向、不破红线，且契约明定「最终选择权在接口实现」，接线时需按形态人工复核，词表扩展已登记报告遗留 4。
3. **Minor · 无实例锚点配置表在 denied()/部门档用户下返回 null（不过滤）= fail-open 子集**——出处：`DataPermConditionAssembler.configCondition`（`failClosedWhenEmpty = instanceIdColumn != null`）。无角色用户查字典类列表将看到全租户数据；行为系 brief 决策 2 明定（无锚点表只承载 create_by 语义），但与 p21 §3.1「列表只返回可见范围」的字面口径存在张力，接线时需确认有意并留痕。
4. **Minor · 触及 application.yml 但验收未含 prod 档真实启动**——出处：diff 中 `application.yml` 新增键 + 报告第二节验收命令（仅单测）。按 L1-execute §6「部署/配置类任务加严」应覆盖目标 profile 真实启动；单键新增、代码默认值 5000 兜底，风险低，建议收尾补一次启动验证。
5. **Minor · 组件 SQL 硬依赖尚不存在的 iot_model_instance（含 deleted 列）**——出处：`DataPermInstanceMapper` 两条 @Select。TASK-0004 建表前组件运行期不可用（单测 mock 不暴露）；报告遗留 2 已登记，需进 TASK-0004 验收对照，本评审确认该依赖真实存在。
6. **Minor · 中台 scopeJson 的 instanceIds 数组长度无上限 + NoOp 期性能遗留**——出处：`resolveClientScope`（未限制 scope 数组元素数，巨型 IN 风险）与 `selectAllForPerm` 全量载入 / `resolveCodesSorted` 每次渲染重查（报告遗留 1 已登记，TASK-0006 Redis 缓存消解）。建议 TASK-0051 开通界面对 scope 数组规模设上限。

## 三、复核过、确认无发现的焦点项（供协调者采信）

- **租户隔离**：`OtTenantLineHandler` 白名单精确覆盖组件 6 张表（sys_user/sys_dept/sys_role/sys_role_dept/sys_user_role + iot_ 前缀），TenantLineInnerInterceptor 对注解 @Select 同样生效，无绕过；组件不重复实现隔离（与 brief 一致）——唯一缺口即 Important-2 的无上下文守卫。
- **ancestors 前缀匹配**：`ancestorsContains`/`ancestorsHitsAny` 均为逗号分隔后整 token 等值比对，`0,11` 不会命中锚点 1（有专门陷阱用例）；实例祖先路径存实例 id、部门展开存部门 id，两套锚点无交叉污染；中段锚点不回看父/兄节点（有用例）。
- **多角色并集空集语义**：无角色/未知 data_scope/自定义空绑定/用户未绑部门均收敛到空集（fail-closed，有用例矩阵）；scope=1 短路跳过全部求解（有 never() 断言）。
- **配置键唯一性**：application.yml 仅新增 `ot.dataperm.subtree-max` 一个键，缩进与注释正确，与代码默认值一致。
- **越界反查**：24 文件全在声明范围；Mapper 补查 sys_user/sys_user_role 属 `dataperm/**` 包内自包含必要扩展，报告偏离 2 已留痕，不构成越界。
- **测试口径**：65 例与报告一致；WARN 逐字符断言、JSON 三态、NULL 归类、默认值、cap 关闭均有锁定；测试全部 mock Mapper 不触库（与 brief 测试口径一致）。

## 四、处置建议

按 L1-execute §13：Important 2 条退回一轮（修 `exactScope` 的 deptIds 语义 + 组件入口租户守卫，二选一可合并为一次小修），复评 scoped 只看修复 diff；Minor 6 条随收尾进 LOG deferred。若协调者认为 Important-1 的放大语义可接受，须先以 ADR 增补留痕再复评通过。
