# TASK-0083 验收报告：发布时未锁定实例自动跟随推进（TD-0059 处置——复用 0081 applyToVersion）

> 执行者：TASK-0083 执行；验收日期：2026-11-24；工作区 `D:\workspace\WisFactory`（main 分支）。
> 开工 BASE = **3cdde58**（`git rev-parse --short HEAD`）；实现提交 = **4d4012e**（3 文件 / +521 −0）。
> 契约遵守：no-subagents；只改 brief 声明范围；未把 TASK 置「已完成」；未改 workitems 库（本报告为唯一例外）；TDD 红转绿留痕；提交注明 TASK-0083；0081 内核 `IotUpgradeApplyServiceImpl` 零改动。
> 环境事实（2026-11-24 实测）：MySQL 8.0.12@3307 / Redis 3.0.504@6379（宿主 phpStudy）；sql-exec.sh 每条语句单独一次调用；dev×1 + prod×1 轮用毕即停（8098 启动前探测空闲、两次 kill 后均确认 LISTENING 归零）；TDengine 本批不涉（启动期 WARN「TDengine REST 不可达 localhost:6041」为既有 job 行为，非本批缺陷）。
> **并行事实**：开工时 HEAD=3cdde58；期间并行道 TASK-0041 于 **9a6cda6** 提交 `calc/**`（双触发执行器）。本任务**未碰 calc/**，其提交与本任务提交在历史中分离；范围核对按「本任务提交 4d4012e^..4d4012e」与「BASE..HEAD 全量」双口径分别呈现（见验收 4）。

---

## 一、改动文件清单

本任务提交 `git show --stat 4d4012e` 原始输出：

```
 .../service/impl/IotPublishServiceImpl.java        | 143 +++++++++
 .../service/impl/IotPublishAutoFollowTest.java     | 353 +++++++++++++++++++++
 .../service/impl/IotPublishServiceImplTest.java    |  25 ++
 3 files changed, 521 insertions(+)
 create mode 100644 backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishAutoFollowTest.java
```

`git diff --name-only 4d4012e^..4d4012e` 原始输出（3 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishAutoFollowTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImplTest.java
```

**逐文件职责**：
- `IotPublishServiceImpl.java`（发布域主代码，+143）：`publish()` 事务内接线点 1 行（`:268 followUnlockedInstances(tenantId, scopeType)`，位于版本落库 + 快照 promote + config_version 回写 + 下发落行之后、`return row` 之前，不另开事务）；新增私有方法 `followUnlockedInstances`（`:290`）——计算域 scope 下按模板筛 `locked=0 AND applied<current` 滞后实例，逐台判 T3（`:343` 跳过）+ 复用 0081 `applyToVersion` + 乐观锁推进 applied；两个私有辅助（版本 content 装载 / 判档解析）。四端点对外签名零改动。
- `IotPublishAutoFollowTest.java`（新测试，+353）：6 条红线用例（未锁定跟随 / 锁定不跟随 / 网关不触发 / T3 跳过 / 失败穿透 / 乐观锁 4002）。
- `IotPublishServiceImplTest.java`（既有 +25）：注入 4 个新依赖 mock + `templateMapper.selectList` 默认空桩，既有发布测试零回归（2401 全绿实证）。

---

## 二、每条验收命令原始输出

### 验收 1：`mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误

ot-core install 段（最终提交 4d4012e 上重跑）：

```
[INFO] BUILD SUCCESS
[INFO] Total time:  4.804 s
```

ot-admin 全量 Results 段（最终提交 4d4012e、并行 0041 已入 HEAD 的完整重跑，原始输出）：

```
[INFO] Results:
[INFO] 
[INFO] Tests run: 2401, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:07 min
[INFO] Finished at: 2026-11-24T01:50:53+08:00
[INFO] ------------------------------------------------------------------------
```

**通过**：2401 通过 / 0 失败 / 0 错误 / 0 跳过。计数说明：0081 修订轮基线 2387 → 本任务新红线测试 6 例 → 2393；并行 TASK-0041（calc/**）在 HEAD 合入带来其契约测试（+8）→ 2401；既有发布/快照/确认/升级链测试零回归。

**TDD 红转绿留痕**（brief 测试口径）：先落 `IotPublishAutoFollowTest` 6 用例，将接线点临时注释为 TDD-RED 探针后首跑——3 例行为面红：

```
[ERROR] Tests run: 6, Failures: 3, Errors: 0, Skipped: 0, Time elapsed: 4.532 s <<< FAILURE! -- in net.otlab.admin.module.iot.publish.service.impl.IotPublishAutoFollowTest
[ERROR] ...IotPublishAutoFollowTest.computePublishFollowsUnlockedLaggingInstance -- FAILURE!
Wanted but not invoked:
[ERROR] ...IotPublishAutoFollowTest.followFailureAbortsPublish -- FAILURE!
  :263 自动跟随失败必须穿透发布链（@Transactional 整批回滚，不产生半态） ==> Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
[ERROR] ...IotPublishAutoFollowTest.followOptimisticConflictAborts -- FAILURE!
  :276 Expected net.otlab.core.exception.BusinessException to be thrown, but nothing was thrown.
[INFO] BUILD FAILURE
EXIT=1
```

恢复接线后目标 6 例全绿（`mvn -pl ot-admin -am test -Dtest=IotPublishAutoFollowTest` EXIT=0；日志含 `[PUBLISH-FOLLOW] skipped (T3 never auto) instanceId=201 ...` 与 `[PUBLISH-FOLLOW] instanceId=201 ... applied 2 -> 3 (userId=10)`）。

**「为什么现在对了」（逐条修复/实现要点）**：
1. **接线点位置**：初版若把 follow 放在 `assemble()` 之前会使当期快照含推进后行、违背 brief「版本落库+快照存储完成后」的时序裁决。为什么现在对了：接线落在 `return row` 前（快照/版本行/下发已落），当期快照保留跟随前基线（冒烟 v10 实证 two /100），下一期快照体现推进后行（v11 实证 E8C /1000）——与 0081「确认后下一期发布哈希变化」同一可测口径，且时序与 brief 逐字一致。
2. **T3 红线**：0081 评审红线「自动跟随不得触碰含 T3 的滞后实例」。为什么现在对了：`followUnlockedInstances` 对每台做 `TemplateContentDiffEngine.diff(applied, current)`，任一 `e.getTier()==TIER_3` 即跳过（`:343`），逐台留 INFO；冒烟 v12 两台均 `skipped (T3 never auto)`、applied 未变、行未逻辑删——T3 永不自动实证。
3. **失败整批回滚**：initially 担心 apply 失败被吞成「版本已发布但实例未跟随」半态。为什么现在对了：follow 不 catch 异常、不另开事务，apply 抛 `BusinessException` 直接逃出 publish 的 `@Transactional(rollbackFor=Exception.class)`；冒烟 WEEKLY 坏码 → publish 400 + 无 v13 版本行 + 实例 calc_period 仍 60（零半态）。
4. **锁定实例**：SQL 筛选 `eq(locked,0)` + 内存二次防线双重保证。为什么现在对了：冒烟 E8D(locked=1) 始终 applied=2 /100，且在 0016 清单可见（total=1 时唯一行）——与 brief「锁定实例保持旧版 + 仍在清单」一致。
5. **网关 scope**：接线点入口 `if (scopeType != SCOPE_COMPUTE) return`。为什么现在对了：单测 `gatewayPublishNoFollow` 断言 `templateMapper.selectList` 零调用；冒烟网关发布后两滞后实例 applied 不变、无新 `[PUBLISH-FOLLOW]` 日志。
6. **并发冲突**：乐观锁 `UPDATE ... WHERE applied=旧值` affected=0 → 4002。为什么现在对了：与 0081 confirm 同范式（单测 `followOptimisticConflictAborts` 断言 4002），不静默跳过。

### 验收 2：dev 冒烟全序列（每步原始输出）

dev 档启动（`mvn spring-boot:run` 默认 dev，8098 启动前探测空闲）：

```
2026-11-24 01:38:01.941 [main] INFO  org.springframework.boot.tomcat.TomcatWebServer - Tomcat started on port 8098 (http) with context path '/'
2026-11-24 01:38:01.957 [main] INFO  net.otlab.admin.OtAdminApplication - Started OtAdminApplication in 13.574 seconds (process running for 14.024)
登录：POST /auth/login/test {"username":"admin","password":"admin123","tenantCode":"default"}
{"code":200,"data":{"token":"9d69f89a-...","userId":1,"username":"admin",...},"message":"操作成功"}
```

启动段 0 ERROR。全程 3 条 ERROR 均为负例探针/边界测试的预期业务拒绝（PARAM_X 未登记码 ×1、WEEKLY 坏码 fail-closed ×2 留痕），非系统缺陷。

**STEP1-2 建链（模板 13 v2 + 两实例，E8D 锁定）**

```
POST /iot/template {"templateCode":"SMOKE0083_TPL",...} → {"code":200}  → 模板 id=13 version=1
POST /iot/template/item（ELEC_CONS 直绑 MINUTE + LOAD_RATE 公式 "{SELF.ELEC_CONS}/100"）→ {"code":200,"data":2}
DB：SELECT id,template_id,version,tier_summary FROM iot_model_template_version WHERE template_id=13 ORDER BY version
17  13  1  null
18  13  2  T1:2
POST /iot/instance shortCode=E8C/E8D → {"code":200} ×2
PUT /iot/instance/lock {"instanceId":33,"locked":1} → {"code":200}
DB：SELECT id,instance_code,template_id,applied_template_version,locked,status FROM iot_model_instance WHERE template_id=13
32  F01.WS13.E8C  13  2  0  1     ← 未锁定
33  F01.WS13.E8D  13  2  1  1     ← 锁定
DB：SELECT id,instance_id,data_code,formula,calc_period FROM iot_data_item WHERE instance_id IN (32,33)
41  32  ELEC_CONS  null                         60
42  32  LOAD_RATE  F01.WS13.E8C.ELEC_CONS/100    60
43  33  ELEC_CONS  null                         60
44  33  LOAD_RATE  F01.WS13.E8D.ELEC_CONS/100    60
```

**STEP3：T2 变更——改 LOAD_RATE 公式 /100 → /1000（item 24，模板 v2→v3 T2:2）**

```
PUT /iot/template/item {"id":24,...,"defaultFormula":"{SELF.ELEC_CONS}/1000","unit":"%"}
{"code":200,"message":"操作成功"}
DB：19  13  3  T2:2     （unit 与 formula 同档 T2，净 diff 无 T3）
```

**STEP4：发布前基线**

```
GET /iot/template/impact-preview?templateId=13
{"code":200,"data":{"targetVersion":10,"affectedInstanceCount":2,"lockedInstanceCount":1,
 "tierSummary":"T2:2","previewToken":"ab6a29f16d044971827d72219cff0a8b",...}}
DB 末版：v9 snapshotHash=64fdb2167bb37504f4fc4ef9250f04764d8da5c4350c61c0d2f4ad7cdffc177e
```

**STEP5：计算域发布 v10（带 token）→ 自动跟随实证**

```
POST /iot/publish/publish {"scopeType":2,"scopeId":0,"confirmToken":"ab6a29f1..."}
{"code":200,"data":{"id":46,"version":10,"snapshotHash":"7d7feb4352cc64bcf1b456e82c30fe534ff00ba5f621756f56d75c74b0bf3465","snapshotUri":"1/2-0-10.json",...}}
DB after：SELECT id,instance_code,applied_template_version,locked FROM iot_model_instance WHERE id IN (32,33)
32  F01.WS13.E8C  3  0     ← applied 2→3 自动跟随
33  F01.WS13.E8D  2  1     ← 锁定保持旧版
DB after：SELECT id,instance_id,data_code,formula FROM iot_data_item WHERE instance_id IN (32,33)
42  32  LOAD_RATE  F01.WS13.E8C.ELEC_CONS/1000   ← 数据面生效
44  33  LOAD_RATE  F01.WS13.E8D.ELEC_CONS/100    ← 保持旧口径
服务日志：
[UPGRADE-APPLY] instanceId=32 templateId=13 applied=2 target=3 summary=T2:2 actions=[LOAD_RATE 公式继承新默认: F01.WS13.E8C.ELEC_CONS/1000, LOAD_RATE 单位继承新默认: %]
[PUBLISH-FOLLOW] instanceId=32 templateId=13 applied 2 -> 3 (userId=1)
   ← 上下文核实实证：applyToVersion 在发布场景直接调用成功、不抛 SecurityUtils/TenantContext 异常，留痕操作人 userId=1 正确（0081 m-1 闭合）
```

**STEP6：快照哈希变化实证（数据面生效）**

```
v10 快照体（backend/ot-admin/data/snapshots/1/2-0-10.json，跟随前组装）：
  grep -o "F01.WS13.E8[CD].ELEC_CONS/[0-9]*" → F01.WS13.E8C.ELEC_CONS/100 + F01.WS13.E8D.ELEC_CONS/100
发布 v11（无 token —— 未锁定滞后已清零，确认门自动放行）：
POST /iot/publish/publish {"scopeType":2,"scopeId":0}
{"code":200,"data":{"version":11,"snapshotHash":"ad44cebf9f37190067542d6391e9dacc4e469c7717ace6cbd3a9eeb8befab457",...}}
v11 快照体（2-0-11.json，跟随生效后）：
  F01.WS13.E8C.ELEC_CONS/1000 + F01.WS13.E8D.ELEC_CONS/100   ← 未锁定新值 + 锁定旧值同快照并存
=== 哈希比对 ===
v9  (T2 变更前)  64fdb2167bb37504f4fc4ef9250f04764d8da5c4350c61c0d2f4ad7cdffc177e
v10 (T2 发布)    7d7feb4352cc64bcf1b456e82c30fe534ff00ba5f621756f56d75c74b0bf3465
v11 (跟随生效)   ad44cebf9f37190067542d6391e9dacc4e469c7717ace6cbd3a9eeb8befab457  ← 变化，数据面生效实证
```

**STEP7：锁定实例仍在 0016 清单**

```
GET /iot/template/upgrade/list?templateId=13
{"code":200,"data":{"total":1,"records":[
  {"instanceCode":"F01.WS13.E8D","appliedTemplateVersion":2,"currentVersion":3,"locked":1,"tierSummary":"T2:2",...}
]}}
   ← E8C 已跟随不在清单；E8D 锁定保持旧版且清单可见
```

**STEP8：网关 scope 发布不触发实例跟随**

```
=== 先制造滞后：T2 改 unit（v3→v4 T2:1），新建 E8E(id=34,applied=3,unlocked) ===
DB lagging：32 F01.WS13.E8C applied=3 / 34 F01.WS13.E8E applied=3（均 <4 且 locked=0）
POST /iot/publish/publish {"scopeType":1,"scopeId":1}
{"code":200,"data":{"version":1,"scopeType":1,"scopeId":1,...}}
DB after：32 applied=3 / 34 applied=3   ← 均未推进
PUBLISH-FOLLOW 日志：仅 STEP5 的 1 行（无网关发布新增行）
   ← 网关 scope 不触发跟随实证
```

**STEP9：T3 变更发布不自动跟随**

```
=== 删 LOAD_RATE（item 24）→ 模板 v5 T3:1；E8C/E8E applied=3 滞后且净 diff 含 T3 ===
GET impact-preview?templateId=13 → tierSummary=T3:1 targetVersion=12 previewToken=04c5ad6a...
POST /iot/publish/publish {"scopeType":2,"scopeId":0,"confirmToken":"04c5ad6a..."}
{"code":200,"data":{"version":12,"snapshotHash":"9a69583b991b80fd87b9f30ffeec2600eeb42dafb50e376dc9b930965dc9b442",...}}
DB after：32 applied=3 / 33 applied=2 / 34 applied=3   ← 均未推进
DB：LOAD_RATE 行 42(实例32)/46(实例34) deleted=0   ← 未逻辑删（不自动迁移）
服务日志：
[PUBLISH-FOLLOW] skipped (T3 never auto) instanceId=32 templateId=13 applied=3 current=5
[PUBLISH-FOLLOW] skipped (T3 never auto) instanceId=34 templateId=13 applied=3 current=5
   ← T3 永不自动实证（两实例仍走 0016 逐台确认）
```

**STEP10：失败整批回滚红线（WEEKLY 坏码 fail-closed）**

```
=== 模板 14 SMOKE0083_FAIL：加 ELEC_CONS(v2) → 实例 35 E9A applied=2 → 改 defaultCalcPeriod=WEEKLY (v3 T2:1) ===
DB before：MAX(version)=12 cnt=12（计算域）; 实例35 applied=2
GET impact-preview?templateId=14 → targetVersion=13 token=e4e8b119bcc24bac83943156dae58f90
POST /iot/publish/publish {"scopeType":2,"scopeId":0,"confirmToken":"e4e8b11..."}
{"code":400,"message":"模板默认计算周期无法识别（支持 SECOND/MINUTE/HOUR/DAY 或秒数值）: WEEKLY"}
DB after：MAX(version)=12 cnt=12   ← 无 v13 版本行（整批回滚）
DB after：实例35 applied=2；iot_data_item id=47 calc_period=60   ← 簿记/数据面均未推进（零半态）
日志：ERROR 业务异常：模板默认计算周期无法识别（...）: WEEKLY（GlobalExceptionHandler 如实留痕）
注：2-0-13.json 暂存/落位文件为无行引用的孤儿（promote 先于事务提交）——快照存储类既有权衡，可自愈，非半态。
```

**验收 2 全序列通过**：dev 全程启动 0 ERROR、3 条 ERROR 均为负例探针预期业务拒绝；逐台 `[UPGRADE-APPLY]`/`[PUBLISH-FOLLOW]` INFO 留痕在位；上下文核实（m-1）实证成立；验毕 taskkill PID 86764 → 8098 LISTENING 归零。

### 验收 3：prod 档真实启动回归（0 ERROR）

启动脚本注入本地连接与密钥（8098 启动前探测空闲）：

```
DB_HOST=127.0.0.1 DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root
REDIS_HOST=127.0.0.1 REDIS_PORT=6379 REDIS_DATABASE=0 SM4_KEY=0123456789abcdef
CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 WECHAT_MP_ENABLED=false
mvn spring-boot:run -pl ot-admin -Dspring-boot.run.profiles=prod
启动日志：
2026-11-24T01:46:10.750+08:00  INFO ... The following 1 profile is active: "prod"
2026-11-24T01:46:24.899+08:00  INFO ... Tomcat started on port 8098 (http) with context path '/'
2026-11-24T01:46:24.912+08:00  INFO ... Started OtAdminApplication in 14.569 seconds (process running for 14.939)
ERROR_COUNT=0 （grep -c " ERROR " prod.log = 0，含启动与全部探针）
```

prod 正式登录链：`GET /auth/captchaImage` → uuid → redis-cli（`/d/phpstudy_pro/Extensions/redis3.0.504/redis-cli.exe`）读 `captcha:code:<uuid>` → SM4 密文口令（复用 0081 报告同 key 密文）→ `POST /auth/login` → `{"code":200,"data":{"token":"394903ea-...","username":"admin"}}`。

```
PROBE1 GET /iot/publish/list?scopeType=2&scopeId=0&pageNum=1&pageSize=3
{"code":200,...} 记录：v12 9a69583b991b80fd / v11 ad44cebf9f371900 / v10 7d7feb4352cc64bc
   ← dev 冒烟跟随链产出的三版本在 prod 可见（档位切换零差异实证）
PROBE2 GET /iot/template/upgrade/list?templateId=13
{"code":200,"data":{"total":3,"records":[
  F01.WS13.E8C applied=3 locked=0 / F01.WS13.E8D applied=2 locked=1 / F01.WS13.E8E applied=3 locked=0]}}
   ← 0016 清单在 prod 档生效
```

**通过**：prod 档真实启动 14.6s 全程 0 ERROR + captcha/SM4 正式登录链 + 发布列表/待升级清单全 200。验毕 taskkill PID 85328 → 8098 LISTENING 归零。

### 验收 4：改动范围核对（`git diff --name-only <BASE>..HEAD`）

**本任务提交**（`git diff --name-only 4d4012e^..4d4012e`，原始输出 3 文件，见「一」）——全落 brief 声明范围：
- `publish/service/impl/IotPublishServiceImpl.java` ∈ `iot/publish/**`（brief 改动范围第 1 条：发布事务接线——最小改动）✓
- `IotPublishAutoFollowTest.java` + `IotPublishServiceImplTest.java` ∈ `src/test/java/**`（brief 第 3 条：红线测试）✓

越界反查：

```
git diff --name-only 4d4012e^..4d4012e | grep -E "calc/|\.ya?ml|resources/sql|ot-core|frontend|ingest|IotUpgradeApplyServiceImpl"
→ NONE
```

即 **0081 内核 `IotUpgradeApplyServiceImpl` 零改动、SQL/yml/ot-core/前端/ingest/calc 零触碰**（0081 接缝仅以调用方身份复用 `IotUpgradeApplyService#applyToVersion`）。

**BASE..HEAD 全量**（`git diff --name-only 3cdde58..4d4012e`，原始输出 = 本任务 3 文件 + 并行 TASK-0041 的 calc/** 24 文件）：calc/** 全部来自并行道独立提交 **9a6cda6**（`TASK-0041: 双触发执行器`），与本任务提交在历史中分离、本任务未参与；按 brief「只改 brief 声明范围」的核对以本任务提交为对象，**通过**。

---

## 三、未做项与原因

1. **0081 编排内核改动**：brief 明示「0081 内核零改动，本任务只是调用方」——本任务只调 `applyToVersion`，未改 `IotUpgradeApplyServiceImpl` 一行（越界反查 NONE）。
2. **T3 实例自动迁移**：brief 决策 2/5「T3 永不自动、仍走 0016 逐台确认」——发布跟随遇 T3 净 diff 即跳过（`:343`），冒烟 STEP9 实证，未实现自动迁移（设计如此）。
3. **锁定实例推进**：brief 决策 5「锁定实例不推进（保持旧版本 + 仍在 0016 清单）」——筛选 `locked=0` 排除，冒烟 STEP5/7 实证。
4. **网关 scope 跟随**：brief 决策 2「scope_type=1 网关域不触发实例跟随」——入口早返回，冒烟 STEP8 实证。
5. **发布四端点契约变更**：brief 决策 5「不改 publish 对外契约（四端点签名不动）」——`IotPublishController` 与 `PublishForm` 零改动，接线在服务内部。
6. **SQL/yml/ot-core/前端/ingest/calc**：brief 改动范围外，零触碰（越界反查 NONE）。
7. **TASK 状态回写**：按契约「不把 TASK 置已完成」，状态保持「进行中」，由协调者收尾。

---

## 四、偏离计划之处

1. **接线时序的快照语义**：brief 决策 1 措辞「版本落库+快照存储完成后」——实现严格照此（接线在 `return row` 前），故**当期发布快照保留跟随前基线**（v10 both /100），数据面推进体现在**下一期快照**（v11 E8C /1000）。这与 brief 验收 2「snapshotHash 变化实证」以「v10→v11 哈希变化」口径满足，与 0081「确认后下一期发布哈希变化」同构。**提示**：若产品语义要求「发布当期快照即含跟随后的新值」，需把 follow 提前到 `assemble()` 之前——这会改变 brief 裁决的接线时序，须另立裁决，本任务按 brief 原文执行、未越界调整。
2. **测试 hook 位置**：红线测试经公共 `publish()` 入口驱动（私有 `followUnlockedInstances` 不可直测），0081 数据面行应用由其自身 12 例单测覆盖，本类聚焦接线/筛选/事务语义——未重复 0081 断言面。
3. **既有测试夹具补桩**：`IotPublishServiceImplTest` 增 4 依赖 mock + `templateMapper.selectList` 默认空桩（+25 行）——为保证既有发布测试在计算 scope 下零回归，非测试用例语义变更。
4. **并行提交历史**：开工 BASE=3cdde58、实现提交=4d4012e，期间并行 0041 于 9a6cda6 提交 calc/**；范围核对按「本任务提交」口径（见验收 4），未因并行提交污染本任务足迹判定。
5. **快照孤儿文件**：失败回滚红线中 promote 先于事务提交，坏码场景留下无行引用的 `2-0-13.json`（同 0024 快照存储类既有权衡，可自愈）——非本任务新增语义，登记备查。

---

## 五、遗留问题（含上下文核实结论）

1. **上下文核实结论（brief 决策 3 / 0081 m-1，核心必答项）**：**适用——不触发停写报告**。实证两条：
   - 静态链路：`publish()` 入口已成功取 `SecurityUtils.getUserId()`（`:202`）与 `TenantContext.getTenantId()`（`:201`）；计算域发布经 `PublishScopeGuard.assertScopeWritable` 要求 `visible.allTenant()`（管理员语义），与 `regenerateForInstance` 内部 `dataPermService.resolve(SecurityUtils.getUserId())` 的可见集要求完全一致（allTenant 直过），上下文必然齐备。
   - 动态实证：dev 冒烟 STEP5 在真实登录会话（`POST /auth/login/test`，userId=1）中直接触发 `applyToVersion`，日志 `[UPGRADE-APPLY] instanceId=32 ... summary=T2:2 actions=[...]` + `[PUBLISH-FOLLOW] ... (userId=1)`——**未抛任何 SecurityUtils/TenantContext 上下文异常，留痕操作人 userId=1 正确**。0081 修订轮 m-1 deferred 项在本任务发布场景闭合，无需为接缝新增免会话入口。
2. **当期快照 vs 跟随快照的时序权衡**：见「四.1」——当期快照保留跟随前基线，数据面在下一期快照体现。已按 brief 原文执行；如产品期望「发布即当期生效快照」，需另立裁决调整接线时序。
3. **快照孤儿文件**：坏码回滚留下无行引用的 `2-0-13.json`（0024 既有存储权衡，可自愈）；未清理（照 0015/0016/0081 冒烟遗留隔离口径，避免与 DB 行引用一致性冲突）。
4. **冒烟数据留存库内**：模板 13（SMOKE0083_TPL v1→v5）/ 14（SMOKE0083_FAIL v3 WEEKLY）/ 实例 32-35 / 版本行 17-21 + 14 的版本行 / 发布 v10-v12（hash 7d7feb43→ad44cebf→9a69583b）/ 快照文件 2-0-10/11/12 + 孤儿 2-0-13；重跑冒烟需换短码（同码撞占用属预期）。
5. **T3 实例仍待 0016 逐台确认**：冒烟 E8C/E8E 因 T3 跳过后仍在 0016 清单（prod PROBE2 total=3 含它们）——按 ADR-0008「T3 永不自动、逐台确认迁移」为预期遗留，非缺陷。
6. **TDengine 未触碰**：本批数据源全为 MySQL 业务表；启动期 `IotQuotaStatsServiceImpl` WARN「TDengine REST 不可达 localhost:6041」为既有 job 行为，与本批无关。
7. **越权 live 冒烟**：以既有单测（发布防线 `assertScopeWritable` 计算 scope 管理员语义 + 0016 确认越权用例）为口径；brief 冒烟序列未含受限账号建链，未 live 验证。

---

## 六、一句话结论

**通过**——验收 1（最终提交 4d4012e 全量 mvn：**2401 测试 0 失败 0 错误 BUILD SUCCESS**，TDD 红 3 例→绿 6 例，接线/T3/失败/锁定/网关/并发六条修复均附「为什么现在对了」）、验收 2（dev 冒烟全序列：模板 v3/T2:2 → **发布 v10 带 token → 未锁定 E8C applied 2→3、公式 `/100`→`/1000`、`[PUBLISH-FOLLOW] userId=1` 上下文实证**，锁定 E8D 保持 applied=2 `/100` 且清单可见 → v10 快照 both /100、**v11 快照 E8C /1000 + E8D /100，hash 7d7feb43→ad44cebf 变化即数据面生效** → 网关发布不触发 → T3 发布 v12 两台 `skipped (T3 never auto)`、applied/行不变 → WEEKLY 坏码发布 400、无 v13 行、calc_period 仍 60 即整批回滚）、验收 3（prod 启动 14.6s **0 ERROR** + captcha/SM4 正式登录链 + 发布列表/升级清单全 200）、验收 4（本任务足迹 3 文件全落 brief 声明范围，0081 内核/SQL/yml/ot-core/前端/ingest/calc 零触碰，calc/** 属并行 0041 独立提交）四条全部以原始输出为证通过；上下文核实（0081 m-1）实证适用——发布事务场景 `applyToVersion` 不抛上下文异常、操作人留痕正确；偏离均登记，遗留仅登记级，无阻断缺陷。

---

---

# 修订轮报告（TASK-0083 round1 diff 评审 F-1 退回 + F-2 顺手修 + F-3 登记 deferred）

> 修订执行者：TASK-0083 执行者；修订日期：2026-11-24；工作区 `D:\workspace\WisFactory`（main 分支）。
> 输入：`TASK-0083-review.md`（F-1 完整事实链 + 越界反查复核）、协调者修订派发（§12，用户定性=计划缺陷、已修 brief 决策 1）。
> 修订 BASE = **4d4012e**（round1 实现提交，开工 `git rev-parse --short HEAD`）；修订提交 = **7628007**（2 文件，+79 −10）；期间并行 TASK-0041 修订轮于 73b8587 提交 calc/**，本轮未触碰。
> 契约：no-subagents；修订授权范围 = 原 brief 范围（backend publish/template 域）；0081 内核零改动、SQL/yml/ot-core/前端/ingest/calc 零触碰；不把 TASK 置「已完成」；workitems 仅追加本报告；提交注明 TASK-0083 修订轮。
> 复评为 scoped：仅看 F-1 修复 diff 与其回归影响，不重开全场。

## 一、改动文件清单（修订轮 2 文件）

`git show --stat 7628007` 原始输出：

```
 .../service/impl/IotPublishServiceImpl.java        | 30 +++++++----
 .../service/impl/IotPublishAutoFollowTest.java     | 59 +++++++++++++++++++++-
 2 files changed, 79 insertions(+), 10 deletions(-)
```

`git diff --name-only 7628007^..7628007` 全部落原 brief 授权范围（publish/service/impl 主代码 + src/test/**）；`git diff --name-only | grep -E "calc/|\.ya?ml|resources/sql|ot-core|frontend|ingest|IotUpgradeApplyServiceImpl"` → NONE；0081 内核/SQL/yml/ot-core/前端/ingest/calc 零触碰，**通过**。

## 二、修复内容与「为什么现在对了」（逐条 TDD 红→绿）

### 1. F-1（Important，必须修）——follow 后置导致当期发布快照仍含旧公式，T2「随发布生效」静默落空

- **根因**（评审事实链逐行核实）：round1 把 `followUnlockedInstances` 接在 `recordDispatch` 之后（assemble→SHA→stage→INSERT→promote→回写→dispatch→**follow**）。计算引擎运行时**只读落位快照文件、不读实例行**（`HttpSnapshotFetcher.endpoint()` 不带 version 拉最新已发布版 → `IotPublishServiceImpl.retrieveSnapshotInternal` 读落位文件 → `SnapshotReconciler` 从 bodyJson 建依赖图 → `DualTriggerExecutor.computeItem` 从图节点取 formula）。故携带 T2 二次确认的发布 v10 快照仍是跟随前 `/100`，新值 `/1000` 只进 v11。更严重：发布后 applied 已推进、0016 清单显示已跟上、`hasPendingTier2Confirm` 窗口关闭（滞后未锁定实例=0）→ 无第二次发布时实例**永久**执行旧公式，簿记/清单均显示正常、无 fail-closed 信号。评审判定 Important；协调者定性为计划缺陷（brief 决策 1 裁决错误）并已修 brief。
- **修法**：把 `followUnlockedInstances(tenantId, scopeType)` 移到确认门**之后、assemble 之前**；新顺序 = 确认门 → follow（推进未锁定实例行）→ assemble（读到新行）→ SHA-256/store → 版本落库 → publish_log/notify。类 javadoc 与接线点注释同步改写（明示「calc 只读快照」根因）。follow 失败整批回滚、零半态不变；网关 scope 早返回、T3 跳过、乐观锁 4002 全部保持。
- **为什么现在对了**：assemble 在 follow 之后执行，`ConfigSnapshotAssembler.assembleComputeScope` 从 `iot_data_item` 读到的已是 0081 `applyToVersion` 改写后的新行——当期发布快照即含新公式，下发到 calc 的就是新值，T2「随发布生效」在当期成立。异常仍不 catch、不另开事务，沿 publish 的 `@Transactional(rollbackFor=Exception.class)` 逃出 → 版本行/快照/下发/实例推进同原子回滚（失败语义不变）。
- **红（对旧时序必红）**：新增用例 `currentSnapshotContainsPostFollowValue`——可变 `IotDataItem` 桩（applyToVersion 调用时把 formula 由 `/100` 改写 `/1000`），断言**当期发布快照落位文件内容**含 `/1000` 且不含旧值 `/100"`：

```
[ERROR] Tests run: 2, Failures: 2, Errors: 0, Skipped: 0 -- in ...IotPublishAutoFollowTest
net.otlab.admin.module.iot.publish.service.impl.IotPublishAutoFollowTest.currentSnapshotContainsPostFollowValue -- FAILURE!
org.opentest4j.AssertionFailedError: 当期快照必须含跟随后新值 /1000（calc 运行时只读快照，T2 随发布生效） ==> expected: <true> but was: <false>
```

- **绿**：同用例在新时序下通过（8 例全绿，见 §三），当期快照体实证见 dev 冒烟 STEP5。

### 2. F-2（Minor，顺手修）——follow 筛选加 status=1，与 0016 清单/确认门一致

- **根因**：round1 follow 筛 `locked=0 AND applied<current`，无 `status` 条件；而确认门 `hasPendingTier2Confirm`（`IotTemplateVersionServiceImpl:326-330`）与 0016 清单均只统计**启用中**（`status=1`）滞后实例。口径不一致：停用实例（`status≠1`）本不参与影响预览 affected 数、也不进 0016 清单，却被 follow 一并推进——方向保守但留痕/一致性有偏。
- **修法**：筛选加 `.eq(IotModelInstance::getStatus, 1)`（SQL 层），javadoc 与注释同步标注「与 0016 清单/确认门一致——修订轮 F-2」。
- **为什么现在对了**：follow 统计口径 = 影响预览 affected = 确认门 lagging = 0016 清单（四处同一 `status=1`），停用实例不再被自动推进——与 ADR-0008「未锁定启用实例随发布生效」一致，避免与清单显示矛盾。
- **红→绿**：`followFilterIncludesStatusEnabled` 用 ArgumentCaptor 截 `instanceMapper.selectList` 的 `LambdaQueryWrapper`，断言 `getSqlSegment()` 含 status 条件。旧代码红：

```
followFilterIncludesStatusEnabled -- FAILURE!
org.opentest4j.AssertionFailedError: follow 实例筛选必须含 status=1（与 0016 清单/确认门口径一致）；
实际 SQL=(template_id = #{...} AND locked = #{...} AND applied_template_version < #{...}) ORDER BY id ASC ==> expected: <true> but was: <false>
```

绿：SQL 段含 `status = #{...}`（目标 8 例全绿）。

### 3. F-3（Minor，登记 deferred 不修）——无会话发布时 TenantContext 短路风险

- **事实**：`templateMapper.selectList(new LambdaQueryWrapper<>())` 无显式 tenant 条件，靠 `OtTenantLineHandler` 在会话有租户时注入；`SecurityUtils.getUserId()` 无上下文会抛。
- **不修理由**（修订派发明示 deferred）：评审者独立 grep 全仓确认**仅 `IotPublishController.publish`（@SaCheckPermission 登录端点）调用 `publishService.publish`**，`PublishSchedulingConfig` 仅注册 ACK 超时扫描、无内部/定时调用——当前无无会话调用方，属潜伏风险而非现存缺陷。为接缝加防御需改 tenant 注入范式或引入显式上下文参数（超授权范围），本轮不改。
- **交接**：若未来引入内部/定时无会话发布，须为 follow 链路显式注入 tenant/userContext 或加无会话短路防御；当前登记 deferred。

## 三、验收命令原始输出（修订轮口径 = 全量 mvn Results 段 + dev 当期快照实证）

```
cd backend && mvn install -pl ot-core -DskipTests    → BUILD SUCCESS（Total time: 1.844 s，EXIT=0）
cd backend && mvn test -pl ot-admin                  → Results 段原始输出：

[INFO] Results:
[INFO] 
[INFO] Tests run: 2403, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:22 min
[INFO] Finished at: 2026-11-24T02:15:15+08:00
[INFO] ------------------------------------------------------------------------
```

2403 = round1 基线 2401 + 本轮新增 2（F-1 当期快照断言 / F-2 status 过滤断言）；既有 2401 零回归。目标测试类 surefire 实测：`IotPublishAutoFollowTest  Tests run: 8, Failures: 0, Errors: 0, Skipped: 0`（原 6 + 修订 2）。

### dev 冒烟重跑关键实证（当期快照含新值，F-1 核心验收）

启动段 0 ERROR（`Started OtAdminApplication in 23.15 seconds`）。**STEP5（发布当期即验）原始输出**：

```
=== 建链 ===
模板 15 SMOKE0083_R2 → items(ELEC_CONS/LOAD_RATE /100) v2 → 实例 36=E8F(unlocked) / 37=E8G(locked) applied=2
T2 变更：item 27 LOAD_RATE /100 → /1000 → 模板 v3 T2:1
impact-preview?templateId=15 → targetVersion=13 tier=T2:1 affected=2 locked=1 token=b4b66a66...
DB before：36 applied=2 formula=F01.WS13.E8F.ELEC_CONS/100 / 37 applied=2 formula=F01.WS13.E8G.ELEC_CONS/100

=== 发布 v13（带 token，仅此一次发布，无 v14）===
POST /iot/publish/publish {"scopeType":2,"scopeId":0,"confirmToken":"..."}
{"code":200,"data":{"version":13,"snapshotHash":"0e2c6386e8adb5e7c6320d782760f45a02173e030e3196c83f39c6b93172c635",
 "snapshotUri":"1/2-0-13.json",...}}
DB after：36 applied=3（自动跟随）/ 37 applied=2（锁定保持）
 36 49 LOAD_RATE F01.WS13.E8F.ELEC_CONS/1000   ← 数据面已改写
 37 51 LOAD_RATE F01.WS13.E8G.ELEC_CONS/100    ← 锁定旧口径
=== F-1 核心断言：当期发布快照 1/2-0-13.json 内容 ===
grep -o "F01.WS13.E8[FG].ELEC_CONS/[0-9]*" backend/ot-admin/data/snapshots/1/2-0-13.json
F01.WS13.E8F.ELEC_CONS/1000     ← 当期快照即含跟随后新值（F-1 通过，旧时序此处必为 /100）
F01.WS13.E8G.ELEC_CONS/100      ← 锁定实例旧值
日志：[UPGRADE-APPLY] instanceId=36 templateId=15 applied=2 target=3 summary=T2:1 actions=[LOAD_RATE 公式继承新默认: F01.WS13.E8F.ELEC_CONS/1000]
     [PUBLISH-FOLLOW] instanceId=36 templateId=15 applied 2 -> 3 (userId=1)   ← 上下文核实（m-1）延续成立
```

**其余红线在修订时序下回归**（简录，原始 DB/日志均已实测）：
- 锁定实例：0016 清单 template 15 `total=1` 仅 E8G applied=2 locked=1 ✓
- 网关 scope：模板 15 新 T2 改 /2000 制造滞后后发布网关 scope（version 2）→ 实例 36 仍 applied=3 不跟随 ✓
- T3 不自动：发布 v14（带 token，模板 13 T3 待确认）→ 实例 32/34 仍 applied=3、日志 `skipped (T3 never auto)` ✓
- 失败整批回滚：模板 14 `defaultCalcPeriod=WEEKLY` → 计算域发布 `code 400 模板默认计算周期无法识别...: WEEKLY`，max version 保持 13、实例 35/36 applied 不变、data_item 无部分写入 ✓
- 上下文核实（m-1）：`[PUBLISH-FOLLOW] ... (userId=1)` 真实会话留痕正确，无 SecurityUtils/TenantContext 异常 ✓
- dev ERROR 共 2 条，均为 WEEKLY 坏码预期业务拒绝（GlobalExceptionHandler 如实留痕），无系统 ERROR ✓

**prod 档真实启动回归**：17.174s、`The following 1 profile is active: "prod"`、`ERROR_COUNT=0`；captcha/SM4 正式登录链成功（token bc3af444-...）；PROBE 发布列表 v14=f8a1633255b8b117 / v13=0e2c6386e8adb5e7（当期快照哈希档位一致）、升级清单 template 15 total=1（仅 E8G locked）全 200。dev/prod 两次 8098 启动前探测空闲、用毕 taskkill 后 LISTENING 均归零（dev 82824 / prod 93852）。

## 四、未做项与原因

1. **F-3 无会话发布防御**：登记 deferred 不修——当前全仓唯一 publish 调用方为登录端点，无无会话场景；详见 §二.3。
2. **0081 内核改动**：brief 明文零改动，本轮仍只调用 `applyToVersion`（越界反查 NONE）。
3. **TASK 状态回写**：保持「进行中」，由协调者收尾。

## 五、偏离计划之处

1. **接线时序纠偏**：round1 严格照旧 brief「快照存储完成后 follow」执行并登记遗留；本轮按已修 brief 决策 1 前置到 assemble 之前——这是协调者定性计划缺陷后的修订，非执行者自发起义。原报告「四.1/五.2『当期快照保留跟随前基线、下一期才变』」遗留描述**作废**，以本报告「当期快照即含新值」为准。
2. **F-2 顺手修范围**：status=1 过滤属 brief「筛选口径」细化（与 0016/确认门对齐），未引入新的接口/契约变化。
3. **修订提交粒度**：单提交 7628007（实现+测试一体，红绿证据在最终代码上复核）；并行 0041 修订（73b8587 calc/**）在历史中分离、未混入本轮足迹。

## 六、修订轮一句话结论

**通过（修订轮，待 scoped 复评）**——F-1（Important）以「follow 前置 assemble 之前、当期发布快照即含跟随后新值」闭合（红：当期快照缺 `/1000` → 绿：v13 快照 `F01.WS13.E8F.ELEC_CONS/1000` 与锁定 E8G `/100` 同现，仅一次发布，T2 随发布生效落空缺口消除，失败整批回滚/零半态不变）；F-2 顺手修 status=1 筛选（红：SQL 无 status → 绿：与 0016/确认门同口径）；F-3 无会话风险登记 deferred（当前唯一调用方为登录端点）；全量 mvn **2403 测试 0 失败 0 错误 BUILD SUCCESS**（round1 基线 2401 + 2 新用例，原文 Results 段见 §三）；dev 冒烟当期快照实证 + 锁定/网关/T3/失败回滚/上下文五条红线回归、prod 启动 17.2s **0 ERROR** + SM4 登录链 + 发布列表/升级清单全 200；修订足迹 2 文件全落 brief 授权范围，0081 内核/SQL/yml/ot-core/前端/ingest/calc 零触碰。待协调者 scoped 复评。
