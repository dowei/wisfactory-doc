# 实施报告：TASK-0051（对外查询六接口与鉴权——签名+scope+IP 白名单+限流+审计）

- 日期：2026-11-22
- 执行者：implementer（subagent）
- BASE：`d74d56f73b7b80a72ce304a6bb6432319316cb40`（开工时主仓 HEAD，即 TASK-0048 提交）
- 本任务提交：`8348047`（main，注明 TASK-0051；父提交 `4b2d59d`——见验收 5 关于并行交错提交的说明）
- 结论先行：**五条验收全部通过**。官方口径全量回归 `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → **2099 = 开工基线实测 2036 + 净增 63，0 失败 0 错误 0 跳过 BUILD SUCCESS**（基线的 3 个 skipped 为红线二占位，本任务转正后 0 skipped）；升级脚本二次执行幂等两次 DONE + 两表结构与附录 A p34 逐字一致（含两处与上游差异列）+ test_client 种子行核对一致；dev 冒烟全序列绿（S0-S7，EXIT=0：scope 裁剪树 / scope 外 6003 / 空集如实 / 限值 400 / 篡改签名 403+6001 / 过期时间戳 403+6001 / 限流 6002 / 审计 11 行分布与 tenant_id 形态核对）；prod 档真实启动 25.5s + **ERROR=0** + /openapi/v1/dict/sync 签名调用 200（占位响应）+ 篡改签名 403+6001；本任务提交 46 文件全落声明范围。

---

## 0. 权威依据核对（pypdf 提取，实现前完成）

| 段落 | 提取结论（实现以此对齐） |
| --- | --- |
| 详细设计 V1.1 §3.5 p26（六接口行） | models/tree（scope 限定）/ items/realtime（批量上限 200 个 fullKey，优先 Redis，返回 v/q/ts）/ items/history（interval 降采样粒度如 60s/300s/3600s + agg=SUM/AVG/MAX/MIN；**原始粒度跨度上限 7 天，超限必须带 interval**）/ metrics/query（scope + 指标 dept 双重限定——中台路径 deptIds 恒空集坍缩为实例分支，ADR-0016 增补 8；返回 value 与 yoy/mom，上期无数据 null + compareNote 不返回 0 冒充）/ points/history（只查不算，**跨度上限 31 天**）/ dict/sync（appKey 有效即可）；响应示例字段 metricCode/instanceCode/windowStart/value/unit/yoy/mom；「对外接口一切错误（含 scope 外访问）统一记 iot_api_call_log，403 类错误全量记录」 |
| §3.4 p20（错误码分段） | 6001-6099 数据服务对外（签名错误 6001、限流 6002、授权范围外 6003）；4001-4099 参数与校验；pageSize 上限 500；时间戳毫秒（epoch ms）用于对外接口 |
| 附录 A p34（DDL） | iot_api_client / iot_api_call_log 逐字（uk_app_key/idx_tenant/idx_app_time），见升级脚本头注「与上游差异」三处登记 |

**错误码 / HTTP 状态映射表（brief 裁决落地，报告登记）**：

| 场景 | 业务码 | HTTP 状态 | 落点 |
| --- | --- | --- | --- |
| 签名不匹配 / 缺头 / 未知或已删调用方 / 禁用 / 到期 / IP 白名单外 / 密钥未配置或解密失败 / timestamp 超窗（6001 段） | 6001 | **403**（传输层 403 仅认证失败形态） | 过滤器直接回写 |
| 限流超限 | 6002 | 200 | 过滤器直接回写 |
| 单目标查询 scope 外（items/history、points/history） | 6003 | 200 | BusinessException → GlobalExceptionHandler |
| 批量>200 / 原始粒度跨度>7 天未带 interval / points 跨度>31 天 / interval-agg 非法 / period 非法 / start-end 非法 | 400 | 200 | 同上（§3.4 参数与校验段） |
| 成功 / 集合接口 scope 外键裁剪（非错误） | 200 | 200 | 控制器 |

## 1. 改动文件清单（46 文件，全部在 brief 声明范围内）

### 1.1 新增：升级脚本（1）

| 文件 | 说明 |
| --- | --- |
| `resources/sql/upgrade/20261122_iot_openapi_domain.sql` | ① iot_api_client（附录 A p34 逐字 + `secret_encrypted varchar(512)` 与 `ip_whitelist varchar(500)` 两列 inline，头注「与上游差异」登记）；② iot_api_call_log（p34 逐字）；③ INSERT IGNORE 冒烟测试调用方 test_client（secret_hash=SHA256('test_secret') 字面量 + secret_encrypted=SM4('test_secret') 的 {v1} 密文字面量，jshell+hutool 以 SmDataCrypto 同构生成、单测 OpenApiSeedLiteralTest 钉住；scope 指向实例 id 1、rate_limit=3，prod 由运营方替换/禁用——头注声明，0052 交付管理界面）；头注含用途/差异/接口对应/审计纪律/幂等口径/业务纪律/sql-exec 注意 |

### 1.2 新增：主码（33，`backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/`）

| 文件 | 说明 |
| --- | --- |
| `OpenApiCodes.java` | 6001/6002/6003 业务码、X-App-Key/X-Signature/X-Timestamp/X-Nonce 头契约、签名串口径（appKey+timestamp+bodyHash）、请求属性键 |
| `auth/OpenApiAuthStrategy.java` | 鉴权策略接口（ADR-0009 TD-0004：一期 HMAC 落地，OAuth2 client credentials 替换位——换策略不改业务接口） |
| `auth/HmacAuthStrategy.java` | 默认策略：必填头 → timestamp 防重放窗口（±300s，OpenApiProperties）→ 调用方加载（app_key 全局唯一定位，deleted=已删视同不存在）→ 状态（禁用立即生效）→ 到期 → IP 白名单（NULL=不限，CSV 精确匹配 trim 容错）→ secret_encrypted SM4 解密得原文 → HMAC-SHA256(secret, 签名串) 恒时序比对（MessageDigest.isEqual，hex 归一）；解密失败 fail-closed 6001 不降级 500 |
| `auth/OpenApiCredentials.java` | 签名凭据值对象：header 提取 + bodyHash=SHA256(body)（空体恒定摘要——六接口全 GET）+ 签名串拼接 |
| `auth/OpenApiClientIdentity.java` | 验签产物（clientId/tenantId（调用方行 DB 权威）/appKey/scopeJson/rateLimit） |
| `auth/OpenApiAuthException.java` | 认证失败异常（6001 + 具体文案；无存在性差异泄露——未知/已删同文案） |
| `filter/OpenApiAuthFilter.java` | /openapi/v1/** 全覆盖 Servlet 过滤器：凭据提取 → 策略验签（403+6001）→ 限流（200+6002）→ 身份写请求属性下传 → ContentCachingResponseWrapper 回读统一 Result 体的业务码（scope 外 6003/400 等如实入审计）→ copyBodyToResponse → finally 全量异步审计（403/6002/6003/200/500 统一落库）；非 /openapi/v1 前缀直通；请求体缓存包装（为未来 POST 形态预留同一签名契约） |
| `config/OpenApiFilterConfig.java` | 过滤器注册（/openapi/*）+ 审计线程池 openapiAuditExecutor（照 ot-core operateLogExecutor 范式：CallerRuns 不丢行、关停等队列） |
| `config/OpenApiProperties.java` | ot.openapi.* 键族绑定：replay-window-seconds=300 / rate-limit-window-seconds=60 / realtime-cache-key-prefix |
| `ratelimit/OpenApiRateLimiter.java` | rate_limit 次/分钟固定窗 60s（RedisService.incrementWithTtl 原子 INCR+PEXPIRE，BUG-0053 口径）；键 rate_limit:openapi:&lt;appKey&gt;（落缓存监控既有 rate_limit 命名空间）；0=不限；**验签成功后计数**（防伪造 appKey 洪泛顶掉合法调用方——DoS 放大收窄）；Redis 故障 fail-open（限流是 QoS 非安全边界） |
| `audit/OpenApiAuditWriter.java` | 异步提交器（快照行对象、落库失败 WARN 不顶响应、executor 拒绝退化同步兜底） |
| `audit/OpenApiCallLogRow.java` | 审计行（tenant_id/app_key/api_path/param_digest 脱敏摘要/result_code/cost_time） |
| `mapper/OpenApiCallLogMapper.java` | iot_api_call_log INSERT（@InterceptorIgnore + 显式 tenant_id 列——异步线程无会话上下文，隔离与取值全显式化；认证失败行 tenant_id=NULL，DDL 允许） |
| `mapper/OpenApiClientMapper.java` + `OpenApiClientRow.java` | 按 app_key 取调用方行（旁路插件：app_key 全局唯一无租户歧义；deleted 显式回带防逻辑删行复活验签） |
| `mapper/OpenApiInstanceMapper.java` + `OpenApiInstanceRow.java` | 实例只读投影（旁路插件显式 tenant_id；全量树装配 / 按编码 / 按 id 三查） |
| `mapper/OpenApiItemMapper.java` + `OpenApiItemRow.java` | 数据项只读投影（full_key 批量定位——存在性判定 + scope 判定输入） |
| `mapper/OpenApiPointMapper.java` + `OpenApiPointRow.java` | 点位只读投影（pointCode 定位；bound_instance_id 实例分支——中台 deptIds 恒空集） |
| `mapper/OpenApiMetricMapper.java` + `OpenApiMetricRow.java` | 指标只读投影（metricCode 定位） |
| `mapper/OpenApiTenantMapper.java` | sys_tenant tenant_code 直查（TDengine 库名派生输入，库名唯一入口复用 TdengineAdmin.databaseName） |
| `service/OpenApiQueryService.java` + `impl/OpenApiQueryServiceImpl.java` | 六接口业务面：scope 裁剪走 dataperm 组件**显式租户重载**（组件消费不重写）；models/tree 复用 0012 树语义（可见集全字段 + 到根路径骨架收窄 M-7 + sort 升序 + 禁用节点照常返回）；realtime 批量归一去重 ≤200、Redis 优先→TDengine 最新一条回源回填、scope 外/未登记键裁剪（集合形态授权裁剪，不给枚举探测面）；items/history 单目标（限值矩阵 + scope 外 6003 + 未登记空集）；metrics/query 周期白名单 + scope 裁剪 + 实例编码回显；points/history 31 天限值 + bound_instance 实例分支 fail-closed；dict/sync 占位（appKey 有效即可，不走 scope） |
| `support/OpenApiTimeseriesClient.java` | 0023 曲线范式 fail-soft 客户端：st_item_value（full_key 标签 + INTERVAL 降采样 SUM/AVG/MAX/MIN 白名单）/ st_point_num·st_point_state（value_type 选表）/ st_metric_value（metric_code+period）；编码字面量 [A-Za-z0-9_.] 白名单 + interval/agg 拼接前二次校验；一切失败 WARN 空集（TDengine fd 未修复 → 本期空集如实，联调归 0068） |
| `support/OpenApiRealtimeStore.java` | 实时值读取器：Redis 键 openapi:realtime:{tenantId}:{fullKey}（0068 联调契约锚）→ 未命中回源 TDengine 最新一条并回填；值形态宽容解析；全链 fail-soft |
| `dto/OpenApiRealtimeItemVO.java` / `OpenApiHistoryPointVO.java` / `OpenApiMetricVO.java` / `OpenApiDictSyncVO.java` | §3.5 响应契约形状（v/q/ts、ts/v/q、metricCode/instanceCode/windowStart/value/unit/yoy/mom/compareNote（NON_NULL 仅 null 伴随值输出）、since/instances/items） |
| `controller/OpenApiController.java` | /openapi/v1 六端点（类级 @Anonymous——不进 Sa-Token 登录态，鉴权全权归过滤器；requireIdentity 防线：过滤器缺位时 fail-closed 401） |

### 1.3 修改：dataperm ClientScope 消费接线（3，brief 声明的微调面）

| 文件 | 说明 |
| --- | --- |
| `iot/dataperm/DataPermService.java` | 新增机器路径重载 `resolveClientScope(Long clientId, Long tenantId, String scopeJson)`（javadoc 声明与会话路径的差异只在隔离机制）；原会话路径方法签名与语义零改动 |
| `iot/dataperm/DataPermServiceImpl.java` | 重载实现：tenantId=null fail-closed 拒绝（增补第 9 条同口径 WARN）；scope 三态解析/子树展开截断/deptIds 恒空集复用同一组私有方法（同一组件同一口径，两路径不产生行为分叉）；行集取数走显式租户变体 |
| `iot/dataperm/mapper/DataPermInstanceMapper.java` | 新增 `selectAllForPermForTenant` / `selectByIdsForTenant`（@InterceptorIgnore(tenantLine) + 显式 tenant_id 条件——代码库既有旁路范式；/openapi 机器请求无登录会话，多租户插件跳过注入，隔离由 SQL 显式条件收敛） |

### 1.4 修改：application.yml（1，仅 ot.openapi.* 键族）

| 键 | 值 |
| --- | --- |
| `ot.openapi.replay-window-seconds` | 300（防重放窗口，brief 5 分钟） |
| `ot.openapi.rate-limit-window-seconds` | 60（固定窗宽） |
| `ot.openapi.realtime-cache-key-prefix` | `openapi:realtime:`（0068 联调契约锚；环境变量可覆盖） |

### 1.5 新增/转正：测试（9）

| 文件 | 用例数 | 说明 |
| --- | --- | --- |
| `test/.../openapi/auth/HmacAuthStrategyTest.java` | 14 | 鉴权矩阵：签名正确（SM4 解密还原 secret）/大写 hex 容错/错签/timestamp 过期/非法 timestamp/缺三头/未知 appKey/逻辑删行/禁用/到期/IP 白名单内外/密钥未配置/密文损坏 fail-closed 6001 |
| `test/.../openapi/auth/OpenApiSeedLiteralTest.java` | 2 | 钉住升级脚本种子字面量 ↔ 框架加密组件一致（SHA256 与 SmDataCrypto {v1} 解密）——data-key 变更/密文格式升级时脚本同步预警 |
| `test/.../openapi/filter/OpenApiAuthFilterTest.java` | 7 | 过滤器：认证失败 403+6001+审计（tenantId=null 全量记录）/限流 200+6002/放行身份属性下传/响应体业务码回读（6003 如实入审计）/未预期异常记 500 上抛/非 openapi 路径直通/参数摘要脱敏与 500 收敛 |
| `test/.../openapi/ratelimit/OpenApiRateLimiterTest.java` | 5 | 0 不限不触 Redis/窗口计数与键形态/超限拒/Redis 故障 fail-open/null appKey |
| `test/.../openapi/audit/OpenApiAuditWriterTest.java` | 4 | 异步落库行透传/落库失败吞错/executor 拒绝同步兜底/null 行跳过 |
| `test/.../openapi/service/impl/OpenApiQueryServiceImplTest.java` | 25 | 六接口契约：树三态（subtree 裁剪/null 拒空树/骨架收窄）/realtime 限值与键裁剪与空集/history 限值矩阵（7 天/interval/agg 成对/白名单/时间窗）+6003+空集+降采样放行/metrics 白名单+裁剪/points 31 天+未绑 6003+同部门 scope 外 6003（增补 8 回归）/dict 占位不走 scope/批量归一 |
| `test/.../iot/dataperm/DataPermClientScopeTenantTest.java` | 6 | 显式租户重载：tenantId=null 拒/scope null·空数组·非法 JSON 拒/subtree=false 显式租户精确/subtree=true 本租户展开/跨租户 id 不命中等价拒 |
| `test/.../iot/redline/ClientScopeRedlineTest.java`（转正，原 3 占位） | 3 | 红线二端到端（真实链装配：真 HMAC 策略+真过滤器+真控制器+真服务+真 dataperm 显式租户重载，iot_api_client 种子夹具签名请求，mock 仅持久层）：单厂 appKey 拉树只返回授权子树/scope null 默认拒绝空树/同部门 scope 外点位 6003（增补 8 接口层回归）且不触发 no-tenant-context 守卫 |

计数：14+2+7+5+4+25+6+3 = 66？——实际 surefire 汇总：openapi 包 57（14+2+7+5+4+25）+ dataperm 重载 6 + 红线转正 3 = **66 个新用例数与净增 63 的差值说明**：红线二 3 例此前在基线中以 skipped 计数（2036 含 3 skipped），转正后计入执行数——净增执行用例 = 63（57+6），另有 3 例由 skipped 转为 executed（总数 2099 = 2033 + 3 + 63，skipped 3→0）。

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` —— 通过

