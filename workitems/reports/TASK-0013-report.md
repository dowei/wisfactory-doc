# 实施报告：TASK-0013（实例创建批量生成数据项与相对引用展开）

- 执行者：implementer（brief：reports/TASK-0013-brief.md）
- BASE：`46e923d` → HEAD：`fb13384`（main，1 个提交，注明 TASK-0013）；修订轮 HEAD：`4d7246d`（第 2 个提交，注明 TASK-0013 修订轮）
- 总体结论：**5 条验收全部通过**（全量回归 1445 = 基线 1400 + 新增 45，0 失败 0 错误；升级脚本两次执行均 DONE 且幂等生效、两表 + uk 双唯一 + 菜单 4036-4040 全核对；dev 冒烟全步通过——SELF/PARENT 展开落库实况、重生成三态幂等、删除保护拒/放两形态；prod 档启动段 0 ERROR + item/list 200；改动范围 17 文件全落在声明内）
- 修订轮（2026-11-22）：评审 Critical 0 / Important 2 / Minor 5，必修四项全部修完（见文末「修订轮」一节），全量复跑 1449 绿 + dev 冒烟复核通过

---

## 0. 核对 PDF 的关键口径（先于实现提取，非猜测）

用 pypdf 对《工厂数据采集系统详细设计 V1.1》提取 p13-14（§1.6 字段表）、p32（附录 A DDL）、p15（§1.7 ref 表）、p20-22（错误码/接口），并对产品设计 V2.2 提取 §5.2（p9-10）：

1. **相对引用语法**：详细设计 §1.6 正文未另立语法（全文提取无 SELF/PARENT 字样）；实际语法在产品设计 V2.2 §5.2 p9——花括号包裹 `{SELF.POWER_A}`、`DELTA({SELF.ELEC_METER})`，跨设备绝对引用同用花括号 `{F01.WS01.EQ002.ELEC_CONS}`。展开目标形态为**裸键**（详细设计 §3.1 p22 公式试算示例 `DELTA(F01.WS01.EQ001.ELEC_CONS, 3600) / 3600` 无花括号；TC-0004 步骤 3「{SELF.ELEC_METER} 展开为 F01.WS01.EQ001.ELEC_METER」同口径）。实现照此：`{SELF.X}`→`<instance_code>.X`、`{PARENT.X}`→`<parent_instance_code>.X`、绝对引用剥壳原样；关键字大小写不敏感。
2. **calc_period 换算**：模板 default_calc_period 为 varchar(16) 周期码（如 MINUTE/HOUR/DAY，ADR-0011 补字段），实例 calc_period 为 int 秒、0=到达驱动（p14 字段表明文）。映射落地：null/空白→0；纯数字串→按秒直取；SECOND→1/MINUTE→60/HOUR→3600/DAY→86400（大小写不敏感）；未知码→**整批失败**（不静默折 0，防「配了周期被当成到达驱动」的语义漂移）。
3. **value_kind 留模板**：实例表无此列（p14 字段表确认），brief 明文不落实例。
4. **DDL 逐字**：iot_data_item / iot_data_item_ref 照 p32 逐字（tenant_id 两表本就有，无「与上游差异」加列）；唯一双 uk + idx_tenant_point/idx_tenant_instance、uk_item_ref + idx_ref_key 全部在位（§2.2 SHOW INDEX 实测）。
5. **ref 表写入规则**（p15）：「公式校验通过并保存时，事务内先删该 item_id 旧边再批量插新边」——生成链只产新项（无旧边），直接批量落边；ref_type=1 数据项、ref_type=2 点位（点位边随绑定动作，本期不产）。

## 1. 改动文件清单（17 个，与验收 5 的 git diff 一致）

**新增（11）**

