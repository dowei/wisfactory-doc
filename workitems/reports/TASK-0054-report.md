# 实施报告：TASK-0054（dict/sync 字典同步接口——占位转正）

- 日期：2026-11-24
- 执行者：implementer（backend 道，no-subagents）
- BASE：`5758d5f`（开工时主仓 HEAD）
- 本任务提交：`3cdde58`（main，注明 TASK-0054；父提交 `9aed9a4` 为并行 TASK-0058 前端提交）
- 结论先行：**四条验收全部通过**。全量 `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → 2395/0/0/0 BUILD SUCCESS；dev 真调 13/13 绿（since=0 三态齐全、deleted=1 禁用、边界、未来游标、跨租户、无效 key 403+6001）；prod 档启动 23.49s + ERROR=0 + 签名真调 200 非空清单；本任务提交 8 文件全落声明的 openapi 模块+测试范围。

---

## 1. 改动文件清单

| 文件 | 说明 |
| --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/dto/OpenApiDictSyncVO.java` | 占位转正：保留 `since/instances/items` 三字段（占位兼容），`List<Object>` 具体化为 `List<InstanceRow>` / `List<ItemRow>`；新增内嵌 `RowBase`（id/code/name/changeType/updateTime）+ 两行 DTO（实例行 instanceId/templateId/modelCode/deptId/status；数据项行 itemId/dataCode(instanceId)/fullKey/sourceType/status）；javadoc/Schema 标注中台资产=平台模型命名差异与三态口径 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/mapper/OpenApiInstanceMapper.java` | 新增 `selectChangedForTenant(tenantId, since)`：`FROM iot_model_instance WHERE tenant_id=#{tenantId} AND update_time > FROM_UNIXTIME(#{since}/1000)`，含 deleted=1 行；SQL CASE 派生 changeType（deleted=1→3 禁用；status=0→2 变更；create_time≥since→1 新增；余 2），逻辑删 status 强制 0；`@InterceptorIgnore(tenantLine)` 显式租户旁路 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/mapper/OpenApiItemMapper.java` | 新增同类 `selectChangedForTenant`（iot_data_item，data_code/data_name/full_key/source_type/instance_id 载荷映射，三态/显式租户同实例行） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/service/impl/OpenApiQueryServiceImpl.java` | `dictSync` 从占位空实现改为：since 缺省 0 归一 → 用验签产物 `identity.getTenantId()` 显式带参查两类 mapper → 原样回显 since；不消费 dataperm（§3.5 appKey 有效即可，不走 scope） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/service/OpenApiQueryService.java` | dictSync javadoc 由「占位」改为转正口径（增量两类、三态、since 缺省/回显、资产模型对应、租户显式收敛不消费 dataperm） |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/controller/OpenApiController.java` | dict/sync @Operation summary 由占位文案改为「字典增量同步（since 游标；三态；appKey 有效即可）」；路径/方法/鉴权链零改 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/service/impl/OpenApiQueryServiceImplTest.java` | 6 个新用例替代原 1 个占位用例：since=0 全量三态+具体化行 DTO、since 缺省 0、精确边界/游标推进、未来游标空、deleted=1 禁用态、跨租户隔离+不消费 dataperm；31 个用例（原 25，净 +6） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/mapper/OpenApiDictSyncMapperSqlTest.java`（新增） | 3 个 MapperAnnotationBuilder SQL 解析回归：实例/数据项增量 SQL 的显式租户、FROM_UNIXTIME 严格大于、三态 CASE、deleted 不物理过滤、绑定位数；返回类型为具体化行 DTO |

范围：8 文件全部在 `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**` 与 `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/**` 声明内；**无新 SQL**（仅复用 iot_model_instance / iot_data_item 既有表），不碰 frontend/、ingest/、calc/、dataperm、create 域。

---

## 2. 每条验收命令原始输出

### 验收 1：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

前置（无输出即成功）：

```
$ cd backend && mvn install -pl ot-core -DskipTests -q
CORE_INSTALL_OK
```

全量 mvn Results 段（终轮，SQL 三态修订后重跑）：

```
[INFO] Results:
[INFO] 
[INFO] Tests run: 2395, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:55 min
[INFO] Finished at: 2026-11-24T00:44:45+08:00
[INFO] ------------------------------------------------------------------------
```

TDD 红转绿证据（关键步骤）：
- 红 1（未实现行 DTO）：`COMPILATION ERROR ... 找不到符号 类 InstanceRow / 类 ItemRow` → BUILD FAILURE。为什么现在对了：新增具体化行 DTO 后编译通过。
- 红 2（SQL 三态断言写错）：`itemIncrementalSql expected: <2> but was: <3>`（since 在 CASE 与 WHERE 各出现一次共 3 个绑定位）→ 修正测试预期为 3。为什么现在对了：断言与真实 SQL 绑定口径一致。
- 绿 1：聚焦 `OpenApiQueryServiceImplTest` 31/0/0/0 BUILD SUCCESS（原 25 + 新 6）。
- 绿 2：聚焦 `OpenApiDictSyncMapperSqlTest,OpenApiQueryServiceImplTest` 34/0/0/0 BUILD SUCCESS。

