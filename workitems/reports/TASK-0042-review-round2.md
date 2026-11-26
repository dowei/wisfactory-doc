# TASK-0042 scoped 复评 round2 报告（代码评审）

- 评审角色：fresh 代码评审子代理（no-subagents；只依据评审 brief 指到的文件 + 工作区源码判定，不共享协调者上下文，不向协调者追问；判断不了的标「无法判定」）
- 评审口径：`.agents/protocol/L1-execute.md` §13「diff 评审」复评条款——**scoped：只看修复 diff 与原发现 I-1，不重开原评审已通过部分**；修订上限 1 退回 + 1 复评，本轮为终审
- 输入（brief 必读，均已读）：
  - 修复 diff 包 `reports/TASK-0042-diff-round2.txt`（`e387c3b..01236fd`，限 `calc/`，2 文件）
  - 原评审 `reports/TASK-0042-review.md`（I-1 Important + M-1/2/3；评审重点 7 判定）
  - 修订轮报告段 `reports/TASK-0042-report.md`「修订轮」段
  - 实施 brief `reports/TASK-0042-brief.md`
  - 工作区源码 `D:\workspace\WisFactory\calc\src\**`
- 被评审提交：`01236fd`（父 `e387c3b`）；实测 HEAD=`01236fd`，父提交哈希精确等于 `e387c3b`，区间仅此一提交、无交错。
- 评审日期：2026-11-24
- 契约遵守：全程只评审、未修改任何项目文件；所有产出仅写入本报告。

---

## 〇、越界反查复核（协调者已比对，评审者独立复核）

- 实测 `git show --name-only 01236fd` = 2 项：`calc/src/main/java/net/otlab/calc/executor/DualTriggerExecutor.java`、`calc/src/test/java/net/otlab/calc/executor/OperatorIntegrationContractTest.java`；`git diff --stat e387c3b..01236fd` = 2 files, 37 insertions, 6 deletions。与 diff 包逐字一致。
- 全部落 `calc/**`（calc/executor）；`backend/`、`frontend/`、`ingest/`、契约文件零触碰。报告 R3.2 记录的 `frontend/**` 工作区并行改动属其他道在途，本提交未 add/未提交/未触碰，不计入本任务越界。
- **复核结论：越界反查成立，0 越界。**

---

## 一、复评三问逐条回答

### 问 1：I-1 是否真修复 —— **通过**

- **结论**：真修复。三问子项全过（严格全串匹配 → 三种复合形态 fail-closed → 纯 DELTA 仍绿 → 负向测试对旧实现必红）。
- **出处**：`calc/src/main/java/net/otlab/calc/executor/DualTriggerExecutor.java:234-240`（正则 `matcher(formula.trim())` + `m.matches()` + 抛 `IllegalStateException`）；同文件 `:153-157, 322-329`（异常捕获 → `markFailed`）；`calc/src/main/java/net/otlab/calc/executor/CalcResult.java:33-34, 37-38`（`failure` 的 sample=null、`isSuccess()=sample!=null`）；`calc/src/test/java/net/otlab/calc/executor/OperatorIntegrationContractTest.java:83-107`（负向测试 3 形态）；旧实现 `git show e387c3b:...DualTriggerExecutor.java:231-246`（`find()` + `CalcResult.success`）。
- **一句话说明**：`matches()` 要求 trim 后整串恰为 `DELTA(key)`，`DELTA(x)/3600`（尾 `/3600`）、`2*DELTA(x)`（前缀 `2*`）、`DELTA(x)+1`（尾 `+1`）三种形态全串匹配必失败 → 抛 `IllegalStateException` → 被 `execute()` 的 `catch (RuntimeException)` 捕获走 `markFailed` → `CalcResult.failure`（sample=null、error 非空、不发假 0、不写 store），单节点隔离；纯 `DELTA(x)` 仍 `matches()` 成功走原生算子（测试 `deltaRunsNativelyThroughExecutorWithBaselineCarriedAcrossPeriods` 仍绿）；旧实现用 `find()` 只取首个 `DELTA(x)` 即 `CalcResult.success`，对三形态都会成功并写出原始差分，故新增 `assertFalse(r.isSuccess())` 对旧实现必红（报告 R1 红证 `expected: <false> but was: <true>`、`failed=0`，逻辑推演一致）。

### 问 2：是否引入新 Critical / Important —— **无新 C/I**

- **结论**：无新 Critical、无新 Important；195 测试零回归经独立复跑确认。
- **出处**：
  - 校验器白名单形态：`calc/src/main/java/net/otlab/calc/graph/FormulaExpressionValidator.java:24`（`WHITELIST=Map.of("DELTA",1)`）、`:27`（`BANNED=INTEGRAL`）、`:99-112`（标识符/函数名切分：`DELTA(` 紧邻才判函数）、`:159-162`（白名单精确 `Map.get`）；`FormulaExpressionValidatorTest.java:86-87, 131-132`；`GraphBuilder.java:145-146`（快照侧强制校验）。
  - 执行器路由：`DualTriggerExecutor.java:219-222`（`containsFunction` 用 `\bDELTA\s*\(`）、`:232-240`。
  - 测试复跑：本评审独立执行 `cd calc && mvn test -q` → `EXIT=0`；`target/surefire-reports/TEST-*.xml` 26 份汇总 `tests=195 failures=0 errors=0 skipped=3`（源码 `@Test` 实测 195，`@TestPropertySource` 非测试注解已排除）。
  - 失败隔离纪律：`DualTriggerExecutor.java:153-157, 322-329`；`CalcResult.java:33-34`；原评审 `TASK-0042-review.md:39, 83-84`（0041 失败隔离 + CONV-02 不写假 0）。
