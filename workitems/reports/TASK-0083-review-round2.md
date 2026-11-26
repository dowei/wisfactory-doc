# 评审报告：TASK-0083 scoped 复评 round2（代码评审 · 终审）

> 评审者：fresh 代码评审子代理（不共享协调者上下文；只依据 brief「必读」与工作区源码判断，判断不了的标「无法判定」）。
> 口径：`.agents/protocol/L1-execute.md` §13「diff 评审」复评条款——只看修复 diff（4d4012e..7628007 限 backend/，2 文件）与原发现 F-1/F-2/F-3，不重开原评审已通过部分。
> 契约：no-subagents；只评审不修改（本报告为唯一产出）；未向协调者追问。
> 独立实测（本轮亲自执行，非引用执行者报告）：
> - `git diff --name-only 4d4012e..7628007 -- backend/` = 2 文件（IotPublishServiceImpl + IotPublishAutoFollowTest），与 diff 包逐字一致；
> - `git diff --name-only 7628007^..7628007` = 同 2 文件（单提交）；`73b8587`（0041 calc 修订）确在区间内，但 12 文件全在 `calc/**`，已被 backend/ 范围过滤隔离；
> - `mvn -pl ot-admin -am test -Dtest=IotPublishAutoFollowTest -Dsurefire.failIfNoSpecifiedTests=false` → **8/8 通过，BUILD SUCCESS**；日志含 `[PUBLISH-FOLLOW] ... applied 2 -> 3 (userId=10)` 与 `skipped (T3 never auto)`；
> - `mvn test -pl ot-admin` → **Tests run: 2403, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**。

---

## 〇、越界反查复核（评审者独立复核 brief「越界反查」）

- 区间全量 `git diff --name-only 4d4012e..7628007` = 2 个 backend 文件 + 14 个 calc 文件；后者全部来自并行提交 `73b8587`（TASK-0041 修订轮，见其 `--stat`），**已按 backend/ 范围隔离剔除**，未混入本任务足迹。
- 修复 diff 本体：`IotPublishServiceImpl.java`（主代码）+ `IotPublishAutoFollowTest.java`（测试），全落原 brief 授权范围；0081 内核 `IotUpgradeApplyServiceImpl` 零改动；SQL/yml/ot-core/前端/ingest/calc 零触碰（diff 无相关路径）。
- 结论：**越界反查通过**（与 brief 声明一致）。

---

## 一、三问逐条结论

### 问 1：F-1 是否真修复？——**真修复（通过）**

- **结论**：follow 已从 round1 的 `recordDispatch` 之后（assemble→SHA→stage→INSERT→promote→回写→dispatch→**follow**）前移到**确认门之后、assemble 之前**，当期发布快照即含跟随后新值；原先「applied 已推进但快照旧值、无第二次发布即永不生效」的静默缺口已消除。
- **出处**：`IotPublishServiceImpl.java:210`（`confirmGate.requireConfirm`）→ **:222 `followUnlockedInstances(tenantId, scopeType)`** → **:225 `assembler.assemble(...)`** → :226 serialize → :228 SHA → :236 stage → :251 INSERT → :260 promote → :269 writeBack → :271 recordDispatch → :273 return。原调用点（`:268` 之后）已删除，注释/javadoc（:216-221、:278-280）同步改写并明示根因。
- **事实链核实**：
  1. `ConfigSnapshotAssembler.assembleComputeScope`（`ConfigSnapshotAssembler.java:191-192`）经 `dataItemMapper.selectList` 读 `iot_data_item` 行组装；真实 0081 `applyToVersion` 通过 `dataItemMapper.update`（`IotUpgradeApplyServiceImpl.java:383`）改写同一批行。二者在**同一 `@Transactional`** 内，follow 先写、assemble 后读，读己所写 → 当期快照反映新值。
  2. calc 运行时**只读落位快照、不读实例行**：`HttpSnapshotFetcher.endpoint()`（:152-160）不带 version 参数，拉最新已发布版；`SnapshotReconciler:176-177` 从 `fetched.bodyJson()` 建依赖图；`DualTriggerExecutor.computeItem:167,179` 从图节点取 `item.getFormula()` 执行。故当期快照内容即为引擎实际执行公式——前移后与新值一致。
  3. **原静默缺口闭合**：新顺序下携带 T2 二次确认的当期发布即把新公式写入快照并下发，不再存在「applied 已推进/0016 显示已跟上/确认窗口关闭，而引擎仍跑旧公式」的窗口；无需第二次发布。
  4. **失败语义/零半态不变**：follow 仍不 catch、不另开事务，异常沿 publish 的 `@Transactional(rollbackFor=Exception.class)`（:197）整批回滚。且因 stage（:236）现位于 follow 之后，follow 失败时连暂存文件都未生成——比 round1「follow 后置、promote 后再失败留孤儿落位文件」**更干净**，无新半态。
  5. **网关早返回 / T3 skip / 乐观锁 4002 保持**：`:299-301` 非计算 scope 早返回；`:351-358` T3 逐台跳过；`:365-368` affected=0 → 4002。红线 2/3/4/5/6 用例本轮实测仍绿。
