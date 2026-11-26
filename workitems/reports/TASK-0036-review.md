# TASK-0036 diff 评审报告（fresh reviewer）

- 评审对象：`TASK-0036-diff.txt`（单提交 9e73d3e，BASE 071dd1f，19 文件 / +2533 −17，主码 15 + 测试 4）
- 评审依据：评审 brief §依据（L1-execute §13 两样：brief 契约一致性 + 明显质量缺陷）；实施 brief 决策 1-8；0013 `RelativeRefExpander` 同口径交叉核对；ADR-0008/0009/0010；DDL `20261122_iot_data_item_domain.sql`（uk_item_ref 实查）
- 完整读入：diff 包 2840 行全文；brief/TASK/报告/ADR/L1 §13/Expander 全文；实现中引用的既有件（overrideParams、resolveParent、requireDataPerm、isInstanceReadableByUser、firstResidualToken、duplicateItem4002、mapCalcPeriod、selectOccupancyByInstance、BaseEntity @TableLogic、IotPoint.status、模板四字段、前端 0057 占位 emit 契约）逐项实查
- 越界反查：`git diff --name-only 071dd1f..HEAD` = 19 文件全落 `dataitem/**` 声明范围（instance/** 未触达，继承链经既有 effective-params 完成，符合 brief 改动范围 1/3 行）

## 结论汇总

**Critical 0 / Important 0 / Minor 8**。brief 决策 1-8 全部落地（偏离 1-5 均已登记且可辩护）；校验链与 0013 Expander 实测同口径；先删后插原子性、租户隔离、SQL 参数化、403 防泄露、权限注解⊆sys_menu 登记集均核实；brief 测试口径逐条覆盖（37+8+4+8=57 例与报告一致，全量 1923=1866+57 闭环）。以下 8 条 Minor 均不阻塞收尾，建议记 TASK `任务结果` 的 deferred（收尾进 LOG）。

## 发现清单

1. **Minor** / 出处：`IotDataItemServiceImpl.saveFormula` 步骤 4（DFS 建图读为快照读，约实现文件 770-800 行） / 一句话说明：**跨项并发保存可滑过环检测**——T1 存 A→B 与 T2 存 B→A 同窗提交时双方 DFS 均只见提交前图、双双放行落库成环；brief 点名的「同一项 ref 重建交错」已被 `deleteByItemIdPhysical` 行锁 + uk_item_ref + DuplicateKeyException→4002 重试兜住（核实成立），跨项无锁属缺口，FEAT-0012 计算引擎为设计上的最终防线（0013 Expander javadoc「循环检测归 FEAT-0012」），建议登记 LOG 并在 calc 引擎侧兜底或后续评估 FOR UPDATE/租户级串行化。

2. **Minor** / 出处：`FormulaCycleDetector.dfs`（递归实现） / 一句话说明：**深图递归栈溢出未防护**——递归深度=引用链长度，租户内长链（跨 PARENT/绝对键多层）极端配置下 StackOverflowError→500；校验先于写入且 Error 触发回滚、无脏写（fail-safe），javadoc 已自登记「量级为配置面小图」，建议改显式栈迭代消除该上限。

3. **Minor** / 出处：`IotDataItemServiceImpl.collectBareFullKeyRefs`（BARE_FULL_KEY 正则） / 一句话说明：**裸键形态兼容的收集盲区**——任意点分 token（含字符串字面量内）都会被当作引用入图（ref_item_id=NULL 边），而无点裸标识符引用（如 `SOMEVAR + 1`）被完全忽略、可零边入库，运行期才失败；与 0013「绝对键不核存在」一期口径同源、真实 Aviator 校验归 FEAT-0012，登记防漏。

4. **Minor** / 出处：实施 brief 决策 1 字面 vs `saveFormula` 存在性核对实现（报告偏离 5） / 一句话说明：brief 写「绝对键逐个核对本实例与父实例数据项」，实现取「仅指回本/父实例前缀的绝对键强制核对（4001）、指向第三方实例不核（0013 被引实例可能后建口径）」——实现与 0013 基线一致、有测试双证（`testSaveFormula_ThirdInstanceRefUnchecked`），但属对 brief 字面的解释取舍，需协调者显式确认接受该口径。

5. **Minor** / 出处：`saveFormula` 租户内两次全量查询（`this.list` 全部数据项 + `selectList` 全部 ref_type=1 边）与 `getItemImpact` 入向边查询 / 一句话说明：每次公式保存按 O(租户) 两查建图、impact 入向边无分页——当前配置面量级可接受，项数/边数规模增长后需按实例/ancestors 子图裁剪或加 LIMIT。

6. **Minor** / 出处：`IotDataItemController.bind` 权限注解（§3.1 行 `iot:item:bind` vs 实现 `iot:item:edit`，报告偏离+遗留 3） / 一句话说明：独立绑定权限码未落——sys_menu 无该码、resources/sql 不在本批改动范围、SaPermissionRegistryConsistencyTest 强制注解⊆登记集，复用 edit 是范围内唯一合理解，但需 SQL 扩行任务真正收口（评审确认跟踪在位，勿悬空）。

7. **Minor** / 出处：`saveRollup` 出边清空口径 + `FormulaCycleDetector` 图构成 / 一句话说明：**经 rollup 项的概念环不在本期 DFS 范围**——汇总子级为计算期动态纳入（brief 决策 1/4，ref 不落子级边），「子项公式引用上级 rollup 项 + rollup 动态纳入该子项」的环在 ref 图上不可见，brief 决策 8 亦未含此类；FEAT-0012 计算期检测是兜底，登记防下游漏配。

8. **Minor** / 出处：`IotDataItemServiceConfigTest.testSaveFormula_FunctionWhitelist4001`（`assertEquals(4002 + (-1), ...)`）、`testImpact_PeerInstanceInvisibleTrimmed`（末行 `vo.toString().contains` 弱断言）、`CODE_VALIDATION_REJECTED=4001` 域内常量 / 一句话说明：测试与杂项小瑕——4001 以算式混淆书写、一处裁剪断言弱；4001 用域内原始码不扩 ot-core ResultCode 枚举（javadoc 已登记理由），留批次收尾 CONV 横切核对（枚举单一来源）确认。

## 已核实无发现的重点（brief「依据」② 逐项）

- **DFS 环检测完备性**：自环（`next==start` 即返、环列表 `[start]`）、两节点/三节点环、跨实例 PARENT 闭环与绝对键跨层环同图覆盖（租户级图 + full_key 兜底解析）、`done` 剪枝对「本项可达自身」语义正确（下游环不误报有测试锚定）、环列表从候选项起按引用方向且起点不重复——算法实推正确，8 例纯函数测试+服务层 4 例闭环。
- **同一项 ref 重建原子性**：`deleteByItemIdPhysical` 物理删理由成立（DDL 实查 uk_item_ref 不含 deleted 列 + BaseEntity @TableLogic 实查），并发同项保存经行锁串行化、末写者胜、唯一键冲突 4002 可重试；InOrder 先删后插断言在位。
- **事务与事件**：校验全链先于任何写、写与 ref 重建同 `@Transactional(rollbackFor=Exception.class)`、事件在写成功后发布（照 0013 范式只发布不建监听）——拒存零写入有 never() 断言。
- **SQL 参数化**：`deleteByItemIdPhysical` 用 `#{}` 占位且显式 tenant_id（照 selectOccupancyByInstance 范式实查在位），其余全走 MP wrapper——无拼接注入面。
- **租户与数据权限**：点位同租户经插件注入（他租户同文案不泄露）、写路径 requireDataPerm（无 createBy 分支，0012 修订轮口径）、读路径 isInstanceReadableByUser（含 OR 分支）、impact 对端按实例可见性裁边、403 文案无实例标识泄露（测试断言）。
- **0057 前端契约**：前端占位仅 `emit('impact-preview')`（`iot-formula-editor.vue` 实查），响应形状由本任务冻结——IotItemImpactVO 含 full_key/实例编码/名称 + outgoing/incoming/metrics 三段 + 指标空集占位，brief 决策 6 全要素在位，接线零结构变更。
- **继承链扩展**：四要素三态经 applyEffective 重载纳入，模板坏周期码读路径容错（mapCalcPeriodQuiet）与生成链 fail-closed 语义区分正确。
- **汇总与阈值**：SUM/AVG 白名单 + IotRollupRules 声明位（质量码 7/PARTIAL_WITH_INCOMPLETE_MARK）；thresholds 为 0014 overrideParams 完全委托（partial/null=清除语义一致、无 @Valid 与 0014 params 端点形态一致——表单刻意不挂 bean validation，实查在位）。
