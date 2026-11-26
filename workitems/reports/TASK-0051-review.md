# 评审报告：TASK-0051（/openapi/v1 对外面——签名/scope/限流/审计四链）

- 评审者：fresh reviewer（不共享协调者上下文，no-subagents，只评审不修改）
- 评审对象：单提交 `8348047`（父 `4b2d59d`），diff 包 46 文件约 4813 行，**已完整逐行读入**
- 依据：L1-execute §13（对外面最严）+ 实施 brief 契约 + ADR-0013 A7 / ADR-0016 裁决 5、增补 8/9 / ADR-0009 TD-0004；关键交叉断言均在代码库实文件反查（application.yml、DataPermProperties、NoOp/RedisVisibleScopeCache 条件装配、RedisService.incrementWithTtl/setCacheObject、TdengineAdmin.databaseName、SysTenant 校验、GlobalExceptionHandler、VisibleScope）
- 越界反查：46 文件逐一清点（33 主码 + 3 dataperm 接线 + application.yml + 8 测试 + 1 升级脚本），全部落在 brief 声明范围——与报告 §1 清单一致，无越界

## 一、契约一致性核对（①项，摘要）

| 契约点 | 结论 |
| --- | --- |
| 签名链 | HMAC-SHA256(secret, appKey+timestamp+bodyHash) 与 brief 定版串形态一致；bodyHash=SHA256(body)、空体恒定摘要；`MessageDigest.isEqual` 恒时序比对 + hex 归一；防重放 ±300s 对称窗口（brief 5 分钟）；校验序（缺头→时间戳→加载→禁用→到期→IP→验签）合理，时间戳检查前置于 DB 加载（防垃圾请求打库） |
| secret_encrypted 双列 | 论证成立：HMAC 验签需服务端持原文，secret_hash（不可逆 SHA256）无法支持——「与上游差异」必要，脚本头注/行注释/Row javadoc 三处登记；secret_hash 不参与验签的偏离（brief 括注在纯 HMAC 通道无「提交 secret」可算）论证成立，报告偏离 2 登记；解密失败 fail-closed 6001 不降级 500，WARN 不落 appKey/密文 |
| 错误码映射 | 逐出口核对：全部认证类失败→过滤器 HTTP 403+6001；限流 6002=HTTP 200（报告偏离 4，与「传输层 403 仅认证失败形态」自洽）；单目标 scope 外 6003=HTTP 200+body 6003（BusinessException→GlobalExceptionHandler，已核实该 handler 返回 `Result` 即 HTTP 200）；限值类=400。边界一致，无错归类出口 |
| 限流 | 键 `rate_limit:openapi:<appKey>` 落既有 rate_limit 命名空间；`incrementWithTtl` 经实查为 Lua 原子 INCR+TTL（BUG-0053 口径），并发正确；0 不限；Redis 故障 fail-open 有论证（QoS 非安全边界）；验签成功后计数（报告偏离 6，DoS 放大收窄论证成立） |
| scope 裁剪 | 显式租户重载复用组件同一组私有方法（三态/子树展开截断/deptIds 恒空集零分叉）；tenantId=null fail-closed；新 mapper 变体 `@InterceptorIgnore(tenantLine)` + 显式 `tenant_id = #{}` 条件——隔离不依赖会话上下文，符合增补第 9 条同向；子树展开上限（subtree-max）经 expandSubtree 保留 |
| 六接口形状 | VO 字段与报告 §0 的 §3.5 提取一致（realtime v/q/ts、history ts/v/q、metrics 七字段+compareNote NON_NULL、dict since/instances/items）；限值矩阵（realtime≤200、原始 7 天、points 31 天、interval/agg 白名单、period 白名单、start<end）与 §3.5 行对齐；pageSize 500 无参数面承载——已按 §3.5 实际行登记，非违约 |
| dict/sync 占位 | appKey 有效即可（不走 scope）、全量空实现+since 回显，与 brief 裁决一致 |
| 审计 | finally 全量异步落库（200/400/6001/6002/6003/500 统一）；403 类含失败全记（冒烟 S6 6001×2 行、认证失败行 tenant_id=NULL 实证）；INSERT 显式列+显式 tenant_id（异步线程无会话）；param_digest=查询串截断 500+空体不落指纹，无凭据面；CallerRuns+shutdown 同步兜底 |
| 泄露面逐路径扫 | 错误回显（见 M-1）、日志（解密失败仅密文长度、TS 客户端 WARN 摘要 200 字符、无 secret/签名头落日志——通过）、审计（无凭据/密文面——通过）、响应（scope 外键静默裁剪不回显存在性；骨架披露见 M-11）、Swagger @Tag 生产暴露面沿既有 apidoc.expose 双保险——通过 |
| 过滤器路径覆盖 | 注册 `/openapi/*` + 内部前缀 `/openapi/v1/` 判定；六控制器全在 `/openapi/v1/` 下；`requireIdentity` fail-closed 兜底过滤器缺位；精确路径 `/openapi/v1`（无尾斜杠）不匹配前缀→框架 404，无未设防数据路径 |
| SQL 参数化 | MySQL 侧全部 `#{}` 绑定 + foreach；TDengine 侧编码字面量 `[A-Za-z0-9_.]+` 白名单前置、interval/agg 拼接前二次白名单、tenant_id/时间窗数值拼接——注入面封死（IAE 边缘见 M-7） |
| 测试真实性 | 新增计数与净增 63/转正 3 的口径与 surefire 汇总自洽；红线转正为真实链装配（真策略/过滤器/控制器/服务/dataperm 重载，mock 仅持久层与 Redis）；单测矩阵覆盖 brief 测试口径全部条目；冒烟/prod 实证 6001/6002/6003/400/200 全谱 |