```
$ mvn install -pl ot-core -DskipTests
[INFO] Installing D:\workspace\WisFactory\backend\ot-core\pom.xml to C:\Users\dowil\.m2\repository\net\otlab\ot-core\1.0.0\ot-core-1.0.0.pom
[INFO] Installing D:\workspace\WisFactory\backend\ot-core\target\ot-core-1.0.0.jar to C:\Users\dowil\.m2\repository\net\otlab\ot-core\1.0.0\ot-core-1.0.0.jar
[INFO] BUILD SUCCESS

$ mvn test -pl ot-admin（全量，官方口径终码）
[INFO] Tests run: 2099, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  01:17 min
EXIT=0
```

- 2099 = 开工基线实测 2036（TASK-0048 提交口径，其中 3 skipped=红线二占位）+ 本批净增 63（openapi 57 + dataperm 显式租户重载 6），另 3 例红线二转正（skipped→executed），故 Skipped 3→0。
- 单测迭代 4 处（均测试侧修正，主码零改动）：① mock stub 用 anyString() 不匹配 null scopeJson → 改 any()；② subtree=true 的树用例误按「未展开集合」授予可见集 → 按组件展开语义授予 {1,2}（**为什么现在对了**：展开语义归组件，服务层直接消费展开结果——与本测对 DataPermService 的 mock 边界一致）；③ metrics 实例编码回显 matcher anyString() 不匹配 null unit/instanceCode → nullable(String.class)；④ 审计 writer 的 executor 拒绝用例对「同步兜底后重复 run」断言冗余 → 按实际契约（拒绝→兜底插入一次）收敛。

