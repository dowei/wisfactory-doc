# 实施报告：TASK-0012（iot_model_instance 与实例树接口：绑部门、权限裁剪）

- 执行者：implementer（brief：reports/TASK-0012-brief.md）
- BASE：`c94c9b7` → HEAD：`c9770ef`（main，1 个提交，注明 TASK-0012）
- 总体结论：**5 条验收全部通过**（全量回归 1390 = 基线 1331 + 新增 59，0 失败 0 错误；升级脚本两次执行均 DONE 且判存幂等生效；dev 冒烟 13 步全过；prod 档启动段 0 ERROR + /iot/instance/tree 200；改动范围 15 文件全落在声明内）

---

## 1. 改动文件清单（15 个，与验收 5 的 git diff 一致）

**新增（10）**

| 文件 | 内容 |
| --- | --- |
| `resources/sql/upgrade/20261122_iot_instance_columns.sql` | iot_model_instance 补两列（information_schema 判存幂等）+ sys_menu 4030-4035 六行（parent_id=4010）+ role 1 授权 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/dto/IotInstanceCreateForm.java` | 创建表单（shortCode @Pattern [A-Za-z0-9_]/instanceName/nodeType/parentId/templateId/deptId/sort） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/dto/IotInstanceUpdateForm.java` | 编辑表单（不可变五字段仅作一致性核对；可变白名单 instanceName/deptId/sort/status） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/dto/IotInstanceTreeVO.java` | 树节点 VO（含 appliedTemplateVersion/locked/status，children 递归） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/mapper/IotInstanceCodeMapper.java` | uk 双道闸第一道旁路预检（含已删行，@InterceptorIgnore + 显式 (tenant_id, instance_code)，照 0011 修订轮 I-1 范式） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/mapper/IotInstanceCodeRow.java` | 占用行最小映射（id/instance_code/deleted） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/IotModelInstanceService.java` | 服务接口（六方法） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImpl.java` | 服务实现（创建选模板/路径式编码/ancestors 仿 sys_dept/绑部门、编辑不可改五字段+锁定语义、删除子实例保护、**树裁剪接线 DataPermService.resolve**、列表 id ∈ 可见集、详情 403=SCOPE_FORBIDDEN 首个消费点） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImplTest.java` | 服务层 37 用例（四类 node_type 模板约束/短码拼接全链/同码含已删行 4002/dept 必填边界/不可改五字段/锁定受限/树裁剪四场景/列表裁剪/详情 403/删除保护/防环） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/controller/IotModelInstanceControllerTest.java` | 控制器 16 用例（权限注解六接口反射断言/路由/参数校验/4002 与 403 透传） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/config/IotInstanceMenuPermissionGuardTest.java` | 守卫测试 3 用例（SQL 4030-4035 ↔ 控制器注解双向一一对应） |

**修改（5）**

