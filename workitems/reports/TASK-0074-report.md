# TASK-0074 实施报告：TDengine 子表名与租户库名转义映射定长化修复（单射性）

- **BASE**：`509a82b`（主仓 main，开工时 `git rev-parse --short HEAD` 现取）
- **最终提交**：`2fc80ba`（main，提交信息注明 TASK-0074；父提交即 BASE `509a82b`，无交错提交，提交边界干净）
- 执行者：Go 道执行者（implementer），2026-11-22
- 修复对象：BUG-0002（TASK-0073 复评 N1 分流）；验收用例 TC-0023
- 工具链：go 1.27.2 windows/amd64；零第三方依赖（go.mod require 段仍为空）

## 0. 交付概述

BUG-0002 根因：首轮 `escapeIdentifier` 的转义集（`_`+十六进制）与恒等集（`_` 原样保留）重叠——被转义字符的展开 `_XX` 本身是另一条合法编码的原样形态，构造性碰撞；且转义像追加的 FNV 后缀串同样是全安全字符串，可被字面编码恒等命中。修复按裁决转义集定长化：字面集收窄为 `[A-Za-z0-9]`、`_` 本身纳入转义（`_5f`），四像分区 + 截断后置，映射在契约合法域的可严格证明区单射（见 §4 论证）。TC-0023 全部断言先红后绿落账。

## 1. 改动文件清单（4 个，全部 `ingest/**`，与提交 `2fc80ba` 一致）

| 文件 | 性质 | 说明 |
| --- | --- | --- |
| `ingest/internal/tdengine/tdengine.go` | 修改 | `escapeIdentifier` 重写：字面集 `[A-Za-z0-9]`（`_` 移出恒等集）、`escapeBody` 定长转义（非字面字节含 `_` → `_xx` 小写十六进制，自同步码）、数字开头不可达标记 `_74`、截断移至转义后（`truncateName`：`_Z`+16hex 违反转义文法后缀，恰 192 字节）；`DBName` 租户域同口径；包注释更新；`safeIdentifier` 保留为映射后不变式断言 |
| `ingest/internal/tdengine/tdengine_contract_test.go` | 修改 | INSERT 形态/分组测试的子表名与库名断言改经映射函数动态取值（契约锚点 USING/TAGS/VALUES 不变）；附录 B DDL 测试保留四条 STABLE 逐字 + 建库/USE 按映射名并注明偏离；`TestContractChildTableNameEscaping` 按新方案重写（恒等/`_5f`/`_74` 标记/截断像 192 字节与后缀形态）；`TestContractPointCodeMappedNotRejected` 期望更新（转义像不再带 FNV 后缀） |
| `ingest/internal/tdengine/tdengine_injective_contract_test.go` | 新增 | TC-0023 全部断言 ×4：探针对回归锚点、点位域成组单射（14 条编码两两不等 + 确定性）、租户域单射（9 条编码两两不等 + 确定性 + iot_ 前缀 + 合法性独立判定）、探针对端到端 INSERT 语句不相同 |
| `ingest/internal/pipeline/pipeline_contract_test.go` | 修改 | 两处硬编码 `iot_group_a` 断言改为 `tdengine.DBName("group_a")` 动态取值（断言对象不变，随映射口径） |

## 2. 验收命令原始输出

### 2.1 `cd ingest && go build ./... && go vet ./...`

```
=== CMD1: go build ./... && go vet ./... ===
EXIT=0 (no output above = clean)
```

### 2.2 `cd ingest && go test ./... -count=1`

```
=== CMD2: go test ./... -count=1 ===
?   	net.otlab/ingest/cmd/ingestd	[no test files]
ok  	net.otlab/ingest/internal/cache	1.414s
ok  	net.otlab/ingest/internal/message	1.493s
ok  	net.otlab/ingest/internal/pipeline	2.557s
ok  	net.otlab/ingest/internal/quality	0.985s
ok  	net.otlab/ingest/internal/snapshot	2.977s
ok  	net.otlab/ingest/internal/tdengine	0.949s
EXIT=0
```

