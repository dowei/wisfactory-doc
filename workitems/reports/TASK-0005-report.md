# 实施报告：TASK-0005（数据权限翻译组件（六步）实现）

- 执行者：implementer（subagent）
- 完成时间：2026-11-22 01:00（本地）
- 基线：d971c10（派发时主仓 HEAD）；本任务提交：**384d58f**（main，注明 TASK-0005）
- 口径权威：ADR-0016（七项裁决逐条落实）；设计素材 reports/TASK-0005-inputs.md；brief TASK-0005-brief.md
- 结果总览：**三条验收命令全部通过**；组件单测 65 例 0 失败 0 错误；全量回归 1200 例 0 失败 0 错误；改动范围核对通过

---

## 一、改动文件清单

提交 384d58f 共 24 个文件，全部落在 brief 声明范围内（新增包 `net.otlab.admin.module.iot.dataperm`）：

主代码（17）：

| 文件 | 职责 |
| --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermService.java` | 入口 API：`resolve(userId)`（六步翻译）+ `resolveClientScope(clientId, scopeJson)`（中台 scope，裁决 5） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermServiceImpl.java` | 六步实现：scope 并集 → 部门集合 → 实例锚点 → ancestors 元素前缀展开（上限截断 + WARN，裁决 1）→ 缓存接缝（先缓存后计算） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/VisibleScope.java` | 产物值对象 `{allTenant, deptIds, instanceIds, createByOnly}`（brief 定版形态，record 防御性拷贝） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermProperties.java` | `ot.dataperm.subtree-max`（默认 5000，裁决 1） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/VisibleScopeCache.java` | 步骤 6 缓存接口（get/put/evict；Redis 与主动失效归 TASK-0006） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/NoOpVisibleScopeCache.java` | NoOp 实现（本期直查） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermConditionAssembler.java` | 条件拼装三态：配置类 / 点位类 / 时序类（裁决 3/4/6）；NULL 结构性 fail-closed、绝不渲染 IS NULL |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/TdengineTagConditionRenderer.java` | TDengine 标签下推渲染器：`instance_key` / `dept_id`（裁决 7），编码单引号转义 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermAccessStrategy.java` | 策略常量 `SCOPE_EMPTY` / `SCOPE_FORBIDDEN` + `isDetailAccess(resource)` 形态判定（裁决 2，不接线） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermRoleRow.java` / `DataPermDeptRow.java` / `DataPermInstanceRow.java` | 组件内部只读行对象（自包含，不复用 system 实体，保证可整体迁出） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermUserMapper.java` | 只读：sys_user.dept_id（按需查询） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermRoleMapper.java` | 只读：sys_role join sys_user_role（启用角色 + data_scope） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermRoleDeptMapper.java` | 只读：sys_role_dept（自定义 scope 部门集合） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermDeptMapper.java` | 只读：sys_dept (id, ancestors)（scope=2 下级部门展开） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermInstanceMapper.java` | 只读：iot_model_instance 全量锚点行 / 按 id 精确行（全部参数化 @Select） |

测试（6，类名均 DataPerm* 以配合验收命令 2）：

| 文件 | 覆盖 |
| --- | --- |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermResolveScopeTest.java` | 五种 data_scope 可见集合、多角色并集（含 4 与部门角色混合）、scope=1 跳过部门求解、自定义查 sys_role_dept、无角色/未知 scope fail-closed、缓存接缝（命中不计算 / NoOp 直查） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermResolveSubtreeTest.java` | ancestors 前缀展开（多级 + 「0,11」≠「0,1」字符串前缀陷阱）、中段锚点只展开子树、上限截断保近端 + **WARN 逐字符断言**（`[DATAPERM] subtree truncated user=7 kept=3 cap=3`）、<=0 关闭上限、NULL 部门实例 fail-closed、默认值 5000 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermClientScopeTest.java` | 中台 scope JSON 三态（null/空=默认拒绝、subtree=false 精确、subtree=true 复用步骤 3-4）、畸形 JSON fail-closed、未知字段宽容、截断 WARN subject=clientId |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermConditionAssemblerTest.java` | 三类条件精确串断言、裁决 4 OR create_by 分支、无锚点表语义、1 = 0 fail-closed、NULL 归类（无 IS NULL）、时序 scope=4 无贡献 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermTagConditionRendererTest.java` | 标签名硬约束 instance_key/dept_id（禁旧称 instance_code/dept）、编码升序 + 引号转义、allTenant null、fail-closed |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermAccessStrategyTest.java` | SCOPE_EMPTY / SCOPE_FORBIDDEN 常量、isDetailAccess 集合尾段 / 标识符尾段 / 未知形态 fail-closed |

配置（1）：

- `backend/ot-admin/src/main/resources/application.yml`——仅新增 `ot.dataperm.subtree-max` 一个键（diff 已核对，无其他改动）：

```diff
+  dataperm:
+    subtree-max: ${OT_DATAPERM_SUBTREE_MAX:5000}   # 子树展开上限（TASK-0005 / ADR-0016 裁决 1）……
```

未触碰：frontend/、ingest/、既有登录链路与拦截器、workitems 库（TASK 详情未回写，按 brief 由协调者收尾）。

---

## 二、验收命令原始输出

### 命令 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 段（exit=0）：

```
[INFO] Installing D:\workspace\WisFactory\backend\ot-core\target\ot-core-1.0.0.jar to C:\Users\dowil\.m2\repository\net\otlab\ot-core\1.0.0\ot-core-1.0.0.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.185 s
[INFO] Finished at: 2026-11-22T00:54:32+08:00
```

ot-admin test 段 Results（exit=0）：

```
[INFO] Tests run: 1200, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 1200, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:14 min
[INFO] Finished at: 2026-11-22T00:55:51+08:00
```

结论：BUILD SUCCESS，0 失败 0 错误（基线 + 新增 65 例）。其中组件 6 个测试类合计：

```
DataPermAccessStrategyTest      Tests run: 5,  Failures: 0, Errors: 0
DataPermClientScopeTest         Tests run: 11, Failures: 0, Errors: 0
DataPermConditionAssemblerTest  Tests run: 20, Failures: 0, Errors: 0
DataPermResolveScopeTest        Tests run: 12, Failures: 0, Errors: 0
DataPermResolveSubtreeTest      Tests run: 7,  Failures: 0, Errors: 0
DataPermTagConditionRendererTest Tests run: 10, Failures: 0, Errors: 0
合计 65 例
```

### 命令 2：`mvn test -pl ot-admin -Dtest='DataPerm*'`（提交后单独复跑，exit=0）

```
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermAccessStrategyTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermClientScopeTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermConditionAssemblerTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermResolveScopeTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermResolveSubtreeTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermTagConditionRendererTest
[INFO] Tests run: 65, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

