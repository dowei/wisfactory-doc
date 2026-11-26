# 任务报告：TASK-0007（越权红线自动化测试——本期管理面红线固化 + 中台部分挂接线义务）

- 执行者：implementer（ZCode agent）
- 完成时间：2026-11-22（修订轮同日追加，见文末「修订轮」节）
- BASE：`1065f9a`（主仓 HEAD，与 brief 声明一致）
- 交付提交：首轮 `5d5a6e7`（main，注明 TASK-0007）；修订轮 `eb01ba2`（验收退回裁决授权的主代码修复）
- 结论：首轮 **3 条验收命令全部通过**（红线测试全绿，零主代码改动）；修订轮 **3 条补充验收全部通过**（详情口径不对称缺陷按裁决修复+红线套件转正+dev 冒烟复核）。红线二-中台部分按 brief 决策 2 占位登记，待 TASK-0051 接线后转正。

---

## 1. 改动文件清单

全部落在声明范围 `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/**`，**零主代码改动**，未触碰既有测试文件（复用夹具的最小调整授权未动用——不需要）：

| 文件 | 行数 | 内容 |
| --- | --- | --- |
| `redline/package-info.java` | 35 | 套件 charter：上线前越权专项=本套件全绿；CI 接入口径（命名规范 `*RedlineTest` + surefire 默认包含）；ADR-0016 判定规则引用；夹具口径声明 |
| `redline/RedlineFixture.java` | 626 | 独立夹具类（AutoCloseable）：内存自建两厂实例树（A 厂 FA/dept101→车间→设备；B 厂 FB/dept201→车间→设备；未绑部门产线 LINE99）+ 五账号（ADMIN scope=1、A 厂长 scope=2/dept101、B 厂长 scope=2/dept201、纯 scope=4、混合 scope=3+4/dept101）+ 数据项 300/400；真实组件链（DataPermServiceImpl → IotModelInstanceServiceImpl / IotDataItemServiceImpl + 0006 缓存链 Publisher→Listener→内存缓存）；Sa-Token 会话模拟照 0012 范式（静态 mock SecurityUtils/TenantContext，loginAs 切换账号）；持久层 mock 为内存"库"（单一数据源，UPDATE 写新行对象保快照语义）；列表/清单 mock 按 wrapper **真实生成的权限条件**求值（SQL 条件模拟器——主代码漏拼权限条件即全量放行、断言即红，不掩饰缺陷） |
| `redline/InstanceRedlineTest.java` | 331 | 管理面实例域八场景（每场景独立测试方法，先构造越权尝试再断言拒绝，详见 §4） |
| `redline/DataItemRedlineTest.java` | 99 | params 查询接口越权用例（effective-params 403 / item list 空清单 / overrideParams 写原语 403）+ 本厂正例对照 |
| `redline/ClientScopeRedlineTest.java` | 50 | 红线二-中台占位测试类（类级 `@Disabled("…待 TASK-0051 接线后转正")` + javadoc 登记接线义务与 3 条待启用用例语义） |

### 测试清单落地对照（brief 八场景 + 「每个查询接口必须有越权用例」）

