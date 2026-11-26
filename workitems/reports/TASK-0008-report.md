# TASK-0008 实施报告（iot_data_dict 表与字典 CRUD 接口 + /iot 命名空间迁移）

- 执行者：implementer（subagent）
- BASE：`88caf08`（主仓 HEAD，派发时回填）；提交：`c1b2615`（main，注明 TASK-0008，13 files / +1086 −31）
- 总体结论：**6 条验收全部通过**（1 条含一处 brief 基线数字过时的如实更正，见偏离 1；均为证据性更正，不影响任何验收结论）。

---

## 1. 改动文件清单（13 个，全部落在 brief 声明范围）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/.../iot/entity/IotDataDict.java` | 新增 | 实体 `@TableName("iot_data_dict")` 继承 BaseEntity；dataCode @NotBlank+@Size(64)、dataName @NotBlank+@Size(100)、category @Pattern 七选一（ELEC/WATER/GAS/STEAM/MATERIAL/ENV/OTHER，REQ-0003 验收 5）、dataType @Pattern 四选一（DOUBLE/INT/BOOL/ENUM）；tenantId 无校验注解（服务层会话覆盖，禁前端传参） |
| `backend/ot-admin/.../iot/mapper/IotDataDictMapper.java` | 新增 | 空 BaseMapper（iot_ 前缀表由 TASK-0002 插件自动注入租户条件，无 @Mapper 旁路需求） |
| `backend/ot-admin/.../iot/service/IotDataDictService.java` | 新增 | 服务接口：selectDictList / insertDict / updateDict / disableDict 四方法 |
| `backend/ot-admin/.../iot/service/impl/IotDataDictServiceImpl.java` | 新增 | ① 重复码双道闸（活跃行预检只按 data_code——租户条件由插件追加；DuplicateKeyException 兜底）均映射 4002，照 SysTenantServiceImpl 范式；② tenant_id 服务层从会话覆盖（`TenantContext.getTenantId()`）；③ 编辑白名单更新：data_code/data_type 创建后锁定（不一致或未携带一律 PARAM_ERROR 拒）、status/tenant_id/审计字段不进更新集；④ disableDict=置 status=0，不物理删除不逻辑删，可重复禁用；写方法一律 `@Transactional(rollbackFor = Exception.class)` |
| `backend/ot-admin/.../iot/controller/IotDataDictController.java` | 新增 | `@RequestMapping("/iot/dict")`，5 接口：GET list（iot:dict:list）/ GET {id}（iot:dict:query）/ POST（iot:dict:add）/ PUT（iot:dict:edit）/ DELETE {id}（iot:dict:remove，@OperateLog businessType=UPDATE——语义=禁用不冒充删除）；javadoc 注明 import 接口归 TASK-0009 |
| `backend/ot-admin/.../iot/controller/SysTenantController.java` | 修改 | 路径迁移：`@RequestMapping("/tenant")` → `@RequestMapping("/iot/tenant")`（brief 裁决 1 连带迁移）+ javadoc 同步 |
| `backend/ot-admin/.../iot/service/SysTenantService.java` | 修改 | 仅 javadoc 路径注释同步（POST /iot/tenant/{id}/init-admin） |
| `backend/ot-admin/.../iot/service/impl/SysTenantServiceImpl.java` | 修改 | 仅 javadoc 路径注释同步（PUT /iot/tenant/{id}/status） |
| `backend/ot-admin/src/test/.../iot/controller/IotDataDictControllerTest.java` | 新增 | 13 例：权限注解五接口反射断言、@OperateLog 三写操作（含 remove=UPDATE 断言）、list/{id}/POST/PUT/DELETE 路由、dataCode 缺失 400、dataType 越界 400、category 越界 400、重复 4002 透传、改码拒透传、禁用不存在透传 |
| `backend/ot-admin/src/test/.../iot/service/impl/IotDataDictServiceImplTest.java` | 新增 | 14 例：新增成功（请求携带 tenantId=999 不落库——会话覆盖佐证）/重复 4002 预检（wrapper 含 data_code）+兜底（DuplicateKeyException）/编辑改码拒/未携带码拒/dataType 改拒/未携带拒/不存在拒/白名单更新（status/tenantId/审计字段不进更新集、dataCode+dataType 库内值回写）/禁用仅写 status=0 且不触 deleteById/deleteByIds/重复禁用幂等/禁用不存在拒/列表四维过滤/无条件不过滤 |
| `backend/ot-admin/src/test/.../iot/controller/SysTenantControllerTest.java` | 修改 | 路径同步：全部 MockMvc 请求与 DisplayName `/tenant` → `/iot/tenant`（brief 裁决 1「相关测试路径同步」） |
| `frontend/src/api/iot/tenant-api.ts` | 修改 | 仅路径常量：六个请求路径 `/tenant...` → `/iot/tenant...` + 头注释（brief 声明文件，前端页面/菜单种子不受影响——走 sys_menu 动态注册） |
| `resources/sql/upgrade/20261121_iot_data_dict.sql` | 新增 | ① CREATE TABLE IF NOT EXISTS（brief DDL 附录 A 六逐字）；② 菜单种子 4018-4022 五行（menu_type=3，parent_id=4008，iot:dict:list/query/add/edit/remove，幂等 INSERT IGNORE）；③ sys_role_menu 授 role_id=1 五行；**不登记 iot:dict:import**（归 TASK-0009，维持「权限标识与接口清单一一对应」）；脚本头含用途/幂等口径/号段占用说明（4000-4007 归 0001、4008-4016 归 0055、4017 归 0002、本脚本占 4018-4022），语句区无 `--` 注释行 |