### 2.3 `cd ingest && go test ./internal/tdengine -run 'Contract' -count=1 -v`（单独复跑）

```
=== CMD3: go test ./internal/tdengine -run 'Contract' -count=1 -v ===
=== RUN   TestContractInsertShapeNumInstant
--- PASS: TestContractInsertShapeNumInstant (0.00s)
=== RUN   TestContractInsertShapeStateAndKindMapping
--- PASS: TestContractInsertShapeStateAndKindMapping (0.00s)
=== RUN   TestContractSuperTableDefinitionsMatchAppendixB
--- PASS: TestContractSuperTableDefinitionsMatchAppendixB (0.00s)
=== RUN   TestContractBatchInsertMultiRowSameChild
--- PASS: TestContractBatchInsertMultiRowSameChild (0.00s)
=== RUN   TestContractWriterGroupsByChildTable
--- PASS: TestContractWriterGroupsByChildTable (0.00s)
=== RUN   TestContractPointCodeMappedNotRejected
--- PASS: TestContractPointCodeMappedNotRejected (0.00s)
=== RUN   TestContractChildTableNameEscaping
--- PASS: TestContractChildTableNameEscaping (0.00s)
=== RUN   TestContractChildTableNameProbePairNotColliding
--- PASS: TestContractChildTableNameProbePairNotColliding (0.00s)
=== RUN   TestContractChildTableNameInjective
--- PASS: TestContractChildTableNameInjective (0.00s)
=== RUN   TestContractDBNameInjective
--- PASS: TestContractDBNameInjective (0.00s)
=== RUN   TestContractProbePairDistinctInsertSQL
--- PASS: TestContractProbePairDistinctInsertSQL (0.00s)
PASS
ok  	net.otlab/ingest/internal/tdengine	0.869s
EXIT=0
```

11 项 Contract 断言全部 PASS（其中 4 项为本轮新增的 TC-0023 单射性断言）。

### 2.4 `git show 2fc80ba --name-only`

```
=== CMD4: git show 2fc80ba --name-only ===
commit 2fc80ba
parent 509a82b
author dowei

ingest/internal/pipeline/pipeline_contract_test.go
ingest/internal/tdengine/tdengine.go
ingest/internal/tdengine/tdengine_contract_test.go
ingest/internal/tdengine/tdengine_injective_contract_test.go
EXIT=0

=== 非ingest文件计数 ===
0
```

4 个文件全部 `ingest/**`，非 ingest 文件计数 0。

### 2.5 全仓同契约点扫描（映射调用点/实现点）

