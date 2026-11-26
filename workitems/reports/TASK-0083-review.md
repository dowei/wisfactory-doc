# TASK-0083 diff 评审报告（代码评审）

> 评审者：fresh 代码评审子代理（不共享协调者上下文；只依据 brief「必读」与工作区源码判断，判断不了的标「无法判定」）。
> 口径：`.agents/protocol/L1-execute.md` §13「diff 评审」。
> 输入：review-brief、diff 包（9a6cda6..4d4012e，3 文件）、实施 brief、执行者报告、TASK 详情、ADR-0008、0081 报告 + round2 复评、工作区源码（HEAD=4d4012e）。
> 独立实测：
> - `git diff --name-only 9a6cda6..4d4012e` = 3 文件（IotPublishServiceImpl + 2 测试），与执行者报告一致。
> - `mvn -pl ot-admin -Dtest=IotPublishAutoFollowTest` → 6/6 通过；日志含 `[PUBLISH-FOLLOW] skipped (T3 never auto)` 与 `... applied 2 -> 3 (userId=10)`。
> - `mvn -pl ot-admin test` 全量 surefire 汇总：**2401 tests / 0 errors / 0 failures / 0 skipped**（与报告一致）。
> 契约：no-subagents；只评审不修改（本报告为唯一产出）；对「评审重点 1 接线时序」给出明确严重级判定。

---

## 〇、越界反查复核（评审者独立核实）

- `git diff --stat 9a6cda6..4d4012e` = 3 文件 / +521 −0；`--name-only` 全部落 brief 声明范围（publish/service/impl 主代码 1 + `src/test/java/**` 2）。
- 0081 内核 `IotUpgradeApplyServiceImpl.java` **零改动**；SQL / yml / ot-core / 前端 / ingest / calc 零触碰；并行 0041 的 `calc/**`（提交 9a6cda6）已在隔离区间外，未混入本任务足迹。
- 结论：**越界反查通过**。

---

## 一、评审重点逐条核对

### 重点 1：接线时序 —— 判定 **Important**（核心阻塞项，详见 §二 F-1）

**事实链（逐行核实）**：
1. `IotPublishServiceImpl.java:217` `assembler.assemble(...)` 读当前 `iot_data_item` 行（`ConfigSnapshotAssembler:191-201`）组装快照；:218-220 序列化 + SHA-256；:228 stage；:243 版本行 INSERT；:252 `promote` 落正式名。**全部先于** follow。
2. `IotPublishServiceImpl.java:268` 才调用 `followUnlockedInstances`，经 `IotUpgradeApplyServiceImpl.applyToVersion` 改写数据项行（`/100 → /1000`）并推进 applied。
3. 计算引擎**运行时只读快照、不读 DB 行**：`HttpSnapshotFetcher.endpoint()` 不带 version 参数 → 拉最新已发布版；`IotPublishServiceImpl:530-551 retrieveSnapshotInternal → readAndVerifySnapshot(:690) → snapshotStorage.readBytes(row.getSnapshotUri())` 取回的是**落位文件**；`SnapshotReconciler:177-180` 从 `fetched.bodyJson()` 构建依赖图；`DualTriggerExecutor.computeItem:174-178` 从图节点 `item.getFormula()` 取公式执行。
4. 由此：携带 T2 确认与发布的 v10，其快照体仍是跟随前 `/100`；新公式 `/1000` 只出现在 v11 快照。执行者报告 STEP6 与「四.1」已如实登记这一事实。

**语义评估（对 ADR-0008「发布生效」）**：
- ADR-0008 二档政策为「随发布生效」，二档定义明确「改默认公式：下次计算起所有未覆盖实例按新公式走」。本任务 TASK 验收标准为「发布后未锁定实例 applied 与数据面按新版本生效」。
- 实际结果：用户做一次 T2 变更 → 影响预览 → 二次确认 → 发布 v10；v10 下发给计算引擎的快照仍是旧公式，实例「当期按旧公式跑」。这与「随发布生效」构成**真实语义缺口**。
- 该缺口是**静默且可能持久**的：发布后实例 applied 已推进（`:349-352`），0016 清单显示「已跟上」；`hasPendingTier2Confirm`（`IotTemplateVersionServiceImpl:306-334`）要求「T2 行窗口未关闭 **且** 存在滞后未锁定实例」，v10 之后两条件均不成立 → 不会再次触发确认门。若此后无第二次模板变更/publish，实例将**永久**执行旧公式，而簿记与清单均显示正常，无 fail-closed 信号。
- 对「与 0081 同构」抗辩的复核：0081 的显式确认是用户独立动作，其工作流天然包含「确认后再发布」这一交付步骤（0081 报告 STEP6 即为「确认后发布取 snapshotHash」）；而 0083 自动跟随的触发源**就是发布本身**，工作流中没有第二次发布。故 0081 的「下一期快照体现变更」不能平移为 0083 的可接受语义——否则「随发布自动生效」实为「需发布两次才生效」，自动化在数据面交付侧失去意义。
- 根因定位：brief 决策 1 明文「版本落库+快照存储完成后调用 applyToVersion」，执行者严格照办并登记为遗留待裁决。**实现与 brief 字面一致，但与 TASK 验收标准 / ADR-0008 语义不一致**。按 §13「与 TASK 验收标准 / brief 接口的一致性」口径，此为实现结果不满足验收语义；根因在计划层裁决（brief 决策 1），§12 分流时宜按「计划缺陷 / 需另立 ADR 澄清『发布生效』是否等于『当期快照含跟随后行』」处理，但**严重级判定为 Important，不通过**。
- 出处：`IotPublishServiceImpl.java:217`（assemble 早于 follow）、`:268`（follow 接线点）、`:349-352`（applied 推进）；`IotPublishServiceImpl.java:530-551,690`（内部端点读落位快照文件）；`calc/.../HttpSnapshotFetcher.java:150-160`、`calc/.../SnapshotReconciler.java:133,177-180`、`calc/.../DualTriggerExecutor.java:174-178`（引擎消费快照公式）；执行者报告 `TASK-0083-report.md:83,289,302`；ADR-0008 二档政策。
- 一句话说明：快照在跟随前冻结并落位，携带确认/发布的当期版本仍下发旧公式，新值要等下一期发布才进入引擎，构成静默且可能持久的「发布生效」语义缺口。