组件测试类数 = 6，用例数 = 65。

### 命令 3：改动范围核对 `git diff --name-only d971c10..HEAD`

说明：BASE（d971c10）与本任务提交（384d58f）之间落入了**并行任务提交** cff0642（TASK-0073 ingest 流水线内核，15 个文件全在 `ingest/` 下，与本任务无交集、非本任务改动）。因此同时给出原始口径与本任务隔离口径：

**本任务提交单独核对 `git diff --name-only cff0642..HEAD`（384d58f 全部改动）——恰好 24 个文件，全落在声明内：**

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermAccessStrategy.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermConditionAssembler.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermDeptRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermInstanceRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermRoleRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/NoOpVisibleScopeCache.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/TdengineTagConditionRenderer.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/VisibleScope.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/VisibleScopeCache.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermDeptMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermInstanceMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermRoleDeptMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermRoleMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/mapper/DataPermUserMapper.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermAccessStrategyTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermClientScopeTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermConditionAssemblerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermResolveScopeTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermResolveSubtreeTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermTagConditionRendererTest.java
```

**原始口径 `git diff --name-only d971c10..HEAD`** = 上述 24 文件 + cff0642 的 15 个 ingest/ 文件（TASK-0073，见上说明）。

---

## 三、未做项与原因

| 未做项 | 原因 |
| --- | --- |
| HTTP 端点接线、菜单种子、权限点、dev 冒烟 | brief 明定：本任务不接线任何端点（接线归 TASK-0006/0007/0012/0051），验收=单测+全量回归 |
| Redis 缓存实现与主动失效 | 归 TASK-0006；本任务只交付 `VisibleScopeCache` 接口 + NoOp 实现，resolve 路径已按「先缓存后计算」写好接缝 |
| TC-0002 红线集成验证（跨厂 403/空集、中台拉树、改绑定即时生效） | 归 TASK-0007（含测试夹具两厂两账号 + appKey）；组件级等价断言已含在单测：越权策略常量与形态判定、点位 NULL fail-closed、子树裁剪语义、scope JSON 三态 |
| TASK 详情状态回写 | brief 明定：不回写 TASK 详情，由协调者收尾；未改 workitems 库任何文件 |
| 未知 data_scope 值的宽松解释 | 有意不做：未知值（含 null）不产出任何可见性（fail-closed，与框架「未知收敛最紧档」同方向），有单测锁定 |

## 四、偏离计划之处

无实质偏离（组件落位、六步形态、测试口径、验收命令均按 brief 执行）。需协调者知悉的实现口径记录：

1. **TDD 过程**：先写 6 个测试类并运行（红灯=实现缺失编译失败），随后实现；首轮实现后 4 例失败，逐一定位根因修复——其中 **1 个实现缺陷**：`computeUserScope` 在无 scope∈{2,3} 角色时也无条件查 sys_user.dept_id（无消费方的查询），改为按需查询（为什么现在对了：用户部门仅本部门/本部门及下属档消费，scope=4/5-only 用户不再触发，与「跳过后续部门求解」语义一致，`never()` 断言通过）；另 2 处为测试缺陷（捕获 List 与 Set 的类型不等断言、渲染器测试漏 stub 编码查询）。复跑 65/65 绿。
2. **自包含 Mapper 的表清单**：brief 列举 sys_dept/sys_role/sys_role_dept/iot_model_instance；完成六步还需取「用户的角色」（sys_user_role join）与「用户的部门」（sys_user.dept_id）——这两个只读 select 补进组件自带 Mapper（仍为参数化 @Select，属自包含边界内的必要扩展）。
3. **mapper 包位置**：放在 `dataperm.mapper` 子包——根 `@MapperScan("net.otlab.admin.module.**.mapper")` 的模式约束，保证运行期可被扫描。
4. **subtree-max 误配口径**：`<=0` 视为关闭上限（不截断、不 WARN），而不是「截断为空集」——ADR 未定义，取「权限组件故障不得阻断业务」同方向（裁决 1 截断哲学），有单测锁定。
5. **中台 resolveClientScope 签名**：带 `clientId` 参数（brief 未定签名）——裁决 1 的 WARN 格式 `user=<id>` 需要主体标识，中台路径以 clientId 充当 subject（有单测断言 `user=5`）。
6. **新增一条补充日志**：畸形中台 scope JSON 发 WARN `[DATAPERM] client scope invalid, deny by default`（ADR 只钉了截断格式；本条同前缀、恒拒绝方向不变，仅提升配置错误可排查性）。
7. **点位类条件含 create_by 分支**：裁决 3 将点位列表归入配置类适用集（scope=4 按 create_by 过滤），故点位条件为「dept_id IN OR bound_instance_id IN [OR create_by=me]」；时序类仍无贡献（裁决 4 原文）。

## 五、遗留问题

1. **直查性能**：NoOp 缓存期每次 resolve 全量载入本租户 sys_dept（仅 scope=2 需要）与 iot_model_instance（步骤 3/4 共用），中台 subtree=true 同样全量载入——TASK-0006 Redis 缓存落地后按 (user_id, 角色版本) 消解；若单租户实例规模远超 subtree-max 5000，截断 WARN 的运维处置（放宽配置或数据治理）需在 TASK-0006 一并评估。
2. **iot_model_instance 建表依赖**：T4（TASK-0004）尚未建表，组件 SQL 含 `AND deleted = 0`（按 iot_ 业务表统一 BaseEntity 逻辑删范式预置）——建表时该表必须含 `deleted` 列，否则组件运行期 SQL 报错；联调点建议写入 TASK-0004 验收对照。
3. **租户上下文前提**：组件多租户隔离完全消费会话上下文（TenantContext + 多租户插件），自身不重复实现——数据服务验签后必须先显式设置租户上下文再调组件（TASK-0051/0052 接线时核对，否则插件跳过注入、翻译结果不保证租户内收敛）。
4. **isDetailAccess 形态词表**：集合语义尾段词表 = 裁决 2 五例 + page/export/count/stat/stats/options；未知形态 fail-closed 判详情类（403）。未来出现新集合动词（如 /timeline）需扩词表或由接口实现自选常量（裁决 2 允许，最终选择权在接口实现）。
5. **上游文档事项**：ADR-0016 影响范围已登记「详细设计 §4.1 标签旧称随 PDF 下游修订对齐」——本任务不处理 PDF 修订；组件与契约 v1.0.0 一致。
6. **消费方接线清单待排**：三类条件 API 与渲染器的具体消费点（实例树 / 点位 / 曲线 / 中台查询等）分属 TASK-0006/0007/0012/0051，本任务交付物已按「条件片段供拼 SQL」契约定型，接线时按类别对号入座即可。

---

## 六、修订轮（2026-11-22，验收退回 §13：Important 2 条）

评审报告 reports/TASK-0005-review.md（Critical 0 / Important 2 / Minor 6），Important 2 条按 ADR-0016 增补第 8/9 条修复。修订轮提交：**509a82b**（TASK-0005 修订轮，单提交）。Minor 6 条按处置建议不动、随收尾进 LOG deferred。

### 修订轮改动文件（6，全在 dataperm 包与测试内，application.yml 本轮未触碰）

- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermServiceImpl.java`——I-1 + I-2 实现
- `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/DataPermService.java`——接口契约 javadoc（守卫语义 + 中台 deptIds 恒空集）
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermClientScopeTest.java`——两条固化放大行为的断言按裁决 8 修正 + 接 mockStatic 租户上下文
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermResolveScopeTest.java`、`DataPermResolveSubtreeTest.java`——接 mockStatic 租户上下文（守卫落地后必需）
- `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/DataPermTenantContextGuardTest.java`——**新增**守卫测试类（4 例）