| 文件 | 内容 |
| --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/IotModelInstance.java` | +两新字段 appliedTemplateVersion/locked（brief「实体扩展」授权项，0011 javadoc 声明在案）+ javadoc 更新 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotModelTemplateServiceImpl.java` | **仅一处**：updateTemplate 增 M-6 守卫（有实例引用禁改 node_type，文案含引用计数；node_type 未变时不查询）——brief 授权例外 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotModelTemplateServiceImplTest.java` | +3 用例（M-6 红转绿/无引用改档放行/未变不触发计数） |
| 其余 2 个测试文件见上表新增 | — |

未触碰：frontend/、ingest/、dataperm 组件本体（仅复用）、字典域、workitems 库其他文件。

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0（-q 静默）。ot-admin 全量：

```
[INFO] Results:
[INFO]
[INFO] Tests run: 1390, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
[INFO] Total time:  51.850 s
[INFO] Finished at: 2026-11-22T04:35:35+08:00
```

1390 = 基线 1331 + 新增 59（实例服务 37 + 实例控制器 16 + 实例守卫 3 + 模板服务 M-6 3）。0 失败 0 错误。

**M-6 红转绿记录**：先补 3 个守卫用例运行 → 红（`testUpdateTemplate_NodeTypeChangeWithInstanceRefRejected: Expected BusinessException to be thrown, but nothing was thrown`，其余 20 绿）→ 加守卫一段 → 同类全绿。

### 2.2 验收 2：升级脚本二次幂等 + 列/菜单核对（sql-exec.sh，127.0.0.1:3307）

**第一次执行**（`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_iot_instance_columns.sql`）：

```
OK(0) ×11          ← SET NAMES / 判存 SET×2 / PREPARE×2 / EXECUTE(两条 ALTER 加列)×2 / DEALLOCATE×2 / IF 分支 SELECT 兜底
OK(6)              ← INSERT IGNORE sys_menu 4030-4035 六行
OK(6)              ← INSERT IGNORE sys_role_menu 六行
DONE
```

**第二次执行**（幂等复核）：

```
OK(0) ×4
note
skip: applied_template_version already exists
OK(0) ×4
note
skip: locked already exists
OK(0) ×3
DONE
```

两次均 DONE、无 SQLERROR；判存幂等生效（列已存在即跳过 ALTER）。

**两新列核对**（`SHOW COLUMNS FROM iot_model_instance` 节选）：

```
applied_template_version	int	YES		null
locked	tinyint	YES		0
```

**菜单 4030-4035 核对**：

```
id	menu_name	menu_type	parent_id	perms
4030	实例查询	3	4010	iot:instance:list
4031	实例详情	3	4010	iot:instance:query
4032	实例新增	3	4010	iot:instance:add
4033	实例修改	3	4010	iot:instance:edit
4034	实例删除	3	4010	iot:instance:remove
4035	实例树查询	3	4010	iot:instance:tree
```

**超管授权核对**：`SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id BETWEEN 4030 AND 4035` → `(1,4030)...(1,4035)` 六行。

### 2.3 验收 3：dev 冒烟（`cd backend && mvn spring-boot:run -pl ot-admin`，日志 backend/target/task0012_dev.log）

启动前端口探测：8098 无 LISTENING。启动：`Started OtAdminApplication in 9.739 seconds (process running for 10.331)`；启动段 ERROR 计数 = **0**（全程 17 条 ERROR 中 16 条为负路径故意触发的业务异常留痕、1 条为用毕即停 taskkill 的 maven 插件留痕）。

```
=== S1) admin 登录（dev /auth/login/test） ===
{"code":200,"data":{"nickname":"管理员","token":"2c9df949-...","userId":1,"username":"admin"},"message":"操作成功",...}

=== S2a) POST /iot/template 模板 SMOKE_INST_EQTPL（nodeType=4）→ 200
=== S2b) POST /dept 冒烟车间部门T12（父=1）→ 200；DB: id=4, ancestors="0,1", status=1
=== S3a) POST /iot/instance F01（工厂，绑部门1）→ 200
=== S3b) POST WS01（车间，父=F01，绑部门4）→ 200
=== S3c) POST WS02（车间，父=F01，绑部门2——无权兄弟）→ 200
=== S3d) POST F02（工厂，绑部门2——无权厂）→ 200
=== S3e) POST EQ001（设备，父=WS01，选模板4）→ 200
=== S3f) 负路径：设备不选模板 → 拒
{"code":400,"message":"设备/组件级实例必须选择模板（TD-0008 强制选模板）"}
=== S3g) 负路径：短码带点 → 拒
{"code":400,"message":"本级短码仅允许字母/数字/下划线"}
=== S3h) 负路径：重复全码 F01 → 4002
{"code":4002,"message":"实例编码已存在: F01"}
=== S3i) 负路径：车间不绑部门 → 拒
{"code":400,"message":"工厂/车间级实例必须绑定部门（数据权限锚点必填）"}

=== S4) 全码/ancestors/白名单 DB 核对（iot_model_instance 全 5 行） ===
id	tenant_id	instance_code	parent_id	ancestors	node_type	dept_id	sort	status	applied_template_version	locked	deleted
2	1	F01	0	0	1	1	1	1	null	0	0
3	1	F01.WS01	2	0,2	2	4	1	1	null	0	0
4	1	F01.WS02	2	0,2	2	2	2	1	null	0	0
5	1	F02	0	0	1	2	2	1	null	0	0
6	1	F01.WS01.EQ001	3	0,2,3	4	null	1	1	1	0	0