| brief 场景 | 测试方法（InstanceRedlineTest 除注明外） | 断言要点 |
| --- | --- | --- |
| 红线一-A 跨厂查询 | `scenario1_redline1A_crossFactoryQuery_treeOmitsB_listWithoutB_detail403` | tree 不含 B 厂任何节点（含骨架）/ list 无 B 厂行且 wrapper 拼装 `id IN` 可见集 / B 厂厂级+设备级详情 403（行存在但越权，非 400 不存在口径）——**tree/list/detail 三查询接口越权用例各就位** |
| 红线一-B 写原语 | `scenario2_redline1B_crossFactoryWrites_putDeleteCreate_allForbidden` | PUT B 厂改绑到自己部门（升级链）403 且库内 dept_id 未变；DELETE B 厂叶子 403 且行未删；B 厂父下 create 403 |
| 未绑部门 fail-closed | `scenario3_unboundDept_failClosed_forNonAdmin_adminSeesIt` | 非 scope=1 用户（A 厂长）对 dept_id=NULL 产线：树/列表均不返回、详情 403；对照面 scope=1 管理员可见（裁决 6） |
| 仅本人语义（空集） | `scenario4_meOnly_pureScope4_treeAndListEmpty_withoutSelfCreatedRows` | 纯 scope=4 无自建行：树空集 + 列表空集（数据查询语义，裁决 3/4） |
| 仅本人语义（配置类回归） | `scenario5_meOnly_selfCreatedRow_entersListOnly_neverTree` | 自建实例仅入列表（配置类 create_by=me 分支）、永不入树（树不挂 create_by 分支，0012 修订轮登记口径） |
| 混合角色 scope=3+4 | `scenario6_mixedRole_deptSetVisible_listIncludesSelfCreated_treeExcludesIt` | 部门集合可见 + 列表=部门可见集 ∪ {create_by=me 自建}（自建行挂 B 厂车间下且 dept_id=NULL，仅 OR 分支可达）+ 树不含自建；wrapper 形态断言 `id IN`+`create_by`+` OR ` 并存（裁决 4 单 OR 分支回归固化） |
| 失效即时性（改绑） | `scenario7_rebindImmediateEffect_cacheEvicted_nextQueryReflectsNewBinding` | 端到端：updateInstance 改绑 B 厂→A 厂部门 → 缓存全量清空（粒度④）→ 不重登录不等 TTL 立即重查：B 厂长失去 B 厂节点（车间/设备锚点仍在）、A 厂长立即获得；roleMapper 二次触达=重查即重算证据；重查回填缓存 |
| 失效即时性（改角色） | `scenario8_roleChangeImmediateEffect_cacheEvicted_nextQueryNarrows` | 角色 scope 2→5（自定义仅绑 A 车间部门）→ 全量失效（粒度②）→ 立即重查收窄为 A 车间子树 |
| params 接口越权 | DataItemRedlineTest `params_bFactoryAccessed_byFactoryAAccount_deniedByShape` | effective-params（详情类）403、/iot/item/list（集合类）空清单、overrideParams（写原语）403；正例对照本厂清单/有效参数正常返回（排除夹具全空假绿） |

---

## 2. 每条验收命令原始输出

### 命令 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

**结果：BUILD SUCCESS，0 失败 0 错误。** 全量 `Tests run: 1550` = 基线 1537 + 新增 13（红线套件 10 执行 + 3 占位 skip）。

install 段尾：

```
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.211 s
[INFO] Finished at: 2026-11-22T08:10:59+08:00
```

Results 段（全量测试收尾原样）：

```
[INFO] Results:
[INFO]
[WARNING] Tests run: 1550, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  54.573 s
[INFO] Finished at: 2026-11-22T08:11:56+08:00
```

红线套件在全量套件中的段落（随标准 `mvn test` 进 CI 的实证；`cache evicted ALL` 为 0006 真实失效链在场景 7/8 触发的运行日志）：

```
[INFO] Running 红线二-中台 scope（占位——TASK-0051 接线后启用）
[WARNING] Tests run: 3, Failures: 0, Errors: 0, Skipped: 3, Time elapsed: 0 s -- in 红线二-中台 scope（占位——TASK-0051 接线后启用）
[INFO] Running net.otlab.admin.module.iot.redline.DataItemRedlineTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.023 s -- in net.otlab.admin.module.iot.redline.DataItemRedlineTest
[INFO] Running net.otlab.admin.module.iot.redline.InstanceRedlineTest
08:11:23.295 [main] INFO net.otlab.admin.module.iot.dataperm.DataPermCacheEvictListener -- [DATAPERM] visible-scope cache evicted ALL
08:11:23.341 [main] INFO net.otlab.admin.module.iot.dataperm.DataPermCacheEvictListener -- [DATAPERM] visible-scope cache evicted ALL
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.059 s -- in net.otlab.admin.module.iot.redline.InstanceRedlineTest
```

### 命令 2：`mvn test -pl ot-admin -Dtest='*RedlineTest'`（单独复跑）

**结果：BUILD SUCCESS，套件类数 3、用例数 13（10 执行全绿 + 3 占位 skip）。**