```
=== 扫描1：全仓 Go 文件中的映射函数定义与调用点 ===
（grep -rn --include="*.go" -E "escapeIdentifier|escapeBody|ChildTableName|childTable|DBName\(|safeIdentifier|isNameByte|truncateName" .）
实现点（全部在 ingest/internal/tdengine/tdengine.go）：
  tdengine.go:81   func isNameByte        —— 字面集判定
  tdengine.go:95   func escapeBody        —— 定长转义展开（唯一转义实现）
  tdengine.go:134  func escapeIdentifier  —— 映射主函数（唯一入口）
  tdengine.go:163  func truncateName      —— 截断像构造
  tdengine.go:177  return "iot_" + escapeIdentifier(tenantCode)   —— DBName（租户域）
  tdengine.go:242  name := escapeIdentifier(pointCode)             —— ChildTableName（点位域）
生产调用点（全部经上述两入口，无旁路）：
  tdengine.go:196  CreateDatabaseSQL → DBName
  tdengine.go:201  UseSQL → DBName
  tdengine.go:262  InsertPointSQL → childTable(first)（= ChildTableName）
  tdengine.go:380  Writer.WritePoints → childTable(r)（= ChildTableName）
测试调用点：tdengine 两测试文件 + pipeline_contract_test.go（经导出函数动态取值，无独立实现）

=== 扫描2：全仓 Go 文件中库名前缀 iot_ 的构造点 ===
唯一构造语句：tdengine.go:177  return "iot_" + escapeIdentifier(tenantCode)
（其余命中均为注释或测试断言，无第二个库名构造点）

=== 扫描4：Go 侧 SQL 语句构造点（INSERT/CREATE/USE） ===
tdengine.go:196  CREATE DATABASE → DBName（已映射）
tdengine.go:201  USE → DBName（已映射）
tdengine.go:207-213  CREATE STABLE ×4 → 契约冻结超级表名（非映射对象，附录 B 逐字）
tdengine.go:271  INSERT INTO %s.%s USING %s.%s → DBName + childTable（均已映射）

=== 扫描5：backend 侧 TDengine 相关 ===
命中文件：DataPermServiceImpl / TdengineTagConditionRenderer / DataPermInstanceRow /
VisibleScope / DataPermInstanceMapper / SysTenant / QuotaProperties / IotQuotaStatsService 等
——均为数据服务查询面（权限标签条件渲染、配额统计读路径），不构造子表名/库名映射，
与 BUG-0002 的写入侧映射是不同契约点；且改动范围纪律（只落 ingest/**）不含 backend。
```

**扫描结论**：映射实现点唯一（`internal/tdengine/tdengine.go` 的 `escapeIdentifier`，点位域经 `ChildTableName`、租户域经 `DBName` 两个出口，全部 SQL 构造点均取值于这两个出口），全仓无第二实现、无绕过路径。

## 3. 先红后绿证据链（BUG 修复加严要求）

### 3.1 红（TC-0023 断言先写、先跑，BUG-0002 实测复现）

新增 4 项单射性测试与按新方案的期望更新后、实现未动，`go test ./internal/tdengine -run 'Contract' -count=1` 实测（关键失败行）：

```
--- FAIL: TestContractPointCodeMappedNotRejected (0.00s)
    tdengine_contract_test.go:224: 子表名未按 _XX 十六进制转义（连字符 0x2D → _2d）: INSERT INTO iot_group_a.F01_2dWS01_2dM01_c75a42f0e2d0fa31 USING ...
    tdengine_contract_test.go:238: DBName 未按 _XX 转义（分号 0x3B → _3b、空格 0x20 → _20）: "iot_tenant_3b_20drop_69d693d9e5334753"
--- FAIL: TestContractChildTableNameEscaping (0.00s)
    tdengine_contract_test.go:258: underscore pointCode = ("F01_WS01_EQ001_M01", <nil>), want F01_5fWS01_5fEQ001_5fM01
    tdengine_contract_test.go:263: digit-first = ("t_01_M01", <nil>), want _7401_5fM01
    tdengine_contract_test.go:272: hyphen = ("F01_2dWS01_2aad8c0256245aa0", <nil>), want F01_2dWS01
--- FAIL: TestContractChildTableNameProbePairNotColliding (0.00s)
    tdengine_injective_contract_test.go:44: BUG-0002 复现：不同 pointCode 映射到同一子表名 "F01_2dWS01_2aad8c0256245aa0"（探针对碰撞）
--- FAIL: TestContractChildTableNameInjective (0.00s)
    tdengine_injective_contract_test.go:91: 单射性破坏：code[0]="F01-WS01" 与 code[1]="F01_2dWS01_2aad8c0256245aa0" 同映射子表名 "F01_2dWS01_2aad8c0256245aa0"
--- FAIL: TestContractDBNameInjective (0.00s)
    tdengine_injective_contract_test.go:132: 租户域单射性破坏：tenant[0]="t-1" 与 tenant[1]="t_2d1_5627091943b2a601" 同库名 "iot_t_2d1_5627091943b2a601"
FAIL
FAIL	net.otlab/ingest/internal/tdengine	1.107s
```

