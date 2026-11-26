# 实施报告：TASK-0052（iot_api_client 调用方管理与调用审计）

- 日期：2026-11-23
- 执行者：implementer（subagent）
- BASE：`e492973`（开工时主仓 HEAD，即 TASK-0070 batch2 提交）
- 本任务提交：`1494e1e`（main，注明 TASK-0052；BASE→本提交无并行交错提交，21 文件线性落库）
- 结论先行：**五条验收全部通过**。官方口径全量回归 `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → **2366 = 开工基线实测 2332（BASE 提交 e492973 独立 worktree 实测）+ 净增 34，0 失败 0 错误 0 跳过 BUILD SUCCESS**；升级脚本二次执行两次 DONE + 三新列在位 + 菜单种子 4075-4078 幂等 + 超管授权 4 行 + 种子行零改动；dev 冒烟全序列 26/26 全绿 EXIT=0（开通明文一次性+唯一性 4002 → 新 appKey 真打六接口（HMAC 过+scope 外 6003）→ 轮换新旧并行双验签 → 过窗（SQL 改 rotated_at=-8d）旧 secret 403+prev 惰性清理实证 → 吊销立即 403 无 sleep → 审计行分布 + @OperateLog 留痕无明文 → 全停）；prod 档真实启动 13.5s + **ERROR=0** + 签名 dict/sync 200 + 篡改 403+6001；本任务提交 21 文件全落声明范围（application.yml 一键登记偏离，见 §4）。

---

## 0. 权威依据与上游差异核对（开工前完成）

| 依据 | 结论 |
| --- | --- |
| TASK-0052 实施 brief | DDL 三新列/管理接口四组/轮换并行窗协议/吊销立即 403/权限码与种子/测试口径/验收 5 条——逐条落地，见 §1/§2 |
| TASK-0051 交付（地基） | HMAC 鉴权链（HmacAuthStrategy 每请求查库读状态/到期/IP 白名单→SM4 解密验签）与六接口已在位；**双列 secret 已在位**（secret_hash 供快速校验保留 + secret_encrypted 供 HMAC）——上游差异沿 0051 登记，本任务只加列不改既有列 |
| REQ-0035 验收 | 1 明文仅一次 ✓（§2.3 S1a/S3）；2 scope 空默认拒绝 ✓（0051 filter 强制 + 管理侧落库 null，S2f/单测）；3 轮换双密钥并行一周 ✓（S4a/S4b + 过窗失效 S5a）；4 吊销立即 403 且无 TTL 空窗 ✓（S6b——现状核对：每请求查库，无缓存，无钩子需求）；5 管理页全操作留日志 ✓（@OperateLog 4 行实证）；6 列表/详情永不返回明文 ✓（S1b/S1c + VO 结构反射断言） |
| ADR-0006（缓存主动失效思路） | **核对结论：无钩子需求**——0051 filter 的客户端状态读取现状为每请求查库（HmacAuthStrategy.authenticate → clientMapper.selectByAppKey 每请求执行，无任何缓存层；单测 statusReadEveryRequestNoCache 以 times(2) 实证 + dev 冒烟 S6b 吊销后无 sleep 立即 403）。ADR-0006 的缓存主动失效思路在本域无作用对象（scope/status 不进缓存），登记于控制器 javadoc 与本报告 |
| 0015 M-2（4002）/0019 掩码与审计范式/0024 菜单种子范式 | ResultCode.DUPLICATE_CODE(4002) 既有（**零新增 ResultCode**）；掩码直接复用 0019 CredentialMask（≥12 前四****后四/8-11 前四****/更短或解密失败 ****）；菜单种子照 0024 sys_menu+sys_role_menu 幂等+超管授权范式 |

## 1. 改动文件清单（21 文件，全部在 brief 声明范围内）

### 1.1 新增：升级脚本（1）

| 文件 | 说明 |
| --- | --- |
| `resources/sql/upgrade/20261123_iot_api_client_rotation.sql` | ① iot_api_client 三新列幂等加列（brief 逐字列定义；MySQL 8 无 ADD COLUMN IF NOT EXISTS → information_schema 探列 + PREPARE 条件执行，照 20261122_iot_instance_columns.sql 范式；AFTER secret_encrypted 顺序落位）；② sys_menu 4075-4078 四行（页面行承载 openapi:client:list——微信模块页面行承载 list 权限为在库先例 03_wechat.sql 2001/2007；按钮 add/edit/remove 挂页面 4075；挂点智慧工厂目录 4000，头注登记挂点收敛说明）；③ sys_role_menu 超管授权 4 行；头注含用途/挂点登记/号段占用/接口对应/上游差异/验签顺序/幂等口径/sql-exec 注意 |

### 1.2 新增：主码（11，`backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/`）

| 文件 | 说明 |
| --- | --- |
| `entity/IotApiClient.java` | MP 实体（@TableName iot_api_client，extends BaseEntity——审计字段 fill + @TableLogic）；列族含轮换三新列；javadoc 声明密钥材料列永不经管理面回传 |
| `mapper/IotApiClientManageMapper.java` | BaseMapper + `rotateSecret` 原子换装 @Update（当前值移入 prev 三列 + secret_rotated_at=NOW + 新密文/哈希入主列——单条 UPDATE 行内换装，无中间态；会话链路多租户插件注入租户条件 + deleted=0 显式声明） |
| `controller/OpenApiClientController.java` | /openapi/client 管理面五端点：POST（开通/编辑同端点——brief「rate_limit 编辑同接口」；@SaCheckPermission OR(add,edit) + 端点内按 id 有无 StpUtil.checkPermission 分支精查——只有 add 码过不了编辑分支、只有 edit 码过不了开通分支）；GET /list、GET /{id}（list 码）；POST /rotate（edit 码）；PUT /status（remove 码）。@OperateLog：开通 INSERT+saveResponse=false、轮换 UPDATE+saveResponse=false（响应含明文——审计响应列不落库）、吊销 UPDATE；列表/详情不挂（0019 口径）。与 /openapi/v1 分层声明（过滤器 PATH_PREFIX 直通不干扰） |
| `service/OpenApiClientService.java` + `service/impl/OpenApiClientServiceImpl.java` | 开通（SecureRandom 32B→base64url 43 字符 ≥32 安全随机；SM4 加密 fail-closed——{v1} 幂等形态拒保存照 0019 守卫；SHA256 落 secret_hash；appKey 预检 + DuplicateKeyException 兜底双闸均转 4002 同文案不给探测面；scope 透传 JSON 合法性校验（parseObj），空白→null 默认拒绝；tenant_id 会话覆盖）；编辑（LambdaUpdateWrapper 白名单字段非空覆盖，scope 三态：不传=不改/空白=显式清空/非空=校验后覆盖；app_key/secret/status 不触）；列表/详情（解密后 CredentialMask 掩码化，解密失败降级 ****）；轮换（rotateSecret 原子换装，影响行数 0 并发删除兜底）；吊销/启用（字段白名单仅 status） |
| `dto/OpenApiClientSaveForm.java` | 开通/编辑表单（jakarta 校验；id 空=开通带值=编辑；javadoc 声明 status 不在表单——启停单设权限） |
| `dto/OpenApiClientVO.java` | 列表/详情视图：**结构上无密钥材料字段**（仅 secretMask + secretRotatedAt 透出轮换状态；单测反射固化） |
| `dto/OpenApiClientSecretVO.java` | 开通/轮换响应（唯一含 secret 明文的形态 + 「仅此一次」notice） |
| `dto/OpenApiClientRotateForm.java` / `OpenApiClientStatusForm.java` | 轮换 {id} / 吊销启停 {id, status 0|1}（jakarta @NotNull/@Min/@Max） |

### 1.3 修改：主码（4，brief 声明的验签顺序扩展面 + brief 明令的 yml 键）

| 文件 | 说明 |
| --- | --- |
| `auth/HmacAuthStrategy.java` | **验签顺序扩展（brief 定版，filter 内核零改动）**：当前列验 → 失败再 prev 列验（且 NOW - secret_rotated_at ≤ rotation-window-days，默认 7）→ 过窗惰性清理 prev（验签路径判定时清理，不建扫描任务——clearRotationWindow 单条 UPDATE，失败 WARN fail-soft，维护性动作非安全闸）；rotated_at 缺失 fail-closed 视同过窗；负时差（时钟回拨）不判过窗（宁宽勿误杀）；prev 解密失败按「旧密钥不参与」处理不抛 500；类 javadoc 同步声明「吊销立即生效——每请求查库读状态，无缓存无 TTL 空窗」 |
| `config/OpenApiProperties.java` | 新键 `rotationWindowDays`（默认 7；≤0 误配回落 7——照修订轮 I-3 误配回落范式，验签路径逐请求消费回落静默防日志刷屏） |
| `mapper/OpenApiClientRow.java` | +3 字段（secretEncryptedPrev/secretHashPrev/secretRotatedAt） |
| `mapper/OpenApiClientMapper.java` | SELECT 扩三列；新增 `clearRotationWindow`（@InterceptorIgnore + 显式 id 条件——机器线程无会话；javadoc 声明为只读投影 Mapper 的唯一写入例外，管理面轮换走 ManageMapper） |
| `application.yml` | ot.openapi 块新增一键 `rotation-window-days: ${OT_OPENAPI_ROTATION_WINDOW_DAYS:7}`（brief §接口与决策 7 明令「yml 新键」；改动范围清单未点名 yml——登记偏离 §4） |

### 1.4 测试（5，`backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/`）

| 文件 | 用例数 | 说明 |
| --- | --- | --- |
| `management/OpenApiClientServiceTest.java`（新） | 18 | 开通（secret ≥32 且 base64url 字符集/SM4 密文可解回明文/SHA256 哈希对账/明文仅 VO/tenant 会话覆盖/status=1 起步；撞键预检 4002/落库 DuplicateKeyException 兜底同 4002/scope 空白落 null/非法 JSON 400/缺必填 400）；编辑（白名单 sqlSet 断言——client_name/rate_limit 在更新集而 app_key/secret/status/tenant_id 全不触；scope 显式清空；行不存在 400）；列表/详情（掩码 test****/解密失败降级 ****/轮换时间透出）；轮换（rotateSecret 参数对账——新密文可解回 VO 明文+哈希=SHA256(新明文)；影响行数 0 → 400）；吊销（仅 status=0 一字段）；generateSecret 两枚互异 |
| `management/OpenApiClientMenuPermissionGuardTest.java`（新） | 3 | SQL 侧 4075-4078 恰四值映射 + 挂点（4000/4075）断言；代码侧控制器权限码并集恰四值；双向核对（四码全登记/四行全挂端点无死行——BUG-0046 口径，照 IotCredentialMenuPermissionGuardTest 范式） |
| `management/OpenApiClientAuditNoPlaintextTest.java`（新） | 5 | 真实 OperateLogAspect 经 AspectJProxyFactory 织入真实控制器（0019 范式）：开通响应列不落库（saveResponse=false）+全列 grep 无明文/轮换同/失败路径 failReason 无明文/吊销 param 承载 id/VO 结构边界（列表详情 VO 无 secret 材料字段 + SecretVO 是唯一明文出口——反射固化） |
| `auth/HmacAuthStrategyTest.java`（扩 +8，14→22） | 8 | 轮换并行窗：窗内旧密钥验签过（prev 路径）+不触发清理；现役密钥优先；过窗（8 天>7 天）旧密钥 6001 且 clearRotationWindow 恰一次；过窗后现役密钥照常过且同步清理；rotated_at 缺失 fail-closed 6001+清理；窗内双失败不清理；prev 密文损坏 6001 不降级 500；**吊销立即 403（两连 authenticate → selectByAppKey times(2)——无实例级缓存实证面）** |
| `config/OpenApiYmlStructureTest.java`（适配） | 2 | openapi 顶层独立块四键→**五键**（+rotation-window-days），dataperm 三键断言不变（I-1 回归语义保留） |

计数：18+3+5+8+2=36，其中净增 34（yml 2 例为既有改造非新增）——**2366 = 2332 + 34**。

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` —— 通过