- **测试左证**：新增 `currentSnapshotContainsPostFollowValue`（测试文件 :284-317）用 `doAnswer` 在 applyToVersion 调用时把数据项 formula 由 `/100` 改写 `/1000`，随后 `storage.readBytes(row.getSnapshotUri())` 断言当期快照含 `/1000` 且不含 `/100"`——**对旧时序必红、新时序必绿**；本轮独立实跑通过。dev 实证 v13 当期快照 `1/2-0-13.json` 含 E8F `/1000` + 锁定 E8G `/100`、仅一次发布（无针对该模板的二次发布）为执行者报告实测（见「无法判定项」）。
- **一句话说明**：follow 前移至 assemble 之前使当期快照即含跟随新值，calc 只读快照的引擎当期即按新公式执行，静默且持久的「发布生效」缺口消除，失败回滚/零半态/网关/T3/4002 全部保持。

### 问 2：F-2 是否真修复？——**真修复（通过）**

- **结论**：follow 实例筛选已在 **SQL 层**加 `status=1`，停用实例不再被自动跟随，口径与确认门/影响预览一致。
- **出处**：`IotPublishServiceImpl.java:315-320` 的 `LambdaQueryWrapper<IotModelInstance>` 链新增 **:318 `.eq(IotModelInstance::getStatus, 1)`**（javadoc :282-283、注释 :311-314 同步标注）。
- **筛选层次与边界核实**：
  - 层次：过滤在 **SQL 层**（wrapper `.eq(status,1)`）完成；内存二次防线（:337-340）仍只兜 `locked`/`applied`，**未**二次校验 status。因 status 由 SQL 精确过滤且无桩漂移场景，此分层可接受（与 locked 的「SQL+内存双防」略不对称，但不构成缺陷）。
  - 边界：`status=null` → SQL `status = 1` 不匹配，保守不跟随；`status=其他值（如停用=2）` → 同样排除。方向与「仅启用中实例随发布生效」一致，合理。
  - 口径比对：`hasPendingTier2Confirm`（`IotTemplateVersionServiceImpl.java:326-330`）含 `.eq(status,1)` ✓；`previewImpact`（:224-226）含 `.eq(status,1)` ✓。二者与 follow 四处同为 status=1，一致。
- **一处措辞不精确（Minor 观察，非缺陷）**：follow 注释称 status=1 与「0016 待升级清单（`IotInstanceUpgradeServiceImpl.selectUpgradeList`）一致」，但该清单实际查询（`IotInstanceUpgradeServiceImpl.java:148-150`）**并无 status 过滤**（它是含锁定实例的人工逐台升级清单，本就不应限启用中）。行为结论（排除停用实例自动跟随）正确且应与确认门/预览对齐；仅注释援引对象略宽，不影响功能。
- **测试左证**：新增 `followFilterIncludesStatusEnabled`（测试文件 :321-335）用 ArgumentCaptor 截 `instanceMapper.selectList` 的 wrapper，断言 SQL 段含 status/参数含 1；对旧代码（无 status）必红，本轮独立实跑通过。
- **一句话说明**：status=1 已加入 follow 的 SQL 筛选，null/停用值被合理排除，与确认门/影响预览同口径，停用实例不再被误跟随。

### 问 3：修复 diff 是否引入新 Critical/Important？——**无新 C/I（通过）**

逐项核brief 点名的四处时序协调：