### I-1 中台授权放大（评审 Important-1 → ADR-0016 增补第 8 条）

- 修复：`exactScope(rows)`（从实例行推导 deptIds 并集）删除，改为 `clientScope(visibleIds)`——**中台路径 deptIds 恒为空集**，点位类条件仅实例分支（fail-closed）。subtree=false 与 subtree=true 两条路径都走此构造。
- 为什么现在对了：原实现把「可见实例绑定的部门」当作中台的部门可见性并入 `VisibleScope.deptIds`，`pointCondition`/`renderPointCondition` 渲染 `dept_id IN (部门集) OR bound_instance_id IN (实例集)` 后，部门分支会放行「与被授权实例同部门、但 scope 外」的其他实例的点位行/TDengine 子表（如 scope 只含某车间一台设备时，同部门兄弟设备的原始点位数据可见）——授权超出 iot_api_client.scope 声明。裁决 8 定版中台授权本体只有实例集合，部门分支不产出，用户路径（deptIds 本就是角色授权本体）不受影响。
- 测试修正（合法红转绿）：`DataPermClientScopeTest` 两条固化放大行为的断言（deptIds={10,11} / {10,20,30}）改为空集断言，DisplayName 与断言消息引用增补第 8 条；修正的是**固化了错误行为的测试**，非削弱断言。