```
$ mvn install -pl ot-core -DskipTests
[INFO] BUILD SUCCESS（ot-core 安装 1.828s）

$ mvn test -pl ot-admin（全量，官方口径终码）
[INFO] Tests run: 2366, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  54.555 s
EXIT=0
```

- 基线独立实测：BASE 提交 `e492973` 于临时 worktree 跑官方口径 → `Tests run: 2332, Failures: 0, Errors: 0, Skipped: 0` + BUILD SUCCESS（worktree 用毕即删）。**2366 = 2332 + 净增 34**（§1.4 计数）。
- 红转绿记录（单测迭代 2 处，均测试侧 stub/编译面修正，主码零语义改动）：
  1. `OpenApiClientServiceTest.createClientDuplicateAppKey` 首轮失败（Expected BusinessException to be thrown, but nothing was thrown）。根因：MP 3.5.17 `ServiceImpl.getOne(wrapper)` 委托 **`BaseMapper.selectOne(wrapper, throwEx)` 两参重载**（javap 反编译 AbstractRepository.getOne 实证 invokeinterface selectOne:(Wrapper,Z)），单参 `selectOne(any())` stub 不命中两参调用路径 → 预检恒空 → 未抛 4002。修复：stub 改 `selectOne(any(), eq(TRUE))`。**为什么现在对了**：stub 与 MP 3.5.17 真实委托链（getOne → 两参 selectOne）对齐，预检路径真实命中既有行。
  2. 编译面三处：实体漏声明 remark 列 → 补齐（DDL 既有列）；`generateSecret` 包私有 → public（跨包测试消费）；HmacAuthStrategyTest 缺 Mockito import → 补齐。**为什么现在对了**：实体列集与 DDL 对齐；测试可见性与工具类访问口径一致。
