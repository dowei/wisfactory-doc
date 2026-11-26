# 实施报告：TASK-0014（实例参数覆盖与阈值继承链）

- 执行者：implementer（brief：reports/TASK-0014-brief.md）
- BASE：`4d7246d`（主仓 HEAD）→ HEAD：`7fa7cb9`（main，1 个提交，注明 TASK-0014）
- 总体结论：**4 条验收全部通过**（全量回归 1480 = 基线 1449 + 新增 31，0 失败 0 错误；dev 冒烟全步通过——覆盖/清除/继承链三态/兄弟隔离/min>max 400/受限用户 403 与可见集正向对照、时间线钩子 NoOp 运行期留痕；prod 档真实启动 0 ERROR + 真实登录 200 + effective-params 200 + 覆盖写 200；改动范围 13 文件全落声明内）
- TDD：红 21（占位桩 behavioral 失败）→ 实现 → 绿 41/41；实现自首次落笔后未再改动，红绿之间三轮均为测试侧修正（MP wrapper 惰性参数渲染的断言方式），详见 §2.1
- 修订轮（2026-11-22）：评审 Critical 0 / Important 1 / Minor 6，必修 I-1（partial 两步绕过配对校验）已修完——配对校验扩为「结果行四列」口径，红 2 → 绿，全量复跑 1483 绿（见文末「修订轮」一节）；Minor 6 条全 deferred 登记

---

## 1. 改动文件清单（13 个，与验收 4 的 git diff 一致）

**新增（6 个 main + 2 个 test）**

| 文件 | 内容 |
| --- | --- |
| `backend/.../dataitem/dto/IotItemParamOverrideForm.java` | 覆盖表单（body 仅四字段；partial 逐字段 null=清除/数值=覆盖；校验在服务层 fail-closed，不挂 bean validation——null 合法语义表达不了） |
| `backend/.../dataitem/dto/IotItemEffectiveParamsVO.java` | 有效参数 VO（逐字段值 + 来源三态，INSTANCE/TEMPLATE/NONE） |
| `backend/.../dataitem/dto/IotParamSource.java` | 继承来源三态枚举（javadoc 登记运行时继承语义与 ADR-0008 二档关联） |
| `backend/.../dataitem/service/IotDataItemParamChange.java` | 时间线钩子载荷 record（itemId/fullKey/operatorId/before/after 四阈值快照；JSON 序列化归 0039 实现方，不预设 JSON 形态） |
| `backend/.../dataitem/service/IotDataItemHistoryService.java` | 时间线钩子接口（`recordParamChange` 占位；javadoc 声明 change_type=PARAM 语义、同事务契约、接线点、0039 零改动接线承诺） |
| `backend/.../dataitem/service/NoOpIotDataItemHistoryService.java` | NoOp 实现（@Service，debug 留痕证明钩子被调用；0039 交付真实实现后整体移除） |
| `backend/.../dataitem/service/impl/IotDataItemServiceImplParamOverrideTest.java` | 服务层 21 用例（四列语义/partial/清除/兄弟隔离/钩子载荷/五类校验拒/存在性/归属/403/三态混合/模板缺失回退） |
| `backend/.../dataitem/service/NoOpIotDataItemHistoryServiceTest.java` | NoOp 契约 2 用例（静默通过 + Spring bean 装配契约） |

**修改（3 个 main + 2 个 test，均在 dataitem 域与 iot 测试声明内）**

