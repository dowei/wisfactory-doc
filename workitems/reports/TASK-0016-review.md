# 代码评审：TASK-0016 diff（实现提交 082d694，BASE=b4e877a，13 文件）

> 评审者：fresh 代码评审子代理（依据 brief + 工作区源码独立核实，未共享协调者上下文）。
> 评审口径：`L1-execute.md` §13「diff 评审」（与 TASK 验收标准/brief 接口一致性 + 明显质量缺陷）。
> 工作区 HEAD=082d694、工作树干净（仅无关 untracked），diff 包与工作区源码一致。
> 路径约定：以下短名均指 `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\` 下的类。

---

## 一、逐项核对结果（brief 评审重点 8 项）

### 1. 锁定开关 —— 通过

- `PUT /iot/instance/lock` 挂 `@SaCheckPermission("iot:instance:edit")`（IotInstanceLockController.java:57），复用既有种子、无新增菜单，符合 brief 决策 1/4。
- locked 列白名单更新：更新实体仅 setId+setLocked，MP null 不进 SET（IotInstanceUpgradeServiceImpl.java:112-115），杜绝 mass assignment；服务层对 locked 0/1 再校验（:100-104，bean 校验外的防御兜底）；单测断言更新集不含 instanceName/status。
- 数据权限照 0012 写路径防线：`requireDataPerm` 与 0012 同文案同口径（IotInstanceUpgradeServiceImpl.java:330-335 vs IotModelInstanceServiceImpl.java:680-685），先于任何状态变更（:110）；403 文案不泄露实例标识。
- 「locked=1 编辑仅允许 status」接口层校验属 0012 既有逻辑（IotModelInstanceServiceImpl.java:319-341），本 diff 未触碰该文件——冒烟 STEP5b/5c 已实证，与本任务「锁定开关是锁定列唯一管理通道、不受该限制」的自洽性成立（否则锁上解不开）。
- 路由共存核实：既有 IotModelInstanceController 映射面为 GET tree/list/{id}、POST ""、PUT ""、DELETE /{ids}（IotModelInstanceController.java:45-93）——PUT /iot/instance/lock 与之方法+路径组合无冲突；偏离 1（新控制器落位 template/controller）合理：IotModelInstance 实体与 Mapper 本在 template 包、instance/controller 在 brief 范围外，落位与依赖方向一致。

### 2. 待升级清单 —— 通过

- 筛选正确性：装载后内存判定，applied==current 跳过（IotInstanceUpgradeServiceImpl.java:161）、applied==null 脏数据防御跳过（:153-156）、applied>current 回退防御跳过并 WARN（:157-159, :171-174）——三类边界语义齐备且可测，符合 brief 决策 5。
- diff 复用 0015 引擎只读复用：`TemplateContentDiffEngine.diff(baseline, currentNode)`（:225）；按 applied 版本缓存（:190-193，同版本只算一次，单测 verify selectOne times(1)）。
- **canonical 构建器正确性（重点核实项）**：`buildCurrentContentJson`（IotTemplateVersionServiceImpl.java:294-301）重新 selectById 模板 + loadItems + loadRules 后调用与 `recordBaseline`/`recordContentChange` 落库**同一个私有 `buildContentJson`**（:419-466，TreeMap 键序 + 清单业务键排序 + NON_NULL）——当前侧与版本历史行 content 逐字节同构，diff 相等性前提成立；既有测试新增契约用例 `buildCurrentContentJsonMatchesBaselineContent` 直接断言与 recordBaseline 落库 content 相等。基线行按 (templateId, applied) 精确等值查询（:215-219），uk 物理唯一保证 LIMIT 1 无歧义。
- 无基线降级：历史行缺失或 content 空白 → diff 留空（:218-220），不误计全量 ADDED，同 0015 影响预览口径。
- 分页：total=裁剪后滞后总数、records=内存切片（:176-183），Page 字段（current/size/pages）由 MP Page 组装，单测覆盖 15/5/第 3 页与越界页；偏离 4（内存分页，模板内十~百量级）与 0012 树裁剪范式同口径，登记合理。
- 数据权限读裁剪：可见集空且无 createByOnly 短路空页（:131-137）；`isReadableByUser` = allTenant ∨ id∈可见集 ∨ (createByOnly ∧ create_by=本人)（:319-327），与 0012 实例列表 OR 分支（IotModelInstanceServiceImpl.java:596-609、详情 :664-668）逐字对齐（ADR-0016 裁决 3/4）。
- instanceCode 增量字段（偏离 5）：纯展示、不影响契约字段面，登记合理。

### 3. 确认流 —— 通过

- 校验链 fail-closed 顺序正确：空批 400（:234-236）→ 存在性（缺一整批 400，:240-246）→ 写路径数据权限（任一不可见 403 整批拒，:249-256）→ 批内同模板（含 null templateId 组织节点防御，:259-264）→ targetVersion==模板当前版（:272-275）→ 逐台状态校验（null/回退/已在目标版，:280-293）→ 逐台乐观锁更新。
- 整批原子：`@Transactional(rollbackFor = Exception.class)`（:230-231），循环内任一 BusinessException 整批回滚，无半成功路径。
- 乐观锁：`SET applied=target WHERE id=? AND applied=旧值`（:294-297），affected=0 → 4002 业务码（:299-304，ResultCode.DUPLICATE_CODE=4002 已核实 ResultCode.java:23），非裸 500，0015 M-2 范式。
- **applied==target 按 4002 前置拒绝（执行者偏离 3）核实成立**：MySQL Connector/J `useAffectedRows` 两态下同值 UPDATE 的影响行数口径歧义使 `rows==0` 判定不可靠，显式前置判定（:288-293）fail-closed 是正确处置；幂等重放提示刷新清单，冒烟 STEP7c 实证。
- locked 实例可确认：确认链路无 locked 拦截（:279-309），单测与冒烟 STEP8a（E6C locked=1 确认通过）双实证——锁定实例的升级通道语义正确。
- 留痕：控制器 `@OperateLog`（IotTemplateUpgradeController.java:131）请求级 + 服务逐台 `[INSTANCE-UPGRADE]` INFO（:306-308），冒烟 STEP10 sys_operate_log 五条实查含失败如实记。
- 权限面（偏离 2）：确认挂 iot:instance:edit（实例写操作）、清单挂 iot:template:query——视线与动作分离，javadoc 登记理由充分；且确认写路径防线维持 instanceIds 口径、**不开放 createByOnly 分支**（:250-256），与 0012「写原语不得借 create_by 分支穿透」教义一致（IotModelInstanceServiceImpl.java:660-668 javadoc）。

### 4. 并发安全 —— 通过（2 项 Minor 见发现清单）

- 确认 vs 确认：并发推进同实例 → 守卫 WHERE applied=旧值 失配 → affected=0 → 4002 整批回滚，败者数据不变。
- 确认 vs 锁定开关：两者更新列集不相交（confirm 仅 applied_template_version；setLock 仅 locked，白名单实体 null 不进 SET），单行 InnoDB 行锁下无丢失更新。
- 确认 vs 发布组装：快照组装读 iot_data_item/iot_data_item_ref/iot_metric 行（ConfigSnapshotAssembler.java:191-214），确认写 iot_model_instance.applied——不同表，组装读不到「新旧半态」；确认门对 applied 的读取是判档咨询面，最坏一轮多余的 5002/放行，无数据损坏。
- 模板升版与确认的 TOCTOU 见 Minor-3（后果良性自愈）。

### 5. 5002 确认门联动 —— 通过（无意外联动）

- 新三接口不 import publish 域、不调用 `PublishConfirmGate`/`hasPendingTier2Confirm`；5002（PUBLISH_NOT_CONFIRMED，ResultCode.java:38）仅在发布链（IotPublishServiceImpl.java:179 确认门接线）触发。
- 确认对门的影响是**间接结果面**（applied 推进 → 门条件② `lt(applied_template_version, row.version)` 计数归零，IotTemplateVersionServiceImpl.java:326-331），即 brief 所述「簿记+数据面编排入口」语义，无意外联动。

### 6. 执行者裁决项独立核实 —— **成立**（见发现清单 Important-1）

按 brief 要求独立核实「数据面生效缺口」外推，代码事实全部确认，且比执行者登记的更尖锐（见发现清单）。协调者据此立后续任务的方向正确。

### 7. 0015 引擎复用无改动面 —— 通过

- diff 13 文件中不含 TemplateContentDiffEngine.java / TemplateTierRuleTable.java / TemplateChangeEntry.java / TemplateChangeTier.java——引擎与 tier 表零改动，纯只读复用。
- 版本服务两文件（接口 + 实现）的改动为**纯增量**：新增 `buildCurrentContentJson` 声明与实现各一方法，既有方法体零改动（diff hunk 仅 +11/+17 行，无删改）；既有测试文件仅追加 2 用例。

### 8. 数据权限/租户插件一致性 —— 通过

- 租户边界：新服务全部查询走 iot_ 前缀表（iot_model_instance / iot_model_template / iot_model_template_version / iot_data_item_template / iot_status_rule），`OtTenantLineHandler` 前缀规则自动注入 tenant_id 条件（OtTenantLineHandler.java:46-47, 61-70，SELECT/UPDATE/DELETE 均覆盖）——清单、diff 基线行、确认更新、锁定更新全部租户隔离；跨租户 id 查询返回 null → 400「不存在或已删除」，与存在性不可区分，无跨租户存在性泄露。
- 数据权限：读裁剪 OR 分支与 0012 实例域读范式对齐、写防线 instanceIds-only 与 0012 写教义对齐（见第 1/2/3 项）。
- 设计内不对称（非缺陷，提示 0070 前端）：纯 scope=4 用户经 createByOnly 分支可在清单看到本人实例，但确认走写路径防线必 403——「看得到、确认不了」符合平台教义，前端确认按钮的可用态需按动作权限而非视线渲染。

---

## 二、发现清单

### Important

- **[Important-1]（非本 diff 实现缺陷——brief 前提偏差/跨任务编排缺口，执行者裁决项成立，供后续任务立题）确认升级仅推进版本簿记，「确认后按新版本生效」在数据面未兑现，且现有 regenerate 通道也补不上 T2 值传播。**
  代码事实链（独立核实）：
  1. 快照组装读实例行、不读 applied 列：`assembleComputeScope` 直查 iot_data_item / iot_data_item_ref / iot_metric（ConfigSnapshotAssembler.java:191-214），javadoc 自述「iot_data_item 行即实例的有效配置内容……组装器读行即按实例有效版本合成（无需特判）」（ConfigSnapshotAssembler.java:48-52）——该「无需特判」前提依赖实例行与 applied 版本对齐。
  2. 确认只写簿记：confirmUpgrade 的更新集仅 `applied_template_version`（IotInstanceUpgradeServiceImpl.java:294-298），全程不触 iot_data_item。
  3. 由此：确认后台离开待升级清单（applied==current，IotInstanceUpgradeServiceImpl.java:161）、确认门条件②归零（IotTemplateVersionServiceImpl.java:326-331），但该实例下次发布快照内容仍为旧版内容——TASK-0016 验收标准「确认后按新版本生效」只兑现了留痕一半；brief 决策 3 的前提「确认后实例按新版本进下次发布快照（0024 组装天然兑现）」与代码事实不符。
  4. **比执行者登记更尖锐的一点**：现有逃生通道 regenerate（IotDataItemController.java:69-72 → IotDataItemServiceImpl.java:362-370）只是 `generateForInstance` 的写权限包装，而 generate 对已占用 (instance_id, data_code) 行幂等跳过、只补插缺失项（IotDataItemServiceImpl.java:218-226），且行值在生成时即从模板 default 复制落库（buildItem，IotDataItemServiceImpl.java:1090-1114）——因此即使「确认+手工 regenerate」两步，也只兑现一档式**新增项落行**，T2 默认公式/有效范围/阈值向既有行值的传播、T3 valueKind/defaultSourceType 的结构迁移均无任何机制承载。
  定性：执行者按 brief 字面交付并如实登记，属正确处置；缺陷主体是 brief 前提（0024「天然兑现」）与平台既有数据面机制（生成即复制、跳过占用行）之间的语义缺口，不是本 diff 引入的代码缺陷。**处置建议：不退回本 diff；按协调者已定方向立后续任务收口**（裁决「确认时自动数据面编排 vs 显式两步」时需同时裁决：既有行 T2 值的重对齐机制——重写 vs 运行时继承——因为 regenerate 只补插、不重对齐，且 buildItem 的「生成即复制」与 0014「运行时继承（非复制落库）」教义在字段面上并存的不一致性也在此一并裁决）。

### Minor

- **[Minor-1] setLock 静默 no-op 竞态**：控制器忽略 `setLock` 布尔返回值（IotInstanceLockController.java:58-61）；实例在 selectById（IotInstanceUpgradeServiceImpl.java:104-108）与 updateById（:115）之间被并发逻辑删 → updateById（@TableLogic deleted=0 条件）影响 0 行，API 仍返回成功而实际未置位。窗口极窄、后果良性（实例已删，锁状态无意义），与 0012 编辑路径同范式；记录备查。
- **[Minor-2] 清单含禁用实例**：selectUpgradeList 查询无 status 过滤（IotInstanceUpgradeServiceImpl.java:135-139），禁用滞后实例也出清单、也可被确认；而影响预览 affected 只计启用（IotTemplateVersionServiceImpl.java:226）、确认门滞后计数只计启用（:328-330）。实现与 brief 行定义字面一致（brief 未限定 status），非违约；登记供协调者/0070 前端知晓（可考虑清单加 status 列或筛选）。
- **[Minor-3] 模板当前版 TOCTOU（后果良性）**：confirmUpgrade 读 `template.getVersion()`（IotInstanceUpgradeServiceImpl.java:266-275）与逐台更新之间无并发护栏，若并发 recordContentChange 升版，确认可能把 applied 压到过期「当前版」。后果自愈：实例重新出现在清单、版本历史单调、applied 乐观锁守卫不受影响，无数据损坏；无需本期处置。
- **[Minor-4]（0015 既有兼容行为的新暴露面）**：M-4 修订前落库的历史行 content 仍含 status 键，此类模板的清单 diff 会多出一条 template.status 的 T1 条目（TemplateTierRuleTable.java:57-60 兼容映射所预期）。纯展示噪声、不误触二档判档，0015 已登记的兼容口径在本次新消费面重现；无需处置。

---

## 三、结论

**实现层面：通过。** 13 文件全落 brief 声明范围（无 SQL/yml/ot-core/publish/前端/ingest 越界）；brief 三接口与测试口径逐项对上；锁定白名单/确认原子性/乐观锁 4002/权限租户一致性/0015 引擎只读复用/canonical 构建器同源等关键面经工作区源码逐点核实无缺陷；7 项登记偏离均有依据且处置得当。

发现总计：**Critical 0 项；Important 1 项（Important-1，即执行者登记的数据面生效缺口——经独立核实成立且已尖锐化为「regenerate 亦不能承载 T2 值传播」，属 brief 前提偏差/跨任务编排缺口，非本 diff 实现缺陷，建议不作为退回依据、作为后续任务立题输入）；Minor 4 项（建议记 TASK `任务结果` 的 deferred，收尾进 LOG）**。

按 §13 口径：本 diff 无实现缺陷级 Critical/Important，可收尾；Important-1 的后续任务（确认→数据面生效编排 + T2 值传播机制裁决）为 REQ-0006 验收 3 完整闭环的前置。