- 冒烟脚本迭代 3 处（脚本侧，主码零改动，见 §2.3 附注）。

### 2.2 验收 2：升级脚本二次幂等 + 三新列 + 菜单种子 + 种子行 —— 通过

第一次执行（17 语句全 OK，加列 3 条各 OK(0)、菜单/授权 INSERT 各 OK(4)）：

```
$ bash scripts/sql-exec.sh ../resources/sql/upgrade/20261123_iot_api_client_rotation.sql
OK(0) ×15 …（SET NAMES + 3×[探列/IF/PREPARE/EXECUTE/DEALLOCATE]）
OK(4)      ← sys_menu 4075-4078 四行
OK(4)      ← sys_role_menu 四行授权
DONE
```

第二次执行（幂等复核，三列全部 skip + INSERT IGNORE 零行变更）：

```
$ bash scripts/sql-exec.sh ../resources/sql/upgrade/20261123_iot_api_client_rotation.sql
OK(0) ×3
note
skip: secret_encrypted_prev already exists
OK(0) ×3
note
skip: secret_hash_prev already exists
OK(0) ×3
note
skip: secret_rotated_at already exists
OK(0) ×2
OK(0)      ← INSERT IGNORE 命中既有行，0 行变更
OK(0)
DONE
```

三新列核对（`SHOW COLUMNS FROM iot_api_client LIKE 'secret%'`）：