| 文件 | 内容 |
| --- | --- |
| `backend/.../dataitem/controller/IotDataItemController.java` | +2 接口：PUT `/iot/instance/{instanceId}/items/{itemId}/params`（iot:item:edit，4039 复用）+ GET `.../effective-params`（iot:item:query，4037 复用）；@OperateLog UPDATE；类 javadoc 更新四接口/权限全景 |
| `backend/.../dataitem/service/IotDataItemService.java` | +2 方法契约（overrideParams/getEffectiveParams 全 javadoc：仅写四列/运行时继承/防线顺序/钩子同事务） |
| `backend/.../dataitem/service/impl/IotDataItemServiceImpl.java` | +实现：overrideParams（存在性→数据权限 403→归属→校验→LambdaUpdateWrapper 显式 set 四列含 NULL→同事务钩子）、getEffectiveParams（存在性→403→归属→模板同名 default 逐字段解析三态）；+常量 decimal(20,6) 口径（scale≤6/整数位≤14）；+requireDataPerm 私有助手（照 0012 修订轮 I-1 同款）；类 javadoc 增 0014 段 |
| `backend/.../dataitem/controller/IotDataItemControllerTest.java` | 7→15 用例（+权限注解两锚点/@OperateLog UPDATE/PUT 成功与 {} 清除/400 403 透传/GET 三态断言/403 透传） |
| `backend/.../dataitem/config/IotDataItemMenuPermissionGuardTest.java` | 锚点扩展：覆盖=iot:item:edit、继承链=iot:item:query（复用既有行不新增）；子集断言下限 2→4（remove 仍预留） |

未触碰：frontend/、ingest/、模板域（只读 default_ 四列）、instance 域（只读）、dataperm（只调 resolve）、resources/sql（无新表无菜单种子——复用 4037/4039 权限点，brief 预告一致）、workitems 库其他文件；TASK 状态保持「进行中」未动。

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0（-q 静默）。ot-admin 全量：

```
[INFO] Tests run: 1480, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  58.825 s
```

1480 = 基线 1449（0013 修订轮）+ 新增 31（服务层 21 + NoOp 2 + 控制器 +8；守卫测试数不变 3）。0 失败 0 错误。

**TDD 红转绿记录**（四轮，实现只在第 2 轮前落笔一次）：

1. 第 1 轮红（占位桩 `UnsupportedOperationException`，`-Dtest=IotDataItemServiceImplParamOverrideTest,IotDataItemControllerTest,NoOpIotDataItemHistoryServiceTest,IotDataItemMenuPermissionGuardTest`）：
   ```
   [ERROR] Tests run: 41, Failures: 9, Errors: 12, Skipped: 0
   ```
   红 21 = 服务层测试类全部 21 个用例（含 testEffective_* 5 个与 testOverride_* 16 个），均为被测行为尚未实现——恰为 brief 决策 6 六项验收的行为面。
2. 第 2 轮（实现落笔后）：`Tests run: 41, Failures: 3, Errors: 1`——3 处失败/1 处错误全部出在**测试侧假设**：MP 3.5.x 的 `formatParam` 为惰性求值（渲染 SQL 段时才写 `paramNameValuePairs`），测试先断言参数表再渲染导致键序/数量假设错误；另有 1 处漏包 `assertThrows`（BusinessException 直接冒出）。修复：测试助手改为「先渲染 getSqlSet()/getSqlSegment()，再从段文本按列名/键名解析参数值」（不依赖参数表迭代序）。**为什么现在对了**：断言对象从「MP 内部参数表布局」改为「最终渲染出的 SQL 段文本」，与真实落库 SQL 同源，键名契约变化时会响亮失败（预期守护）。
3. 第 3 轮：`Tests run: 41, Failures: 2`——setParamValue 键提取带了尾缀 `}`（段文本 `col=#{ew.paramNameValuePairs.MPGENVAL1}` 的 substring 越界），改正则提取。
4. 第 4 轮绿：
   ```
   [INFO] Tests run: 41, Failures: 0, Errors: 0, Skipped: 0
   [INFO] BUILD SUCCESS
   ```
   实现类自首次落笔后零改动——四轮红绿全部由测试侧断言方式收敛，被测行为一次写对。

### 2.2 验收 2：dev 冒烟（`cd backend && mvn spring-boot:run -pl ot-admin`，日志 backend/target/task0014_dev.log）

启动前端口探测：8098 无 LISTENING。启动：`Started OtAdminApplication in 10.349 seconds (process running for 10.91)`；启动段 ERROR 计数 = **0**（全程 9 条 ERROR 均为负路径故意触发的业务异常留痕：租户码笔误 1 + min>max/负值/归属错/超小数位/超整数位各 1 + 挂载禁用父实例（夹具改道留痕）1 + 403 两形态 2，无意外错误）。用毕即停：taskkill（PID 5312）→ 8098 复核无 LISTENING。