两个探针对的实测碰撞结果与复评 N1 逐字节一致（子表名 `F01_2dWS01_2aad8c0256245aa0`、库名 `iot_t_2d1_5627091943b2a601`，含独立计算的 FNV 值 `2aad8c0256245aa0`/`5627091943b2a601` 也与评审报告相同）——同一根因的独立复现确认。

### 3.2 绿（实现后复跑）

修复提交后同命令 11/11 PASS（§2.3 原始输出），全仓 `go test ./... -count=1` 6 包全绿（§2.2），`go vet` 无告警（§2.1）。

### 3.3 为什么现在对了（单射性论证）

1. **转义集定长化消除构造性碰撞族**：字面集收窄为 `[A-Za-z0-9]`，`_` 与一切非字面字节统一展开为 `_xx`（2 位小写十六进制）。展开是自同步码：输出中 `_` 只作为转义引导出现、后随恰好 2 位小写十六进制，字面字节永不产生 `_`——因此任意输出串可被唯一解码回唯一输入，逐字节单射 → 整体单射。首轮的碰撞构造（「字面编码恰等于另一编码的完整转义结果」）不再成立：`F01-WS01` → `F01_2dWS01`（转义像，含 `_` 引导），而字面 `F01_2dWS01…` → `F01_5f2dWS01…`（`_` 被转义），两者永不相等。
2. **像分区**：恒等像（全 `[A-Za-z0-9]`、字母开头）不含 `_`；转义像至少含一个 `_`（转义路径仅在编码含非字面字节或数字开头时进入）——两像不相交。
3. **数字开头标记不可达**：数字开头编码加前缀 `_74`（字节 0x74='t' 的转义形）。't' 属字面集、永远不会被转义，故任何自然转义输出都不可能以 `_74` 开头（若输出以 `_74` 开头则首字节必为 0x74='t'，矛盾）——标记像与转义像不相交；标记像含 `_` 与恒等像不相交。
4. **截断后置 + 违法文法后缀**（裁决要点）：截断在**转义后**进行——超长（转文体 >192 字节，如 96 个汉字=288 字节）时截断像 = 转文体前 174 字节（数字开头 171 + `_74` 标记）+ `_Z` + 16 位小写十六进制 FNV = 恰 192 字节。`_Z` 中 'Z' 不是小写十六进制，违反转义文法——任何合法转义输出都不含 `_Z`，故截断像与恒等像/转义像/标记像均不相交，堵死「B=decode(截断名)」的构造碰撞（首轮遗留风险）。截断像类内：TC-0023 的「满长仅末尾不同」对为同长输入，FNV-1a 每步 `(h XOR b)·p`（p=16777619 为奇数，XOR 与乘以奇数皆双射）状态差保持到末步——**可证**散列不同，不依赖概率。
5. **租户域同口径**：`DBName = "iot_" + escapeIdentifier(tenant)`，与点位域同一映射函数同一论证；两租户探针对 `t-1`/`t_2d1_5627091943b2a601` 分别映射 `iot_t_2d1`/`iot_t_5f2d1_5f5627091943b2a601`，不再收敛同库。
6. `safeIdentifier` 保留为映射后的不变式断言（恒等像/转义像/标记像/截断像均满足标识符文法），不再构成对契约合法数据的拒绝路径；测试侧另用独立实现的正则判定（不依赖实现侧函数）双重校验。

## 4. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 复评 N2（required 非空校验超出裁决字面）、N3（重排死信缺 batchID/点位上下文） | 评审裁决记 TASK `任务结果` deferred，不在本 BUG 修复范围 |
| backend 查询面的 TDengine 相关代码 | 数据服务读取路径（权限标签条件渲染），非写入侧名字映射契约点；改动范围纪律只落 `ingest/**` |
| `-race` 竞态检测 | 本机无 C 编译器（cgo 不可用），沿用 TASK-0073 报告遗留口径 |
| 真 TDengine 实例上验证 192 字节标识符上限与自动建子表行为 | 本地无 TDengine，挂 TD-0053（TASK-0068 联调）/ TASK-0029 接线 |

