# TASK-0019 实施报告：iot_channel_credential 凭据接口与 SM4 加密

- 派发 brief：TASK-0019-brief.md（BASE `fa12df9`）
- 提交：`2ff499b`（main，注明 TASK-0019；单提交 22 文件）
- 总体结论：**5 条验收全部通过**（全量回归 1723 = 基线 1673 + 新增 50，0 失败 0 错误；升级脚本两次执行均 DONE 幂等 + 表结构/4051-4052 核对通过；dev 冒烟 11 步全绿含 DB/审计无明文 grep 核对（首轮发现 dev DEBUG 请求体日志泄明文 → toString 源端治理 → 修订轮复跑 0 泄漏）；prod 档干净单轮真实登录 200 + credential/list 200 + 内部拉取 200 + 全程 0 ERROR；提交 22 文件全落四条声明路径内）。

## 0. TDD 红转绿记录

先写 4 个测试文件（服务 24 + 管理控制器 11 + 内部控制器 5 + 审计 6 + 守卫 3），跑 `-Dtest=IotCredential*` 捕获 **RED**（生产类不存在，test-compile 全量「找不到符号」失败，RED 输出留档 backend/target/ 临时终端记录）；随后实现凭据域 12 个主文件 + gateway/auth 完整化接线，复跑 **GREEN**（49/49）。修订轮 toString 治理补第 50 用例（`testToStringExcludesSensitiveFields`）先行入审计套件（RED：无排除注解时 toString 含明文）再实现排除，终码套件 7/7。过程中两处测试自身缺陷如实登记偏离 3/4。

## 1. 改动文件清单（提交 2ff499b，22 文件全落 brief 四条声明路径）

**新增——凭据域主代码（backend/ot-admin/src/main/java/net/otlab/admin/module/iot/credential/）**

| 文件 | 说明 |
| --- | --- |
| `controller/IotCredentialController.java` | 管理面四接口 /iot/channel/credential：list（iot:channel:list，掩码）/write（iot:credential:edit，@OperateLog INSERT **saveParam=false**）/view/{id}（iot:credential:view，@OperateLog OTHER **saveResponse=false**）/revoke（iot:credential:edit，@OperateLog UPDATE） |
| `controller/IotCredentialInternalController.java` | 内部拉取 GET /internal/v1/credential/pull（@Anonymous 无 Sa-Token；InternalAuthChecker 拦截器+控制器双调用，照 0017 范式；无 @OperateLog） |
| `domain/CredentialTypes.java` | cred_type 字典四值 DEVICE_TOKEN/MQTT_PASSWORD/HTTP_SECRET/CERT（§1.3A p8） |
| `domain/CredentialMask.java` | 掩码器：≥12 前四\*\*\*\*后四 / 8-11 首四\*\*\*\* / <8 纯占位——只要露段隐藏段恒 ≥4 字符 |
| `dto/CredentialWriteForm.java` | 写入表单（channelId/credType/credential 明文 @NotBlank @Size(1000)；**credential 字段 @ToString.Exclude**——防 DEBUG 请求体日志泄明文） |
| `dto/CredentialItemVO.java` | 列表项（id/channelId/credType/credentialMask/keyVersion/status；**结构上无密文/明文字段**——单测反射断言固化） |
| `dto/CredentialViewVO.java` | 明文查看 VO（唯一管理面明文出口；credential 字段 @ToString.Exclude） |
| `dto/CredentialPullVO.java` | 拉取 VO（channelId/credType/ciphertext/keyVersion；无明文字段；ciphertext 字段 @ToString.Exclude） |
| `entity/IotChannelCredential.java` | iot_channel_credential 实体（extends BaseEntity；ciphertext/key_version/status） |
| `mapper/IotChannelCredentialMapper.java` | 空 BaseMapper（多租户插件注入） |
| `service/IotCredentialService.java` / `service/impl/IotCredentialServiceImpl.java` | 编排：SM4 {v1} 加密落库（fail-closed：明文 {v1} 前缀拒存）、同 channel 同 cred_type 轮换 upsert（key_version 递增、DuplicateKey 转轮换）、列表解密掩码化（解密失败降级纯占位）、吊销 status=0 白名单更新、内部拉取显式条件定位 + 通道/凭据启用校验 + 租户一致断言 + 统一 403 |

**修改——gateway/auth 完整化接线（brief 明文要求，4 文件，功能改动仅 1 行）**