```
[INFO] Running 红线二-中台 scope（占位——TASK-0051 接线后启用）
[WARNING] Tests run: 3, Failures: 0, Errors: 0, Skipped: 3, Time elapsed: 0.009 s -- in 红线二-中台 scope（占位——TASK-0051 接线后启用）
[INFO] Running net.otlab.admin.module.iot.redline.DataItemRedlineTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.706 s -- in net.otlab.admin.module.iot.redline.DataItemRedlineTest
[INFO] Running net.otlab.admin.module.iot.redline.InstanceRedlineTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.176 s -- in net.otlab.admin.module.iot.redline.InstanceRedlineTest
[WARNING] Tests run: 13, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
[INFO] Total time:  7.996 s
```

### 命令 3：`git diff --name-only 1065f9a..HEAD`（改动范围核对）

**结果：5 个文件全部落在声明范围 `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/**` 内；无任何主代码 diff。**

```
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/ClientScopeRedlineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/DataItemRedlineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/InstanceRedlineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/RedlineFixture.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/package-info.java

5 files changed, 1121 insertions(+)
```

工作区余量核对：仅 `.agents/`（任务开始前已存在的未跟踪目录，与本任务无关，未触碰）。

---

## 3. 未做项与原因

| 未做项 | 原因 |
| --- | --- |
| 红线二-中台集成自动化用例本体 | 依赖 iot_api_client 表与 /openapi/v1 拉树接口（TASK-0051/0052 未交付）。按 brief 决策 2 以 `ClientScopeRedlineTest` 占位登记（类级 @Disabled + javadoc + 3 条待启用用例语义），组件级三态等价断言已由既有 `DataPermClientScopeTest` 固化。**验收口径按 brief 调整执行：红线一（管理面）+ 附加场景全绿 = 本任务验收；红线二标记「待 0051 接线后转正」，TASK-0051 brief 将带红线二集成用例义务。** |
| 独立 CI 配置文件 | 仓库现状无独立 CI 配置（已核实 ot-admin/父 pom 均无 surefire include/exclude 定制）。按 brief 决策 4，「CI 接入」实现为命名规范 `*RedlineTest` + surefire 默认包含——命令 1 全量套件中红线类随跑的日志即为随 CI 的实证。 |
| 上线前越权专项的实际执行 | 按 brief 决策 5 只做架构验收登记：本套件交付即载体（上线前专项执行归 REQ-0032 收口核对项：跑 `-Dtest='*RedlineTest'` 全绿 + 红线二转正后全量）。 |
| 既有测试文件的最小调整授权 | 未动用——夹具完全独立，无需复用既有测试内夹具。 |
| TC-0002 步骤 7「A 厂长查工厂 B 点位原始数据（TDengine 时序）」 | 点位原始数据/时序查询接口尚未交付（NoOp 占位），无被测面；params 类查询接口（effective-params/item list）已按「点位参数读取受控」口径覆盖，时序类按裁决 3 属空集语义，随接口交付任务（0039/0040 等）补挂。

## 4. 偏离计划之处

