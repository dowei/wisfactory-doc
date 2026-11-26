# 评审报告：TASK-0013（实例创建批量生成数据项与相对引用展开）

- 评审者：fresh reviewer（brief：reports/TASK-0013-review-brief.md）
- 对象：单提交 `fb13384`（BASE `46e923d`），diff 包 17 文件 2479 行，已完整读入
- 依据：L1-execute.md §13 + 评审 brief 两样（brief 契约一致性 / 明显质量缺陷），重点为相对引用展开与依赖图的计算正确性地基，从严
- 交叉印证（自己的 diff 判断为准，仓库实读核实）：`createInstance` 确为 `@Transactional(rollbackFor=Exception.class)`（IotModelInstanceServiceImpl.java L134-135，生成事务内一致成立）；`instance_code varchar(96)`（20261122_iot_template_domain.sql L100）→ full_key 上限 96+1+64=161 ≤ `full_key varchar(180)`，列宽自洽；0011 公式校验为「限长 + 括号/花括号配平」降级口径、真实 Aviator 校验登记归 FEAT-0012；字典 data_code 仅限长 64、无字符集约束（IotDataDictServiceImpl L303）；路由无冲突（/iot/instance/{id}/items/regenerate 与 0012 既有映射不撞）；ref 图写入 / 幂等三态 / 双 uk / 事件四元 payload / config_version=NULL / 删除保护活跃行计数 / 菜单 4036-4040 一一对应，均与 brief 契约相符。

## 结论

总体实现质量高：生成事务内一致（触发一同事务含实例行回滚）、重生成三态幂等（跳过/补生成/历史保留，含已删行旁路预检）、PARENT 顶级失败整批、ref 逐引用落库（批内去重 + ref_item_id 三级回填）、新增项不新增上级引用（一档口径，冒烟 S4 实测父汇总项出边 0）、双道闸 4002、事件只发布不建监听、删除保护接线一处、SQL 全参数化——均落实且有测试固化。发现 0 Critical / 2 Important / 5 Minor，无退回级（Critical）缺陷；Important 两条不阻塞收尾判定，建议随 FEAT-0012 公式校验器接线或独立修复轮处理。

## 发现清单

### Important

1. **Important** / 出处：`backend/.../dataitem/service/support/RelativeRefExpander.java`（expand 的 RELATIVE_REF/BRACE_REF 两个正则）+ `IotDataItemServiceImpl.java`（生成循环，无展开终检）
   畸形/超纲花括号 token 静默 fail-open：RELATIVE_REF 代码段限 `[A-Za-z0-9_]`，而上游字典 data_code 无字符集约束（仅限长 64）、0011 公式校验只做括号配平——`{ self.X }`（含空格）、`{SELF.ELEC-METER}`（连字符/非 ASCII 编码）这类 token 既不展开、不核存在性、也不被 BRACE_REF 剥壳，**原样带花括号落进 instance.formula 且依赖图缺边**，无「展开后公式不得残留 `{`/`}`」终检兜底；坏公式随模板批量传染到该模板所有实例，仅在未来计算期才报错。评审 brief 点名的「畸形表达式」边界未闭合；建议展开终检残留花括号即整批拒（一行修复），并评估把编码字符集约束上推到字典域。

2. **Important** / 出处：`IotDataItemServiceImpl.java` generateForInstance 的 selfScope 构建（活跃占用 ∪ **全部** templateItems 编码）
   SELF 存在性集合把「已删行占用但模板仍在册」的编码计入存在：该编码因 occupancy skip 不会在本批生成，但引用它的公式通过存在性检查照常落库——公式指向已删项 full_key、ref 边 ref_item_id=NULL 指向死键，依赖图静默悬空（与「已删键永久占用、无 API 复活」的已登记语义组合出的死角）。本期无删除 API、可达性低（仅 DB 直删），但 0023 清项接口交付后即成真实路径；修复为 selfScope 只并入「本批将插入项」（newItems）编码而非全部模板编码。

### Minor

3. **Minor** / 出处：`IotDataItemServiceImpl.java` generateForInstance 末段 eventPublisher.publishEvent
   领域事件在事务内发布（提交前）：外层事务（触发一为 createInstance 事务）若提交失败则产生幻影事件（generatedItemIds 已回滚）；当前无监听器零影响，事件 javadoc 与报告遗留问题 4 已登记 @TransactionalEventListener 提示——按评审口径「事件发布时机与事务回滚交互」计 Minor 登记，FEAT-0012 接线时必须裁决。