| 文件 | 说明 |
| --- | --- |
| `gateway/auth/GatewayInternalAuthConfig.java` | 拦截器模式 `/internal/v1/gateway/**` → **`/internal/v1/**`**（0019 完整化：同一认证器统一收口全部内部端点，credential 域零注册接入；javadoc 同步） |
| `gateway/auth/InternalAuthChecker.java` | javadoc 更新（统一覆盖口径；升级路径=替换实现类，网关设备令牌链归 0020/0027） |
| `gateway/auth/InternalAuthInterceptor.java` | javadoc 更新（同上） |
| `gateway/auth/StaticTokenInternalAuthChecker.java` | javadoc 更新（静态令牌即凭据拉取准入，照 brief 裁决） |

**新增——测试（backend/ot-admin/src/test/java/net/otlab/admin/module/iot/credential/，5 套件 50 用例）**

| 文件 | 用例数 | 要点 |
| --- | --- | --- |
| `service/impl/IotCredentialServiceImplTest.java` | 24 | SM4 {v1} 往返/随机 IV、明文 {v1} 前缀 fail-closed 拒存、新建 key_version=1、轮换 1→2 不新增行 + 吊销复活、key_version 递增口径（null/非数值回落）、并发撞键转轮换、列表掩码边界 + VO 结构反射断言（无 ciphertext/credential/plaintext 字段）、解密失败降级、view 解密往返、吊销白名单（不触密文/租户/类型）、拉取 8 态（成功/通道缺失/通道禁用/凭据缺失/未知类型/已吊销/租户不一致/参数缺失——除 400 外统一 403） |
| `controller/IotCredentialControllerTest.java` | 11 | 四端点权限注解契约、@OperateLog 四动作 + saveParam/saveResponse 敏感标志反射断言、list 响应 jsonPath 无 ciphertext/credential 键、写入 bean 校验、异常透传 |
| `controller/IotCredentialInternalControllerTest.java` | 5 | @Anonymous 无 SaCheckPermission 无 @OperateLog、checker 双调用、令牌无效服务不触达、参数缺失 400、403 透传 |
| `audit/IotCredentialAuditNoPlaintextTest.java` | 7 | **grep 级审计无明文**：真实 OperateLogAspect 经 AspectJProxyFactory 织入真实控制器，捕获落库 SysOperateLog 全列断言——写入 param 列 NULL、查看 response 列 NULL、失败路径 failReason/param 无明文、{v1} 前缀禁令（param/response/failReason 动态列）、toString 源端治理断言 |
| `config/IotCredentialMenuPermissionGuardTest.java` | 3 | SQL 4051-4052 ↔ 控制器权限码双向核对（iot:channel:list 跨脚本并集核对 0018 登记） |

**新增——升级脚本**：`resources/sql/upgrade/20261122_iot_credential_domain.sql`（附录 A p35 DDL 逐字 + 菜单 4051-4052 + 超管授权，幂等 + 头注全套纪律）。

## 2. 验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0。ot-admin 全量（**终码复跑**，BASE 基线 1673）：

```
[INFO] Tests run:  7, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.audit.IotCredentialAuditNoPlaintextTest
[INFO] Tests run:  3, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.config.IotCredentialMenuPermissionGuardTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.controller.IotCredentialControllerTest
[INFO] Tests run:  5, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.controller.IotCredentialInternalControllerTest
[INFO] Tests run: 24, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.service.impl.IotCredentialServiceImplTest
[INFO] Results:
[WARNING] Tests run: 1723, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS（Total time: 01:10 min）
```

（1723 = 基线 1673 + 凭据域 50；skipped 3 为基线既有。toString 治理前的首轮全量为 1722 = 1673 + 49。）

### 2.2 验收 2：升级脚本二次幂等 + 表结构 + 4051-4052

第一次执行 `sql-exec.sh ../../resources/sql/upgrade/20261122_iot_credential_domain.sql`：

```
OK(0)
OK(0)
OK(2)
OK(2)
DONE
```

第二次执行：

