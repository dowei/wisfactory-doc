# TASK-0015 scoped 复评报告 round2（代码评审，fresh）

> 评审者：fresh 代码评审子代理（round2 最终评审）；评审日期：2026-11-23。
> 输入：修复 diff 包 `TASK-0015-diff-round2.txt`（e1b6266..cebf17a，单提交 8 文件）、原评审 `TASK-0015-review.md`（I-1/I-2 + Minor①②③④）、修订轮报告 `TASK-0015-report.md`「修订轮」段、实施 brief，及工作区源码直接核验（gate / 版本历史服务 / 状态规则服务与控制器 / 分档规则表 / 实体继承与逻辑删配置 / 多租户插件 / 快照组装器 / 发布调用点）。
> 口径：L1-execute.md §13「diff 评审」复评条款——scoped，只看修复 diff 与原发现的处置，不重开原评审已通过部分。
> 实证基线：HEAD=cebf17a（`git log` 实证）；工作区 iot 模块与 cebf17a 逐字节一致（`git diff cebf17a -- ...iot/` 为空）；仅 `.agents/` 未跟踪目录（非评审对象）。
> **独立实测**：本评审自行重跑 `mvn test -pl ot-admin` → **Tests run: 2298, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**（与修订轮报告 §三.1 数字一致）；四个目标测试类明细 IotTemplateVersionServiceImplTest 17/17、IotStatusRuleServiceImplTest 36/36、TemplateAwarePublishConfirmGateTest 9/9、TemplateTierRuleTableTest 9/9 全绿（surefire 报告实证）。

## 越界反查（复核）

通过。`git diff --name-only e1b6266..cebf17a` 实测 8 文件 = publish/gate 1 + statusrule/service/impl 1（修订授权扩展，且仅 IotStatusRuleServiceImpl 单文件，未触 statusrule 的 controller/dto/mapper）+ template/service·impl·tier 3 + 对应 test 3；无 SQL/yml/ot-core/前端/ingest 触点。全落「原 brief 范围 + statusrule 写路径扩展」授权内。

## 一、复评三问逐条结论

### 问 1：I-1（模板域漂移盲区）是否真修复——**是，修复成立**

- **结论**：复合凭据哈希闭合了模板域漂移盲区；模板域内容任何变更（含逻辑删）必然使哈希漂移 → gate fail-closed 拒绝 → 5002。
- **出处**：`IotTemplateVersionServiceImpl.computeConfirmCredentialHash`（impl:347-359，SHA-256(canonical{"compute","template"})）+ `computeTemplateDomainContentHash`（impl:368-383，租户内全部模板 × buildContentJson 全量内容快照按 templateCode 索引）；下发侧 `previewImpact`（impl:263-267 写 `复合哈希:token`）与校验侧 gate（TemplateAwarePublishConfirmGate.java:108）**共用同一方法同参重算**；测试 `credentialHashChangesWhenTemplateDomainChanges`（红：两预览哈希逐字节相同 b49d0384… → 绿：漂移必现，本轮独立复跑绿）。
- **逐项核验**：
  1. **两侧同参一致性成立**：gate 以发布请求的 (tenantId, scopeType, scopeId, targetVersion) 重算；凭据键含全部四参——发布参数与预览不一致时 Redis 取空先拒（凭据缺失），能走到哈希比对即参数必然与下发侧一致（预览恒写 2:0；发布 scope 非 2:0 时键不存在 → fail-closed）。
  2. **锚定面完整性**：模板域快照覆盖模板头 4 字段（templateCode/templateName/nodeType/remark）+ 数据项全 13 业务列 + 状态规则全 7 列（含 conditionExpr/priority/enabled/bindingMap/statusCode/statusName/color）——与分档规则表映射面一致，无「模板域可变列未入快照」遗漏。
  3. **逻辑删必漂移**：三实体均 extends BaseEntity（BaseEntity.java:39 `deleted` 字段）+ 全局 `logic-delete-field: deleted`（application.yml:180-181）——select 路径自动排除已删行：删模板 → domain TreeMap 条目消失；删规则/数据项 → 清单变化；均必改哈希。
  4. **租户隔离成立**：`modelTemplateMapper.selectList(new LambdaQueryWrapper<>())` 无显式租户条件，但 OtTenantLineHandler 对 `iot_` 前缀表白名单注入 `tenant_id=会话租户`（OtTenantLineHandler.java:46-47,61-70）——域哈希按租户收敛，无跨租户耦合。
  5. **无残留「模板变更不改哈希」路径**：唯一不改哈希的模板域变更是**模板启停（status）**——这是 M-4 裁决的显式排除（「启停是运营动作非内容变更」，javadoc/测试/报告三处固化），非缺口；其余模板域写路径（模板头编辑、数据项增删改、规则增删改、逻辑删）全部漂移。