```
Field	                Type	        Null	Key	Default	Extra
secret_hash	        varchar(128)	NO		null
secret_encrypted	varchar(512)	YES		null
secret_encrypted_prev	varchar(512)	YES		null
secret_hash_prev	varchar(128)	YES		null
secret_rotated_at	datetime	YES		null
```

菜单种子与超管授权核对：

```
$ SELECT id, parent_id, menu_name, menu_type, perms FROM sys_menu WHERE id BETWEEN 4075 AND 4078
4075	4000	调用方管理	2	openapi:client:list
4076	4075	调用方新增	3	openapi:client:add
4077	4075	调用方修改	3	openapi:client:edit
4078	4075	调用方吊销	3	openapi:client:remove

$ SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id BETWEEN 4075 AND 4078
1	4075
1	4076
1	4077
1	4078
```

种子行零改动核对（`SELECT id, tenant_id, app_key, status, deleted FROM iot_api_client`，0051 种子原样）：

```
1	1	test_client	1	0
```

### 2.3 验收 3：dev 冒烟（8098，dev 档）—— 通过（26/26，EXIT=0）

启动：`mvn package -DskipTests -pl ot-admin`（终码 jar）→ `dev.ps1 start`（8098，Started in 11.51s）→ 冒烟脚本 `backend/target/t0052_smoke.py`（HMAC 链 python hmac 本地重算，与 HmacAuthStrategy 同口径；DB 核对走显式 Git Bash 全路径调 sql-exec.sh）终轮完整原始输出：