### 2.2 验收 2：升级脚本二次幂等 + 两表结构 + 种子行 —— 通过

第一次执行：

```
$ bash scripts/sql-exec.sh ../resources/sql/upgrade/20261122_iot_openapi_domain.sql
OK(0)
OK(0)
OK(0)
OK(1)
DONE
```

第二次执行（幂等复核）：

```
$ bash scripts/sql-exec.sh ../resources/sql/upgrade/20261122_iot_openapi_domain.sql
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

两次均 DONE；第二次 0 行变更=不改已有行（INSERT IGNORE 命中 uk_app_key 既有种子行）。逐条：SET NAMES + CREATE TABLE iot_api_client + CREATE TABLE iot_api_call_log + 种子 INSERT。

表结构核对（`SHOW CREATE TABLE`，附录 A p34 逐字 + 两差异列）：

```
iot_api_client	CREATE TABLE `iot_api_client` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '调用方 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `app_key` varchar(64) NOT NULL COMMENT '调用方标识（全局唯一）',
  `secret_hash` varchar(128) NOT NULL COMMENT '密钥哈希（不存明文）',
  `secret_encrypted` varchar(512) DEFAULT NULL COMMENT '密钥 SM4 密文（{v1} 前缀——本任务新增列与上游差异：HMAC 验签需服务端持原文，上游 secret_hash 无法支持）',
  `client_name` varchar(100) NOT NULL COMMENT '调用方名称',
  `scope` json DEFAULT NULL COMMENT '授权模型范围（空为无权限）',
  `rate_limit` int DEFAULT '0' COMMENT '限流（次/分钟），0 不限',
  `ip_whitelist` varchar(500) DEFAULT NULL COMMENT 'IP 白名单（CSV，NULL=不限——本任务新增列与上游差异，ADR-0013 增补 A7）',
  `expire_time` datetime DEFAULT NULL COMMENT '密钥到期时间',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用',
  `remark` varchar(500) DEFAULT NULL COMMENT '备注',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_app_key` (`app_key`),
  KEY `idx_tenant` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='对外调用方表'

iot_api_call_log	CREATE TABLE `iot_api_call_log` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '日志 ID',
  `tenant_id` bigint DEFAULT NULL COMMENT '租户 ID',
  `app_key` varchar(64) DEFAULT NULL COMMENT '调用方标识',
  `api_path` varchar(200) DEFAULT NULL COMMENT '接口路径',
  `param_digest` varchar(500) DEFAULT NULL COMMENT '参数摘要（脱敏）',
  `result_code` int DEFAULT NULL COMMENT '响应业务码',
  `cost_time` bigint DEFAULT NULL COMMENT '耗时（毫秒）',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '调用时间',
  PRIMARY KEY (`id`),
  KEY `idx_app_time` (`app_key`,`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='对外调用审计表（可选）'
