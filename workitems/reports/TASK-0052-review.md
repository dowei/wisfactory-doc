# diff 评审：TASK-0052（iot_api_client 调用方管理 / 双密钥轮换并行窗 / 吊销立即生效）

- 评审日期：2026-11-23
- 评审对象：提交 `1494e1e`（BASE=`e492973`，21 文件）——diff 包 `TASK-0052-diff.txt`
- 评审口径：`L1-execute.md` §13 diff 评审 + 本任务评审 brief 九项重点
- 评审方法：fresh 全量读 diff 21 文件 + 工作区源码复核（HEAD 即 `1494e1e`，工作区干净）；本地独立复跑官方口径全量测试；支撑组件（filter/aspect/mapper/租户插件/SM4/掩码/Sa-Token 配置）逐一实读，未采信任何未核实的报告声明。

---

## 一、评审重点逐项核对（brief 九项）

### 1. 明文一次性纪律 —— 通过

- 全 diff 扫描：明文 secret 唯一出口为 `OpenApiClientSecretVO.secret`（开通/轮换两端点响应体）；`OpenApiClientVO` 结构上无任何密钥材料字段（仅 `secretMask`），且反射单测固化。
- @OperateLog：开通/轮换两端点 `saveResponse=false`（controller 63/102 行）——实读 `OperateLogAspect.java:141` 确认 `annotation.saveResponse()` 为 false 时 response 列不序列化落库；`OpenApiClientAuditNoPlaintextTest` 经 AspectJProxyFactory 织入**真实切面 + 真实控制器**捕获落库实体 grep——「真实切面织入」声明属实（非注解纸面检查）。
- 列表/详情/日志：`OpenApiClientServiceImpl` 全部 5 处 log 只记 id/长度/status（132/172/208/238/262 行）；`HmacAuthStrategy` 日志只记密文长度（181/226 行）；无任何明文/密文入日志。
- 审计行（iot_api_call_log）：`OpenApiAuthFilter.baseAuditRow` 只落 appKey/apiPath/paramDigest（SHA-256 指纹）/resultCode/costTime——六接口查询串无 secret、签名头不入审计；403 行 tenant_id=NULL 如实。
- sys_operate_log param：开通入参无 secret（服务端生成）；失败路径 failReason 文案无凭据值（单测覆盖）。
- 响应体掩码：`CredentialMask` 0019 同款（≥12 前四后四/8-11 前四/更短或解密失败 `****`），解密失败降级不崩不泄底（实读 CredentialMask.java 与实现一致）。

### 2. 轮换协议 —— 通过

- **原子换装**：`IotApiClientManageMapper.rotateSecret`（@Update 37-42 行）单条 UPDATE 行内换装，MySQL 单表 UPDATE SET 子句自左向右求值（官方文档 13.2.13 明载），当前子句序 prev←encrypted → prev←hash → rotated_at → encrypted←new → hash←new 语义正确——不存在「旧值既不在 prev 也不在主列」的可见中间态（行锁 + 单语句原子）；并发轮换语义自洽（后到者按执行时刻行内值为 prev）。
- **验签顺序扩展**：`HmacAuthStrategy` 86-98 行——过窗判定先行，过窗即不给 prev 任何验签通道（95 行 `!rotationWindowExpired` 门）并惰性清理；未过窗时当前列失败才试 prev 列。rotated_at 缺失 fail-closed 视同过窗（145-147 行）；负时差（时钟回拨/误写未来值）不判过窗（150-152 行，宁宽勿误杀，注释登记）。
- **0051 单密钥路径兼容**：prev 三列全 NULL 时 `isRotationWindowExpired`/`hasPrevSecret` 恒 false → 流路与 0051 逐字等价；`decryptSecret` fail-closed 6001 未动；既有 HmacAuthStrategyTest 全绿（本次评审独立复跑全量实证）。
- prev 解密失败按「旧密钥不参与」WARN 不抛 500（170-185 行）；secret_hash/secret_hash_prev 保留列不参与验签口径未变。

### 3. 并行窗 —— 通过