**②项质量缺陷**：见下清单。整体判断：四链主结构（签名/防重放/scope 三态/限流/审计全量/限值契约）实现与 brief 契约一致且质量较高；Important 全部集中在**配置与数据回源路径的边界缺陷**（其中 3 项被本期数据空集掩盖，0068 联调前必须修复），无鉴权绕过、无注入、无租户隔离漏洞。

## 二、发现清单

### Important（4）

| # | 出处 | 说明 |
| --- | --- | --- |
| I-1 | `backend/ot-admin/src/main/resources/application.yml`（ot.openapi 块插入位置，diff 行 2891-2902；已实读仓库现文件确认） | **ot.openapi 块插在 ot.dataperm 块中间，把既有两键实际移入 openapi 命名空间**：`cache-enabled`/`cache-ttl-seconds` 现为 `ot.openapi.*` 子键，`ot.dataperm.cache-enabled` 消失——TASK-0006 交付的故障兜底开关静默失效（`NoOpVisibleScopeCache` 的 `@ConditionalOnProperty("ot.dataperm.cache-enabled", havingValue="false")` 永不命中，`RedisVisibleScopeCache` `matchIfMissing=true` 恒装配），`OT_DATAPERM_CACHE_ENABLED`/`OT_DATAPERM_CACHE_TTL_SECONDS` 两个环境变量全部失效（TTL 恰好靠 DataPermProperties 默认 600 侥幸不变）。brief 只允许「新增 ot.openapi.* 键族」，此改动字面全是新增、语义上改写既有键。修复=整块移出 dataperm 之后即可；单测不加载 yml、prod 冒烟未翻转开关，故全绿仍漏 |
| I-2 | `openapi/support/OpenApiTimeseriesClient.java` `queryItemLatest`（经 `queryItemHistory` ORDER BY ts ASC LIMIT 20000 取末行） | **「最新一条」在行数>20000 时取错行**：按 ts 升序取前 20000 行的最后一行——1s 采样约 5.5 小时即超 20000 行，此后 realtime 回源返回的是第 20000 条旧值而非真实最新值；应 `ORDER BY ts DESC LIMIT 1`。本期 TDengine fail-soft 空集掩盖，0068 联调数据一通即错 |
| I-3 | `openapi/support/OpenApiRealtimeStore.java` `backfill`（`setCacheObject(key, value)` 无 TTL） | **回填键永不过期**：①Redis 键随查询过的 fullKey 无限累积（含已删数据项，无回收）；②ingest 停写后旧值长期留缓存充当「实时值」（调用方仅能靠 ts 自辨）。与代码库自身口径相悖——DataPermProperties 注释明文「直写无 TTL 的 SET 会产生永不过期的永久键」、BUG-0053 同危害类。回填应带短 TTL（0068 契约锚口径一并定版） |
| I-4 | `OpenApiTimeseriesClient` 三查询共用的 `ROW_LIMIT=20000` | **对外序列静默截断无标记**：原始粒度 7 天窗（≥每 30s 一点即超限）只返回前 20000 行，无续传游标/截断标志/超限拒绝，调用方拿到「看似完整」的截断序列——对外数据完整性契约缺陷。0068 前必须三选一：分页游标 / 超限 400 / 响应携带 truncated 标记 |

### Minor（11）

