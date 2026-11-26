# TASK-0011 diff 评审报告

- 评审者：fresh reviewer（仅依据评审 brief「必读」与「依据」清单，未共享协调者上下文）
- 评审对象：提交 `6126aa1`（21 文件，+3062 −1），diff 包 `TASK-0011-diff.txt` 全文 3200 行已完整读入
- 依据：L1-execute.md §13（一致性 + 明显质量缺陷，从严）；评审 brief 两样口径
- 交叉取证的代码库事实（本评审自行核验，非采信报告）：`OtTenantLineHandler`（iot_ 前缀白名单自动纳入，fail-closed）、`BaseEntity`（@TableLogic deleted）、`IotDataDictServiceImpl.validateForTemplateRef`（三态实现）、`DictCodeStatus`（文案常量）、`ResultCode.DUPLICATE_CODE=4002`、详细设计 V1.1 PDF（§1.5 p10 / §1.6 p13 / §3.1 p21 / 附录 A p31-32 原文提取）、既有菜单脚本（4009 在位、列清单一致）、框架批量删除范式（`{ids}` + split + Long 解析为 SysDictData/SysConfig 等既有口径）

## 一、与 brief 契约一致性核对（逐项，均通过）

1. **三表 DDL 逐字性**：附录 A p31/p32 原文与脚本逐列比对——iot_model_template、iot_model_instance 逐字一致（注释文字含全角空格差异属字形规范化，不改语义）；iot_data_item_template 上游列逐字一致 + 两处声明补列在位（default_calc_period=ADR-0011 TD-0011、tenant_id=§0.3 改造二规则且 §1.6 字段表原文确实无此列）；idx_ancestors(768) 前缀索引与头注披露一致（utf8mb4 varchar(1000)=4000B>3072B 上限，物理必要）。uk_tenant_template_code / uk_template_data_code 逐字保持。
2. **九接口 ↔ 权限一一对应**：九个端点与 §3.1 p21 语义一致；六个权限码（4024-4029）与注解并集由 `IotTemplateMenuPermissionGuardTest` 双向锁定（SQL 正则解析 ↔ 注解反射，双向相等断言）；菜单行 parent_id=4009 与 20261121_iot_console_menus.sql 实际在位的 4009 吻合。上游 §3.1 数据项行写 `iot:template:edit` 与本落地 `iot:template:item:edit` 的口径差已按 brief 六行清单执行并在报告遗留 2 登记，非违约。
3. **字典校验接线**：全项编码收集去重后**一次** validateForTemplateRef（测试断言 times(1)）；三态非 OK 按 `DictCodeStatus.messageFor` 0010 原文案拒整批不落库；首失败项按**请求顺序**取（调用方遍历 items 请求序做 get 判定，不依赖 Map 迭代序，HashSet 入参不影响保序语义）；500 上限（ITEM_BATCH_MAX）在 requireTemplateExists 之前、先于任何查库。校验器实现核实：每条入参编码必有返回项（null 租户全判 NOT_REGISTERED、空码判 NOT_REGISTERED、查无判 NOT_REGISTERED）——调用方的 `status == null` 防御分支实际不可达，无 fail-open 通道。
4. **data_code / template_code 不可改**：两编辑接口均「未携带或与库中不一致即拒」，白名单更新集不含不可变字段（连 no-op 都不进 SET）。
5. **删除保护两类引用汇总**：实例引用（提示实例编码）+ 模板数据项（提示模板编码与项数）任一命中整批拒绝、两类来源汇总为一条错误；批内缺失 id fail-closed 整批拒绝；removeByIds 逻辑删。租户过滤已核实：OtTenantLineHandler 对 iot_ 前缀表白名单式纳入，引用计数查询天然只数本租户活跃实例（评审 brief 点名的跨租户计数失真风险**不存在**）；@TableLogic 确保已删行不计数。
6. **平台预置降级**：insertTemplate 强制 tenant_id=会话值、status=1、deleted=0，请求携带不落库（测试断言 999/888/1 均被丢弃）；插件对 INSERT 补列与会话同值双保险。
7. **公式占位校验边界**：非空时长度 ≤2000 + 括号/大括号配平，仅此——Aviator 真语法归 FEAT-0012 的遗留义务已在服务类 javadoc 与报告遗留 1 登记，未越界实现。
8. **0010 Minor①② 消化**：IotDataDictService javadoc 仅 +5 −1 保序契约三句（方法签名/实现零改动，brief 必读节授权例外成立）；调用侧 500 上限常量 + 先于查库。
9. **安全面**：全部查询走 MP Wrapper 参数化（无字符串拼 SQL），LIKE 经 SqlLikeUtils.escape，ids 解析 Long.valueOf 数值化，无文件解析（XXE 不适用）；mass assignment 各写入口均白名单构建且有负路径测试。
10. **事务与并发**：saveItems/insertTemplate @Transactional(rollbackFor=Exception.class)，DuplicateKeyException 捕获后重抛 BusinessException 触发整批回滚，预检+物理唯一双道闸成立。