```
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

核对（SHOW CREATE TABLE 摘录 + 菜单/授权）：

```
  UNIQUE KEY `uk_channel_cred_type` (`channel_id`,`cred_type`),
  KEY `idx_tenant_channel` (`tenant_id`,`channel_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='通道凭据表'

id	parent_id	menu_name	menu_type	perms
4051	4013	凭据写入	3	iot:credential:edit
4052	4013	凭据查看	3	iot:credential:view
role_id	menu_id
1	4051
1	4052
```

### 2.3 验收 3：dev 冒烟（mvn spring-boot:run，OT_GATEWAY_INTERNAL_TOKEN=t0019-internal-token-xyz；日志 backend/target/task0019_dev.log）

启动前 8098 无 LISTENING；`Started OtAdminApplication in 13.741 seconds`；启动段 ERROR 计数 = 0。冒烟脚本 backend/target/t0019_smoke.py **修订轮**（终码，编码 GW_T0019R1/CH_T0019R_M1）原始输出：

```
=== S1) admin dev login -> 200 token= abd40594...
=== S1b) 无 view 权限测试用户 login（仅 iot:credential:edit） -> 200 token= 015c31ec...
=== S2) 建网关 GW_T0019R1 + MQTT 通道 CH_T0019R_M1 ===
S2a POST gateway -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2b 网关 id = 9
S2c POST MQTT channel -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2d 通道 id = 9
=== S3) admin 写凭据 MQTT_PASSWORD（明文只走 HTTPS 入参，服务端 SM4 加密） ===
S3a POST credential MQTT_PASSWORD -> HTTP 200 code 200 | 操作成功  [期望200] OK
=== S4) 同通道同类型重复写入=轮换（key_version 递增，不新增行） ===
S4a POST credential again -> HTTP 200 code 200 | 操作成功  [期望200] OK
=== S5) 列表只回类型/掩码/key_version/状态（永不回密文/明文） ===
S5a list rows = 1 | code 200
S5b row: {"channelId": 9, "credType": "MQTT_PASSWORD", "credentialMask": "T19-****wxyz", "id": 3, "keyVersion": "2", "status": 1}
S5c 断言: 掩码 = T19-****wxyz [期望 T19-****wxyz] OK
S5d 断言: keyVersion = 2 (轮换后期望 2) OK
S5e 断言: 无 ciphertext/credential 键 = True | 无明文/密文子串 = True OK
=== S6) admin view/{id} 解密展示（唯一明文出口） ===
S6a view -> code 200 | credential = T19-mqtt-secret-abcd1234wxyz
S6b 断言: 明文还原 = True OK
=== S7) 无 view 权限用户直调 view -> 403 + 审计（PermissionDeniedAuditListener 链） ===
S7a limited view -> HTTP 200 code 403 | 无权访问该资源  [期望403] OK
=== S7b 同用户持 edit 权限写 DEVICE_TOKEN（edit/view 分权旁证，供 S10 拉取） ===
S7b limited POST credential DEVICE_TOKEN -> HTTP 200 code 200 | 操作成功  [期望200] OK
=== S8) 内部拉取负路径（无令牌/错令牌 -> 401；未知 credType -> 403） ===
S8a no token -> HTTP 200 code 401 | 内部令牌无效  [期望401] OK
S8b wrong token -> HTTP 200 code 401 | 内部令牌无效  [期望401] OK
S8c unknown credType -> HTTP 200 code 403 | 凭据不存在或已吊销  [期望403] OK
S8d 通道不存在 -> HTTP 200 code 403 | 通道不存在、已删除或已禁用  [期望403] OK
=== S9) admin 吊销 MQTT_PASSWORD 凭据 -> 立即生效（拉取随即 403） ===
S9a DELETE credential -> HTTP 200 code 200 | 操作成功  [期望200] OK
S9b pull revoked -> 403 -> HTTP 200 code 403 | 凭据不存在或已吊销  [期望403] OK
S9c list after revoke status=0 -> （脚本展示步格式缺陷未打印，status=0 以 S12a DB 核对为准）
=== S10) 内部拉取带正确令牌 -> 密文 + key_version 返回（服务端不解密传输） ===
S10a pull -> code 200 | ciphertext = {v1}WijBPRzRR+fVkO5u... | keyVersion = 1
S10b 断言: {v1} 密文 = True | 无明文字段 = True OK
```

DB 核对（S12，sql-exec.sh）：

```
S12a 凭据行：
id	channel_id	cred_type	cipher_head	key_version	status	deleted
4	9	DEVICE_TOKEN	{v1}WijBPR	1	1	0
3	9	MQTT_PASSWORD	{v1}Zt8ThD	2	0	0
S12b 库内无明文 grep：plaintext_leak_rows = 0（ciphertext 列不含明文子串）
S12c sys_operate_log「通道凭据」四动作审计行：
id	user_name	module	business_type	status	url	param_col	resp_col
235	admin	通道凭据	1	1	/iot/channel/credential	<NULL>	{"code":200,...}
236	admin	通道凭据	1	1	/iot/channel/credential	<NULL>	{"code":200,...}
237	admin	通道凭据	7	1	/iot/channel/credential/view/3	{}	<NULL>
239	smoke_cred_t19	通道凭据	1	1	/iot/channel/credential	<NULL>	{"code":200,...}
240	admin	通道凭据	2	1	/iot/channel/credential/3	{}	{"code":200,...}
S12c 全表 grep：audit_leak_rows = 0（sys_operate_log 全表 param/response 列不含明文/密文/{v1}）
S12d 权限校验 403 审计行：
238	smoke_cred_t19	status=0	/iot/channel/credential/view/3	缺少权限：iot:credential:view；无此权限：iot:credential:view
```

应用日志无明文核对（REQ-0011 验收 4）：

```
grep -ac "T19-mqtt-secret\|T19-device-token" task0019_dev.log  ->  0
grep -ac "Written.*{v1}\|Read.*{v1}" task0019_dev.log          ->  0
（DEBUG 请求体绑定日志现为：Read ... to [CredentialWriteForm(channelId=9, credType=MQTT_PASSWORD)]——明文字段被 toString 排除）
```

用毕即停：taskkill → 8098 复核无 LISTENING。判定：**通过**。

### 2.4 验收 4：prod 档真实启动回归（干净单轮）

打包 `mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（ot-admin-1.0.0.jar）。启动（沿 0017 确认命令 + 三密钥 + 内部令牌 env）：

```
Started OtAdminApplication in 21.124 seconds (process running for 22.005)
=== prod 全程 ERROR 计数 ===
0
=== prod 真实登录（Redis 预置 captcha:code:task0019d="AB15"（JSON 带引号，EX 600）+ SM4 admin123 密文） ===
LOGIN HTTP 200 code 200 message 操作成功
LOGIN token 7176bbb3...
=== prod GET /iot/channel/credential/list?channelId=9 ===
LIST HTTP 200 code 200
LIST rows: [{"channelId": 9, "credType": "MQTT_PASSWORD", "credentialMask": "T19-****wxyz", "id": 3, "keyVersion": "2", "status": 0}, {"channelId": 9, "credType": "DEVICE_TOKEN", "credentialMask": "T19-****7766", "id": 4, "keyVersion": "1", "status": 1}]
LEAK-CHECK: no ciphertext key = True | no credential key = True | no plaintext value = True | no cipher value = True | ALL-CLEAN = True
=== prod GET /iot/channel/list?gatewayId=9（旁证） ===
LIST HTTP 200 code 200 total 1
=== prod GET /internal/v1/credential/pull?channelId=9&credType=DEVICE_TOKEN（带内部令牌） ===
PULL HTTP 200 code 200 keyVersion 1 cipherHead {v1}WijB... no-plaintext-field True
```

用毕即停：taskkill → 8098 复核无 LISTENING。判定：**通过**（启动+全程 0 ERROR、真实登录 200、credential/list 200、拉取密文 200）。

### 2.5 验收 5：改动范围核对

brief 原命令 `git diff --name-only fa12df9..HEAD` 实测 51 文件——**BASE 快照过期所致**：`fa12df9` 之后、本任务开工之前，TASK-0056（前端第一组 CRUD 页，提交 `424031b`）已先行入库 29 个 frontend 文件；该提交非本任务产物、brief 声明「不碰 frontend/」亦未触碰（其工作区另有 3 个他人未提交的 vue 修改，保持原样未动）。**本任务提交 `2ff499b` 的 diff（`git diff --name-only 424031b..HEAD`）恰好 22 文件，全部落在四条声明路径内**（credential/** 主代码 12 + 测试 5 + gateway/auth 4 + resources/sql/upgrade 1；frontend/、ingest/、通道域、配置快照链路零触碰；workitems 库零改动，本报告文件除外；TASK 状态未改）。

## 3. 未做项与原因

1. **网关设备令牌链（校验令牌归属通道）未实现**（brief 明文）：本期内部拉取准入=InternalAuthChecker 静态令牌 + IP 白名单（0017 既有实现），channelId/credType 显式条件定位 + 通道/凭据启用校验 + 租户一致断言。「校验令牌归属通道」的设备令牌→网关→通道绑定归 0020/0027 接线（届时替换 checker 实现类，调用点不扩散——接口即接缝），brief 明文登记遗留。
2. **凭据不进配置快照与 config.changed 载荷——本期无快照链路可改**（ADR-0006 排除项）：快照链路归 0024 交付，本期登记口径（升级脚本头注 + 拉取控制器 javadoc 均已写入），0024 brief 需承接该排除项。
3. **无密钥自动轮换**（REQ-0011 非目标）：一期单密钥（ot.crypto.data-key），key_version 按写入/轮换代次递增（brief 裁决口径）；多密钥并行选钥为密钥服务替换位预留（§1.3A p8「按此字段选钥解密」的完整语义二期承接）。
4. **列表/拉取无分页**：§3.1 续 p23 未约定分页（对照「数据字典分页列表」的显式措辞）；凭据列表按通道查、一行/类型（uk_channel_cred_type 物理上限），返回普通 List。
5. **通道删除对凭据的级联/保护未加**（通道域只读约束）：删通道不校验其下凭据——软删后凭据行保留但拉取 403（通道不存在）、管理 list/view 亦按通道校验拒（数据不可达，无泄漏面）。若要求「有凭据的通道不可删」，属 0018 通道域 guard 扩展，归 0021/0024 裁决（遗留 5）。

## 4. 偏离计划之处

1. **dev 冒烟首轮发现 DEBUG 请求体日志泄明文 → toString 源端治理（偏离修复，非缩水）**：首轮冒烟后 grep 应用日志发现 3 处明文命中——dev 档 `org.springframework.web: DEBUG` 时 Spring 的 RequestResponseBodyMethodProcessor 以 toString 记录请求体绑定结果，Lombok @Data 的 toString 带出 credential 明文（REQ-0011 验收 4「应用与访问日志无明文」）。修复：`CredentialWriteForm.credential` / `CredentialViewVO.credential` / `CredentialPullVO.ciphertext` 三字段 **@ToString.Exclude**（源端治理，不动日志配置——prod 本无该 DEBUG 且未来 dev 调级不复发；Jackson/审计切面均走 getter 不受影响）；补第 50 用例固化。修订轮复跑 grep = 0。**为什么现在对了**：泄漏点在「对象字符串化出口」而非日志级别，排除字段后任何 DEBUG 级/任何出口（含未来新出的调试日志）都不会带出敏感值。
2. **写入接口 @OperateLog(saveParam=false)、查看接口 saveResponse=false**：brief 要求四动作全审计 + 日志无明文（硬项）。框架切面将全部方法实参 JSON 序列化进 param 列，而框架脱敏白名单（DataMaskingUtils）不含 credential 类字段名（ot-core 不在本任务改动范围，不能扩白名单）——「入参含明文的写入」参数列整列不落（saveParam=false），「响应含明文的查看」响应列整列不落（saveResponse=false）；审计上下文损失由 url（view 的凭据 id 在 URI 上）+ 凭据行 create_by/update_by 审计列 + sys_operate_log 时间关联恢复。冒烟 S12c 实测：写入行 param=<NULL>、查看行 response=<NULL>、全表 grep 0 泄漏。
3. **掩码取自明文前四后四（解密后掩码化）**：brief「掩码格式自定如 abcd\*\*\*\*wxyz 前四后四」未指明取材。取密文前四后四无识别价值（{v1} 开头 + 随机 IV 每轮变化）；取明文前四后四为业界惯例（GitHub/AWS 同款），服务列表页操作者核对凭据。泄漏上界已收口：仅长 ≥8 才露首四、≥12 才露尾段，露段时隐藏段恒 ≥4（`CredentialMask` 边界单测固化）；短凭据（<8）纯占位。解密失败降级纯占位（列表不崩不泄底）。该口径已写入类 javadoc，评审如要求更严（全占位）仅改一处。
4. **吊销后重复写入=复活启用**：brief 未定义「写入已吊销凭据」的语义。按「写入/轮换」upsert 语义处理为重新配发（rotate 时 status 置 1）——否则该通道该类型永久死锁（uk 不含 deleted、吊销非软删，不复活就只能换 cred_type 或直改库）。测试/冒烟固化，评审如要求「吊销后写入须先显式恢复」为一处状态判断的改动。
5. **测试自身缺陷两处（TDD 过程如实登记）**：(a) MP 3.5.17 `ServiceImpl.getOne` 实测走 selectOne 语义与桩预期不符 → 服务改用 `getBaseMapper().selectOne(...)` 直查（0017 tenantMapper.selectOne 同款范式，uk 保证 ≤1 行，桩化确定性更强，语义不变）；(b) 审计测试 {v1} 禁令最初覆盖 content 列，而 content=开发者撰写的 @Operation 静态摘要合法指称 {v1} 格式名——禁令收窄到 param/response/failReason 三个动态列（brief 硬验收口径本就是「请求参数/返回值字段」）。
6. **prod 回归三轮完成**：首轮回归全绿但日志留有 1 条 ERROR——为验证码一次性消费后我的复登尝试被拒（业务异常，非应用缺陷）；为交付「全程 0 ERROR」的干净证据重启两轮（第 2 轮再被我的附加脚本污染一次），第 3 轮单脚本单登录跑完整序列（2.4 节原始输出）。冒烟脚本 t0019_prod.py 的 LEAK-CHECK 表达式首轮过严（掩码 T19- 前缀误报）已修正。

## 5. 遗留问题

1. **网关设备令牌链（TASK-0020/0027）**：拉取准入目前为静态令牌（服务间信任级），0020/0027 落地「令牌→网关→通道」绑定后应替换 `StaticTokenInternalAuthChecker` 实现类（不改调用点），并在拉取链路加网关维度隔离（伪造他网关 channelId 的 403 已由统一拒绝语义覆盖）。
2. **配置快照排除项（TASK-0024）**：凭据不进快照体与 config.changed 载荷（ADR-0006）——0024 交付时需在快照 Schema/发布链路显式排除并落「是否有凭据、凭据类型」标记（REQ-0011 核心规则；TC-0006 步骤 5 的快照 SHA-256 校验用例届时承接）。
3. **通道删除的凭据保护/级联（归 0021 或协调者裁决）**：见未做项 5——如需「有凭据的通道不可删」或删通道级联吊销凭据，属通道域 guard 扩展（本任务受「不碰通道域」约束未做）。
4. **key_version 多密钥并行语义（二期/密钥服务）**：一期按写入/轮换代次递增（brief 裁决）；密钥服务替换位就位后，key_version 应承载「解密选钥」语义（§1.3A p8），需配套存量密文迁移（ADR-0006 影响范围已登记）。
5. **dev 库冒烟留痕（是否清理由协调者裁决）**：iot_gateway 一行（GW_T0019R1，id=9，未删）+ iot_channel 一行（CH_T0019R_M1，id=9，未删）+ iot_channel_credential 两行（id 3 MQTT_PASSWORD key_version=2 status=0 已吊销 / id 4 DEVICE_TOKEN status=1）+ sys_operate_log 审计行（225-240 段）+ 冒烟专用账号（sys_user `smoke_cred_t19` id=9 / sys_role `冒烟凭据edit无view角色T19` role_code=smoke_cred_t19，仅持 4051）+ sys_menu 4051-4052/授权（升级脚本产物，保留）。首轮冒烟另留 GW_T0019_01(id=8)/CH_T0019_M1(id=8)/凭据 id 1-2（同口径）。prod Redis 种子 captcha:code:task0019a/d 已 EX 600 自动清理。
6. **前端凭据管理入口未做**（brief 改动范围不含 frontend/）：凭据按钮权限行（4051-4052）与后端接口已就绪，页面挂接归前端任务；ADR-0006「无 view 权限的角色不可见入口」的前端菜单裁剪随该任务落地（后端直调 403 + 审计已兜底）。

（报告完。五项验收证据原始输出见第 2 节；本地留档：backend/target/task0019_fulltest.log、task0019_dev.log、task0019_prod.log、t0019_smoke.py、t0019_prod.py。）

---

## 修订轮（TASK-0019 验收退回一轮，§13 diff 评审）

- 评审报告：TASK-0019-review.md（Critical 0 / Important 1 / Minor 7；Minor 7 条不动记 deferred）
- 修复范围：仅 `IotCredentialServiceImpl.rotate()`（I-1 并发轮换丢失更新）+ `ResultCode` 追加 `VERSION_CONFLICT(5001)`（裁决授权）+ 服务测试同口径调整；提交 `46d9391`（main，注明 TASK-0019 修订轮，3 文件 +101/-39）。

### R-1 必修项修复内容（I-1）

| 项 | 修复 | 实现 |
| --- | --- | --- |
| I-1 并发轮换丢失更新 | `rotate()` 改**原子条件 UPDATE 乐观并发闸**（不引入 @Version）：`UPDATE iot_channel_credential SET ciphertext=<新密文>, status=1, key_version = IFNULL(key_version,0) + 1 WHERE id=? AND key_version=<前置读值> AND deleted=0`——影响行数 0 = 版本已被并发改写 → `BusinessException(5001, "凭据已被并发轮换，请刷新后重试")` | `LambdaUpdateWrapper`：`setSql("key_version = IFNULL(key_version,0) + 1")`（**服务端行内自增**，非客户端赋值）+ `.eq(id)` + `.eq(前置读 key_version)`（legacy NULL 行走 `.isNull` 等价条件）；deleted=0 由 @TableLogic 对 wrapper UPDATE 自动追加（MP AbstractMethod sqlWhereEntityWrapper 含 logic-delete 条件，回源核实） |
| 错误码 | `ResultCode` 追加 `VERSION_CONFLICT(5001, "版本冲突")`（详细设计 §3.4 分段：5001=配置与发布版本冲突段；语义全局，抛出方携带具体提示） | ot-core 常量追加，javadoc 注明 TASK-0019 修订轮先行使用 |

**取舍说明（密文与掩码，裁决二选一）**：取「密文在原子 UPDATE 内一并落」形态——SET 语句同帙携带新密文与 key_version 自增，行内原子生效；**掩码无回填需求**：本设计掩码非库列，由列表服务层「读时解密→掩码化」派生（`CredentialMask`），任何时刻读到的 (密文, key_version) 与掩码天然一致，不存在「掩码落后于密文」的窗口（`nextKeyVersion` 客户端递增函数随本修复删除，M-1 所指边界随之消失；IFNULL 兼容 legacy null/非数值行首次轮换归一 1，与原回落语义一致）。

### R-2 红转绿记录（TDD）

新增 2 用例（`IotCredentialServiceImplTest`，原 `testKeyVersionIncrement` 随 `nextKeyVersion` 删除移除，净 +1）：

**RED（临时还原旧 updateById 实现重演——首轮 RED 捕获被共享工作区并行任务（0020/0009 中途态）的编译阻断干扰，故做 A/B 重演取干净证据）：**

```
[ERROR] Tests run: 25, Failures: 4, Errors: 0 -- IotCredentialServiceImplTest
[ERROR]   testRotate_ConcurrentConflict5001      （Expected BusinessException to be thrown, but nothing was thrown——正是 I-1 缺陷写照：丢失更新静默双成功）
[ERROR]   testRotate_TwoCallsSamePreRead_OneWins （同前置读两调用双双成功——无版本闸）
[ERROR]   testWrite_RotateUpdatesExistingRow / testWrite_DuplicateKeyFallsBackToRotate（旧 updateById 路径断言不适配）
```

**GREEN（新实现恢复后同套件复跑）：**

```
[INFO] Tests run: 51, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential（5 套件合计）
[INFO] BUILD SUCCESS
```

（凭据域 51 = 修订前 50 - 1 删除 + 2 新增；新用例断言：条件 UPDATE 影响行数 0 → 5001 文案精确匹配；同前置读双调用先 200 后 5001；两个 UPDATE wrapper 均带服务端自增 SET 片段 `key_version = IFNULL(key_version,0) + 1` 与同一前置读条件——「不丢更新」的机制本体。）

### R-3 补充验收 1：全量复跑 `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0（含 VERSION_CONFLICT 常量）。ot-admin 全量（终码，本轮树内含并行任务 0020/0009 已交付测试）：

```
[INFO] Tests run:  7, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.audit.IotCredentialAuditNoPlaintextTest
[INFO] Tests run:  3, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.config.IotCredentialMenuPermissionGuardTest
[INFO] Tests run: 11, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.controller.IotCredentialControllerTest
[INFO] Tests run:  5, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.controller.IotCredentialInternalControllerTest
[INFO] Tests run: 25, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.credential.service.impl.IotCredentialServiceImplTest
[INFO] Results:
[WARNING] Tests run: 1755, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS（Total time: 01:08 min）
```

（1755 = 修订轮终码基线 1723 + 凭据域净增 1 + 树内并行任务新增测试 31；0 失败 0 错误。）

### R-4 补充验收 2：dev 冒烟复核（backend/target/t0019_rev_smoke.py；日志 task0019_rev_dev.log）

启动前 8098 无 LISTENING；`Started OtAdminApplication in 11.56 seconds`；启动段 ERROR 计数 = 0。原始输出：

```
=== R1) admin dev login -> 200 token= 012b0481...
R1b 前置状态: 凭据 id = 3 | key_version = 43 | status = 1
=== R2) 连续两次正常轮换 -> key_version 递增两次（服务端自增原子 UPDATE） ===
R2a 第 1 次轮换 -> HTTP 0.1 code 200 | 操作成功 [期望200] OK
R2b 第 1 次后 key_version = 44 （前置 43 → 递增 1） OK
R2c 第 2 次轮换 -> HTTP 0.1 code 200 | 操作成功 [期望200] OK
R2d 第 2 次后 key_version = 45 （ 44 → 递增 1，两次轮换共 +2） OK
=== R3) 模拟并发：另一管理员先提交轮换（独立 DB 会话 FOR UPDATE 持行锁 12s，持锁期间覆盖
      轮换代次 key_version='42' 后提交——复刻「前置读之后、UPDATE 之前版本被改写」窗口；
      探针（innodb_lock_wait_timeout=1 的 FOR UPDATE）确认锁在位后才发请求） ===
R3-pre 行锁已由并发会话持有（探针 SQLERROR 确认） = True
R3a 并发轮换 -> HTTP 200 code 5001 | 凭据已被并发轮换，请刷新后重试 | 耗时 8.9 s（行锁等待至并发方提交后条件不命中） [期望5001] OK
R3b 锁持会话已提交（模拟的并发管理员 key_version=42 落库；sql-exec 输出:）
   OK(0) / id 3 / SLEEP(12) 0 / OK(1) / OK(0) / DONE
R3c 冲突后 key_version = 42 （并发方代次 42 胜出，败者未覆盖——不丢更新、不双成功） OK
=== R4) 刷新后重试（按提示重新读版本再轮换） -> 恢复 200 ===
R4a 重试轮换 -> HTTP 0.1 code 200 | 操作成功 [期望200] OK
R4b 重试后 key_version = 43 （42 → 43，恰 +1） OK
```

DB/审计终态核对（sql-exec.sh）：

```
id	cred_type	key_version	status	cipher_head
3	MQTT_PASSWORD	43	1	{v1}sDNG
（并发冲突拒绝亦留审计——sys_operate_log 通道凭据段最新 3 行：）
270	admin	通道凭据	1	0	凭据已被并发轮换，请刷新后重试	<NULL>（param 列 NULL——审计无明文在失败路径同样成立）
271/269	admin	通道凭据	1	1	<NULL>	<NULL>（成功轮换）
```

用毕即停：taskkill → 8098 复核无 LISTENING。判定：**补充验收两条全部通过**（连续两次正常轮换 key_version 递增两次；锁持模拟并发 → 5001 + 败者不覆盖 + 刷新重试恢复恰 +1）。

### R-5 修订轮改动清单与登记

- 改动文件（3）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/credential/service/impl/IotCredentialServiceImpl.java`（rotate 原子条件 UPDATE + 5001 拒绝 + javadoc；nextKeyVersion 删除）、`backend/ot-core/src/main/java/net/otlab/core/result/ResultCode.java`（VERSION_CONFLICT 5001）、`backend/ot-admin/src/test/java/net/otlab/admin/module/iot/credential/service/impl/IotCredentialServiceImplTest.java`（-1/+3 用例同口径）。提交 `46d9391`，3 文件 +101/-39，全落「credential 域 + ResultCode（裁决授权）」。
- Minor 7 条不动记 deferred：M-1（nextKeyVersion 边界——随本修复整体消失，顺带闭环）/ M-2（拉取拒绝文案区分度）/ M-3（轮换审计 businessType=INSERT 语义）/ M-4（实体 toString 含 ciphertext 源端治理）/ M-5（掩码泄露度差异化策略）/ M-6（框架 HttpMessageNotReadable WARN 回显畸形体片段）/ M-7（仓库内 dev 默认 data-key 与「密钥不入库」口径张力）——收尾时按评审裁决进 LOG。
- 本地留档新增：backend/target/task0019_fulltest_rev.log、task0019_rev_dev.log、t0019_rev_smoke.py。

### R-6 修订轮过程登记（共享工作区并行任务干扰，如实记录）

1. **首轮 RED 被编译阻断**：共享工作区并行任务（TASK-0020/0009）中途态先后两次使 ot-admin 主代码不可编译（接口方法未实现/测试引用缺失类）——首轮 `-Dtest` RED 捕获被阻断，改为 A/B 重演（临时还原旧 rotate → 4 用例红 → 恢复 → 51 绿）取干净红转绿证据；全部等待复跑均在并行任务提交间隙完成，未触碰其文件。
2. **ot-core 本地仓 jar 被并行 install 覆盖**：dev 冒烟 R3a 首跑返回 500（`NoSuchFieldError: VERSION_CONFLICT`）——并行任务在 11:47 重装了不含本修订常量的 ot-core jar（覆盖本人 11:39 install）。处置：重装 ot-core 并以 python zipfile 校验 jar 内字节含 `VERSION_CONFLICT` 后重启复跑（11:50 起全绿）。
3. **python subprocess 的 bash 解析坑**：并发模拟的锁持会话经 python `subprocess` 启动 `bash` 时解析到 System32 的 **WSL bash**（/mnt/d 路径佐证），其环境无 jshell/mysql-connector → sql-exec.sh 静默失败（RC 2 空输出）——冒烟脚本显式钉死 `C:\Program Files\Git\bin\bash.exe` 并以探针（短锁超时 FOR UPDATE 的 SQLERROR）门控确认锁在位后才发并发轮换请求，消除时序不确定性。