1. **夹具账号从三账号扩为五账号**：brief 决策 3 列三账号（A 厂长/B 厂长/纯 scope=4），但第 1 条场景清单要求「混合角色 scope=3+4」与改绑操作者（scope=1）——补 USER_MIX（14）与 ADMIN（10），均服务场景清单逐条结论，无新增口径。
2. **「仅本人语义」拆为两个测试方法**：brief 场景写「纯 scope=4 用户 instance 树/列表空集（数据查询语义，ADR-0016 裁决 3）」，而裁决 3 同时定版「实例属配置类对象，列表按 create_by=me 过滤」（0012 修订轮已按此实现并评审通过）。取两句并集固化：场景 4（无自建行）断言树/列表空集（brief 字面红线），场景 5（有自建行）断言仅入列表、永不入树（裁决 3/4 配置类语义回归）——两句同时全绿，互相不矛盾。
3. **standalone 内存口径声明**（brief 允许自选）：夹具不触真实库、不依赖 dev 库手工数据（自建数据随夹具创建销毁）；为让红线断言具有「查询层拦截」的证明力，列表/清单 mock 按 wrapper **真实生成的 SQL 权限条件**求值（条件模拟器）——若主代码漏拼权限条件，模拟器全量放行、断言即红，不会掩饰缺陷；树查询一次取全量后服务内存裁剪、详情/写路径均走真实代码，行为与真实库语义对齐。
4. **改角色即时性的挂点代表**：改绑链路（场景 7）走真实写服务端到端；改角色链路（场景 8）的 system 模块写点（updateDataScope 等）不在本夹具装配范围，以 TASK-0006 交付的同一发布组件入口（DataPermCacheEvictPublisher）代表该挂点，验证「失效→立即重查即生效」的缓存链与失效粒度②（全量）。
5. **夹具实现中排掉的两个环境坑**（备忘，非产品缺陷）：① MP 3.5.17 条件参数惰性物化——必须先渲染 `getSqlSegment()` 再读 `paramNameValuePairs`；② 内存"库"的 UPDATE 必须写新行对象（服务层持有的 existing 是取行快照），原地改对象会让主代码 `form.getDeptId().equals(existing.getDeptId())` 恒等、掩盖失效挂点（首跑 scenario7 即因此红，修正夹具后全绿——修正属夹具而非主代码）。
6. **红线一-A 树断言口径补充**：除「不含 B 厂节点」外，增加「B 厂节点连骨架路径也不出现」的全树 id 集断言与「树可见集=A 厂子树全集」精确断言（骨架语义归 0012 修订轮 M-7，本套件只锁行为不锁实现）。

## 5. 遗留问题

1. **红线二转正义务（已登记）**：TASK-0051 接线时启用 `ClientScopeRedlineTest`（摘除 @Disabled、按 javadoc 内 TODO 以 iot_api_client 夹具 + /openapi/v1 端到端实现 3 条用例）；转正前上线前专项的判定口径=红线一套件全绿。
2. ~~**观察项（非本任务缺陷，登记待裁决）**：纯 scope=4 用户列表可见 `create_by=me` 自建实例……~~ **【已闭环】**该观察项经 §13 diff 评审裁决确认为口径不对称缺陷，已在本报告文末「修订轮」节按授权修复（详情读路径补 create_by=me 分支）并完成补充验收，红线套件对应观察用例已转正为正式断言。
3. **CI 配置演进注意**：未来引入独立 CI 配置文件时，需保持 `*RedlineTest` 命名在 surefire 包含集内（redline 包 javadoc 已声明该契约）；上线前专项执行登记于 REQ-0032 收口核对。
4. **TASK 详情回写**：按 brief 契约未回写 TASK-0007 详情、未改状态（协调者收尾）；「git 提交 / 修改文件」可追溯信息已随本报告 §1/§2 留痕（提交 `5d5a6e7`）。

---

# 修订轮（2026-11-22，§13 验收退回——详情口径不对称缺陷修复）

## R.0 裁决与授权

- diff 评审：Critical 0 / Important 0（红线一全绿确认）；**观察项（纯 scope=4 用户列表可见自建实例但详情 403）裁决确认为口径不对称缺陷**，授权本修订轮修复。
- 裁决口径：详情可见性判定从「instanceIds.contains」扩为「instanceIds.contains || (scope.createByOnly() && 实例.create_by == 当前用户)」——与列表 OR 分支（ADR-0016 裁决 4）对齐：自建实例可见即可开（否则 create_by 分支在实践中不可用）；ADR-0016 裁决 3「仅本人适用集=iot_ 配置表的管理面查询」**含详情类**（留痕引用，ADR 正文未动）。
- 授权范围：instance 服务详情判定一处 + dataitem（params/detail 同口径需要）最小范围；允许一个 commit 注明 TASK-0007 修订轮；自起实例用毕即停。
- **create_by 语义确认**（裁决要求）：`MybatisPlusConfig.insertFill`（MetaObjectHandler）→ `strictInsertFill("createBy", …, getCurrentUserId())` → `StpUtil.getLoginIdAsLong()`——**create_by=创建者 userId**；dev 冒烟 S2 实测新建实例行 `createBy: 8`（登录用户 id）与语义一致；红线夹具断言行值（`assertEquals(USER_ME, own.getCreateBy())`）。