```
OK   | P0                                             HTTP 200
OK   | S0                                             200 (200, 'token len=36')
OK   | S1a                                            200 200 len=43 notice=有
OK   | S1b                                            mask=nkZp****wr5o rows=1 明文出现=False
OK   | S1c                                            mask=nkZp****wr5o rotatedAt=None 明文出现=False
OK   | S1d                                            200 4002 appKey 已存在（全局唯一，不可重复）: mid_client_0052
OK   | S2a                                            200 200 nodes=10
OK   | S2b                                            200 200 data={}
OK   | S2c                                            200 200 rows=0
OK   | S2d                                            200 200 rows=0
OK   | S2e                                            200 6003 点位不属于调用方授权范围（scope 外 6003）
OK   | S2f                                            200 6003 fullKey 不属于调用方授权范围（scope 外 6003）
OK   | S2g                                            200 200
OK   | S3                                             200 200 len2=43 旧明文出现=False
OK   | S3b                                            ['secret_encrypted_prev IS NOT NULL\tsecret_hash_prev IS NOT NULL\tsecret_rotated_at IS NOT NULL', '1\t1\t1']
OK   | S4a                                            200 200
OK   | S4b                                            200 200
OK   | S5a                                            403 6001 签名不匹配
OK   | S5b                                            ['secret_encrypted_prev IS NULL\tsecret_hash_prev IS NULL\tsecret_rotated_at IS NULL', '1\t1\t1']
OK   | S5c                                            200 200
OK   | S6a                                            200 200
OK   | S6b                                            403 6001 调用方已禁用
OK   | S7a                                            dist={'200': 8, '6003': 2, '6001': 2} rows=12
OK   | S7b                                            tenants=['1', 'NULL']
OK   | S8a                                            rows=4
OK   | S8b                                            泄漏行=0

TOTAL=26 OK=26 FAIL=0
SMOKE DONE
```

序列对照 brief：S1a 开通（明文 43 字符 + notice「仅此一次」）→ S1b/S1c 列表/详情只回掩码且全文无明文 → S1d 唯一性 4002 → S2a-S2g 新 appKey 真打六接口（models/tree scope 子树含 F01 10 节点 / realtime / history 带降采样 / metrics / dict/sync 全 200；points 未绑点位 6003 与 scope 外 fullKey 6003 照 0051 口径）→ S3 轮换（新明文 43 字符且响应不含旧明文；S3b prev 三列带走旧值 DB 实证）→ S4a/S4b **并行窗内新旧双验签都过** → S5 SQL 改 `secret_rotated_at = NOW() - INTERVAL 8 DAY` → S5a 旧 secret 403+6001 → S5b **prev 三列已置 NULL（惰性清理 DB 实证）** → S5c 新 secret 照常过 → S6a 吊销 → S6b **立即 403+6001「调用方已禁用」（无 sleep——无 TTL 空窗）** → S7 审计行分布（200×8/6003×2/6001×2；认证失败行 tenant_id=NULL、业务行 tenant=1）→ S8 @OperateLog 留痕 4 条且全列无明文。

审计落库实样（`SELECT id, tenant_id, app_key, api_path, result_code, cost_time FROM iot_api_call_log WHERE app_key='mid_client_0052' ORDER BY id`）：