- **一句话说明**：预览下发与发布校验两侧同参重算同一复合哈希，计算域或模板域任一内容在预览后变更（含逻辑删）即凭据失效——I-1 盲区闭合。

### 问 2：I-2（状态规则写路径绕过留痕）是否真修复——**是，修复成立**

- **结论**：状态规则域四个写原语全部落库后**同事务**钩 `recordContentChange`，T2 规则变更必然产生版本行并经 hasPendingTier2Confirm 触发确认门；钩子遗漏面复核为零遗漏。
- **出处**：`IotStatusRuleServiceImpl`——createRule（impl:202，钩前有 save）、updateRule（impl:252）、saveBindingMap（impl:319）、deleteRules（impl:348-351，`removeByIds` 后按 distinct templateId 逐模板钩）；四方法均 `@Transactional(rollbackFor=Exception.class)`，recordContentChange 事务传播 REQUIRED 加入外层事务——升版/基线失败整体回滚，与模板域写路径同事务语义成立。
- **逐项核验**：
  1. **deleteRules distinct 正确性**：`existingList` 在删除前经 listByIds 全量载入（fail-closed：任一不存在整批拒绝），随后对每条规则的 templateId 去重逐模板钩——批内同模板多规则只钩一次、跨模板批逐一钩；recordContentChange 按该模板删后现状重建内容并 diff 升版，语义正确。
  2. **钩子遗漏面复核 = 零遗漏**：`IotStatusRuleMapper` 在 main 代码中仅两处消费——IotStatusRuleServiceImpl（唯一写方）与 IotTemplateVersionServiceImpl.loadRules（只读）；控制器恰好 5 端点（list/add/edit/binding-map/remove）全部路由到上述四写原语；服务接口仅此 5 方法；「调优先级=删旧建新」走 deleteRules+createRule（均已钩）；全仓 grep `iot_status_rule` 无其他运行时写方（无 seed/其他模块直写）。状态规则域写路径完整收敛于四原语。
  3. **校验失败不钩**：各原语校验（优先级预检/表达式/字典锚定/点位存在等）全部在落库前抛出，钩子在落库后——`validationFailureDoesNotHookVersionTracking` 测试锁定，本轮复跑绿。
  4. **版本行→确认门链路**：`statusRuleConditionChangeProducesTier2Row` 实测版本行 tier_summary=T2:1、条目 statusRule.conditionExpr/T2（本轮复跑绿，服务日志 `template upgraded ... summary=T2:1` 实证）；gate 侧 hasPendingTier2Confirm 对 T2 行的判定为原评审已过部分（pendingTier2Detected 锁定），不重开。
- **一句话说明**：状态规则四个写原语（新增/修改/直绑映射/批量删除）落库后同事务钩升版，域内无其他写路径——T2 规则变更不再绕过版本留痕与确认门，I-2 闭合。

### 问 3：修复 diff 是否引入新 Critical/Important——**否**