=== S5a) POST /role 受限角色（dataScope=2 本部门及下属，menuIds=[4010,4030,4031,4035]）→ 200（DB: id=3, tenant_id=1）
=== S5b) POST /user smoke_inst_t0012（dept=4 车间部门，密码 SM4 密文提交，jshell ROUNDTRIP-OK=Smoke@2026）→ 200（DB: id=3, tenant_id=1）
=== S5c) PUT /user/3/role {roleIds:[3]} → 200；DB: role_menu (3,4010/4030/4031/4035) 四行 + user_role (3,3)

=== S6a) 受限用户登录 → 200（userId=3）
=== S6b) 受限用户 GET /iot/instance/tree → code=200
id=2 code=F01 nodeType=1 deptId=1 status=1 ancestors=0          ← 根路径骨架
  id=3 code=F01.WS01 nodeType=2 deptId=4 status=1 ancestors=0,2
    id=6 code=F01.WS01.EQ001 nodeType=4 deptId=null status=1 ancestors=0,2,3
（WS02 与 F02 不返回——无权兄弟/无权厂不出现；F01 无权子树其余分支不出现）
=== S6c) 受限用户 GET /iot/instance/list → code=200 total=2（WS01、EQ001——id ∈ 可见集）
=== S6d) 受限用户 GET /iot/instance/2（F01）→ {"code":403,"message":"无权访问该实例（数据权限不命中）"}
=== S6e) 受限用户 GET /iot/instance/3（WS01）→ code=200 data 含 appliedTemplateVersion 等全字段

=== S7a) admin GET /iot/instance/tree → 全树两厂五节点（F01→WS01→EQ001 / F01→WS02 / F02）
=== S7b) admin GET /iot/instance/list → total=5
=== S8)  admin PUT /iot/instance 改 parent（F01 挂到自身子孙 WS01 下）→ 拒
{"code":400,"message":"父实例不允许修改（挪树一期禁止，TD-0010）"}
=== S8b) admin PUT /iot/instance 改 instanceCode → 拒
{"code":400,"message":"实例编码不允许修改: F01"}

=== S9a) DB 置位锁定：UPDATE iot_model_instance SET locked=1 WHERE id=4 → OK(1)
=== S9b) 锁定后改名 → {"code":400,"message":"实例已锁定，仅允许修改状态"}
=== S9c) 锁定后改部门 → {"code":400,"message":"实例已锁定，仅允许修改状态"}
=== S9d) 锁定后改 status=0 → {"code":200}
=== S9e) DB 核对：id=4 locked=1 / status=0 / instance_name 未变

=== S10) admin DELETE /iot/instance/2（F01 有子实例）→ 拒 + 提示子实例编码
{"code":400,"message":"存在子实例，禁止删除（子实例: F01.WS01, F01.WS02）"}
=== S11) M-6 守卫冒烟：PUT /iot/template 改 SMOKE_INST_EQTPL nodeType 4→2（有实例引用）→ 拒
{"code":400,"message":"模板已被实例引用，节点类型不允许修改（引用实例 1 个）"}
```

用毕即停：dev 实例 taskkill（PID 37716）→ 8098 复核无 LISTENING。判定：**通过**（brief 冒烟序列全步覆盖 + 三条负路径创建 + M-6 冒烟佐证）。

### 2.4 验收 4：prod 档真实启动回归

打包：`mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（全量测试已在验收 1 同一代码态覆盖）。