夹具前置：模板 TPL14EQ（id=9，设备级，1 项 ELEC_CONS 直绑，default 阈值 0/100000/1/99）经模板域 API 建立（字典码 ELEC_CONS 复用 0013 登记）；实例 EQ14A（id=13）/EQ14B（id=14）兄弟同挂 F01.WS13（id=8）、同选模板 9——生成链拷贝默认值入实例行（0013 语义，两行四列均 0/100000/1/99）。

```
=== S1) admin 登录（dev /auth/login/test，tenantCode=default）===
{"code":200,"data":{"nickname":"管理员","token":"760aebc9-...","userId":1,"username":"admin"},"message":"操作成功"}
=== S2a) POST /iot/template TPL14EQ (nodeType=4) → {"code":200,...}
=== S2b) POST /iot/template/item（ELEC_CONS，default 阈值 0/100000/1/99）→ {"code":200,"data":1,...}
=== S3a) POST /iot/instance EQ14A（shortCode，父=8，模板=9）→ 200；DB: id=13 F01.WS13.EQ14A
=== S3b) POST /iot/instance EQ14B（兄弟同模板）→ 200；DB: id=14 F01.WS13.EQ14B
   DB 复核：item id=13/14 生成拷贝默认值（valid 0.000000/100000.000000，alarm 1.000000/99.000000）

=== S4)  GET /iot/instance/13/items/13/effective-params（生成拷贝态→全 INSTANCE）=== {"code":200,...全四字段 INSTANCE，0/100000/1/99}
=== S4b) GET .../14/items/14/effective-params（兄弟基线快照）=== {"code":200,...全四字段 INSTANCE，0/100000/1/99}

=== S5)  PUT /iot/instance/13/items/13/params 覆盖 valid 对 + 清除 alarm 对（partial）===
{"validMin":5,"validMax":200,"alarmLow":null,"alarmHigh":null} → {"code":200,...}
=== S6)  GET effective-params EQ14A → 混合三态实况：
  validMin=5.000000/INSTANCE、validMax=200.000000/INSTANCE（覆盖值）
  alarmLow=1.000000/TEMPLATE、alarmHigh=99.000000/TEMPLATE（清除后回归模板默认）
=== S6b) GET effective-params EQ14B（兄弟实例同码项）→ 与 S4b 逐字节一致
  （0/100000/1/99 全 INSTANCE）——覆盖仅本实例，兄弟不受影响 ★REQ-0005 验收 5 前半

=== S7)  PUT 全 null=清除覆盖（合法）→ {"code":200,...}
=== S7b) GET effective-params EQ14A → 回归 TEMPLATE 来源：0/100000/1/99 全 TEMPLATE ★
=== S8)  PUT {"validMin":101,"validMax":100} → {"code":400,"message":"有效范围下限不能大于上限（valid_min > valid_max）"}
=== S8b) PUT {"validMin":-1} → {"code":400,"message":"有效范围下限 valid_min不允许负值"}

=== S9a) POST /role 受限角色 smoke item role T14（dataScope=2 本部门及下属，menuIds=[4037,4039]）→ 200（DB: id=5, tenant_id=1）
=== S9b) POST /user smoke_item_t0014（dept=4 车间部门，密码 SM4 密文提交，jshell ROUNDTRIP=Smoke@2026）→ 200（DB: id=5）
=== S9c) PUT /user/5/role {roleIds:[5]} → 200；DB: role_menu (5,4037)+(5,4039)，user_role (5,5)
=== S9d) 受限用户 /auth/login/test 登录 → 200（userId=5）

=== S10) 受限用户 PUT /iot/instance/13/items/13/params → {"code":200}
  ★正向对照：EQ14A 挂 F01.WS13（dept_id=4=受限用户本部门），经组件步骤 3-4 子树展开 ∈ 可见集——
   菜单权限（4039）过 + 数据权限命中 → 放行；随 PUT 清除恢复原态（S10d 200）。行为符合模型预期。
=== S11) admin 建不可见目标链：POST /iot/instance WS14（父=F02 id=5，dept=2）→ 200（DB: id=15 F02.WS14）
=== S11b) POST /iot/instance EQ14C（父=15，模板=9）→ 200（DB: item id=15，四列=生成拷贝值）

=== S12) 受限用户 PUT /iot/instance/16/items/15/params（不可见实例项）→ 403：
  {"code":403,"message":"无权操作该实例（数据权限不命中）"}——文案不含实例编码/id ★
=== S12b) 受限用户 GET .../16/items/15/effective-params → 403：
  {"code":403,"message":"无权访问该实例（数据权限不命中）"}
=== S12c) admin DB 复核 item id=15 四列仍为 0/100000/1/99（403 拒绝零写入）

=== S13) admin PUT /iot/instance/13/items/14/params（item 归属他实例）→ 400「数据项不存在或不属于该实例: 14」
=== S14) PUT {"validMin":0.0000001} → 400「…小数位超限（decimal(20,6) 上限 6 位小数）」
=== S14b) PUT {"validMin":100000000000000} → 400「…整数位超限（decimal(20,6) 上限 14 位整数）」
=== S14c) PUT {"validMin":9999999999.999999}（上界内）→ 200（边界合法）
=== S15) 清尾：清除 EQ14A 覆盖 → GET effective-params 全 TEMPLATE（库内 item 13 四列=NULL 复核一致）
```

