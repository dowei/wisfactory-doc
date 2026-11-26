# 复评报告：TASK-0051 修订轮 scoped 复评（I-1~I-4）

- 评审者：fresh reviewer（不向协调者追问、no-subagents、只评审不修改任何源文件）
- 复评对象：修订提交 `a1c6649`（当前 HEAD）；修复 diff 包约 1135 行完整读入；关键断言在仓库实文件逐一反查（application.yml、OpenApiProperties、OpenApiTimeseriesClient、OpenApiRealtimeStore、RedisService、OpenApiController、ClientScopeRedlineTest、DataPermProperties）
- 独立验证：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` 复跑实测 **Tests run: 2112, Failures: 0, Errors: 0, Skipped: 0, BUILD SUCCESS**（与实施报告 §6.2.1 声称一致）
- 范围纪律：只判 I-1~I-4 是否修复与修复本身有无新缺陷；原 11 条 Minor 已 defer 不重开（含 M-7 同类新拼接点，见「新引入缺陷排查」——未恶化）

## 一、结论：四项全部已修复

| # | 判定 | 依据（实文件反查 + 单测 + 修订冒烟 R1-R5） |
| --- | --- | --- |
| I-1 | **已修复** | application.yml 实文件：`ot.openapi` 为 ot 下顶层独立块恰四键（replay/rate-limit/realtime-cache-key-prefix/realtime-cache-ttl-seconds），`ot.dataperm` 回归恰三键（subtree-max/cache-enabled/cache-ttl-seconds）——`OT_DATAPERM_CACHE_ENABLED`/`OT_DATAPERM_CACHE_TTL_SECONDS` 环境变量恢复权威。命名空间锁测试 `OpenApiYmlStructureTest`：SnakeYAML 解析 classpath 真实 application.yml，断言键的**命名空间归属**（dataperm 两键在位 + openapi 块误移键缺席，失败信息解释 NoOp 条件装配后果）——锁的是结构而非值断言；`dataperm==3 / openapi==4` 恰计数为有意的防漂移 tripwire（带说明，质量合格；跨域耦合属测试维护摩擦，不计缺陷）。运行面双向实证：冒烟 R1c（cache-enabled=false → Redis 无 `*dataperm*` 键 = NoOpVisibleScopeCache 条件装配命中，0006 兜底开关恢复生效）+ R5c 正控制（默认启动 → 键出现 = 未把开关方向修反） |
| I-2 | **已修复** | 实文件 `OpenApiTimeseriesClient` 专用 SQL `ORDER BY ts DESC LIMIT 1`（不再经升序 history 路径「取尾」）。fake TDengine（JDK HttpServer）单测双断言：回录 SQL 含 `ORDER BY ts DESC LIMIT 1` + 返回 ts=60000 最新行——fake 数据面 ts 1..60000，「升序取尾」在 LIMIT 20000 下必拿 ts=20000 旧值，**超限场景真实构造并实证**；dev 冒烟 R2b/R2c（60000 行数据面 realtime 回源返回最新行 + 回录 SQL 实证）。既有调用方影响：grep 实证 `queryItemLatest` 唯一调用方为 realtime 回源路径，三条 history 查询保持 `ORDER BY ts ASC` 升序前缀语义不变——降序改动无既有调用方受影响 |
| I-3 | **已修复** | `backfill` 落 `setCacheObject(key, payload, ttl, TimeUnit.SECONDS)`（RedisService line 55 带 TTL 重载实查在位）；`ot.openapi.realtime-cache-ttl-seconds` 默认 60，`OpenApiProperties` 绑定前缀 `ot.openapi` 与 yml 键位一致——**配置键命名空间正确**（不与 dataperm 两键交叉）；<=0 误配回落 60 有单测（backfillTtlFallbackOnBadConfig）；值载体 hutool JSONObject → LinkedHashMap，读取侧 `parse()` 对 VO/JSONObject/Map/String 四形态宽容解析——**回源回填→过期→再回源路径不被破坏**（cacheHitSkipsSource / noBackfillWhenSourceEmpty 两面单测 + 冒烟 R3b 真实 redis-cli TTL=59 ∈ (0,60] 实证） |
| I-4 | **已修复** | 三查询共用 `FETCH_LIMIT=ROW_LIMIT+1`（20001）fetch-one-extra；判定 `truncated = rows.size() > ROW_LIMIT`——**恰好 20000 行时取回 20000、20000 > 20000 为 false，不误标**（判定标准达成）；`OpenApiPage`（rows+truncated，List.copyOf 不可变）服务层透传（metrics 多码 OR 聚合不吞标记，historyTruncationFlagPassthrough 在位）；控制器 `markTruncated` 以 `OpenApiCodes.HEADER_TRUNCATED`（"X-Truncated"）设头、未截断不设头（缺省=false 语义）——字段与响应头对外契约标记一致，类 javadoc/@Operation/OpenApiPage 契约注释四处同步；体形状 `Result<List<…>>` 不变（§3.5 契约保持，标记走传输头）。冒烟 R4b/R4c：20001 行数据面 → 恰 20000 行升序前缀 + 客户端实测响应头 `X-Truncated: true`（经真实过滤器链 ContentCachingResponseWrapper 到达调用方——头透传无衰减） |

## 二、新发现（1 条 Minor）

| # | 级别 | 出处 | 说明 |
| --- | --- | --- | --- |
| N-1 | Minor | `config/OpenApiProperties.realtimeCacheTtlSeconds` javadoc ↔ `support/OpenApiRealtimeStore.backfill` | javadoc 承诺「<=0 视为误配，回落默认 60 **并由调用方 WARN**」，backfill 实现为**静默回落**（无 WARN 日志）——文档与实现不符。回落行为本身正确且有单测/冒烟双证，功能无害；建议补一行 WARN 或修订 javadoc（随 0068 联调顺带清理即可，不阻塞收尾） |

## 三、新引入缺陷排查（brief 指定三面 + Minor 恶化检查）

- **降序查询对既有调用方的影响**：无。queryItemLatest 为专用路径、唯一调用方 realtime 回源；三条 history 查询 ASC 语义不变，升序前缀截断契约自洽。
- **TTL 配置键命名空间**：`ot.openapi.realtime-cache-ttl-seconds` 绑定前缀 `ot.openapi` 正确，与 dataperm 两键无交叉；yml 注释与 OpenApiProperties javadoc 口径一致。
- **断言放松**：无。ClientScopeRedlineTest 的 6003 授权放大断言原样保留（仅加 `MockHttpServletResponse` 参数适配一行）；既有用例 `page.rows()`/`page.truncated()` 为语义等价适配；新增 13 例全部加严（SQL 形态断言、TTL 捕获断言、恰 20000 行收敛断言）。
- **Minor 11 条恶化检查**：I-2 修复使 queryItemLatest 成为 `database(tenantCode)` 字面量拼接的又一处（M-7 同类——IAE 逃出 executeQuietly 包装），但旧实现经 queryItemHistory 同样在 executeQuietly 之外调 database()，危害面等价、未恶化；M-7 本身已 defer。

## 四、判定理由（一句）

四项修复全部落在协调者裁决口径上且经实文件反查、2112 全量绿独立复跑与修订冒烟 R1-R5 三重实证，修复本身仅引入一条文档-实现不符的 Minor（误配 WARN 承诺未实现），不构成回退理由。