启动（沿 TASK-0011 已确认命令 + 三密钥环境变量）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
启动前端口探测：8098 无 LISTENING
Started OtAdminApplication in 17.603 seconds (process running for 18.313)
=== prod 启动段 ERROR 计数（Started 之前） ===
0
=== prod 真实登录（Redis 预置 captcha:code:task0012a（Jackson 序列化带引号值）="AB12" + SM4 admin123 密文，jshell 往返解密验证） ===
{"code":200,"data":{"nickname":"管理员","token":"a04b02da-...","userId":1,"username":"admin"},"message":"操作成功",...}
=== prod GET /iot/instance/tree ===
tree HTTP_STATUS:200（body code=200，两厂五节点全树，EQ001 携带 appliedTemplateVersion=1/locked=0）
=== prod GET /iot/instance/list?pageNum=1&pageSize=10 ===
list HTTP_STATUS:200（body code=200，total=5，codes=[F01, F01.WS01, F01.WS01.EQ001, F01.WS02, F02]）
```

用毕即停：prod 实例 taskkill（PID 12128）→ 8098 复核无 LISTENING。判定：**通过**（启动段 0 ERROR + /iot/instance/tree 200 + list 200）。
全程 ERROR 行共 2 条，均为登录前自造种子失误的一次性留痕（Redis captcha 种子未按 Jackson 序列化加引号 → SerializationException），修正种子后登录 200，非应用缺陷，全程无其他 ERROR。

### 2.5 验收 5：改动范围核对 `git diff --name-only c94c9b7..HEAD`

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/controller/IotModelInstanceController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/dto/IotInstanceCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/dto/IotInstanceTreeVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/dto/IotInstanceUpdateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/mapper/IotInstanceCodeMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/mapper/IotInstanceCodeRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/IotModelInstanceService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/IotModelInstance.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotModelTemplateServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/config/IotInstanceMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/controller/IotModelInstanceControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotModelTemplateServiceImplTest.java
resources/sql/upgrade/20261122_iot_instance_columns.sql
```

核对结论：15 文件 = `instance/**` 8 个（声明内）+ `resources/sql/upgrade/20261122_iot_instance_columns.sql`（声明内）+ `iot/**` 测试 4 个（声明内）+ `template/entity/IotModelInstance.java`（brief 代码结构「实体扩展」明文授权项）+ `template/service/IotModelTemplateServiceImpl.java`（brief 授权例外，diff 实测仅 M-6 守卫一段）。**通过**。

## 3. 未做项与原因

1. **删除保护的「实例数据项引用」计数未接线**：iot_data_item 表与其生成链归 TASK-0013 交付（本期该表不存在，计数恒 0，不构成拒绝路径——brief 明文口径）。`deleteInstances` 已留注释声明届时接线。
2. **锁定动作本身无 API**：REQ-0005 锁定口径只约束「锁定后配置变更受限」的编辑语义；置位动作（谁锁、何时锁）上游未定义，冒烟经 DB UPDATE 置位（S9a）。若产品需要锁定开关接口，归后续任务裁决。
3. **平台预置模板不可用于创建实例**（既有口径沿用非新决策）：多租户插件对 iot_model_template 注入会话租户条件，tenant_id=NULL 的平台预置模板对租户会话不可见，故设备级实例创建只能引用本租户模板——与 0011「平台预置模板走 SQL 种子、接口一期降级」自洽，未做特判。

## 4. 偏离计划之处

1. **编辑白名单无 remark（brief「instanceName/deptId/sort/status/remark 类」按库内实际可变集落地）**：iot_model_instance 附录 A p31 逐字 DDL（0011 建库）**无 remark 列**，实体无 remark 字段；持久化 remark 需第三处「与上游差异」加列，而 brief 增量脚本明文只授权两列（applied_template_version/locked）。故创建/编辑表单均无 remark 字段，锁定语义相应收窄为「locked=1 时仅允许修改 status」。代码内注释与 DTO javadoc 已登记，随上游修订清单一并核对。
2. **树接口权限点为 iot:instance:tree（4035 单设）**：详细设计 §3.1 p22 原文该接口 perms=iot:instance:list；brief 裁决「树接口单设权限点」——按 brief 落地，差异已登记于升级脚本头注「接口对应」段（上游修订清单同类）。
3. **环境：WSL VM 反复回收导致容器重启**（非本任务引入）：验收中途 WSL VM idle 回收（`wsl uptime` 实测 up 0 min 循环），wisfactory-mysql/redis 随 daemon 反复重启、3307 间歇拒绝连接。处置：起 4h keepalive 会话（`wsl sleep 14400` 后台）维持 VM 与容器稳定后继续；数据无失（两新列/菜单行/冒烟数据均在位）。约 4h 后会话自然回收，届时按 0011 报告方式（`wsl -d Ubuntu -- service docker start`）恢复。
4. **prod 登录首次两次 500 ERROR 为自造种子失误**：redis-cli 直存 `AB12`（裸串）不符合 Jackson 序列化器期望（需 `"AB12"` 带引号 JSON 串），登录抛 SerializationException；修正种子后登录 200。留痕在 prod 日志 2 条 ERROR（04:55:54 / 04:56:03），启动段 0 ERROR、业务段无其他 ERROR。
5. **列表 createByOnly 口径按 brief 实现为「空集」**（非偏离，登记供协调者知悉）：ADR-0016 裁决 4 对配置类列表有 `OR create_by=me` 合并语义，brief 对 /iot/instance/list 明文「createByOnly → 空集」（与树查询同一数据查询语义）——按 brief 实现（单测 testList_CreateByOnlyEmpty 固化）。若后续要按裁决 4 对齐，仅需改列表一处短路分支并补 `OR create_by` 拼装。

