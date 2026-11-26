# TASK-0021 diff 评审报告（主体 9a162a8 + 修订轮 a61fd24）

- 评审者：fresh reviewer（不共享协调者上下文；只评审不修改）
- 输入：TASK-0021-diff.txt（BASE 46d9391 → a61fd24，4891 行全文读入）、TASK-0021-brief.md、tasks/TASK-0021.md、TASK-0021-report.md（含 §7 修订轮）、ADR-0011、ADR-0016、L1-execute.md §13
- 补充核证（评审者自行在代码仓/文档库做的只读验证，用于支撑结论）：
  - 附录 A p31 逐字复核：自提 `docs/factory-data-platform-detail-design-v1.1.pdf` 第 31 页（page index 30）与 `20261122_iot_point_domain.sql` 的 iot_point DDL 逐列/逐索引比对——**一致**（列序/列型/注释/PK/uk/三索引全同；两处声明级偏离：`CREATE TABLE IF NOT EXISTS` 幂等化、未显式 COLLATE 随库继承——均已在脚本头注与报告登记）。
  - §3.1 点位段接口行复核：详细设计 p21-22 实提 **8 行**（list/detail/add/edit/remove/import/export/changeStatus）——本批交付 6 行与 §3.1 逐行对齐（权限列含 changeStatus=iot:point:edit 一致），import/export 归 TASK-0022、详情反查归 TASK-0023；「九接口」为口径表述，无契约缺口。
  - 配额链路：`SysTenantServiceImpl.checkPointQuota`（quota 空/0 不限、`current >= quota` 拒 4003）+ `QuotaPointCounterImpl`（活跃行 COUNT、tenant_id 显式 eq、@TableLogic 过滤）+ createPoint 在 insert 前调用——接线正确；TOCTOU 残余按 0003 M-7 已登记。
  - F5 裁剪：`DataPermConditionAssembler.pointCondition(scope, userId, deptCol, instCol, createByCol)` 实读确认（双分支单分支裸片段/多分支括号 OR/零分支 1=0 fail-closed；createByColumn 传 null 即无该分支）；点位 list 用 `dept_id`+`bound_instance_id`、网关 list 用 `dept_id`+`instance_id`，NULL 不特判；详情 403 文案不泄露标识；**网关控制器无 detail 端点**（仅 list/add/edit/delete），故 F5「详情 403」仅点位适用，无网关详情缺口。
  - 修订轮 guard 调用面：grep 全仓确认 `ChannelReferenceGuard.countPoints/pointCodes` 调用点仅 `IotChannelServiceImpl.deleteChannel` 一处（pointCodes 仅拒绝分支）；Placeholder 已删、唯一实现为 IotPointChannelReferenceGuard。
  - 参数化：IotPointCodeMapper `#{}`+foreach 参数化；wrapper 条件参数化；Assembler 字面 IN 值均为 Long（无注入面）；LIKE 走 SqlLikeUtils.escape。
  - 多租户：sys_dept 在 OtTenantLineHandler 显式表清单内，validateDept 的同租户保证成立。
  - ResultCode：4002/4003/403/400 与用法一致。

## 契约一致性核对（通过项）

- DDL p31 逐字 + 幂等 + 头注 + 菜单 4053-4057（parent 4011）+ 超管授权：通过（含守卫测试 IotPointMenuPermissionGuardTest 双向锁定）。
- 六端点与 §3.1 点位段对齐、批量启停独立端点（§3.1 有独立行）、权限码一一对应：通过。
- unit 强制 iot_unit 字典校验（类型缺失/条目不存在/停用均拒 + 归口运营方提示）创建/编辑同口径：通过。
- 双唯一并发兜底（含已删行旁路预检 + DuplicateKey→4002）、point_code 不可改、编码归档不复用：通过。
- 配额接线即生效（计数器真实实现 + add 前调用 + 4003）、过渡期单测按 brief 移除：通过（残余见 M-5）。
- PointEnabledEvent 只发布不建监听、payload 五元、0013 范式：通过（边界见 M-2/M-3）。
- F5 点位/网关列表同批接线（条件形态、NULL fail-closed、可见集空 1=0、create_by 不适用口径留痕）+ 详情 403 + 红线套件增补（Point 6 / Gateway 5 用例，模拟器漏拼条件即红）：通过。
- 修订轮（0018 遗留通道侧 guard 转正）经协调者授权，7 文件在授权范围、调用面完整、红转绿留痕（编译红 + 旧 jar 行为红 + 复跑 1814 绿）：通过。
- 改动范围：本任务 30 文件（23+7）全落声明范围 + redline 授权例外；diff 包另混入 TASK-0057 前端 6 文件（见 M-6，非实现越界）。

## 发现清单

### Critical（0）

无。

### Important（2）