### I-2 无租户上下文静默跨租户（评审 Important-2 → ADR-0016 增补第 9 条）

- 修复：`resolve` / `resolveClientScope` 入口检查 `TenantContext.getTenantId()==null` → 返回空 VisibleScope（allTenant=false、集合全空）+ WARN `[DATAPERM] no-tenant-context user=<id 或 unknown>`（用户路径 subject=userId、中台路径 subject=clientId、null 兜底 unknown）。
- 为什么现在对了：无上下文时 `OtTenantLineHandler.ignoreTable` 对所有表跳过租户注入，组件继续计算会静默产出**跨租户全量** VisibleScope（本组件设计上要防的最严重失效模式），且同线程下游查询同样无租户谓词；守卫使该失效模式从「静默放大」变为「显式拒绝 + 可 grep 告警」。守卫是防失效模式而非重复实现隔离——租户过滤仍完全由多租户插件按会话上下文注入（brief「消费上下文不重复实现」边界不破）。
- 新增 `DataPermTenantContextGuardTest`（4 例）：无上下文 resolve 空集 + WARN 逐字符断言（`[DATAPERM] no-tenant-context user=7`）+ verifyNoInteractions（缓存与全部 Mapper 不触达）；中台路径同（user=5）；null subject 走 `user=unknown` 档；有上下文（mock 注入租户）守卫放行、正常进入六步计算（不误伤合法调用）。
- 测试设施：resolve 系三个既有测试类以 `mockStatic(TenantContext)` 注入会话租户（getTenantId→1L）。
- 修订轮调试发现（记入测试设施注释）：**mockStatic 未打桩时 Mockito 默认答案对包装类型 Long 返回 0 而非 null**——守卫测试首版红灯实际未触发守卫（getTenantId=0 骗过入口检查，走 mock 空角色路径，与无上下文的空集形态巧合一致，仅 WARN 断言失败暴露）。修复为显式 `thenReturn(null)` 模拟真实无上下文形态（真实 TenantContext 无会话返回 null），并留注释防复发。

