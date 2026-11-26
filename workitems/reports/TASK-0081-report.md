# TASK-0081 验收报告：确认升级数据面生效编排（实例行按新模板版本应用与 T3 逐台迁移）

> 执行者：TASK-0081 执行；验收日期：2026-11-23；工作区 `D:\workspace\WisFactory`（main 分支）。
> 开工 BASE = **09357d1**（`git rev-parse --short HEAD`）；实现提交 = **b491e34**（9 文件 / +1202 −17）。
> 契约遵守：no-subagents；只改 brief 声明范围；未把 TASK 置「已完成」；未改 workitems 库（本报告为唯一例外）；TDD 红转绿留痕；提交注明 TASK-0081。
> 环境事实（2026-11-23 实测）：MySQL 8.0.12@3307（root/root，宿主 phpStudy）/ Redis 3.0.504@6379（宿主 phpStudy，redis-cli 位于 `/d/phpstudy_pro/Extensions/redis3.0.504/redis-cli.exe`）；sql-exec.sh 每条语句单独一次调用；8098 启动前探测空闲、dev×1 + prod×1 轮用毕即停（两次 kill 后均确认监听归零）；TDengine 未触碰（本批不涉）；EMQX 联调 agent 在途（仅 env/配置，无源码冲突，未影响本批）。
> 并行事实：开工时工作区存在未跟踪 `.agents/`（环境产物），本批未触碰、未入库（`git status` 末尾始终仅余 `?? .agents/`）。

---

## 〇、开工复核性实证结论（brief 第一项：三项裁决对照源码，偏差即停）

执行者开工按 brief/TASK 正文逐条对照源码，**四项复核全部与裁决一致，无偏差，不触发停写报告条款**。证据：

1. **裁决①-a 基线 content 可解析且字段齐备**：`IotTemplateVersionServiceImpl.buildContentJson`（`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotTemplateVersionServiceImpl.java:419-465`）逐字段写入 canonical Map（TreeMap 键序）：`defaultFormula`/`defaultSourceType`/`defaultCalcPeriod`/`defaultValidMin`/`defaultValidMax`/`defaultAlarmLow`/`defaultAlarmHigh`/`aggMethod`/`valueKind` 九字段全在；`parse`（:488-497）以 `PARSE_MAPPER.readTree` 可解析（冒烟版本历史行 content 实读解析成功）。
2. **裁决①-b 兜底形态复现正确**：
   - `mapCalcPeriod`（`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java:1159-1184`）：null/空白→0、纯数字→秒直取、SECOND→1/MINUTE→60/HOUR→3600/DAY→86400、坏码 fail-closed 抛参错——冒烟模板 `MINUTE`→实例 `calc_period=60` 实证（见验收 2 STEP2 DB 行）;
   - `RelativeRefExpander.expand`（`.../dataitem/service/support/RelativeRefExpander.java:86-130`）：`{SELF.ELEC_CONS}/100` → `F01.WS13.E8A.ELEC_CONS/100`（冒烟 STEP2 实例行 formula 实证）；
   - null→1 兜底：`buildItem`（`IotDataItemServiceImpl.java:1099-1100`）`defaultSourceType != null ? 值 : SOURCE_TYPE_BIND(1)`——模板 `defaultSourceType` 为空生成落 `source_type=1`。三形态复现与裁决原文逐字相符。
3. **裁决③ 逻辑删双 uk 不含 deleted**：`resources/sql/upgrade/20261122_iot_data_item_domain.sql:94-95` 双唯一键 `uk_tenant_instance_data (tenant_id, instance_id, data_code)` / `uk_tenant_full_key (tenant_id, full_key)` 均不含 deleted；`BaseEntity.java:38-39` `@TableLogic` + `application.yml:181-183` 全局逻辑删配置（delete=1/not=0）；`IotDataItemMapper.selectOccupancyByInstance` 注释实证「逻辑删行永久占用键」——冒烟 T3 删项后 E8A 的 PARAM_C 行 `deleted=1`、活跃清单不再含该项（见验收 2 STEP8），裁决成立。
4. **裁决① 组装器读实例行不读 applied 列**：`ConfigSnapshotAssembler.assembleComputeScope`（`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/ConfigSnapshotAssembler.java:191-214`）`dataItemMapper.selectList` 读 iot_data_item 行、`dataItemRefMapper.selectList` 读引用边，`toDataItemObject`（:264-287）逐字段映射，**全文 grep `applied` 零命中**（仅在 javadoc 版本语义说明中出现）；冒烟 publish before/after 的 snapshotHash 随实例行公式变更而变（见验收 2 STEP3/STEP6），数据面生效路径实证成立。

---

## 一、改动文件清单（验收 4：改动范围核对）