**时间线钩子运行期留痕**（dev 日志 DEBUG，覆盖与清除各一条）：

```
DEBUG n.o.a.m.i.d.service.NoOpIotDataItemHistoryService - [ITEM-HISTORY] NoOp recordParamChange: itemId=13, fullKey=F01.WS13.EQ14A.ELEC_CONS（时间线表归 TASK-0039，本期不落库）
INFO  n.o.a.m.i.d.service.impl.IotDataItemServiceImpl - 实例参数覆盖完成: instanceId=13, itemId=13, fullKey=F01.WS13.EQ14A.ELEC_CONS, valid=[5,200], alarm=[null,null]
```

DB 审计复核（S15 后）：item id=13 四列全 NULL、update_by=1、update_time 非空（wrapper+空实体的 updateFill 生效）；item id=14 全程未动。

### 2.3 验收 3：prod 档真实启动回归

打包：`mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（Total time 33.754 s；全量测试已在验收 1 同一代码态覆盖）。

启动（沿 TASK-0012/0013 已确认命令 + 三密钥环境变量）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
启动前端口探测：8098 无 LISTENING
Started OtAdminApplication in 17.429 seconds (process running for 18.22)
=== prod 启动段 ERROR 计数（Started 之前） ===
0
=== prod 全程 ERROR 计数 ===
0
=== prod 真实登录（Redis 预置 captcha:code:task0014a="7K2M"（Jackson 序列化带引号，EX 300）+ SM4 hex 密文 base64 携带，jshell ROUNDTRIP=admin123） ===
{"code":200,"data":{"nickname":"管理员","token":"8b2ee481-...","userId":1,"username":"admin"},"message":"操作成功"}
=== prod GET /iot/instance/13/items/13/effective-params ===
HTTP_STATUS:200（body code=200：validMin=0/TEMPLATE、validMax=100000/TEMPLATE、alarmLow=1/TEMPLATE、alarmHigh=99/TEMPLATE——dev 清尾后回归态在位）
=== prod PUT /iot/instance/13/items/13/params（valid 对覆盖 5/200 + alarm 清除） ===
HTTP_STATUS:200（body code=200）
=== prod GET effective-params 复核 ===
HTTP_STATUS:200（validMin=5/INSTANCE、validMax=200/INSTANCE、alarmLow=1/TEMPLATE、alarmHigh=99/TEMPLATE——三态与覆盖值正确）
=== prod 清尾：PUT 全 null 清除 → 200，effective-params 复核全 TEMPLATE ===
```

用毕即停：prod 实例 taskkill（PID 21156）→ 8098 复核无 LISTENING。判定：**通过**（启动段与全程 0 ERROR + 真实登录 200 + effective-params 200 + 覆盖写读全链 200）。