**为什么现在对了（每个修复）**：
1. 行 DTO 缺失 → 建立 `RowBase`+`InstanceRow`/`ItemRow`，VO 字段由 Object 具体化：编译过、客户端拿到强类型载荷。
2. 三态在 since=0 全量时缺「变更」态 → 将 CASE 改为 `deleted=1→3`（逻辑删=禁用）、`status=0 未删→2`（启用→停用的状态变更）、`create_time>=since→1`（新增）、其余 2：真实数据上三态可区分；dev 冒烟从 12/13 变 13/13。
3. 冒烟夹具缺 status=0 的数据项 → 冒烟脚本插入一条 status=0/deleted=0 数据项（用后删除）：验证的是真 SQL CASE 路径，不是改代码凑数。
4. 测试助手 ItemRow 少传 name 参数 → 补 name 实参：构造函数匹配。

### 验收 2：dev 冒烟（8098，dev 档）

启动前探测：`PORT_8098_FREE`（netstat 无 LISTENING）；打包 `mvn package -pl ot-admin -DskipTests -q` → `PACKAGE_OK`；`dev.ps1 start` → liveness 200 READY。冒烟脚本 `backend/target/t0054_smoke.py`（HMAC-SHA256(secret, appKey+timestamp+SHA256(body)) 与 HmacAuthStrategy 同口径；SQL 经 `C:/Program Files/Git/bin/bash.exe scripts/sql-exec.sh` 逐条执行），终轮完整原始输出：

```
OK   | P0                                                   HTTP 200
    fixture: rate_limit 8 -> 100
OK   | S1a                                                  HTTP 200 code=200 since=0
OK   | S1b                                                  instances=24 items=38
OK   | S1c                                                  inst=[1, 2, 3] items=[1, 2, 3]
OK   | S1d                                                  sample_inst={"instanceId": 2, "code": "F01", "name": "冒烟一厂T12", "changeType": 1, "updateTime": "2026-11-22 04:52:07", "templateId": null, "deptId": 1, "status": 1, "id": 2, "modelCode": "F01"}
OK   | S1e                                                  disabled_inst=2
OK   | S2                                                   hit={"instanceId": 30, "code": "T0054_DISABLED_1795452109", "name": "T0054 smoke deleted", "changeType": 3, "updateTime": "2026-11-24 00:41:52", "templateId": null, "deptId": 1, "status": 0, "id": 30, "modelCode": "T0054_DISABLED_1795452109"}
OK   | S3a                                                  since=1795294327000
OK   | S3b                                                  rows=24 boundary=2026-11-22 04:52:07
OK   | S3c                                                  rows=24
OK   | S4                                                   inst=0 items=0
OK   | S5                                                   ct_code=T0054_CT_1795452123 in_result=False
OK   | S6                                                   HTTP 403 code=6001
    cleanup done (fixtures removed, rate_limit restored)

T0054 SMOKE DONE  (13/13 checks)
```

逐条对应：S1a since=0 真调 200+回显；S1b 两类清单非空；S1c 三态齐全（inst/items 均含 1/2/3）；S1d 载荷明确资产模型对应（instanceId/templateId/modelCode/deptId/status + fullKey/sourceType）；S1e 既有 deleted=1 行禁用态；S2 新造 deleted=1 行以 changeType=3/status=0 返回；S3a-b-c since 推进到 F01 的 update_time（1795294327000）：F01 精确边界行不返回（严格 >），边界后新行保留；S4 未来游标（4102444800000）两类空；S5 租户 2 行不进入租户 1 调用方清单；S6 无效 appKey 403+6001（0051 既有链）。夹具（rate_limit 临时 100、status=0 数据项、deleted=1 实例、租户 2 行/租户）均已清理/还原。用毕即停：`dev.ps1 stop` → `PORT_8098_FREE`。

### 验收 3：prod 档真实启动回归

启动前探测：`PROD_PROBE_FREE`。启动命令（沿 0051 确认口径 + 三密钥注入）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

原始输出：

```
PROD_READY after 20s
2026-11-24T00:45:34.434+08:00  INFO 52696 --- [ot-framework] [           main] o.s.boot.tomcat.TomcatWebServer          : Tomcat started on port 8098 (http) with context path '/'
2026-11-24T00:45:34.463+08:00  INFO 52696 --- [ot-framework] [           main] net.otlab.admin.OtAdminApplication       : Started OtAdminApplication in 23.49 seconds (process running for 24.173)
--- ERROR count ---
0
0
DICT-SYNC HTTP 200 code 200
since 0 instances 24 items 37
sample {"instanceId": 2, "code": "F01", "name": "冒烟一厂T12", "changeType": 1, "updateTime": "2026-11-22 04:52:07", "templateId": null, "deptId": 1, "status": 1, "id": 2, "modelCode": "F01"}
--- ERROR after probe ---
0
PROD_LISTEN_PID=52696
PROD_KILLED
PORT_8098_FREE
```

