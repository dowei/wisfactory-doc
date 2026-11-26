# 评审报告：TASK-0014 diff 评审

- 评审者：fresh reviewer（brief：reports/TASK-0014-review-brief.md）
- 评审对象：diff 包 `TASK-0014-diff.txt`（提交 7fa7cb9，13 文件 1562 行，已完整读入），并实测核对仓库 HEAD（7fa7cb9）与 diff 逐点一致（`git log`/`git diff --name-only 4d7246d..HEAD` = 13 文件，全部落在 brief 声明范围 dataitem 域 main 9 个 + iot 测试 4 个，无越界）。
- 独立复跑四个新增/调整测试类（surefire 实测）：`IotDataItemServiceImplParamOverrideTest` **21/21**、`NoOpIotDataItemHistoryServiceTest` **2/2**、`IotDataItemControllerTest` **15/15**、`IotDataItemMenuPermissionGuardTest` **3/3**——**41/41 全绿**，与报告 §2.1 一致。

## 一、与 brief 契约一致性（逐条核过，全部一致）

1. **覆盖仅写实例行四列**：`overrideParams` 用 LambdaUpdateWrapper 显式 set 四列（含 null 落 SET=清除），实体传空对象仅走 updateFill 补审计元数据——updateById 空值忽略策略做不到的 partial 语义用 wrapper 正确表达；测试锁「sqlSet 恰 4 列、参数值逐一核对、where=本行主键」（兄弟实例行不可达）。与 brief 决策 1 逐字吻合（含状态规则门限/规则行不复制口径、批量覆盖不做）。
2. **运行时继承三态**：`getEffectiveParams` 逐字段解析实例值→INSTANCE / 模板 default_ 同名值→TEMPLATE / 双空→NONE（null 透传不判定），无任何复制落库写路径；模板已删项/组织节点（templateId=null）经 selectOne 空/tableId null 分支正确落到逐字段 NONE（两个专门用例）。与 brief 决策 2 一致。
3. **min<=max 校验**：valid 对与 alarm 对各自成立、负值/小数位>6/整数位>14 全拒（PARAM_ERROR fail-closed）、四字段全 null=清除合法——与 brief 决策 3 字面一致；decimal(20,6) 边界（`9999999999.999999` 合法、`100000000000000` 拒）有正反用例。**配对校验只覆盖同次提交的双值**——见发现 I-1。
4. **时间线钩子占位**：`IotDataItemHistoryService` 接口（change_type=PARAM 语义、同事务契约、0039 接线点与零改动承诺 javadoc 齐备）+ `NoOpIotDataItemHistoryService`（@Service、debug 留痕、无持久化副作用）+ 调用方同事务接线，不建表不写历史行。与 brief 决策 4 一致。
5. **数据权限**：写/读两接口均存在性→resolve 校验（allTenant 直过、可见集成员 403）→归属，403 文案不含实例编码/id（测试双断言 + 冒烟 S12/S12b 双形态取证）——照 0012 修订轮 I-1 同款范式。与 brief 决策 5 一致。
6. **权限复用**：覆盖=iot:item:edit（4039）、继承链=iot:item:query（4037），不新增菜单种子；守卫测试把两接口锚点锁进五值集合、子集下限 2→4（remove 仍预留）。与 brief 决策 1/验收 4 一致。
7. **TDD 六项行为面**（brief 决策 6）：兄弟隔离、清除回归、三态、min>max 拒、钩子载荷 before/after 断言（含校验失败零钩子）、数据权限 403——全部有用例且断言真实（wrapper 断言锁的是最终渲染 SQL 段文本而非 MP 内部布局；403 文案泄露双查；钩子载荷逐字段捕获）。未发现断言放松或假绿灯。

## 二、发现清单

### Critical（0）

无。数据权限两接口均已接线、403 不泄露标识、校验 fail-closed、无注入面、无范围越界。

### Important（1）

