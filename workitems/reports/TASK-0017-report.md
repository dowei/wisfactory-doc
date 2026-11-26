# 实施报告：TASK-0017（iot_gateway 网关 CRUD 与在线状态回写）

- 执行者：implementer（brief：reports/TASK-0017-brief.md）
- BASE：`eb01ba2` → HEAD：`042d13f`（main，1 个提交，注明 TASK-0017）
- 总体结论：**5 条验收全部通过**（全量回归 1609 = 基线 1551 + 新增 58，0 失败 0 错误；升级脚本两次执行均 DONE 幂等；dev 冒烟全步通过（含 IP 白名单外 403）；prod 档启动段 0 ERROR + /iot/gateway/list 200 + 内部回写 200；改动范围 25 文件全落在声明内）

---

## 0. 权威核对（pypdf 提取，brief「必读」要求先行）

- **附录 A p30 `iot_gateway` DDL**：逐字提取核对通过——列 tenant_id/gateway_code(64)/gateway_name(100)/instance_id/dept_id/online_status/last_heartbeat_time/heartbeat_period(默认 30)/version(32)/config_version/ip_addr(64)/status 与三索引 `uk_tenant_gateway_code (tenant_id, gateway_code)`、`idx_tenant_instance (tenant_id, instance_id)`、`idx_online (online_status, last_heartbeat_time)` 全部照抄进升级脚本（MySQL 8 InnoDB 键长均可行，无 0011 idx_ancestors 式前缀降级需求，头注如实登记「与上游差异：无」）。
- **§3.1 p21 接口行**：`/api/v1/iot/gateway` 实测 **4 行**——GET /list（iot:gateway:list，含在线状态与心跳时间）、POST（add）、PUT（edit，gateway_code 不可改）、DELETE /{id}（软删，有点位拒绝）。**brief 记「五行」与 PDF 实测不符**（§3.1 无 gateway detail 行）——按 PDF 落地 4 端点，菜单五行照 brief（iot:gateway:query 预登记不挂端点，见 §4.1）。
- **p21-22 内部段**：§3.1 p21-22 实测无「心跳回写」独立接口行；内部接口规范在 §3 p20（/internal/v1 前缀与管理面物理隔离、系统间鉴权=内部令牌+IP 白名单）与 §3.2/3.3 各段（/ingest/v1/auth/mqtt、/internal/v1/conﬁg/* 均标「内部令牌」）。brief 裁决的 `POST /internal/v1/gateway/heartbeat`（body 携带 tenantCode、字段组白名单五字段）即本任务落点，路径符合 p20 前缀规范。

## 1. 改动文件清单（25 个 = 验收 5 git diff 实测）

**新增（24）**

| 文件 | 内容 |
| --- | --- |
| `resources/sql/upgrade/20261122_iot_gateway_domain.sql` | iot_gateway 建表（p30 逐字）+ sys_menu 4041-4045 五行（parent_id=4012）+ role 1 授权；CREATE TABLE IF NOT EXISTS + INSERT IGNORE 幂等 |
| `…/iot/gateway/entity/IotGateway.java` | 网关实体（p30 DDL 十三业务列；BaseEntity 审计五列） |
| `…/gateway/mapper/IotGatewayMapper.java` | 空 BaseMapper（多租户插件注入会话条件） |
| `…/gateway/mapper/IotGatewayCodeMapper.java` | uk 双道闸第一道旁路预检（含已删行，@InterceptorIgnore + 显式 (tenant_id, gateway_code)，照 0012 IotInstanceCodeMapper 范式） |
| `…/gateway/mapper/IotGatewayCodeRow.java` | 占用行最小映射（id/gateway_code/deleted） |
| `…/gateway/dto/IotGatewayCreateForm.java` | 创建表单（gatewayCode @Pattern [A-Za-z0-9_]/@Size 64、gatewayName、instanceId、deptId、heartbeatPeriod；**不携带回写字段组五字段与 status/tenantId**） |
| `…/gateway/dto/IotGatewayUpdateForm.java` | 编辑表单（gatewayCode 仅一致性核对；白名单 gatewayName/instanceId/deptId/heartbeatPeriod/status） |
| `…/gateway/dto/GatewayHeartbeatForm.java` | 回写批量 typed 载荷（tenantCode + items） |
| `…/gateway/dto/GatewayHeartbeatItem.java` | 回写条目（gatewayCode 定位键 + 字段组五字段） |
| `…/gateway/auth/InternalAuthChecker.java` | 内部认证检查器接口（check(HttpServletRequest)，失败抛 BusinessException——完整令牌链 0019/0020 的升级接缝） |
| `…/gateway/auth/StaticTokenInternalAuthChecker.java` | 占位实现：ot.gateway.internal-token（header X-Internal-Token 常量时间比对；**空配置 fail-closed 拒绝全部**）+ ot.gateway.internal-ip-whitelist（默认 127.0.0.1 代码默认值，OtIpUtils.getClientIp 受 behind-proxy 开关控制） |
| `…/gateway/auth/InternalAuthInterceptor.java` | 拦截器层校验入口（brief「校验在过滤器/拦截器层」） |
| `…/gateway/auth/GatewayInternalAuthConfig.java` | 拦截器注册（仅 /internal/v1/gateway/**，管理面不受影响） |
| `…/gateway/service/IotGatewayService.java` | 服务接口（list/add/edit/remove/writeHeartbeats 五方法，照实例域范式不继承 IService） |
| `…/gateway/service/impl/IotGatewayServiceImpl.java` | 服务实现（双道闸 4002/字符集长度拒/instance+dept 归属校验/period 默认 30/编辑改码拒+白名单更新集/删除 guard 调用点/**回写五字段更新集+显式 (tenant_id, gateway_code) 定位+租户解析**） |
| `…/gateway/service/GatewayReferenceGuard.java` | 删除保护引用计数接缝（countChannels/countPoints——0018/0021 按表扩展） |
| `…/gateway/service/impl/PlaceholderGatewayReferenceGuard.java` | 占位实现恒零引用（TODO 注释登记 0018/0021 接线义务） |
| `…/gateway/controller/IotGatewayController.java` | 管理面四端点（/iot/gateway/list POST PUT DELETE/{id}；@SaCheckPermission 四权限码；写操作 @OperateLog） |
| `…/gateway/controller/IotGatewayInternalController.java` | 内部回写端点（POST /internal/v1/gateway/heartbeat；**@Anonymous 无 Sa-Token**；原始 JSON 层字段组白名单校验（顶层键/条目键越界直接拒）+ 类型归一化（字符串/毫秒时间戳）+ 控制器层显式调用 InternalAuthChecker；**无 @OperateLog**） |
| 测试 5 个：`…/iot/gateway/service/impl/IotGatewayServiceImplTest.java`（26）、`…/controller/IotGatewayControllerTest.java`（12）、`…/controller/IotGatewayInternalControllerTest.java`（10）、`…/auth/StaticTokenInternalAuthCheckerTest.java`（7）、`…/config/IotGatewayMenuPermissionGuardTest.java`（3） | 合计 58 用例 |

**修改（1）**

| 文件 | 内容 |
| --- | --- |
| `backend/ot-admin/src/main/resources/application.yml` | **仅新增 `ot.gateway.internal-token` 一个键**（diff 实测 +2 行：gateway 段两行——键与行内注释），env 注入 OT_GATEWAY_INTERNAL_TOKEN，空值 fail-closed |

未触碰：frontend/、ingest/、instance/template 域（只读复用 IotModelInstanceMapper/SysDeptMapper/SysTenantMapper）、字典域、workitems 库其他文件（TASK-0017 详情未回写——brief「收尾回写：不要回写 TASK 详情（协调者收尾）」）。

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0（-q 静默）。ot-admin 全量（BASE 基线经 stash 复跑实测为 **1551**，与 brief 基线一致）：

```
[INFO] Results:
[INFO]
[WARNING] Tests run: 1609, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] BUILD SUCCESS
[INFO] Total time:  54.447 s
```

1609 = 基线 1551 + 新增 58（服务 26 + 管理面控制器 12 + 内部控制器 10 + 认证检查器 7 + 菜单守卫 3）；3 skipped 为既有「红线二-中台 scope（占位——TASK-0051 接线后启用）」套件，非本任务引入。**TDD 红转绿记录**：首轮运行 6 红 → 逐一定位根因修复后全绿（§2.6）。

### 2.2 验收 2：升级脚本二次幂等 + 表结构/菜单核对（sql-exec.sh，127.0.0.1:3307）

**第一次执行**（`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_iot_gateway_domain.sql`）：

```
OK(0)    ← SET NAMES
OK(0)    ← CREATE TABLE IF NOT EXISTS iot_gateway
OK(5)    ← INSERT IGNORE sys_menu 4041-4045
OK(5)    ← INSERT IGNORE sys_role_menu
DONE
```

**第二次执行**（幂等复核）：

```
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

**SHOW CREATE TABLE iot_gateway**（节选，与 p30 逐字一致）：

```
`gateway_code` varchar(64) NOT NULL COMMENT '网关编码（租户内唯一，用作 MQTT 主题与设备身份）',
`online_status` tinyint DEFAULT '0' COMMENT '在线状态 0 离线 1 在线',
`last_heartbeat_time` datetime DEFAULT NULL COMMENT '最后心跳时间',
`heartbeat_period` int DEFAULT '30' COMMENT '心跳周期（秒）',
`config_version` bigint DEFAULT NULL COMMENT '当前生效配置版本号',
`ip_addr` varchar(64) DEFAULT NULL COMMENT '最近连接来源 IP',
`status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用',
PRIMARY KEY (`id`),
UNIQUE KEY `uk_tenant_gateway_code` (`tenant_id`,`gateway_code`),
KEY `idx_tenant_instance` (`tenant_id`,`instance_id`),
KEY `idx_online` (`online_status`,`last_heartbeat_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 ... COMMENT='网关表'
```

**菜单 4041-4045 核对**：

```
id	parent_id	menu_name	menu_type	perms
4041	4012	网关查询	3	iot:gateway:list
4042	4012	网关详情	3	iot:gateway:query
4043	4012	网关新增	3	iot:gateway:add
4044	4012	网关修改	3	iot:gateway:edit
4045	4012	网关删除	3	iot:gateway:remove
role_id	menu_id
(1,4041)...(1,4045) 五行
```

### 2.3 验收 3：dev 冒烟（mvn spring-boot:run，OT_GATEWAY_INTERNAL_TOKEN=t0017-internal-token-xyz；日志 backend/target/task0017_dev.log）

启动前端口探测 8098 无 LISTENING；`Started OtAdminApplication in 10.0 seconds`；启动段 ERROR 计数 = **0**。冒烟脚本（backend/target/t0017_smoke.py，Git Bash curl 中文 body 有 GBK 编码问题，改用 python urllib UTF-8 提交）原始输出：

```
=== S1) admin dev login -> 200 token= 5fec126b...
=== S2) 创建网关（挂实例 F01=2 + 部门 1；二号仅必填字段 period 缺省）
S2a POST GW_T0017_01 period=15 -> HTTP 200 code 200 | 操作成功
S2b POST GW_T0017_02 仅必填 -> HTTP 200 code 200 | 操作成功
=== S3) 创建负路径
S3a 重复编码 -> 期望4002 -> code 4002 | 网关编码已存在: GW_T0017_01
S3b 连字符编码 -> 期望400 -> code 400 | 网关编码仅允许字母/数字/下划线（作 MQTT 主题与设备身份段）
S3c 65字符编码 -> 期望400 -> code 400 | 网关编码长度不能超过 64
S3d 实例不存在 -> 期望400 -> code 400 | 实例不存在或已删除: 9999
S3e 实例禁用(id=4 WS02) -> 期望400 -> code 400 | 实例已禁用，不可挂载网关: F01.WS02
S3f 部门不存在 -> 期望400 -> code 400 | 部门不存在或已删除: 9999
S3g 周期 0 -> 期望400 -> code 400 | 心跳周期仅允许 1-86400 秒
=== S4) 编辑（改码拒 / 白名单改名+启停）
S4a list ids: {'GW_ASCII_T1': 1, 'GW_T0017_01': 2, 'GW_T0017_02': 3}
S4a 改码 -> 期望400 -> code 400 | 网关编码不允许修改: GW_T0017_01
S4b 改名+周期60 -> 期望200 -> code 200 | 操作成功
S4c 启停 status=0 -> 期望200 -> code 200 | 操作成功
S4d 恢复 status=1 -> 期望200 -> code 200 | 操作成功
=== S5) list 含在线状态与心跳时间列
S5 list total: 2 cols: [configVersion, createBy, createTime, deleted, deptId, gatewayCode, gatewayName,
      heartbeatPeriod, id, instanceId, ipAddr, lastHeartbeatTime, onlineStatus, status, tenantId, updateBy, updateTime, version]
S5 onlineStatus 列在响应中: True
=== S6) 内部心跳回写（正确令牌；二号含毫秒时间戳）
S6a 正确令牌回写 2 条 -> 期望200 data=2 -> HTTP 200 code 200 | 操作成功
=== S7) 回写负路径
S7a 顶层越界键 status -> 期望400 -> code 400 | 回写字段越界（顶层不支持的键: [status]）——仅允许 tenantCode/items
S7b 条目越界键 gatewayName -> 期望400 -> code 400 | 回写字段越界（条目不支持的键: [gatewayName]）——回写字段组固定为 [gatewayCode, configVersion, ipAddr, version, onlineStatus, lastHeartbeatTime]
S7c 错令牌 -> 期望401 -> code 401 | 内部令牌无效
S7d 缺令牌 -> 期望401 -> code 401 | 内部令牌无效
S7e 租户不存在 -> 期望400 -> code 400 | 租户不存在: ghost_tenant
S7f 未知网关编码 -> 期望400 -> code 400 | 网关编码不存在或已删除: [GW_GHOST]
=== S8) 删除（guard 本期恒零引用放行）
S8a DELETE GW_T0017_02 -> 期望200 -> code 200 | 操作成功
S8b 重复删除 -> 期望400 -> code 400 | 网关不存在或已删除
```

**S9) IP 白名单外拒**（经本机 LAN 接口 192.168.31.103 发起——来源 IP 非 127.0.0.1，白名单默认仅含 127.0.0.1）：

```
HTTP 200 {"code":403,"data":null,"message":"来源 IP 不在内部接口白名单","timestamp":1795309798971}
```

**S10) 心跳更新落库 DB 核对**（回写五列断言 + 白名单编辑核对 + 软删核对）：

```
id	tenant_id	gateway_code	gateway_name	instance_id	dept_id	online_status	last_heartbeat_time	heartbeat_period	version	config_version	ip_addr	status	deleted
2	1	GW_T0017_01	网关冒烟T17一号改名	2	1	1	2026-11-22 09:10:00	60	v1.0.0	3	192.168.31.103	1	0
3	1	GW_T0017_02	网关冒烟T17二号	null	null	1	2026-11-22 09:10:00	30	v1.0.1	null	192.168.31.104	1	1
```

（id=2 五列全落且 gateway_name=「改名」白名单生效；id=3 毫秒时间戳 1795309800000 正确归一化为 09:10:00、config_version 未携带保持 null、deleted=1 软删。）

用毕即停：taskkill（PID 47100）→ 8098 复核无 LISTENING（仅 TIME_WAIT 残留）。判定：**通过**。

### 2.4 验收 4：prod 档真实启动回归

打包 `mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（ot-admin-1.0.0.jar）。启动（沿 0011/0012 确认命令 + 三密钥 + 本任务内部令牌 env）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  OT_GATEWAY_INTERNAL_TOKEN=t0017-internal-token-xyz \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
Started OtAdminApplication in 17.887 seconds (process running for 18.589)
=== prod 全程 ERROR 计数 ===
0
=== prod 真实登录（Redis 预置 captcha:code:task0017a="AB12"（Jackson 带引号，EX 600）+ SM4 admin123 密文，jshell ROUNDTRIP="admin123" 往返验证） ===
LOGIN HTTP 200 code 200 token 19d6eaec...
=== prod GET /iot/gateway/list?pageNum=1&pageSize=10 ===
LIST HTTP 200 code 200 total 2
codes ['GW_ASCII_T1', 'GW_T0017_01']
cols_present True          ← onlineStatus / lastHeartbeatTime 列在响应中
（total=2：软删行 GW_T0017_02 被 @TableLogic 正确过滤）
=== prod POST /internal/v1/gateway/heartbeat（正确令牌，五字段组） ===
HEARTBEAT HTTP 200 {"code":200,"data":1,"message":"操作成功"}
=== prod 回写 DB 核对 ===
id	gateway_code	online_status	last_heartbeat_time	version	config_version	ip_addr
1	GW_ASCII_T1	1	2026-11-22 09:30:00	v1.0.0	5	127.0.0.1
```

用毕即停：taskkill（PID 16656）→ 8098 复核无 LISTENING。判定：**通过**（启动段+全程 0 ERROR、list 200、内部回写 prod 档可用）。

### 2.5 验收 5：改动范围核对 `git diff --name-only eb01ba2..HEAD`

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/auth/GatewayInternalAuthConfig.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/auth/InternalAuthChecker.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/auth/InternalAuthInterceptor.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/auth/StaticTokenInternalAuthChecker.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/controller/IotGatewayController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/controller/IotGatewayInternalController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/dto/GatewayHeartbeatForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/dto/GatewayHeartbeatItem.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/dto/IotGatewayCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/dto/IotGatewayUpdateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/entity/IotGateway.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/mapper/IotGatewayCodeMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/mapper/IotGatewayCodeRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/mapper/IotGatewayMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/GatewayReferenceGuard.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/IotGatewayService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/impl/IotGatewayServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/impl/PlaceholderGatewayReferenceGuard.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/gateway/auth/StaticTokenInternalAuthCheckerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/gateway/config/IotGatewayMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/gateway/controller/IotGatewayControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/gateway/controller/IotGatewayInternalControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/gateway/service/impl/IotGatewayServiceImplTest.java
resources/sql/upgrade/20261122_iot_gateway_domain.sql
```

核对结论：25 文件 = `iot/gateway/**` 主代码 18（声明内）+ `iot/**` 测试 5（声明内）+ 升级脚本（声明内）+ application.yml（声明内，diff 实测仅 +2 行 = 唯一授权键及其注释）。**通过**。提交：`042d13f`（main，注明 TASK-0017）。

### 2.6 TDD 红转绿记录（首轮 6 红 → 根因修复）

首轮运行（网关域 51 用例中的 6 个红，其余全绿）：

```
IotGatewayInternalControllerTest.testHeartbeat_InvokesAuthChecker        JSON path "$.code" expected:<200> but was:<500>
IotGatewayInternalControllerTest.testHeartbeat_CoercionAndOptionals      同上 500
IotGatewayInternalControllerTest.testHeartbeat_ControllerLevelCheck…     同上 500
IotGatewayInternalControllerTest.testHeartbeat_TopLevelExtraKeyRejected  TooFewActualInvocations: check() wanted 2 but was 1
IotGatewayServiceImplTest.testCreate_ActiveRowOccupied4002               expected: <4002> but was: <400>
IotGatewayServiceImplTest.testCreate_DeletedRowOccupied4002 / …Fallback  同上
IotGatewayServiceImplTest.testList_Filters                               编码过滤参数断言失败（值实为 %GW\_F01%）
```

根因与修复（每条附「为什么现在对了」）：

1. **500（NPE）**：控制器 parseForm 向 `form.getItems().add(...)` 加条目时 GatewayHeartbeatForm.items 为 null——修复：parseForm 内显式 `form.setItems(new ArrayList<>())`（typed 载荷的集合初始化归一化点移到控制器解析处；服务层照旧全量校验非空，语义不变）。
2. **4002 用例得 400**：重复编码用例未 stub 实例 Mapper，而服务实现「归属校验先于唯一性预检」（照实例域实现顺序），`instanceId=2` 走到 validateInstance 即抛 400——修复：用例聚焦唯一性闸，置 instanceId/deptId=null 绕过归属分支（实现顺序与实例域同构属有意为之：先便宜校验后落库闸）。
3. **LIKE 参数断言**：SqlLikeUtils.escape 会包裹 `%..%` 且转义下划线（`GW_F01`→`%GW\_F01%`）——修复：断言改为「任一参数值包含 GWF01」（用不含下划线的查询词），并补注释说明防通配注入包裹（断言放松仅此一处，语义仍是「过滤参数确实进 wrapper」）。
4. **checker 调用次数**：控制器原「先 parseForm 后 check」，越界载荷在控制器级 check 之前被拒 → 拦截器只调了 1 次——修复：控制器改为**认证先于一切载荷处理**（未认证请求不享受解析行为的安全顺序）；修复后拦截器 1 次 + 控制器 1 次 = 2 次，与服务语义一致。

## 3. 未做项与原因

1. **detail 端点未建**：§3.1 p21 网关段实测仅 4 行（无 GET /{id} detail），brief 裁决「无则 list/detail 合并按 list 权限」——管理面 4 端点落地，前端详情走 list 数据；`iot:gateway:query`（4042）按 brief 五行菜单要求预登记、不挂端点（守卫测试将此差异显式固化，防止误删或误挪用）。
2. **LWT 与 graceful 区分未实现**：归 TASK-0020 裁决（brief 明文）——上游 DDL 无独立区分字段（p30 iot_gateway 仅 online_status/last_heartbeat_time），本任务保证回写字段组五字段透传正确（单测+冒烟固化），0020 若引入区分字段走同接口扩展条目键白名单。
3. **删除保护计数实现未接线**：iot_channel（0018）/ iot_point（0021）两表未建——guard 接缝（GatewayReferenceGuard 双计数接口 + 调用点 + 拒绝分支）就位且被单测固化（计数>0 即拒），占位实现恒零引用，接线义务登记 0018/0021 brief。
4. **完整网关令牌链未实现**：静态令牌 + IP 白名单为 brief 裁决占位；完整令牌链归 0019（凭据域令牌）/0020（LWT/准入）——InternalAuthChecker 接口即升级接缝，届时替换实现类不改调用点。

## 4. 偏离计划之处

1. **brief「/api/v1/iot/gateway 五行」与 PDF 实测不符**：p21 实测 4 行（§1 已述），按 PDF 实现；brief 记载差异在此留痕，随上游修订清单一并核对。
2. **application.yml 仅新增一个键，IP 白名单走代码默认值**：brief 同时要求「仅允许新增 ot.gateway.internal-token 一个键」与「ot.gateway.internal-ip-whitelist 配置项（默认 127.0.0.1）」——调和：白名单以 `@Value("${ot.gateway.internal-ip-whitelist:127.0.0.1}")` 代码默认值交付（不占 yml 键），部署可经环境变量 OT_GATEWAY_INTERNAL_IP_WHITELIST 覆盖，语义与 brief 默认值要求一致。
3. **内部控制器不注入 ObjectMapper bean**（实现细节）：Spring 上下文无 ObjectMapper bean（首启 APPLICATION FAILED 实测），改为直接处理 `Map<?,?>`（JSON 反序列化产物键恒为 String）——更少依赖、行为等价；单测同步调整。
4. **回写接口不拦「网关禁用」**：REQ-0009 验收 3「禁用网关后上报被拒」的拦截点在接入服务准入（/ingest/v1/auth/mqtt，归 0020）——回写接口保持放行（禁用网关的 LWT/离线态仍应可落，保状态准确）；「租户禁用拒」已实现（单测+冒烟 S7e 佐证租户不存在拒；租户禁用拒有单测）。brief 测试口径列「租户禁用拒」而未列「网关禁用拒」，与本实现一致。
5. **网关列表未接数据权限裁剪**：详细设计 §3 p21 通用规范称「未标注的查询类接口同样走数据权限过滤」，但本任务 brief 决策清单（「逐条结论，不再讨论」）未含数据权限项，且网关可见域语义（instance_id/dept_id 双锚点如何映射 VisibleScope）无 ADR 裁决——按 brief 落地纯过滤分页，登记遗留待裁决（见 §5.2）。
6. **冒烟通道偏离 curl**：Git Bash curl 对中文 body 走 GBK 编码致 JSON 解析失败（S2 首轮实测），改用 python urllib UTF-8 提交；SM4 密文计算需 jshell 带 BouncyCastle classpath（hutool SM4 依赖 BC）。均为本机环境事项，不影响交付物。
7. **TDD 顺序**：本轮为「实现与测试同批交付 + 首轮运行红转绿」（§2.6），非严格逐用例先红后绿——六红的根因修复均有「为什么现在对了」留痕。

## 5. 遗留问题

1. **dev 库冒烟留痕**（是否清理由协调者裁决）：iot_gateway 三行——GW_ASCII_T1（id=1，prod 档回写过心跳：online=1/09:30:00/v1.0.0/config_version=5）、GW_T0017_01（id=2，改名「网关冒烟T17一号改名」period=60，dev 档心跳五列全落）、GW_T0017_02（id=3，已逻辑删 deleted=1）。重跑冒烟需换码（同码撞 4002/已被删除记录占用属预期）。prod Redis 种子 captcha:code:task0017a 已 EX 600 自动清理。
2. **网关列表数据权限裁剪缺位**（§4.5）：建议协调者裁决——若按 §3 通用规范接线，需先定「网关可见域 = instance_id ∈ VisibleScope（dept 子树展开）」口径（网关无 createByOnly 形态语义），归数据权限补线任务或 0020 一并处理。
3. **接线义务（已按 brief 登记，此处汇总）**：①删除 guard 计数实现 → 0018（iot_channel 按 gateway_id）/0021（iot_point 归属列按其建表裁决），替换 PlaceholderGatewayReferenceGuard 不改调用点；②内部认证升级 → 0019（凭据域完整令牌链）/0020（LWT 与 graceful 区分、准入拦截 /ingest/v1/auth/mqtt），替换 StaticTokenInternalAuthChecker 不改接口；③回写接口条目键白名单若 0020 增字段（如区分 LWT），在 IotGatewayInternalController.ITEM_KEYS 与 GatewayHeartbeatItem 同步扩展（两层白名单一处接口一处 typed）。
4. **iot:gateway:query 预留**：4042 已登记无端点——前端任务启用详情弹层时挂 GET 端点即可；期间该权限码授予不产生任何接口访问面（无害），SaPermissionRegistryConsistencyTest 与域内守卫测试均兼容此形态。
5. **prod 档 0 ERROR 的取证面**：全程 ERROR 计数 0（含登录/业务段）；日志在 backend/target/task0017_prod.log（不入库，target 目录）。

---

## 6. 验收命令回执汇总

| # | 命令/动作 | 结果 |
| --- | --- | --- |
| 1 | mvn install ot-core + mvn test ot-admin | 1609 = 1551+58，0 失败 0 错误，BUILD SUCCESS |
| 2 | 升级脚本二次执行 + SHOW CREATE TABLE + 4041-4045 | 两次 DONE；表结构与 p30 逐字一致；5 行菜单 + 5 行超管授权 |
| 3 | dev 冒烟（S1-S10 序列） | 全步通过（含越界字段拒/错令牌 401/IP 白名单外 403/五列落库断言） |
| 4 | prod 档启动回归 | 0 ERROR；/iot/gateway/list HTTP 200 code 200；内部回写 200 |
| 5 | git diff --name-only eb01ba2..HEAD | 25 文件全落声明内；提交 042d13f |
