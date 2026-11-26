# TASK-0064 diff 评审报告（代码评审）

- 评审对象：提交 `537366e`（BASE=`7771181`，单提交，4 文件 630 行新增）——高危权限点机械差集收口（种子 SQL + 守卫测试 + 两个比对/导出脚本）
- 评审口径：`L1-execute.md` §13「diff 评审」（与 TASK 验收标准 / brief 接口一致性 + 明显质量缺陷）
- 评审方式：fresh 评审者，只依据评审 brief「必读」清单与工作区源码 / PDF 权威 / 现库实态独立复核；未派 subagent；除本报告外未修改任何文件
- 评审日期：2026-11-23

## 结论：通过

无 Critical / Important 发现。6 项 Minor 见下文「发现清单」（按 §13 记 TASK `任务结果` deferred，不进修订轮）。

## 评审重点逐条核对（brief 六项）

### 1. 机械差集正确性 —— 核实成立

- **40 项提取清单可独立复现**：评审者用 pypdf 对 §3.1 上=p21 / 下=p22 / 续=p23（0 基 20-22，共 36 页 PDF）独立重跑提取（含连字还原 + 换行并行 + 斜杠组展开 + config→publish 归一），得到与报告完全一致的 40 项清单。
- **权限列抽样复核（pypdf 原文）**：`POST /api/v1/iot/point/import → iot:point:import`（p21）、`GET /api/v1/iot/point/export → iot:point:export`（p21）、`PUT /api/v1/iot/item/bind → iot:item:bind`（p22）、`PUT /api/v1/iot/item/formula → iot:item:edit`（公式未单列，p22）、`POST /api/v1/iot/config/rollback → iot:config:publish`（回滚复用发布位，p22——与报告偏离项 2 一致）、凭据写/吊销=`iot:credential:edit` + 解密查看=`iot:credential:view` 双分离（p23）——全部与报告核验表一致。
- **§3.4/§3.5 豁免核实**：p26 实为「3.4 计算引擎内部接口；3.5 数据服务对外 API」，表列为「方法/路径/调用方/说明」，无 sys_menu perms 权限列——种子头注的豁免理由成立。
- **差集恰 3 项为真**：`git grep` 在 BASE=`7771181` 的 `resources/sql/` 全集内三个码（iot:point:import / iot:point:export / iot:item:bind）零命中；BASE 代码侧仅 javadoc 注释提及 iot:item:bind「未在 sys_menu 登记」的缺口记录（IotDataItemController.java:42 @7771181），本提交后已收口。
- **两处抽取伪影修正正确且无残留**：p21-23 全文本中连字字符仅 U+FB01 一类（脚本同时防御性处理 U+FB02），还原后无残留；斜杠组正则在三页上的全部命中恰为 3 处真实权限单元格（template/instance/metric 的 `add / edit / remove`），无误捕跨行拼接。
- **现库种子全集对照**：评审者以独立实现（列序感知 + 剥注释的 Python 模拟，非复用执行者代码）解析 44 个种子文件得 177 个注册值，与主代码反射口径提取的 171 个注解值比对——**缺失为 0**；`bash resources/scripts/perm-diff-check.sh` 现场实跑两轮，`缺注册` 段输出 `(空) —— 零缺失`，退出码 0。

### 2. 首轮 grep 漏四段式修正 —— 核实覆盖

- 修正后的两条提取口径均无段数上限：守卫测试 Java 解析器按显式列清单取 `perms` 列原值（IotPermissionsCompleteGuardTest.java:214-226 一带，不依赖固定列位）；`perm-diff-check.sh` 的 grep 正则 `'[a-z][a-z0-9_]*(:[a-z0-9_*]+){1,}'` 允许 2 段及以上任意段数。四段式 `iot:template:item:edit`（4029 行实态在库，20261122_iot_template_domain.sql:155）在两条口径下均被正确计入注册集——评审者独立模拟的 177 注册集含该值，171 注解值缺失为 0，交叉证实。
- `INSERT...SELECT` 形态（20260925 微信脚本范式）在守卫测试中有专门分支（selectBody，:200-207），微信种子解析实测覆盖。