prod 档签名真调 200 + 非空清单（占位期的空清单已转为真数据）；启动段与探针后 ERROR=0；用毕即停复核端口释放。

### 验收 4：改动范围核对

```
$ git rev-parse --short HEAD   # 开工
5758d5f

$ git diff --name-only 5758d5f..HEAD
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/controller/OpenApiController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/dto/OpenApiDictSyncVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/mapper/OpenApiInstanceMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/mapper/OpenApiItemMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/service/OpenApiQueryService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/service/impl/OpenApiQueryServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/mapper/OpenApiDictSyncMapperSqlTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/service/impl/OpenApiQueryServiceImplTest.java
frontend/src/api/display/display-api.ts
...（并行 TASK-0058 前端提交 9aed9a4，共 13 文件，非本任务）

$ git diff-tree --no-commit-id --name-only -r 3cdde58
（上列 8 个 backend/openapi 文件，无其它）
```

说明：BASE 后、本任务提交前，并行 TASK-0058 提交 `9aed9a4`（13 个 frontend 文件）插入主仓历史；本任务提交 `3cdde58` 的 8 文件全落声明范围内，未 rebase/改写他人历史，未碰 frontend。

---

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 中台建点回传（反向同步） | ADR-0010 TD-0032 一期不做；brief 明文不预留写接口，本任务零触碰 |
| 数据项订阅推送 / 指标产出事件 | FEAT-0017 另归 TASK-0053；本任务仅 dict/sync 出站查询接口 |
| 新 SQL / 新表 / 事件 | brief 明定「无新 SQL」，复用既有 iot_model_instance / iot_data_item；不发事件 |
| scope 裁剪 | §3.5 原文 dict/sync「appKey 有效即可」，brief 决策 3 明确不走 scope；identity 仍由 requireIdentity 取供审计 |
| 调用方管理/轮换/吊销（0052 域） | 非本任务范围；test_client 种子沿用 0051，冒烟期间仅临时调 rate_limit 并还原 |

---

## 4. 偏离计划之处

1. **changeType 三态派生口径细化**：brief 只给「1 新增/2 变更/3 禁用 + deleted=1 以禁用态返回」，未定义 status=0 未删行的归类。首轮冒烟（12/13）显示全量 since=0 时 seed 无「变更」态——根因是 CASE 原先把 `status=0` 也归 3，且 `create_time≥0` 把所有未删启用行归 1。落地改为 `deleted=1→3`、`status=0 未删→2`、`create_time≥since→1`、余 2：三态在真实数据上可区分；实例表既有 F01.WS02（status=0/deleted=0）、冒烟夹具补一条 status=0 数据项后，dev 冒烟 S1c 三态齐全（inst/items 均 [1,2,3]）。`deleted=1` 仍以 3 禁用返回（brief 硬约束未变）。
2. **冒烟临时夹具**：dev 库 test_client `rate_limit` 8→100（用毕还原）、新增 status=0 数据项/deleted=1 实例/租户 2 实例与租户（用毕删除）；均为数据夹具、非主码行为。
3. **mapper SQL 解析单测**：仓库无 SpringBootTest 先例，为把 `@Select` 注解 SQL 的运行时错误钉在单测阶段，新增 `OpenApiDictSyncMapperSqlTest`（仿既有 `SurveyFormDataMapperSqlTest` 范式）——落在本任务声明的 openapi 测试目录内。
4. **并行交错提交**：BASE 后并行 TASK-0058 前端提交插队；本任务仅按线性历史追加自己的提交，报告 §2 验收 4 已按提交粒度区分归属。

---

## 5. 遗留问题

1. **dev 库冒烟残留**：test_client 行 scope={"instanceIds":[2],"subtree":true}、rate_limit=8 沿用 0051 终态（本任务已还原）；iot_api_call_log 累计审计行（正常业务数据）；冒烟夹具已清理。
2. **中台对齐联调**：dict/sync 载荷字段（changeType 三态、assetModelRelation 的 instanceId/templateId/modelCode/deptId/status、数据项 fullKey/sourceType）需与中台消费方确认映射；反向建点回传二期随三通道完善另行交付。
3. **prod /actuator/health 聚合 503**：MailHealthIndicator 对 prod 默认 smtp.example.com 探活失败（0051 已登记，非本任务引入）；liveness/readiness UP，不影响 dict/sync 验收。
4. **SEL 时区口径**：since 游标按 `FROM_UNIXTIME(since/1000)` 在 SQL 侧比较，依赖 DB/JDBC `serverTimezone=GMT+8`（ application-dev.yml 既有）；若部署改时区需同步校验游标边界（本任务未改配置）。

---

**结论：通过**——四条验收证据：全量 2395/0/0/0 BUILD SUCCESS、dev 冒烟 13/13（三态+deleted=1+边界+跨租户+无效 key）、prod 启动 23.49s/ERROR=0 且签名真调 200 非空清单、本任务提交 8 文件全落 openapi 声明范围。