新增测试合计 **27 例**（服务 14 + 控制器 13），TDD 先红（编译失败：IotDataDict 等符号不存在，实测）后绿。

## 2. 验收命令原始输出

### 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
mvn install -pl ot-core -DskipTests:
[INFO] BUILD SUCCESS
[INFO] Total time:  4.053 s
[INFO] Finished at: 2026-11-22T00:00:34+08:00

mvn test -pl ot-admin（Results 段）:
[INFO] Results:
[INFO]
[INFO] Tests run: 1132, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:13 min
[INFO] Finished at: 2026-11-22T00:01:51+08:00
```

判定：**通过**。1132 = BASE（`88caf08`）基线 1105 + 新增 27（基线证据：BASE 状态下 target/surefire-reports 汇总 1105；brief 所写「基线 1063」为过时数字，见偏离 1）。逐类：IotDataDictServiceImplTest 14、IotDataDictControllerTest 13、SysTenantControllerTest 16（迁移路径后原量全绿）、SysTenantServiceImplTest 13、SaPermissionRegistryConsistencyTest 6（守卫测试确认 iot:dict:* 注解 ⊆ SQL 登记，含本任务新脚本）。

### 验收 2：升级脚本二次幂等 + 4018-4022 核对

第 1 次：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261121_iot_data_dict.sql`

```
OK(0)   ← SET NAMES utf8mb4
OK(0)   ← CREATE TABLE IF NOT EXISTS（首次建表）
OK(5)   ← INSERT IGNORE sys_menu 4018-4022（首插 5 行）
OK(5)   ← INSERT IGNORE sys_role_menu (1,4018..4022)（首插 5 行）
DONE
```

第 2 次（幂等）：

```
OK(0)
OK(0)
OK(0)   ← 5 行已存在 → 0 行
OK(0)   ← 5 行已存在 → 0 行
DONE    ← exit 0，无 SQLERROR
```

核对（原始输出）：

```
SELECT id, menu_name, menu_type, perms FROM sys_menu WHERE id BETWEEN 4018 AND 4022;
id	menu_name	menu_type	perms
4018	字典查询	3	iot:dict:list
4019	字典详情	3	iot:dict:query
4020	字典新增	3	iot:dict:add
4021	字典修改	3	iot:dict:edit
4022	字典删除	3	iot:dict:remove

SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id BETWEEN 4018 AND 4022 ORDER BY menu_id;
role_id	menu_id
1	4018
1	4019
1	4020
1	4021
1	4022

SHOW CREATE TABLE iot_data_dict（关键段）:
  `tenant_id` bigint(20) NOT NULL COMMENT '租户 ID',
  `data_code` varchar(64) NOT NULL COMMENT '数据编码（租户内唯一）',
  ...
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_data_code` (`tenant_id`,`data_code`),
  KEY `idx_tenant_category` (`tenant_id`,`category`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='业务数据字典表'
```

判定：**通过**（两次 DONE 无 SQLERROR；4018-4022 五行 perms 逐一对齐接口清单、(1,4018..4022) 授权在位；DDL 与 brief 附录 A 六逐字一致且 IF NOT EXISTS 幂等落盘；无 iot:dict:import 行）。

### 验收 3：dev 冒烟（启动：`cd backend && mvn spring-boot:run -pl ot-admin`，日志 /tmp/task0008_dev.log）

启动前端口探测：8098 无 LISTENING。就绪：`READY after ~10s (captchaImage HTTP 200)`。启动段：`Started OtAdminApplication in 10.35 seconds`；启动日志 grep ERROR 计数 = 0。

```
=== S1) admin 登录（dev /auth/login/test，tenantCode 缺省 -> default）
{"code":200,"data":{"nickname":"管理员","token":"6841db01-6114-4091-9eeb-3da9a817e2bf","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795277087071}

=== S2) GET /iot/tenant/list（路径迁移回归）→ HTTP_STATUS:200, total=8（含 default/e/f/tenanta/tenantb/tna/tenantb/tenantc，与 TASK-0003 冒烟遗留一致）

=== S2b) 旧路径 GET /tenant/list（迁移后应失效）
{"code":404,"data":null,"message":"请求路径不存在: /tenant/list"}   ← HTTP 信封 200、业务码 404（框架 NotHandler 约定），旧扁平路径已不注册

