# TASK-0003 diff 评审（fresh reviewer）

- 评审对象：diff 包 `TASK-0003-diff.txt`（BASE `17e09b1` → `88caf08`，22 文件，全文逐行读入）
- 评审依据：实施 brief（接口契约与 mock 验收裁决）、TASK 详情、实施报告（R1-R8 交叉印证）、契约 `tdengine-super-tables.md` 附录 B、`L1-execute.md` §13
- 越界反查：22 文件中 21 个在声明内；`SysJobServiceImpl` 2 行白名单为已申报偏离（协调者已追认），本次评审按 brief 要求核其改动质量与安全影响（见下，通过）
- 分级汇总：**Critical 0 / Important 0 / Minor 8**（Minor 记 TASK `任务结果` 的 deferred，收尾进 LOG，不进修订轮）

## 一、一致性核对（brief 接口契约逐条）

| 契约点 | 结论 | 依据 |
| --- | --- | --- |
| 库名派生唯一入口与安全校验 | 通过 | `TdengineAdmin.databaseName` 为唯一派生入口（静态方法，`[A-Za-z0-9_]+` 且 ≤32，非法即 IllegalArgumentException）；三个 REST 方法均在实现内部派生，无外部传库名路径；`SysTenant.tenantCode` @Pattern 入口侧双保险。注入面闭合：URL 路径段（`/rest/sql/<db>`）、DDL 拼接（CREATE DATABASE + db）、SQL 字面量（`db_name = '%s'`）三条路径的输入全部经同一校验，无注入口 |
| 建库语句与契约附录 B 逐字性 | 通过 | `CREATE DATABASE IF NOT EXISTS iot_<code> KEEP 1825d DURATION 10d BUFFER 256` 与附录 B 逐字一致；四张超级表（st_point_num/st_point_state/st_item_value/st_metric_value）列定义与 TAGS 逐字一致，四张 TAGS 均含 `tenant_id BIGINT`；`USE` 以 REST 库上下文 `/rest/sql/<db>` 替代（taosAdapter 惯例，brief 裁决 1 明示 REST 实现），语义等价。测试逐字断言五条语句 |
| fail-soft 语义 | 通过 | 连接失败/超时/HTTP≥400/非 0 code/不可解析一律映射 BusinessException（含通道、派生库名、异常摘要，摘要截断 200 字符）；建库失败 WARN 留 db_created=0；单租户失败不中断同轮；统计侧防风暴（首轮+每 N 轮 WARN、恢复清零+INFO）与 brief 裁决 4 一致；扫描侧无节流但 brief 裁决 2 未要求节流，实施报告遗留 2 已认列持续 WARN 重试 |
| 开通挂点 | 通过 | insertTenant 落库前强制 `dbCreated=0`（mass assignment 防护，请求携带 1 被忽略，有单测）；`updateTenant` 为字段白名单构建更新实体、不携带 dbCreated（仓库实读核实，更新路径同样无 mass assignment 口）；afterCommit 仅在 save 成功后注册，回滚不触发，异常收敛不炸穿已提交事务；status=null 视同启用（与 DB 默认 1 对齐，R7 修复+单测）；insertTenant 确有 `@Transactional`（仓库实读核实），同步器挂点成立 |
| 配额校验边界 | 通过 | point_quota 空/0=不限；`current >= quota` 抛 `BusinessException(4003, "点位配额不足（上限 N）")` 与 brief 裁决 3 逐字一致；计数源未装配过渡期视为 0 放行；租户缺失报参数错误。`ResultCode.QUOTA_EXCEEDED(4003)` 仅追加枚举，4001-4099 段语义全局 |
| sys_job 白名单（已申报偏离） | 通过 | 仅追加 2 个 bean 名 + 1 行注释，最小改动；框架既有安全设计未被破坏——bean 白名单之外仍有方法黑名单（remove/updateById/saveOrUpdate/truncate 等，仓库实读核实）与字符串参数限制，白名单两个新 bean 的继承自 ServiceImpl 的写库方法仍被方法黑名单拦截；`SysJobInvokeWhitelistContractTest` 反射锁定登记防误删 |
| application.yml | 通过 | 逐行核对 diff，仅新增 `ot.tdengine.*` 与 `ot.quota.*` 两键族，env 可注入形态 |
| 升级 SQL | 通过 | information_schema 判存条件化 ALTER + invoke_target 判存 INSERT...WHERE NOT EXISTS，幂等口径与既有脚本同款；default 置 1、两条任务播种（cron 60s/5min）与 brief 一致；验收 2 实测二次执行幂等 |

## 二、质量核对（错误处理 / 边界 / 安全重点）

