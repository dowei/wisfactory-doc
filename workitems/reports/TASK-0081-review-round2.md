# TASK-0081 scoped 复评 round2 报告（代码评审）

> 评审者：fresh 代码评审子代理（不共享协调者上下文；只依据 brief「必读」与工作区源码判断，判断不了的标「无法判定」）。
> 口径：`.agents/protocol/L1-execute.md` §13「diff 评审」复评条款；**scoped**——只看修复 diff `b491e34..5758d5f`（2 文件）与原发现的 I-1 / m-1 / m-2，不重开原评审已通过部分。
> 输入：round2-brief、diff 包、原评审 `TASK-0081-review.md`、修订轮报告 `TASK-0081-report.md`、实施 brief、工作区源码（HEAD=`5758d5f`，工作区仅余 `?? .agents/`）。
> 独立实测：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → **Tests run: 2387, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**（与执行者报告一致）。
> 契约：no-subagents；只评审不修改（本报告为唯一产出）。

---

## 〇、越界反查复核（评审者独立核实）

- `git diff --stat b491e34..5758d5f` = 2 文件 / +87 −7：`IotUpgradeApplyServiceImpl.java`（主代码）+ `IotUpgradeApplyServiceImplTest.java`（测试），全落原 brief 授权范围 `template/**` 主代码 + `src/test/**`。
- `git log --oneline -5` 确认 `5758d5f` 为单提交、`b491e34` 为其父；组装器 / SQL / yml / ot-core / 前端 / ingest / calc 零触碰；原实现 9 文件足迹未被改写。
- 结论：越界反查通过。

---

## 一、复评三问逐条

### 问 1：I-1 是否真修复 — **通过**

- **结论**：真修复。新默认非法周期码 fail-closed 抛 `BusinessException(PARAM_ERROR)` → `confirmUpgrade` 事务整批回滚；旧默认仍宽容保留历史升级；两侧语义均正确；红→绿负向测试对旧实现必红，有真实区分力。
- **出处**：
  - 主代码：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotUpgradeApplyServiceImpl.java:257-267`（`defaultCalcPeriod` 分支）、`:534-540`（`safeMapPeriod`）、`:542-549`（新增 `requireMapPeriod`）。
  - 换算源：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java:1159-1184`（`mapCalcPeriod`：SECOND/MINUTE/HOUR/DAY→1/60/3600/86400，坏码 `default:` 抛错）、`:1194-1196`（`paramErrorStatic` → `ResultCode.PARAM_ERROR`）。
  - 异常层级：`backend/ot-core/src/main/java/net/otlab/core/exception/BusinessException.java:22`（`extends RuntimeException`）。
  - 测试：`IotUpgradeApplyServiceImplTest.t2NewDefaultCalcPeriodBadCodeFailsClosed`、`t2OldDefaultCalcPeriodBadCodeTolerant`（diff 新增 2 例）。
- **一句话说明**：`patch.setCalcPeriod(newSeconds == null ? 0 : newSeconds)` 被替换为 `requireMapPeriod` 直传 `mapCalcPeriod` 且去掉 0 兜底，非法码在写库前即抛参错，与 0013 生成链 `buildItem`（`IotDataItemServiceImpl.java:1104` 直接调 `mapCalcPeriod`）同源纪律。
- **两侧语义核实**：
  - 新默认 `WEEKLY`：`requireMapPeriod` → `mapCalcPeriod` `default:` → `paramErrorStatic("…无法识别…: WEEKLY")` → 抛出；`applyFieldChanges` / `applyToVersion` / `confirmUpgrade` 逐台循环均无 catch（`:286-321`），异常逃出 `@Transactional(rollbackFor = Exception.class)`（`:236`）代理边界 → 整批回滚，`calc_period` 绝不被写 0。✓
  - 旧默认 `WEEKLY`：仍走 `safeMapPeriod` → catch `RuntimeException` → null；实例现值 3600 时 `3600.equals(null)=false` 且非空 → 判为已覆盖、不写库，历史坏码版本仍可升级到合法新默认，不阻断、不误写。✓
- **负向测试区分力**：`t2NewDefaultCalcPeriodBadCodeFailsClosed` 夹具为旧 `MINUTE`/新 `WEEKLY`/实例 `calc_period=60`（命中「未覆盖→将写新默认」分支）。对**旧实现**：`safeMapPeriod("WEEKLY")=null` → `setCalcPeriod(0)` → 无异常抛出，`assertThrows` 必失败（与修订轮报告记录的红 `Expected RuntimeException to be thrown, but nothing was thrown` 一致）；对**新实现**：抛参错且 `verify(never()).update(...)` 通过 → 绿。区分力成立。✓

### 问 2：m-2 是否真修复 — **通过**

