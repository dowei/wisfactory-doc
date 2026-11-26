# 评审报告：TASK-0012 diff 评审（iot_model_instance 与实例树接口：绑部门、权限裁剪）

- 评审者：fresh reviewer（brief：reports/TASK-0012-review-brief.md）
- 评审对象：diff 包 `TASK-0012-diff.txt`（单提交 c9770ef，BASE c94c9b7，15 文件约 2725 行，已完整读入）
- 依据：L1-execute §13；与实施 brief 契约一致性 + 明显质量缺陷；组件契约按 ADR-0016 裁决 2/3/4/6 与 TASK-0005-inputs §4 步骤 3-5。评审中另实际读入组件源码（VisibleScope / DataPermServiceImpl / DataPermAccessStrategy / OtTenantLineHandler / SqlLikeUtils / BaseEntity / ResultCode / 0011 建库 DDL / updateTemplate 上下文）以核验消费语义，未越出 brief 点到的判断对象。
- 越界反查：15 文件全部落在 brief 声明范围（`instance/**` 8 + 升级脚本 1 + 测试 4 + 实体扩展 1[授权项] + template 服务 M-6 守卫 1[授权例外]，与报告 §2.5 一致）。

## 一、契约核对（brief 逐项，全部实测通过）

- 创建：TD-0008 设备/组件强制模板 + 模板存在/启用/node_type 一致 ✅；工厂/车间 dept 必填（服务层防线）✅；短码 `[A-Za-z0-9_]`（@Pattern + 服务层复核）+ 全码 ≤96 预检 ✅；ancestors 仿 sys_dept（顶级 "0"、父链拼接）✅；applied_template_version=模板当前 version（选模板时）/locked=0/status 强制 1/tenant_id 会话覆盖 ✅。
- 编辑：不可改五字段（携带不一致即拒、null 跳过）✅；锁定仅 status（remark 收窄已登记）✅；白名单构建更新、不可变字段不进 SET、无 mass assignment ✅。
- 删除：子实例保护提示编码 ✅；批内缺失 fail-closed ✅；逻辑删（BaseEntity @TableLogic）✅；数据项引用计数留待 0013（brief 口径）✅。
- 树裁剪：allTenant 全树 / 可见集+根路径骨架（ancestors 整 token 解析且防御已删祖先）/ 无权兄弟不返回 / createByOnly 空树 / 空可见集短路 / 禁用节点照常返回带 status / sort 升序 ✅。
- 列表：id ∈ 可见集 IN 裁剪 + 五项过滤（like 经 SqlLikeUtils.escape）✅；createByOnly/空集短路不查库 ✅。
- 详情：SCOPE_FORBIDDEN 首消费点，403 文案不泄露实例标识 ✅。
- M-6 守卫：updateTemplate 仅一处、node_type 变更才查计数、文案含计数，红转绿记录在报告 ✅。
- 增量脚本：information_schema 判存幂等（PREPARE/EXECUTE）、4030-4035 六行 parent=4010、role 1 授权、守卫测试双向一一对应 ✅。
- 安全面：无注入（全码拼进查询走 #{} 参数绑定 + 短码字符集白名单 + like 转义）；跨租户由插件覆盖（iot_ 前缀 + sys_dept 在 OtTenantLineHandler 白名单内，旁路 Mapper 显式 tenant_id 取会话）；组件无租户上下文守卫使树/列表短路空集 ✅；重复 code 并发兜底（预检 + DuplicateKeyException→4002）✅。

## 二、发现清单

### Critical（0）

无。

### Important（2）

