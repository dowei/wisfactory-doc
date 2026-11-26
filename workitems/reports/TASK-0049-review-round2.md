# 代码评审：TASK-0049 scoped 复评 round2（终审）

- 评审角色：代码评审（口径 `D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）
- 评审对象：修复 diff `fded29b..331f860`（单提交 6 文件，限 `calc/**`）
- 评审输入（仅依据 brief 指定文件 + 工作区源码）：
  - 修复 diff 包 `TASK-0049-diff-round2.txt`；原评审 `TASK-0049-review.md`（基线 F-01/F-02 Important + F-03~F-07 Minor）；修订报告 `TASK-0049-report.md`「修订轮」段（R1 红绿 / R4 真机补验条件）；实施 brief `TASK-0049-brief.md`
  - 契约：`resources/contract/config-snapshot-schema.md:165`、`resources/contract/tdengine-super-tables.md:15,37`、读侧 `backend/ot-admin/.../OpenApiTimeseriesClient.java`
  - 工作区源码（HEAD=`331f860`）：`calc/src/main/java/net/otlab/calc/status/{TdengineTraceWriter,StatusEvaluatorRuntime,StatusEvaluator}.java` + 两份契约测试 + `calc/README.md` + `docs/workitems/decisions/ADR-0010.md`
- 契约遵守：fresh 判定（未向协调者追问）、no-subagents（未再派）、只评审不修改（产出只进本报告）。

## 越界反查（复核）

`git diff --name-only fded29b..331f860` = 6 文件（README + 3 主码 + 2 测试），全部 `calc/**`；`git log fded29b..331f860` 仅 `331f860` 一提交，无交错；backend/frontend/ingest/契约文件零触碰；工作区仅 `?? .agents/`、`?? .tmp-t0044v/`（开工前既有）。**越界：无。** diff 包正文与实际 `git diff` 关键 hunk 逐行一致（已核）。

---

## 复评三问逐条结论

### 问 1 · F-02 是否真修复（full_key 前缀契约错位）

**结论：通过（真修复）。**

- 出处：
  - 修复 `calc/src/main/java/net/otlab/calc/status/TdengineTraceWriter.java:141-149`：`String fullKey = instanceCode + "." + STATUS_DATA_CODE;` 写入 `TAGS (1L, '<fullKey>', ...)`，旧 `'full:' + ...` 前缀已删除。
  - 测试 `calc/src/test/java/net/otlab/calc/status/TdengineTraceWriterContractTest.java:34-65` 新增 `tagsSixTupleUsesContractFullKeyWithoutPrefix`。
  - 契约 `config-snapshot-schema.md:165`（fullKey = instance_code + "." + data_code，TDengine 标签/消息/公式直接使用）、`tdengine-super-tables.md:15,37`（六 tag 顺序 tenant_id/full_key/instance_code/data_code/category/source_type）、读侧 `OpenApiTimeseriesClient.java:106,119,124`（`WHERE full_key = '<fullKey>'`，白名单 `SAFE_CODE=[A-Za-z0-9_.]+` 不含冒号）。
- 一句话说明：工作区源码已把 full_key 改为规范值（测试样例 `F01.WS01.EQ001.STATUS` = instance_code + "." + STATUS），与冻结契约及读侧等值查询口径逐字一致；新增测试经 `tagsOf()` 解析 `TAGS (...)` 后对六元组**逐值精确断言**（`tags[0]=1` / `tags[1]=F01.WS01.EQ001.STATUS` / `tags[2]=F01.WS01.EQ001` / `tags[3]=STATUS` / `tags[4]=status` / `tags[5]=1`）+ `assertFalse(sql.contains("full:"))`，非仅 SQL 形态断言——对旧 `full:` 前缀实现 `tags[1]` 必不匹配，红转绿成立（报告 R1.1 红证 `expected F01.WS01.EQ001.STATUS but was full:F01.WS01.EQ001.STATUS` 与代码一致）。附：旧带冒号值还会被读侧白名单拒绝，修复后规范值可被读侧查询命中。

### 问 2 · F-01 是否充分处置（真写入证据不足）

**结论：通过（本轮可做到的上限已闭合；真机补验按遗留项，不阻断代码收尾）。**

- 出处：
  - 修复 `TdengineTraceWriter.java:82-117`：`rest()` 的 `body.contains("\"code\":0")` 替换为 `requireTaosSuccess(body)`；新增 `requireTaosSuccess`（空/无 code/非 0 一律 fail-closed）与 `extractCode`（正则 `"code"\s*:\s*(-?\d+)`，容错冒号后空白、零新增依赖）。
  - 测试 `TdengineTraceWriterContractTest.java:85-104` `restParsesCapturedTaosAdapterResponseSamples`：code:0 三形态 + 冒号带空格变体通过；code 9728 / 空 / 无 code 抛 `TraceWriteException`。
  - 报告 `TASK-0049-report.md`「修订轮」R1.2 / R4（R4 明确「不能声称 st_item_value 留痕『真实写入可用』」「在 1-4 完成前标记为未验真（环境阻断）」）、R5（「当前标记未验真」）。
- 一句话说明：应答解析已抽成可离线单测的纯逻辑并用真 taosAdapter 样本（含 9728 错误码与空格变体）锁定——旧 `contains("\"code\":0")` 对 `{"code": 0, ...}` 字面不匹配、必判失败，「空格变体必红」成立；真机「写入 + 读回」因 REST 仍故障（本人复核报告原始输出：21s reset / 15s 超时 / HTTP=000，服务器侧故障，非代码问题）未能在环境完成，但报告 R4 已把它清晰登记为**环境恢复后的交付条件**并全程标注「未验真」，未误导为「已验真」——按协调者裁决该条件作为遗留项不阻断本轮代码收尾。

### 问 3 · 修复 diff 是否引入新 Critical/Important

**结论：通过（无新 Critical / Important）。** 逐项核：

1. **F-03 isOffline 六形态与恢复路径** — 出处 `StatusEvaluatorRuntime.java:92-110`（`isOffline`）、`207-217`（回调 `evaluate(offline, now)`）；测试 `StatusRuntimeContractTest.java:103-129`（`lwtPayloadParsesOnlineField` 六形态 + `lwtOnlineEventReturnsToRuleEvaluationNotOffline` 端到端）。正则 `"online"\s*:\s*(true|false|"true"|"false")` CASE_INSENSITIVE + `!group(1).contains("true")`：布尔/字符串 false、缺字段、坏 payload 均 → offline=true；布尔/字符串 true → offline=false。端到端测试实证 RUNNING→LWT offline(OFFLINE/LWT)→online:true 回 RUNNING/RULE。离线事件仍走 `evaluate(true)` → `StatusEvaluator.evaluate:48-51` 离线优先裁决 OFFLINE（`lwtOfflineEventTriggersReevaluationAndOverrides` 锁定），「LWT 离线优先」未被破坏；恢复路径 offline=false→规则求值（RULE）闭合，且无第二状态写口。正确，无新 C/I。
2. **F-05 文档顺序订正** — 出处 `calc/README.md:40,112,113`、`StatusEvaluator.java:14` javadoc 均改为「LWT 离线优先（ADR-0010）→ 直绑短路 → 规则」；实现 `StatusEvaluator.evaluate:48-70` 顺序确为离线→直绑→规则。`ADR-0010.md:12` 载明「离线只是判定输入、最终状态经优先级链裁决」，`ADR-0009.md:14` 载明「直绑优先（于规则）」——改动保持直绑优先于规则，仅把 LWT 置于最前且与代码/ADR-0010 一致，未引入语义冲突。无新 C/I。
3. **163 测试零回归（159+4）** — 报告 R2.1 原始输出：163 测试 0F0E3Skip BUILD SUCCESS；工作区 `@Test` 计数 `StatusRuntimeContractTest`=8（原 6，+2）、`TdengineTraceWriterContractTest`=5（原 3，+2），合计 +4 且 `StatusEvaluatorContractTest`=8 不变，与报告数字自洽。无回归证据。
4. **F-04/F-06/F-07 deferred 留痕** — 修复 diff 6 文件不含 `Resp2StateWriter`（F-06）、不含序号种子/fixture 值修订（F-07，`StatusRuntimeContractTest.java:23` 仍 FAULT=3）、`StatusEvaluatorRuntime.evaluate:135` 仍先 `current.put` 再写（F-04 原样）；报告 R3 已逐条登记 deferred。仅留痕未改码，符合裁决。

---

## 新发现（分级）

- **Critical：无。**
- **Important：无。**
- **Minor：无。**（修复 diff 未引入新问题；原 Minor F-03/F-05 本轮已顺手修正，F-04/F-06/F-07 按裁决 deferred，不重开原评审已通过部分。）

---

## 总体结论

**通过。** 三问全过且无新 Critical/Important，按 brief「三问全过且无新 C/I 时明确给出『通过』结论」：

- F-02 真修复，六元组逐值断言锁定，与 `config-snapshot-schema.md:165` 及读侧 `OpenApiTimeseriesClient` 查询口径逐字一致；
- F-01 应答解析抽出并离线单测（含空格变体必红证据），真机写入+读回已清晰登记为环境恢复后的交付条件（R4），标注「未验真」、未误导「已验真」；
- F-03/F-05 顺手修正正确且不破坏既有语义，163 测试零回归，F-04/F-06/F-07 仅 deferred 留痕。

**遗留项（不阻断代码收尾）**：TDengine 恢复后按报告 R4 序列补跑「建 scratch 库 → 经生产 `TdengineTraceWriter.rest()` 真写 → 按规范 fullKey 读回断言 value=seq/quality=0/UNKNOWN -1+3 → 未变不重写 → 清理」，完成前 st_item_value「真实写入可用」维持「未验真（环境阻断）」。此项属用户侧环境动作，非代码缺陷。