### 2.4 验收 4：改动范围核对 `git diff --name-only 4d7246d..HEAD`

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemEffectiveParamsVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemParamOverrideForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotParamSource.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotDataItemHistoryService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotDataItemParamChange.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotDataItemService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/NoOpIotDataItemHistoryService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/config/IotDataItemMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/NoOpIotDataItemHistoryServiceTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImplParamOverrideTest.java
```

核对结论：13 文件 = `dataitem/**` main 9 个（声明内）+ `iot/**` 测试 4 个（声明内）；无 frontend/、无 ingest/、无 instance/模板/dataperm 域改动、无 resources/sql（无新表无菜单种子）。提交：`7fa7cb9`（main，注明 TASK-0014）。**通过**。

## 3. 未做项与原因

1. **时间线真实落库未做**（brief 明文口径）：iot_data_item_history 表与真实实现归 TASK-0039。本期交付接口 `IotDataItemHistoryService` + `NoOpIotDataItemHistoryService` 占位 + javadoc 接线点（`IotDataItemServiceImpl.overrideParams` 同事务调用）+ 遗留登记；0039 交付后移除 NoOp、补真实实现即零改动接线（接口签名即合同）。
2. **状态规则门限覆盖未做**（brief 明文）：本期覆盖接口只处理 valid/alarm 四字段；状态规则门限载体归 FEAT-0016/模板配置，规则行不复制。
3. **批量覆盖不做**（brief 明文）：一期单项（单实例单数据项）。
4. **无新表无菜单种子**（brief 预告一致）：iot:item:edit（4039）/iot:item:query（4037）复用 0013 既有菜单行，守卫测试已把两接口锚点锁进五值集合（防漂移双向）。
5. **锁定实例的参数覆盖未设专门防线**（brief 未要求，登记待裁决）：REQ-0005 锁定口径约束「实例编辑仅允许 status/remark」，未明文覆盖数据项阈值覆盖动作。当前 locked=1 实例的数据项仍可覆盖（锁的是实例行字段）。若产品要求锁定连带冻结数据项配置，改动点为 overrideParams 增一道 locked 守卫——请协调者裁决后另行安排。

## 4. 偏离计划之处

1. **受限用户 403 冒烟的夹具改道**（brief 序列内步骤的执行路径修正，非实现偏离）：按「0012 夹具范式」建受限用户（dept=4）后，0013 冒烟建立的 F01.WS13（dept_id=4）使 EQ14A/EQ14B 经组件子树展开落入受限用户可见集——首轮对 EQ14A 的覆盖返回 200 而非 403（数据权限模型正确行为：可见即放行，留作正向对照 S10）。403 取证改在新建的不可见链 F02.WS14（dept=2）→ EQ14C 上完成（S11-S12c，PUT/GET 双 403 + 零写入复核）。中途一次 PUT 尝试挂在已禁用的 F01.WS02 下被 400 拒（0012 冒烟禁用态），亦为改道留痕。
2. **无**其他偏离：接口路径/权限/校验/钩子/继承语义均按 brief 契约逐条落地；「报告写到 reports/TASK-0014-report.md」「不回写 TASK 详情」照办。

## 5. 遗留问题

1. **dev/prod 库冒烟留痕**（是否清理由协调者裁决）：模板 TPL14EQ(id=9) 及其模板数据项；实例 F02.WS14(id=15)、F01.WS13.EQ14A(id=13)/EQ14B(id=14)/F02.WS14.EQ14C(id=16) 及实例数据项 4 行（item id=13 四列 NULL、id=14/15 生成拷贝值、id=16 同）；角色 smoke item role T14(id=5)、用户 smoke_item_t0014(id=5)（dept=4）；部门/字典复用既有无新增。同码重跑会撞占用（预期）。
2. **锁定与覆盖的边界**（§3.5）：locked 实例的数据项覆盖当前放行，待协调者裁决是否增防线。
3. **NoOp 移除时序**：0039 交付真实实现时需移除 `NoOpIotDataItemHistoryService`（唯一 Spring 装配冲突点）并保证 `recordParamChange` 与调用方同事务（REQ-0018 验收 1「保存失败则无记录」的实证归 0039）。
4. **存在性 vs 权限的探测差异**（与 0013 修订轮 deferred M-4 同口径）：不存在实例返回 400、存在但不可见返回 403——受限用户可借此探测 id 存在性；文案均不泄露实例标识。如需收敛（先权限后存在性），改动点在 overrideParams/getEffectiveParams 首两段顺序，协调者裁决后可作修订轮小项。
5. **环境**：WSL docker 容器（wisfactory-mysql/redis）全程存活未复启；Redis captcha 种子 task0014a 已 EX 300 自动过期；8098 全程纪律（起前探测无 LISTENING、用毕 taskkill 复核无 LISTENING）两轮均执行。
6. **TC-0004 联测项**：步骤 5「实例覆盖告警阈值 → 其他同模板实例不受影响、时间线新增 PARAM 记录」在本任务 dev 冒烟以 EQ14A/EQ14B 实测（S5/S6b 隔离 + 钩子 DEBUG PARAM 留痕）；时间线可查形态归 0039。

---

## 修订轮（2026-11-22，验收退回一轮 §13：评审 Critical 0 / Important 1 / Minor 6）

- 评审报告：reports/TASK-0014-review.md；协调者裁决必修 I-1（partial 覆盖两步绕过 min&lt;=max 配对校验），按评审建议方向修：`validateThresholds` 配对校验扩为「**结果行四列**」口径。
- 修订提交：`ef7a84d`（单提交，注明 TASK-0014 修订轮），改动 3 文件均落 dataitem 域声明内：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotDataItemService.java          （overrideParams javadoc 契约同步）
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java （validateThresholds 扩参+配对解析；类 javadoc 一句）
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImplParamOverrideTest.java （+3 用例、1 用例边界取证调整）
```

### 修订内容（validateThresholds 一处修完）

1. **I-1 配对校验「结果行四列」口径**：配对输入=提交值优先、**缺省列取当前行已有值**参与配对（form 与行内均空 → 该列结果 NULL 不参与比较）；valid 对与 alarm 对各自成立，命中即 400「阈值配对非法：结果行 valid_min > valid_max（本次提交缺省列取行内现值参与配对）」（alarm 对同构文案）。封堵两步序列：先 PUT {validMin:5000}（200）→ 再 PUT {validMax:100}，第二步按行内现值 5000 配对即拒。**与模板默认的配对关系仍归计算引擎运行期判定——该部分口径维持不变**（裁决明文）。写路径四列 clear 语义、接口签名、钩子载荷均未动；per-field 负值/decimal(20,6) 校验只看提交值（行内现值历史上均过校验或来自生成拷贝），不变。
2. **测试**：新增 3 用例——两步绕过 valid 对拒 / 两步绕过 alarm 对拒 / 行内合法对 (0,100000) 上 partial 覆盖另一侧仍合法 → 200 不误伤；1 用例取证调整——decimal(20,6) 整数位上界合法值改按整对提交（修订后单提交 validMin 边界值会与行内 valid_max=5000 配对比较，原单值提交形态不再适用，注释已登记原因）。

### 红转绿记录（修订轮测试先行）

新 3 用例在**修订前实现**（HEAD=7fa7cb9）上运行 → **红 2**（恰为两步绕过被测行为——修复前第二步 200）：

```
[ERROR] Tests run: 24, Failures: 2, Errors: 0, Skipped: 0
[ERROR]   IotDataItemServiceImplParamOverrideTest.testOverride_TwoStepBypassRejectedAlarmPair:520 Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
[ERROR]   IotDataItemServiceImplParamOverrideTest.testOverride_TwoStepBypassRejectedValidPair:504 Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
```

应用修订实现后同类全绿（不误伤用例与边界调整用例两轮均绿——纯回归守护）：

```
[INFO] Tests run: 24, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

### 补充验收 1：全量复跑 `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
ot-core install 退出码 0（-q 静默）
[INFO] Tests run: 1483, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  58.128 s
```

1483 = 1480（首轮）+ 3（修订轮红转绿）。0 失败 0 错误。

### 补充验收 2：dev 冒烟复核（日志 backend/target/task0014_rev_dev.log）

启动前端口探测：8098 无 LISTENING。启动：`Started OtAdminApplication in 9.336 seconds (process running for 9.89)`；启动段 ERROR=0（全程 8 条 ERROR = 1 条 R2 故意负路径业务留痕 + 7 条用毕即停 taskkill 的 maven 插件留痕，无意外错误）。夹具沿用首轮：item 13（EQ14A，修订前 S15 清尾态四列 NULL）。用毕即停：taskkill（PID 61144）→ 8098 复核无 LISTENING。

```
=== R1) PUT /iot/instance/13/items/13/params {"validMin":5000}（两步绕过第一步）=== {"code":200,...}
=== R2) PUT {"validMax":100}（两步绕过第二步：缺省列取行内现值 5000 参与配对）→ 400 ★修订行为：
  {"code":400,"message":"阈值配对非法：结果行 valid_min > valid_max（本次提交缺省列取行内现值参与配对）"}
=== R2b) DB 复核零写入：item 13 行内仍 (5000, null, null, null)
=== R3) PUT {"validMin":5,"validMax":200}（整对合法化）→ {"code":200,...}
=== R4) PUT {"validMax":100}（合法 partial：行内 (5,200) → 配对 (5,100) 合法）→ {"code":200,...} ★不误伤
=== R5) GET effective-params → effective 正确（clear 语义运行时继承）：
  validMin=0.000000/TEMPLATE（行内被清→继承模板默认）、validMax=100.000000/INSTANCE、alarmLow=1/TEMPLATE、alarmHigh=99/TEMPLATE
=== R6) 清尾：PUT 全 null → 200；DB 复核 item 13 回到 (null×4)（与修订前冒烟终态一致）
```

判定：**通过**（裁决两步全覆盖：绕过序列第二步 400 + 零写入取证；合法 partial 200 且 effective-params 正确）。

### 修订轮 Minor 处置登记（6 条全 deferred，协调者裁决不动）

- **M-1** decimal(20,6) 上界示例写错（注释级三处，校验逻辑本身正确；真上界 14 位整数+6 位小数=99999999999999.999999，代码中「9999999999.999999」示例误导后人）——deferred，本修订轮未顺手改（裁决「Minor 6 条不动」）。
- **M-2** 并发覆盖无版本守卫 last-write-wins；行 select 与 update 之间被删则 0 行更新仍 200 且钩子照记；钩子 before 快照竞态下可能过期——iot 域全模块无 @Version 属既有惯例，配置类数据可接受；若 0039 时间线要求精确 before，需届时一并裁决。
- **M-3** requireDataPerm 与 403 文案跨域字面复制（0012 实例域 vs 本域）——公共助手提取另行微改，不入本任务。
- **M-4** 钩子载荷缺 instanceId（仅 itemId/fullKey/operatorId/before/after）——0039 派发 brief 时须显式确认载荷形态是否终版（与「零改动接线」承诺的张力点）。
- **M-5** 存在性 400 vs 不可见 403 的 id 存在性探测差异（首轮报告遗留 4，与 0013 deferred M-4 同口径）——维持 deferred。
- **M-6** locked=1 实例的参数覆盖放行（首轮报告遗留 2 待裁决）——维持 deferred。

### 修订轮推论边界登记（口径推论，非遗漏）

「缺省列取行内现值参与配对」的一个推论：**行内若遗留非法对，清除类提交（含全 null 清除）也会被 400 拒**（缺省列取行内现值参与比较命中）。逃逸路径明确：提交一对合法整值（如 PUT {validMin:X, validMax:Y} 且 X≤Y）即可整对覆写合法化。行内非法对在 API 面不可再造（同提交配对被拒、生成拷贝源模板默认合法），实际可达性低；如协调者希望「清除动作恒放行」，改动点为 validateThresholds 对全 null 提交跳过配对比较，另行裁决。

### 修订轮环境留痕

WSL VM idle 回收致容器重启（0011/0012/0013 报告同类现象）：首轮 dev 启动尝试（06:51）因 Redis 未就绪失败一次（Application run failed: RedisConnectionException，日志被二次启动覆盖）；按同款处置改用前台保持 keepalive 会话（`wsl sleep 14400` 后台任务，4h）后重启 dev 实例，启动干净（9.336s，启动段 0 ERROR）。修订轮冒烟期间容器稳定；用毕即停已执行（8098 复核无 LISTENING）。