1. **Important / 出处：`IotModelInstanceServiceImpl.updateInstance / deleteInstances / createInstance`（无 resolve 调用）**——编辑、删除、创建三个写路径完全无数据权限校验（仅 iot:instance:edit/remove/add 菜单权限），与同 id 详情 403 形成读写不对称：受限范围用户可对不可见实例直调写接口——最 concrete 的升级链是 `PUT {"id":X,"deptId":<自己部门>}` 把无权工厂/车间的绑部门改挂到自己部门（编辑表单其余字段全可空、validateImmutability 全跳过、dept 仅验存在启用），经组件步骤 3-4 子树展开后该不可见子树整体变为可见（TC-0002 红线「A 厂查 B 厂」被写原语绕过），另可禁用（停算）不可见实例、删除不可见叶子实例；详细设计 p21 虽将组件定位为查询层拦截、写操作记日志，但 REQ-0002「菜单权限与数据权限分离」之下「持编辑菜单 + 窄数据范围」是合法常态配置，建议二选一裁决：写路径补 resolve+成员校验（改动小）或 ADR 显式豁免并登记归 TASK-0007 红线扩展。
2. **Important / 出处：`IotModelInstanceServiceImpl.selectInstanceTree / selectInstanceList` 短路条件 `scope.createByOnly() || scope.instanceIds().isEmpty()`**——把组件 VisibleScope 的 createByOnly（=「存在 scope=4 角色」的标记，DataPermServiceImpl.computeUserScope 对混合角色返回非空 instanceIds + createByOnly=true）误当「仅本人用户」消费：scope∈{2,3,5} 与 scope=4 并存的多角色用户（REQ-0002 明示多角色并集为一等场景）在树与列表均被清空，丢失其部门角色应得的可见集——与 ADR-0016 裁决 4「部门集合仅由 scope∈{1,2,3,5} 并集产出、scope=4 只是不再增补」直接冲突（纯 scope=4 用户 instanceIds 本就为空，删去 `createByOnly()` 一项不改变纯仅本人行为）；brief「createByOnly=true 按空树/空集」的字面口径与裁决 4 存在歧义、实施报告遗留 2 仅登记了列表缺 `OR create_by` 一面而未涵盖此混合角色丢失面，建议改短路条件为仅 `instanceIds().isEmpty()` 并补混合角色用例，随协调者定版。

### Minor（8）

1. **Minor / 出处：`IotModelInstance` 实体 javadoc 与 `IotInstanceUpdateForm` 类级 @Schema**——两处仍写「锁定后仅 status/remark 可变」的收窄前措辞，与已登记的「表无 remark 列、锁定仅 status」口径（及 UpdateForm 自身 javadoc 段）自相矛盾，实施报告 §4.1「DTO javadoc 已登记」与实际不符。
2. **Minor / 出处：`IotModelInstanceServiceImpl.deleteInstances` + 控制器 parseIds**——批内重复 id（如 `/3,3`）使 `existingList.size() < ids.size()` 误判缺失且 missing 过滤后为空列表，报错文案成「实例不存在或已删除: []」；fail-closed 不误删，建议入口先去重。
3. **Minor / 出处：`IotModelInstanceServiceImpl.deleteInstances`**——locked=1 实例可被直接删除（锁定防线只落在编辑）；brief 删除段未提及锁定，但 REQ-0005「锁定后配置变更受限」是否含删除需显式裁决。
4. **Minor / 出处：`createInstance/resolveParent`**——无父子 node_type 层级校验（设备可直挂工厂、组件可直挂车间/工厂），与 §1.5「工厂>车间>产线>设备>组件」层级描述不符；brief 未要求、不影响数据权限，请确认是否有意一期不做。
5. **Minor / 出处：`IotModelInstanceServiceImpl.getInstanceDetail`**——尾段恒为数字 id、isDetailAccess 恒 true，SCOPE_EMPTY 分支与 return null 为不可达死代码；系按裁决 2「按形态选用常量」范式书写，可简化为直接 403 免误导后续维护者。
6. **Minor / 出处：`createInstance` 与 `deleteInstances` 事务边界**——「校验父活跃→插子」与「查子→删父」两事务交错（子插入晚于父删除的子检查）可产生孤儿节点（父逻辑删、子 ancestors 指向已删 id）；buildTree 防御性按根展示、概率低，建议后续硬化（父行锁或删除后复检）。
7. **Minor / 出处：`IotInstanceTreeVO`/树骨架节点**——到根路径骨架节点返回全字段（instanceCode/instanceName/deptId/templateId/locked 等），与 brief「路径节点仅作骨架展示」的表述相比未做字段收窄（详情对同节点 403），泄露面为同厂祖先的身份信息；冒烟 S6b 即如此呈现，是否收窄由协调者定版。
8. **Minor / 出处：`resources/sql/upgrade/20261122_iot_instance_columns.sql` 头注**——「与上游差异」段仅记两列一处，未随 0011 脚本范式补记树接口权限点（上游 §3.1 p22 原文 perms=iot:instance:list → 本脚本单设 iot:instance:tree）的差异；该差异已由 brief 明文授权并登记于实施报告 §4.2，仅脚本头注自身缺记。

### 无法判定项

无——评审 brief 要求核对的各点均有源码/组件/报告证据可判。

## 三、汇总

- Critical 0 / Important 2 / Minor 8。
- Important 1（写路径无数据权限）建议进退回路径或 ADR 显式裁决；Important 2（混合角色 createByOnly 短路）为组件消费语义错位，一行条件 + 用例可修，随 brief/ADR 口径定版一并处理。
- 契约主线（创建语义/五字段不可改/锁定收窄/树裁剪四场景/列表 IN 裁剪/详情 403/M-6 红转绿/脚本幂等与 4030-4035 一一对应）全部达成，越界反查干净。