### 重点 2：筛选口径 —— 基本正确，无 C/I（Minor 观察见 §二 F-2/F-3）

- scope_type=2 才触发：`:291 if (scopeType != SCOPE_COMPUTE) return`，网关早返回；红线 3 断言 `templateMapper.selectList` 零调用。✓
- 同租户：三 Mapper 无 `@InterceptorIgnore`，`OtTenantLineHandler` 对 `iot_` 前缀表在会话有租户时注入 `tenant_id`（`ot-core/.../OtTenantLineHandler.java:61-70`）。发布为登录会话（`:201-202` 已取 TenantId），故同租户成立。✓（无会话场景的隐患见重点 4）
- locked=0：SQL 过滤 `:306` + 内存二次防线 `:325-326`。✓
- applied<current：SQL 过滤 `:307` + 内存防线 `:325`。✓
- T3 永不自动：`:340-345` 每台做 `diff(applied,current)`，含 `TIER_3` 即 `continue` 并留 INFO；红线 4 真断言 `applyToVersion/update` 零调用。✓
- 乐观锁 4002：`:349-355` `UPDATE ... WHERE applied=旧值`，affected=0 抛 `DUPLICATE_CODE(4002)`；红线 6 断言 code。✓
- `applied=0/null` 边界：`lt(applied,current)` 下 null 不匹配（SQL NULL 语义）被静默跳过；但 `IotModelInstanceServiceImpl:185` 创建时 applied=模板当前版（选模板时），null 仅见于无模板实例（本就不滞后），实际影响可忽略。Minor 观察，不阻断。

### 重点 3：事务与失败语义 —— 正确，无 C/I

- follow 加入 publish 现有 `@Transactional(rollbackFor=Exception.class)`（`:197`），不 catch、不另开事务；apply 抛异常 → 整批回滚（版本行 / config_version 回写 / publish_log 行 / 实例推进同原子）。红线 5 断言异常穿透 + `update` 零调用；冒烟 WEEKLY 坏码 → 400、无 v13 行、`calc_period` 仍 60。✓
- **孤立文件评估**：`promote`（`:252`）在事务提交前执行，follow 抛异常时版本行回滚但落位文件保留（`2-0-13.json`）。评审判断**可接受、非半态**：危险方向（已提交行指向缺失文件）由「promote 失败即抛错回滚」挡住；本方向为无行引用的孤儿文件，`SnapshotStorage.promote` 以 `REPLACE_EXISTING` 覆盖历史孤儿（`:105-115` 类注释明确「自愈路径」），下个同版号发布自愈。该权衡源自 0024、非本任务新增，任何 promote 之后的异常（含 publishLog 失败）都会产生，0083 未引入新语义。**不构成发现**（仅作非阻断观察登记，与执行者报告「四.5/五.3」一致）。

### 重点 4：上下文核实（0081 m-1）—— 对当前调用面充分；潜在缺口 Minor

- 静态链：`publish()` 入口 `:201-202` 取 `TenantContext.getTenantId()` + `SecurityUtils.getUserId()`，`assertScopeWritable` 计算 scope 要求 `allTenant()`（`PublishScopeGuard:57-60`）；`applyToVersion → regenerateForInstance → dataPermService.resolve(SecurityUtils.getUserId())` 与之一致。
- 调用面核实（评审者独立 grep）：**全仓仅 `IotPublishController.publish`（:93）调用 `publishService.publish`**，为 `@SaCheckPermission` 登录端点；`PublishSchedulingConfig` 仅注册 ACK 超时扫描，无内部/定时调用 publish。故对**现存**调用面，m-1 会话依赖成立，冒烟 STEP5 `userId=1` 留痕正确即为实证。✓
- 潜在缺口：若未来内部/定时无会话调用 publish，`TenantContext` 为 null → `OtTenantLineHandler.ignoreTable` 短路 → `templateMapper.selectList(new LambdaQueryWrapper<>())`（`:294`，无显式 tenant 条件）将扫描**全租户模板**，且 `SecurityUtils.getUserId()`（`:358`）无上下文。当前无此调用方，属**潜伏风险而非现存缺陷**，记 Minor / deferred，不阻断。