| # | 出处 | 说明 |
| --- | --- | --- |
| M-1 | `auth/HmacAuthStrategy.authenticate` 6001 文案矩阵 | 存在性探测面：未知/已删统一「调用方不存在」，但禁用/到期/IP 白名单外/密钥未配置文案互异——未验签请求可区分「appKey 存在与否」（javadoc「无存在性差异泄露」言过其实，仅覆盖未知/已删两支）；appKey 枚举无数据风险，建议 0052 收敛为统一文案 |
| M-2 | `auth/OpenApiCredentials.signaturePayload` | 签名串不绑定 method/path（brief 契约即此形态）：300s 窗口内捕获的 GET 签名因六端点 bodyHash 恒同（空体）可跨端点重放；与 nonce 只收不验同属一期既定口径，建议随 0052 nonce 幂等表一并把 path 纳入签名串 |
| M-3 | `filter/OpenApiAuthFilter` + `ratelimit/OpenApiRateLimiter` 计数位置 | 验签前置通道无节流：伪造签名洪水每请求打一次 `selectByAppKey`（uk_app_key 索引点查）——限流在验签成功后才计数，认证通道自身无 per-IP 频控；DoS 面为 DB 点查压力，建议 0052 评估前置粗粒度 IP 限频 |
| M-4 | `OpenApiAuthFilter.doFilter` finally/catch | `copyBodyToResponse()` 先于审计提交且不捕获 IOException——客户端断连等 IO 失败会丢审计行并吞原始异常；认证失败路径 `writeJson` 抛 IOException 时 resultCode 被外层 catch 覆写为 500（审计码失真）；finally 内应 try-catch 收敛 |
| M-5 | `OpenApiCredentials.extract`（`request.getRemoteAddr()`） | IP 白名单在反代/网关后恒见代理 IP——ADR-0013 A7 的白名单实际按代理地址配置，X-Forwarded-For 未纳入；部署拓扑语义需在 0052/运维文档明确，否则白名单形同虚设或误伤 |
| M-6 | `OpenApiAuthFilter.requestQueryString`（恒返 null 占位）＋ `mapper/OpenApiInstanceMapper.selectByCodesForTenant`（主码零消费） | 死代码两处：前者占位注释留残，后者与 Point/Metric 变体命名对称但无调用方——删除或接用 |
| M-7 | `OpenApiTimeseriesClient` 三处 SQL 拼接中的 `database(tenantCode)` | `TdengineAdmin.databaseName` 的 IllegalArgumentException 会逃出 `executeQuietly` 的 fail-soft 包装（其只包 `execute`）——tenant_code 创建侧已 `[A-Za-z0-9_]+` 校验（SysTenant @Pattern）实际不可达，但与类 javadoc「任何失败一律 WARN 空集」不变量不一致 |
| M-8 | `service/impl/OpenApiQueryServiceImpl.metricsQuery` | unit 恒 null：iot_metric 无 unit 列、`queryMetrics(..., null)` 恒传——§3.5 示例含 `"unit":"m3"`，字段在 VO 但永不输出；本期空集占位掩盖，FEAT-0013/0068 接线时需补 unit 来源（来源数据项） |
| M-9 | `config/OpenApiFilterConfig.openapiAuditExecutor` | `setWaitForTasksToCompleteOnShutdown(true)` 未配 `setAwaitTerminationSeconds`——停机时排队审计行可能直接丢弃（照抄 ot-core operateLogExecutor 范式，两处同病）；建议补 awaitTerminationSeconds（如 5-10s） |
| M-10 | `dto/OpenApiDictSyncVO`（`List<Object>` ×2） | 弱类型占位：instances/items 元素结构未定形（0054 增量语义交付时必须改强类型 DTO，避免 Object 漏进对外契约） |
| M-11 | `service/impl/OpenApiQueryServiceImpl.modelsTree` 骨架装配（0012 M-7 同构） | scope 外祖先节点的编码/名称经骨架路径披露给窄 scope 调用方（id/code/name/nodeType/status）——与登录用户面语义同构（brief 明文「同登录用户拼装」，实现守约）；但对外面是否收窄骨架披露（如仅返回 id 占位）属契约决策，建议登记待上游/0052 定夺 |

## 三、结论

- Critical 0；Important 4；Minor 11。
- I-1 为必须修复项（一行级修复：openapi 块移出 dataperm 块）；I-2/I-3/I-4 同属 0068 联调前的必修边界缺陷，本期数据空集掩盖了它们的症状，但不修复则联调即触发对外数据错误——建议随本任务退回一并修复（I-1）或显式挂 0068 前置义务（I-2/I-3/I-4，须在 0068 brief 登记为阻塞项，不能只进 deferred）。
- 鉴权四链主结构无 Critical：签名/防重放/恒时序比对/fail-closed 密钥链、scope 三态显式租户隔离（SQL 显式条件收敛）、限流原子窗、审计全量（含 403 类）均与 brief 契约一致并经冒烟/prod 实证。