=== S3) POST /iot/dict 建字典 ELEC_CONS/DOUBLE
{"code":200,"data":null,"message":"操作成功","timestamp":1795277104847}

=== S4) 重复 data_code（预期 4002）
{"code":4002,"data":null,"message":"数据编码已存在: ELEC_CONS","timestamp":1795277111825}

=== S5a) PUT 编辑改 data_code（预期 400 拒）
{"code":400,"data":null,"message":"数据编码不允许修改","timestamp":1795277111955}

=== S5b) PUT 编辑改 data_name/default_unit（同码同型携带，预期 200）
{"code":200,"data":null,"message":"操作成功","timestamp":1795277116872}

=== S5c) GET /iot/dict/1 回读
{"code":200,"data":{"id":1,"tenantId":1,"dataCode":"ELEC_CONS","dataName":"总用电量","category":"ELEC","dataType":"DOUBLE","defaultUnit":"MWh","status":1,...}}   ← 名称/单位已改，编码未变

=== S5d) PUT 编辑改 data_type DOUBLE→INT（预期 400 拒，创建后锁定）
{"code":400,"data":null,"message":"数据类型不允许修改","timestamp":1795277124484}

=== S6a) DELETE /iot/dict/1（语义=禁用）
{"code":200,"data":null,"message":"操作成功","timestamp":1795277124568}

=== S6b) GET /iot/dict/1 禁用后详情（预期 status=0 且行仍在）
{"code":200,"data":{"id":1,"tenantId":1,"dataCode":"ELEC_CONS","dataName":"总用电量",...,"status":0,"deleted":0,...}}   ← 行在、status=0、deleted=0

=== S6c) 重复禁用（幂等，预期 200）
{"code":200,"data":null,"message":"操作成功","timestamp":1795277129644}

=== S7) 禁用后重建同码 ELEC_CONS（预期 4002，编码不复用）
{"code":4002,"data":null,"message":"数据编码已存在: ELEC_CONS","timestamp":1795277129740}

=== S8) DB 层核对行仍在且未逻辑删（只增不删）
SELECT id, tenant_id, data_code, data_name, status, deleted FROM iot_data_dict;
1	1	ELEC_CONS	总用电量	0	0

=== S8b) 列表过滤（admin 会话，default 租户内 1 行已禁用条目）
GET /iot/dict/list?category=ELEC   → total=1
GET /iot/dict/list?category=WATER  → total=0
GET /iot/dict/list?status=0        → total=1（禁用行）
GET /iot/dict/list?status=1        → total=0
GET /iot/dict/list?dataCode=ELEC   → total=1
GET /iot/dict/list?dataCode=NOSUCH → total=0
GET /iot/dict/list?dataName=用     → total=1（名称模糊）

=== S9a) POST /iot/tenant 建租户 T（TDengine 本地不可用 → 开通主流程不受影响，建库走扫描重试，TASK-0003 既有 fail-soft 行为）
{"code":200,"data":null,"message":"操作成功","timestamp":1795277155306}

=== S9b) GET /iot/tenant/list?tenantCode=T 定位 id → T_id=12

=== S9c) POST /iot/tenant/12/init-admin（password=前端同口径 SM4 密文，jshell+hutool+bcprov 预计算 Passw0rd123 的密文并本地往返验证）
{"code":200,"data":9,"message":"操作成功","timestamp":1795277161857}