```
88	1	mid_client_0052	/openapi/v1/models/tree	200	19
89	1	mid_client_0052	/openapi/v1/items/realtime	200	25
90	1	mid_client_0052	/openapi/v1/items/history	200	22
91	1	mid_client_0052	/openapi/v1/metrics/query	200	15
92	1	mid_client_0052	/openapi/v1/points/history	6003	17
93	1	mid_client_0052	/openapi/v1/items/history	6003	23
94	1	mid_client_0052	/openapi/v1/dict/sync	200	12
95	1	mid_client_0052	/openapi/v1/models/tree	200	23     ← S4a 旧密钥（并行窗）
96	1	mid_client_0052	/openapi/v1/models/tree	200	22     ← S4b 新密钥
97	null	mid_client_0052	/openapi/v1/models/tree	6001	18   ← S5a 过窗旧密钥
98	1	mid_client_0052	/openapi/v1/models/tree	200	14     ← S5c 新密钥
99	null	mid_client_0052	/openapi/v1/models/tree	6001	5    ← S6b 吊销后立即
```

@OperateLog 留痕实样（`SELECT id, business_type, method, url, response, param, status FROM sys_operate_log WHERE module='调用方管理'`）：

```
5589	1	…OpenApiClientController.save()        /openapi/client         (empty)                     {"appKey":"mid_client_0052","clientName":"数据中台冒烟(T   1
5590	1	…OpenApiClientController.save()        /openapi/client         (empty)                     {"appKey":"mid_client_0052","clientName":"重复开通"}      0
5591	2	…OpenApiClientController.rotate()      /openapi/client/rotate  (empty)                     {"id":4}                                                 1
5592	2	…OpenApiClientController.updateStatus() /openapi/client/status {"code":200,…               {"id":4,"status":0}                                      1
```

（开通/轮换两行 response=(empty)——saveResponse=false 生效，明文不进 sys_operate_log；吊销响应无敏感面默认落库。）

dev 日志 ERROR 分类（7 条全部为 GlobalExceptionHandler「业务异常」预期拒绝留痕）：6003 文案 ×4（S2e/S2f 两轮）、4002 文案 ×2（S1d）、`metricCodes 不能为空` ×1（首轮冒烟脚本缺陷，修复见下）。无未预期错误。

冒烟脚本迭代 3 处（脚本侧问题，主码零改动）：
1. metrics/query 首轮 400——脚本漏传 `metricCodes`（服务契约必填）。修复：URL 补 `metricCodes=GEN_T43_M1`。**为什么现在对了**：与 §3.5 metrics/query 契约（metricCodes 必填 + period 白名单）对齐。
2. 审计分布断言把 sql-exec 的**列头行**计成数据（dist 出现 'result_code': 1）。修复：解析前滤列头行。**为什么现在对了**：sql-exec 结果集首行恒为列标签，剥离后计数才是真实行分布。
3. subprocess 读 sql-exec 输出 UnicodeDecodeError（jshell 管道输出混非 UTF-8 字节）。修复：`encoding="utf-8", errors="replace"`。**为什么现在对了**：sql-exec 输出经 jshell/tr/grep 管道，编码不保证纯 UTF-8，replace 保行数与 ASCII 断言不受损。
- 另：首轮冒烟后重跑前做了脚本夹具复位（DELETE 冒烟 appKey 行/审计行/操作日志行）——夹具动作非主码行为。

用毕即停：`dev.ps1 stop`（PID=56484 已停止）→ `netstat :8098` 复核无 LISTENING。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn package -DskipTests -pl ot-admin`（终码 jar，133.6MB）。启动命令（沿 0051 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

干净单轮（日志 /tmp/t0052_prod.log）：

```
2026-11-23T05:39:09.354+08:00  INFO 57880 --- [ot-framework] [main] net.otlab.admin.OtAdminApplication :
    Started OtAdminApplication in 13.497 seconds (process running for 14.036)