## 5. 遗留问题

1. **dev 库冒烟留痕**（是否清理由协调者裁决）：模板 SMOKE_INST_EQTPL（id=4）；实例 5 行（F01/F01.WS01/F01.WS02/F01.WS01.EQ001/F02，id=2-6，其中 WS02 locked=1+status=0）；部门 4「冒烟车间部门T12」；角色 3「smoke_inst_ws_t0012」（dataScope=2，菜单 4010/4030/4031/4035）；用户 3「smoke_inst_t0012」（dept=4，密码 Smoke@2026）。重跑冒烟需换码（同码撞 4002 属预期）。
2. **列表 createByOnly 语义确认**（§4.5）：brief「空集」口径与 ADR-0016 裁决 4 的张力，请协调者定版是否随 REQ-0002 验收口径统一（改动点已标注在服务实现注释）。
3. **锁定置位入口**（§3.2）：无 API，冒烟走 DB 置位；0013/前端任务如需锁定开关，服务层语义已就绪（locked 只读进编辑防线，不可经 API 注入）。
4. **环境**：WSL keepalive 会话约 4h 后自然回收（见 §4.3）；redis captcha 种子 task0012a 已过期自动清理（EX 600）。
5. **TC-0004 联测项**（步骤 3「公式 {SELF.ELEC_METER} 展开为 F01.WS01.EQ001.ELEC_METER」）依赖 0013 实例数据项生成，本任务仅交付实例域，未构造模板数据项联动冒烟。

---

## 6. 修订轮（2026-11-22，验收退回一轮 §13：评审 Critical 0 / Important 2 / Minor 8）