=== S9d) operator 登录（dev /auth/login/test，tenantCode=T）
{"code":200,"data":{"nickname":"租户T管理员","token":"d74bb126-8800-4595-8df6-2ac21be8076c","tokenTimeout":86400,"userId":9,"username":"operator"},"message":"操作成功","timestamp":1795277162026}

=== S9e) operator(T) 建同码 ELEC_CONS（跨租户同码各自成功——uk_tenant_data_code 按 (tenant_id, data_code) 隔离，预期 200）
{"code":200,"data":null,"message":"操作成功","timestamp":1795277168450}

=== S10a) operator(T) GET /iot/dict/list → total=1
=== S10b) admin 会话 GET /iot/dict/list → total=1（default 租户自己的）

=== S10c) operator(T) list 记录明细（只见 T 的）
"records":[{"id":2,"tenantId":12,"dataCode":"ELEC_CONS","dataName":"T租户用电量","category":"ELEC","dataType":"DOUBLE","defaultUnit":"kWh","status":1,"createBy":9,...}]

=== S10d) admin list 记录明细（只见 default 的，无 T 的行）
"records":[{"id":1,"tenantId":1,"dataCode":"ELEC_CONS","dataName":"总用电量","category":"ELEC","dataType":"DOUBLE","defaultUnit":"MWh","status":0,...}]

=== S10e) admin GET 他租户字典 /iot/dict/2（预期 data=null）
{"code":200,"data":null,"message":"操作成功","timestamp":1795277175299}

=== S10f) operator 禁用他租户字典 /iot/dict/1（预期 400）
{"code":400,"data":null,"message":"字典不存在或已删除","timestamp":1795277175380}
```

判定：**通过**。brief 冒烟清单 10 步全覆盖（登录与 list 回归 / 建字典 / 重复 4002 / 改码拒+改名成功 / DELETE 禁用后详情 status=0 且行仍在 / 重建同码 4002 / category+status 过滤 / 租户 T+init-admin+operator 建字典 / admin 看不到 T 的（list 与 GET 双证据）/ operator 只见 T 的），另加 S2b/S5d/S6c/S10e/S10f 零成本佐证。dev 实例完成后停止（mvn 包装进程停止后残留 java 子进程 PID 78416 显式 taskkill），8098 复核 0 监听。

### 验收 4：prod 档真实启动回归

打包：`cd backend && mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS，`ot-admin/target/ot-admin-1.0.0.jar`（132,578,270 字节；-DskipTests 因全量测试已在验收 1 同一代码态覆盖，TASK-0001/0002/0003 先例）。

启动（沿 TASK-0001/0002/0003 已确认命令 + prod fail-fast 必需三密钥环境变量，取 dev 默认值非新密钥）：

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
启动前端口探测：8098 无 LISTENING
READY after ~25s (captchaImage HTTP 200)
2026-11-22T00:09:03.881+08:00  INFO 62928 --- [ot-framework] [main] net.otlab.admin.OtAdminApplication : Started OtAdminApplication in 34.93 seconds (process running for 35.942)
grep -ac "ERROR" /tmp/task0008_prod.log → 0

=== P1) prod POST /auth/login（真实口：验证码经 Redis 键 captcha:code:task0008a 预置 JSON 字符串 "AB12"（jshell+jedis 写入并回读验证），密码 admin123 经 hutool SM4-ECB 加密 hex 再 base64（往返验证）；不传 tenantCode → 走 default）
{"code":200,"data":{"nickname":"管理员","token":"2b28d8e3-d80f-4737-b63f-a2ff119b854f","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795277366587}

=== P2) prod GET /iot/tenant/list（带 token）→ HTTP_STATUS:200，total=9（含冒烟新建的租户 T）
{"code":200,"data":{"current":1,"pages":2,"records":[{"id":12,"tenantCode":"T",...},...],"size":5,"total":9},"message":"操作成功"}

=== P3) prod /auth/login/test 拒绝（dev-only 防线回归）
{"code":500,"data":null,"message":"测试登录接口仅限开发环境使用"}
```

判定：**通过**（Started 无 ERROR + 真实登录 200 + /iot/tenant list 200）。prod 实例完成后停止，8098 复核 0 监听（`netstat` grep LISTENING 无匹配，exit=1）。

### 验收 5：前端回归（路径迁移验证）

`cd frontend && pnpm build:test`：

```
✓ built in 2m 36s
EXIT=0
```

（chunk >500kB 警告为存有行为，非错误。）判定：**通过**（tenant-api.ts 路径常量改动编译通过）。

### 验收 6：改动范围核对

命令：`git diff --name-only 88caf08..HEAD`（HEAD=c1b2615）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/IotDataDictController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/SysTenantController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/IotDataDict.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/mapper/IotDataDictMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/IotDataDictService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/SysTenantService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/IotDataDictControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/SysTenantControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImplTest.java
frontend/src/api/iot/tenant-api.ts
resources/sql/upgrade/20261121_iot_data_dict.sql
```

