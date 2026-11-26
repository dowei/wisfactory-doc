# TASK-0022 + TASK-0023 实施报告：点位导入导出 + 反查/批量启停/曲线占位（微改攒批合并报告）

执行者：implementer（no-subagents 单体会话，两任务同批实施统一测一次——brief 微改攒批口径）
开工时间：2026-11-22 13:30 前后；报告完成：2026-11-22 15:30 前后
BASE：a961fd1（主仓 HEAD，派发时实测一致）
本批提交（main，两任务分离提交便于追溯）：
- **963dc71** feat(iot) 点位批量导入导出（**TASK-0022**）
- **bfdc11b** feat(iot) 点位详情反查/批量启停响应体/近 24h 曲线 fail-soft 占位（**TASK-0023**）

**四条验收全部通过**（原始输出见 §2）：全量回归 1863 = brief 基线 1822 + 本批净增 41，0 失败 0 错误 BUILD SUCCESS；dev 冒烟 S1-S13 连续全绿（含配额联动不豁免 4003、幂等整批拒、导出逐行比对、删通道拒提示编码、反查三要素、停用影响计数 + 重新发布明示、受限用户整批 403、曲线 fail-soft 空集 200）；prod 档真实启动 10.765s 全程 **ERROR=0** + 真实登录 200 + point/list 200 + 详情聚合/曲线端点探针 200；本批 23 文件全落声明范围。TASK 状态维持「进行中」未回写（协调者收尾）。

---

## 1. 改动文件清单（a961fd1..HEAD，23 文件；commit1=13 / commit2=14）

### 1.1 主码（16，全在 point/**——brief 声明范围 1）

| 文件 | 任务 | 性质 |
| --- | --- | --- |
| `point/dto/PointImportRow.java` | 0022 | 新增——导入行 DTO（十三列 + 行号 + isSkippable 说明行/空行判定，照 0018 DictImportRow 同构） |
| `point/dto/PointImportRowResult.java` | 0022 | 新增——逐行回执（11 个点位域原因码：必填/超长/编码合规/枚举×2/数值列/量程/单位字典/通道不存在/通道歧义/重复编码） |
| `point/dto/PointImportReceipt.java` | 0022 | 新增——导入回执（total/success/fail/persisted/rows，配额按校验通过行数联动） |
| `point/dto/PointImportRejectException.java` | 0022 | 新增——整批拒绝异常（PARAM_ERROR 400 + data 携带回执，4002 留单点） |
| `point/dto/PointImportRowLimitException.java` | 0022 | 新增——10000 行 fail fast 流式中止异常 |
| `point/dto/PointExportRow.java` | 0022 | 新增——导出行（与模板十三列同构，channelCode 回填） |
| `point/service/support/PointImportExcelSaxParser.java` | 0022 | 新增——SAX 流式解析器（0009/0018 修订轮三要点继承：r 属性物理行号保真/缺格 null 保真/超限 fail fast，十三列点位域映射） |
| `point/service/IotPointService.java` | 0022/0023 | 修改——IMPORT_MAX_ROWS=10000 常量 + importPoints/selectExportRows（0022）+ selectPointReferences/getPointCurve24h + 详情聚合/启停响应签名（0023） |
| `point/service/impl/IotPointServiceImpl.java` | 0022/0023 | 修改——导入两遍校验（字段级 + 库内级一次 IN：unit 字典批查/channelCode 解析歧义拒/编码占用旁路预检含已删行）+ 配额联动不豁免 + 白名单落库不发事件；导出与列表同源 buildListWrapper；三读公共闸 requireReadablePoint + 反查/详情聚合/曲线 + 启停响应体 |
| `point/controller/IotPointController.java` | 0022/0023 | 修改——import/template/error-report/export 四端点 + TEMPLATE_NOTE（0022）；references/curve 端点 + changeStatus 响应体与 /status 别名路由（0023） |
| `point/dto/IotPointStatusChangeVO.java` | 0023 | 新增——启停响应（targetStatus/changedCount/affectedDataItemCount/effectNote「变更已保存，重新发布后对接入生效」） |
| `point/dto/PointReferenceVO.java` | 0023 | 新增——反查行（itemId/fullKey/instanceCode/dataName） |
| `point/dto/IotPointDetailVO.java` | 0023 | 新增——详情聚合（继承 IotPoint + referenceCount + curve24h） |
| `point/dto/IotPointCurveVO.java` / `PointCurvePointVO.java` | 0023 | 新增——曲线响应/数据点（ts 原文透传 + value/quality） |
| `point/service/support/PointCurveQueryClient.java` | 0023 | 新增——st_point_num 全限定名查询客户端（0003 HttpTdengineAdmin 形态：base-url 复用 TdengineProperties 只读注入、Basic 认证、fail-soft 空集不抛异常、编码白名单防注入、LIMIT 5000 保护上限） |