| 文件 | 内容 |
| --- | --- |
| `resources/sql/upgrade/20261122_iot_data_item_domain.sql` | iot_data_item + iot_data_item_ref 建表（p32 逐字，CREATE TABLE IF NOT EXISTS 幂等）+ sys_menu 4036-4040 五行（parent_id=4010）+ role 1 授权 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/entity/IotDataItem.java` | 实例数据项实体（p32 逐字列映射；javadoc 登记 full_key/双 uk/编码不复用纪律） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/entity/IotDataItemRef.java` | 引用关系实体（ref_type 1 数据项 2 点位；ref_item_id 可空回填） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/mapper/IotDataItemMapper.java` | BaseMapper + `selectOccupancyByInstance` 含已删行旁路（@InterceptorIgnore + 显式 (tenant_id, instance_id)，照 0011/0012 旁路范式） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/mapper/IotDataItemRefMapper.java` | 引用关系 BaseMapper（写入规则 javadoc 登记） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/event/InstanceItemsGeneratedEvent.java` | 领域事件 record（instanceId/tenantId/生成项 id 列表/依赖图变更标记；只发布不建监听器） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/RelativeRefExpander.java` | 相对引用展开器（纯函数：SELF/PARENT 展开 + 绝对引用剥壳 + 引用集收集去重；顶级 PARENT 抛明确文案；`containsParentRef` 供 lazy 父解析） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotDataItemService.java` | 服务接口（generateForInstance / regenerateForInstance / selectItemsByInstanceId / countByInstanceIds） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java` | 生成主链（清单映射/展开/依赖图/事件/双道闸 4002/@Transactional 整批失败）+ 清单查询数据权限裁剪 + 删除保护引用计数 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemController.java` | GET /iot/item/list（iot:item:list）+ POST /iot/instance/{id}/items/regenerate（iot:item:add，brief 裁决路由） |
| 测试 4 个：`RelativeRefExpanderTest`(8) / `IotDataItemServiceImplTest`(21) / `IotDataItemControllerTest`(7) / `IotDataItemMenuPermissionGuardTest`(3) | 展开/生成/依赖图/幂等/事件/数据权限/接口契约/菜单守卫 |

**修改（6，均在声明内）**

| 文件 | 内容 |
| --- | --- |
| `backend/.../instance/service/impl/IotModelInstanceServiceImpl.java` | **两处授权接线**：① createInstance 保存成功后同事务调 `generateForInstance(entity.getId())`（触发一）；② deleteInstances 增数据项引用保护（0012 预留点，有项拒删并提示项数）+ 注入 dataItemService |
| `backend/.../instance/service/IotModelInstanceService.java` | **仅 javadoc**（2 行）：deleteInstances 契约注释由「归 TASK-0013 交付后自然生效」改为「TASK-0013 接线后生效」——同步已过时注释，无行为改动（登记为文档同步项） |
| `backend/.../instance/service/impl/IotModelInstanceServiceImplTest.java` | +6 用例（创建触发生成 3 + 删除保护 3）+ setUp 注入 dataItemService mock 与默认行为（既有 47 用例语义不变） |

未触碰：frontend/、ingest/、模板域（只读其数据项清单）、dataperm 组件、workitems 库其他文件；TASK 状态保持「进行中」未动。

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0（-q 静默）。ot-admin 全量：

```
[INFO] Tests run: 1445, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  51.882 s
```

1445 = 基线 1400 + 新增 45（展开器 8 + 生成服务 21 + 控制器 7 + 守卫 3 + 实例接线 6）。0 失败 0 错误。

**TDD 红转绿记录**（两轮）：
1. `IotDataItemServiceImplTest` 首轮红 8（Tests run: 21, Failures: 3, Errors: 5）——根因不是断言放松而是**实现缺陷**：父实例解析为 eager（无 PARENT 引用的生成也查父行，测试夹具未 stub 时抛「父实例不存在」，且暴露了「无 PARENT 引用时父查询多余」的设计问题）。修复：实现改为 lazy——`RelativeRefExpander.containsParentRef(formula)` 命中才 resolveParent（顶级判定/父缺失报错都在首次需要时发生）；测试侧同批补 PARENT 存在性所需的父实例项 stub（loadParentItems 数据源）。**为什么现在对了**：父解析时机与 PARENT 引用严格绑定，无 PARENT 引用的生成不再依赖父行存在（顶级组织节点生成零依赖）；PARENT 路径的失败语义（顶级拒绝/父缺失拒绝/父项缺失拒绝）仍全部保留在各自测试点。同轮 testGenerate_ExpandedFormulaTooLong 夹具 SELF.X 不存在提前失败——补既有活跃项 stub 后长度校验成为失败点。
2. `IotDataItemControllerTest` 红 1（instanceId 缺失期望 HTTP 400 实为 200）——根因：全局异常处理器把 MissingServletRequestParameterException 转为 body code=400（HTTP 200），与既有控制器测试口径一致。修测试断言（对照 GlobalExceptionHandler.handleMissingServletRequestParameter 确认），实现无改动。

### 2.2 验收 2：升级脚本二次幂等 + 两表/uk/菜单核对（sql-exec.sh，127.0.0.1:3307）

**第一次执行**（`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_iot_data_item_domain.sql`）：

```
OK(0)      ← SET NAMES
OK(0)      ← CREATE TABLE iot_data_item
OK(0)      ← CREATE TABLE iot_data_item_ref
OK(5)      ← INSERT IGNORE sys_menu 4036-4040 五行
OK(5)      ← INSERT IGNORE sys_role_menu 五行
DONE
```

**第二次执行**（幂等复核）：

```
OK(0)
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