4. **Minor** / 出处：`IotDataItemServiceImpl.java` selectItemsByInstanceId（先验实例存在性、后裁数据权限）
   受限用户探测同租户 id 可区分「存在但不可见（返回空清单）」与「不存在（400 实例不存在）」，与该方法 javadoc「不泄露不可见实例的存在」的宣称不完全一致；照 regenerateForInstance 的 403 范式（先权限后存在性）统一即可消除。

5. **Minor** / 出处：`RelativeRefExpander.java`（绝对引用剥壳无长度校验）+ ref 落库路径
   绝对引用 `{...}` 剥壳后的键超过 `ref_full_key varchar(180)` 时触发 SQL Data-too-long → 500（非优雅 400）；整批回滚虽 fail-closed，但错误形态粗糙，建议对剥壳键补长度校验并入 paramError 口径。

6. **Minor** / 出处：`IotDataItemTemplate.java`（defaultFormula 无 @NotNull、0011 无耦合校验）+ `IotDataItemServiceImpl.buildItem`（照单全收）
   source_type=2（公式）与 default_formula 非空的耦合校验模板域与生成链两侧都缺：可产出 source_type=2、formula=NULL 的实例项（计算引擎侧必然失败）。跨任务缺口（0011 已登记 Aviator 校验归 FEAT-0012），建议生成链对「source_type=2 且公式空白」整批拒绝（fail-closed 与本任务其他校验同纪律）。

7. **Minor** / 出处：`backend/.../instance/service/IotModelInstanceService.java`（deleteInstances javadoc 2 行）
   brief 改动范围声明未列该接口文件（声明 impl「仅一处」），字面越界 2 行注释同步；零行为改动、报告 §4.1 主动登记并提供回退选项、协调者越界反查已确认——按 L1 §13「越界通常 Important 起」衡量，因纯文档同步 + 显式登记降为 Minor 留档。

## 契约一致性核对（①的逐项结论，均通过）

- 生成事务内一致：触发一在 createInstance（@Transactional 实核 L134）内经代理调 generateForInstance，同事务，失败含实例行回滚（测试 testCreate_GenerationFailurePropagates 固化传播语义）。
- 重生成幂等三态：已存在（含已删行占用，selectOccupancyByInstance 旁路预检）跳过 / 模板新增补生成 / 模板已删项保留（无任何 delete 调用，测试三态各一）。
- 字段映射：data_code/data_name/category/unit/default_source_type→source_type（空兜底 1，登记）/default_formula→formula（展开后）/agg_method→rollup_method/四阈值/default_calc_period→calc_period（码→秒换算，未知码整批 fail-closed，报告 §0.2 说明口径）；value_kind 留模板（实例无列）。
- 相对引用展开：语法按 PDF 实际提取（花括号包裹，报告 §0.1 交叉印证产品设计 V2.2 §5.2），SELF./PARENT. 大小写不敏感、展开后裸键形态；顶级 PARENT 抛明确文案整批失败（expander 内，登记 §4.3）；SELF/PARENT 存在性整批失败（登记 §4.4 扩展）；绝对引用剥壳不核存在性（登记一期口径）。
- ref 依赖图：逐引用落库、批内去重（uk_item_ref 口径前置）、ref_type=1、ref_item_id 三级回填（本批/父实例/按 full_key 兜底查询）。
- full_key 冗余与双 uk：= instance_code.data_code，uk_tenant_instance_data + uk_tenant_full_key 双物理唯一在位（SHOW INDEX 实测），双道闸 4002 兜底并发（同实例重生成并发窗口由预检 + DuplicateKeyException→4002 整批重试覆盖）。
- 新增项不新增上级引用（ADR-0008 一档）：只写出边不改上级引用集，冒烟 S4 实测父汇总项出边 0。
- 领域事件：四元 payload 与 brief 一致，只发布不建监听（见 Minor 3 时机登记）。
- config_version 置 NULL：buildItem 显式置 null。
- 删除保护接线：deleteInstances 一处守卫 + 注入，活跃行计数（@TableLogic），提示项数，批删逐实例列明（fail-closed）。
- 菜单 4036-4040：SQL 五行 parent_id=4010、perms 一一对应，超管授权五行，守卫测试双向锁定（SQL 恰好 + 注解 ⊆）。
- SQL 参数化：唯一手写 @Select 用 #{} 显式 tenant_id+instance_id 且 tenant 取自 TenantContext 禁请求携带；其余全走 MyBatis-Plus wrapper。