liveness=200
DICT-SYNC (200, '{"code":200,"data":{"since":null,"instances":[],"items":[]},"message":"操作成功",…}')
TAMPER 403 {"code":6001,"message":"签名不匹配","timestamp":1795383583296}
grep -ac " ERROR " /tmp/t0052_prod.log → 0
```

- prod 启动 13.5s、**ERROR=0**；签名调用 200（占位契约）+ 篡改 403+6001——prod 档鉴权链与轮换扩展代码同套路径等价生效。
- 用毕即停：`taskkill /F /PID 57880` → `netstat :8098` 仅余 TIME_WAIT 客户端残留、无 LISTENING。

### 2.5 验收 5：改动范围核对 —— 通过

```
$ git diff --name-only e492973..1494e1e | wc -l
21      ← 本任务提交（BASE→本提交无并行交错提交，21 文件全落声明范围，清单见 §1）
```

逐一核对：14 主码文件均 `module/openapi/**`；5 测试文件均 `test/.../module/openapi/**`；升级脚本 1（即 brief 指定路径）；`application.yml` 1（仅 ot.openapi 块新增 rotation-window-days 一键——brief §接口与决策 7 明令「yml 新键」，改动范围清单未点名，登记偏离 §4-1）。ResultCode.java 零改动（4002 既有码复用，brief「如需新码——最小化」→ 无需）。不碰 0051 六接口行为/前端/ingest/dataperm；HmacAuthStrategy 仅验签顺序扩展与状态读取核查（filter 内核零改动）；TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 调用方管理前端页面（iot/apiclient/index 组件） | brief 明文「不碰前端」；本任务交付菜单种子（4075 页面行 component=iot/apiclient/index 占位）与后端接口，前端页归后续前端任务 |
| scope 授权树形选择器（从实例树勾选生成 scope JSON） | 同上——前端面；管理接口已透传 scope JSON 并校验合法性（实例 id 存在性校验归前端选择器 + 消费侧 fail-closed：不存在 id 等价无权限） |
| 缓存主动失效钩子 | ADR-0006 思路核对结论：0051 验签路径每请求查库（无 scope/status 缓存），吊销天然立即——无钩子作用对象（brief 预案「若每请求查库则天然立即（报告说明）」命中，§0 登记） |
| nonce 幂等表 | brief 未列入本任务四组接口；0051 登记口径维持（一期只收不验） |
| 审计 180 天按月归档 | REQ-0027 口径归后续运维任务（0051 已登记），本任务不重复建设 |
| 0019 修订轮 deferred Minor 中与调用方域相关项 | 不在本 brief 面；0051 评审 §二 Minor 表为登记源 |

## 4. 偏离计划之处

1. **application.yml 一键新增**（rotation-window-days）：brief 改动范围清单未点名 application.yml，但 §接口与决策 7 明文「7 天常量 ot.openapi.rotation-window-days（默认 7，yml 新键）」——两处冲突以条目级明令执行，落一键于既有 ot.openapi 顶层块（未动其它键），并同步 OpenApiYmlStructureTest（四键→五键）守结构。登记于此，建议后续 brief 改动范围清单补记该文件。
2. **菜单挂点**：brief 字面「新页挂系统管理下合适父节点」——实测菜单树：管理后台「系统管理」目录为 sys_menu 1（module_id=1，上游框架面），智慧工厂控制台目录为 sys_menu 4000（module_id=4）；而「iot 菜单一律用 4000 段」为在库号段纪律（TASK-0055 号段裁决），调用方管理属智慧工厂对外开放域（FEAT-0017）且目标角色为租户管理员（REQ-0035，在智慧工厂控制台操作）。故挂点落**智慧工厂目录（4000）第 11 个子页**（与字典/模板/实例等管理页同层），挂点收敛与理由全部登记于升级脚本头注「挂点登记」段 + 守卫测试锁定挂点（挪 parent 即红）。
3. **页面行承载 list 权限**（4075 perms=openapi:client:list，按钮仅 add/edit/remove 三行）：brief 限定「菜单种子 4075-4078」恰 4 行而权限码恰 4 个——页面行承载 list 使「新页 + 四码」在 4 行内自洽；页面行承载 list 权限为在库先例（03_wechat.sql 2001/2007 等、20260925_wechat_mp_template_send.sql 2020），iot 域页面行 perms=NULL 的惯例仅适用于「页 + 多按钮」五至六行形态（如 metric 4058-4062）。守卫测试双向锁定（含无死行）。
4. **开通/编辑同端点**（POST /openapi/client，OR(add,edit) + 分支精查）：brief「rate_limit 编辑同接口」与「uk_app_key 撞键 → 4002」并存——带 id 分支为编辑（白名单覆盖），无 id 分支为开通（撞键 4002）；分支精查保证 add-only/edit-only 角色越权直调 403（NotPermissionException → GlobalExceptionHandler 403 语义）。编辑面同时承载 scope/ip_whitelist/expire_time/client_name/remark 白名单（REQ-0035 验收 5「管理页可完成开通/scope/轮换/吊销全操作」要求 scope 可编辑，brief 未单列编辑端点）。
5. **TDD 红转绿形态**：brief 期待「红转绿留痕」——实际记录 2 条（§2.1）：1 条测试断言级（stub 与 MP 3.5.17 委托链错位）+ 3 处编译面（合并记 1 条）；未产出「主码实现错误的红」——实现前范式核对（0019/0024/0051 同款）吸收了主要风险面，红集中于测试侧。
6. **冒烟夹具复位 DELETE**：首轮冒烟后重跑前 DELETE 了冒烟 appKey 行/审计行/操作日志行（dev 库脚本夹具动作，非主码行为）；test_client 种子行全程零改动（§2.2 核对）。

## 5. 遗留问题

1. **dev 库冒烟残留**（不影响仓库与提交）：mid_client_0052 行（吊销态 status=0、scope 指向实例 2 子树、prev 列已惰性清理为 NULL——恰为吊销终态可直接作后续核对面）；iot_api_call_log 12 行审计（app_key=mid_client_0052，正常业务数据）；sys_operate_log 4 行调用方管理留痕。如需清理由收尾方处置。
2. **前端页缺位**：菜单 4075 已入树但 component=iot/apiclient/index 无对应前端组件——超管点入即空白路由；前端任务交付页面时无需动本任务任何产物（接口契约与权限码已就位）。
3. **逻辑删行占用 app_key 的复用路径**：uk_app_key 物理唯一含逻辑删行——逻辑删后的 appKey 无法直接重新开通（落库撞键 → 4002，文案同既有行不给探测面）；如业务需要「删后复用」，需先物理清理行（登记为运维口径，不自行加物理删除接口——REQ-0035 无删除语义）。
4. **scope 深层结构语义**：管理侧仅校验 JSON 对象合法性（brief「透传校验 JSON 合法性」口径）；instanceIds 指向不存在实例/结构键名错误等深层面由消费侧 fail-closed 兜底（dataperm 空集拒绝——不产生越权面），但无管理面即时反馈——前端选择器交付时可加树内勾选消除该面。
5. **并行窗过窗惰性清理的触发依赖验签流量**：无流量的调用方过窗后 prev 列残留（下次验签或下次轮换时收敛）——brief 明文「不建扫描任务」，符合设计；DB 层面 prev 残留无安全影响（验签路径判定先行失效）。
6. **0054 交接**：dict/sync 增量语义（0054 既有登记）；本任务不涉。**0068 联调**：实时值/历史数据路径（0051 既有登记）——本任务轮换链不依赖数据面。

---

## 6. 一句话结论

**通过**——五条验收全绿：官方口径全量 2366（基线 2332 实证 + 净增 34）0 失败 BUILD SUCCESS、升级脚本两次 DONE + 三新列/种子 4075-4078/超管授权核对一致、dev 冒烟 26/26（开通明文一次性+4002、轮换并行双验签、过窗失效+惰性清理 DB 实证、吊销立即 403 无 TTL 空窗、审计与操作日志全量留痕且无明文）、prod 启动 0 ERROR、提交 1494e1e 的 21 文件全落声明范围（application.yml 一键按 brief 条目明令落，已登记偏离）。