### 1.2 测试（7，全在 test/iot/**——brief 声明范围 3）

| 文件 | 任务 | 性质 |
| --- | --- | --- |
| `point/service/impl/IotPointImportServiceTest.java` | 0022 | 新增 18 用例（回执/幂等/配额边界/10000 上限/并发兜底/字典批查一次 IN/通道歧义/字段白名单/不发事件） |
| `point/controller/IotPointImportExportControllerTest.java` | 0022 | 新增 9 用例（真实 xlsx 解析行号保真/整批拒 400 透传/10000+1 fail fast/模板结构/错误清单渲染/导出同构/注解） |
| `point/service/impl/IotPointServiceImplTest.java` | 0022/0023 | 修改——+12 用例（导出裁剪同源 2/反查 3/详情聚合 1/曲线 3/启停响应体与影响计数 3）+ 夹具接线 |
| `point/controller/IotPointControllerTest.java` | 0023 | 修改——detail/changeStatus 用例随 VO 契约对齐（+/status 别名路由断言） |
| `point/service/support/PointCurveQueryClientTest.java` | 0023 | 新增 4 用例（JDK HttpServer 桩 taosAdapter：全限定名 SQL 断言/列序解析/非 0 code/连接拒绝/编码白名单短路） |
| `redline/RedlineFixture.java` | 0023 | 修改——点位服务接线补 pointQuotaCounter/曲线客户端 fail-soft 桩 + itemMapper 补 selectCount 桩与 point_id 条件模拟器分支 |
| `redline/PointRedlineTest.java` | 0023 | 修改——changeStatus 用例随 VO 签名对齐（断言 changedCount/effectNote） |

净增用例 = 18+9+12+4 = **+43 个方法级用例**（其余为既有用例随签名对齐的改写，不新增计数）；全量回归实测 1863 − brief 基线 1822 = **+41**（IotPointControllerTest 两用例为原地改写不计净增，差异 2 条即此口径，明细见 §3 偏离 8）。