两次均 DONE、无 SQLERROR（CREATE TABLE IF NOT EXISTS + INSERT IGNORE 幂等生效）。

**两表 + uk 双唯一核对**（`SHOW TABLES` / `SHOW INDEX`）：

```
Tables_in_ot_wisfactory (iot_data_item%)
iot_data_item
iot_data_item_ref
iot_data_item_template

iot_data_item	0	uk_tenant_instance_data	1	tenant_id
iot_data_item	0	uk_tenant_instance_data	2	instance_id
iot_data_item	0	uk_tenant_instance_data	3	data_code
iot_data_item	0	uk_tenant_full_key	1	tenant_id
iot_data_item	0	uk_tenant_full_key	2	full_key
iot_data_item_ref	0	uk_item_ref	1	item_id
iot_data_item_ref	0	uk_item_ref	2	ref_full_key
iot_data_item_ref	0	uk_item_ref	3	ref_type
```

**菜单 4036-4040 核对**：

```
id	menu_name	menu_type	parent_id	perms
4036	数据项查询	3	4010	iot:item:list
4037	数据项详情	3	4010	iot:item:query
4038	数据项生成	3	4010	iot:item:add
4039	数据项修改	3	4010	iot:item:edit
4040	数据项删除	3	4010	iot:item:remove
```

**超管授权核对**：`SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id BETWEEN 4036 AND 4040` → `(1,4036)...(1,4040)` 五行。

### 2.3 验收 3：dev 冒烟（`cd backend && mvn spring-boot:run -pl ot-admin`，日志 backend/target/task0013_dev.log）

启动前端口探测：8098 无 LISTENING。启动：`Started OtAdminApplication in 11.118 seconds (process running for 11.958)`；启动段 ERROR 计数 = **0**（全程 6 条 ERROR 均为负路径故意触发的业务异常留痕：字典未登记 2 条 + 删除保护拒绝 2 条 + 实例不存在 1 条 + 重试前同 1 条，无意外错误）。用毕即停：taskkill（PID 66180）→ 8098 复核无 LISTENING。

夹具前置：模板 TPL13WS（id=5，车间级，1 项 ELEC_CONS 汇总 SUM/MINUTE）与 TPL13EQ（id=6，设备级，4 项：ELEC_METER 直绑 / ELEC_CONS 公式 `DELTA({SELF.ELEC_METER})` / LOAD_RATE 公式 `{SELF.ELEC_CONS} / {PARENT.ELEC_CONS} * 100` / TOTAL_ELEC 汇总 SUM/DAY）经模板域 API 建立；字典补 4 码（ELEC_METER/ELEC_CONS/LOAD_RATE/PARAM_C，TOTAL_ELEC 复用既有登记——经 SQL 种子，冒烟夹具口径）。