```

逐字核对：两表与附录 A p34 pypdf 提取逐字一致（字形规范化不改语义），唯一键与二级索引（uk_app_key/idx_tenant/idx_app_time）在位；两差异列名/长度/注释与头注登记一致。`COLLATE=utf8mb4_0900_ai_ci` 为 MySQL 8 默认随库继承（与点位/指标域脚本同口径）。

种子行核对（冒烟夹具更新后终态；种子原始 rate_limit=3，冒烟 S0 按序列确定性需要调至 8）：

```
$ SELECT id, tenant_id, app_key, client_name, rate_limit, ip_whitelist, expire_time, status, deleted FROM iot_api_client WHERE app_key='test_client';
id	tenant_id	app_key	client_name	rate_limit	ip_whitelist	expire_time	status	deleted
1	1	test_client	测试调用方（TASK-0051 冒烟专用）	8	null	null	1	0
```

### 2.3 验收 3：dev 冒烟（8098，dev 档）—— 通过

启动：`dev.ps1 start`（8098，jar 为终码打包）→ 冒烟脚本 `backend/target/t0051_smoke.py`（签名链 python hmac 本地重算，与 HmacAuthStrategy 同口径）原始输出（终轮完整贴回；EXIT=0）：

```
=== S0) 夹具：scope=F01 子树(id=2, subtree=true) + rate_limit=8 ===
OK   | S0 scope/rate_limit 更新（UPDATE OK）  OK(1) / DONE
OK   | S0b 夹具确认 rate_limit=8  [{'app_key': 'test_client', 'rate_limit': '8', 'status': '1', 'deleted': '0'}]
    审计基线 id=62（DB 时区 UTC，S6 比对走 id 游标而非时间窗）
    （等待限流固定窗重置 62s……）
=== S1) 正确签名调 models/tree → 200，只返回 F01 子树（TC-0002 步骤 6 语义） ===
OK   | S1a HTTP 200 + body 200  200 200 操作成功
OK   | S1b 树非空且全部节点在 scope 子树内  codes=['F01', 'F01.WS01', 'F01.WS01.EQ001', 'F01.WS02', 'F01.WS13', 'F01.WS13.EQ14A', 'F01.WS13.EQ14B', 'F01.WS13.EQ36', 'F01.WS13.R2DQ', 'F01.WS13.R2OKQ']
OK   | S1c scope 外实例（F02/FMIX）不出现  交集=set()
=== S2) scope 外 fullKey（F02.WS14.EQ14C.ELEC_CONS）→ HTTP 200 + body 6003 ===
OK   | S2 scope 外 6003（HTTP 200）  200 6003 fullKey 不属于调用方授权范围（scope 外 6003）
=== S2b) 未登记 fullKey → 空集如实（数据不存在类返回空集，与 6003 相区分） ===
OK   | S2b 空集 200  200 200 data=[]
=== S2c) realtime（scope 内 fullKey，Redis/TDengine 本期无数据）→ 空集如实 ===
OK   | S2c realtime {} 且 scope 外键裁剪  200 200 data={}
=== S2d) items/history 跨度 8 天未带 interval → 400（限值契约） ===
OK   | S2d 400（跨度超限）  200 400 items/history 原始粒度跨度上限 7 天（当前 8 天），超限必须带 interval 降采样
=== S2e) points/history 未绑点位（bound_instance_id NULL）→ 6003（中台 fail-closed） ===
OK   | S2e 6003  200 6003 点位不属于调用方授权范围（scope 外 6003）
=== S3) 篡改签名 → HTTP 403 + body 6001 ===
OK   | S3 403+6001  403 6001 签名不匹配
=== S4) 过期时间戳（-400s，超防重放窗口）→ HTTP 403 + body 6001 ===
OK   | S4 403+6001  403 6001 签名时间戳超出防重放窗口（±300 秒）
=== S5) dict/sync（占位 200）+ 限流触发（rate_limit=8 固定窗） ===
OK   | S5a dict/sync 前 2 次 200（占位响应 instances/items 空清单）  codes=[200, 200] data={'since': 1763712000000, 'instances': [], 'items': []}
OK   | S5b 第 3 次 → 6002（窗口计数 9 > 8，since 原样回显已由 S5a 契约覆盖）  200 6002 请求超出调用方限流上限（次/分钟），请稍后重试
=== S6) 审计核对：iot_api_call_log 行数与结果码分布（异步落库轮询，id 游标） ===
OK   | S6a 审计行数=11（403/6003/6002 类全量记录）  rows=11 dist={200: 5, 6003: 2, 400: 1, 6001: 2, 6002: 1}
OK   | S6b 结果码分布  dist={200: 5, 6003: 2, 400: 1, 6001: 2, 6002: 1} expected={200: 5, 6003: 2, 400: 1, 6001: 2, 6002: 1}
OK   | S6c 认证失败行 tenant_id=NULL（调用方未定位），业务行 tenant_id=1  tenants=['1', 'null']
OK   | S6d cost_time 全部落库  sample={'result_code': '200', 'api_path': '/openapi/v1/models/tree', 'tenant_id': '1', 'cost_time': '593'}
=== S7) 红线复核：prod 预检——test_client 行（脚本种子仍在） ===
OK   | S7 种子行在位且未禁用  [{'app_key': 'test_client', 'rate_limit': '8', 'status': '1', 'deleted': '0'}]