核对结论：13 个文件全部落在 brief 声明内——`backend/.../module/iot/**` 8 个（字典域 5 新增 + SysTenantController 迁移 + 2 个仅注释路径同步）、`backend/.../src/test/java/net/otlab/admin/**` 3 个、`resources/sql/upgrade/20261121_iot_data_dict.sql` 新增、`frontend/src/api/iot/tenant-api.ts` 仅路径常量；未碰 ResultCode（4002 已存在复用）、application*.yml、frontend 其他文件、ingest/、workitems 库其他文件。**通过**。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| `iot:dict:import` 接口与权限行登记 | brief 明文：import 接口归 TASK-0009，届时登记权限行（本任务不登记，维持「权限标识与接口清单一一对应」；守卫测试 SaPermissionRegistryConsistencyTest 亦以此口径通过） |
| 导入逐行回执（TC-0003 步骤 3）与模板选码校验联动（TC-0003 步骤 4） | 归 TASK-0009（导入）与 TASK-0010（引用校验联动，CONV-01 注明）；本任务交付字典域本体 |
| 「禁用后既有引用可解析、新引用被拒」的端到端验证 | 引用方（模板数据项）尚不存在，TASK-0010 交付校验执行点后联测；本任务已交付禁用语义本体（status=0、行在、编码不复用——重建同码 4002 实证） |
| tenant_id 会话值的单测断言 | standalone 单测无 Sa-Token 会话，TenantContext.getTenantId() 返回 null——单测断言「请求携带 tenantId=999 不落库」（mass assignment 防护），会话真实注入由多租户插件 INSERT 补列 + dev 冒烟 S9e/S10c（operator 建 dict 落 tenantId=12）实证 |

## 4. 偏离计划之处

1. **基线数字更正（证据性，无行为影响）**：brief 验收 1 写「基线 1063」，实测 BASE=`88caf08`（TASK-0003 最终提交态）surefire 留痕基线为 1105（TASK-0003 报告「1105 = 1064 + 新增 41」同源）。本次 1132 = 1105 + 27，逐类可核。1063 为 brief 引用的过时数字，判定与结论不受影响（0 失败 0 错误 + BUILD SUCCESS 才是验收本体）。
2. **实体值域校验的取舍**：dataCode 未加「英文大写+下划线」@Pattern（原始来源口径）——brief 接口契约未要求，且 FEAT/REQ 正文未把编码字符集立为硬规则；仅 @NotBlank+@Size(64)。反向地，category 七选一与 dataType 四选一的 @Pattern 是**新增**的（REQ-0003 验收 5 要求取值与详细设计 §1.5A 一致，控制器层 @Valid 拦截越界值，S5 系列单测覆盖）。两处均为实体层校验的如实取舍，未改变 brief 的 5 接口契约与错误码口径。
3. **DELETE 接口的 @OperateLog businessType 用 UPDATE 而非 DELETE**：语义=禁用（不物理删除），操作日志如实记「修改」不冒充「删除」（审计口径与 REQ-0003「只增不删」对齐）；控制器测试显式断言此约定。若协调者倾向 DELETE（接口动词口径），一行可改。
4. **验收命令形态**（沿前序任务先例）：a) prod 打包 -DskipTests（全量测试已在同一提交态覆盖）；b) prod 启动补 SM4_KEY/CRYPTO_DATA_KEY/CRYPTO_INDEX_KEY 三环境变量（prod fail-fast 必需，取 dev 默认值，非新密钥）；c) 验收 6 的 diff 核对按先例先 git commit（c1b2615，main，注明 TASK-0008）再比较，提交严格限于上列 13 文件（任务前已存在的未跟踪 `.agents/` 未纳入）。
5. **冒烟额外佐证步**（超出 brief 清单但零成本）：S2b 旧路径 /tenant/list 失效（业务码 404）、S5d data_type 锁、S6c 重复禁用幂等、S10e 跨租户 GET data=null、S10f 跨租户禁用 400——均为迁移与隔离语义的补充证据。
6. **进程纪律**：dev 与 prod 两个自起实例均先探测 8098（无 LISTENING）再启动；dev 停止时 mvn 包装进程 killed 后残留 java 子进程（PID 78416）显式 taskkill 后复核 0 监听；prod 停止后复核 0 监听。