### 红转绿记录

修订轮红灯（实现前运行 `mvn test -Dtest='DataPerm*'`，69 例中 5 失败，全部为本次修复目标）：

```
DataPermClientScopeTest.testClientScope_ExactMatch_NoSubtreeExpansion
  expected: <[]> but was: <[10, 11]>            ← 增补第 8 条：固化放大行为的断言转红
DataPermClientScopeTest.testClientScope_SubtreeTrue_ExpandsDescendants
  expected: <[]> but was: <[30, 10, 20]>        ← 同上
DataPermTenantContextGuardTest.testResolve_NoTenantContext_DeniedWithWarn
  expected: <[[DATAPERM] no-tenant-context user=7]> but was: <[]>   ← 守卫缺失
DataPermTenantContextGuardTest.testResolveClientScope_NoTenantContext_DeniedWithWarn
  expected: <[[DATAPERM] no-tenant-context user=5]> but was: <[]>   ← 同上
DataPermTenantContextGuardTest.testResolve_NullSubject_WarnUnknown
  expected: <[[DATAPERM] no-tenant-context user=unknown]> but was: <[]>  ← 同上
[INFO] Tests run: 69, Failures: 5, Errors: 0, Skipped: 0
```

实现落地后首轮复跑：增补第 8 条两条断言即绿；守卫三条仍红（WARN 缺失）→ 定位根因为 mockStatic 默认答案返 0L（见上）→ 显式置 null 后全绿：

```
DataPermAccessStrategyTest        Tests run: 5,  Failures: 0
DataPermClientScopeTest           Tests run: 11, Failures: 0   ← 含修正后的中台 scope 断言
DataPermConditionAssemblerTest    Tests run: 20, Failures: 0
DataPermResolveScopeTest          Tests run: 12, Failures: 0
DataPermResolveSubtreeTest        Tests run: 7,  Failures: 0
DataPermTagConditionRendererTest  Tests run: 10, Failures: 0
DataPermTenantContextGuardTest    Tests run: 4,  Failures: 0   ← 新增守卫类
[INFO] Tests run: 69, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

### 补充验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`（修订轮提交后全量复跑）

ot-core install 段（exit=0）：`[INFO] BUILD SUCCESS`（Total time 6.025 s）。ot-admin test 段（exit=0）：

```
[INFO] Tests run: 1204, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 1204, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
```

1200（修订轮前全量）+ 4（新增守卫用例）= 1204，0 失败 0 错误。

### 补充验收 2：`mvn test -pl ot-admin -Dtest='DataPerm*'`（修订轮提交后单独复跑）

```
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermAccessStrategyTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermClientScopeTest
[INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermConditionAssemblerTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermResolveScopeTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermResolveSubtreeTest
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermTagConditionRendererTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataperm.DataPermTenantContextGuardTest
[INFO] Tests run: 69, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

组件测试类数 = 7（原 6 类 + 新增守卫类），用例数 = 69，全绿（含修正后的中台 scope 断言）。

### 修订轮范围核对

`git diff --name-only c2ff33e..509a82b`（509a82b = 修订轮提交；c2ff33e 为并行任务 TASK-0073 修订轮提交，与本任务无关）：上列 6 个文件，全部在 dataperm 包与测试范围内；application.yml、frontend/、ingest/、workitems 库均未触碰。