- **结论**：8 文件 diff 本身未引入新 Critical/Important；M-1/M-2/M-4 处置实现正确，新观察均为 Minor 登记级（见「二」）。
- **逐项核验**：
  1. **复合哈希正确性/性能**：两侧同一方法（非复制实现）——一致性无分歧面；CANONICAL 复合（TreeMap + ORDER_MAP_ENTRIES_BY_KEYS）确定性成立；性能 N+1（每模板 items+rules 两查）为真实代价但**修订轮报告 §六.4 已登记**（租户模板数量级十位、仅二档路径+预览触发、缓存演进点明确）——非新发现。
  2. **M-1 四步校验畸形值 fail-closed 成立**（gate:98-103）：旧协议裸哈希（无 `:`，separator=-1）、前导 `:`（separator=0）、尾随 `:`（separator=length-1）、多 `:`（storedHash 含 `:` 必与 64 位十六进制期望不符）、token 段任意构造——全路径拒绝；签发 token 为 UUID 去横线十六进制（不含 `:`），lastIndexOf 分段协议自洽；测试 token 不匹配拒/畸形拒/匹配绿三向锁定，本轮复跑绿。复合哈希比对先于 token 比对，次序无放行面。
  3. **M-2 4002 映射完整**：`insertVersionRow` 收口三处 insert（recordBaseline:156 / 无基线补录:183 / 升版主插入:202）——无遗漏插入点；DuplicateKeyException → BusinessException(4002) 与同域 insertTemplate/saveItems 范式一致；@Transactional(rollbackFor=Exception.class) 下败者整事务回滚语义不变；测试锁定，本轮复跑绿。
  4. **M-4 语义一致性成立**：status 移出 buildContentJson 模板头（impl:402-412 现 header 仅 4 字段）→ 仅启停经 updateTemplate 钩后内容逐字节一致 → 零版本噪声（`statusToggleAloneProducesNoVersionRow` 实证返回 null 零升版）；规则表保留 template.status→T1（TemplateTierRuleTable.java:60 + 兼容注释）——0015 首版历史行的 status 键 diff 出 T1 条目，不误触未登记字段 fail-closed 二档默认，T2 门不被误开；`statusToggleMixedWithFormulaChangeOnlyReportsFormula` 实证混合保存仅公式条目。规则 `enabled` 与模板 `status` 的不对称（前者入快照 T2、后者不入）是 ADR-0008 语境下的有意区分（规则启停改变判定流、模板启停是生命周期动作）——javadoc/测试/报告三处固化，自洽。
  5. **gate 异常传播面（edge，fail-closed）**：computeConfirmCredentialHash 在 DB/序列化故障时抛 BusinessException，gate 布尔契约下异常直接传播（发布以非 5002 业务异常失败而非放行）——fail-closed，触发条件为真实故障，无放行面，不构成发现。
- **一句话说明**：8 文件 diff 的实现与测试互证（全量 2298 绿为独立实测背书），未引入新 Critical/Important。

## 二、新发现清单

### Critical

无。

### Important

无。

### Minor（登记级，均不阻断）

- **N-1 确认凭据锚定面不含状态字典等引用域（登记性观察，非本轮 diff 引入）**。出处：computeTemplateDomainContentHash（impl:368-383，锚定=模板+数据项+状态规则）+ ConfigSnapshotAssembler（:146-196，快照装载=channels/points/credentials/dataItems/refs/metrics，无 iot_status_dict）。预览→发布窗口内禁用/启用状态字典编码（或直绑映射引用的点位等引用域变化）不使凭据漂移；但引用域内容本身也不入发布快照（发布内容不含字典），属发布链外的域生命周期——与本 diff 的复合锚定设计边界一致，非缺口回归。登记备忘：若后续把字典/引用域纳入发布内容或 T2 语境，需同步评估凭据锚定面（同 M-3 防遗忘条目格式）。
- **N-2 复合哈希 N+1 装载的 gate 侧放大面（已登记项的复核确认）**。出处：修订轮报告 §六.4 已登记 + gate:108 每次二档发布尝试调用。复核结论：报告登记准确（触发频率=仅二档待确认路径、租户模板数量级十位、缓存失效点=模板域四写路径钩子处已指明）——无需新增动作，此处仅确认登记存在且口径正确。

## 三、结论

**通过。** 复评三问全部成立：I-1 复合凭据哈希闭合模板域漂移盲区（两侧同参同法重算、逻辑删必漂移、唯一不漂移路径=模板启停为 M-4 显式裁决）；I-2 状态规则四写原语同事务钩升版且域内写路径零遗漏；修复 diff 无新 Critical/Important（M-1 fail-closed 四步校验、M-2 三插入点全收口 4002、M-4 启停零噪声+规则表 T1 兜底均实证成立）。原评审 Minor①②（M-1/M-2）已修、③（M-3）按裁决 deferred 登记、④（M-4）按裁决语义落地——处置与协调者裁决一致。全量 2298 测试 0 失败 0 错误为本评审独立实测（非转抄）。新发现仅 2 条 Minor 登记级（N-1/N-2），均不阻断；修订轮报告的偏离登记（M-4 执行口径说明、凭据协议变更无迁移需求、gate 日志文案）与工作区源码实况一致。