1. **partial 覆盖可分两步绕过 min<=max 配对校验，行内非法对可落库**。出处：`IotDataItemServiceImpl.java:500`（`validateThresholds` 仅当四字段中同对**双方均在本次提交**时才做 compareTo 校验）。说明：单次 PUT `{validMin:5000, validMax:100}` 被 400 拒，但先 PUT `{validMin:5000}` 再 PUT `{validMax:100}` 两次均合法——行内留下 validMin=5000 > validMax=100 的非法对（effective-params 会如实报 INSTANCE/INSTANCE），与 TASK 验收标准「非法配置拒绝」存在张力；javadoc 已自陈「单侧清除后与模板默认的配对关系归计算引擎运行期判定」，但**同属实例行的两列配对在保存期完全可知**，与模板侧无关（模板默认值变更确实只能运行期判定，此部分口径成立）。建议：配对校验扩为「结果行四列」——缺省侧取当前行值（form 为 null 时该列结果=清除，与模板侧比较仍归运行期）；或由协调者显式裁决「保存期只管提交对、行内对归运行期」并把该口径回写 TASK/REQ-0005。修复点小、不触接口签名。

### Minor（6，deferred 记录，不进修订轮）

1. **decimal(20,6) 上界示例写错（注释级，校验逻辑本身正确）**。出处：`IotDataItemServiceImpl.java:129`、`:497`、测试 `:444` DisplayName。说明：三处写「上限 9999999999.999999」——该值仅 10 位整数，真上界是 14 位整数 + 6 位小数 = 99999999999999.999999；`precision-scale>14` 的判定本身正确，仅示例误导后人。
2. **并发覆盖无版本守卫，last-write-wins；update 返回值被忽略**。出处：`IotDataItemServiceImpl.java:400-433`。说明：iot 域全模块无 @Version（grep 证实，属项目既有惯例）——两管理员并发覆盖后者胜；行在 select 与 update 之间被删则 0 行更新仍返回 200 且钩子照记；钩子 before 快照取自前置读，竞态下可能过期。配置类数据可接受，登记即可（若 0039 时间线要求精确 before，需届时一并裁决）。
3. **requireDataPerm 与 403 文案跨域字面复制**。出处：`IotModelInstanceServiceImpl.java:645` vs `IotDataItemServiceImpl.java:535`。说明：声明「0012 修订轮 I-1 同源同口径」实为复制私有方法+文案字面量——后续防线口径演进（如 I-1 或 deferred M-4 的顺序收敛）需两处同步，存在漂移风险；可考虑提取 dataperm 域公共助手（另行微改，不入本任务）。
4. **钩子载荷缺 instanceId**。出处：`IotDataItemParamChange.java:15-23`（仅 itemId/fullKey/operatorId/before/after）。说明：0039 若需按实例过滤时间线（数据项删除后仅剩 fullKey——含实例编码而非 id），需扩载荷 record，与「接口签名即合同/零改动接线」承诺有张力；建议 0039 派发 brief 时显式确认载荷形态是否终版。
5. **存在性 400 vs 不可见 403 的 id 存在性探测差异**（实施报告遗留 4 已登记，与 0013 deferred M-4 同口径，维持 deferred）。出处：`IotDataItemServiceImpl.java:402-407` / `:442-447`。说明：不存在实例 400、存在但不可见 403，受限用户可探测 id 存在性；文案均不泄露实例标识。如需收敛（先权限后存在性）改动点在首两段顺序，协调者裁决。
6. **locked=1 实例的参数覆盖放行**（实施报告遗留 2 已登记待裁决）。出处：`overrideParams` 无 locked 守卫。说明：REQ-0005 锁定口径约束「实例编辑仅允许 status/remark」，未明文覆盖数据项阈值覆盖；当前锁住实例行字段但数据项四列仍可覆盖。若产品要求锁定连带冻结，改动点为增一道 locked 守卫——待协调者裁决。

## 三、结论

- 契约一致性：brief 六项决策与验收四项（全量回归 1480=1449+31、dev 冒烟全步、prod 真实启动 0 ERROR、范围核对 13 文件）与报告输出、仓库实况三方一致；独立复跑 41/41 绿。
- 判定：**Critical 0 / Important 1 / Minor 6**。Important 1（partial 两步绕过配对校验）按 L1-execute §13 进退回/裁决路径——修复点小（validateThresholds 扩为结果行四列配对）或由协调者显式裁决口径后回写；Minor 6 全部 deferred 进 TASK 任务结果（其中 2 条实施报告已自行登记，评审复核确认）。
