# TASK-0015 diff 评审报告（代码评审，fresh）

> 评审者：fresh 代码评审子代理；评审日期：2026-11-23。
> 输入：diff 包 `TASK-0015-diff.txt`（隔离区间 d02ac44..HEAD，29 文件）、实施 brief、执行者报告、TASK 详情、ADR-0008 / FEAT-0005 / REQ-0006 / TC-0005 / FEAT-0008、上游 `TASK-0024-report.md`，以及工作区源码直接核验（gate / publish 服务 / 模板域 / 快照组装器 / 分档引擎 / DDL / ResultCode / 状态规则域）。
> 口径：L1-execute.md §13「diff 评审」（与 TASK 验收标准 / brief 接口的一致性 + 明显质量缺陷）。
> HEAD=b3d1e0c 实证（git log）；工作区仅 ingest/** 在途改动（TASK-0027 Go 道，与本评审无关）。

## 一、评审重点逐条结论

1. **分档引擎 vs ADR-0008 逐条一致性——吻合**。规则表（TemplateTierRuleTable.buildFieldTiers）逐字段对照 ADR 三档：T1=新增数据项/新增状态规则（对象级 ADDED 表）/dataName/category/templateName/remark/nodeType/status/statusName/color；T2=defaultFormula/aggMethod/defaultCalcPeriod/defaultValidMin/Max/defaultAlarmLow/High/unit/conditionExpr/priority/enabled/bindingMap/statusCode+删除状态规则（二档——既有判定结果改变，合理推论）；T3=REMOVED dataItem/valueKind/defaultSourceType；FORBIDDEN=dataCode/templateCode 登记+写路径实拒（IotDataItemTemplateServiceImpl.java:178-180 dataCode 不一致即 400，templateCode 同范式 IotModelTemplateServiceImpl.java:173-175）。未登记字段保守默认 T2（fail-closed），表驱动可演进成立。ADR 提及的「显示顺序」「可覆盖参数默认值」在 iot_data_item_template 现有 13 业务列中无对应列（DDL 实证）——见发现 5。
2. **逐条目判档——成立**。diffObjectFields 逐字段独立出 CHANGED 条目、对象级 ADDED/REMOVED 各一条，无最高档合并；TemplateContentDiffEngineTest.mixedFieldsOnOneObject/mixedSavePerEntryTiers 与 IotTemplateVersionServiceImplTest.recordContentChangeMixedTiers（T1:1,T2:1）锁定。
3. **版本历史钩子覆盖面——模板头与数据项写路径完整，状态规则漏钩（发现 2）**。枚举：创建（insertTemplate→recordBaseline:156，幂等）/更新（updateTemplate→recordContentChange:210，version 不进更新集改由服务权威升版——REQ-0006「递增留痕」依据，既有测试按新语义改写并登记）/数据项增（saveItems:165）改（updateItem:208，templateId 不进更新集故单模板钩子足够）删（deleteItems:225-228 逐 distinct 模板）/删除模板（deleteTemplates 无钩子——删除前置保护要求零实例零数据项，历史行孤儿留存不可达、对 gate 判档无影响，可接受）。基线/补录/无变化零噪声三语义实现+测试锁定；同秒窗口边界按「已发布」处理（1 秒误差窗，执行者登记）。并发安全：uk_template_version 物理兜底、败者整体回滚数据安全——但无 DuplicateKeyException 友好映射，见发现 4。createTime 由 MetaObjectHandler INSERT 填充（MybatisPlusConfig.java:66-68），hasPendingTier2Confirm 窗口比较无 NPE 风险。
4. **影响预览——计数分列与锁定口径正确，previewToken 锚定机制成立但有漂移盲区（发现 1）**。affected=同租户引用且启用总数、locked 子集单列（ADR-0008 锁定口径，VO javadoc+单测 previewImpactCountsAndToken/previewImpactEdgeCases 锁定）；diff 基线=未锁定跟随实例最小有效版本（累计待生效口径，锁定滞后归 0016 待升级清单）；存量模板无基线时 diff 留空不误计全量 ADDED（FEAT-0005 边界口径）。previewToken：Redis 键 `iot:pub:confirm:{tenantId}:2:0:{targetVersion}`、TTL=ot.publish.confirm-token-ttl-seconds（默认 300）、value=计算域快照 SHA-256，gate 侧同链路重算比对（TemplateAwarePublishConfirmGate.java:93-94 与 IotTemplateVersionServiceImpl.computePublishContentHash:369 同一 assemble/serialize/sha256 链路）；并发发布使 targetVersion 错位 → 键不匹配 fail-closed。
5. **gate 真实现——装配/签名/放行语义成立**。NoOp 去 @Component 退役为测试替身（IotPublishServiceImplTest 显式构造），生产唯一 Bean=TemplateAware（grep 实证 implements PublishConfirmGate 仅两实现、@Component 仅 TemplateAware）；接口四参签名不动+五参 default 委托，0024 调用方/测试零改动（fourArgSignatureStillSupported 锁定）；四参覆写=无凭据同判档 fail-closed。**网关 scope 恒放行理由成立**：ConfigSnapshotAssembler 网关域仅 channels+points 两键（javadoc:40-45+代码 120-126），模板内容（数据项/公式/指标）只经计算域快照进入发布链，且按实例有效版本行组装——0024 组装口径交叉核实一致，快照天然不含未升级实例内容。**仅二档触发**：hasPendingTier2Confirm 两条件（T2 行窗口开启+滞后未锁定启用实例）与 brief「本次待发布内容相对实例有效版本存在 T2 条目」口径一致（计算域=租户粒度发布，租户内任一模板 T2 待确认即需确认——正确不漏）；三档不入判定（pendingTier2OnlyForTier2 锁定）。**T2 窗口关闭语义（执行者登记取舍）自洽**：「已经过一次计算域发布」关闭（行 create_time vs 计算域末版 publish_time，publish_time 缺省 fail-closed 按未发布）；「发布后模板又改但未再发布→下一次发布正确要求确认」经代码路径与单测 pendingTier2Detected/pendingTier2WindowClosed 双向验证成立。窗口语义与凭据校验的合成效果有缺口，见发现 1。
6. **5002 登记——闭环**。ot-core ResultCode 仅一处触点（PUBLISH_NOT_CONFIRMED(5002)，与 5001 相邻无冲突）；IotPublishServiceImpl.java:183-187 gate false→BusinessException(5002)→响应码，冒烟步骤 5 实测 5002。FEAT-0008「未确认 5002」兑现。
7. **回归面——成立**。NoOp 替换对一档/无模板变更路径零行为变化（hasPendingTier2Confirm=false 直接放行、网关恒放行，不触 Redis）；执行者实测 2286 全量绿（报告验收 1，含 0024/0025/0026 既有发布测试）。
8. **SQL 与 yml——逐字达标**。20261123_iot_template_version.sql DDL 与 brief 逐字一致（COLLATE 为 MySQL 8 默认补齐，脚本与 brief 均未写——等价非差异）；头注登记「附录 A 无此表——ADR-0008 落地件」+菜单种子无新增（复用 iot:template:query 4025）；二次幂等实测。application.yml 仅 confirm-token-ttl-seconds 一键，与 PublishProperties.confirmTokenTtlSeconds 绑定（prefix=ot.publish）。
9. **越界反查（复核）——干净**。29 文件净足迹全落 brief 声明；b3d1e0c 回退后 IotPublishController 不在净足迹（diff 包与报告文件清单双证）。

## 二、发现清单

### Critical

无。

### Important

- **I-1 内容漂移校验对模板域变更不生效——预览后新增的 T2 模板变更可随旧凭据过门，未经确认即发布**。
  出处：TemplateAwarePublishConfirmGate.java:93-94,107-111（contentHash=计算域快照 SHA-256）+ IotTemplateVersionServiceImpl.java:369-374,251；根因交叉证据：ConfigSnapshotAssembler.java:40-52（计算域快照装载源=iot_data_item/iot_data_item_ref/iot_metric 实例行）与模板域零写实例行（grep 实证 template 包无 IotDataItemMapper 引用）。
  说明：模板域编辑（模板头/模板数据项/状态规则）不触碰实例行，故预览→再改 T2→凭 TTL 内旧 token 发布时哈希不变、凭据仍在、窗口开启仅要求「token 非空」→ 门放行，第二个 T2 变更从未被任何 diff/预览/确认覆盖即随发布关闭窗口——与 gate javadoc 自述「内容漂移（预览后待发布内容已变更）→ false」的保证不符，也弱化 ADR-0008 T2「改错一条公式全厂错」的确认防线（对实例/点位/通道域变更漂移校验仍有效，TTL 300s 且仍强制「TTL 内发生过一次预览」为部分缓解；brief 协议将 value 固定为「待发布内容摘要哈希」是缺口的设计来源，执行者照实现并如实记载——处置宜修订轮将凭据哈希锚定改含模板当前内容摘要，或窗口判定纳入「预览后无新 T2 行」）。
- **I-2 状态规则写路径未钩升版——T2 状态规则变更整类绕过版本留痕与确认门**。
  出处：net/otlab/admin/module/iot/statusrule/service/impl/IotStatusRuleServiceImpl.java（createRule:136 / updateRule:195 / saveBindingMap:241 / deleteRules:305 均无 recordContentChange；grep statusrule 包零引用 templateVersionService）。
  说明：规则已纳入 content 快照、diff 引擎与规则表已映射其档位（conditionExpr/priority/enabled/bindingMap=T2、删除=T2），但独立保存规则只改 iot_status_rule——不升版、不开 T2 窗口、gate 判档永远看不到，ADR-0008 T2 明文含「状态规则门限/优先级」、REQ-0006 验收 2 覆盖全部 T2；执行者已在报告「五」3 登记（brief 改动范围仅 template/**，statusrule/** 不可动——brief 继承缺口非执行越界，接线建议已给），处置建议按 §12 建后续接线项（修订轮或 0016），非本 diff 退回项。

### Minor

- **M-1 confirmToken 值不被校验——凭据存在时任意非空字符串可通过**。出处：TemplateAwarePublishConfirmGate.java:81（仅判非空）+87（凭据键不含 token、值仅内容哈希——brief 协议原样）。确认门实际语义=「TTL 内发生过一次影响预览+实例级内容未漂移」，token 本身无绑定权重；brief 继承弱点、javadoc 如实记载，建议后续把 token 并入 Redis 值（如 hash:token）使凭据可验。
- **M-2 同模板并发保存撞 uk_template_version 无友好映射**。出处：IotTemplateVersionServiceImpl.java:191（insert 无 DuplicateKeyException 捕获）。两并发保存同 templateId 时败者以原始 500 失败（整体回滚、数据安全；对比同域 insertTemplate:158-161 / saveItems:157-162 的 4002 兜底范式），低频场景建议照双道闸范式补 catch「请重试」。
- **M-3 ADR-0008 两类字段暂无 schema 对应物，规则表对「显示顺序/可覆盖参数默认值」的 T1/T2 映射目前不可达**。出处：TemplateTierRuleTable.buildFieldTiers（映射表）vs iot_data_item_template DDL（20261122_iot_template_domain.sql:121-145，13 业务列无 sort/override 列）。内容 JSON 已覆盖全部现有可变列、未知字段 fail-closed 默认 T2 可兜未来新增列——届时须同步纳入 content 快照与规则表，登记性观察防「加了列忘了快照」。
- **M-4 模板启停（status）计入内容快照——每次启停产生 T1 版本行**。出处：IotTemplateVersionServiceImpl.buildContentJson（header 含 status）+ TemplateTierRuleTable（template.status=T1）。启用/禁用属生命周期操作而非「内容」，会制造版本历史噪声并轻微放大 hasPendingTier2Confirm 扫描面；无害且可演进（表驱动改一档映射即可），登记观察。

## 三、结论

无 Critical；Important 2 项（I-1 凭据哈希锚定面的模板域漂移盲区、I-2 状态规则钩子缺口——后者 brief 继承且已登记）。

按 brief 要求「若整体无 Critical/Important，明确给出通过结论」——本评审存在 Important 发现，故**不给「通过」**：I-1 属实现可修（修订轮调整凭据哈希锚定面或窗口判定补「预览后无新 T2 行」条件，建议连同 M-1 一并定协议）；I-2 属 brief 范围外接线缺口（建议建后续接线项，不退回本 diff）。其余核对项（分档逐条一致性、逐条目判档、钩子覆盖、预览分列、gate 装配/签名/放行语义、窗口语义、5002 闭环、SQL/yml、越界反查、回归面）全部成立，工程质量与登记纪律良好。