- HttpClient：单例（构造器建一次），connect 超时在 builder、读超时在每请求，中断时恢复中断位再抛业务异常；无逐调用客户端/连接资源分配，无泄漏路径。
- 错误处理：`execute` 覆盖全部失败模式并映射 BusinessException；`getMessage()==null` 回退 `toString()`（R8）；摘要首行截断防刷屏。
- 建库幂等与并发：`CREATE DATABASE/STABLE IF NOT EXISTS` 幂等，扫描轮重叠（60s cron + runOnce）安全；置位 `updateById` 仅携带 id+dbCreated 单字段。
- 判存自愈：先 databaseExists 再 create，人工补建/部署脚本建库场景自愈置位，测试锁定次序。
- 统计比对：除法形态 `usage/quota >= ratio`，quota 字节换算 long 无溢出（int GB × GB 上限远小于 Long.MAX）；配额空/0 与阈值越界 (0,1] 外关闭的双层防护。
- 测试面：6 类 41 例覆盖 brief 裁决 5 全部断言点（建库参数/置待建标志/扫描置 1/失败留 0+WARN/配额边界/告警与静默）；HttpTdengineAdminTest 用 JDK HttpServer 记录请求形态；Mockito 再打桩用 doThrow/doReturn 正确形态（R3）。

## 三、发现清单

| # | 级别 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | `SysTenantServiceImpl.registerAfterCommitProvision` → `TenantDbProvisionServiceImpl.provisionImmediately` | afterCommit 在请求线程同步执行即时触发，TDengine 挂起时给开通接口响应附加最多约一个 REST 超时（默认 3-5s，可配）的延迟；落库与正确性不受影响且有扫描兜底，如需体验更优可改异步执行（不阻断开通事务提交后的请求返回）。 |
| 2 | Minor | `TenantDbProvisionServiceImpl.tryProvision` / `markCreated` | 本地 `updateById` 抛异常会被外层 catch 记为「TDengine 建库失败」（归因误导排障）；返回 0 时记成功返回 true（db_created 仍 0，下一轮扫描重拾自愈）——无正确性影响，但错误归因与返回值处理不够精确。 |
| 3 | Minor | `TdengineAdmin.databaseName` | 允许派生最长 36 字符库名（`iot_` + 32 位编码），TDengine 目标版本标识符长度上限未实测，且实施报告 TD-0053 联调清单未显式列此项（仅列了 DDL 参数解析），建议补入联调核对。 |
| 4 | Minor | `IotQuotaStatsServiceImplTest`（imports）/ `IotQuotaStatsServiceImplTest`、`TenantDbProvisionServiceImplTest` 查询条件断言 | 导入未使用的 `ReflectionTestUtils`；扫描/统计的查询条件断言仅 `getSqlSegment().contains(列名)` 未断言取值（mock 口径下可接受，hygiene 级）。 |
| 5 | Minor | `SysTenant.tenantCode` @Pattern | 若现场存量存在非 `[A-Za-z0-9_]` 编码的租户，其编辑请求会在校验层被拒（早于「编码不可改」业务校验）；当前库无此类数据、属已申报防御性收紧（报告偏离 5），建议在部署/升级说明中向现场提示。 |
| 6 | Minor | `IotQuotaStatsServiceImpl.warnIfOverThreshold` 注释 / 实施报告 R5 | 「乘法形态在 8GB/10GB@0.8 恰达阈值会漏报」的论据不成立——双精度下 10737418240×0.8 的舍入误差（4.77e-7）小于该量级半 ulp（9.54e-7），恰舍入回 8589934592，两种形态该场景均判等；除法形态实现本身正确，仅注释与报告依据失准。 |
| 7 | Minor | `SysTenantServiceImpl.checkPointQuota` | check-then-act 无并发防护：并发点位创建可少量越限（TOCTOU）；测试已按「超限同样拒绝」兜底口径覆盖，实际并发语义由 TASK-0021 接线方承接，接线时注意（如需强一致可在点位表加配额约束或在计数侧加锁）。 |
| 8 | Minor | `application.yml` ot.tdengine.password / `TdengineProperties` | 提交 TDengine 工厂默认口令 `taosdata` 作 dev 默认值（env 可覆盖，与库内 dev 默认密钥先例一致）；prod 档 fail-fast 必需注入集（SM4/CRYPTO 三密钥）未含 TDengine 凭据，现场启用 TDengine 时建议将其列入必需注入并禁用默认口令。 |

## 四、结论

diff 与 brief 全部接口契约、TASK 验收标准（mock 口径）、契约附录 B 逐字性一致；fail-soft 边界、开通挂点事务语义、注入面防护、白名单最小性均成立，未发现 Critical / Important 缺陷。8 条 Minor 均记 deferred，不阻塞收尾；其中 #3、#5、#8 建议随 TD-0053 联调或部署说明一并消化。