## R.1 改动文件清单（修订轮提交 `eb01ba2`，5 文件 +94/-9）

| 文件 | 性质 | 内容 |
| --- | --- | --- |
| `instance/service/impl/IotModelInstanceServiceImpl.java` | **主代码（授权一处）** | `getInstanceDetail` 可见性判定改走新私有谓词 `isInstanceReadableByUser(scope, instance)`：`instanceIds.contains || (createByOnly && create_by=当前用户)`；403/strategy 分支原样保留；写路径防线（requireDataPerm：update/delete/create-under）**维持 instanceIds 口径不变**（改绑升级链防线，红线一-B）——javadoc 留痕裁决 3 引用与写路径不动理由 |
| `dataitem/service/impl/IotDataItemServiceImpl.java` | **主代码（授权"同口径需要"）** | 两个读路径锚定同一实例可见性谓词：`selectItemsByInstanceId`（清单，集合类）与 `getEffectiveParams`（params 详情类）改走 `isInstanceReadableByUser`——自建实例可见即可开清单/参数（否则 create_by 分支在实践中不可用，与裁决理由同构）；`overrideParams`/`regenerateForInstance` 写防线维持 instanceIds 口径不动 |
| `redline/InstanceRedlineTest.java` | 测试 | scenario5 转正：自建实例详情 200（断言 createBy 行值=创建者 userId 语义）+ 他建实例（A 厂 F01，create_by=ADMIN）详情 403——「分支不放大越权面」双向断言 |
| `redline/DataItemRedlineTest.java` | 测试 | 新增 `params_ownCreatedInstance_byPureScope4User_visibleViaCreateByBranch`：纯 scope=4 自建实例（dept_id=NULL，仅 create_by 分支可达——最锐利对齐证明）清单返回 1 项、effective-params 200；他建实例清单仍空、params 仍 403 |
| `redline/RedlineFixture.java` | 测试 | 增 `addItemRow`/`newItemRow`（场景自建数据项构建入口，种子构建器复用） |

## R.2 补充验收命令原始输出

### 补充 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`（全量复跑，含新断言）

**结果：BUILD SUCCESS，0 失败 0 错误。全量 `Tests run: 1551` = 首轮 1550 + 1（DataItem 新用例）。**

```
[INFO] Results:
[INFO]
[WARNING] Tests run: 1551, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  56.553 s
[INFO] Finished at: 2026-11-22T08:27:16+08:00
```

（ot-core install 段 BUILD SUCCESS 无告警；日志 `backend/target/task0007r2_full_test.log`）

### 补充 2：`mvn test -pl ot-admin -Dtest='*RedlineTest'`（单独复跑）

**结果：BUILD SUCCESS，套件类数 3、用例数 14（11 执行全绿 + 3 占位 skip）。**

```
[INFO] Running 红线二-中台 scope（占位——TASK-0051 接线后启用）
[WARNING] Tests run: 3, Failures: 0, Errors: 0, Skipped: 3, Time elapsed: 0.011 s -- in 红线二-中台 scope（占位——TASK-0051 接线后启用）
[INFO] Running net.otlab.admin.module.iot.redline.DataItemRedlineTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.813 s -- in net.otlab.admin.module.iot.redline.DataItemRedlineTest
[INFO] Running net.otlab.admin.module.iot.redline.InstanceRedlineTest
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.167 s -- in net.otlab.admin.module.iot.redline.InstanceRedlineTest
[WARNING] Tests run: 14, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
[INFO] Total time:  8.566 s
```

### 补充 3：dev 冒烟复核（`mvn spring-boot:run -pl ot-admin`，日志 `backend/target/task0007r2_dev.log`）

启动前 8098 无 LISTENING；`Started OtAdminApplication in 9.283 seconds (process running for 9.957)`（08:22:48）。用毕即停：taskkill（PID 40036）→ 8098 复核仅 TIME_WAIT、无 LISTENING。全程 ERROR 8 条 = S5 负路径 403 业务异常留痕 1 条 + taskkill 的 maven 插件终止留痕，无意外错误。