- 常量 `ot.openapi.rotation-window-days`（application.yml 唯一新键，默认 7）→ `OpenApiProperties.rotationWindowDays`（Lombok @Data）→ `HmacAuthStrategy.rotationWindowDays()`：≤0 误配回落 7 静默（语义注释登记，配置审计发现误配）。
- 惰性清理触发点=验签路径判定时（87-90 行），brief 明令「不建扫描任务」相符；并发安全：`clearRotationWindow` 为幂等 NULL UPDATE（物理主键 + deleted=0 显式条件），多请求同时触发经 MySQL 行锁串行、结果一致；首个成功后 prev=NULL 即不再触发；失败 WARN fail-soft 不影响验签结论（维护性动作非安全闸，定性正确——旧密钥已被判定逻辑失效）。
- 测试覆盖：窗内旧密钥过/现役优先/过窗旧失效+清理恰一次/过窗现役照常+同步清理/rotated_at 缺失/双失败不清理/prev 密文损坏 fail-closed 共 8 例 + dev 冒烟 S5a/S5b DB 实证。

### 4. 吊销立即 403 —— 通过（预案 A 成立）

- **独立复核 0051 filter 现状确无缓存**：`OpenApiAuthFilter.doFilter` 每请求调 `authStrategy.authenticate` → `selectByAppKey` 每请求查库（普通 @Mapper @Select，无 Spring Cache/Caffeine/本地 Map/Redis 任何缓存层——全模块 grep 排除，唯一 Redis 使用面为限流计数与实时值缓存，均与调用方状态无关）。无 TTL 空窗结论成立，ADR-0006「无钩子作用对象」核对结论成立，预案 A 无需升级。
- 实证面：单测 `statusReadEveryRequestNoCache`（两连 authenticate → selectByAppKey times(2)）+ dev 冒烟 S6b 吊销后无 sleep 立即 403+6001。

### 5. 管理面权限 —— 通过

- 四码落位：POST /openapi/client = `SaMode.OR(add, edit)` + 端点内按 id 有无 `StpUtil.checkPermission` 分支精查（add-only 过不了编辑分支、edit-only 过不了开通分支，NotPermissionException → GlobalExceptionHandler 403 语义已实读确认）；GET list/{id}=list 码；rotate=edit；PUT status=remove。与 brief「rate_limit 编辑同接口」一致。
- 双面边界：`/openapi/v1` 控制器类级 `@Anonymous`（登录拦截器经注解跳过 + HMAC filter 鉴权），`/openapi/client` 无 @Anonymous → Sa-Token 登录拦截 + @SaCheckPermission——管理面与机器验签面不混淆（SaTokenInterceptorConfig 实读确认，filter `PATH_PREFIX="/openapi/v1/"` 对 /openapi/client 直通）。
- 种子：4075-4078 INSERT IGNORE 显式 id 幂等 + sys_role_menu role_id=1 四行；页面行承载 list 权限有在库先例（03_wechat.sql 2001/2007）；守卫测试（SQL 正则解析 + 控制器反射）双向锁定四码↔四行 + 挂点；`SaPermissionRegistryConsistencyTest` 全局强制注解码 ⊆ sys_menu 登记（实读确认机制覆盖新控制器）。

### 6. 4002 唯一性 —— 通过

- 双闸：预检（本租户可见范围，LIMIT 1）+ `DuplicateKeyException` 兜底，均转 `ResultCode.DUPLICATE_CODE(4002)` 同文案（不给存在性差异探测面）；uk_app_key 物理全局唯一（0051 DDL 实读确认，含逻辑删行），逻辑删占用亦归 4002（登记为运维口径）。
- scope：空白 → null 落库（默认拒绝，0051 filter 强制空 scope 无数据权限）；非空必须 `JSONUtil.parseObj` 通过（非法 → 400），三态编辑（不传=不改/空白=显式清空/非空=覆盖）落库与测试齐备；scope 列为 MySQL JSON 类型，DB 层兜底校验。

### 7. SQL —— 通过

- 加列幂等：三列各自 information_schema 探列 + PREPARE 条件执行（20261122 范式），列定义逐字同 brief，AFTER 定位与实体声明序一致；二次执行「skip ×3」实证。
- test_client 种子零改动：脚本对 iot_api_client 无任何 DML，仅加列（DDL），种子行 `1/1/test_client/1/0` 核对原样。
- 4075-4078 挂点：brief 字面「系统管理」收敛为智慧工厂目录 4000——偏离已登记且理由充分（iot 域 4000 号段纪律、FEAT-0017 属智慧工厂对外开放域、目标角色租户管理员在控制台操作），守卫测试锁挂点，**追认合理**。

### 8. ResultCode 零改动 —— 通过