### 重点 5：0081 内核零改动 —— 实证通过

- diff 包仅 3 文件，`IotUpgradeApplyServiceImpl` 未出现。✓ 仅以调用方身份复用 `applyToVersion`。无发现。

### 重点 6：测试有效性 —— 通过，无 C/I

- 6 条红线均真断言（行号以工作区测试文件为准）：红线1 `verify(applyToVersion).applyToVersion(capture,eq(3))` + `update` 一次（:206-208）；红线2/4 `never().applyToVersion/update`（:221-222/:250-251）；红线3 `never().templateMapper.selectList`（:236 区）；红线5 `assertThrows(BusinessException)` + `never().update`（:258-265）；红线6 `assertThrows` + code 4002（:272 区）。✓
- 独立实测 6/6 绿 + 全量 2401/0/0/0，TDD 红 3→绿 6 留痕可采信。✓
- 上下文以静态桩固定 `SecurityUtils/TenantContext`、hash 变化由冒烟承担，分工合理。
- 观察：测试未覆盖「当期快照体内容」（即重点 1 的时序语义），6 用例全部经 mock 驱动、未断言快照内容——这是 F-1 的下游表现，不单列发现。

### 重点 7：对外契约 —— 通过

- publish 控制器四端点签名未改（不在 diff 内）；prod 登录链 + 发布列表/待升级清单 200（报告验收 3），diff 层无契约变化可证。无发现。

---

## 二、发现清单

| 编号 | 严重级 | 出处（文件:行） | 一句话说明 |
| --- | --- | --- | --- |
| **F-1** | **Important** | `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java:217` vs `:268`、`:349-352`；快照取回 `:530-551,690`；`calc/.../HttpSnapshotFetcher.java:150-160`、`calc/.../SnapshotReconciler.java:177-180`、`calc/.../DualTriggerExecutor.java:174-178`；`TASK-0083-report.md:83,289`；ADR-0008 二档 | 快照在 follow 前组装落位，携带 T2 确认的当期发布仍下发旧公式，新值须等下一期发布才进计算引擎——静默且可能持久的「发布生效」语义缺口（详见 §一重点 1）。 |
| F-2 | Minor | `IotTemplateVersionServiceImpl.java:322-330`（确认门含 `eq(status,1)`） vs `IotPublishServiceImpl.java:304-308`（follow 无 status 过滤） | 确认门只统计启用中滞后实例，follow 却会把停用（status≠1）未锁定滞后实例一并跟随：口径不一致，与影响预览 affected 数不符，方向保守但留痕/一致性有偏。 |
| F-3 | Minor | `IotPublishServiceImpl.java:294,358`（`templateMapper.selectList(new LambdaQueryWrapper<>())` 无显式 tenant 条件 + `SecurityUtils.getUserId()`） | 若未来出现无登录会话的内部/定时发布调用，租户插件短路将扫描全租户模板且上下文缺失；当前无此调用方，属潜伏风险，建议 deferred 加防御。 |

- **Critical**：无。
- **Important**：F-1（1 条，接线时序）。
- **Minor**：F-2、F-3（不进修订轮，记 deferred）。

---

## 三、结论

**不通过 —— 退回一轮（§12）**。唯一阻塞项为 **F-1：接线时序（Important）**：实现严格照 brief 决策 1 把 follow 放在快照组装/落位之后，导致携带二次确认与发布的当期版本仍向计算引擎下发旧公式，数据面「随发布生效」在当期不成立，且因 applied 已推进、确认门窗口关闭而静默、在无后续发布时持久。其余评审重点（筛选口径、事务/失败语义与回滚覆盖、0081 内核零改动、测试有效性、对外契约、越界反查）均通过；F-2/F-3 为 Minor/deferred。

**退回说明**：F-1 根因在 brief 决策 1 的时序裁决（执行者已登记遗留、自陈改动需另立裁决），纯「重做」不足以修复；协调者宜按 §12 定性为**计划缺陷**——或补 ADR 澄清「发布生效」不包含当期快照、接受该语义并把红线测试补上「当期快照体」断言；或修订 brief 决策 1 将 follow 前置到 `assemble()` 之前（配合确认门/事务回归验证）后退回执行者重做。两种路径均需先定裁决再决定是否需要修订轮；本次评审不替协调者选择。复评为 scoped：仅看 F-1 修复 diff 与其回归影响，不重开全场。

**无法判定项**：dev/prod 冒烟与 prod 登录链为执行者报告实测，受 fresh/只评审不修改契约约束，本评审未独立重跑 live 服务（仅独立重跑单测与范围/快照消费链源码核实）；该部分以报告实测为准，标「无法判定（未独立 live 复验）」。
