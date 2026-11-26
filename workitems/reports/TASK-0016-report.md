# TASK-0016 验收报告：实例版本锁定与待升级清单——后端接口组

> 执行者：TASK-0016 执行；验收日期：2026-11-23；工作区 D:\workspace\WisFactory（main 分支）。
> 开工 BASE = **b4e877a**（`git rev-parse --short HEAD`）；实现提交 = **082d694**（13 文件）。
> 环境：WSL docker wisfactory-mysql（3307）/ wisfactory-redis（6379）全程在跑（保活循环规避 WSL2 空闲关停，全程未复发）；8098 启动前探测空闲、dev×1 + prod×1 轮用毕即停（两次 kill 后均确认监听归零）；TDengine 未触碰（本批不涉）；ingest/**（Go 道 TASK-0028 在途）未触碰——提交逐文件显式 add（未用 git add -A，.agents/ 环境产物未入库）。
> 无新 SQL（brief 前提：锁定两列 0012 已建、版本历史表 0015 已建——实现全程未发现缺表/缺列，未触发停下条款）。

---

## 一、改动文件清单（验收 4：改动范围核对）

`git diff --name-only b4e877a..HEAD` 原始输出（13 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/controller/IotInstanceLockController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/controller/IotTemplateUpgradeController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/dto/IotInstanceLockForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/dto/TemplateUpgradeConfirmForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/dto/TemplateUpgradeRowVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/IotInstanceUpgradeService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/IotTemplateVersionService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotInstanceUpgradeServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotTemplateVersionServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/controller/IotInstanceLockControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/controller/IotTemplateUpgradeControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotInstanceUpgradeServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotTemplateVersionServiceImplTest.java
```

**比对结论：通过。** 13 文件全落 brief「改动范围」声明内：
- `iot/template/**` 主代码 9 文件：新控制器 2（IotInstanceLockController=brief 接口 1 / IotTemplateUpgradeController=brief 接口 2/3）+ 新 DTO 3（锁定表单/确认表单/清单行 VO）+ 新服务接口与实现（IotInstanceUpgradeService 三方法）+ 既有版本服务接口与实现各扩 **buildCurrentContentJson** 一方法（待升级清单 diff 当前侧与版本历史行同一 canonical 构建器）；
- `src/test/java/**` 测试 4 文件：新服务测试 19 用例 + 新控制器测试 10 用例 + 既有版本服务测试补 canonical 一致性契约 2 用例；
- **无 SQL、无 yml、无 ot-core、无 publish、无前端、无 ingest 改动**（净足迹逐字核对如上）。

## 二、每条验收命令原始输出

### 验收 1：单测全量（BUILD SUCCESS、0 失败 0 错误）

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`（在最终提交 082d694 上完整重跑）。

ot-core install 段：`BUILD SUCCESS`（EXIT=0）。

ot-admin 测试 Results 段（全量日志尾段）：

```
[INFO] Results:
[INFO]
[INFO] Tests run: 2332, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:04 min
[INFO] Finished at: 2026-11-23T03:43:25+08:00
```

**通过**：2332 通过 / 0 失败 / 0 错误 / 0 跳过（0015 修订轮交付时 2298，本任务净增 34：新服务 19 + 新控制器 10 + 版本服务 canonical 契约 2 + 既有测试自然波动 3 归基线口径）；0024 快照组装既有测试全绿——**锁定实例未确认前保持旧口径的快照组装面零变化**（brief 测试口径「快照组装不受影响」）。

**TDD 红转绿留痕**（brief 测试口径）：

红（stub 阶段首跑，`mvn test -Dtest='IotInstanceUpgradeServiceImplTest,IotTemplateVersionServiceImplTest#buildCurrentContentJson*'`）：

```
[ERROR] Tests run: 19, Failures: 10, Errors: 9, Skipped: 0 -- in ...IotInstanceUpgradeServiceImplTest
java.lang.UnsupportedOperationException: TASK-0016 实现待落地   ← 19 例全数红（行为级）
[ERROR] IotTemplateVersionServiceImplTest.buildCurrentContentJsonTemplateMissing:432
  Unexpected exception type thrown, expected: <net.otlab.core.exception.BusinessException>
  but was: <java.lang.UnsupportedOperationException>
```

（首轮 stub 连依赖字段也未声明时 19 例全量 setUp 报错=接线缺失实证 `Could not find field 'instanceMapper'`；补 @Autowired 字段后红落在行为面——同 0015 修订轮红取证口径。控制器 10 用例为接口契约断言——权限注解/路由/bean 校验属声明性检查，stub 阶段即绿，同 0015「纯函数引擎直接绿」先例，行为红由服务层 19 例承载。）

绿（实现交付后目标 4 测试类全绿）：

```
[INFO] Tests run:  5, Failures: 0, Errors: 0, Skipped: 0 -- in ...controller.IotInstanceLockControllerTest
[INFO] Tests run:  5, Failures: 0, Errors: 0, Skipped: 0 -- in ...controller.IotTemplateUpgradeControllerTest
[INFO] Tests run: 19, Failures: 0, Errors: 0, Skipped: 0 -- in ...service.impl.IotInstanceUpgradeServiceImplTest
[INFO] Tests run: 19, Failures: 0, Errors: 0, Skipped: 0 -- in ...service.impl.IotTemplateVersionServiceImplTest
[INFO] Tests run: 48, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

红转绿期间修复附根因与「为什么现在对了」：

1. **LambdaUpdateWrapper 缺 import 编译错**：实现引用了 `com.baomidou.mybatisplus.core.conditions.update.LambdaUpdateWrapper` 未导入。为什么现在对了：补标准 import 后乐观锁更新包装器编译通过，`getSqlSet()` 断言「更新集仅 applied_template_version 列」与 `getParamNameValuePairs()` 断言「守卫旧值=确认前 applied、目标值=当前版」在单测中如实命中。
2. **测试夹具伪条目**（listRowWithTier3Entry 红：`expected: <T3:1> but was: <T1:1,T3:1>`）：夹具的「当前内容 JSON」模板头缺 `remark` 键而基线有——diff 引擎如实报出 template.remark 的 CHANGED（T1）。为什么现在对了：夹具模板头与基线对齐（只动数据项面）后，diff 恰为预期的单条 REMOVED T3——引擎逐字段 diff 行为正确，噪声是测试侧构造不严谨，非实现缺陷。

### 验收 2：dev 冒烟全序列（每步原始输出）

dev 档启动（`mvn spring-boot:run` 默认 dev，8098 启动前探测空闲）：`Started OtAdminApplication in 9.992 seconds`，profile `"dev"`，登录走 `/auth/login/test`（dev 专用，code 200，token=5cd4adc1-…）。启动段 0 ERROR（下述全程 6 条 ERROR 均为负例探针的预期业务拒绝——GlobalExceptionHandler 对「编码占用/锁定改名拒/目标版本校验拒/重复提交 4002」的如实留痕，非系统缺陷，见各步骤）。

**STEP1-2 建链（模板 + 数据项 + 三实例）**

```
STEP1 POST /iot/template {"templateCode":"SMOKE0016_TPL","nodeType":4}
{"code":200,...}  —— 模板 id=11，version=2（基线 v1 + 两数据项 T1 自动升版）
STEP2 POST /iot/template/item（templateId=11，ELEC_CONS 直绑 + LOAD_RATE 公式 "{SELF.ELEC_CONS}/100"）
{"code":200,"data":2,...}
DB 核对：
SELECT id,template_id,version,tier_summary FROM iot_model_template_version WHERE template_id=11 ORDER BY version
5  11  1  null    ← 建模基线（tier_summary=NULL）
6  11  2  T1:2    ← 新增两数据项自动升版（一档×2）
STEP2b 三台设备实例（parentId=0，templateId=11；短码 E6A/E6B/E6C——EQ1x 已被 0015 遗留实例占用，
首次尝试返回 4002「实例编码已存在」×3，即双道闸实测）→ 23/24/25：
SELECT id,instance_code,template_id,applied_template_version,locked,status FROM iot_model_instance WHERE template_id=11
23  E6A  11  2  0  1
24  E6B  11  2  0  1
25  E6C  11  2  0  1   ← applied=模板当前 version（TD-0008 语义在位）
```

**STEP3：T2 变更——改 LOAD_RATE 默认公式（item 19，"/100" → "/1000"）**

```
STEP3 PUT /iot/template/item（item 19 实体直传）→ {"code":200}
DB：版本历史新增行 7 | T2:1；模板 version 自动 2→3
```

**STEP4：待升级清单接口（brief 接口 2 首验）**

```
STEP4 GET /iot/template/upgrade/list?templateId=11&pageNum=1&pageSize=10
{"code":200,"data":{"current":1,"pages":1,"records":[
 {"appliedTemplateVersion":2,"currentVersion":3,"diff":[{"objectType":"dataItem","objectKey":"LOAD_RATE",
  "changeType":"CHANGED","field":"defaultFormula","oldValue":"{SELF.ELEC_CONS}/100",
  "newValue":"{SELF.ELEC_CONS}/1000","tier":"T2"}],"instanceCode":"E6A","instanceId":23,
  "instanceName":"Smoke16 E6A","locked":0,"tierSummary":"T2:1"},
 {... E6B id=24 locked=0 ...},{... E6C id=25 locked=0 ...}],"size":10,"total":3},...}
—— 3 行全出（applied=2 < current=3），diff 逐条目带档位（T2 defaultFormula），
   tierSummary=T2:1；instanceCode 为纯增量展示字段（偏离登记见「四」）
```

**STEP5：锁定开关（brief 接口 1 首验）+ 锁定编辑受限（0012 既有口径接口层校验确认）**

```
STEP5a PUT /iot/instance/lock {"instanceId":25,"locked":1}
{"code":200,"data":null,"message":"操作成功"}
STEP5b 负例：锁定实例改名 PUT /iot/instance {"id":25,"instanceName":"RENAMED-SHOULD-FAIL"}
{"code":400,"message":"实例已锁定，仅允许修改状态"}   ← 0012 既有锁定编辑受限口径确认（本任务未改动该逻辑）
STEP5c 正例：锁定实例仅 status 放行（{"id":25,"status":0} → 200；{"id":25,"status":1} → 200 恢复）
DB：SELECT id,instance_code,locked,status FROM iot_model_instance WHERE id=25
25  E6C  1  1                                     ← locked=1 落库
服务日志：[INSTANCE-LOCK] instanceId=25 locked 0 -> 1 (userId=1)
```

**STEP6：锁定实例保持旧口径——影响预览 affected/locked 分列（照 0015 冒烟）**

```
STEP6 GET /iot/template/impact-preview?templateId=11
{"code":200,"data":{"affectedInstanceCount":3,...,"currentTemplateVersion":3,
 "diff":[{"...LOAD_RATE defaultFormula ... T2"}],"lockedInstanceCount":1,
 "previewToken":"e1852671...","targetVersion":4,...}}
—— 分列正确：affected=引用且启用总数（3），locked=locked 子集单列（1，即 E6C）
```

**STEP7：目标版本校验负例 + 单台确认 + 重复提交口径**

```
STEP7a 负例 POST /iot/template/upgrade/confirm {"instanceIds":[23],"targetVersion":2}
{"code":400,"message":"目标版本必须等于模板当前版本 3（不允许任意目标版）"}   ← 不允许任意目标版
STEP7b 单台确认 POST /iot/template/upgrade/confirm {"instanceIds":[23],"targetVersion":3}
{"code":200,"data":null,"message":"操作成功"}
STEP7c 幂等重放同台 → {"code":4002,"message":"实例已在目标版本（并发确认或重复提交），
  请刷新待升级清单后重试: E6A"}     ← 4002 口径（偏离登记见「四」3）
DB：23 E6A applied=3（2→3）；24/25 仍 applied=2
服务日志：[INSTANCE-UPGRADE] confirmed instanceId=23 templateId=11 applied 2 -> 3 (userId=1)
```

**STEP8-9：批量确认余下（含锁定实例）→ 清单清空**

```
STEP8a POST /iot/template/upgrade/confirm {"instanceIds":[24,25],"targetVersion":3}
{"code":200,"data":null,"message":"操作成功"}        ← 25 为锁定实例——锁定实例升级通道实证
DB：SELECT id,instance_code,applied_template_version,locked FROM iot_model_instance WHERE template_id=11
23  E6A  3  0
24  E6B  3  0
25  E6C  3  1                       ← 全部 applied=当前版 3（25 保持 locked=1）
STEP9 GET /iot/template/upgrade/list?templateId=11
{"code":200,"data":{"current":1,"pages":0,"records":[],"size":10,"total":0},...}   ← 清单清空
服务日志（逐台留痕）：
[INSTANCE-UPGRADE] confirmed instanceId=24 templateId=11 applied 2 -> 3 (userId=1)
[INSTANCE-UPGRADE] confirmed instanceId=25 templateId=11 applied 2 -> 3 (userId=1)
```

**STEP10：operate log 留痕核对（sys_operate_log 实查）**

```
SELECT id,module,business_type,method,url,LEFT(param,150),status,create_time
  FROM sys_operate_log WHERE url LIKE '%/iot/instance/lock%' OR url LIKE '%/iot/template/upgrade%'
  ORDER BY id DESC LIMIT 8
5565  待升级清单确认升级  2  ...IotTemplateUpgradeController.confirm()  /iot/template/upgrade/confirm
      {"instanceIds":[24,25],"targetVersion":3}                                    1  2026-11-23 03:45:36
5564  待升级清单确认升级  2  ...confirm()  /iot/template/upgrade/confirm
      {"instanceIds":[23],"targetVersion":3}                                       0  03:45:28  ← 重复提交失败如实记
5563  待升级清单确认升级  2  ...confirm()  /iot/template/upgrade/confirm
      {"instanceIds":[23],"targetVersion":3}                                       1  03:45:28  ← 单台确认
5562  待升级清单确认升级  2  ...confirm()  /iot/template/upgrade/confirm
      {"instanceIds":[23],"targetVersion":2}                                       0  03:45:28  ← 目标版本校验失败如实记
5558  实例版本锁定        2  ...IotInstanceLockController.setLock()  /iot/instance/lock
      {"instanceId":25,"locked":1}                                                 1  03:45:17
—— 每次确认/锁定请求一条 @OperateLog 留痕（param 含全量 instanceIds 逐台可追溯；
   失败请求 status=0 如实记），服务层另有逐台 [INSTANCE-UPGRADE] INFO 留痕
```

**验收 2 全序列通过。**

### 验收 3：prod 档真实启动回归

启动（本地回归口径，环境变量注入本地连接与密钥，8098 启动前探测空闲）：

```
cd backend/ot-admin && DB_HOST=127.0.0.1 DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=127.0.0.1 REDIS_PORT=6379 REDIS_DATABASE=0 SM4_KEY=0123456789abcdef \
CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 WECHAT_MP_ENABLED=false \
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

启动日志：

```
2026-11-23T03:46:40.542+08:00  INFO ... The following 1 profile is active: "prod"
2026-11-23T03:46:49.530+08:00  INFO ... Started OtAdminApplication in 9.347 seconds (process running for 9.712)
ERROR_COUNT=0 （`grep -c " ERROR "` = 0——含启动与全部探针，全程无 ERROR）
```

prod 下认证走正式链（`GET /auth/captchaImage` → uuid=8a42b155-… → WSL redis 读 `captcha:code:<uuid>` 码 F9H6 → SM4 传输密文（hutool SM4 encryptHex(admin123) 再 Base64，BC provider 注册后生成）→ `POST /auth/login`）登录成功（token 32c717cb-…）。

prod 各端点探针（新端点在 prod 生效 + 既有端点回归）：

```
PROBE1 GET /iot/template/upgrade/list?templateId=11
{"code":200,"data":{"current":1,"pages":0,"records":[],"size":10,"total":0},...}
 —— 新端点 200（清单数据面空=dev 冒烟已全量确认的延续状态，端点链路在 prod 生效）
PROBE2 PUT /iot/instance/lock {"instanceId":23,"locked":1} → 200；{"instanceId":23,"locked":0} → 200
 —— 锁定开关 prod 往返（净零）；新端点写路径在 prod 生效
PROBE3 GET /iot/template/impact-preview?templateId=11
{"code":200,"data":{"affectedInstanceCount":3,...,"lockedInstanceCount":1,...,"diff":[],...}}
 —— 0015 端点回归 200（locked=1 即 E6C 保持锁定；diff 空=全部实例已确认、无滞后跟随实例，语义自洽）
PROBE4 GET /iot/publish/list?scopeType=2&scopeId=0&pageNum=1&pageSize=3
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":30,...,"version":3,...},...],...}} —— 既有端点 200
```

**通过**：prod 档真实启动 0 ERROR + 正式登录链 + 新端点（清单/锁定）与既有端点（预览/发布列表）全 200。验毕后 backend 已停止（8098 监听确认归零，纪律保持）。

### 验收 4：改动范围核对

见「一、改动文件清单」——`git diff --name-only b4e877a..HEAD` = 本任务提交 082d694 的净足迹 13 文件全部落在 brief 声明范围内（template/** 主代码 9 + src/test 4）；**无 SQL/yml/ot-core/publish/前端/ingest 改动**，**通过**。

## 三、未做项与原因

1. **TDengine**：按协调者指令本批不涉；本任务链路（实例锁定/清单/确认）数据源为 MySQL 业务表，冒烟与 prod 回归均未触 TDengine。
2. **实例数据面重生成（确认升级的「按新版本生效」数据侧）**：brief 决策 3 的确认机制穷举为「逐台 UPDATE applied_template_version（乐观锁）+ 逐台写操作日志」，未含数据项重生成——本任务按 brief 字面交付簿记面推进（确认门滞后判定 hasPendingTier2Confirm 的 `lt(applied_template_version, row.version)`、待升级清单、预览 locked 分列三个消费方即时兑现，冒烟全序列实证）。实例数据项按新模板版本对齐（新增项落实例行）可经既有 `POST /iot/instance/{id}/items/regenerate`（0013 交付）达成，其与确认动作的编排（确认时自动 regenerate 或保持显式两步）不在本 brief 三接口决策内——登记为遗留裁决项（「五」1，0024 组装读实例数据项行，applied 推进本身不改变快照内容，该口径如实登记）。
3. **前端**：brief 明文不碰——0070 批二接本 API（三端点已就绪：PUT /iot/instance/lock、GET /iot/template/upgrade/list、POST /iot/template/upgrade/confirm）。
4. **菜单种子**：无新增（清单查询复用 iot:template:query 4025 既有、锁定/确认复用 iot:instance:edit 4034 既有——brief 决策 4）。
5. **数据权限裁剪的 live 冒烟**：单元测试 4 例锁定（锁定 403 / 确认 403 / 清单可见集裁剪 / createByOnly OR 分支）；live 侧需受限账号建链（角色/部门/用户三链），brief 冒烟序列未含——以单测留痕为口径。

## 四、偏离计划之处

1. **锁定开关控制器落位 template/controller（新类 IotInstanceLockController）而非改 IotModelInstanceController**：brief 改动范围只声明 `iot/template/**`（instance/controller 在范围外，「越界即违约」）——而接口路径 /iot/instance/lock 属实例域。处置：新控制器落位 template/controller（与路径契约一致），包位依据=IotModelInstance 实体与 Mapper 本就随 0011 建库先行落位 template 包、锁定/升级三接口同属 ADR-0008 模板变更控制落地件（清单 diff 直接消费版本历史）；类级 @RequestMapping("/iot/instance") 与既有实例控制器共存（PUT /lock 与既有 PUT ""/GET /{id} 方法+路径组合不冲突，prod 启动与探针实证）。javadoc 已登记。
2. **确认升级权限选 iot:instance:edit**（brief 决策 4 给出「复用既有 iot:instance:edit / iot:template:query」两可面）：确认动作逐台推进实例行（实例写操作），与锁定开关同权限面——持模板查询而无实例编辑权限的用户可看清单不可确认（视线与动作分离）；清单查询取 iot:template:query（brief 决策 2 明文）。javadoc 已登记理由。
3. **applied == targetVersion 的重复提交按 4002 整批拒绝（非静默跳过）**：同值 UPDATE 的影响行数受 JDBC 驱动 affected/found 口径歧义（MySQL Connector/J useAffectedRows 两态），乐观锁守卫对「已在目标版本」的判定不可靠——显式前置判定 fail-closed，幂等重放提示刷新清单重选（冒烟 STEP7c 实证）；与乐观锁冲突同归 4002（brief「冲突返回 4002 口径照 0015 修订轮范式」的延伸解释）。
4. **清单筛选与分页在内存**（一次查出模板实例 → 内存判定 applied<current/回退跳过/权限裁剪 → 切片分页，Page<VO> 手工组装 total/records/current/size/pages 与实例列表序列化面一致）：照 0012 树裁剪「一次查出后内存过滤」范式——三类边界语义（无差异不出现/回退跳过+WARN/裁剪）因此可在单元测试逐例锁定；模板内实例量级十~百，全量装载可接受（同 0012 树查询口径）。
5. **清单行 VO 增 instanceCode 展示字段**（brief 字段面之外的纯增量）：前端清单渲染设备编码所需——照 0015 影响预览 VO「附加字段为前端展示所需纯增量信息」同口径，javadoc 已登记。
6. **冒烟短码换用 E6A/E6B/E6C**：EQ1x 已被 0015 遗留实例（20/21/22）占用（双道闸 4002 实测留痕）——冒烟序列语义不受影响。
7. **提交粒度**：单提交 082d694（实现+测试一体，红绿证据均在最终代码上复核）。

## 五、遗留问题

1. **[确认升级数据面边界登记（重要，建议协调者裁决）]** 确认升级推进的是版本簿记（applied_template_version）：确认门窗口关闭、待升级清单清空、预览 locked 分列三个消费面即时兑现（本批冒烟实证）；实例数据面「按新版本生效」（新增模板项落实例行、T2 默认值向实例行传播）不在本批三接口机制内——现有 regenerate 端点可对齐新增项，但「确认后自动 regenerate」还是「保持显式两步」未决；0024 快照组装读实例数据项行（不读 applied 列），故确认本身不改变下次快照内容。REQ-0006 验收 3「逐台确认后按新版本生效」的完整闭环与 0015 报告遗留 #1 的「实例升级链」注意项在此收口待裁。
2. **[观察项]** 冒烟数据留存库内（模板 11 / 实例 23-25 / 版本历史行 5-7 / 操作日志 5558-5565），照 0015/0024 冒烟遗留同口径隔离（E6C 保持 locked=1 为 prod PROBE3 locked 分列探针的延续状态）。
3. **[观察项]** IotInstanceLockController 与 IotModelInstanceController 共享 /iot/instance 基础路径（偏离 1 的包位取舍）——如后续修订轮要求包位归一（锁定开关并入实例控制器），需同步扩 brief 改动范围授权。
4. **[环境]** WSL2 空闲关停同 0024/0025/0026/0015 报告登记（本批以保活循环规避，全程未复发）；8098 dev/prod 两次启动均用毕即停且确认监听归零。

## 六、一句话结论

**通过**——验收 1（最终提交 082d694 全量 mvn：**2332 测试 0 失败 0 错误 BUILD SUCCESS**，TDD 红 19 例行为级 `UnsupportedOperationException`→绿 48 目标测试，2 项修复附根因）、验收 2（dev 冒烟全序列：T2 改公式升版 v3/T2:1 → 清单 3 行 diff 带 T2 条目 → 锁 E6C（DB locked=1、锁定编辑受限 400「仅允许修改状态」+status 放行）→ 预览 affected=3/locked=1 分列 → 目标版本负例 400 → 单台确认 applied 2→3 → 重复提交 4002 → 批量确认含锁定实例 applied 全=3 → 清单清空 total=0 → sys_operate_log 五条留痕实查含失败如实记）、验收 3（prod 档启动 9.3s 全程 **0 ERROR** + captcha/SM4 正式登录链 + 新端点清单/锁定与既有预览/发布列表全 200）、验收 4（净足迹 13 文件全落 brief 声明范围、无 SQL/yml/ot-core/publish/前端/ingest 改动）四条全部以原始输出为证通过；偏离均登记（锁定控制器包位、确认权限面、applied==target 4002 口径、内存分页），遗留仅登记级与「确认升级数据面编排」待裁项，无阻断缺陷。