- ResultCode.java 不在 21 文件 diff 中；`DUPLICATE_CODE(4002)` 既有（ot-core ResultCode.java:23 实读确认）；新增业务拒绝全部复用 PARAM_ERROR/4002，未造新码。

### 9. 0051 既有测试零回归 —— 通过（评审者独立复跑）

- 本评审独立执行官方口径：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → **Tests run: 2366, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**——与报告声明完全一致（2332 基线 + 净增 34）；auth/filter 既有用例（含单密钥路径 8 例）在验签顺序扩展下全绿。

### 越界反查复核 —— 与协调者结论一致

- 21 文件逐一核对：14 主码全落 `module/openapi/**`、5 测试全落 `test/.../module/openapi/**`、升级脚本 1（brief 指定路径）、application.yml 1。yml 偏离：唯一触点即 `ot.openapi.rotation-window-days` 一键，brief 条目 7「7 天常量 ot.openapi.rotation-window-days（默认 7，yml 新键）」属条目级明令，改动范围清单未点名系 brief 自身两处不一致——执行者登记偏离 + OpenApiYmlStructureTest 四键改五键守结构，**追认**。

---

## 二、发现清单

无 Critical、无 Important。以下 Minor 不阻塞收尾（§13 口径：记 deferred，不进修订轮）：

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| M1 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/service/impl/OpenApiClientServiceImpl.java:150-171` | 编辑请求仅带 id（六白名单字段全缺）时 LambdaUpdateWrapper 无任何 SET 子句，`this.update(upd)` 生成非法 UPDATE 落 SQL 语法错误 → 500 而非 400 参数拒绝；建议服务层对空更新集显式 400 或幂等成功。 |
| M2 | Minor | 同上 `:171, :262`（对照 `:233-236`） | `editClient`/`updateStatus` 不核 `update`/`updateById` 影响行数——预检与更新之间并发逻辑删会静默返回成功；`rotateSecret` 有 `updated==0 → 400` 兜底，防御姿态不一致。 |
| M3 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/dto/OpenApiClientSaveForm.java:45, :48`（对照 impl `:158, :164`） | `ipWhitelist`「空=不限」/`expireTime`「空=不过期」的 @Schema 文案与实现不符——空白/null 均按「不改」跳过，编辑路径无法清空这两字段（仅 scope 有显式清空三态）；前端按此文档做「清空白名单/清到期时间」将静默 no-op。 |
| M4 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/mapper/IotApiClientManageMapper.java:37-42` | 原子换装的正确性承载于 MySQL 单表 UPDATE SET 子句「generally 自左向右求值」（官方文档口径偏软）——子句顺序即正确性，单测 mock 了 mapper 无法拦截未来重排（本次仅 dev 冒烟 DB 级实证）；建议 SQL 处补顺序依赖注释或改为参数传旧值。 |
| M5 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/auth/HmacAuthStrategy.java:149` 对照 `IotApiClientManageMapper.java:39` | 过窗判定用应用 JVM `LocalDateTime.now()` 对比 DB `NOW()` 写入的 `secret_rotated_at`——双时钟源，应用与 DB 主机时钟/时区偏差会平移有效并行窗；与既有 expire_time 处理同款，属部署注意事项非代码缺陷。 |
| M6 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/controller/OpenApiClientController.java:63` | 开通/编辑双用途端点 @OperateLog businessType 静态 INSERT——编辑分支也落业务类型 1（sys_operate_log 语义上「开通」），人工审计需靠 method 摘要区分；双端点静态注解的既知权衡，登记即可。 |
| M7 | Minor（设计备注） | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/auth/HmacAuthStrategy.java:87-90` | 过窗惰性清理 UPDATE 位于验签前——未认证流量即可驱动维护性写（每次请求一条，直至首个清理成功收敛）；幂等 + fail-soft + brief「验签路径判定时清理」明令设计，无安全影响，仅记录。 |

---

## 三、结论

**通过**。

- 九项评审重点全部核实（含两项关键独立复核：0051 验签链确无缓存——预案 A「无 TTL 空窗」成立；审计无明文测试确为真实切面织入）；yml 偏离与菜单挂点偏离均属 brief 条目明令/登记充分，追认。
- 全量回归由评审者本地独立复跑：2366/0/0 BUILD SUCCESS，与报告一致。
- 发现 7 条 Minor（M1-M7），无 Critical/Important——按 §13 记 deferred 收尾进 LOG，不进修订轮。