### 3. 守卫测试全局口径 —— 核实为真全局；BUG-0046 表述非本任务笔误

- **真全局（反射而非 grep）**：`ClassPathScanningCandidateComponentProvider` 扫描 `net.otlab.admin.module` 基包 + `AnnotationTypeFilter(RestController)`，类级 + 方法级（getDeclaredMethods）反射取值；工作区实测 75 个 `@RestController` 全部落在该基包下，无遗漏面。常量形态注解（如 `MonitorActuatorController` 的 `@SaCheckPermission(ActuatorAccessPolicy.MONITOR_PERMISSION)`）运行期正常解析——静态 grep 反而漏它，反射口径是对的。无控制器继承链（无 extends），getDeclaredMethods 无父类盲区。防呆断言（controllers>50 / perms>150 / scripts>30）兜住扫描失效。
- **误报/漏报面**：漏报面=非 @RestController 的注解（无此类）与父类注解方法（无此形态）；误报面见 Minor-4（SELECT 体 WHERE 子句码并入注册集的潜在弱化），当前种子实测无误报。
- **BUG-0046 出处核实**：该表述是**既有守卫测试范式原句**——既有 `SaPermissionRegistryConsistencyTest` 的 DisplayName 即「BUG-0046 回归：全部 @SaCheckPermission 权限码均已登记 sys_menu SQL」、既有 `IotPointMenuPermissionGuardTest.java:90,95` 同款「BUG-0046 口径」，另含主代码 MonitorCacheController 在内共 8 个既有文件使用同一编号。台账（bugs.md / BUG-0001~0005）确无 BUG-0046，但该编号缺口是**先于本任务的历史现象**（L1-execute.md §9 亦引用超出台账的试点编号 BUG-0007/0010），本任务只是沿用既有口径，不构成本 diff 的缺陷。台账与代码引用的编号断层可作为台账治理的独立待办，不阻塞本任务。

### 4. 种子 SQL —— 核实成立

- **挂点**：4079/4080→4011（点位管理页）、4081→4010（实例管理页），父行均在既有种子（20261121_iot_console_menus.sql:43-44）；4081 与 4036-4040 数据项按钮同挂 4010，与设计「数据项归属实例」一致。
- **号段顺延**：既有 4000 段最大 4078（TASK-0052），4079-4081 顺延无冲突；头注「AUTO_INCREMENT=3000 口径」与 01_base.sql:20,33 一致。
- **超管授权三行**：`sys_role_menu` 恰 (1,4079),(1,4080),(1,4081)，照 0024（4070-4074）/0052（4075-4078）范式逐字一致；普通角色零反向授权，由守卫测试 `superadminIsTheOnlyGrantedRole` 机械锁定。
- **幂等**：INSERT IGNORE 语义成立（显式 id 主键冲突忽略）；报告实测两轮 DONE（二次 OK(0)）；评审者现场实跑核对查询确认现库恰 3+3 行。
- **头注登记**：八项高危核验 / 上游差异 / 执行口径 / 执行后核对齐全（瑕疵见 Minor-1）。

### 5. 脚本质量 —— 核实可重复执行且不误报

- `perm-diff-check.sh` 现场实跑两轮均成功、退出码 0、输出一致；`set -euo pipefail` + python 失败即中断（不会静默产出空清单）；mktemp + trap 清理临时文件；双源（种子文件 ∪ 现库实态）口径正确；DB 侧只读 SELECT。结果段「新种子三行现库实态」与报告逐值一致。
- `export-design-perms.py` 输出与评审者独立复现一致（40 项逐值相等）。
- 落点合规：brief 改动范围明文允许「清单导出脚本/比对脚本（临时产物放 `resources/scripts/`）」；4 文件越界反查通过（`git diff --name-only 7771181..537366e` 与声明逐一相符，`resources/scripts/` 为授权落点）。

### 6. 既有测试零回归 —— 独立复跑证实