SMOKE DONE
```

审计落库实样（`SELECT ... FROM iot_api_call_log ORDER BY id DESC LIMIT 15`，冒烟轮次切片）：

```
48	1	test_client	/openapi/v1/dict/sync	6002	7
47	1	test_client	/openapi/v1/dict/sync	6002	9
46	1	test_client	/openapi/v1/dict/sync	200	10
45	1	test_client	/openapi/v1/dict/sync	200	11
44	null	test_client	/openapi/v1/models/tree	6001	2
43	null	test_client	/openapi/v1/models/tree	6001	6
42	1	test_client	/openapi/v1/points/history	6003	21
41	1	test_client	/openapi/v1/items/history	400	11
40	1	test_client	/openapi/v1/items/realtime	200	35
39	1	test_client	/openapi/v1/items/history	200	12
38	1	test_client	/openapi/v1/items/history	6003	22
37	1	test_client	/openapi/v1/models/tree	200	32
```

dev 日志 ERROR 分类（全程 8 条，全部为 GlobalExceptionHandler「业务异常」预期拒绝留痕，无未预期错误）：6003 文案 ×6（两轮冒烟的 S2/S2e）、400 文案 ×2（S2d）。

冒烟脚本两处迭代（脚本侧问题，主码零改动）：
1. subprocess 以默认 PATH 调 `bash` 命中 System32 的 WSL bash（cygpath/jshell 链不通 → sql-exec 全空输出）→ 显式 `C:\Program Files\Git\bin\bash.exe`。**为什么现在对了**：sql-exec.sh 依赖 Git Bash 工具链（cygpath 转换 mysql-connector jar 路径），与工具自身文档的运行环境一致。
2. 审计比对最初用时间窗（`create_time >= 本地时刻`）——MySQL 容器时区为 UTC（create_time 落 UTC），本地时刻比对恒空 → 改「冒烟开始时捕获 iot_api_call_log MAX(id) 作游标」。**为什么现在对了**：行数/分布核对以自增主键游标切片，与时区无关、且天然排除历史运行行。

用毕即停：`dev.ps1 stop` → 进程已随轮次收尾停止 → `netstat :8098 LISTENING` 复核无输出。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn package -DskipTests -pl ot-admin -am` → ot-admin-1.0.0.jar（终码）。启动命令（沿 0048 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

干净单轮（日志 /tmp/t0051/prod_clean.log，无任何探针外请求）：

```
2026-11-22T19:55:57.614+08:00  INFO 38308 --- [ot-framework] [main] net.otlab.admin.OtAdminApplication :
    Started OtAdminApplication in 25.541 seconds (process running for 26.315)
ERROR count（启动段+探针后复核）: 0
DICT-SYNC HTTP 200 code 200 | 操作成功 | data= {'since': 1763712000000, 'instances': [], 'items': []}
TAMPER HTTP 403 {'code': 6001, 'message': '签名不匹配', 'timestamp': 1795348568618}
```

- /openapi/v1/dict/sync 签名调用 200 + 占位响应（占位契约：instances/items 空清单 + since 回显）——**prod 验收 4 达成**；审计行同步落库（result_code=200、tenant_id=1、cost_time=323）。
- 篡改签名 403+6001（prod 档鉴权链等价生效，secret_encrypted 在 CRYPTO_DATA_KEY 同值注入下可解密验签）。
- `curl /actuator/health` 聚合 503 = MailHealthIndicator（prod profile 默认 smtp.example.com 不可达）——**上游 profile 既有行为、非本任务引入**（application-prod.yml 邮件段本任务零改动）；`/actuator/health/liveness`、`/actuator/health/readiness` 均 UP，应用本体健康。登记遗留 5 供运维侧处置。
- 用毕即停：进程已 kill → `netstat :8098 LISTENING` 复核无输出。

### 2.5 验收 5：改动范围核对 —— 通过（附并行交错提交说明）

说明：开工 BASE 记录为 `d74d56f`；**BASE 之后、本任务提交之前，并行实现者的 TASK-0076 两笔提交（f0f4533、4b2d59d，19:24 落库）插入了主仓历史**（ingest HTTP 上报端点+网关令牌链+前端半 mock，15 文件，含 frontend/ 与 ingest/——非本任务改动）。故：

- `git diff --name-only d74d56f..HEAD` = 61 文件，其中 15 文件属 TASK-0076 两笔交错提交（越界面属并行任务，不属本任务）；
- **本任务提交自身 diff**：`git diff --name-only 4b2d59d..8348047` = **46 文件**（清单见 §1），逐一核对全部落在 brief 声明范围：openapi 主码包（33，均 `module/openapi/**`）/ dataperm 三文件（仅 ClientScope 消费接线——接口重载+Impl+mapper 显式租户变体）/ `application.yml`（仅新增 ot.openapi.* 三键）/ 测试 8 文件（均 `test/.../module/**`）/ 升级脚本 1（即 brief 指定路径）。不碰 frontend/、ingest/、管理面既有域（只读消费）、workitems 库其他文件；TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。

```
$ git diff --name-only d74d56f..4b2d59d | wc -l
15      ← TASK-0076 交错提交（并行任务面）
$ git diff --name-only 4b2d59d..8348047 | wc -l
46      ← 本任务提交（46 文件全落声明范围，清单见 §1）
```

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| dict/sync 增量语义（since 游标推进、新增/变更/禁用三态清单） | brief 明文「路由占位——增量语义归 TASK-0054」；本期交付 appKey 有效即可的全量字典空实现（instances/items 空清单 + since 回显），契约形状就位 |
| metrics/query 真实数据 | FEAT-0013 指标计算未交付（st_metric_value 无写入方）——brief 既定「空集占位，契约就位」；yoy/mom null→compareNote 的转译语义已实现（读取侧如实转译 NULL，区分语义归计算引擎写值保证） |
| items/history、points/history、realtime 回源的真实数据 | TDengine 192.168.31.145 fd 未修复 + ingest Redis 写链未交付——brief 既定「fail-soft 空集如实，联调归 0068」；查询客户端按 0023 范式实现（连接失败/超时/非 0 code 一律 WARN 空集），0068 修 fd 后无需改契约 |
| 调用方管理 CRUD / scope 授权界面 / 密钥轮换双密钥 / 吊销缓存主动失效 / nonce 幂等表 | REQ-0035 管理面归 TASK-0052（brief：本期无管理界面，脚本内 INSERT 测试调用方为最小开通方式）；nonce 一期只收不验（brief：幂等表登记 0052/后续） |
| OAuth2 策略实现 | ADR-0009 TD-0004：一期签名方式落地，OpenApiAuthStrategy 接口预留替换位（REQ-0027 非目标） |
| pageSize 上限 500 的对外消费面 | §3.4 pageSize≤500 为通用分页规范；§3.5 六接口行均无 pageNum/pageSize 参数（树/字典不分页、其余为显式批量/跨度限值形态）——无参数面可承载（登记，不自行加参数） |
| 审计保留期 180 天按月归档 | REQ-0027 口径归后续运维任务交付；本任务交付审计写入链（表结构与异步写入），不承载归档逻辑（脚本头注登记） |
| 缓存监控 openapi:realtime: 前缀命名空间登记 | brief 允许的 application.yml 改动面仅 ot.openapi.* 键族（ot.cache.monitor 登记项超出）；键现走「未归类」兜底分组，登记建议随 0068 联调一并落 |

