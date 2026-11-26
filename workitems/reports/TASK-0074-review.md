# 评审报告：TASK-0074（BUG-0002 修复，BUG 修复类加严档）

- 评审对象：`2fc80ba`（git show = TASK-0074-diff.txt，4 文件全落 ingest/**：tdengine.go、tdengine_contract_test.go、tdengine_injective_contract_test.go（新）、pipeline_contract_test.go）
- 依据：BUG-0002 单（根因/修复方案）、TC-0023（步骤 2-5）、TASK-0074 详情（转义集定长化裁决 + 加严要求）、TASK-0074-report、TASK-0073-review.md 复评节 N1（原发现）
- 方法：diff 全文读完；两项加严证据独立复验（非转抄）；四像分区做对抗样张实测（仓库外临时副本探针，不入库）

## 0. 范围与复跑核实

- `2fc80ba` 父提交 = BASE `509a82b`，无交错提交；4 文件全落 ingest/**，非 ingest 文件 0（git 核实）。
- 独立复跑：`go build ./... && go vet ./...` 干净；`go test ./... -count=1` 6 包全 ok；`go test ./internal/tdengine -run 'Contract' -v` **11/11 PASS**；全仓 `-run 'Contract'` 0 FAIL。

## 1. 加严证据 1：全仓同契约点扫描 — 通过（独立复扫）

- **实现点唯一**：`escapeIdentifier` / `escapeBody` / `truncateName` / `isNameByte` 全部仅存在于 `ingest/internal/tdengine/tdengine.go`；对外出口仅 `ChildTableName`（点位域）与 `DBName`（租户域，`"iot_" + escapeIdentifier(...)` 为全仓唯一库名构造语句）。
- **无绕过路径**：生产 SQL 构造点全部取值于两出口——`CreateDatabaseSQL`/`UseSQL` → `DBName`；`InsertPointSQL` 与 `Writer.WritePoints` → `childTable` → `ChildTableName`。全仓 grep `"iot_"` 前缀与 `INSERT INTO|CREATE DATABASE|USE` 构造：tdengine.go 之外仅命中注释与测试断言；测试侧全部经导出函数动态取值（`tdengine.DBName(...)`/`ChildTableName(...)`），无第二个映射实现。
- **backend 侧**：无 TDengine 子表名/库名构造（查询面仅渲染标签条件 `instance_key IN (...)`/`dept_id IN (...)`，属另一契约点），与 BUG-0002 写入侧映射不重叠。

## 2. 加严证据 2：回归测试先红后绿 — 通过（红轮独立复现）

- **红（真实性核验通过）**：在仓库外临时副本以 BASE `509a82b` 的 tdengine.go + 本轮新增 TC-0023 测试复跑，实测复现 5 项 FAIL，探针对碰撞结果与报告 §3.1 及 TASK-0073 复评 N1 探针**逐字节一致**（子表名 `F01_2dWS01_2aad8c0256245aa0`、库名 `iot_t_2d1_5627091943b2a601`，FNV 值 `2aad8c0256245aa0`/`5627091943b2a601` 相同）——红轮证据真实，非事后编造。
- **绿（独立复跑通过）**：修复后 11/11 PASS、全仓全绿、vet 干净（§0）。
- **TC-0023 断言完整落地**：步骤 2 探针对 → `TestContractChildTableNameProbePairNotColliding`；步骤 3 成组单射 → `TestContractChildTableNameInjective`（14 条编码两两不等，覆盖探针对/下划线/中文/首数字/三类满长仅末尾不同）；步骤 4 租户域 → `TestContractDBNameInjective`（9 条租户两两不等 + `iot_` 前缀 + 测试侧独立正则合法性判定，不依赖实现侧 `safeIdentifier`）；步骤 5 确定性 → 两测内重复映射断言；另有 `TestContractProbePairDistinctInsertSQL` 端到端写语句闭环。无缺项。

## 3. 特别核验点（新缺陷排查）— 均通过

1. **四像分区严密性**：恒等像（全 `[A-Za-z0-9]` 字母开头）不含 `_`；转义像中 `_` 只作转义引导、后随恰 2 位小写十六进制（自同步码、可唯一解码，`_` 纳入转义后首轮「字面 `_XX` vs 转义 `_XX`」构造族不复存在）；标记像 `_74` 不可达（0x74='t' 属字面集恒不转义，自然转义输出不可能以 `_74` 开头）；截断像含 `_Z`（大写 Z 非小写十六进制，违反转义文法，其余三像均不可能产出）。对抗样张实测（旧 BUG 构造族、字面 `_Z`+16hex 伪装、`_74` 开头伪装、63/64/65 连字符跨截断边界、满长仅末尾不同、中文、96 满长字面）：**无任何碰撞**，边界行为精确（64×3=192 为转义像、65×3=195 进截断像、截断像恰 192 字节）。
2. **截断域与声明一致性**：跨长度截断碰撞为 FNV-64 工程抗碰撞（鸽笼定理下 288 字节域→192 字节像全域严格单射不可能），报告遗留① 已如实声明；TC-0023 的截断断言对为「满长仅末尾不同」——末字节差异在 FNV 末步双射下**可证**不同，不依赖概率；**全部 TC 断言域落在严格单射区或可证区，无断言依赖未证区**，声明与实际覆盖面一致。
3. **租户域同口径**：`DBName` 与点位域同一映射函数同一论证，独立测试 + N1 租户探针对回归锚点落实。
4. **原 TASK-0073 行为语义**：`pipeline.go` 生产代码未动，`pipeline_contract_test.go` 仅把硬编码 `iot_group_a.F01_...` 断言锚点改为经映射函数动态取值（断言对象不变）；重排上限 3 次/死信路由/去重/丢最旧等语义未被触碰，全量测试绿。

## 4. 发现清单

### Critical（0 条）

无。

### Important（0 条）

无。

### Minor（2 条）

1. **M1 · 单射性声明措辞过宽（文档精度，不改行为）**
   - 出处：`ingest/internal/tdengine/tdengine.go` escapeIdentifier 注释「类内对『同长度、任意位置不同』的输入可证不同」；TASK-0074-report 遗留①「已证部分：同长度不同输入……可证不同」同向过宽。
   - 说明：FNV-1a 中途差异后状态可重合（后一步相等只需 `h⊕h' == c⊕c'`，8 位即可补偿），数学上可证的只有「差异全部位于末字节」的同长输入（TC-0023 的满长仅末尾不同对恰在此区，断言域不受影响）；报告 §3.3.4 措辞（「满长仅末尾不同……可证」）是准确的。建议随 0029 命名定版把注释与遗留①收窄为「仅末尾差异可证、其余同长/跨长为 64 位工程抗碰撞」。
2. **M2 · 附录 B 示例租户库名字面偏离（裁决固有代价，已声明）**
   - 出处：`ingest/internal/tdengine/tdengine.go` DBName + `TestContractSuperTableDefinitionsMatchAppendixB` 注释；报告偏离 1。
   - 说明：`_` 纳入转义（裁决 prescribed）后，契约附录 B 示例租户 `group_a` 的库名由字面 `iot_group_a` 变为映射名 `iot_group_5fa`——保 `_` 字面则探针对碰撞复活，偏离为单射性必然代价，报告与测试注释均已声明；0027/0029 建库脚本与查询必须统一走 `DBName()`。备查：存在保字面示例的另一单射方案（字面集排除 `Z`、以 `_Z`+2hex 为转义引导，原理同自同步码），若上游/运维在意字面示例可在命名定版时评估，非缺陷。

### 留白/遗留确认（非缺陷）

- 截断像跨长度工程抗碰撞（遗留①，鸽笼定理固有，TC 断言域不受影响）。
- TAGS 值含单引号的 SQL 字面转义（沿 TASK-0073 遗留口径，随 0029 真执行器统一处理）。
- `_74`/`_Z` 标记与 192 字节上限的真机行为验证挂 TD-0053/TASK-0029。
- 复评 N2/N3 按 TASK-0073 复评结论保持 deferred，不在本 BUG 范围。

## 5. 结论

- 两项 BUG 修复加严证据均独立核验通过：映射实现点唯一无绕过；先红后绿真实（红轮碰撞逐字节复现）且 TC-0023 全部断言落地。
- 修复方案（转义集定长化、四像分区、截断后置）经论证与对抗样张实测均成立，无新引入缺陷；2 条 Minor 均为文档精度/已声明偏离记录级，不进修订轮。

VERDICT: PASS