评审者以官方口径复跑：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → **Tests run: 2370, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**（与报告 2370 一致，= 既有基线 + 本守卫 4 用例）。既有守卫测试不受全局口径扩展影响：`IotPointMenuPermissionGuardTest` 的行匹配正则限定 `405\d` 且仅读本域脚本（新行 4079/4080 不落入），注解集断言对象未变；`IotDataItemMenuPermissionGuardTest`、`SaPermissionRegistryConsistencyTest` 等全部绿。

## 发现清单

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | resources/sql/upgrade/20261123_iot_permissions_complete.sql:33-40 | 头注「号段登记」自称为全段位占用图，但漏 4008-4016（iot 控制台目录页 9 行，20261121_iot_console_menus.sql），且 4067-4069 实为状态字典（statusdict）三行却并入「4063-4069 状态规则」——登记图不精确，无功能影响 |
| 2 | Minor | resources/sql/upgrade/20261123_iot_permissions_complete.sql:59 | 4081 sort=6 与同父 4010 下既有 4035（实例树查询，sort=6）重号——菜单管理内排序歧义，纯展示问题（4079=6/4080=7 挂 4011 无冲突） |
| 3 | Minor | resources/scripts/perm-diff-check.sh:22-27 | 注册集提取按「全文件引号内冒号串」grep（含注释行、非 perms 列、SELECT 体 WHERE 子句引用码），有把非注册值计入注册集、掩盖真实缺口的潜在误报面——当前种子实测无误报（评审者以更严的列序感知解析交叉验证 0 缺失），且权威裁决在守卫测试的列感知解析器；属信息性脚本的潜在弱化 |
| 4 | Minor | backend/.../iot/config/IotPermissionsCompleteGuardTest.java:195-207 | INSERT...SELECT 分支把语句体内（含 WHERE NOT EXISTS 子句）全部引号权限样式字面量并入注册集，javadoc 仅以「条件引用的正是本语句登记的码」为前提——未来脚本若 WHERE 引用其它码会被误计入注册集（守卫弱化）；另 statementsOf 以首次 `VALUES` 关键字定位（:265 一带），SELECT 体若含 VALUES 字样会误判形态。当前种子实测均无误触发 |
| 5 | Minor | resources/scripts/export-design-perms.py:17-19 | 默认 PDF 路径硬编码本机绝对路径（D:/Documents/dowei/...）进可提交文件——AGENTS.md 有「不将文档库路径硬编码进可共享配置或提交」的告诫；有 argv 覆盖入口、brief 声明其为临时产物，故降为 Minor |
| 6 | Minor | backend/.../iot/config/IotPermissionsCompleteGuardTest.java:43,116,125 | 「BUG-0046 口径」沿用既有守卫测试原句（SaPermissionRegistryConsistencyTest / IotPointMenuPermissionGuardTest 等 8 个既有文件），非本任务笔误；但台账 bugs.md 仅 BUG-0001~0005，编号断层系历史遗留——建议另行治理（补登记或在代码侧改引既有编号），不阻塞本任务 |

## 独立验证记录（评审者实测）

1. `git diff --stat 7771181..537366e` → 4 files changed, 630 insertions(+)，与 diff 包一致；HEAD 即 537366e，工作区干净（仅未跟踪 .agents/）。
2. pypdf 独立提取 §3.1/续（p21-23）→ 40 项与报告清单逐值相等；权限列抽样 6 处 + §3.4/§3.5 豁免核实（p26 无权限列）。
3. BASE 提交 `git grep` 三码在 resources/sql 零命中 → 差集 3 项为真。
4. 独立 Python 模拟（列序感知 + 剥注释）解析 44 个种子文件：注册 177 值 vs 注解 171 值（含数组形态 `@SaCheckPermission(value={...}, mode=OR)`），缺失 0。
5. `bash resources/scripts/perm-diff-check.sh` 现场两轮：零缺失裁决、退出码 0、现库 4079-4081 三行 + 三条超管授权与报告一致。
6. 官方口径全量测试复跑：`Tests run: 2370, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS`。