- **I-1**｜`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/impl/IotPointServiceImpl.java`（createPoint 实体装配，行 247-261）｜新增点位 remark 字段被静默丢弃：`IotPointCreateForm` 声明并接收 remark（@Size 500、DDL varchar(500)），updatePoint 有 `upd.setRemark`（行 334）而 createPoint 白名单遗漏 setRemark——建档备注无落库路径，且无用例覆盖该字段（表单字段与落库白名单应一一对应）。
- **I-2**｜`IotPointServiceImpl` 写路径三方法（updatePoint / changePointStatus / deletePoints）｜点位写路径无数据权限防线：list/detail 已裁剪，但持有 iot:point:edit/remove 的受限用户可按可枚举自增 id 直改/启停/删除本租户内对其不可见的点位（横向越权写）；brief F5 段确未声明写防线（契约缺口，报告 §4.3 已登记遗留），但按 0012 修订轮 I-1 同型先例（实例写路径已补防线）应补线或立 BUG 挂 FEAT-0007 跟进，从严定 Important。

### Minor（8）

- **M-1**｜`IotPointServiceImpl.resolveUnit` + createPoint/updatePoint 装配｜unit 空白串落库：resolveUnit 把空白视为「未携带」（跳过字典校验），但实体装配仍把 `"  "` 写入库（非 NULL 亦非拒绝）——建议空白归一为 null（或显式拒绝），避免脏数据与「清空单位」语义含混。
- **M-2**｜`IotPointServiceImpl.changePointStatus` + `PointEnabledEvent` javadoc｜对已启用点位再次置 1 会重复发布事件（冒烟 11 条留痕含重复 pointId=9010）；「重复事件由消费方幂等承接」只写在服务行内注释，未进事件类 javadoc 契约——0029 消费方按事件类文档对接时看不到该口径，建议补进事件契约。
- **M-3**｜`createPoint`/`changePointStatus` 事件发布时机｜同事务发布（与 0013 `InstanceItemsGeneratedEvent` 实读范式一致，事件 javadoc 已声明「提交后语义由 0029 用 @TransactionalEventListener 决定」）：事务提交失败的残余风险已声明并委托消费方，判定为可接受；登记为 0029 接线时的确认项。
- **M-4**｜`IotChannelGatewayReferenceGuard.countPoints`（gateway 侧 guard）｜占位 TODO 未转正（修订轮授权仅通道侧，brief 漏带）：当前删网关先卡通道计数（真实）→删通道先卡点位计数（真实）传递性受保护，属纵深防御缺口而非直接越权；代码 TODO 与报告 §4.2/§7.4 均登记，建议随 TASK-0022/0024 裁决「点位→通道→网关」间接计数实现。
- **M-5**｜`SysTenantServiceImpl.checkPointQuota`（行 406）｜过渡期 fail-open 兜底分支保留：`pointQuotaCounter != null ? ... : 0` 在真实 bean 在位后不可达，但若未来装配失效配额强控将静默放行；brief「兜底语义随真实 bean 移除」因校验器不在本批改动范围未能字面执行（报告 §3.5 已登记），建议后续批次授权清理（改必需注入或显式拒绝）。
- **M-6**｜`TASK-0021-diff.txt`（合并包）｜diff 包混入 TASK-0057 六个前端文件（formula-rules.ts / iot-formula-editor.vue / iot-formula-ref-picker.vue / template-item-modal.vue / 两个 spec）——BASE 漂移（另一会话 587d5e6）混入 `46d9391..HEAD` 合并包；评审 brief「越界反查：全部落在 brief 声明范围」的表述与 diff 包实际内容不符。实施报告 §3.2 已识别并按 `587d5e6..HEAD` 正确完成范围核对（实现无越界），属打包/表述瑕疵，非代码缺陷；TASK-0057 文件不在本评审范围内评审。
- **M-7**｜`TASK-0021-report.md` §1.2｜表头「测试（10）」实列 9 行（合计 23 文件与 §2.5 路径清单一致）——报告笔误，不影响范围核对结论。
- **M-8**｜`resources/sql/upgrade/20261122_iot_point_domain.sql` 头注｜「三处二级索引」括号内列出四个键名（uk + 三个 idx_*）——注释表述歧义，纯笔误。

## 结论

契约一致性总体通过（DDL 逐字经评审者独立复核 PDF 证实；六端点/配额接线/F5 裁剪/事件范式/守卫测试/双道闸与 brief 及 ADR-0016 口径一致；修订轮授权与调用面完整）。按 §13：**Important 2 条（I-1 建档 remark 丢失、I-2 点位写路径越权防线缺口）建议进退回/立 BUG 路径**——I-2 为 brief 层缺口且已登记遗留，走「实现缺陷建 BUG 挂 FEAT-0007」或授权修订轮补线由协调者裁决；I-1 为实现缺陷，建议随同一修订轮补 setRemark + 补用例。Minor 8 条记 TASK 任务结果 deferred（收尾进 LOG），不阻塞收尾。