- 评审报告：reports/TASK-0012-review.md；协调者裁决必修四项：I-1（写路径无数据权限）、I-2（混合角色 createByOnly 短路丢失部门可见集）、M-3（锁定实例可被删除）、M-7（树骨架节点全字段泄露）。
- 修订提交：`46e923d`（单提交，注明 TASK-0012 修订轮），改动 2 文件均落 instance 域声明内（template 域未动）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImplTest.java
```

### 6.1 修订内容

1. **I-1 写路径数据权限防线**（与读裁剪同源同口径）：`updateInstance`/`deleteInstances` 校验目标实例 ∈ 可见集（allTenant 直过；否则 instanceIds.contains；不在 → 403「无权操作该实例（数据权限不命中）」，文案不泄露实例标识，照详情范式）；`createInstance` 校验父实例 ∈ 可见集（顶级实例创建不受限——顶层建厂是运营动作，菜单权限已控），文案「无权在该实例下创建子实例（数据权限不命中）」。防线顺序 fail-closed：编辑=存在性→数据权限→不可变校验；删除=存在性→数据权限→锁定→子实例保护（锁定与存在性细节均不向无权者泄露）。封堵评审指出的升级链：把不可见实例的 dept_id 改绑到自己部门，经组件步骤 3-4 子树展开后该不可见子树整体变为可见（TC-0002 红线被写原语绕过）。
2. **I-2 混合角色可见集**：树/列表短路条件改为仅 `scope.instanceIds().isEmpty()`——createByOnly 只是「存在 scope=4 角色」标记（DataPermServiceImpl 对混合角色返回非空 instanceIds + createByOnly=true），不得清空部门角色产出的可见集（ADR-0016 裁决 4）；纯 scope=4 用户 instanceIds 本为空，行为不变。**列表**补裁决 4 OR 分支：`id IN (instanceIds) OR create_by = 当前用户`（createByOnly 时；实例属配置类对象=裁决 3 适用集，0010 评审 Minor③ 的真实回归）；**树**不挂 create_by 分支（层级导航视图，scope=4 只对配置类列表生效、不贡献树可见性——口径留痕随本节登记）。
3. **M-3 锁定禁删**：`deleteInstances` 增 locked=1 → 400「实例已锁定，禁止删除」（REQ-0005「锁定后配置变更受限」含删除；此前锁定防线只落在编辑）。数据权限防线先于锁定防线：locked+不可见实例对受限用户 403（不泄露锁定状态），对 admin 400（锁定拒绝）。
4. **M-7 骨架字段收窄**：树骨架路径节点（可见集之外的到根路径节点）字段收窄为 id/instanceCode/instanceName/nodeType/status；可见集节点维持全字段。与详情 403 的不泄露面一致（无权节点连详情都不可看，骨架不应反而携带 deptId/templateId/locked/ancestors 等锚点与全键面）。

### 6.2 红转绿记录（修订轮测试先行）

修订轮新增/调整测试 13 个（净增 10：新增 13、原 testList_CreateByOnlyEmpty 按裁决 4 口径改写为 create_by 用例）。先在**首轮实现**（HEAD=c9770ef）上运行新测试集 → **红 9**（恰为四项修订的被测行为）：

```
[ERROR] Tests run: 47, Failures: 9, Errors: 0, Skipped: 0
[ERROR] IotModelInstanceServiceImplTest.testCreate_InvisibleParent403        Expected BusinessException to be thrown, but nothing was thrown.
[ERROR] IotModelInstanceServiceImplTest.testDelete_InvisibleTarget403        Expected BusinessException to be thrown, but nothing was thrown.
[ERROR] IotModelInstanceServiceImplTest.testDelete_LockedAndInvisible403     Expected BusinessException to be thrown, but nothing was thrown.
[ERROR] IotModelInstanceServiceImplTest.testDelete_LockedRejected            Expected BusinessException to be thrown, but nothing was thrown.
[ERROR] IotModelInstanceServiceImplTest.testList_MixedRoleOrCreateBy         Wanted but not invoked: baseMapper.selectPage（短路未过）
[ERROR] IotModelInstanceServiceImplTest.testList_PureCreateByOnlyQueriesByCreateBy  Wanted but not invoked: baseMapper.selectPage（短路未过）
[ERROR] IotModelInstanceServiceImplTest.testTree_MixedRoleKeepsDeptVisibleSet 混合角色用户的部门可见集不得被 createByOnly 清空 ==> expected: <1> but was: <0>
[ERROR] IotModelInstanceServiceImplTest.testTree_RestrictedVisiblePlusSkeleton 骨架节点不携带 deptId（数据权限锚点面）==> expected: <null> but was: <10>
[ERROR] IotModelInstanceServiceImplTest.testUpdate_InvisibleTarget403        expected: <403> but was: <400>
```

应用修订实现后同类全绿：

```
[INFO] Tests run: 47, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

（中间一次测试内修测试非断言放松：testTree_RestrictedVisiblePlusSkeleton 对照断言 ancestors 夹具值笔误 "0,2"→"0,1"，WS01(id=2) 夹具 ancestors 实为 "0,1"。）

### 6.3 补充验收 1：全量复跑 `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
ot-core install 退出码 0
[INFO] Results:
[INFO]
[INFO] Tests run: 1400, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
[INFO] Total time:  56.476 s
[INFO] Finished at: 2026-11-22T05:18:15+08:00
```

1400 = 1390（首轮）+ 10 净增（修订轮 13 用例 − 1 改写替换）。0 失败 0 错误。

### 6.4 补充验收 2：dev 冒烟复核（日志 backend/target/task0012_rev_dev.log）

启动前端口探测：8098 无 LISTENING。启动：`Started OtAdminApplication in 10.77 seconds`；启动段 ERROR=0。用毕即停：taskkill（PID 79632）→ 8098 复核无 LISTENING；全程 ERROR 13 条均为负路径业务异常留痕 + maven 插件终止留痕，无意外错误。

夹具前置（admin）：沿用首轮冒烟库（角色 3=本部门及下属 dataScope=2 / 用户 3 smoke_inst_t0012 dept=4 / 实例 F01(2)、WS01(3)、WS02(4，locked=1+status=0)、F02(5)、EQ001(6)）；本轮新增：角色 4（smoke_inst_me_t0012r2，dataScope=4 仅本人，菜单含 4032 add）+ 用户 4 smoke_mixed_t0012（dept=4，roles=[3,4] 混合）+ **角色 3 补授 4033/4034（iot:instance:edit/remove）**——评审 I-1 的被测形态正是「持编辑菜单+窄数据范围」的合法配置（未补授前菜单权限层先 403「无权访问该资源」，测试不到服务层防线，补授后按裁决复测）。

```
=== R1a) 混合角色用户创建顶级工厂 FMIX → 200（顶层创建不受限，create_by=4）
=== R1b) 混合角色用户 GET /iot/instance/tree → code=200
id=2 code=F01 deptId=None locked=None          ← 骨架（M-7 收窄实况）
  id=3 code=F01.WS01 deptId=4 locked=0         ← 部门可见集保留（I-2：不被 createByOnly 清空）
    id=6 code=F01.WS01.EQ001 deptId=None locked=0