## 二、发现清单

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| I-1 | Important | `IotDataItemTemplateServiceImpl.saveItems`（库内预检 @TableLogic 只看活跃行 + DuplicateKeyException 兜底文案）× `deleteItems`（逻辑删）× DDL uk_template_data_code 不含 deleted 列 | **逻辑删的模板数据项永久占用 (template_id, data_code) 物理唯一索引，且无任何 API 途径复活/清除（deleted 不进更新集）**：用户经 DELETE /iot/template/item/{ids} 删项后对同模板重存同码，预检放行→落库撞索引→整批 4002「模板数据项编码并发冲突，整批未入库，请重试」——**永久性死路被报成瞬态并发冲突，重试永远无效**。模板域同款占用已在控制器 javadoc 明示为「编码不再语义复用」的有意决策且文案准确（「模板编码已存在」）；数据项域既无同款决策声明也无准确文案。建议二选一：预检含已删行并给出准确文案（如「编码已被删除记录占用，不可复用」），或明示数据项同模板编码逻辑删后不可复用并修文案。 |
| M-1 | Minor | `resources/sql/upgrade/20261122_iot_template_domain.sql` 头注 | 头注写「与上游差异（**两处**…）」但实际枚举三处（default_calc_period / tenant_id / idx_ancestors 前缀），与报告 §0「三处头注」自相矛盾，随上游修订清单回写时一并修正。 |
| M-2 | Minor | `IotDataItemTemplateServiceImpl.saveItems`（paramError 映射）vs 实施 brief「4002 语义映射在调用方」 | 字典三态失败落 400(PARAM_ERROR) 而非 4002——brief/0010 javadoc 措辞（「异常转换与 4002 错误码映射由调用方决定」）存在两种读法，实现者按 TASK-0009 先例取 400 并在报告 §4.6 显著披露（一处常量可改）；**需协调者裁决定版**。 |
| M-3 | Minor | `IotModelTemplateServiceImpl.insertTemplate/updateTemplate`（TenantContext.getTenantId() 直用，无空值守卫） | 无租户上下文时插件 ignoreTable 短路跳过注入：insert 将产出 tenant_id=NULL 的平台预置行（恰为一期降级禁止的 API 入口）、预检失租户条件跨租户计数。HTTP 认证路径实际不可达（会话必带租户），但与数据项侧「校验器 null 租户 fail-closed」的防线深度不对称，建议补 fail-closed 守卫。 |
| M-4 | Minor | `IotModelTemplateServiceImpl.deleteTemplates`（缺失判定 `existingList.size() < ids.size()`） | 批内重复 id（如 `1,1`）触发 size 不等分支但 missing 为空列表，报「模板不存在或已删除: []」——语义误导（fail-closed 本身成立，只是文案与判定方式不精确）。 |
| M-5 | Minor | `IotModelTemplateServiceImpl.updateTemplate` / `IotDataItemTemplateServiceImpl.updateItem` | 编辑路径无服务层长度/值域防线（templateName/remark 超长、valueKind/category 值域仅靠控制器 bean 校验），与新增路径「insertTemplate 服务层防线」的自我声明不对称——standalone 服务调用可绕过。 |
| M-6 | Minor | `IotModelTemplateServiceImpl.updateTemplate`（nodeType 白名单可任意改，无引用守卫） | 有实例引用的模板可自由改 node_type，与已生成实例的层级语义漂移（node_type 约束实例可建层级）；一期实例尚不存在故无现网风险，brief 亦未要求——**建议 TASK-0012 交付实例生成时补「有实例引用禁改 node_type」守卫或明确放行口径**。 |
| M-7 | Minor | `IotModelTemplateServiceImpl.deleteTemplates`（引用来源 selectList 全量取回，仅展示前 5） | 删除保护引用计数把全部引用行拉进内存（实例 selectList 无上限）只为展示 ≤5 条——大模板量级下应改 COUNT + LIMIT 5 取样，防消息与内存双膨胀。 |

## 三、结论

- 契约一致性 10 项全过（DDL 逐字性以 PDF 原文比对核实；租户插件过滤、校验器完备性两处 brief 点名风险实核无虞）。
- **Critical 0 / Important 1 / Minor 7**。I-1 建议走退回修一轮（预检口径或文案二选一，改动面一处）；Minor 全部记 deferred，其中 M-2 需协调者裁决、M-6 转 TASK-0012 注意事项。