## 5. 遗留问题

1. **冒烟数据留痕**：租户 T（id=12）、operator（userId=9，role_code=tenant_admin_12）、字典 2 行（default 的 ELEC_CONS 禁用行 + T 的 ELEC_CONS）、Redis 验证码键（5 分钟自然过期）与 token（会话过期）——均为预期交付数据，重复验收可直接重跑（脚本与流程幂等；重建同码 4002 属预期行为）。
2. **data_code 编码规范强校验缺口**：当前仅必填+64 长度，未强制「英文大写+下划线」（原始来源口径未立为规则）。若后续模板/TDengine 标签需要编码安全约束（类似 SysTenant.tenantCode 的 @Pattern 防线），须先在 REQ/CONV 立规则再收紧，本任务不擅自扩权。
3. **编辑接口清空可选字段的能力**：category/defaultUnit 传 null 时不更新（MyBatis-Plus null 跳过，照 SysTenant 范式）——如需「置空」语义，前端传空串即可（实体无 @NotBlank），后端无需改动；记录口径防止误判为缺陷。
4. **TASK-0010 联动提示**：字典禁用后「既有引用可解析、新引用被拒」的校验执行点在模板数据项保存（REQ-0004 侧调用），本任务未提供查询接口之外的联动 API；TASK-0010 可直接复用 `IotDataDictService.getById`（插件租户隔离）/`getOne(data_code)` 语义——建议其按 data_code+status=1 校验（禁用行不在可选集）。

---

## 附 1：根因链（TDD 红→绿）

| # | 现象（红） | 根因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- | --- |
| R1 | 两个新测试类编译失败：`IotDataDict`/`IotDataDictMapper`/`IotDataDictServiceImpl`/`IotDataDictController` 找不到符号 | TDD 红灯预期：实体/Mapper/Service/Controller 未写 | 实现四个主类 + Service 接口 | 红→绿符合 TDD 流程；断言逻辑未放松 |
| R2 | 冒烟首轮 S3/S4「请求参数格式错误，请检查JSON格式是否正确」（业务码 400） | Git Bash 内联 JSON 中的中文经控制台代码页转码产生非法字节序列 | 冒烟请求体统一改为 UTF-8 文件 + `curl --data-binary @file`（S3 起全部成功） | 传输层编码问题，非接口缺陷；后续所有带中文的请求体（编辑/建租户/init-admin/operator 建字典）均走文件通道，全部按预期返回 |
| R3 | 冒烟 S2b 旧路径返回 HTTP 200 令人起疑 | 框架对未注册路径统一回 200 信封 + 业务码 404（`请求路径不存在: /tenant/list`） | 无需修复——读响应体业务码判定 | 与 GlobalExceptionHandler 既有约定一致（前序任务 403 同口径）；迁移证据以业务码 404 为准 |
| R4 | jshell 计算 SM4 密文首次失败：`SmUtil.sm4(...)` 结果为 null / `NoSuchAlgorithmException: SM4/ECB/PKCS5Padding` | 默认 JVM 缺 SM4 provider——hutool 的 SM4 依赖 BouncyCastle，hutool-all 不捆绑 bcprov | classpath 追加 .m2 中 bcprov-jdk18on-1.80.jar，显式 `SmUtil.sm4(key)` | 密文计算成功且本地往返解密验证（Passw0rd123 / admin123），与 SysUserServiceImpl.decryptSm4 同一 hutool 口径；init-admin 与 prod 登录均据此通过 |

## 附 2：契约自查