`git diff --name-only 09357d1..b491e34` 原始输出（9 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/controller/IotTemplateUpgradeController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/dto/TemplateUpgradeResultVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/IotInstanceUpgradeService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/IotUpgradeApplyService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotInstanceUpgradeServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotUpgradeApplyServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotInstanceUpgradeServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotUpgradeApplyServiceImplTest.java
```

**比对结论：通过。** 9 文件全落 brief「改动范围」声明内：
- `iot/template/**` 主代码 5：新编排服务接口与实现（`IotUpgradeApplyService`/`IotUpgradeApplyServiceImpl`——`applyToVersion` 可复用接缝）、新逐台结果 VO（`TemplateUpgradeResultVO`）、既有确认服务接口与实现扩同事务接线与逐台结果返回、确认控制器扩响应面；
- `iot/dataitem/**` 主代码 1：`IotDataItemServiceImpl` 仅一处 `mapCalcPeriod` 可见性 `static→public static`（供编排服务复用 0013 换算范式，零语义改动，`git diff` 逐行实证）；
- `src/test/java/**` 测试 2：新编排服务测试 9 用例 + 既有确认服务测试补接线/边界 4 用例（原 19→21，既有 17 例适配 List 返回值）；
- **publish/snapshot 组装器零改动、无 SQL、无 yml、无 ot-core、无前端、无 ingest、无 calc**（`git diff --name-only 09357d1..b491e34 | grep -E "publish|resources/sql|\.yml|ot-core|frontend|ingest|calc"` 输出 NONE，逐字核对如上）；
- D-1 发布自动跟随已拆 TASK-0083（前置=0081）——本批**未实现发布侧接线**，仅留 `applyToVersion` 接缝（服务接口 javadoc 明示）。

---

## 二、每条验收命令原始输出

### 验收 1：单测全量（BUILD SUCCESS、0 失败 0 错误）

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`（最终提交 b491e34 上完整重跑）。

ot-core install 段：

```
[INFO] BUILD SUCCESS
[INFO] Total time:  3.852 s
```

ot-admin 测试 Results 段（全量日志尾段原样）：

```
[INFO] Results:
[INFO] 
[INFO] Tests run: 2384, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:09 min
[INFO] Finished at: 2026-11-23T23:42:31+08:00
[INFO] ------------------------------------------------------------------------
```

**通过**：2384 通过 / 0 失败 / 0 错误 / 0 跳过（开工基线 2373，本任务净增 11：新编排服务 9 + 确认服务接线/并发边界 2；既有确认服务 17 例 + 控制器 5 例零回归）。既有的发布/快照组装测试全绿——发布面零改动实证。

**TDD 红转绿留痕**（brief 测试口径）：先行落 `IotUpgradeApplyServiceImplTest` 9 用例，服务层以 `UnsupportedOperationException("TASK-0081 实现待落地")` stub 阶段首跑红（依赖字段先声明，红落在行为面）：

```
[ERROR] Tests run: 9, Failures: 0, Errors: 9, Skipped: 0, Time elapsed: 2.953 s <<< FAILURE! -- in ...IotUpgradeApplyServiceImplTest
（surefire 报告逐例：java.lang.UnsupportedOperationException: TASK-0081 实现待落地 ×9）
```

（首轮发生过两处接线缺失型 setUp 红——`Could not find field 'modelTemplateMapper'/'dataItemTemplateMapper'`，补 stub 字段后红落在上述行为面，同 0015/0016 红取证口径。）实现交付后目标 4 测试类全绿：

```
[INFO] Tests run: 5,  Failures: 0, Errors: 0, Skipped: 0 -- in ...controller.IotTemplateUpgradeControllerTest
[INFO] Tests run: 21, Failures: 0, Errors: 0, Skipped: 0 -- in ...service.impl.IotInstanceUpgradeServiceImplTest
[INFO] Tests run: 9,  Failures: 0, Errors: 0, Skipped: 0 -- in ...service.impl.IotUpgradeApplyServiceImplTest
[INFO] Tests run: 35, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

**每个修复附「为什么现在对了」**：
1. **测试夹具公式形态错**：实例行公式写 `"EQ.A.ELEC / 100"`（带空格）而 0013 展开产物为 `"EQ.A.ELEC/100"`（裸键紧邻算子）——三方合并旧默认展开比对误判为已覆盖。为什么现在对了：夹具对齐 0013 展开形态后，未覆盖判定如实命中「实例值=旧默认实例态」，写新默认分支按预期执行。
2. **版本 content mock 脱敏**：standalone MP 下 `LambdaQueryWrapper.getParamNameValuePairs()` 为空（params 脱敏），按 version 参数区分 applied/target 两行的桩失效。为什么现在对了：改为实现固定调用序（实现先查 applied 再查 target 两次连续调用）按序返回——两行 content 如实进入 diff 引擎，9 用例全绿。
3. **批量测试 fixture 用了同 content**：`applyToVersionBatch` 用相同 applied/target content → diff 空零操作提前返回，未走到 `selectList`。为什么现在对了：夹具改真实变更 content（`/100`→`/1000`），两台均执行数据面应用，调用序断言如实命中。
4. **确认服务测试桩缺新依赖**：`IotInstanceUpgradeServiceImpl` 注入 `upgradeApplyService` 后 3 例确认用例 `NullPointerException`。为什么现在对了：测试补 `@Mock IotUpgradeApplyService` + 默认返回零操作 VO 桩，17 既有语义零回归；另补 2 例接线断言（InOrder 验 apply 先于簿记 UPDATE、乐观锁冲突时 apply 已调用→整批回滚语义）。
5. **复合变更引用边陈旧**（最终代码加固）：`defaultFormula` 与 `defaultSourceType` 同批变更且新取值方式=直绑/汇总时，公式字段分支已收集新引用、结构迁移分支又清公式——可能重建出无公式的陈旧出边。为什么现在对了：引入 `formulaCleared` 标志，清公式时清空 `newRefs` 且边重建改「只删不插」——与 0013/0036「先删后插」在清退场场景的「先删不插」语义一致。
6. **格式化瑕疵**：早期 `public static int mapCalcPeriod(...) {        if (...)` 同行（可编译但不规范）。为什么现在对了：拆回两行标准格式，`git diff` 逐行实证仅 `static→public static` 一处可见性变化，零语义改动。

### 验收 2：dev 冒烟全序列（每步原始输出）

dev 档启动（`mvn spring-boot:run` 默认 dev，8098 启动前探测空闲）：

```
2026-11-23 23:29:17.495 [main] INFO  org.springframework.boot.tomcat.TomcatWebServer - Tomcat started on port 8098 (http) with context path '/'
2026-11-23 23:29:17.514 [main] INFO  net.otlab.admin.OtAdminApplication - Started OtAdminApplication in 12.871 seconds (process running for 13.254)
登录：POST /auth/login/test {"username":"admin","password":"admin123","tenantCode":"default"}
{"code":200,"data":{"userId":1,"username":"admin","token":"1b91e64d-...","nickname":"管理员",...},"message":"操作成功"}
```

启动段 0 ERROR；全程 6 条 ERROR 均为负例探针/边界测试的预期业务拒绝（GlobalExceptionHandler 如实留痕，见文末 ERROR 清单），非系统缺陷。

**STEP1-2 建链（模板 v2 + 两实例 applied=2）**

```
STEP1 POST /iot/template {"templateCode":"SMOKE0081_TPL","templateName":"Smoke0081","nodeType":4,"status":1}
{"code":200,"message":"操作成功"}   —— 模板 id=12 version=1
STEP2 POST /iot/template/item（ELEC_CONS 直绑 MINUTE + LOAD_RATE 公式 "{SELF.ELEC_CONS}/100" MINUTE alarm 10/90）
{"code":200,"data":2,"message":"操作成功"}
DB 核对：
SELECT id,template_id,version,tier_summary FROM iot_model_template_version WHERE template_id=12 ORDER BY version
12  12  1  null
13  12  2  T1:2      ← 新增两数据项自动升版（一档×2）
STEP2b 两实例（parentId=8, nodeType=4, templateId=12, deptId=4）：
POST /iot/instance shortCode=E8A/E8B → {"code":200} ×2 → 实例 26/27
SELECT id,instance_code,template_id,applied_template_version,locked,status FROM iot_model_instance WHERE template_id=12
26  F01.WS13.E8A  12  2  0  1
27  F01.WS13.E8B  12  2  0  1   ← applied=模板当前 version
SELECT id,instance_id,data_code,source_type,formula,calc_period,alarm_low,alarm_high FROM iot_data_item WHERE instance_id IN (26,27)
34  26  ELEC_CONS  1  null                        60  null      null
35  26  LOAD_RATE  2  F01.WS13.E8A.ELEC_CONS/100  60  10.000000 90.000000
36  27  ELEC_CONS  1  null                        60  null      null
37  27  LOAD_RATE  2  F01.WS13.E8B.ELEC_CONS/100  60  10.000000 90.000000
   ← 0013 生成范式实证：相对引用展开为实例唯一键裸键、mapCalcPeriod MINUTE→60、告警默认落行
```

**STEP3：T2 变更——改 LOAD_RATE 默认公式（item 21，/100 → /1000）**

```
PUT /iot/template/item {"id":21,"templateId":12,"dataCode":"LOAD_RATE",...,"defaultFormula":"{SELF.ELEC_CONS}/1000",...}
{"code":200,"message":"操作成功"}
DB：SELECT id,version,tier_summary FROM iot_model_template_version WHERE template_id=12 ORDER BY version
14  12  3  T2:1     ← 模板 version 2→3
```

**STEP3b：确认前发布取 snapshotHash（基线）**

```
POST /iot/publish/publish {"scopeType":2,"scopeId":0}
{"code":200,"data":{"id":44,...,"version":8,
 "snapshotHash":"ad7e64714056a25051157217969a817885887ed64f8b626bdffa84d587ea69a5",
 "snapshotUri":"1/2-0-8.json","publishStatus":1,...}}
```

**STEP4：待升级清单（确认前）**

```
GET /iot/template/upgrade/list?templateId=12
{"code":200,"data":{"total":2,"records":[
 {"instanceId":26,"instanceCode":"F01.WS13.E8A","appliedTemplateVersion":2,"currentVersion":3,
  "tierSummary":"T2:1","diff":[{"objectType":"dataItem","objectKey":"LOAD_RATE","changeType":"CHANGED",
   "field":"defaultFormula","oldValue":"{SELF.ELEC_CONS}/100","newValue":"{SELF.ELEC_CONS}/1000","tier":"T2"}]},
 {"instanceId":27,"instanceCode":"F01.WS13.E8B",... 同上 ...}
]}}
```

**STEP5：确认升级 E8A（id=26）→ 逐台结果 + 数据面实证**

```
POST /iot/template/upgrade/confirm {"instanceIds":[26],"targetVersion":3}
{"code":200,"data":[{
 "actions":["LOAD_RATE 公式继承新默认: F01.WS13.E8A.ELEC_CONS/1000"],
 "appliedVersion":2,"targetVersion":3,"tierSummary":"T2:1",
 "diff":[{"objectType":"dataItem","objectKey":"LOAD_RATE","changeType":"CHANGED",
   "field":"defaultFormula","oldValue":"{SELF.ELEC_CONS}/100","newValue":"{SELF.ELEC_CONS}/1000","tier":"T2"}],
 "instanceCode":"F01.WS13.E8A","instanceId":26,"rowUpdated":true
}],"message":"操作成功"}
DB 确认后：
26  F01.WS13.E8A  applied=3    27  F01.WS13.E8B  applied=2    ← E8B 保持旧口径
35  26  LOAD_RATE  F01.WS13.E8A.ELEC_CONS/1000   ← 行态已重写（三方合并写新默认）
37  27  LOAD_RATE  F01.WS13.E8B.ELEC_CONS/100    ← 未确认实例保持旧值
服务日志：[UPGRADE-APPLY] instanceId=26 templateId=12 applied=2 target=3 summary=T2:1
          actions=[LOAD_RATE 公式继承新默认: F01.WS13.E8A.ELEC_CONS/1000]
          [INSTANCE-UPGRADE] confirmed instanceId=26 ... applied 2 -> 3 rowUpdated=true
```

**STEP6：确认后发布取 snapshotHash（数据面生效实证）**

```
POST /iot/publish/publish {"scopeType":2,"scopeId":0}（无 token）
{"code":5002,"message":"发布未通过确认门：存在待确认的第二档模板变更..."}   ← E8B 仍滞后，确认门 fail-closed（0015 既有语义零回归）
GET /iot/template/impact-preview?templateId=12
{"code":200,"data":{"previewToken":"5f0935feefcb4d9caea3a12a5b28b33e","targetVersion":9,
 "affectedInstanceCount":2,"diff":[...T2...],"tierSummary":"T2:1"}}
POST /iot/publish/publish {"scopeType":2,"scopeId":0,"confirmToken":"5f0935feefcb4d9caea3a12a5b28b33e"}
{"code":200,"data":{"id":45,...,"version":9,
 "snapshotHash":"64fdb2167bb37504f4fc4ef9250f04764d8da5c4350c61c0d2f4ad7cdffc177e",
 "snapshotUri":"1/2-0-9.json","publishStatus":1,...}}
=== 哈希比对 ===
确认前 ad7e64714056a25051157217969a817885887ed64f8b626bdffa84d587ea69a5 (v8)
确认后 64fdb2167bb37504f4fc4ef9250f04764d8da5c4350c61c0d2f4ad7cdffc177e (v9)  ← 变化，数据面生效实证
快照体逐字段实证（backend/ot-admin/data/snapshots/1/2-0-9.json）：
  grep -o "F01.WS13.E8[AB].ELEC_CONS/[0-9]*"  →  1× F01.WS13.E8A.ELEC_CONS/1000
                                               1× F01.WS13.E8B.ELEC_CONS/100
  ← 组装体读实例行裁决①实证：已确认实例新值 + 未确认实例旧值同快照并存
```

**STEP7：锁定实例保持旧口径且仍在清单**

```
GET /iot/template/upgrade/list?templateId=12 → total=1
F01.WS13.E8B applied=2 tier=T2:1   ← 未确认（含锁定语义）实例仍在清单
DB：37  27  LOAD_RATE  F01.WS13.E8B.ELEC_CONS/100  ← 保持旧口径
```

**STEP8：T1 补插 + T3 逐台迁移 diff 与结果**

```
=== T1 补插（新增 PARAM_C → 模板 v4）===
POST /iot/template/item（PARAM_C 直绑 MINUTE）→ {"code":200,"data":1} → 版本行 15 | T1:1
POST /iot/template/upgrade/confirm {"instanceIds":[26,27],"targetVersion":4}
{"code":200,"data":[
 {"instanceId":26,"appliedVersion":3,"targetVersion":4,"tierSummary":"T1:1","rowUpdated":true,
  "actions":["新增项补插: PARAM_C（0013 生成范式）"],"diff":[{objectType:dataItem,objectKey:PARAM_C,changeType:ADDED,tier:T1}]},
 {"instanceId":27,"appliedVersion":2,"targetVersion":4,"tierSummary":"T1:1,T2:1","rowUpdated":true,
  "actions":["新增项补插: PARAM_C（0013 生成范式）","LOAD_RATE 公式继承新默认: F01.WS13.E8B.ELEC_CONS/1000"],
  "diff":[{LOAD_RATE CHANGED.defaultFormula T2},{PARAM_C ADDED T1}]}
]}
DB：38  26  PARAM_C  deleted=0   39  27  PARAM_C  deleted=0   ← 0013 生成范式补插落行

=== T3 删项（DELETE 模板 item 22 = PARAM_C → 模板 v5）===
DELETE /iot/template/item/22 → {"code":200} → 版本行 16 | T3:1
GET /iot/template/upgrade/list?templateId=12 → 两实例均："tierSummary":"T3:1",
  diff=[{objectType:dataItem,objectKey:PARAM_C,changeType:REMOVED,tier:T3}]
POST /iot/template/upgrade/confirm {"instanceIds":[26],"targetVersion":5}
{"code":200,"data":[{"instanceId":26,"appliedVersion":4,"targetVersion":5,"tierSummary":"T3:1",
 "actions":["PARAM_C 逻辑删停用（键永久占用，历史行保留；入边与指标引用不清理）"],
 "diff":[{objectType:dataItem,objectKey:PARAM_C,changeType:REMOVED,tier:T3}],"rowUpdated":true}]}
DB 核对（含已删行）：
38  26  PARAM_C  deleted=1   ← 逻辑删（@TableLogic 退出快照，历史行自身保留）
39  27  PARAM_C  deleted=0   ← 未确认实例不受影响
SELECT id,instance_id,data_code FROM iot_data_item WHERE instance_id=26 AND deleted=0 → 仅 34(ELEC_CONS)/35(LOAD_RATE)
```

**STEP9：四类边界断言**

```
边界1 空：POST confirm {"instanceIds":[],"targetVersion":5}
  → {"code":400,"message":"请选择要确认升级的实例"}  （零操作拒绝，empty=400 防线；apply 层「diff 空=零操作」
     由单测 emptyDiffZeroOperations 断言行/边/regenerate 零调用）
边界2 越权：单测 confirmPermDenied —— 受限用户批内任一不可见 → 403 且零更新（0016 防线并入后不变）
边界3 并发/重复：POST confirm {"instanceIds":[26],"targetVersion":5}（已在目标版本）
  → {"code":4002,"message":"实例已在目标版本（并发确认或重复提交），请刷新待升级清单后重试: F01.WS13.E8A"}
  （同事务整批拒绝；单测 confirmDataPlaneAppliedButOptimisticConflict 断言 apply 已调用但 UPDATE affected=0 → 4002 整批回滚）
边界4 超限/无效批：POST confirm {"instanceIds":[26,999999],"targetVersion":5}
  → {"code":400,"message":"实例不存在或已删除: [999999]"}（整批拒绝零半态）
  POST confirm {"instanceIds":[26],"targetVersion":99}
  → {"code":400,"message":"目标版本必须等于模板当前版本 5（不允许任意目标版）"}
  （一次升版实例数分批口径由调用方 instanceIds[] 控制——超限靠分批，事务规模受控）
```

**验收 2 全序列通过。** dev 全程 6 条 ERROR 均为负例探针/边界测试的预期业务拒绝（租户不存在×1、确认门未确认发布×1、目标版本错×2、重复提交×1、无效 id×1），启动段 0 ERROR；逐台 `[UPGRADE-APPLY]` INFO 留痕 5 条实查在位。用毕 kill PID 94008 → 8098 监听归零。

### 验收 3：prod 档真实启动回归（0 ERROR）

启动（脚本 `/tmp/task0081_prod_run.sh` 注入本地连接与密钥，8098 启动前探测空闲）：

```
DB_HOST=127.0.0.1 DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root
REDIS_HOST=127.0.0.1 REDIS_PORT=6379 REDIS_DATABASE=0 SM4_KEY=0123456789abcdef
CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 WECHAT_MP_ENABLED=false
mvn spring-boot:run -Dspring-boot.run.profiles=prod
启动日志：
2026-11-23T23:36:46.656+08:00  INFO ... The following 1 profile is active: "prod"
2026-11-23T23:37:00.717+08:00  INFO ... Tomcat started on port 8098
2026-11-23T23:37:00.736+08:00  INFO ... Started OtAdminApplication in 14.628 seconds (process running for 15.095)
ERROR_COUNT=0 （`grep -ac " ERROR " /tmp/task0081_prod.log` = 0，含启动与全部探针）
```

prod 下认证走正式链：`GET /auth/captchaImage` → uuid=2fa598ca-... → redis-cli 读 `captcha:code:<uuid>`="JCNL" → hutool SM4 encryptHex(admin123)（BC provider 注册后，Base64 密文 `YmY0MGY0YjExNmM2NzFmODU5NDEwNmZhZWZkMDA5N2U=`）→ `POST /auth/login` 返回 `{"code":200,"data":{"token":"ee7526c5-...","username":"admin"}}`。

```
PROBE1 GET /iot/template/upgrade/list?templateId=12
{"code":200,"data":{"current":1,"pages":0,"records":[],"size":10,"total":0},...}  ← 新响应面 prod 生效（dev 冒烟已全量确认的延续状态）
PROBE2 GET /iot/publish/list?scopeType=2&scopeId=0&pageNum=1&pageSize=3
{"code":200,"data":{"pages":3,"records":[{"id":45,...,"version":9,
 "snapshotHash":"64fdb2167bb37504f4fc4ef9250f04764d8da5c4350c61c0d2f4ad7cdffc177e",...}]}}
  ← dev 冒烟确认后发布的 v9 快照哈希在 prod 可见（档位切换零差异实证）
PROBE3 GET /iot/template/impact-preview?templateId=12
{"code":200,"data":{"affectedInstanceCount":2,"currentTemplateVersion":5,"diff":[],
 "tierSummary":null,"previewToken":"f65b9a165e994c758e10e91365ca92bf","targetVersion":10,...}}
  ← 既有 0015 端点回归 200（全部实例已确认、diff 空语义自洽）
```

**通过**：prod 档真实启动 14.6s 全程 0 ERROR + 正式登录链 + 新端点（确认响应面）与既有端点（发布列表/影响预览）全 200。验毕 kill PID 68644 → 8098 监听确认归零。

### 验收 4：改动范围核对

见「一、改动文件清单」——`git diff --name-only 09357d1..b491e34` = 本任务提交 9 文件全部落在 brief 声明范围内（template/** 主代码 5 + dataitem/** 可见性 1 + test 2 + 上方未列第 9 为确认控制器——清单逐行核对）；`publish/snapshot` 组装器、SQL、yml、ot-core、前端、ingest、calc 零触碰（`grep -E "publish|resources/sql|\.yml|ot-core|frontend|ingest|calc"` 输出 NONE），**通过**。

---

## 三、未做项与原因

1. **发布自动跟随接线（D-1）**：用户已裁决拆出 TASK-0083（前置=0081），brief 明文「本任务不实现发布侧接线，仅留 applyToVersion 接缝」——本批只交付可复用方法面（服务接口 javadoc 明示供 0083 复用），未碰 `IotPublishServiceImpl`/`publish/**`。
2. **iot_data_item_history 写路径（0039）**：留痕过渡口径=响应逐台 VO + 逐台 INFO 日志 + @OperateLog 请求级；不建表、不写 history、不改 0014 NoOp 接缝——删项历史落点=逻辑删行自身（冒烟 STEP8 `deleted=1` 行含 full_key/全量值列/审计列，0039 交付后可零改动纳入时间线）。
3. **状态规则类 T2 生效面**：无实例行载体（0015 遗留 #3），本任务不经——diff 引擎对 statusRule 条目仍出 diff 明示，但 apply 层不写实例行（无对应列）。
4. **valueKind（数据类型）实例行落列**：0013「留模板不落实例」实证——实例表无 value_kind 列，T3 改类型经继承判定覆盖的公式/结构字段（source_type/formula/rollup）重物化承载，逐台 actions 明示（单测 t3SourceTypeChangeStructuralMigration 覆盖）。
5. **被引用入边与指标引用清理**：ADR-0008 明示后果——上级汇总少贡献者/指标来源断裂，清理他人公式属二次破坏性变更；本任务不清理，逐台 actions 字面明示「入边与指标引用不清理」（单测 t3LogicalDeleteRemovedItem 断言不删入口边）。
6. **前端逐台结果展示（0070）**：确认响应由计数 int 扩为逐台结果 VO 列表属增量契约变化，前端展示后续批次接入（VO javadoc 登记；旧调用方按 `results.size()` 可取台数，JSON 数组增量兼容）。
7. **TDengine**：按协调者指令本批不涉——全链路数据源 MySQL 业务表，冒烟与 prod 回归均未触；TDengine 标签污染防护经双 uk 不含 deleted 的键纪律兑现（逻辑删永久占用键）。
8. **越权 live 冒烟**：以单测 `confirmPermDenied`（受限用户 403 + 零更新）为口径——live 侧需受限账号建链（角色/部门/用户三链），brief 冒烟序列未含，依 0016 同款处理。

---

## 四、偏离计划之处

1. **`mapCalcPeriod` 可见性放宽**：brief「dataitem/** 仅复用 0013 生成/展开/边重建组件与逻辑删——不改组件语义」。实现将 `IotDataItemServiceImpl.mapCalcPeriod` 由包级 `static` 放宽为 `public static`（唯一一处改动，零语义/零行为变化），供 template 域编排服务复用同一换算范式——避免复制第二份换算逻辑导致漂移。`git diff` 逐行实证仅签名一处可见性变化；在报告「验收 4」明确登记。
2. **确认接口响应面从计数扩为逐台 VO 列表**：0016 既有契约为 `Result<Void>`（`data=null`），本批按 TASK 正文「逐台结果=响应逐台 VO」扩为 `Result<List<TemplateUpgradeResultVO>>`。属增量兼容变化（旧前端忽略 data 不破坏；台数可取 `data.size()`），控制器测试既有 5 例零回归——偏离 0016 响应形态但为 brief 明确要求，登记备查。
3. **T3 出边清理范围**：删项逻辑删同时清理被删项**自身出边**（其公式不再生效），但**入边与指标引用不清理**（ADR 明示）。brief 只写「入边与指标引用不清理」未明说自身出边——按「公式随项停用」的一致性推定清理自身出边；如需保留可移除 `dataItemRefMapper.deleteByItemIdPhysical(row.id)` 一行（单测同步）。
4. **三方合并「空=未覆盖」的阈值/名称字段**：除 brief 明列的公式/周期/取值方式外，对 `aggMethod`/`defaultValidMin/Max`/`defaultAlarmLow/High`/`dataName`/`category`/`unit` 同样按「行值为空或等于旧默认→写新默认」处理（均属 T2/T1 行承载字段，口径同构）——brief 列的是必测面，非穷举禁用面；已逐条记入 actions 便于逐台 diff 明示。
5. **并发/幂等语义沿用 0016**：applied==target 按 4002 整批拒绝（而非静默跳过）；apply 发生在簿记乐观锁 UPDATE 之前——若并发败者已执行 apply 后撞 4002，@Transactional 将 apply 的行变更一并回滚（单测 confirmDataPlaneAppliedButOptimisticConflict 实证），无半态。
6. **冒烟环境差异**：Windows Git Bash 的 curl 传中文字段产生 `Invalid UTF-8 middle byte 0xc3` 解析错——探针改 ASCII dataName 后全绿（应用侧零缺陷，仅探针调整）；prod 认证依 0015/0016 报告同款 SM4 正式链（redis-cli 位于 phpStudy 扩展目录，非 WSL docker）。
7. **提交粒度**：单提交 b491e34（实现+测试一体，红绿证据均在最终代码上复核）；`.agents/` 环境产物未入库（逐文件显式 add，未用 `git add -A`）。

---

## 五、遗留问题（含开工复核性实证结论）

1. **开工复核性实证结论**：四项（content 字段面/mapCalcPeriod+展开+null→1 兜底/双 uk 逻辑删/组装器读行不读 applied）**逐条与裁决一致，零偏差**——按 brief 继续开工，未触发停写报告条款。详细证据见「〇」节。
2. **D-1 发布自动跟随（TASK-0083）**：已拆独立任务、前置=0081；本任务 `applyToVersion` 接缝为其预留——0083 实现时需注意「自动跟随不得触碰含 T3 条目的滞后实例」（评审报告红线），以及 gate 窗口语径/发布事务规模三个新增风险面。
3. **冒烟数据留存库内**：模板 12（SMOKE0081_TPL）/实例 26-27（F01.WS13.E8A/E8B）/版本历史行 12-16（T1:2→T2:1→T1:1→T3:1）/发布 v8-v9（snapshotHash ad7e6471…→64fdb216…）/数据项行 34-39（PARAM_C 行 38 deleted=1、39 deleted=0）；对应快照文件 `backend/ot-admin/data/snapshots/1/2-0-8.json`、`2-0-9.json` 未清理（照 0015/0016 冒烟遗留同口径隔离，与 DB 行引用一致性优先）。重跑冒烟需换短码（同码撞占用属预期）。
4. **确认凭据漂移**：确认编排改写实例行使 0015 复合凭据哈希（计算域快照摘要）漂移——确认后旧 previewToken 失效须重取（冒烟 STEP6 5002→重预览→200 实证）；fail-closed 正确方向，无缺陷。
5. **三方合并误向**：「显式覆盖为旧默认同值」被判为未覆盖写新默认——保守方向（模板批量一致核心价值），TASK 正文裁决明示接受，登记备查。
6. **超限分批**：本任务未设硬阈值——一次升版实例数分批由调用方 `instanceIds[]` 控制（FEAT-0005 既有口径），超限场景靠分批而非任务内拒绝；如后续要求服务端硬上限须另立裁决（需改确认接口契约）。
7. **越权 live 未冒烟**：以单测为口径（见「三」8）；如需 live 验证须建受限账号三链。

---

## 六、一句话结论

**通过**——验收 1（最终提交 b491e34 全量 mvn：**2384 测试 0 失败 0 错误 BUILD SUCCESS**，TDD 红 9 例 `UnsupportedOperationException`→绿 35 目标测试，6 项修复均附根因）、验收 2（dev 冒烟全序列：模板 v2→T2 改公式 v3/T2:1→**确认前发布 snapshotHash=ad7e6471…(v8)**→清单 2 行 T2 diff→确认 E8A 逐台结果 actions/rowUpdated=true 且行公式 `/100`→`/1000`→**确认后发布无 token 5002（E8B 仍滞后，确认门 fail-closed）→重预览带 token 发布 snapshotHash=64fdb216…(v9)，与确认前不同即数据面生效实证**，快照体同现 E8A 新值+E8B 旧值→E8B 未确认保持旧公式仍在清单→T1 补插 PARAM_C 两实例落行→T3 删项 diff REMOVED/T3:1+逻辑删 `deleted=1` 留历史行→四类边界逐条断言）、验收 3（prod 档启动 14.6s 全程 **0 ERROR** + captcha/SM4 正式登录链 + 确认响应面/发布列表/影响预览全 200）、验收 4（净足迹 9 文件全落 brief 声明范围，publish/snapshot 组装器/SQL/yml/ot-core/前端/ingest/calc 零触碰，mapCalcPeriod 仅可见性放宽逐行实证）四条全部以原始输出为证通过；开工复核性实证四项与裁决一致零偏差；偏离均登记，遗留仅登记级与已拆 TASK-0083（发布自动跟随），无阻断缺陷。

---

---

# 修订轮报告（TASK-0081 diff 评审 I-1 退回 + m-2 顺手修 + m-1 登记 deferred）

> 修订执行者：TASK-0081 执行者；修订日期：2026-11-24；工作区 `D:\workspace\WisFactory`（main 分支）。
> 输入：`TASK-0081-review.md`（I-1 完整论证 + 越界反查复核）、协调者修订派发（§12/§13）。
> 修订 BASE = **b491e34**（开工时 `git rev-parse --short HEAD`，原任务实现提交）；修订提交 = **5758d5f**（2 文件，87 行增 / 7 行删）。
> 契约：no-subagents；修订授权范围 = 原 brief 范围（template/dataitem 域）；组装器/SQL/yml/ot-core/前端/ingest/calc 零触碰；不把 TASK 置「已完成」；workitems 库仅追加本报告；提交注明 TASK-0081 修订轮。
> 复评为 scoped：仅看 I-1 修复 diff 与 m-1/m-2 是否采纳，不重开全场。

## 一、改动文件清单（修订轮 2 文件）

`git diff --name-only b491e34..5758d5f` 原始输出：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotUpgradeApplyServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotUpgradeApplyServiceImplTest.java
```

范围核对：2 文件全落原 brief 声明的 `template/**` 主代码 + `src/test/**`；`git diff --name-only | grep -E "publish|resources/sql|\.yml|ot-core|frontend|ingest|calc"` 输出 NONE；原实现 9 文件足迹不变，组装器/SQL/yml/ot-core 仍零触碰，**通过**。

## 二、修复内容与「为什么现在对了」（逐条 TDD 红→绿）

### 1. I-1（Important，必须修）——新默认非法 `defaultCalcPeriod` 静默写 0 → 改 fail-closed 整批回滚

- **根因**：原 `safeMapPeriod`（:525-531）吞掉 `mapCalcPeriod` 的异常返回 null，T2 分支对新默认 `newSeconds == null` 直接 `patch.setCalcPeriod(0)`——正是 0013 `mapCalcPeriod` 注释明令禁止的「配了周期却被当成到达驱动」语义漂移；模板项 `defaultCalcPeriod` 仅受 `@Size(max=16)` 无格式校验，`WEEKLY`/`MINUT` 可入库并经 `buildContentJson` 写入版本 content，确认后实例 `calc_period` 由 60 静默变 0。0013 生成链同类坏码整批失败，升级链静默降级——同源换算范式行为不一致。
- **修法**：新默认改走新增的 `requireMapPeriod`（直接调 `IotDataItemServiceImpl.mapCalcPeriod`，非法码抛 `BusinessException(PARAM_ERROR)`）——由调用方 `confirmUpgrade` 的 `@Transactional(rollbackFor=Exception.class)` 整批回滚；旧默认仍用 `safeMapPeriod` 宽容（返回 null → 实例值非 null 视为已覆盖保留，不阻断历史升级）。
- **为什么现在对了**：新默认坏码与 0013 生成链同 fail-closed 纪律——非法周期码在写库前即抛异常、整批回滚（含本台已 apply 的行变更与其他台簿记），实例行绝不会被写入 0；旧默认坏码保持宽容，历史版本含坏码的实例仍能升级到合法新默认（旧默认 null → 实例现值不等于 null → 用户覆盖保留，安全方向）。

**红（`t2NewDefaultCalcPeriodBadCodeFailsClosed`，新默认 WEEKLY）**：

```
[ERROR] Tests run: 12, Failures: 2, Errors: 0, Skipped: 0
IotUpgradeApplyServiceImplTest.t2NewDefaultCalcPeriodBadCodeFailsClosed:198
  新默认周期坏码必须抛异常（整批回滚），不得静默写 0 降级为到达驱动
  ==> Expected java.lang.RuntimeException to be thrown, but nothing was thrown.
```

**绿**：同用例通过（12/12，见 §三）。附带旧默认宽容用例 `t2OldDefaultCalcPeriodBadCodeTolerant`（旧 WEEKLY/新 HOUR/实例现值 3600 → 保留不写、零 UPDATE）同轮新增并绿——证明「旧默认宽容」边界未被误伤。

### 2. m-2（Minor，顺手修）——T1 补插 VO 无条件报成功 → 核对 `regenerateForInstance` 返回值

- **根因**：原代码 `addedCodes` 非空即调用 regenerate 并写死 `rowUpdated=true` + 「新增项补插」文案；`regenerateForInstance` 按占用幂等跳过（含逻辑删行永久占键，`generateForInstance` :215-227），新增 data_code 被已删行占用时实际返回 0、不插行，逐台 VO 却报补插成功——数据正确性无损（占用跳过是裁决明示设计），但留痕失真。
- **修法**（<10 行）：`int inserted = dataItemService.regenerateForInstance(...)`；`inserted > 0` 才置 `rowUpdated=true` 且动作文案带实际行数；`inserted == 0` 时文案改「新增项占用跳过未补插…（键已占用含逻辑删行，regenerate 返回 0）」，`rowUpdated` 保持 false。
- **为什么现在对了**：逐台结果的 rowUpdated/actions 如实反映实际行变更——复评或 0070 前端展示不再把「占用跳过」误读为补插成功；0013 幂等占用语义不变（跳过仍发生），仅留痕面精确化。

**红（`t1InsertOccupiedSkippedNoFalseSuccess`，regenerate 桩返回 0）**：

```
[ERROR] IotUpgradeApplyServiceImplTest.t1InsertOccupiedSkippedNoFalseSuccess:283
  regenerate 返回 0——无实际行变更，不误报成功 ==> expected: <false> but was: <true>
```

**绿**：同用例通过（12/12），断言动作文案含「占用跳过/未补插」。

### 3. m-1（Minor，登记 deferred 不修）——`applyToVersion` 经 regenerate 间接依赖会话/权限上下文

- **事实**：`IotUpgradeApplyServiceImpl:153` 调 `dataItemService.regenerateForInstance`，其内部 `dataPermService.resolve(SecurityUtils.getUserId())`（`IotDataItemServiceImpl:365-367`）依赖当前登录用户 + `TenantContext.getTenantId()`。
- **不修理由**（协调派发明示「登记 deferred 不修」）：确认链路已同用户鉴权，当前无缺陷；0083 未实现、不在本轮范围，是否致害无法判定。为接缝去耦合需改 regenerate 语义或新增免会话补插入口（超出授权范围 + 可能触 0013 语义），属 0083 设计阶段的明确核实项。
- **交接给 0083**：设计发布自动跟随链路时须显式核实「发布侧无用户会话/权限模型不同」场景——若触发 403/NPE，为接缝提供免会话补插入口或显式注入上下文；本轮不改代码。

## 三、验收命令原始输出（修订轮口径 = 全量 mvn Results 段）

```
cd backend && mvn install -pl ot-core -DskipTests    → ===CORE-INSTALL-DONE===（EXIT=0）
cd backend && mvn test -pl ot-admin                  → Results 段原始输出：

[INFO] Results:
[INFO] 
[INFO] Tests run: 2387, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:24 min
[INFO] Finished at: 2026-11-24T00:05:51+08:00
[INFO] ------------------------------------------------------------------------
```

2387 = 原基线 2384 + 修订轮净增 3（I-1 两例：新默认坏码 fail-closed / 旧默认宽容；m-2 一例：占用跳过不误报）。既有 2384 例零回归（含原任务 9 例编排测试、确认/预览/发布/快照组装既有链全绿）。

目标测试类明细（surefire 实测）：

```
IotUpgradeApplyServiceImplTest   tests="12" errors="0" failures="0"   （原 9 + 修订 3）
```

## 四、未做项与原因

1. **m-1 会话耦合**：见 §二.3，登记 deferred 供 TASK-0083 复用时评估，本轮不改。
2. **dev/prod 冒烟未重跑**：修订轮契约验收口径 = 全量 mvn Results 段（未列冒烟）；I-1/m-2 修复均为确定性单元行为（坏码抛异常/返回值消费），改动面限单类两处，集成面无新增分歧点；原任务 dev 冒烟 v8→v9 snapshotHash 变化实证与本修订无交集（修订不改发布链）。如 scoped 复评需要 live 复验，可按原报告「验收 2」环境口径构造模板 `defaultCalcPeriod=WEEKLY` → 确认升级应整批 400 回滚（新增负向 live 序列）。
3. **TASK 状态**：保持「进行中」未回写（协调者收尾）。

## 五、偏离计划之处

1. **I-1 异常类型选择**：`requireMapPeriod` 直接传播 `mapCalcPeriod` 抛出的 `BusinessException(PARAM_ERROR=400)`（由 `paramErrorStatic` 产生）——与 0013 生成链同类坏码的响应码一致；评审建议「抛异常整批回滚」未指定码位，取同源范式 400 而非自造新码（ot-core 零触碰约束下亦无需扩枚举）。
2. **m-2 文案粒度**：修订轮新增「实际 N 行」到补插成功文案（原为固定串）——纯日志/VO 文案增量，前端按 `rowUpdated`+`actions` 展示时不受影响。
3. **修订提交粒度**：单提交 5758d5f（实现+测试一体，红绿证据在最终代码上复核）；`.agents/` 环境产物未入库（逐文件显式 add）。

## 六、修订轮一句话结论

**通过（修订轮）**——I-1（新默认非法 calcPeriod 静默写 0）以「新默认 `requireMapPeriod` fail-closed 抛异常整批回滚 / 旧默认 `safeMapPeriod` 宽容」闭合（红：WEEKLY 坏码未抛异常→绿：抛 400 + 零 UPDATE，旧默认宽容用例同绿）；m-2 顺手修 T1 补插留痕核对 regenerate 返回值（红：返回 0 仍报 rowUpdated=true→绿：如实报占用跳过）；m-1 会话耦合登记 deferred 供 TASK-0083 评估；全量 mvn **2387 测试 0 失败 0 错误 BUILD SUCCESS**（原基线 2384 + 3 新用例，原文 Results 段见 §三）；修订足迹 2 文件全落原 brief 范围，组装器/SQL/yml/ot-core/前端/ingest/calc 零触碰。待协调者 scoped 复评。