## 5. 偏离计划之处

1. **附录 B 示例租户的库名偏离字面示例**：契约附录 B 示例租户 `group_a` 的字面库名为 `iot_group_a`；按裁决「租户库名同口径处理」后 `DBName("group_a") = "iot_group_5fa"`（`_` 必须转义才保单射，否则租户域探针对仍碰撞）。库名规则 `iot_<tenant_code>` 本身未变，映射偏离已在 DDL 契约测试中注释声明（`groupa` 等无下划线编码仍恒等，`iot_groupa` 形态与契约示例同构）。0027/0029 接线与运维建库脚本须以 `DBName()` 输出为准。
2. **恒等域收窄的波及面**：`_` 纳入转义后，所有含下划线的编码（含契约示例点位 `F01_WS01_EQ001_M01`、租户 `group_a`）的子表名/库名从恒等形态变为转义形态。契约只定「每点位一张子表、命名方案归写入方」，形态变化合规；TAGS 的 point_code 值保留原编码不变，查询侧按标签过滤不受子表名影响。
3. **转义像不再追加 FNV 后缀**：首轮为「转义像与恒等集区分」加的后缀在 `_` 纳入转义后失去必要性（两像靠下划线有无天然分区），移除以缩短名字、减少一处非必要变换；期望更新随本轮测试一并落账。

## 6. 遗留问题

1. **截断像跨长度对为工程抗碰撞（非证明单射）**：契约合法域（≤96 字符任意 UTF-8，至多 288 字节）到 192 字节标识符的全域严格单射受鸽笼定理限制数学上不可能（2^2304 > 2^1165）。已证部分：同长度不同输入（含「满长仅末尾不同」）经 FNV-1a 逐步双射可证不同；恒等像/转义像/标记像/截断像四像两两不相交为构造性证明。未证部分：不同长度的超长编码经「截断+64 位 FNV」理论上存在碰撞可能（需转文体前 174 字节相同且 64 位 FNV 相撞），现实编码空间下概率可忽略；若上游将来对 pointCode 收紧字符集或长度（如 ≤64 字节），可升级为全域严格单射，建议随上游 §3.3 补充稿一并观察。
2. **TAGS 值含单引号的 SQL 字面转义**（TASK-0073 报告遗留 7 沿用）：子表名经映射已无此风险，TAGS 字面量仍原样放入编码，随 0029 真执行器统一处理字符串转义口径。
3. **`_74`/`_Z` 标记与 TDengine 版本行为**：标识符合法性（192 字节上限、首字符规则）按 TDengine 3.x 文档口径实现，真机行为待 0029 接线验证；若未来版本放宽/收紧标识符规则，仅需调整 `maxIdentifierLen` 与 `safeIdentifier`。

## 7. 可追溯性

- git 提交：`2fc80ba`（main；只 add `ingest/` 明确路径 4 个文件；父提交 = BASE `509a82b`）
- BUG/TC 追溯：BUG-0002 根因（转义集与恒等集重叠）→ 修复方案（转义集定长化）逐条落地；TC-0023 步骤 2（探针对回归锚点）、步骤 3（成组两两不等：探针对/纯字母数字/下划线/中文/首数字/三类满长仅末尾不同）、步骤 4（租户域两两不等）、步骤 5（确定性）全部入自动化断言（`TestContractChildTableNameProbePairNotColliding` / `TestContractChildTableNameInjective` / `TestContractDBNameInjective`，另有 `TestContractProbePairDistinctInsertSQL` 端到端闭环）
- 验收标准核对：TC-0023 全部断言 PASS（§2.3）；go build/vet/test 全绿（§2.1/2.2）；全仓扫描无第二个映射实现点（§2.5）；先红后绿证据在 §3