## 4. 偏离计划之处

1. **secret_encrypted 的加密密钥口径**：brief 字面「SM4 加密明文，框架 sm4.key」——落地用 `ot.crypto.data-key`（SmDataCrypto，{v1} 前缀）而非登录传输密钥 `sm4.key`。理由：SmDataCrypto 类注释强制「存储加密密钥与登录密码传输密钥用途不同、独立不复用，杜绝密钥复用」，iot_channel_credential.ciphertext（SM4 密文 {v1} 前缀）为同面先例；brief 最终裁决的本体口径（服务端解密得 secret 原文重算 HMAC 比对）已照办。OpenApiSeedLiteralTest 钉住种子密文与组件/默认 data-key 的一致性。
2. **secret_hash 快速校验括注未参与验签**：brief 括注「SHA256(提交 secret) 比对通过才走 SM4 解密」在纯 HMAC 通道无「提交 secret」可算（调用方提交的是签名而非密钥原文，服务端无法由签名反算 SHA256(明文)）——secret_hash 列照建（SHA256(test_secret) 已落种子），本期不参与验签，保留给 0052 轮换/快速校验消费（升级脚本头注登记）。
3. **dataperm 显式租户重载**（接口+Impl+mapper 变体，§1.3）：/openapi 机器请求无登录会话——TenantContext 取不到租户、多租户插件跳过注入，组件原重载的会话守卫会把全部机器调用 fail-closed 拒绝且插件缺位有跨租户放大风险。落地为「租户显式带参 + SQL 显式 tenant_id 条件收敛」（增补第 8/9 条 fail-closed 同向——隔离不依赖会话上下文，比上下文守卫更硬）；scope 三态语义、子树展开截断、deptIds 恒空集与登录用户路径完全复用同一组私有方法（组件消费不重写，两路径零行为分叉）；会话路径原方法与既有 DataPermClientScopeTest 零改动（2059 全量绿佐证）。
4. **6002 的 HTTP 状态**：brief 只定「认证类=403、授权类=200+6003」且明文「传输层 403 仅认证失败形态」——限流既非认证失败形态，落 HTTP 200 + body 6002（映射表见 §0，报告登记）。
5. **集合接口 scope 外键静默裁剪**（realtime/metrics）：REQ-0027 验收 1 与 ADR-0016 裁决 2 的集合形态口径（scope 外/数据不存在返回空集，不回显键存在性——避免给调用方全租户 fullKey 枚举探测面）；6003 仅用于单目标接口（items/history、points/history，详情类语义——TC-0017 步骤 3 的触发面）。
6. **限流计数位置=验签成功后**：brief 未细定；未验证身份的请求不占调用方额度，攻击者无法凭伪造 appKey 洪泛把合法调用方顶到 6002（DoS 放大收窄）；TC-0017「并发超限→6002」场景以正确签名触发，兼容。
7. **红线二转正形态**：本套件无 SpringBootTest 先例（2036 基线全为 standalone 单测）——端到端义务以「真实链装配」履行（真实 HMAC 策略+真实过滤器+真实控制器+真实服务+真实 dataperm 显式租户重载，iot_api_client 种子夹具真实签名请求；mock 仅持久层与 Redis）。显式 tenant_id SQL 的真实收敛由 dev 冒烟/prod 回归（同一套代码路径）复核。
8. **冒烟种子 rate_limit 调整**：脚本种子 rate_limit=3 → 冒烟 S0 夹具按序列确定性需要 UPDATE 至 8（前置 6 次已验签调用 + 限流触发窗口计数控）——夹具动作非主码行为，登记于脚本头注与冒烟输出。
9. **并行交错提交**（见验收 5）：TASK-0076 两笔提交插在 BASE 与本任务提交之间——本任务全程未 rebase/改写他人历史，仅按线性历史落自己的提交。

## 5. 遗留问题

1. **dev 库冒烟残留**（不影响仓库与提交）：test_client 行 scope= {"instanceIds":[2],"subtree":true}、rate_limit=8（冒烟终态；0052 管理界面交付后按正式口径回收/重配）；iot_api_call_log 累计审计行（正常业务数据，可作 0052/0054 的对照面）。
2. **0068 联调清单**（本任务交付契约，联调归 0068）：① TDengine 192.168.31.145 fd 修复后复核 items/history、points/history、metrics/query、realtime 回源四条读路径（fail-soft 客户端已就位，关注降采样 INTERVAL 窗口语义与 quality 列口径）；② Redis 实时值键契约对齐：键 `openapi:realtime:{tenantId}:{fullKey}`、值 JSON {"v","q","ts"}（ot.openapi.realtime-cache-key-prefix 可覆盖），ingest 写链落地时以本契约为锚；③ 缓存监控 openapi:realtime: 前缀登记（现走「未归类」兜底）。
3. **0052 交接清单**：调用方 CRUD/scope 授权界面/密钥轮换（双密钥并行一周——secret_hash 列的快速校验消费点）/吊销缓存主动失效（REQ-0035 验收 4）/nonce 幂等表；test_client 种子行届时回收。
4. **上游修订清单建议追加**（升级脚本头注已登记「与上游差异」）：① iot_api_client 增列 secret_encrypted（HMAC 验签需服务端持原文）；② iot_api_client 增列 ip_whitelist（ADR-0013 A7）；③ 测试调用方种子行（上游无种子数据）。
5. **prod /actuator/health 聚合 503（非本任务引入）**：MailHealthIndicator 对 prod 默认 smtp.example.com 探活失败（application-prod.yml 邮件段上游默认值，本任务零改动）；liveness/readiness 均 UP、应用功能正常。建议运维任务处置（注入真实 MAIL_HOST 或关闭 mail 健康指示），不阻塞本期验收。
6. **环境备注（非本任务引入）**：WSL docker 空闲自停导致 3307 短暂拒连（重试即恢复）；`dev.ps1 start` 在 jar 已存在时不重建——冒烟前需显式 `mvn package` 或 `dev.ps1 build`；本机 PATH 的 `bash` 命中 System32 WSL bash，脚本化调用 sql-exec.sh 需显式 Git Bash 全路径（smoke 脚本已内置）。