**未触碰**：frontend/、ingest/、模板/实例/字典域主码、dataperm 组件本体、channel/service/**（见 §3 偏离 2——转正已在 BASE 就位）、workitems 库（本报告除外）、TASK 详情。

---

## 2. 验收命令原始输出

### 2.1 验收 1：全量回归 —— 通过

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[INFO] ot-core ................................ SUCCESS (总时间 2.005 s，install -DskipTests)
[WARNING] Tests run: 1863, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

基线口径：brief 给定 1822（本批开工前主仓基线）；1863 = 1822 + 41（净增明细：IotPointImportServiceTest 18 + IotPointImportExportControllerTest 9 + PointCurveQueryClientTest 4 + IotPointServiceImplTest 净增 10——其中 2 条既有用例原地改写不计，见 §3 偏离 8）；3 skipped 为红线二-中台 scope 占位（TASK-0051 接线后启用，基线同值）。

红转绿过程（TDD 记录，两处测试自身缺陷定位为夹具问题后修正）：① 导入可空数值列首版被误判 INVALID_NUMBER（服务层把「未携带」当「携带但非法」）——定位为服务层校验分支缺陷，修为「仅携带但非法才拒」转绿（「为什么现在对了」：可空列语义与单点建档 DDL 可空口径一致，未携带放行、携带非法拒）；② 批内撞码用例首版 `verify(never())` 占用预检断言错误——撞码行放行进入占用预检是刻意保留的并发兜底，改断言后转绿。

### 2.2 验收 2：dev 冒烟（两任务串一起）—— 通过

启动：dev.ps1（8098，dev 档，含本批两提交的 jar）。冒烟脚本 `backend/target/t0022_smoke.py` 最终轮（连续单轮，EXIT=0）原始输出（openpyxl 样式告警行略）：

```
=== S1) admin dev login === S1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S1 token= 1d013d91-4...
    PUT /iot/tenant pointQuota=None -> HTTP 200 code 200 | 操作成功  [期望200] OK
=== S2) 建网关+通道（MQTT，导入文件 channelCode 指向它） ===
S2a gateway -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2b channel -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2 断言: 网关 id=9105 通道 id=15 建立完成 OK
=== S3) 导入 1000 行文件（含 5 类错误行配方）→ 整批拒 + 逐行回执 ===
import file built: target/t0022_import.xlsx data rows = 1005
S3 import -> HTTP 200 code 400 | 导入校验未通过，整批未入库（校验通过 1000 行 / 失败 5 行），请按回执修正后重新导入  [期望400] OK
S3 断言: 整批拒 persisted=false + 5 类错误行逐行回执原因码全部命中 OK
=== S4) 配额联动不豁免（brief 裁决 4）：quota=当前活跃 N → 修正文件 1000 行导入整批拒 4003 ===
import file built: target/t0022_import.xlsx data rows = 1000
    当前活跃点位数 N = 2
    PUT /iot/tenant pointQuota=2 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S4 import(quota=N) -> HTTP 200 code 4003 | 点位配额不足（上限 2，当前 2，本次导入 1000 行超限），请清理或扩容配额后重试  [期望4003] OK
S4 断言: 校验通过 1000 行 + 当前 2 > quota 2 → 整批拒 4003（豁免即绕过，被阻断） OK
=== S5) 放宽配额 N+1000 → 修正重导 200 + 配额计入 ===
    PUT /iot/tenant pointQuota=1002 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5 import(OK) -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5 断言: successCount=1000 persisted=true；活跃点位 2 -> 1002（导入 1000 行计入配额计数） OK
=== S6) 按编码幂等：重导同文件 → 整批 DUPLICATE_CODE 类拒绝（brief 裁决 3） ===
S6 reimport -> HTTP 200 code 400 | 导入校验未通过，整批未入库（校验通过 0 行 / 失败 1000 行），请按回执修正后重新导入  [期望400] OK
S6 断言: 重导同文件 1000 行全部 DUPLICATE_CODE、整批未入库 OK
=== S7) 导出 xlsx 与导入内容一致（数据权限裁剪照 list——admin 全量对照） ===
S7 断言: 导出 1000 行与导入文件逐字段一致（pointCode/pointName/dataType/valueType/unit/channelCode/min/max/samplePeriod） OK
=== S8) 删通道拒（其下 1000 点位——提示点位编码，ChannelReferenceGuard 转正在位复核） ===
S8 delete channel -> HTTP 200 code 400 | 通道存在点位，禁止删除（点位 1000 个: T22C_P_0001、T22C_P_0002、T22C_P_0003、T22C_P_0004、T22C_P_0005，先清理点位）  [期望400] OK
S8 断言: 删通道拒 400 + 文案携带点位数与编码示例 OK
=== S9) 反查接口（idx_tenant_point）+ 详情聚合（referenceCount + curve24h 占位） ===
--- sql-exec ---
OK(1)
DONE
--- sql-exec ---
id	instance_id	point_id	full_key
17	6	11015	F01.WS01.EQ001.t22smoke
DONE
S9a references -> HTTP 200 code 200 | 操作成功  [期望200] OK
S9a 断言: 反查返回 full_key/instanceCode/dataName 三要素 OK
S9b detail -> HTTP 200 code 200 | 操作成功  [期望200] OK
S9b 断言: 详情聚合 referenceCount + curve24h 占位字段齐备 OK
=== S10) 批量停用含引用点位 → 响应携带受影响数据项计数 + 「重新发布后对接入生效」明示 ===
S10 disable -> HTTP 200 code 200 | 操作成功  [期望200] OK
S10 断言: affectedDataItemCount=1 + effectNote 明示（保存≠生效——0024 发布链未交付） OK
=== S11) 受限用户批量启停不可见点位 → 整批 403（fail-closed） ===
S11a login -> HTTP 200 code 200 | 操作成功  [期望200] OK
S11b disable invisible -> HTTP 200 code 403 | 无权操作该点位（数据权限不命中）  [期望403] OK
S11 断言: 受限用户批量启停不可见点位整批 403、目标点位状态未变 OK
=== S12) 24h 曲线端点空集 200（TDengine fail-soft 占位——base-url localhost:6041 不可达） ===
S12 curve -> HTTP 200 code 200 | 操作成功  [期望200] OK
S12 断言: 曲线端点 200 + dataPoints 空集（真实数据归 0029 写库后联调） OK
=== S13) 收尾清理：先删 seed 数据项（解绑 T22C_P_0001）→ 批量删点位（pageSize 上限 500 循环取删） → 删通道 → 删网关 → 配额还原 ===
--- sql-exec ---
OK(1)
DONE
S13 delete points 1-200 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S13 delete points 201-400 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S13 delete points 401-500 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S13 delete points 501-700 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S13 delete points 701-900 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S13 delete points 901-1000 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S13 delete channel -> HTTP 200 code 200 | 操作成功  [期望200] OK
S13 delete gateway -> HTTP 200 code 200 | 操作成功  [期望200] OK
--- sql-exec ---
OK(1)
tenant_code	point_quota
default	null
DONE
    清理后活跃点位 total = 2 （软删 1000 行编码归档占用——不复用，残留见报告）
SMOKE DONE
```

S3 逐行回执原因码命中明细（脚本断言内部，5 类配方逐一核对）：`null→MISSING_REQUIRED`、`T22C_P_中文编码→INVALID_CODE`、`T22C_P_E3_UNIT→UNIT_NOT_IN_DICT`、`T22C_P_E4_CH→CHANNEL_NOT_FOUND`、`T22C_P_E5_RANGE→INVALID_RANGE`。

事件语义留痕（brief 裁决 6——导入落库的启用点位不发 PointEnabledEvent）：

```
$ grep -c "点位启用事件已发布" backend/logs/ot-admin-dev.log
0
$ grep "T22C" backend/logs/ot-admin-dev.log | grep -c "点位启用事件已发布"
0
```

本轮导入落库 1000 行启用点位（status=1）**零事件发布**（配置迁移语义，建子表由重生成/启用动作触发——0029 消费方联调对齐）。

冒烟 SQL（seed 数据项 + 收尾，均 dev 库冒烟配置不入库）：

```
$ bash sql-exec.sh "INSERT INTO iot_data_item (tenant_id, instance_id, data_code, full_key, data_name, source_type, point_id, enabled, status) VALUES (1, 6, 't22smoke', 'F01.WS01.EQ001.t22smoke', '冒烟反查数据项T22', 1, 11015, 1, 1);"
OK(1)
$ bash sql-exec.sh "DELETE FROM iot_data_item WHERE full_key='F01.WS01.EQ001.t22smoke';"
OK(1)
$ bash sql-exec.sh "UPDATE sys_tenant SET point_quota=NULL WHERE id=1; SELECT tenant_code, point_quota FROM sys_tenant WHERE id=1;"
OK(1)
tenant_code  point_quota
default      null
```

受限用户账号：dev 库既有冒烟账号 smoke_perm_a_t0006（user 6，dept 4，role 6 data_scope=2，iot:point:list/edit 已授权——0021/0021 修订轮 dev 配置沿用）。不可见点位用 T21_SMOKE_DEPT2（id 9002，dept 2 ∉ 可见集）。

### 2.3 验收 3：prod 档真实启动回归 —— 通过

打包 `mvn package -pl ot-admin -am -DskipTests` → ot-admin-1.0.0.jar（含两提交）。启动命令（沿 0021 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

干净单轮（日志 /tmp/task0022/prod_clean.log，Redis 预置 captcha:code:t0022p="AB21" JSON 字符串 EX 600 + SM4 admin123 密文）：

```
2026-11-22T15:16:53.686+08:00  INFO ... [main] net.otlab.admin.OtAdminApplication : Started OtAdminApplication in 10.765 seconds (process running for 11.36)
$ grep -c " ERROR " /tmp/task0022/prod_clean.log
0
$ python target/t0022_prod.py
LOGIN HTTP 200 code 200 message 操作成功
LOGIN token a7170d94...
LIST /iot/point/list HTTP 200 code 200 total 2
POINT-LIST rows: [{"pointCode": "T21_SMOKE_DEPT4", "status": 1, "deptId": 4}, {"pointCode": "T21_SMOKE_DEPT2", "status": 1, "deptId": 2}]
DETAIL /iot/point/9001 HTTP 200 code 200 referenceCount 0 curve24h []
DETAIL-AGG OK
CURVE /iot/point/9001/curve HTTP 200 code 200 windowHours 24 dataPoints []
CURVE-FAILSOFT OK
LIST /iot/gateway/list HTTP 200 code 200 total 12
PROD PROBE DONE
```

真实登录 200 + point/list 200 + **详情聚合（referenceCount/curve24h 字段在 prod 装配齐备）+ 曲线端点 fail-soft 空集 200** + gateway/list 200，启动段与全程 **ERROR=0**。用毕即停：taskkill PID 46956 → `netstat :8098 LISTENING` 复核无输出 → Redis 验证码键 DEL → 0。

### 2.4 验收 4：改动范围核对 —— 通过

`git diff --name-only a961fd1..HEAD`（23 文件，全贴回）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/controller/IotPointController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/IotPointCurveVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/IotPointDetailVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/IotPointStatusChangeVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointCurvePointVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointExportRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointImportReceipt.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointImportRejectException.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointImportRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointImportRowLimitException.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointImportRowResult.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointReferenceVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/IotPointService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/impl/IotPointServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/support/PointCurveQueryClient.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/support/PointImportExcelSaxParser.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/controller/IotPointControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/controller/IotPointImportExportControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/service/impl/IotPointImportServiceTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/service/impl/IotPointServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/service/support/PointCurveQueryClientTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/PointRedlineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/RedlineFixture.java
```

全落声明范围（主码 point/** 16 文件 + 测试 test/iot/** 7 文件）；commit 分离核对：a961fd1..963dc71=13 文件（0022）、963dc71..bfdc11b=14 文件（0023）。未触碰 frontend/、ingest/、模板/实例/字典域、dataperm、workitems 库其他文件、TASK 详情。

---

## 3. 偏离计划之处

1. **批量启停路径**：brief 裁决 4 写 `PUT /iot/point/status`——§3.1 点位段独立行为 `changeStatus` 且 0021 已按该契约交付同端点。落地为**双路由同 handler**：`@PutMapping({"/changeStatus", "/status"})`——§3.1 契约路径不变、brief 字面路径同时可用（冒烟 S10 走 changeStatus、控制器单测双路由均验证），不新增重复端点；响应体按 brief 升级为 `IotPointStatusChangeVO`（body code 200 语义不变）。
2. **ChannelReferenceGuard 点位计数转正（裁决 4 前半）零改动**：已在 BASE 就位——TASK-0021 修订轮提交 a61fd24（IotPointChannelReferenceGuard 实查 iot_point.channel_id + 拒绝文案提示点位编码前 5 条），本批按「微改攒批不重复交付」原则仅做冒烟复核（S8：删通道拒 400 + 文案携带「点位 1000 个: T22C_P_0001、…」）；裁决 4 后半「网关侧间接计数仍占位登记（0024 裁决）」维持占位（IotChannelGatewayReferenceGuard.countPoints 恒 0 + TODO 挂 0021 接线义务原文，归 0024）。channel/service/** 因此零 diff——brief 改动范围 2 为授权范围而非必改清单。
3. **导入模板通道列取 channelCode**：brief 裁决 1「channelId 或 channelCode」二选一，按裁决 2 校验项钦定「channelCode→id 解析，不存在拒」取 channelCode（人读友好 + 与导出回填闭环）；boundInstanceId/deptId 列为数值 id（brief 原文「可空」）。
4. **通道编码歧义拒**：channelCode→id 解析在租户内命中多通道时（uk 为 (gateway_id, channel_code)，同码跨网关可并存）报 `CHANNEL_AMBIGUOUS` 整批拒——brief 未枚举该情形，静默取首个会把点位挂错通道，故按「不存在拒」同一防线口径显式拒绝（原因码为 brief 十一类原因码体系的超集项）。
5. **详情聚合 curve24h 走真实 fail-soft 查询**（与曲线端点同一服务方法），非静态恒空字段——TDengine 不可达时行为与「占位」契约一致（空数组不报错），真实数据到位后详情与端点同步受益；代价为 TDengine 停机时详情读路径有一次 REST 失败等待（连接拒绝瞬时；跨网段不可达时受 connect-timeout 3s 上限，fail-soft 不炸穿）。
6. **PointImportExcelSaxParser 为点位域自含实现**（照 0009/0018 修订轮 SAX 范式逐行为继承）：未泛化改造字典解析器——DictImportExcelSaxParser 固定五列字典域映射且文件在 iot 根包（泛化超出本批声明范围），两解析器结构同源、行为口径一致（r 属性行号/缺格 null/超限 fail fast/共享字符串/数值归一）。
7. **通道列空白允许**：channelCode 空白=不挂通道（与单点建档 channelId 可空口径一致——0021 表单未强制）；brief 列清单未标注通道必填。
8. **净增用例计数口径**：方法级净增 43、回归净增 41——差 2 条为 IotPointControllerTest 的 testDetail/testChangeStatus 原地改写（随 VO 契约对齐，不新增方法）。
9. **反查行含 itemId**：brief 三要素（full_key/实例编码/数据项名）之外附 itemId 行锚（超集，前端行定位用）；曲线 SQL 带 `LIMIT 5000` 保护上限（1s 采样 24h 理论 86400 行，窗口下采样策略归 0068 联调）且 ts 原文透传（taosAdapter 应答形态适配归 0068）。
10. **冒烟轮次修正**（脚本缺陷轮未采信，最终连续单轮全绿贴回——0021 先例）：①首轮 sql-exec 子进程用 Windows 盘符路径被 msys/WSL bash 语义漂移吞掉（rc=127 静默）→ 显式锁 Git Bash 绝对路径修正；②S13 清理首版先删点位后删 seed，被绑定保护整批拒触发 runaway 循环（已被「防线先于业务」语义拦下——恰好实证了绑定保护与不半批变更）→ 改为先删 seed 并加防 runaway 熔断；③编码归档不复用语义使 GW_T22_M1/M2 残留占用 → 冒烟逐轮换后缀（M1→M2→M3），S2a 拒绝文案「网关编码已被删除记录占用，不可复用」反为该语义提供了活体实证。

## 4. 未做项与原因

1. **网关侧间接计数（点位→网关）转正**：brief 裁决 4 原文「仍占位登记（0024 裁决）」——维持 `IotChannelGatewayReferenceGuard.countPoints` 恒 0 占位，登记归 0024。
2. **发布联动真实现**：批量启停「变更生效走 FEAT-0008 发布链」归 TASK-0024（未交付）——本期变更即时落库但不产生接入生效，接口响应以 effectNote 明示（REQ-0010 保存≠生效同款口径）。
3. **曲线真实数据与 192.168.31.145 环境联调**：端点契约与查询客户端本期就位（fail-soft 空集占位），真实写库归 TASK-0029、环境联调归 TASK-0068（TD-0053 同批口径）——本批未做任何真实 TDengine 写入/联调。
4. **OPC UA 发现字段不在模板**：protocol_addr 非模板十三列（brief 列清单未含）——OPC UA 节点标识/浏览路径随单点建档 protocolAddr JSON 预留（ADR-0013），导入侧二期随协议地址配置同批评估。
5. **TASK 详情回写**：brief 契约「协调者收尾」——TASK-0022/0023 状态维持「进行中」。

## 5. 遗留问题

1. **dev 库冒烟残留（不影响仓库与提交）**：三轮冒烟的软删点位行（T22_P_×1000 + T22B_P_×1000 + T22C_P_×1000——编码归档占用永不复用）、已删网关/通道行（GW_T22_M1/M2/M3、CH_T22_M1/M2/M3）；iot_data_item 冒烟 seed 已硬删（full_key=F01.WS01.EQ001.t22smoke）；sys_tenant.point_quota 已还原 NULL。后续任务冒烟可直接复用「已删行占用」语义做预检用例，不需要时手工清理。
2. **配额 TOCTOU 残余**：导入侧配额预检（当前计数 + 校验通过行数）与逐行落库之间同样存在并发窗口——0003 评审 M-7 裁决接受并登记的同款残余，导入侧一并登记（物理唯一索引兜底编码正确性，仅计数可能瞬时超限）。
3. **同态 no-op 仍发事件（启用侧）**：changeStatus 置 1 对已是启用状态的行仍逐行发 PointEnabledEvent（0021 评审后保留行为，消费方幂等承接）；changedCount 只计实际变更行——事件数与 changedCount 可不等，前端展示以 changedCount 为准。
4. **手工建档无编码合规白名单**：导入侧强制 [A-Za-z0-9_.]（brief 裁决 2），单点 create 沿 0021 契约仅长度校验——曲线查询对白名单外编码 fail-soft 空集兜底（不拼接进 SQL）。两入口口径差异登记为观察项，如需拉齐归后续任务裁决（涉及 0021 已验收契约）。
5. **管理面列表 pageSize 上限 500**：导出不受影响（selectExportRows 无分页全量）；冒烟清理脚本因此改为循环取删。框架既有行为，仅记录。

---

## 6. 修订轮（TASK-0022 验收退回，diff 评审 I-1 必修项；0023 分节维持全绿不动）

评审报告：TASK-0022-0023-review.md（Critical 0 / Important 1 / Minor 7——仅 0022 节退回）。
修订轮提交：**071dd1f**（main，fix(iot) TASK-0022 修订轮，3 文件 +144/−6——改动严格限 point 域导入校验器与测试：`PointImportRowResult.java` / `IotPointServiceImpl.java` / `IotPointImportServiceTest.java`）。

### 6.1 修复内容（I-1：导入缺 deptId/boundInstanceId 存在性/同租户校验）

- **取舍说明（原因码）**：新增独立原因码 `INVALID_REF`——格式非法已由既有 `INVALID_NUMBER` 承载（数值列解析），「引用存在性/同租户」属独立语义域，与单位字典（UNIT_NOT_IN_DICT）/通道解析（CHANNEL_NOT_FOUND）平级，复用会混淆用户修正路径；失败文案标注缺失列名与 id（如 `归属引用不存在或已删除: deptId(777)`），逐行回执可直接定位修正。
- **落地形态**：导入第二遍库内级校验新增 `markReferenceViolations`——deptId/boundInstanceId 去重后批内**一次 IN 预检**（sys_dept/iot_model_instance 均在多租户插件覆盖清单内，会话租户条件注入即同租户保证、@TableLogic 过滤已删行——与单点建档 validateDept/validateInstance 完全同口径），未命中标 INVALID_REF；未携带的行不触发校验（DDL 可空口径与单点建档一致）。校验位于占用预检之前、配额预检之前——整批拒语义不变（任一行失败整批不落库）。
- **红转绿（TDD 记录）**：红轮 3 用例失败（`testImport_DepartmentRefNotFoundRejected`/`testImport_BoundInstanceRefNotFoundRejected`/`testImport_ReferenceResolvedPassesWithBatchQuery`——实证修订前悬空 deptId=777/boundInstanceId=888 被**静默放行**、selectByIds 零调用），补 `markReferenceViolations` 后 21/21 转绿；「为什么现在对了」：归属锚点写入面与单点建档收敛到同一契约（存在/同租户/未删），数据权限裁剪与统计不再有经导入注入悬空 id 的通道。

### 6.2 补充验收 1：全量复跑 —— 通过

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[INFO] Tests run: 1866, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

1866 = 修订轮前 1863 + 3（新增归属引用用例，红转绿记录见 §6.1）；0 失败 0 错误（3 skipped 为红线二-中台 scope 占位，与基线同值）。

### 6.3 补充验收 2：dev 冒烟复核 —— 通过

重建 jar（含 071dd1f）→ dev.ps1 start（8098，启动 14.286s）→ 冒烟脚本 `backend/target/t0022rev_smoke.py` 原始输出（openpyxl 告警行略）：

```
=== S1) admin dev login === S1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
    当前活跃点位数 = 2
=== S2) 导入含不存在 deptId / boundInstanceId 行的文件 → 整批拒 + 逐行回执标注 INVALID_REF ===
file built: target/t0022rev_bad.xlsx
S2 import(bad refs) -> HTTP 200 code 400 | 导入校验未通过，整批未入库（校验通过 2 行 / 失败 2 行），请按回执修正后重新导入  [期望400] OK
S2 断言: 2 行 INVALID_REF（deptId=999999 / boundInstanceId=888888 逐行标注）+ 整批拒 persisted=false OK
S2 断言: 整批未落库 OK
=== S3) 修正重导（去掉两行坏引用）→ 200 + 配额计入 ===
file built: target/t0022rev_ok.xlsx
S3 import(corrected) -> HTTP 200 code 200 | 操作成功  [期望200] OK
S3 断言: 修正后 2 行成功落库、无归属列落 NULL OK
=== S4) 收尾清理：删除修订轮冒烟点位，配额还原确认 ===
S4 delete -> HTTP 200 code 200 | 操作成功  [期望200] OK
    清理后活跃点位 total = 2 （软删编码归档占用——不复用，残留见报告）
SMOKE-REV DONE
```

S2 回执明细（脚本断言内部）：`T22R_P_0003 → INVALID_REF，文案含 "deptId(999999)"`；`T22R_P_0004 → INVALID_REF，文案含 "boundInstanceId(888888)"`；坏引用行与正常行混批**整批拒**（persisted=false、任何行未落库）。S3 修正文件（去两行坏引用）重导 200（successCount=2/persisted=true），修正行落库且无归属列落 NULL。首版脚本 S3 因 @RepeatSubmit 3s 防重窗口被拒（sleep 1s 过短），间隔 4s 复跑即绿——脚本缺陷轮未采信，最终轮全绿贴回（沿 0021 先例口径）。

自起实例用毕即停：dev.ps1 stop → PID 31184 已停 → `netstat :8098 LISTENING` 复核无输出。

### 6.4 修订轮改动范围核对

`git diff --name-only bfdc11b..071dd1f`：3 文件，全落契约（point 域导入校验器与测试；0023 已验收部分零触碰）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/PointImportRowResult.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/impl/IotPointServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/service/impl/IotPointImportServiceTest.java
```

### 6.5 Minor 7 条登记（评审裁定不动、记 deferred，收尾进 LOG）

- **TASK-0022**：1-2 failRow/rowResultOf 按 rowIndex 线性扫描最坏 O(n²)（改 rowIndex→result Map 即消除）；1-3 逐行 save 建议改 saveBatch/JDBC 批量（时延优化）；1-4 损坏/非 xlsx 上传宜映射 PARAM_ERROR 而非系统级 500 形态；1-5 selectExportRows 全量无行数上限（quota=null 租户内存放大面）。
- **TASK-0023**：2-1 曲线客户端行解析短行防御（现靠服务层双保险兜住，fail-soft 结果不变）；2-2 详情单次 resolve 两次 + 曲线 REST 尝试延迟（偏离 5 已登记项的量化）；2-3 批量启停逐行 updateById/同态行也发事件（0021 已验收遗留，本批仅改响应体）。

### 6.6 修订轮遗留

1. dev 库冒烟残留新增：T22R_P_0001/0002 软删点位行（编码归档占用不复用）；sys_tenant.point_quota 维持 NULL。
2. 冒烟脚本残留文件 backend/target/t0022rev_*.xlsx（target 不入库）。
