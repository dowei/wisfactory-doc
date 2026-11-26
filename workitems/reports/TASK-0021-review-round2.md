# TASK-0021 修订轮 scoped 复评报告（round 2）

- 评审者：fresh reviewer（不共享协调者上下文；只评审不修改代码）
- 输入：TASK-0021-review2-brief.md、TASK-0021-review.md（原发现 I-1/I-2 与 Minor 8 条）、TASK-0021-fixdiff.txt（BASE a61fd24 → a961fd1，348 行全文读入）、TASK-0021-report.md §8（修订轮节）
- 补充核证（评审者自行在代码仓做的只读验证）：
  - commit a961fd1 实际改动 4 文件（IotPointServiceImpl + 3 测试文件，240 行纯新增、零删改）——与报告 §8.1 一致，无越界文件。
  - 主码实读：`IotPointServiceImpl.java` 行 297 `entity.setRemark(form.getRemark())` 在位；行 231-249 `requireWritePerm` 单/批两重载在位；三写原语接线实读确认——updatePoint 行 333（存在性→数据权限→业务校验）、changePointStatus 行 402（防线先于 updateById 循环与事件发布）、deletePoints 行 440（防线先于绑定校验）。
  - 可见口径与列表一致性实读：`requireWritePerm` 复用 `isPointReadableByUser`（行 218-221，与 getPointDetail 同源函数）——allTenant 直过 / dept_id ∈ 部门集 OR bound_instance_id ∈ 可见集 / 双 NULL 行 fail-closed；无 create_by 分支（纯 scope=4 用户零命中=不可见 403，与列表 assembler「createByColumn 传 null 零分支 1=0」语义等价）——与裁决口径逐项吻合。
  - 403 文案「无权操作该点位（数据权限不命中）」不含点位编码/ID/部门实例标识。
  - 表单字段与白名单一一对应复核：IotPointCreateForm 15 字段（pointCode/pointName/dataType/valueType/unit/min/max/samplePeriod/deadband/channelId/protocolAddr/boundInstanceId/deptId/status/remark）全部映射进 createPoint 装配块，无遗漏无多余。
  - 测试实读：服务测试 5 用例（remark 落库 captor 断言 + 编辑/启停/删除 403 与可见回归，含「防线先于绑定校验」的 `verify(dataItemMapper, never()).selectList` 断言、不半批的 `never updateById/deleteByIds` 断言）；PointRedlineTest 场景 6 三用例走真实 `DataPermServiceImpl`（USER_A scope=2 部门 A 厂，POINT_B_FACTORY 602 不可见 / POINT_A_FACTORY 600 可见）——非纯 mock 自证。RedlineFixture 新桩为纯增量（selectByIds 批量定位 + updateById/deleteByIds 返回 1 + itemMapper 访问器），不改变既有只读用例语义。
  - 断言无放松：diff 为 240 行纯新增，既有用例零改动；新增断言均为精确断言（403 码、文案不泄露、写方法不触达、事件不发布）。
  - 事务边界：三方法 @Transactional 内防线均先于任何写操作/事件发布，BusinessException 回滚无半批残余。
  - 验收输出复核：1822 = 1814 + 8（服务 +5、红线 +3）数目吻合；Redline 套件 28（Point 9 含新增写越权 3）；dev 冒烟 S2-S5 写防线三路径 403 + 可见编辑 200 + S6 remark 建档回读一致，均留原始输出。

## 结论

**两项均判定已修复。**

- **I-1（建档 remark 静默丢失）——已修复**：createPoint 白名单补 `setRemark`，表单 15 字段与落库白名单一一对应原则恢复；单测 captor 断言 remark 进 insert 实体 + dev 冒烟 S6 建档携带→回读一致，落库路径打通。
- **I-2（点位写路径无数据权限防线）——已修复**：updatePoint/changePointStatus/deletePoints 三写原语共用 `requireWritePerm`（单目标 + 批量重载），可见口径与列表裁剪同条件同源（allTenant 或 dept_id ∈ 部门集或 bound_instance_id ∈ 可见集，NULL fail-closed，无 create_by 分支保持资产类 fail-closed）；批量任一不可见整批拒绝不半批落库，删除防线先于绑定校验不泄露绑定状态；403 文案不泄露点位标识。受限上下文横向越权写三路径封死，可见点位写操作不误伤（回归用例 + dev 冒烟 S5 双证）；Redline 套件含新增写越权 3 用例（28 总数）。

## 新发现

无。

（核查过的潜在项均不成立：防线 resolve 与列表同源无口径漂移；防线先于写无事务半批风险；diff 纯新增无断言放松；夹具新桩不影响既有用例；remarks/表单字段无新越界。原 Minor 8 条已 defer 不在本轮范围，修复未使其恶化。）

## 一句判定理由

I-1 白名单补齐且有 captor + 冒烟双证、I-2 三写原语以与列表同源同口径的防线封死横向越权写（403 不泄露、批量 fail-closed、可见回归不误伤）且红转绿留痕、全量 1822 绿——两项按裁决口径落地闭环，修复本身零新缺陷。