- 未派 subagent；未把 TASK 置「已完成」（详情状态留协调者收尾）；未改 workitems 库其他文件（仅本报告）。
- 改动 13 文件全部在 brief 声明内；ResultCode 未动（4002 复用 DUPLICATE_CODE）；application*.yml 未动；frontend 仅 tenant-api.ts 路径常量。
- 全量 1132 例 0 失败 0 错误（验收 1 原始输出贴回）；6 条验收命令全部执行且原始输出贴回；TDD 先红后绿（附 1 R1）。
- CONV-01 照做：iot_data_dict 独立成域（iot/dict 包），未触 sys_dict 任何代码；控制器与实体 javadoc 双注明两类字典不混用。
- 多租户纪律照做：tenant_id 服务层从会话覆盖、禁前端传参；预检/查询租户条件全由 TASK-0002 插件注入，未开任何 @Mapper 旁路；冒烟租户隔离四证据（list/GET/跨租户禁用拒）。
- 8098 纪律照做：启动前探测、自起实例完成后停止、复核 0 监听（dev/prod 各一次）。
- 路径裁决照做：iot 域统一 /iot/<resource>；SysTenantController /tenant → /iot/tenant 连带 frontend 一行常量与测试路径同步；旧路径实测失效。

---

# 6. 修订轮（验收退回 / 评审 I-1 修复与补充验收）

- 评审出处：`reports/TASK-0008-review.md`——Critical 0 / Important 1 / Minor 3；Important 1（insertDict mass assignment）进修订轮修复，Minor 3（分页无上限 / data_code 字符集未约束 / 租户隔离未落单测）按裁决记 deferred 归收尾 LOG，不进本轮。
- 修订轮提交：`d971c10`（main，注明 TASK-0008 修订轮，2 files / +117 −18）。

## 6.1 修复内容（为什么现在对了）

改动文件（严格限裁决范围）：

| 文件 | 说明 |
| --- | --- |
| `backend/ot-admin/.../iot/service/impl/IotDataDictServiceImpl.java` | ① `insertDict` 改字段白名单构建（照 SysTenantServiceImpl.updateTenant 既有范式）：请求仅可携带 dataCode/dataName/category/dataType/defaultUnit 五业务字段；status 强制 1（写侧仅 DELETE 禁用语义可置 0，实体不变量落地）；deleted 强制 0（直带 1 的幽灵行全 API 不可见且永久占用 uk_tenant_data_code——编码永不复用+无删路径=该编码不可见地报废且无法经 API 排查）；id/审计字段不复制（id 框架自增，createBy/createTime/updateBy/updateTime 由 MetaObjectHandler 回填——strictInsertFill 仅填空值，直落请求实体即被注入）；tenant_id 仍会话覆盖。② `updateDict` 收紧：dataCode/dataType 不再以库内值回写 no-op——不可变字段一律不进 UPDATE SET，仅 dataName/category/defaultUnit 可变（status/deleted/tenant_id/审计字段维持排除，status 变更只走 DELETE 禁用端点） |
| `backend/ot-admin/src/test/.../iot/service/impl/IotDataDictServiceImplTest.java` | +3 例（14→17）：`testInsertDict_RequestFieldsNotTrusted`（POST 携带 status=0/deleted=1/id=888/审计字段 → 落库 status=1/deleted=0、其余全 null）、`testInsertDict_GhostRequestThenDuplicateHitsPrecheckGate`（幽灵请求落库为正常行后同码重建走预检闸 4002、全程仅一笔触 insert=非幽灵占用）、`testUpdateDict_ReviveDisabledViaStatusNotEffective`（PUT 携带 status=1/deleted=1 复活禁用条目不生效）；`testUpdateDict_WhitelistFieldsSaved` 断言收紧（dataCode/dataType 由「库内值回写」改「不进更新集」） |

**为什么现在对了**：新增与编辑的落库实体全部由服务层白名单新建（`new IotDataDict()` + 显式赋值），请求体除五个业务字段外没有任何直达持久层的通道——status/deleted 的取值由服务层定死（1/0），id 与审计字段由框架管线填充，与 updateDict 的既有白名单纪律、SysTenantServiceImpl 范式完全对齐；「编码永不复用」的物理前提（uk_tenant_data_code）不再可能被客户端制造的幽灵行堵死。

## 6.2 TDD 红转绿记录

红（修复前，仅跑 IotDataDictServiceImplTest）：

```
[ERROR] Tests run: 17, Failures: 3, Errors: 0, Skipped: 0 ... <<< FAILURE!
[ERROR] ...testInsertDict_RequestFieldsNotTrusted:155 status 请求携带 0 不得落库：新增恒启用（status 只能经 DELETE 禁用语义置 0） ==> expected: <1> but was: <0>
[ERROR] ...testInsertDict_GhostRequestThenDuplicateHitsPrecheckGate:174 落库非幽灵行（deleted=0，API 可见可管理） ==> expected: <0> but was: <null>
[ERROR] ...testUpdateDict_WhitelistFieldsSaved:300 data_code 已锁死为库内值，不进更新集（UPDATE SET 不含不可变字段） ==> expected: <null> but was: <ELEC_CONS>
[INFO] BUILD FAILURE
```