```
=== S1) admin 登录（dev /auth/login/test）=== {"code":200,...}
=== S2a/S2b) POST /iot/template TPL13WS(2)/TPL13EQ(4) → 200
=== S2c/S2d) POST /iot/template/item → data:1 / data:4（字典未登记的首次调用被 0011 校验器拒，补登记后成功——字典防线正向佐证）
=== S3a) POST /iot/instance WS13（父=F01，选 TPL13WS，绑部门4）→ 200；DB: id=8，数据项 ELEC_CONS full_key=F01.WS13.ELEC_CONS source_type=3 rollup=SUM calc_period=60 生成
=== S3b) POST /iot/instance EQ13（父=WS13，选 TPL13EQ）→ 200；DB: id=9，四项生成——
  ELEC_METER   full_key=F01.WS13.EQ13.ELEC_METER        source_type=1  point_id=null  calc_period=0
  ELEC_CONS    formula=DELTA(F01.WS13.EQ13.ELEC_METER)   （SELF 展开）calc_period=60（MINUTE→秒）valid_min=0/valid_max=100000
  LOAD_RATE    formula=F01.WS13.EQ13.ELEC_CONS / F01.WS13.ELEC_CONS * 100（PARENT 展开为父实例键）calc_period=3600（HOUR）
  TOTAL_ELEC   rollup_method=SUM  calc_period=86400（DAY）
  全部：enabled=1 status=1 config_version=null deleted=0 point_id=null
=== S4) ref 图 DB 核对（iot_data_item_ref JOIN，恰 3 条边、ref_type=1、ref_item_id 全回填）：
  item 3 ELEC_CONS  → F01.WS13.EQ13.ELEC_METER  → ref_item_id=2
  item 4 LOAD_RATE  → F01.WS13.EQ13.ELEC_CONS   → ref_item_id=3
  item 4 LOAD_RATE  → F01.WS13.ELEC_CONS        → ref_item_id=1（父实例项）
  ★ WS13 汇总项（item 1）出边为 0 条——新增项未新增任何上级对自己的引用（ADR-0008 一档实现落点实测）
=== S4b) GET /iot/item/list?instanceId=9 → 200，四项 full_key/展开后公式/rollup_method/calc_period 与 DB 一致
=== S5a) POST /iot/instance/9/items/regenerate（全部已存在）→ data:0（幂等跳过）
=== S5c) 补模板新项 PARAM_C（POST /iot/template/item data:1）
=== S5d) regenerate → data:1（模板新增项补生成；DB: id=6 PARAM_C F01.WS13.EQ13.PARAM_C）
=== S5e) DELETE /iot/template/item/10（删模板项 PARAM_C → 200 逻辑删）
=== S5f) regenerate → data:0 且 DB 实例项 PARAM_C（id=6）仍在 deleted=0（模板已删项保留，历史不删）
=== S6)  DELETE /iot/instance/9（有 5 项）→ 拒：
  {"code":400,"message":"实例存在数据项，禁止删除（先清理数据项: F01.WS13.EQ13（5 项））"}
=== S6b) POST /iot/instance/999/items/regenerate（实例不存在）→ {"code":400,"message":"实例不存在或已删除: 999"}
=== S7a) 清项（本期无数据项删除 API，DB 逻辑删 instance_id=9 五项 + 三条 ref 边——报告登记）→ remaining=0
=== S7b) DELETE /iot/instance/9 → {"code":200}；DB: id=9 deleted=1
=== S8)  DELETE /iot/instance/8（WS13，1 项）→ 拒（{"code":400,...F01.WS13（1 项）...}）——删除保护对任意含项实例生效
```

判定：**通过**（brief 冒烟序列全步覆盖：建模板含 SELF./PARENT. 公式项 → 建父实例 → 建子实例 → 查清单（full_key/公式展开后/ref 图）→ 重生成幂等三态 → 删子实例拒（有项）→ 清项后删成功）。

### 2.4 验收 4：prod 档真实启动回归

打包：`mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（全量测试已在验收 1 同一代码态覆盖）。

启动（沿 TASK-0012 已确认命令 + 三密钥环境变量）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
启动前端口探测：8098 无 LISTENING
Started OtAdminApplication in 20.026 seconds (process running for 20.817)
=== prod 启动段 ERROR 计数（Started 之前） ===
0
=== prod 真实登录（Redis 预置 captcha:code:task0013b="CD14"（Jackson 序列化带引号）+ SM4 hex 密文 base64 携带，jshell ROUNDTRIP=admin123） ===
{"code":200,...}（前两次 500 为自造种子失误留痕：①密文未按 base64(hex) 携带；②验证码被首次尝试消费后未补种——见 §4.4）
=== prod GET /iot/item/list?instanceId=8 ===
HTTP_STATUS:200（body code=200：ELEC_CONS / F01.WS13.ELEC_CONS / sourceType=3 / rollup=SUM / calcPeriod=60——dev 冒烟数据在位）
=== prod GET /iot/item/list?instanceId=9（已删实例） ===
HTTP_STATUS:200（body code=400「实例不存在或已删除」——负路径留痕，属预期业务拒绝）
```