- **结论**：真修复。`regenerateForInstance` 返回值确实等于实际落行的 data item 行数；`rowUpdated` / 动作文案由该值决定，返回 0 时如实报「未补插」。
- **出处**：
  - 调用侧：`IotUpgradeApplyServiceImpl.java:152-163`（`int inserted = …regenerateForInstance(...)`；`inserted > 0` 才置 `rowUpdated=true` 并写实际行数，else 写「占用跳过未补插」）。
  - 返回源：`IotDataItemServiceImpl.java:360-370`（`regenerateForInstance` 权限校验后 `return generateForInstance(instanceId)`）、`:193-227`（实例/模板空与 `newItems.isEmpty()` 均 `return 0`）、`:306-320`（`toInsert` 逐项 `this.save(item)`）、`:355-357`（`return toInsert.size()`，日志 `generated=generatedIds.size()`）。
  - 测试：`IotUpgradeApplyServiceImplTest.t1InsertOccupiedSkippedNoFalseSuccess`（桩 `thenReturn(0)`，断言 `rowUpdated=false` + 文案含「占用跳过/未补插」）。
- **一句话说明**：返回值取的是 `toInsert.size()`——即通过占用预检并逐条 `save` 成功的 data item 行数，不是模板项数、不是引用边数、也不是其他标志量，因此 `inserted > 0` 与实际行变更一一对应。
- **语义核实**：占用跳过路径（含逻辑删行永久占键）在 `generateForInstance:215-227` 被过滤为 `newItems.isEmpty()` → 返回 0，正是 m-2 要拦截的失真场景；在 T1 补插上下文 `addedCodes` 非空，模板项与模板 id 均非空，故返回 0 的实际成因只能是占用跳过，文案归因为「键已占用含逻辑删行」准确。✓ 测试对旧实现必红（旧实现写死 `rowUpdated=true` 且文案「新增项补插」，`assertFalse(rowUpdated)` 失败）。✓

### 问 3：修复 diff 是否引入新 Critical / Important — **通过（无新 C/I）**

- **结论**：未引入新 Critical / Important。
- **出处与逐项核实**：
  1. **异常类型与事务传播**：`BusinessException extends RuntimeException`（ot-core `:22`），`rollbackFor = Exception.class`（`IotInstanceUpgradeServiceImpl.java:236`）覆盖之；全链路无 catch 吞异常（`applyFieldChanges` 的 switch 无 try；`applyToVersion` 无 try；confirm 循环 `:286-321` 无 try；嵌套 `@Transactional` 的 `regenerateForInstance`/`generateForInstance` 默认 REQUIRED 加入外层，异常照常标记回滚）。PARAM_ERROR 被整批回滚而非吞掉。✓
  2. **既有测试零回归**：独立复跑 2387/0/0/0 BUILD SUCCESS（原基线 2384 + 修订净增 3），全部既有确认 / 预览 / 发布 / 快照组装测试绿。✓
  3. **m-1 仅登记不改、0083 复用核实项留痕**：修订报告 `TASK-0081-report.md` §二.3（`:406-410`）明记会话/权限耦合事实、deferred 不修理由及「交接给 0083：须显式核实发布侧无用户会话/权限模型不同场景」；§四.1（`:440`）与一句话结论（`:452`）复述。`workitems/tasks/TASK-0083.md` 存在且描述复用 `applyToVersion`。留痕成立（受「仅追加本报告」契约约束，未在 0083 正文复述，不构成缺口）。✓
  4. **m-2 变更面**：仅新增一次返回值消费与纯 VO/日志文案（`实际 N 行` / `占用跳过未补插`），不改 0013 幂等占用语义、不改数据落行路径，无新异常路径。✓
  5. **I-1 变更面**：`safeMapPeriod` 保留且注释化旧默认宽容；`requireMapPeriod` 为新增私有方法；无对其它字段分支的连带修改。✓

---

## 二、新发现分级

- **Critical**：无。
- **Important**：无。
- **Minor**：无（发现清单为空）。

**非阻断观察（不构成发现，不列级）**：`newSeconds = requireMapPeriod(...)` 位于写入条件 `if (row.getCalcPeriod() == null || …)` **之前**，故即使实例已覆盖 calc_period（本不会写库），新默认坏码仍会抛异常整批回滚，较原评审建议的最小修正面更严。方向为 fail-closed、与 0013 同源纪律一致，不产生错误写入或数据损坏；且含非法默认的模板版本在 0013 生成链同样不可用，判定为可接受的更严口径。若后续希望「已覆盖实例不受坏默认阻断」，需另立裁决调整判定顺序，不属本轮退回理由。

---

## 三、结论

**通过（scoped 复评 round2，终审）**。

三问全过：I-1 双侧语义正确且负向测试对旧实现必红；m-2 以 `toInsert.size()` 为真实落行数、留痕如实；修复 diff 未引入新 Critical/Important。独立实测 2387 测试全绿、既有 2384 零回归、足迹 2 文件全落授权范围、m-1 已留痕 deferred。原评审 I-1 阻塞项闭合，m-1/m-2 处理符合 brief 要求，无需再退回。