---

## 6. 修订轮（评审退回，2026-11-22）

- 评审报告：`reports/TASK-0051-review.md`（§13 diff 评审：Critical 0 / Important 4 / Minor 11）
- 修订提交：`a1c6649`（main，注明 TASK-0051 修订轮；13 文件——主码 8 改 + application.yml 1 改 + 新增 OpenApiPage/OpenApiYmlStructureTest/OpenApiTimeseriesClientTest/OpenApiRealtimeStoreTest 4 + 测试适配 2，全部限 openapi 域 + application.yml + 测试，契约不变）
- 结论先行：**必修四项全部落地，补充验收全部通过**。全量复跑 2112 = 2099 + 修订净增 13，0 失败 0 错误 0 跳过 BUILD SUCCESS；修订轮冒烟复核 R1-R5 全绿（NoOp 回归 / DESC 最新行实证 / 回填键 TTL=59 / 超限 20000 行 + X-Truncated 头 / 正控制键出现）；自起实例两轮均 `dev.ps1 stop` 停净、8098 复核无监听。Minor 11 条不动记 deferred（建议 0052/0068 承接，评审 §二 Minor 表为登记源）。

### 6.1 必修四项落地说明

| # | 修复 | 落点 |
| --- | --- | --- |
| I-1 | `ot.openapi` 块移出 dataperm 块成为 ot 下顶层独立块；`ot.dataperm.cache-enabled`/`cache-ttl-seconds` 两键回归 dataperm 命名空间（0006 缓存兜底开关恢复生效，OT_DATAPERM_CACHE_ENABLED/TTL_SECONDS 环境变量恢复权威） | `application.yml`（结构修正）；`OpenApiYmlStructureTest`（2 例：解析 application.yml 断言 dataperm 块恰三键含 cache-enabled/ttl + openapi 顶层独立块恰四键且不含 dataperm 两键——单测不加载 Spring 配置的命名空间面由本测试锁定） |
| I-2 | `queryItemLatest` 弃「升序取尾」改为专用 SQL `ORDER BY ts DESC LIMIT 1`（真实最新行——旧实现在行数>20000 后 realtime 回源永久返回第 20000 条旧值） | `OpenApiTimeseriesClient.queryItemLatest`；`OpenApiTimeseriesClientTest`（内置 fake TDengine REST——JDK HttpServer，按收到的 SQL 形态应答并回录 SQL）：断言生成 SQL 含 `ORDER BY ts DESC LIMIT 1`、返回 ts=60000 最新行（fake 全表 ts 1..60000，升序取尾会拿 20000 旧值） |
| I-3 | realtime 回填键带短 TTL：新键 `ot.openapi.realtime-cache-ttl-seconds`（默认 60s，<=0 误配回落 60 并 WARN）——键过期即回源最新一条，杜绝 ingest 停写后旧值长期充当「实时值」与查询过的 fullKey 无限累积 | `OpenApiProperties` + `application.yml` + `OpenApiRealtimeStore.backfill`（`setCacheObject(key, payload, ttl, SECONDS)`；值载体改普通 Map——RedisTemplate Jackson 序列化面最稳）；`OpenApiRealtimeStoreTest`（4 例：TTL 60 默认/误配回落/命中不回源不回填/源空不回填） |
| I-4 | 行数上限截断判定 fetch-one-extra：SQL `LIMIT ROW_LIMIT+1` 取数，命中行数>ROW_LIMIT 即截断事实成立 → `OpenApiPage`（rows + truncated）→ 服务层透传 → 控制器以响应头 **`X-Truncated: true`** 携带（体形状与 §3.5 不变，标记走传输头）；items/history、points/history、metrics/query 三读路径同契约；类 javadoc/头注/OpenApiCodes.HEADER_TRUNCATED/控制器 @Operation 四处声明 | `OpenApiPage`（新）、`OpenApiTimeseriesClient`（三查询改页返回 + FETCH_LIMIT=20001）、`OpenApiQueryService/Impl`（三方法返回页）、`OpenApiController`（markTruncated 设头）；`OpenApiTimeseriesClientTest`（3 例：超限 truncated=true 收敛 20000 行升序前缀 / 未超限 false / points+metrics 同契约）+ `OpenApiQueryServiceImplTest.historyTruncationFlagPassthrough`（服务层不吞标记） |

### 6.2 补充验收原始输出