用毕即停：prod 实例 taskkill → 8098 复核无 LISTENING。判定：**通过**（启动段 0 ERROR + item/list 200；全程 4 条 ERROR 均为登录种子失误与故意负路径留痕，非应用缺陷）。

### 2.5 验收 5：改动范围核对 `git diff --name-only 46e923d..HEAD`

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/entity/IotDataItem.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/entity/IotDataItemRef.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/event/InstanceItemsGeneratedEvent.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/mapper/IotDataItemMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/mapper/IotDataItemRefMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotDataItemService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/RelativeRefExpander.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/IotModelInstanceService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/config/IotDataItemMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/support/RelativeRefExpanderTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImplTest.java
resources/sql/upgrade/20261122_iot_data_item_domain.sql
```

核对结论：17 文件 = `dataitem/**` 9 个（声明内）+ `resources/sql/upgrade/20261122_iot_data_item_domain.sql`（声明内）+ `iot/**` 测试 5 个（声明内）+ `instance/service/impl/IotModelInstanceServiceImpl.java`（brief 授权接线点，diff 实测仅 import+注入+触发一+删除保护两段）+ `instance/service/IotModelInstanceService.java`（javadoc-only 文档同步 2 行，§4.1 登记）。**通过**。

## 3. 未做项与原因

1. **query/edit/remove 三权限行的接口未实现**（iot:item:query/edit/remove 菜单行 4037/4039/4040 已登记）：对应接口（详情/公式保存/汇总设置/点位绑定/数据项删除，详细设计 §3.1 p22）归 0015（模板变更三档/公式编辑）、0023（点位绑定）等后续任务——brief 本期仅要求重生成路由/权限/参数。守卫测试按「SQL 五行恰好 + 注解 ⊆ 五值」双锁定，预留行防漂移（IotDataItemMenuPermissionGuardTest javadoc 登记）。
2. **ref_type=2 点位引用边未产**（brief 明文）：点位绑定动作归 FEAT-0007 / TASK-0023，本期生成只处理公式类引用。
3. **汇总项（source_type=3）的子级引用边未产**（设计口径，非遗漏）：产品设计 §5.3「系统在计算时动态查找当前全部子节点，不需要把每个子节点逐个写进公式」——汇总项依赖为运行时动态查找；ADR-0008 一档「新增项默认不参与上级汇总」恰由「生成不写父→子边」实现（汇总参与开关载体=上级汇总项依赖图，开关打开时才落边——开关动作接口归后续任务）。dev 冒烟 S4 实测父汇总项出边 0 条。
4. **数据项删除 API 未做**（归 query/edit/remove 后续接口任务）：冒烟「清项后删成功」经 DB 逻辑删（sql-exec UPDATE deleted=1）完成，已在冒烟步骤标注。
5. **领域事件监听器未建**（brief 明文）：InstanceItemsGeneratedEvent 只发布不注册消费方；计算引擎（FEAT-0012）为未来消费方，接线时可用 @TransactionalEventListener 调整为提交后语义（事件 javadoc 已登记此提示）。

## 4. 偏离计划之处

1. **IotModelInstanceService 接口 javadoc 2 行同步**（brief 声明 impl「仅一处」，接口文件在范围字面之外）：deleteInstances 契约注释原为「实例数据项引用保护归 TASK-0013 交付后自然生效（一期该计数恒 0）」，接线后即为不实陈述——同步为「TASK-0013 接线后生效」。纯注释、零行为改动；如评审认为越界，回退该 2 行不影响任何行为与测试。
2. **instance impl 实际为两处接线而非字面「一处」**：brief 改动范围声明该文件为授权例外，其内 D1（创建触发）与 D7（删除保护）本就是两个授权动作，均落在此文件（import+字段注入+createInstance 尾部调用+deleteInstances 守卫段），未触碰文件其他任何逻辑。
3. **顶层 PARENT 引用的失败语义放在展开器而非生成服务**：brief「顶级实例的 PARENT 引用=生成失败并报明确文案」——实现为 RelativeRefExpander.expand 对 parentInstanceCode=null + PARENT token 抛 BusinessException（文案含「顶级实例」+引用原文+实例编码），生成服务将异常透传、@Transactional 整批回滚（含触发一的实例行）。语义与 brief 一致，落点选择使展开器可独立单测（8 用例覆盖）。
4. **SELF.X / PARENT.X 引用存在性整批失败**：brief 只明文「展开失败的项：整批失败」；本实现把「引用的编码不在实例/父实例项集」也判为展开失败（整批拒绝，文案含引用原文与实例/父实例编码）——fail-closed 防悬空公式落库；跨设备**绝对引用不核存在性**（被引实例可能后建，剥壳入图不阻断），一期口径已登记于实现类 javadoc。若协调者认为 SELF/PARENT 存在性应放宽为仅告警，改动点在 IotDataItemServiceImpl 生成循环内两段（各一处 if）。
5. **default_source_type 为空的模板项兜底 source_type=1（直绑）**：模板列可空（p13「否」）、实例列 NOT NULL（p14「是」）——未配取值方式的项按最简直绑落地（point_id=NULL，绑定归 0023），不阻断实例创建。测试 testGenerate_NullDefaultSourceType_BindsToBind 固化。
6. **环境**：WSL VM 再次 idle 回收导致容器重启（0012 报告同类现象），按同款处置起 4h keepalive 会话（`wsl sleep 14400` 后台）后继续；数据无失。prod 登录前两次 500 为自造种子失误留痕（密文须 base64(hex) 携带、验证码一次性消费需补种），修正后登录 200，全程无应用缺陷。

## 5. 遗留问题

1. **dev 库冒烟留痕**（是否清理由协调者裁决）：字典 5 码（ELEC_METER/ELEC_CONS/LOAD_RATE/TOTAL_ELEC/PARAM_C，id=4-7 与 PARAM_C）；模板 TPL13WS(id=5)/TPL13EQ(id=6) 及其模板数据项（id=1-10，其中 id=10 PARAM_C 已逻辑删）；实例 F01.WS13(id=8，1 项) 与 F01.WS13.EQ13(id=9，已逻辑删)；实例数据项 6 行（id=1-6，其中 instance_id=9 的 5 行已逻辑删）+ ref 边 3 行（已逻辑删）。重跑冒烟需换码（同码撞占用属预期）。
2. **「清项」API 缺口**：实例删除保护生效后，用户若无数据项删除接口则只能删实例前找 0023/后续任务交付的清项入口——0023（点位绑定）或独立清项任务交付前，dev/管理面删除含项实例会持续被拒（fail-closed 符合设计意图，但操作链路依赖后续任务，请协调者确认任务排期覆盖）。
3. **循环检测仍归计算引擎**（FEAT-0012）：生成链对 SELF 自引用（自环边）照常落库不拒（RelativeRefExpanderTest 固化该口径）；公式 validate/preview 接口（p22）归 T5 任务。
4. **事件消费时序**：InstanceItemsGeneratedEvent 在事务内发布（落库成功后、提交前）；未来消费方若需「提交后」语义须自行用 @TransactionalEventListener——已在事件 javadoc 登记，FEAT-0012 接线时裁决。
5. **跨租户复制 / 平台预置模板**：与既有口径自洽未特判——多租户插件使租户会话只见本租户模板（含其数据项清单），平台预置模板（tenant_id=NULL）对租户会话不可见即不参与生成（0012 报告 §3.3 同口径）。
6. **环境**：WSL keepalive 会话约 4h 后自然回收（见 §4.6），届时按 0011/0012 方式（`wsl -d Ubuntu -- service docker start`）恢复；redis captcha 种子 task0013a/b 已过期自动清理（EX 600）。
7. **TC-0004 联测项**：步骤 3「公式 {SELF.ELEC_METER} 展开为 F01.WS01.EQ001.ELEC_METER」在本任务 dev 冒烟以 TPL13EQ/F01.WS13.EQ13 实测通过（DELTA 展开）；步骤 4/5（data_code 不可改、参数覆盖时间线）归 0011/参数覆盖任务范畴，本任务未重复构造。

---

## 修订轮（2026-11-22，验收退回一轮 §13：评审 Critical 0 / Important 2 / Minor 5）

- 评审报告：reports/TASK-0013-review.md；协调者裁决必修四项：I-1（畸形 token 静默 fail-open）、I-2（SELF 存在性集合含已删占用码 → 依赖图悬空）、M-5（剥壳键超长 500 形态）、M-6（source_type=2 且公式空白落库）。
- 修订提交：`4d7246d`（单提交，注明 TASK-0013 修订轮），改动 2 文件均落 dataitem 域声明内：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImplTest.java
```

### 修订内容（全部在生成循环一处修完）

1. **I-1 展开终检**（生成循环内，展开+存在性核对之后）：展开后公式不得残留 `{` 或 `}`——畸形/超纲 token（如 `{ self.X }` 带空格、`{SELF.ELEC-X}` 连字符）不匹配展开与剥壳两个正则、不核存在性即原样残留，残留即整批拒（PARAM_ERROR，文案含原始编码与 token 示例「示例 { self.X } + 1」，示例截 40 字符防消息过长）。一行防线封静默 fail-open——坏公式不再随模板批量传染到该模板所有实例。编码字符集约束上推字典域**不在本任务**（0008 评审已登记为待立规则项，deferred）。
2. **I-2 selfScope 收敛**：SELF 存在性集合由「活跃占用 ∪ **全部模板编码**」改为「活跃占用 ∪ **本批将插入项（newItems）编码**」——「已删行占用但模板在册」的编码因 occupancy skip 不会在本批生成，引用它的公式在展开期即整批失败（文案指向引用的编码），ref 不落悬空边（formula 指死键 / ref_item_id=NULL 的组合死角封口）。
3. **M-5 剥壳键限长**：展开终检段对 resolvedRefs 逐键校验 >180（ref_full_key varchar(180) 列宽）即 paramError（文案含键、上限与实际长度），不触 SQL Data-too-long 的 500 形态。
4. **M-6 公式项缺公式**：source_type=2 且 default_formula 空白（null/空白串）→ 整批拒「公式项缺少公式: <code>」——与生成链其他校验同纪律（fail-closed）；模板域侧耦合校验仍归 FEAT-0012/0015。

### 红转绿记录（修订轮测试先行）

修订轮新增测试 4 个（`IotDataItemServiceImplTest` 21→25：I-1 一例两形态、I-2 死键引用、M-5 键超长、M-6 null+空白两形态）。先在**首轮实现**（HEAD=fb13384）上运行新测试集 → **红 4**（恰为四项必修的被测行为——修复前均静默落库）：

```
[ERROR] Tests run: 25, Failures: 4, Errors: 0, Skipped: 0
[ERROR]   IotDataItemServiceImplTest.testGenerate_AbsoluteRefKeyTooLong_Fails:524 Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
[ERROR]   IotDataItemServiceImplTest.testGenerate_FormulaItemMissingFormula_FailsWholeBatch:545 Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
[ERROR]   IotDataItemServiceImplTest.testGenerate_MalformedTokenResidualBrace_FailsWholeBatch:471 Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
[ERROR]   IotDataItemServiceImplTest.testGenerate_SelfRefToDeletedOccupiedCode_FailsWholeBatch:505 Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
```

应用修订实现后同类全绿：

```
[INFO] Tests run: 25, Failures: 0, Errors: 0, Skipped: 0
```

### 补充验收 1：全量复跑 `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
ot-core install 退出码 0
[INFO] Tests run: 1449, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  55.114 s
```

1449 = 1445（首轮）+ 4（修订轮红转绿）。0 失败 0 错误。

### 补充验收 2：dev 冒烟复核（日志 backend/target/task0013_rev_dev.log）

启动前端口探测：8098 无 LISTENING。启动：`Started OtAdminApplication in 11.963 seconds (process running for 12.806)`；启动段 ERROR=0。夹具前置：字典补 4 码（R2_MALFORMED/R2_DEAD1/R2_DEAD2/R2_REF2，SQL 种子）；模板 TPL13R2M(id=7，1 项畸形公式)/TPL13R2D(id=8，直绑+SELF 公式两项) 经模板域 API 建立。用毕即停：taskkill（PID 49072）→ 8098 复核无 LISTENING。

```
=== R1b) POST /iot/template/item 畸形公式项（{ self.X } 带空格——0011 括号配平校验通过可入库，佐证畸形公式可达生成链）
{"code":200,"data":1,...}
=== R2)  POST /iot/instance R2MQ（选 TPL13R2M → 创建同事务触发生成 → I-1 终检整批拒）→ 拒 + 实例行回滚：
{"code":400,"message":"公式包含无法识别的引用 token（展开后残留花括号）: 项 R2_MALFORMED，示例 { self.X } + 1"}
DB 核对：F01.WS13.R2MQ 实例行 0 条（回滚）、R2_MALFORMED 数据项 0 条（零落库）

=== R3a) POST /iot/template/item TPL13R2D 两项（直绑 R2_DEAD1 + 公式 DELTA({SELF.R2_DEAD1}) R2_DEAD2）→ data:2
=== R3b) POST /iot/instance R2DQ（选 TPL13R2D）→ 200；DB: id=11，两 items 在位（R2_DEAD1/R2_DEAD2 活跃）
=== R3c) DB 置 R2_DEAD1 实例项 deleted=1（模拟已删占用——本期无数据项删除 API，DB 直删口径同首轮冒烟）
=== R3d) POST /iot/template/item 补 R2_REF2（公式 {SELF.R2_DEAD1} 引用已删占用码）→ data:1
=== R3e) POST /iot/instance/11/items/regenerate（I-2：引用已删占用码 → 展开期整批拒）：
{"code":400,"message":"公式引用的数据项不存在: SELF.R2_DEAD1（实例 F01.WS13.R2DQ，项 R2_REF2）"}
DB 核对：R2_REF2 实例项 0 条（拒绝即零落库，未落悬空边）
（登记：R2_DEAD2 生成期落过的 ref 边（R2_DEAD1 尚活跃时）仍指向其 full_key——存量边属删除前历史，
本期无 ref 清理链路，0015/公式编辑链路「先删旧边再插新边」规则覆盖存量重建；本修订封的是生成期新增悬空边）

=== R4)  POST /iot/instance R2OKQ（正常模板 TPL13EQ 回归 → 生成成功）→ 200；DB: id=12，四项生成
  ELEC_METER full_key=F01.WS13.R2OKQ.ELEC_METER calc_period=0
  ELEC_CONS  formula=DELTA(F01.WS13.R2OKQ.ELEC_METER)（SELF 展开）calc_period=60
  LOAD_RATE  formula=F01.WS13.R2OKQ.ELEC_CONS / F01.WS13.ELEC_CONS * 100（PARENT 展开）calc_period=3600
  TOTAL_ELEC rollup=SUM calc_period=86400
=== R5)  GET /iot/item/list?instanceId=12 → 200，四项公式/full_key 与 DB 一致（正常链路零回归）
```

修订轮日志全程 ERROR 共 2 条，均为两条负路径故意触发的业务异常留痕（I-1 拒 + I-2 拒），无意外错误。判定：**通过**（畸形 token 拒 + 死键引用拒 + 文案准确 + 正常模板回归生成成功，补充验收两步全覆盖）。

### 修订轮 Minor 处置登记

- **本轮已修**：M-5（剥壳键限长）、M-6（公式项缺公式）。
- **deferred（协调者裁决不动）**：M-3（事件发布时机与事务回滚交互——无监听器零影响，FEAT-0012 接线时必须裁决 @TransactionalEventListener 语义）；M-4（selectItemsByInstanceId 先权限后存在性统一——受限用户可探测 id 存在性的 400/空清单差异，改动点在服务方法首两段顺序）；M-7（IotModelInstanceService 接口 javadoc 2 行越界——纯文档同步已留档，协调者越界反查已确认，维持现状）。