1. **与 0024 两段式落位（stage/promote）**：follow 现位于 stage（:236）/promote（:260）**之前**。异常路径更干净（follow 失败时无暂存/落位文件产生）；正常路径正式名仍只被 INSERT 胜者写一次（:250-267 撞键补偿只删自身暂存文件），哈希/文件一致性不变。**无新半态**。
2. **快照哈希计算**：hash = `sha256Hex(serialize(assemble(...)))`（:225-228），stage 写入同一 json、promote 原子落同一字节；取回侧 `readAndVerifySnapshot`（:702-727）重算文件 sha256 与行哈希比对恒一致。**无哈希失配**。
3. **publish_log 派生**：`recordDispatch(row)`（:271 → `IotPublishLogServiceImpl.recordDispatch:93-101`）按版本行 scopeType/scopeId 派生目标行，与 follow 无关；调用点数量/位置未变。**无派生异常**。
4. **NotifySender afterCommit**：`MqttNotifySender.notify`（:82-107）载荷在调用点（事务内）捕获，注册 `afterCommit` 后发送。follow 前移后，版本行哈希已反映跟随新值，通知哈希与当期快照一致；下游拉取不带 version 取最新快照，重算 SHA 与通知哈希吻合。每目标行仍只触发一次通知。**无重复通知、无哈希失配**。
5. **确认门时序（关键）**：follow 置于**确认门之后**（:210 先于 :222）——T2 发布时门先看到滞后未锁定实例、要求 token；若误置于门前，`hasPendingTier2Confirm` 会因滞后清零而放行、绕过二次确认。diff 顺序正确，**未引入确认门失效**。
6. **回归计数**：全量 `mvn test -pl ot-admin` 本轮实测 **2403 = 2401 基线 + 2（F-1 当期快照断言 / F-2 status 过滤断言），0 失败 0 错误 0 跳过**；目标类 8/8（原 6 + 修订 2）。**零回归**。
7. **F-3**：仅登记 deferred，未修改。复核 `templateMapper.selectList(new LambdaQueryWrapper<>())`（:302）仍无显式 tenant 条件；全仓唯一 `publishService.publish` 调用方为登录端点 `IotPublishController.java:93`（`@SaCheckPermission`），无无会话调用方。留痕正确，维持 deferred，非本轮新发现。

**新发现分级**：Critical 无；Important 无；Minor 1 条观察（问 2 所述注释援引 0016 不精确，非缺陷，不阻断）。

---

## 二、发现清单（本轮）

| 编号 | 严重级 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| （无新 C/I） | — | — | 三问涉及的修复点与时序协调均核实通过。 |
| O-1 | Minor（观察，非缺陷） | `IotPublishServiceImpl.java:312-314` 注释 vs `IotInstanceUpgradeServiceImpl.java:148-150` | 注释称 status=1 与 0016 清单一致，但该清单实际无 status 过滤；行为正确、仅措辞援引略宽，建议后续更正注释，不阻断本轮。 |

- **Critical**：无。
- **Important**：无（原 F-1 已闭合）。
- **Minor**：O-1（观察级，非缺陷）；F-3 维持 deferred 留痕（原 Minor，本轮未改、非新发现）。

---

## 三、结论

**通过**（TASK-0083 scoped 复评 round2 终审）。三问全过：

1. **F-1 真修复**——follow 前移至确认门后、assemble 前，当期发布快照即含跟随新值，calc 只读快照的引擎当期执行新公式，静默/持久的「发布生效」缺口消除，失败回滚/零半态/网关早返回/T3 skip/乐观锁 4002 全部保持；
2. **F-2 真修复**——follow 的 SQL 筛选加 `status=1`，null/停用值合理排除，与确认门/影响预览同口径；
3. **无新 Critical/Important**——0024 两段式落位、快照哈希、publish_log 派生、NotifySender afterCommit 四处时序协调核实无新半态/哈希失配/重复通知，确认门时序正确；2403 测试零回归（2401 基线 + 2）；F-3 仅登记 deferred 留痕。

原评审唯一阻塞项 F-1（Important）已闭合，F-2（Minor）顺手修已核实，F-3（Minor）deferred 保持。无新增 C/I。**给出「通过」结论**，无退回。

---

## 四、无法判定项

- **dev/prod live 实证**：v13 当期快照 `1/2-0-13.json` 含 `/1000`（无 v14）、prod 启动 0 ERROR + SM4 登录链等为执行者报告实测。受 fresh / 只评审不修改契约约束，本轮**未独立重跑 live 服务或直连 DB 取快照文件**；该部分以报告实测为准，标「无法判定（未独立 live 复验）」。但 F-1/F-2 的核心语义已由本轮独立实跑的单元级快照文件内容断言（8/8）与源码时序链核实覆盖。
- **InnoDB 事务隔离下的读己所写**：assemble 与 applyToVersion 的同事务可见性经源码（同 `@Transactional`、MP 同连接）与 dev 实证间接确认；本轮未在真实 MySQL REPEATABLE READ 场景下独立复验该隔离细节，极端隔离配置下的表现标「无法判定（未独立 live 复验）」。依据 dev 实证观察到的是新值 `/1000`，当前配置和实现下读己所写成立。