#### 6.2.1 验收 1：官方口径全量复跑 —— 通过（含红转绿记录）

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[INFO] Tests run: 2112, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
EXIT=0
```

- 2112 = 修订前 2099 + 修订净增 13（yml 结构回归 2 + 客户端 fake TDengine 6 + 回填 TTL 4 + 截断透传 1）。
- 红转绿记录（修订轮单测迭代 2 处，均测试侧问题，主码零改动）：
  1. `OpenApiTimeseriesClientTest.pointsAndMetricsTruncation` 首轮失败——根因：fake TDengine 行宽统一 3 列（ts/value/quality），metrics 应答需 4 列（ts/value/yoy/mom），行全部被 `<4` 守卫跳过返回 0 行。修复：fake 按 SQL 形态（含 yoy）生成 4 列行。**为什么现在对了**：fake 应答与 st_metric_value 真实列序一致。
  2. `OpenApiQueryServiceImplTest.historyTruncationFlagPassthrough` 首轮 NPE——根因：stub 用 `anyString()` 不匹配 null interval/agg（原始粒度查询两参为 null）→ mock 返回 null。修复：改 `nullable(String.class)`。**为什么现在对了**：matcher 与实参 nullability 对齐。
- 既有 3 例（history/points/metrics 返回类型断言）随 OpenApiPage 适配（`page.rows()`/`page.truncated()`），断言语义不变。

#### 6.2.2 验收 2：dev 冒烟复核 —— 通过（自起实例，用毕即停）

修订轮冒烟 `backend/target/t0051rev_smoke.py`（自管两轮 dev 启停 + 内置 fake TDengine REST :18041 + wsl docker redis-cli 核对），终轮完整原始输出（EXIT=0）：

```
=== Phase A：OT_DATAPERM_CACHE_ENABLED=false + OT_TDENGINE_BASE_URL=fake:18041 ===
OK   | dev 实例已起（8098，cache-enabled=false + fake TDengine）
OK   | R1a dev 登录 200  200 200
OK   | R1b 拉树 200（resolve 链路走通）  200 200
OK   | R1c Redis 无 *dataperm* 键（NoOpVisibleScopeCache 生效——I-1 修复后开关重新有权威性）  keys=[]
OK   | R2a realtime 200 且命中回源  200 200 data={'F01.WS13.R2OKQ.ELEC_METER': {'v': 123.4, 'q': 0, 'ts': 1763712005000}}
OK   | R2b 回源值=真实最新行（ORDER BY ts DESC LIMIT 1 实证——I-2 旧实现会取 ts=1000000 旧值）  item={'v': 123.4, 'q': 0, 'ts': 1763712005000}
OK   | R2c 回源 SQL 为降序取 1 行  SELECT ts, value, quality FROM iot_default.st_item_value WHERE tenant_id = 1 AND full_key = 'F01.WS13.R2OKQ.ELEC_METER' ORDER BY ts DESC LIMIT 1
OK   | R3a 回填键存在  keys=['ot_framework:openapi:realtime:1:F01.WS13.R2OKQ.ELEC_METER']
OK   | R3b 回填键 TTL ∈ (0,60]（键过期即回源——I-3）  TTL=59
OK   | R4a items/history 200  200 200
OK   | R4b 超限行收敛：data 恰 20000 行（升序前缀）  len=20000 head={'ts': 1, 'v': 1.0, 'q': 0} tail={'ts': 20000, 'v': 20000.0, 'q': 0}
OK   | R4c 响应头 X-Truncated: true（对外契约不允许静默丢数据——I-4）  headers={'X-Truncated': 'true'}
=== Phase A 实例已停 ===
=== Phase B：默认环境变量（不翻开关）——命名空间修复后开关两向权威性正控制 ===
OK   | R5a dev 登录 200  200 200
OK   | R5b 拉树 200  200 200
OK   | R5c Redis 出现 *dataperm* 键（RedisVisibleScopeCache 正常装配——正控制：I-1 修复未把开关方向修反）  keys=['iot:dataperm:scope:1:0']
=== Phase B 实例已停 ===
（fake TDengine 已停、dev.env 临时文件已删）

REV-SMOKE DONE
```

复核点逐条对应裁决要求：
- **I-1（yml 修正后 NoOp 走通）**：R1c（cache-enabled=false 启动 → 登录拉树 → Redis 无 `*dataperm*` 键 = NoOpVisibleScopeCache 条件装配命中，0006 语义回归）+ R5c（不翻开关 → 键出现 = RedisVisibleScopeCache 正常装配，开关两向权威性正控制）。
- **I-2（queryItemLatest 超限场景返回最新行）**：R2b/R2c——fake TDengine 数据面全表 ts 1..60000（远超 20000），realtime 回源返回 ts=1763712005000 最新行，且回录 SQL 实证 `ORDER BY ts DESC LIMIT 1`（升序取尾旧实现在此场景会返回 ts=20000 旧值）；单元侧另有 fake 服务 SQL 断言与「ts=60000」断言双保险。
- **I-3（回填键带 TTL，redis-cli 核对）**：R3a/R3b——`redis-cli --scan --pattern '*openapi:realtime:*'` 命中回填键，`redis-cli TTL` = 59 ∈ (0,60]。
- **I-4（截断标记）**：R4b/R4c——20001 行数据面下 items/history 返回恰 20000 行（ts 1..20000 升序前缀）+ 响应头 `X-Truncated: true`；另 R2 路径（LIMIT 1 不截断）无头、单测覆盖未超限 false 分支。

冒烟脚本迭代 1 处（脚本侧，主码零改动）：login 响应 token 字段实取 `data.token`（首版按 tokenValue 取 → KeyError）。**为什么现在对了**：以实测响应形状为准（`/auth/login/test` 返回 data.token）。

用毕即停：Phase A/B 两轮均 `dev.ps1 stop`（脚本内联）→ 终态 `netstat :8098 LISTENING` 复核无输出；fake TDengine 随脚本退出 shutdown；临时 dev.env 已删除（gitignored 路径，未入库）。

### 6.3 修订轮改动文件清单（13，全落契约面）

| 类别 | 文件 |
| --- | --- |
| 主码改（7） | `module/openapi/OpenApiCodes.java`（HEADER_TRUNCATED）、`config/OpenApiProperties.java`（realtimeCacheTtlSeconds）、`controller/OpenApiController.java`（三端点页返回 + markTruncated）、`service/OpenApiQueryService.java` + `service/impl/OpenApiQueryServiceImpl.java`（三方法返回 OpenApiPage）、`support/OpenApiRealtimeStore.java`（回填 TTL + Map 载体）、`support/OpenApiTimeseriesClient.java`（DESC latest + fetch-one-extra + 页返回） |
| 主码增（1） | `module/openapi/support/OpenApiPage.java`（截断页值对象） |
| yml（1） | `application.yml`（I-1 结构修正 + I-3 新键 realtime-cache-ttl-seconds） |
| 测试增（3） | `test/.../openapi/config/OpenApiYmlStructureTest.java`、`support/OpenApiTimeseriesClientTest.java`、`support/OpenApiRealtimeStoreTest.java` |
| 测试改（2） | `service/impl/OpenApiQueryServiceImplTest.java`（页断言适配 + 截断透传例）、`redline/ClientScopeRedlineTest.java`（pointsHistory 调用签名适配一行） |

### 6.4 修订轮未做项 / 遗留

- **Minor 11 条全部不动记 deferred**（裁决明文）：M-1 6001 文案收敛、M-2 签名串纳 path/nonce 幂等、M-3 认证通道前置限频、M-4 finally 收敛、M-5 XFF 拓扑语义、M-6 死代码两处、M-7 database() IAE 包裹、M-8 metric unit 来源、M-9 停机 awaitTerminationSeconds、M-10 dict 强类型 DTO、M-11 骨架披露收窄——承接建议：M-1/M-2/M-3/M-5/M-11 归 0052（对外面运营决策与对接期），M-4/M-7/M-9/M-10/M-6/M-8 随 0068 联调/对应域任务一并清（评审 §二 Minor 表为登记源）。
- I-2/I-3/I-4 的真实数据路径已由 fake TDengine 实证（dev 冒烟），0068 真联调时无需重复回归——0068 brief 如引用本验收可直接引用 §6.2.2。