（第三例红首跑时 ghost 请求未显式携带 deleted=1/status=0，补齐攻击字段后维持红——修复验证用例如实代表攻击向量。）

绿（修复后，IotDataDictServiceImplTest + IotDataDictControllerTest）：

```
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.controller.IotDataDictControllerTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.service.impl.IotDataDictServiceImplTest
[INFO] Tests run: 30, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

## 6.3 补充验收 1：全量复跑

`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`：

```
mvn install -pl ot-core -DskipTests → BUILD SUCCESS
mvn test -pl ot-admin（Results 段）:
[INFO] Tests run: 1135, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Finished at: 2026-11-22T00:29:44+08:00
```

判定：**通过**。1135 = 基线 1105 + 首轮 27 + 修订轮 3（数学闭环；0 失败 0 错误）。

## 6.4 补充验收 2：dev 冒烟复核三步

启动前端口探测：8098 无 LISTENING。就绪：`READY after ~10s (captchaImage HTTP 200)`；`Started OtAdminApplication in 12.49 seconds`；启动日志 ERROR 计数 = 0。admin 经 /auth/login/test 登录（R0，code=200）。

```
=== R1) POST /iot/dict 携带 {"status":0,"deleted":1,...}（WATER_CONS/DOUBLE，预期 200 且落库强制 status=1/deleted=0）
{"code":200,"data":null,"message":"操作成功","timestamp":1795278621173}

=== R1b) GET /iot/dict/list?dataCode=WATER_CONS 回读（预期 status=1/deleted=0）
"records":[{"id":3,"tenantId":1,"dataCode":"WATER_CONS","dataName":"用水量","category":"WATER","dataType":"DOUBLE","defaultUnit":"t","status":1,"createBy":1,"createTime":"2026-11-22 00:30:21","deleted":0,"updateBy":1,"updateTime":"2026-11-22 00:30:21"}]

=== R2) 同码重复 POST WATER_CONS（预期 4002，走预检闸=可见活跃行占用，非幽灵）
{"code":4002,"data":null,"message":"数据编码已存在: WATER_CONS","timestamp":1795278629022}

=== R3) PUT /iot/dict 携带 {"status":1,"deleted":1} 尝试复活已禁用 ELEC_CONS(id=1)（信封 200，状态不变）
{"code":200,"data":null,"message":"操作成功","timestamp":1795278629137}

=== R3b) GET /iot/dict/1 回读（预期 status 仍 0，未被复活）
{"code":200,"data":{"id":1,"tenantId":1,"dataCode":"ELEC_CONS","dataName":"总用电量","category":"ELEC","dataType":"DOUBLE","defaultUnit":"MWh","status":0,"createBy":1,"createTime":"2026-11-22 00:05:05","deleted":0,"updateBy":1,"updateTime":"2026-11-22 00:30:29"},"message":"操作成功"}
```

DB 层佐证（全表无幽灵行）：

```
SELECT id, tenant_id, data_code, status, deleted FROM iot_data_dict ORDER BY id;
id	tenant_id	data_code	status	deleted
1	1	ELEC_CONS	0	0
2	12	ELEC_CONS	1	0
3	1	WATER_CONS	1	0
SELECT COUNT(*) AS ghost_rows FROM iot_data_dict WHERE deleted <> 0;
ghost_rows
0
```

判定：**通过**（三步全过；请求携带的 status/deleted 在新增与编辑两侧均被白名单拦截，落库行全部 deleted=0）。dev 实例用毕即停（mvn 包装进程 killed 后残留 java 子进程 PID 71692 显式 taskkill），8098 复核 0 监听。

## 6.5 修订轮契约自查

- 改动严格限裁决范围：仅 `IotDataDictServiceImpl.java` + `IotDataDictServiceImplTest.java` 两文件（git show d971c10 --name-only 可核）；未碰控制器/实体/SQL/frontend/workitems 库其他文件。
- 一个 commit 注明 TASK-0008 修订轮（d971c10）。
- 自起实例（修订轮 dev ×1）用毕即停，8098 复核 0 监听。
- Minor 3 项未动（按裁决记 deferred，归协调者收尾进 LOG）。
- 未派 subagent；未把 TASK 置「已完成」；本节为本报告唯一追加改动。