- **一句话说明**：
  - **不误伤合法形态**：校验器白名单对函数名区分大小写（`DELTA` 精确匹配），小写 `delta(x)` 在校验器即被「未知函数」拒绝，`DELTA (x)`（名与括号间空白）同样被校验器以「相邻操作数」拒绝——均非合法形态；执行器正则对纯 `DELTA(x)`（含首尾空白、括号内空白、点号 fullKey）与校验器合法集一致，唯一新增拒绝的是校验器接受但执行器本就无法正确求值的**复合表达式**，而这正是 I-1 要求的 fail-closed，不是误伤。
  - **异常路径纪律一致**：非纯形态抛 `IllegalStateException`，经既有 `catch (RuntimeException)` → `markFailed` → `CalcResult.failure`（sample=null、error 非空），与 0041「单节点失败隔离」及 CONV-02「不打假 0」同源，错误文案含 `DELTA`/`形态` 可被下游与测试识别；失败结果不进入 `store.put`，同构写侧对 `!isSuccess()` 跳过（原评审已通过部分，未改动）。
  - **零回归**：独立复跑 195 测试 0F0E3Skip、26 类全绿；= 194 基线逐类保持 + 新增 1（`compositeDeltaExpressionFailsClosedNotSilentlyWrongValue`，内含 3 形态循环断言）；唯一注入算子桥的测试类即 `OperatorIntegrationContractTest`，纯 DELTA 用例仍绿。
- **边界说明（非新发现）**：校验器接受复合 DELTA、执行器改为拒绝，二者存在「快照期接受 / 运行期隔离」的不对称；这是 I-1 裁决的**在范围内、已登记取舍**（报告 R1「复合表达式求值归 FEAT-0012 表达式引擎，本轮不引 FEAT-0012」），把静默错值换成隔离失败，数据完整性严格更优，不构成本轮新 C/I。

### 问 3：Minor 处置确认 —— **如实登记、未隐藏**

- **结论**：M-1/M-2/M-3 均如实登记且可在报告定位；round1 重点 7 结论不因本修订改变。
- **出处**：`reports/TASK-0042-report.md` 修订轮 R2 表 `:362-364`；首部 `:4`（BASE 双字段）；§三未做项 `:273-275`、§五遗留 `:308-309`；原评审 `reports/TASK-0042-review.md:54, 64, 86`。
- **一句话说明**：
  - **M-1**（满量程字段缺 → 溢出码 1 主链不可达）：R2 `:362` 登记 deferred，理由为快照无满量程字段、`maxRange` 恒 null，接线需点位档案或 CONV-03 契约升级（超修订范围）；§三未做项 2 `:274` / §五遗留 2 `:309` 同源演进位并存——未隐藏。
  - **M-2**（INTEGRAL ts 跳变缺口）：R2 `:363` 登记 deferred，保留原评审「无法判定」（上游 quality=3 补标契约工作区不可确认），INTEGRAL 主链未接线，本轮不改算子语义——未隐藏。
  - **M-3**（BASE 字段订正）：R2 `:364` 记「报告订正」，首部 `:4` 已改为 `BASE=331f860`（开工 HEAD）+ `diff/评审 BASE=e49fea8`（提交父节点）双字段并列——已落实。
  - **round1 重点 7**（TDengine 真机未验真）：原评审 `:54/:86` 判定「充分（环境阻断下可达上限），不阻断收尾；保持未验真标记」；本修订 diff 仅动 DELTA 公式形态判定，未触真机写口/live 证据，报告 §三 `:273`、§五 `:308` 仍持「未验真（环境阻断）」标记——**结论不变，且未被据以宣称已验真**。

---

## 二、新发现清单

| 级别 | 编号 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| Critical | 无 | — | — |
| Important | 无 | — | — |
| Minor | 无（本轮新增） | — | 原 M-1/M-2/M-3 为 round1 已登记项，本轮仅复核处置（见问 3），非新发现；round1 重点 7 结论不变。 |

---

## 三、结论

**通过。**

- 三问全过：问 1（I-1 真修复，三复合形态 fail-closed、纯 DELTA 仍绿、负向测试对旧实现必红）、问 2（无新 Critical/Important，195 测试 0F0E3Skip 独立复跑零回归，未误伤合法形态、异常路径符合 0041/CONV-02 纪律）、问 3（M-1/M-2/M-3 如实登记未隐藏，重点 7 结论不变）。
- 越界反查复核成立：提交 `01236fd` 2 文件全落 `calc/**`，区间单提交无交错。
- 按 L1-execute §13，本轮为「1 轮退回 + 1 轮 scoped 复评」的终审，复评通过，无新 C/I 需再退回。保持「未验真（环境阻断）」的 TDengine 真机写入+读回补验条件（报告 §三末 1-5）仍为环境恢复后的强制交付后条件，补验前不得宣称真实写入可用。