（FMIX 不出现——树不挂 create_by 分支，scope=4 不贡献树可见性）
=== R1c) 混合角色用户 GET /iot/instance/list → code=200 total=3
  id=3 F01.WS01 createBy=1     ← id IN 可见集分支
  id=6 F01.WS01.EQ001 createBy=1
  id=7 FMIX createBy=4         ← OR create_by=me 分支（裁决 4 落地实况）

=== R2g) 受限用户重新登录（角色 3 补授 edit/remove 后——菜单权限过、数据权限拦）
=== R2h) PUT /iot/instance {"id":2,"deptId":4}（不可见工厂 F01 改绑到自己部门）→ 403
{"code":403,"message":"无权操作该实例（数据权限不命中）"}
=== R2i) DELETE /iot/instance/5（不可见叶子 F02）→ 403
{"code":403,"message":"无权操作该实例（数据权限不命中）"}
=== R2j) DELETE /iot/instance/4（locked=1 且不可见 WS02）→ 403（权限防线先于锁定防线）
{"code":403,"message":"无权操作该实例（数据权限不命中）"}
=== R2k) DB 核对：F01.dept_id=1 未被改 / F02 在 / WS02 locked=1 在（写全部被拦）

=== R3) 混合角色用户（持 add 权限）POST 不可见父 F02 下创建 → 403
{"code":403,"message":"无权在该实例下创建子实例（数据权限不命中）"}
=== R4) admin DELETE /iot/instance/4（locked=1，allTenant 直过数据权限）→ 400
{"code":400,"message":"实例已锁定，禁止删除"}
=== R5) 受限用户树复核（M-7 收窄实况，逐节点键清单）：
骨架 F01 keys=[id, instanceCode, instanceName, nodeType, status]
可见 WS01 keys=[ancestors, deptId, id, instanceCode, instanceName, locked, nodeType, parentId, sort, status]
可见 EQ001 keys=[ancestors, appliedTemplateVersion, id, instanceCode, instanceName, locked, nodeType, parentId, sort, status, templateId]
=== R6) 受限用户 DELETE /iot/instance/3（可见但有子）→ 400 子实例保护照常
{"code":400,"message":"存在子实例，禁止删除（子实例: F01.WS01.EQ001）"}
```

判定：**通过**（补充验收 2 四步全覆盖：不可见改绑 403 / 不可见删除 403 / 混合角色树列表回归 / locked 实例 DELETE——受限用户 403（权限防线先行）+ admin 400（锁定拒绝）两形态取证）。

### 6.5 Minor 处置登记

- **本轮已修**：M-3（锁定禁删）、M-7（骨架收窄）。
- **deferred（协调者裁决不动）**：M-1（实体 javadoc 与 UpdateForm @Schema 的收窄前措辞）、M-4（父子 node_type 层级校验，一期不做待确认）、M-5（详情 isDetailAccess 死代码简化）、M-6（创建/删除事务边界硬化：父行锁或删除后复检）、M-8（升级脚本头注补记树接口权限点差异）。
- **登记待裁决**：评审 Minor 2（删除批内重复 id 去重——`/3,3` 场景 fail-closed 不误删但文案成「实例不存在或已删除: []」）未列入必修四项与 deferred 清单，本修订轮未动，留协调者定版。
