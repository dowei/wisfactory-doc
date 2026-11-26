# TASK-0063 scoped 复评 round2 终审报告（代码评审）

- 评审者：fresh 独立代码评审（no-subagents；只评审、只写本报告；未改动任何源码 / workitems / 任务状态）
- 口径：L1-execute §13「diff 评审」；scoped 复评，只覆盖修订范围（round1 的 F-1 / F-2）+ 回归确认 + 终审红线
- 评审对象：隔离 diff `fb8f66d..640c9bd` 中的 **4 文件**（DDL + CalcReadServiceImpl javadoc + 2 契约测试）；区间内并行提交 `2fc2a4d`（TASK-0084，ingest/**）已隔离、不在本范围
- 本轮性质：**终审**（已用尽「1 退回 + 1 复评」上限）
- 证据来源：brief 指定文件（round2 diff 包 / round1 review / round1 brief / 报告修订轮段 / 实施 brief）+ 工作区 `D:\workspace\WisFactory` 源码与 surefire 报告 + 独立只读探针

## 独立核验动作（本次实做，非引用报告自证）

1. **范围核对**：`git show --name-only --format="%H %s" 640c9bd` = 4 文件（SQL 1 + calcread main 1 + test 2）；`git diff-tree -r 640c9bd` 同 4 项。`fb8f66d..640c9bd` 区间共 11 文件，其中 7 个 `ingest/**` 属 `2fc2a4d`（0084），排除后恰为上述 4 文件。**calc/** / frontend / ingest / 冻结契约在 640c9bd 中零改动。**
2. **F-1 切分算法逐字比对**：`backend/scripts/sql-exec.sh:34` 为 `sql.split(";\\s*\\n")`、`:38-42` 逐行剥 `--` 注释；`IotCalcDdlSplitContractTest.executableSegments()`（测试 :41-58）逐字复刻同一算法。判定「真实复刻」成立。
3. **F-1 红/绿独立模拟**（纯文本、未落盘、未连库）：把当前 DDL 的建号 3 行折回旧单行形态后跑同一算法 → OLD(13 段、2 段含残余分号) 两项断言均 **RED**；当前 NEW(17 段、0 残余分号、6 条建号语句各命中 1 次) 两项均 **GREEN**。与报告 §修订轮二的 RED 原文（两条 exact bad segment）及 GREEN 一致。
4. **surefire 独立聚合**：ot-admin 236 份报告 **2461 / 0F / 0E / 0Skip**；calc 43 份报告 **288 / 0F / 0E / 3Skip**。新增两测试类报告存在且 2/2、2/2 全绿（mtime 07:33，与报告修订轮四一致）。守卫 `IotPermissionsCompleteGuardTest` 4/0/0/0、`OperateLogCoverageGuardTest` 3/0/0/0 全绿。3 Skip 落 `RedisArrivalSubscriberLiveTest`(1) + `StatusRedisLiveTest`(2)，即 0041/0049 env 门控，与报告口径一致。
5. **只读探针（本轮实跑，未执行任何 DDL/未创建任何对象）**：经 `sql-exec.sh` 查 `information_schema` → `SCHEMA_NAME` 仅表头无 `iot_calc` 行、`rw_account=0`、`ro_account=0`、`DONE`。确证用户 phpStudy 库无该 schema、无 rw/ro 账号，DDL 仍未执行。
6. **支持类 API 核对**：`OpenApiInstanceRow`（Lombok `@Data` → `setId` 存在）、`VisibleScope.allTenant`/构造器、`SecurityUtils.getUserId`、`CalcStateVO.getFullKey`、`instanceCodeOf()` 取 `parts[2]` 均与测试调用一致。

---

## 一、F-1（Important）闭合判定 —— **已闭合（真修复）**

| 复评点 | 证据 | 判定 |
| --- | --- | --- |
| rw/ro 建号 `PREPARE/EXECUTE/DEALLOCATE` 逐条独立成行 | DDL 最终态 `:65-67`（rw）、`:76-78`（ro）各 3 行、每行以 `;\n` 结束；头注 `:14-15` 补记「按 `;\n` 切分、每条语句必须独占一行」 | ✓ |
| 与 `sql-exec.sh` 的 `;\n` 切分契约一致 | 切分算法逐字比对一致；独立模拟 NEW 17 段、0 残余分号，每段恰一条可执行语句（含 SET/PREPARE/EXECUTE/DEALLOCATE/GRANT/FLUSH） | ✓ |
| 契约测试真实复刻切分算法 | `executableSegments()` 与 `sql-exec.sh:34-42` 逐字一致，非字符巧合式断言 | ✓ |
| 对旧脚本必红、修后绿 | 独立模拟确认 OLD 两项 RED / NEW 两项 GREEN；报告 RED 原文与模拟 bad segment 一致；surefire `IotCalcDdlSplitContractTest` 2/0/0/0 绿 | ✓ |
| 新增「每行最多一条语句」机械断言覆盖全文 | `everyStatementIsOnItsOwnLine()` 遍历全脚本所有切分段（非只建号处），`segments.size()>=8` 防解析口径失真；`accountStatementsEachOnOwnLine()` 锁 6 条建号语句各独占一行 | ✓ |
| 只读探针实证通道可用且未执行 DDL/未建对象 | 报告 §修订轮三探针全 `information_schema`/`SELECT`（`OK/OK/OK/1/OK/DONE`）；本轮独立探针确认无 schema、无 rw/ro 账号；未创建任何对象 | ✓ |

round1 F-1 的**可应用性缺陷**（单行建号在 `sql-exec.sh` 通道报 `SQLERROR near 'PREPARE...'`、留下无账号半套 schema）已被结构性消除，并从「脚本文本 + 全脚本机械守卫 + 真实通道探针 + 红转绿」四层锁定。**判定闭合。**

## 二、F-2（Minor）闭合判定 —— **已闭合（订正到位）**

| 复评点 | 证据 | 判定 |
| --- | --- | --- |
| javadoc 改为「单段 fullKey 仅 allTenant 可见、受限角色 fail-closed 裁掉」 | `CalcReadServiceImpl.java:35-39` 现文：「单段 fullKey（如 state 侧 metricCode，无实例段）无法定位实例：仅 allTenant 角色可见；部门/实例受限角色一律裁掉（返回空集，fail-closed 不泄露）……」 | ✓ |
| 与 `visible()` 实装一致 | `visible()`（`:125-139`）：`allTenant()` 直过；`instanceCodeOf()` 为 null → `return false`；越权实例 → `return false`。与订正文案逐字相符 | ✓ |
| 与真实写入口径一致 | 与 round1 已确认的写入链一致：state fullKey 来自 `item.fullKey()`（单段 metricCode），log 为四段全键；文案明确区分两者 | ✓ |
| 契约测试走真实 `visible()`（非 deps=null 捷径） | `CalcReadDataPermContractTest.serviceWithDataPerm()` 用 `new CalcReadServiceImpl(jdbc, instanceMapper, dataPermService)` 三参构造，`state()` 经 `dataPermService.resolve()` → `visible()` 逐行裁剪；非 `new CalcReadServiceImpl(jdbc)` 捷径 | ✓ |
| 锁定受限角色裁单段/越权行、allTenant 全通过 | 受限角色用例：3 行中仅 `T\|M.B\|I01\|day` 存活（单段 `M.A` 裁掉、越权 `I99` 裁掉）；allTenant 用例：3 行全过含 `M.A`。断言走行为、非恒真 | ✓ |
| 裁剪逻辑未被改动（安全方向不动） | 640c9bd 中 `CalcReadServiceImpl` 仅有 javadoc hunk 变化，`visible()` 逻辑零改；报告 §修订轮五-2 已登记「受限角色单段 fullKey 空集为安全保守口径，口径变更另开任务」 | ✓ |

round1 F-2 的**文档/实现不一致 + 该路径无端到端测试**两点均已消除：文档如实反映 fail-closed 行为，并新增真实 `visible()` 路径的 2 项端到端锁定。**判定闭合**（闭合的是「订正与证据」；底层保守裁剪行为按 round1「方向正确、不改逻辑」的指令保持，属已知可接受口径，不构成新发现）。

## 三、回归与越界确认 —— **通过**

- **回归**：ot-admin 2461 = 2457 基线 + 4 新增（DDL 切分守卫 2 + 数据权限端到端 2），独立 surefire 聚合 0F0E0Skip；calc 288 与修订前逐包计数一致（state 6+4+4+5+5+7=31、config=12 含 +1），0F0E3Skip（0041/0049 env 门控不变）。报告修订轮四的 Results 段与实测吻合。
- **0047/0050/0053 既有测试零回归**：全量守卫绿、逐类计数一致，`CalcStateStoreReplaceabilityContractTest`（0047 注入缝）4/0/0/0 仍绿。
- **范围**：640c9bd 恰 4 文件全落声明范围（DDL + calcread 包），无新增越界；并行 `2fc2a4d`（0084 ingest 7 文件）已隔离、不在本 diff。
- **纪律**：DDL 仍「只产出不执行」——报告声明 + 全部测试 H2/mock + 本轮只读探针实证无 schema/无账号。本评审未执行任何写库语句。

## 四、新 Critical / Important 检查 —— **无**

本轮改动仅：DDL 纯格式拆行 + 头注（机械安全、可应用性改善）、javadoc 文案订正（对齐既有行为）、2 个纯新增契约测试（仅增量覆盖）。**生产逻辑（裁剪/读写/端点）零改动**，未引入新的 Critical 或 Important，也未引入新的 Minor 越界或泄露面。

### 非阻断观察（不构成发现）

1. `everyStatementIsOnItsOwnLine()` 以「段内残余 `;`」为 F-1 形态判据；若未来脚本在字符串字面量（如 COMMENT）中引入分号会误报为红。测试注释已声明当前脚本无此类字面量，且误报方向为「失败即红」的 fail-safe，不产生漏放或越权，故不构成发现。
2. 报告 §修订轮三的 `sql-exec.sh` 切分探针输出（PREPARE/EXECUTE 的对象均为 `SELECT` 常量）为执行者自证，本轮未逐字回放该探针；但「切分算法逐字比对 + 红/绿独立模拟 + surefire 绿 + 只读探针无对象」已形成充分独立证据链，不影响判定。

## 五、结论

**通过（终审）**。

round1 的 **F-1（Important）已真修复并闭合**：DDL 建号六语句拆为各占一行，与 `sql-exec.sh` 的 `;\n` 切分契约一致，`IotCalcDdlSplitContractTest` 逐字复刻切分算法、对旧脚本必红（独立模拟 2F）修后转绿，全脚本「每行最多一条语句」机械断言覆盖全文，真实通道只读探针实证无 DDL/无对象创建。**F-2（Minor）已订正到位并闭合**：javadoc 改为「单段 fullKey 仅 allTenant 可见、受限角色 fail-closed 裁掉」，与 `visible()` 实装及真实单段 metricCode 写入口径一致；新增 `CalcReadDataPermContractTest` 走真实 `visible()` 路径（非 deps=null 捷径），锁定受限角色裁单段/越权行、allTenant 全通过，裁剪逻辑零改。

**无新增 Critical / Important**。回归可信（ot-admin 2461 / calc 288 独立 surefire 复核一致，0047/0050/0053 零回归）；范围仍仅 640c9bd 的 4 文件、并行 0084 已隔离、零新增越界；DDL 仍严守「只产出不执行」（本轮只读探针确认无 schema/无账号）。

本评审未修改任何源码、未推进任务状态、未派子代理（唯一写操作为本报告）。TASK-0063 保持「进行中」，任务状态由协调者按流程处置。
