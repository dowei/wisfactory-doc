# 实施报告：TASK-0050（状态展示读口——数据服务统一状态查询）

- 执行者：TASK-0050 执行者（backend 道；no-subagents；brief 驱动）
- BASE：`331f860`（开工时主仓 main HEAD）
- 本任务提交：`e49fea8`（main，注明 TASK-0050；15 文件全落 brief 声明范围，calc/frontend/ingest 零触碰）
- 执行日期：2026-11-24
- 环境：MySQL/Redis 宿主 phpStudy（8.0.12@3307 / 3.0.504@6379）实测正常；TDengine 192.168.31.145:6041 REST 故障（TCP 通但 15s 超时/reset，服务器侧，TD-0053 登记）——历史时间线做代码+契约测试（假 taosAdapter 捕获 SELECT SQL 精确断言），真机读回列为环境恢复后交付条件，**不阻塞代码收尾、不宣称已验真**；8098 三次起停均起前探测 FREE / 用毕即停复核 FREE；sql-exec.sh 每条语句单独一次调用；wsl docker 容器未碰。
- 结论先行：**验收 4 条全部通过**——① 全量 `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 2423/0/0/0 BUILD SUCCESS（开工基线 2403 + 净增 20，既有 0051/0048 逐类零回归）；② dev 冒烟 10/10 绿（Redis 状态直读 + 字典动态翻译 + 翻译随字典变化 + 对外 scope 6003 + UNKNOWN 空态 + 历史读口空集 + 参数 400 + SQL 契约单测）；③ prod 档真实启动 18.7s + ERROR=0 + 签名状态真调 200/UNKNOWN；④ `git diff --name-only 331f860..HEAD` 恰为本任务 15 文件，全落声明范围。

---

## 1. 改动文件清单（15 文件，全部在 brief 声明范围）

### 1.1 新增：状态读口模块 `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusread/`（7）

| 文件 | 职责 |
| --- | --- |
| `StatusCurrentVO.java` | 当前状态载荷（instanceId/instanceCode/statusCode/statusName/color/seq/quality/changeTime）——管理面与对外 API 共用单一形状；无状态 UNKNOWN（seq=null/quality=3） |
| `StatusHistoryPointVO.java` | 历史时间线点（ts/seq/statusCode/name/color/quality；写侧 -1 哨兵对外归一为 null） |
| `StatusReadService.java` | 读口服务接口：管理面（会话租户+数据权限）/对外面（显式 tenantId，scope 已由 openapi 侧裁剪）两组重载 |
| `controller/IotStatusReadController.java` | 管理面 `GET /api/v1/iot/status/current?instanceId=` + `/history?instanceId=&from=&to=`（权限在服务层裁剪；本层只做透传与参数校验） |
| `impl/StatusReadServiceImpl.java` | 核心实现：rawStringRedisTemplate 直读 `iot:st:{tenantCode}:{instanceCode}`（0049 写入形态，不现算 ADR-0004）；名称/颜色只从 iot_status_dict + iot_status_rule 动态翻译（丢弃 calc 硬编码 statusName/color——0049 F-06 反例）；历史经 timeseriesClient 读 st_item_value 按序号翻译；Redis 故障/脏值 fail-soft UNKNOWN；管理面会话租户 + dataperm 可见集裁剪（越权 403） |
| `mapper/StatusTranslationMapper.java` | 翻译只读 Mapper：dict by code / rule by code / dict by seq，三条 SQL 均 `@InterceptorIgnore(tenantLine)` + 显式 tenant_id（机器路径无会话）；每次查询实时读表（规则/字典发布变化即时跟随） |
| `mapper/StatusTranslationRow.java` | 翻译行投影（编码/名称/颜色/启用态） |

### 1.2 修改：openapi 面接线（4，均 `module/openapi/**`）

| 文件 | 说明 |
| --- | --- |
| `OpenApiQueryService.java` | 接口新增 `statusCurrent`/`statusHistory` 两方法（TASK-0050 对外状态查询契约） |
| `service/impl/OpenApiQueryServiceImpl.java` | 实现两方法：instanceId 空→400；`requireInstanceVisible`（复用 0051 既有 scope 裁剪）scope 外→6003；scope 内调状态读口；statusReadService 以 `@Autowired` setter 注入（既有 8 参构造与 0051 测试零改动）；读口缺位 fail-closed 500 不静默造假 UNKNOWN |
| `controller/OpenApiController.java` | 新增 `GET /openapi/v1/status/current` + `/status/history`（类级 @Anonymous + requireIdentity 防线不变，鉴权/审计/限流全归 0051 过滤器链） |
| `support/OpenApiTimeseriesClient.java` | 新增 `queryStatusHistory`：SELECT ts,value,quality FROM `iot_{tenantCode}.st_item_value` WHERE tenant_id=? AND `full_key='{instance}.STATUS'`（0049 F-02 规范形态、无 full: 前缀）+ ts 范围下推 + ORDER BY ts ASC；fail-soft 空页 |

### 1.3 新增：测试（4，均 `backend/ot-admin/src/test/java/**`）

| 文件 | 用例数 | 覆盖 |
| --- | --- | --- |
| `iot/statusread/impl/StatusReadServiceImplTest.java` | 9 | Redis 直读 + 字典动态翻译（断言不采纳 calc 硬编码的 RUNNING/#00FF00）/ 翻译随字典变化 / 规则表兜底 / 无键 UNKNOWN / 字典登记 UNKNOWN / Redis 故障 fail-soft / calc 写侧 UNKNOWN JSON 解析 / 历史序号翻译 / UNKNOWN 哨兵 -1 |
| `iot/statusread/support/StatusHistorySqlContractTest.java` | 1 | 假 taosAdapter（JDK HttpServer）捕获真实 SELECT：库名 iot_default + `full_key='F01.WS01.EQ001.STATUS'` 精确形态无 full: 前缀 + tenant_id + 时间范围 + ORDER BY ts ASC |
| `iot/statusread/controller/IotStatusReadControllerTest.java` | 5 | 管理面 current/history 参数透传 + UNKNOWN 载荷 + 参数 400 + 越权 403 直达 |
| `openapi/service/impl/OpenApiStatusQueryTest.java` | 5 | 对外 scope 内载荷 / scope 外 6003 / 参数 400 / 历史 scope 6003+透传 / 历史参数 400 |

计数：9+1+5+5 = **20** 新增用例。

---

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

前置：

```
$ mvn install -pl ot-core -DskipTests
CORE_EXIT=0
[INFO] BUILD SUCCESS
[INFO] Total time:  1.776 s
[INFO] Finished at: 2026-11-24T03:35:38+08:00
```

全量 mvn Results 段（终码提交 `e49fea8` 对应源码）：

```
[INFO] Running net.otlab.admin.module.openapi.service.impl.OpenApiQueryServiceImplTest
[INFO] Tests run: 31, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.611 s -- in net.otlab.admin.module.openapi.service.impl.OpenApiQueryServiceImplTest
[INFO] Running net.otlab.admin.module.iot.statusread.impl.StatusReadServiceImplTest
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0
[INFO] Running net.otlab.admin.module.iot.statusread.support.StatusHistorySqlContractTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] Running net.otlab.admin.module.iot.statusread.controller.IotStatusReadControllerTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] Running net.otlab.admin.module.openapi.service.impl.OpenApiStatusQueryTest
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] Running net.otlab.admin.module.iot.statusrule.service.impl.IotStatusDictServiceImplTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 2423, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:20 min
[INFO] Finished at: 2026-11-24T03:37:01+08:00
[INFO] ------------------------------------------------------------------------
EXIT=0
```

- 2423 = 开工基线 2403（前序并行提交 0054/0058/0081 等累计）+ 本批净增 20（StatusReadServiceImplTest 9 + HistorySqlContract 1 + Controller 5 + OpenApiStatusQuery 5）。
- 既有 0051（OpenApiQueryServiceImplTest 31 / 0054 dict/sync 用例）、0048（IotStatusDictServiceImplTest 13）逐类零回归；skipped 0（红线二已由 0051 转正）。
- TDD 红转绿留痕（每修复附「为什么现在对了」）：
  1. **红**（编译期）：全部 4 个测试类首次运行 → `找不到符号 类 StatusReadService / StatusCurrentVO / IotStatusReadController / 程序包 net.otlab.admin.module.iot.statusread.mapper 不存在` → `BUILD FAILURE`（EXIT=1）。**为什么这是有效红**：所有生产类尚不存在，测试先定义契约（读口接口/载荷/SQL 契约/错误段），编译即红是最硬的红。
  2. **实现后第二轮红**（构造不匹配）：`无法将接口 StatusReadService 的方法 current 应用到给定类型: 需要 Long, 找到 Long,String,Long,String` + `找不到符号 方法 anyLong()/anyString()`。**为什么现在对了（两处修复）**：① 核心读方法（带显式 tenantCode/instanceCode）是 impl 的内聚实现细节、不在对外接口，测试字段类型由 `StatusReadService` 改为 `StatusReadServiceImpl`——单一职责：接口只暴露面向调用方的 4 方法，impl 内部方法供直测与两路径复用；② 补 `anyLong()/anyString()` 静态 import，Mockito matcher 与实参类型对齐。
  3. **绿**：聚焦 4 类 surefire 原始输出 `StatusReadServiceImplTest tests=9 errors=0 failures=0` / `StatusHistorySqlContractTest tests=1` / `IotStatusReadControllerTest tests=5` / `OpenApiStatusQueryTest tests=5`，EXIT=0；全量 2423/0/0/0。**为什么现在对了**：读侧不现算（只读 Redis/表）、名称/颜色每次查询从规则+字典实时翻译（无缓存/无硬编码）、历史 SQL 精确契约在假适配器上锁定 full_key/时间范围/序列、对外 scope/参数错误复用 0051 既有 6003/400 口径——四条验收面分别有机械断言。

### 2.2 验收 2：dev 冒烟（8098，dev 档）

起前探测 `PORT_8098_FREE`；`dev.ps1 build` BUILD SUCCESS（44s）；`dev.ps1 start` → liveness HTTP 200（`Started ...` 日志见 start log）。冒烟脚本 `backend/target/t0050_smoke.py`（raw RESP2 写 Redis 状态、管理面 Bearer token、对外 HMAC 签名、SQL 夹具经 Git Bash sql-exec.sh 逐条执行），终轮完整原始输出（EXIT=0）：

```
OK   | P0                                                   HTTP 200
OK   | P1                                                   token=434d444f...
    fixture: rate_limit 8->100, Redis SET iot:st:default:F01.WS01.EQ001 = {"statusCode":"RUNNING","statusName":"RUNNING","color":"#00FF00","seq":1,"quality":0,"ts":1795462000000}
OK   | S1                                                   管理面当前状态：Redis 直读 + 字典动态翻译（不采纳 calc 硬编码） | HTTP 200 code=200 data={"instanceId": 6, "instanceCode": "F01.WS01.EQ001", "statusCode": "RUNNING", "statusName": "运行", "color": "#00FF00", "seq": 1, "quality": 0, "changeTime": 1795462000000}
OK   | S2                                                   翻译随字典发布变化（改名改色立即跟随，无硬编码） | name=运转中 color=#1E90FF
OK   | S3                                                   对外当前状态（scope 内）：状态载荷一致 | HTTP 200 code=200 data={"instanceId": 6, "instanceCode": "F01.WS01.EQ001", "statusCode": "RUNNING", "statusName": "运行", "color": "#00FF00", "seq": 1, "quality": 0, "changeTime": 1795462000000}
OK   | S4                                                   对外 scope 外实例 → 6003 | HTTP 200 code=6003 msg=instanceId 不属于调用方授权范围（scope 外 6003）
OK   | S5                                                   管理面 UNKNOWN 空态：无 Redis 状态 → seq=null/quality=3 不报错 | HTTP 200 data={"instanceId": 3, "instanceCode": "F01.WS01", "statusCode": "UNKNOWN", "statusName": "UNKNOWN", "color": null, "seq": null, "quality": 3, "changeTime": null}
OK   | S6                                                   对外 UNKNOWN 空态一致 | code=200 data={"instanceId": 3, "instanceCode": "F01.WS01", "statusCode": "UNKNOWN", "statusName": "UNKNOWN", "color": null, "seq": null, "quality": 3, "changeTime": null}
OK   | S7                                                   历史时间线读口可用（TDengine 故障期空集如实，不炸） | HTTP 200 code=200 data=[]
OK   | S8                                                   对外历史参数错（from>=to）→ 400 | code=400 msg=start/end 必填且须 epoch 毫秒、start < end
    cleanup done (Redis key DEL, dict restored, rate_limit restored)

T0050 SMOKE DONE  (10/10 checks)
EXIT=0
```

逐条对应：S1 真 Redis 状态行（故意写入 calc 硬编码 statusName=RUNNING/color=#00FF00）→ 读侧返回字典翻译 `运行/#00FF00`（证明不采纳硬编码）；S2 UPDATE 字典为 运转中/#1E90FF 后即刻跟随（对应用例 TC-0016 步骤 6 / REQ-0007 验收 4）；S3 对外 HMAC 签名调用 scope 内实例同载荷（三处一致性证明）；S4 scope 外实例 24（E6B，不在 F01 子树）→ 6003；S5/S6 无 Redis 状态键的实例 3（F01.WS01，scope 内）→ UNKNOWN/seq=null/quality=3 两面一致不报错；S7 历史读口真调 200 空集（TDengine REST 故障 fail-soft，契约已由假适配器单测锁定）；S8 参数错 400。首轮 5/10 失败为脚本侧问题（Bearer token 前缀缺失 + UNKNOWN 实例选在 scope 外），修正后 10/10（主码零改动）。

**历史 SQL 契约单测原始输出**（假 taosAdapter 捕获 SELECT，摘要自 surefire 文本）：

```
StatusHistorySqlContractTest.historySelectSqlContract:
  SQL=SELECT ts, value, quality FROM iot_default.st_item_value WHERE tenant_id = 1 AND full_key = 'F01.WS01.EQ001.STATUS' AND ts >= 1795211995000 AND ts <= 1795212995000 ORDER BY ts ASC LIMIT 20001
  → full_key 规范形态无 full: 前缀（0049 F-02）；tenant/时间范围/序列升序全部锁定；tests=1 fail=0
```

清理复核：`dev.ps1 stop` → PID=40560 已停 → `netstat :8098 LISTENING` 无输出；字典 `RUNNING 运行 #00FF00` 已还原；Redis `DEL iot:st:default:F01.WS01.EQ001` → `:0`（已删）。

### 2.3 验收 3：prod 档真实启动回归

启动前探测 `PROD_PROBE_FREE`；打包 `mvn package -DskipTests -pl ot-admin -am` EXIT=0。启动命令（沿 0051/0054 确认口径 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

干净单轮（无负向探针，日志 /tmp/t0050_prod3.log）：

```
PROD_READY after 18s
2026-11-24T03:43:35.470+08:00  INFO 86084 --- [ot-framework] [           main] net.otlab.admin.OtAdminApplication       : Started OtAdminApplication in 18.672 seconds (process running for 19.18)
DICT-SYNC STYLE PROD PROBE: HTTP 200 code=200 statusCode=UNKNOWN quality=3
=== ERROR count (startup + clean probe) ===
0
0
PROD_PID=86084
成功: 已终止 PID 为 86084 的进程。
PORT_8098_FREE
```

补充探针对照（另一轮 /tmp/t0050_prod2.log）：对外 current 在 scope 内 200/UNKNOWN、history 200/[]、scope 外 200/6003——唯一 1 条 ERROR 为 `GlobalExceptionHandler 业务异常：instanceId 不属于调用方授权范围（scope 外 6003）`，即负向探针的预期拒绝留痕（与 0048/0051 报告同形态），非启动错误；干净轮 ERROR=0 已单轮实证。prod 档 `/auth/login/test` 为 dev-only（`/auth/login/test` 返回 500 testLoginDevOnly），故 prod 用签名对外调用做探针（与 0054 同口径）。

### 2.4 验收 4：改动范围核对

```
$ git rev-parse --short HEAD   # 开工 BASE
331f860

$ git diff --name-only 331f860..HEAD
（15 文件，全部 backend/ot-admin/**：statusread 7 + openapi 4 + test 4；详见 §1）

$ git diff-tree --no-commit-id --name-only -r e49fea8 | grep -v 'backend/ot-admin/src/main/java/net/otlab/admin/module/\(iot/statusread\|openapi\)/' | grep -v 'backend/ot-admin/src/test/java/'
0 — ALL IN SCOPE

$ git diff-tree --no-commit-id --name-only -r e49fea8 | grep -E '^(calc|frontend|ingest)/'
0 — ZERO TOUCH
```

- 本任务提交 `e49fea8` 15 文件全落 brief 声明范围：管理面状态模块 `iot/statusread/**`（新）+ `openapi/**`（对外状态接线）+ `test/**`；不碰 calc/frontend/ingest、契约、status 规则表配置域（0048）、workitems 库（报告除外）。
- 工作区另有并发任务的未提交改动（`calc/**` 与 `.tmp-t0042/`、`.agents/` 等）——非本任务产生，本任务仅显式 stage 上述 15 文件，未 add/未提交/未触碰。TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。

---

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 历史时间线真机读回（TDengine 真数据 SELECT 验证） | TDengine 192.168.31.145:6041 REST 故障（TCP 通但 15s 超时/reset，服务器侧，TD-0053 登记）；按 brief 口径以代码 + 假适配器契约测试（假 taosAdapter 捕获 SELECT SQL 精确断言 full_key/时间范围/序列）闭合，真机读回列为环境恢复后交付条件，**当前标记未验真，不宣称已验真** |
| 详情页（0059）/色块（0058 批二）前端消费接入 | brief 明文不碰前端；三处一致性由本读口单一来源保证（前端只消费不改口径），前端接线归后续任务 |
| 状态规则随配置发布链真下发 | 0048 已登记「iot_status_rule 不在冻结快照五对象集内、发布承载对象上游未定」；翻译实时查表，规则/字典发布后自动跟随，发布链承接归 0024/演进位 |
| Redis 多实例部署缓存一致性 | 0049 同族演进位（未变不重写为进程内缓存）；本任务只读不写，不涉及 |
| 状态告警（三期）/状态时长还原 | 超 FEAT-0016 一期范围；st_item_value 序列已可读，时长还原属展示侧后续任务 |
| TASK 置「已完成」 | 硬约束未置，保持「进行中」待评审；未改 workitems 库（本报告除外） |

**TDengine 真机读回补验条件（环境恢复后执行，协调者 TD-0053 安排）**：环境恢复后，经本读口 `GET /api/v1/iot/status/history?instanceId=<有留痕实例>&from=&to=` 真调一次，断言返回时间线点 seq/statusCode/name/color 与 0049 写入的 value=稳定序号/quality=0 对应（UNKNOWN 行 value=-1/quality=3 → 对外 seq=null/UNKNOWN）；单测已锁定 SELECT SQL 形态（iot_<tenantCode>.st_item_value + full_key='<instance>.STATUS' + ts 范围 + ORDER BY ts ASC），真机补跑为形态复核。

---

## 4. 偏离计划之处

1. **翻译主查字典、规则表兜底**（brief 决策 2「以规则表 iot_status_rule + 字典为准」未定优先级）：实现为 dict by code 优先 → rule by code 兜底 → 以编码自身作名称（颜色 null）；理由：0048 裁决 TD-0013 字典为稳定序号单一来源（编码/名称/颜色权威），规则表 statusCode 必须字典锚定，故字典为翻译权威、规则表承接自定编码场景；二者都是动态查表（无硬编码），翻译随发布变化口径不变。
2. **状态读口管理面不放 @SaCheckPermission**（brief 只说「走数据权限拦截（管理面 dept 裁剪）」未给权限码）：本任务不改 resources/sql（不在范围），既有实例类菜单权限码 iot:instance:query（4031）可复用但 brief 未指定；数据权限裁剪落在 service 层（dataperm.resolve + 可见集，越权 403），与 0012/0014 单实例读范式一致；未自行新增权限码种子。如评审要求显式按钮权限码，可后续在管理面加 `@SaCheckPermission("iot:instance:query")` 并归口菜单（本任务不动 SQL）。
3. **statusReadService 以 setter 注入 openapi 服务**（而非扩构造）：0051/0054 既有 8 参构造被自身测试直调，扩构造会迫改多个既有测试；setter 注入保持 0051 测试零改动且 Spring 装配正常（dev/prod 启动实证）。
4. **Redis 读取用 rawStringRedisTemplate**（无 key-prefix）：calc 0049 以 RESP2 直写原始键 `iot:st:{tenant}:{instance}`（无应用前缀、db 与 ot 同源 db 0），项目业务模板会二次拼 key-prefix（dev 为 ot_framework:）导致读空——沿用 RedisVisibleScopeCache 的 raw 模板范式（BUG-1110 同款），键空间与写侧严格对齐。
5. **历史 SQL 由 OpenApiTimeseriesClient 新增 queryStatusHistory 实现**（而非新建状态专用客户端）：0049 写入侧 full_key=instance.STATUS 与 0051 时序客户端 st_item_value 查询同表同构，复用既有 fail-soft/白名单/截断/库名派生基础设施，仅增一个方法（openapi/** 范围内），零新类零新配置。
6. **dev 冒烟脚本两处修正**（脚本侧，主码零改动）：首轮 5/10 —— ① 管理面 Authorization 缺 Bearer 前缀（Sa-Token token-prefix=Bearer 配置）→ 401；② UNKNOWN 实例选 id=20（EQ1A）在 test_client scope 外 → 6003 而非 UNKNOWN；改用 scope 内无状态键实例 3（F01.WS01）后 10/10。**为什么现在对了**：探针与真实契约（token 前缀 / scope 可见集）对齐，而非改代码迎合脚本。

---

## 5. 遗留问题

1. **TDengine 真机读回未验**（环境阻断）：环境恢复后按 §3 条件补验，当前标记未验真；契约测试已锁定 SQL 形态，补跑为复核非重做。
2. **prod 负向探针 ERROR 留痕**：scope 外 6003 由 GlobalExceptionHandler 按 0051 既有约定记 ERROR 级业务异常（干净启动轮 0 ERROR 已实证）；如运维要求降级为 WARN 属 0051 全局异常handler口径，非本任务范围。
3. **dev 库夹具终态**：test_client rate_limit 已还原 8、字典 RUNNING 已还原、Redis 状态键已 DEL；冒烟脚本 `backend/target/t0050_smoke.py` 在 gitignored target/ 内（与 0048/0051/0054 冒烟脚本同处理，未入库）。
4. **工作区并发未提交改动**：`calc/**` 与 `.tmp-t0042/` 属其他任务现场，本任务未触碰/未提交；提交 e49fea8 仅含本任务 15 文件。
5. **TASK 状态**：保持「进行中」待评审，未置「已完成」；未回写 workitems 库（本报告除外）。

---

## 一句话结论

**通过**——验收 1（全量 2423/0/0/0 BUILD SUCCESS，既有 0051/0048 逐类零回归 + 新增 20）、验收 2（dev 冒烟 10/10：Redis 直读+字典动态翻译+随字典变化+对外 scope 6003+UNKNOWN 两面一致+历史空集+参数 400，SQL 契约由假 taosAdapter 精确断言）、验收 3（prod 启动 18.7s + 签名状态真调 200/UNKNOWN + ERROR=0）、验收 4（提交 e49fea8 15 文件全落声明范围、calc/frontend/ingest 零触碰）四条全部以原始输出为证通过；TDengine 真机读回因环境故障未做（TD-0053 登记，补验条件已明确，当前未验真），不影响四条验收判定，TASK 保持「进行中」待评审。