冒烟夹具（SQL 预置）：用户 8 `smoke_me_t0007`（dept=4、tenant=1、密码哈希复用 0012 冒烟用户同款 BCrypt）+ `sys_user_role(8→4)`——**纯 scope=4**（角色 4 `smoke_inst_me_t0012r2` dataScope=4，菜单 4010/4030/4031/4032/4035 为 0012 修订轮既有授权，未改动该角色）。登录走 dev `/auth/login/test`（0012 范式）。

```
=== S1) POST /auth/login/test smoke_me_t0007 → 200 userId=8
{"code":200,"data":{"nickname":"smoke_me_t0007","token":"9eb1f602-…","userId":8,"username":"smoke_me_t0007"},…}
=== S2) POST /iot/instance ME07（顶级产线 nodeType=3，无 deptId——纯 create_by 锚点）→ 200
{"code":200,…}   DB：id=18, instance_code=ME07, dept_id=NULL, create_by=8  ← create_by=创建者 userId 实证
=== S3) GET /iot/instance/list → 200 total=1
  id=18 ME07 createBy=8   ← create_by=me 分支（既有列表口径）
=== S4) GET /iot/instance/18（自建 → 期望 200，修订轮修复点）→ 200
{"code":200,"data":{"id":18,…,"instanceCode":"ME07","createBy":8,…},…}
=== S5) GET /iot/instance/2（他建 F01，create_by=1 → 期望 403）→ 403
{"code":403,"data":null,"message":"无权访问该实例（数据权限不命中）"}
=== S6) GET /iot/instance/tree → 200 空树（树不挂 create_by 分支语义不变）
{"code":200,"data":[],…}
```

判定：**通过**（裁决冒烟序列全覆盖：列表见自建 / 自建详情 200 / 他建详情 403；树语义回归不变佐证）。

**用毕即停与清理**：dev 服务已停（8098 无 LISTENING）；冒烟数据已清理——`iot_model_instance id=18`、`sys_user id=8`、`sys_user_role (8,4)` 三行 DELETE 后复查计数均 0（硬删，防 ME07 编码被逻辑删行永久占用）；角色 4 及其菜单授权为 0012 既有夹具未动；`sys_login_log`/`sys_oper_log` 中 user 8 的审计留痕保留（真实事件日志，不回删）。环境插曲：启动首试因 WSL VM 回收瞬窗 Redis 6379 拒连失败一次（0012 报告已登记的同款环境问题），按其 keepalive 处置（`wsl sleep 14400` 后台）后重启即稳定。

## R.3 修订轮未做项与偏离

1. **dataitem 清单端点一并改口径**（超出裁决字面"详情"、属授权内"params/detail 同口径需要"）：裁决理由「自建实例可见即可开（否则 create_by 分支在实践中不可用）」同样约束 item list——若只改 effective-params 而清单仍空，用户无法发现 itemId，分支依旧不可用；两读路径共用同一实例可见性谓词，口径一致。写防线（overrideParams/regenerate）明确不动。
2. **写路径不加 create_by 分支**（口径边界留痕）：裁决只授权详情读路径；写原语借 create_by 放行会重开「改绑 dept_id 反向扩大可见集」升级链面（红线一-B 防线语义），如产品需要「仅本人可编辑自建实例」另行走裁决。
3. **树语义不变**：树仍不挂 create_by 分支（0012 修订轮登记口径，S6 冒烟 + 红线场景 5 双重固化），未受本轮影响。

## R.4 修订轮遗留

1. ADR-0016 正文未动（裁决 3 的「含详情类」以本报告与代码 javadoc 留痕引用）；若协调者认为需在 ADR 正文补一行修订记录，归协调者收尾。
2. 红线二-中台占位（待 0051 转正）状态不变；上线前越权专项口径=红线套件全绿（现 14 用例）。
3. dev 库 0012/0014/0006 等历史冒烟留痕数据（角色 3-7、用户 3-7、实例行等）沿用首轮登记，是否清理由协调者统一裁决。
