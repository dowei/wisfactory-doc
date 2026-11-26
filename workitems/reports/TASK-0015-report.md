# TASK-0015 验收报告：模板变更分档判定与影响预览——发布确认门真接线

> 执行者：TASK-0015 执行；验收日期：2026-11-23；工作区 D:\workspace\WisFactory（main 分支）。
> 开工 BASE = **f195b58**（`git rev-parse --short HEAD`）；实现提交 = **fa064cd**（28 文件）+ **83b99c3**（确认门日志参数修正）+ **b3d1e0c**（范围回退，见「四」2）。
> **范围核对基准修正（并行道事实）**：BASE 记录后、本任务首个提交前，TASK-0070 前端批次 **d02ac44** 落入 main（fa064cd 的父提交即 d02ac44，`git log --format="%h parent=%p"` 实证）——净足迹核对以 `git diff --name-only d02ac44..HEAD` 为准（29 文件，全部落 brief 声明范围）；`f195b58..HEAD` 会混入 TASK-0070 的 8 个前端文件（非本任务改动）。
> 环境：WSL docker wisfactory-mysql（8.0.42，127.0.0.1:3307）/ wisfactory-redis（6379，backend db0）；8098 启动前探测空闲、dev×1 + prod×2 轮用毕即停（三次 kill 后均确认监听归零）；TDengine 未触碰；WSL 保活循环全程未复发空闲关停；ingest/**（Go 道 TASK-0027 在途改动）未触碰。

---

## 一、改动文件清单（验收 5：改动范围核对）

`git diff --name-only d02ac44..HEAD` 原始输出（29 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/config/PublishProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/PublishForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/gate/NoOpPublishConfirmGate.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/gate/PublishConfirmGate.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/gate/TemplateAwarePublishConfirmGate.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/controller/IotModelTemplateController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/IotModelTemplateVersion.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/dto/TemplateChangeEntryVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/entity/dto/TemplateImpactPreviewVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/mapper/IotModelTemplateVersionMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/IotTemplateVersionService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotDataItemTemplateServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotModelTemplateServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotTemplateVersionServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/tier/TemplateChangeEntry.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/tier/TemplateChangeTier.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/tier/TemplateContentDiffEngine.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/tier/TemplateTierRuleTable.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/gate/TemplateAwarePublishConfirmGateTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/controller/IotModelTemplateControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotDataItemTemplateServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotModelTemplateServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotTemplateVersionServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/tier/TemplateContentDiffEngineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/tier/TemplateTierRuleTableTest.java
backend/ot-core/src/main/java/net/otlab/core/result/ResultCode.java
resources/sql/upgrade/20261123_iot_template_version.sql
```

**比对结论：通过。** 29 文件全落 brief「改动范围」声明内：
- `iot/template/**`（15 主代码：tier 引擎 4 + 版本历史实体/Mapper/服务 5 + 钩子接线 2 + 控制器 1 + VO 2 + 版本实体 1）——分档引擎/diff/版本历史/影响预览；
- `iot/publish/gate/**`（3：接口加默认五参重载、NoOp 去 Bean 退役为测试替身、真实现 TemplateAwarePublishConfirmGate）+ `IotPublishServiceImpl.java`（仅 gate 调用点适配：改调五参重载传 confirmToken + 拒绝码 5002，最小改动）+ `PublishForm.java`（扩 confirmToken）+ `PublishProperties.java`（扩 confirmTokenTtlSeconds）；
- **ot-core 触点仅 ResultCode.java 一处**（新增 5002 PUBLISH_NOT_CONFIRMED）——与 brief 声明一致；
- `application.yml` 逐行 diff 仅新增一键：

```
+    confirm-token-ttl-seconds: ${OT_PUBLISH_CONFIRM_TOKEN_TTL_SECONDS:300}   # 二次确认凭据 TTL（秒，TASK-0015）：影响预览 previewToken 的 Redis 有效期（键 iot:pub:confirm:{tenantId}:{scopeType}:{scopeId}:{targetVersion}，value=待发布内容摘要哈希）；二档模板变更发布校验 token 存在+内容摘要一致，缺失/过期/内容漂移报 5002 下发未确认
```

- 测试 7 文件（新增 5：tier 表/diff 引擎/版本历史服务/确认门 gate + 控制器用例；适配 2：模板服务、数据项服务测试注入版本服务 mock）；
- 升级 SQL 1 文件（头注登记「附录 A 无此表——ADR-0008 落地件新增」上游差异；无菜单种子——影响预览复用 iot:template:query，4025 既有）。

---

## 二、每条验收命令原始输出

### 验收 1：单测全量（BUILD SUCCESS、0 失败 0 错误）

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`（在最终提交代码 b3d1e0c 上完整重跑）。

ot-core install 段：

```
[INFO] BUILD SUCCESS
```

ot-admin 测试 Results 段（全量日志尾段）：

```
[INFO] Results:
[INFO]
[INFO] Tests run: 2286, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:31 min
[INFO] Finished at: 2026-11-23T02:24:35+08:00
```

**通过**：2286 通过 / 0 失败 / 0 错误 / 0 跳过（0026 交付时 2244，本任务净增 42：新测试 36 + 既有测试适配拆分）。

**TDD 红转绿留痕**（brief 测试口径）：先行落 4 个新测试类（tier 表 9 + diff 引擎 8 + 版本历史服务 12 + gate 7 = 36 用例），服务/gate 以 UnsupportedOperationException stub 阶段首跑为红——

```
[ERROR] Tests run: 7, Failures: 0, Errors: 7, Skipped: 0, Time elapsed: 1.713 s <<< FAILURE! -- in net.otlab.admin.module.iot.publish.gate.TemplateAwarePublishConfirmGateTest
[ERROR] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0 ... <<< FAILURE! -- in net.otlab.admin.module.iot.template.service.impl.IotTemplateVersionServiceImplTest（Errors: 12）
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.template.tier.TemplateContentDiffEngineTest（纯函数引擎直接绿）
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.template.tier.TemplateTierRuleTableTest（同上）
```

真实实现交付后目标 4 测试类全绿：

```
[INFO] Tests run: 7,  Failures: 0, Errors: 0, Skipped: 0 -- in ...TemplateAwarePublishConfirmGateTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0 -- in ...IotTemplateVersionServiceImplTest
[INFO] Tests run: 8,  Failures: 0, Errors: 0, Skipped: 0 -- in ...TemplateContentDiffEngineTest
[INFO] Tests run: 9,  Failures: 0, Errors: 0, Skipped: 0 -- in ...TemplateTierRuleTableTest
[INFO] Tests run: 36, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

红转绿期间修复均附根因与「为什么现在对了」：

1. **测试桩 mock 方法不匹配**（recordContentChangeNoChange/FormulaTier2/Mixed 红）：实现契约=最新历史行走 `selectOne`（orderByDesc+LIMIT 1），测试误桩 `selectList` → mock 返回 null 走了「补录基线」分支。为什么现在对了：桩对齐真实查询路径后，「无变化返回 null 不升版」「有变化升版+落行」两分支如实命中实现逻辑。
2. **预览 diff 误计全量 ADDED**（previewImpact 两用例红）：无未锁定跟随实例（minApplied=null）或历史基线缺失时 `diff(null, cur)` 产出全量 ADDED。为什么现在对了：该场景本就无从对比——diff 留空（FEAT-0005 异常边界「空数据=模板无实例提示即可」），实现改为 `minApplied!=null && baselineNode!=null` 才进 diff，锁定滞后归待升级清单（0016）语义如实呈现。
3. **gate WARN 日志占位符参数错位**（冒烟实测 `tenant=0 scope=2-2 targetVersion={}`）：三处 warn 缺 tenantId 实参（83b99c3）。为什么现在对了：占位符与实参一一对应，证据日志不再误导排障。
4. **sortBy 泛型通配编译错**：`Function<T, ? extends Comparable<?>>` 不满足 Comparator.comparing 推断。为什么现在对了：改标准惯用法 `<T, U extends Comparable<? super U>>`，Integer/String 两种键（priority/dataCode）均编译通过。

### 验收 2：升级脚本二次幂等 + 表结构核对

命令：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261123_iot_template_version.sql` 连续执行两次。

```
===== RUN 1 =====
OK(0)
OK(0)
DONE
===== RUN 2 =====
OK(0)
OK(0)
DONE
```

表结构核对（`SHOW CREATE TABLE iot_model_template_version` 尾段）：

```
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_template_version` (`tenant_id`,`template_id`,`version`),
  KEY `idx_template` (`tenant_id`,`template_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='模板版本历史表（ADR-0008 落地件）'
```

**通过**：两次 DONE（CREATE TABLE IF NOT EXISTS 幂等重跑零变更）；uk_template_version 与 brief DDL 逐字一致（COLLATE 为 MySQL 8 默认补齐，脚本与 brief 均未写 COLLATE）。无菜单种子（头注登记：影响预览复用既有 iot:template:query 权限 4025）。

### 验收 3：dev 冒烟全序列（每步原始输出）

dev 档启动（`mvn spring-boot:run` 默认 dev）：`Started OtAdminApplication in 10.387 seconds`，启动日志 `grep -c " ERROR "` = **0**，Tomcat 8098。登录走 `/auth/login/test`（dev 专用，code 200，token=9b715ba3-…）。

**步骤 1-2 建链（模板 + 数据项 + 三实例）**

```
STEP1 POST /iot/template {"templateCode":"SMOKE0015_TPL","nodeType":4}
{"code":200,...}  —— 模板 id=10，version=1；版本历史基线行即时落库（见步骤 2 后核对）
STEP2 POST /iot/template/item（templateId=10，ELEC_CONS 直绑 + LOAD_RATE 公式 "{SELF.ELEC_CONS}/100"）
{"code":200,"data":2,...}
DB 核对：
SELECT id,tenant_id,template_id,version,tier_summary,create_time FROM iot_model_template_version WHERE template_id=10 ORDER BY version
1  1  10  1  null    2026-11-23 02:14:24   ← 建模基线（tier_summary=NULL）
2  1  10  2  T1:2    2026-11-23 02:14:30   ← 新增两数据项自动升版（一档×2）
SELECT id,version FROM iot_model_template WHERE id=10
10  2                                      ← 模板 version 自动递增 1→2
STEP2b 三台设备实例（parentId=0，templateId=10）→ 20/21/22：
SELECT id,instance_code,template_id,applied_template_version,locked,status FROM iot_model_instance ...
20  EQ1A  10  2  0  1
21  EQ1B  10  2  0  1
22  EQ1C  10  2  0  1                       ← applied=模板当前 version（TD-0008 语义在位）
```

**步骤 3：T2 变更——改 LOAD_RATE 默认公式（item 17，"/100" → "/1000"）**

```
PUT /iot/template/item → {"code":200}
DB：版本历史新增行 3 | T2:1（02:14:53）；模板 version 自动 2→3
服务日志：[TEMPLATE-VERSION] template upgraded templateId=10 2 -> 3 summary=T2:1
          entries=[dataItem[LOAD_RATE] CHANGED.defaultFormula {SELF.ELEC_CONS}/100 -> {SELF.ELEC_CONS}/1000 (T2)]
```

**步骤 4：影响预览**

```
GET /iot/template/impact-preview?templateId=10
{"code":200,"data":{"affectedInstanceCount":3,"confirmScopeId":0,"confirmScopeType":2,
 "confirmTokenTtlSeconds":300,"currentTemplateVersion":3,
 "diff":[{"objectType":"dataItem","objectKey":"LOAD_RATE","changeType":"CHANGED",
   "field":"defaultFormula","oldValue":"{SELF.ELEC_CONS}/100","newValue":"{SELF.ELEC_CONS}/1000","tier":"T2"}],
 "lockedInstanceCount":0,"previewToken":"441bb7a9e8484cce927f791ee254150a",
 "targetVersion":2,"templateId":10,"tierSummary":"T2:1"},...}
—— 受影响实例数 3（引用且启用）+ diff 逐条目（对象类型/业务键/字段/旧值/新值/档位）+ previewToken，
   targetVersion=计算域末版+1（确认凭据锚定）
```

**步骤 5：不带 token 发布 → 5002**

```
POST /iot/publish/publish {"scopeType":2,"scopeId":0}
{"code":5002,"data":null,"message":"发布未通过确认门：存在待确认的第二档模板变更，请先获取影响预览并二次确认
（token 缺失/过期/内容漂移——重新预览后携 confirmToken 发布）",...}
服务日志：[TEMPLATE-VERSION] pending tier2 detected tenant=1 templateId=10 version=3 laggingInstances=3
          [CONFIRM-GATE] publish rejected (token missing) ...
```

**步骤 6：带 token 发布 → 200**

```
POST /iot/publish/publish {"scopeType":2,"scopeId":0,"confirmToken":"441bb7a9e8484cce927f791ee254150a"}
{"code":200,"data":{"id":29,...,"version":2,"snapshotHash":"4d5e05b141a95b9548c258fe7d5a30284382713c2a4f89af9298b551f4d55e85",
 "snapshotUri":"1/2-0-2.json","publishStatus":1,"publishTime":"2026-11-23 02:15:16",...}}
—— version=2 与预览 targetVersion 一致；服务日志 [CONFIRM-GATE] tier2 publish confirmed tenant=1 scope=2-0 targetVersion=2
Redis 凭据实测（docker exec wisfactory-redis redis-cli，键带框架前缀 ot_framework:）：
  --scan --pattern '*confirm*' → ot_framework:iot:pub:confirm:1:2:0:4 / ot_framework:iot:pub:confirm:1:2:0:2
  ttl 'ot_framework:iot:pub:confirm:1:2:0:4' → 276（TTL 300 生效中）
  get 'ot_framework:iot:pub:confirm:1:2:0:4' → "3278d8faba85410c0505afe74cf724a473517cf149a0b2167072383c6906915f"（待发布内容摘要哈希）
```

**步骤 7-8：T1 变更（展示属性改名）→ 无 token 发布直接 200**

```
PUT /iot/template/item（item 16 dataName 改名）→ {"code":200}
DB：版本历史行 4 | T1:1；模板 version 3→4
POST /iot/publish/publish {"scopeType":2,"scopeId":0}   ← 无 confirmToken
{"code":200,"data":{"id":30,...,"version":3,"snapshotHash":"b89df2060ed1a23446e6d624b5588230f7608e1d9111f5e0337c22bd05ab5b2e",...}}
—— T2 已随步骤 6 经确认发布落地（窗口关闭），一档变更无 token 直接放行（ADR-0008 一档政策）
```

**步骤 9：锁一台实例 → 预览 lockedInstanceCount 分列**

```
SQL 置位 locked=1（锁定开关 API 归 TASK-0016，0015 无该端点——见「四」7）：
UPDATE iot_model_instance SET locked=1 WHERE id=22 → OK(1)
GET /iot/template/impact-preview?templateId=10
{"code":200,"data":{"affectedInstanceCount":3,...,"lockedInstanceCount":1,...,
 "diff":[{"...ELEC_CONS dataName ... T1"},{"...LOAD_RATE defaultFormula ... T2"}],
 "tierSummary":"T1:1,T2:1","targetVersion":4,...}}
—— 分列正确：affected=引用且启用总数（3），locked=locked 子集单列（1）；
   diff 为未锁定跟随实例有效版本（v2）→ 当前内容（v4）的累计待生效变更（一档+二档并存）
```

**验收 3 全序列通过。**

### 验收 4：prod 档真实启动回归

启动（本地回归口径，环境变量注入本地连接与密钥）：

```
cd backend/ot-admin && DB_HOST=127.0.0.1 DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=127.0.0.1 REDIS_PORT=6379 REDIS_DATABASE=0 SM4_KEY=0123456789abcdef \
CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 WECHAT_MP_ENABLED=false \
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

（首轮启动后探针自己打错 captcha 字段产生一条业务 ERROR（auth.captchaBlank，探针笔误非系统缺陷）——按 0026 报告先例，修正字段后**干净完整重跑**，本轮为准。）

干净重跑启动日志：

```
2026-11-23T02:21:02.539+08:00  INFO ... The following 1 profile is active: "prod"
2026-11-23T02:21:02.539+08:00  INFO ... Started OtAdminApplication in 9.995 seconds (process running for 10.376)
ERROR_COUNT=0 （`grep -c " ERROR "` = 0——含启动与全部探针，全程无 ERROR）
```

prod 下认证走正式链（`GET /auth/captchaImage` → uuid=4337834e-… → WSL redis 读 `captcha:code:<uuid>` 码 3CTE → SM4 传输密文（hutool SM4 encryptHex(admin123) 再 Base64）→ `POST /auth/login` 字段 `captcha`）登录成功（token 2f8467b9-…）。

prod 各端点断言（本任务新增端点与配置在 prod 生效——部署/配置类加严条款：application.yml 新键与升级 SQL 均被触及，previewToken 实测落 Redis 带 TTL）：

```
GET /iot/template/impact-preview?templateId=10
{"code":200,"data":{"affectedInstanceCount":3,...,"lockedInstanceCount":1,"previewToken":"bc217731b82a4006b98fd3edb79579a4",
 "targetVersion":4,...}} —— 新端点 200（版本历史/实例计数/Redis 发token/组装哈希全链在 prod 生效）
GET /iot/publish/list?scopeType=2&scopeId=0&pageNum=1&pageSize=5
{"code":200,"data":{...,"records":[{"id":30,...,"version":3,...},{"id":29,...,"version":2,...},...],...}} —— 200
```

**通过**：prod 档真实启动 0 ERROR + 正式登录链 + 0015 新端点/既有发布端点全 200。验毕后 backend 已停止（8098 监听确认归零，纪律保持）。

### 验收 5：改动范围核对

见「一、改动文件清单」——`git diff --name-only d02ac44..HEAD`（= 本任务三提交 fa064cd + 83b99c3 + b3d1e0c 的净足迹）29 文件全部落在 brief 声明范围内；ot-core 触点仅 ResultCode.java 一处；yml 仅新增一键；无 publish 其余端点行为改动、未碰前端/ingest/openapi/dataperm，**通过**。

---

## 三、未做项与原因

1. **TDengine**：按协调者指令本批不涉；本任务链路（模板域写路径/影响预览/发布确认门）数据源为 MySQL 业务表 + Redis，冒烟与 prod 回归均未触 TDengine。
2. **实例升级链（未锁定实例发布后数据面自动跟随）**：归 TASK-0016（其任务正文「待升级清单页逐台 diff、逐台确认升级」及 REQ-0006 验收 3/4 的逐台语义）。本任务按 brief 交付面止步于「分档判定/影响预览/确认门」——发布不推进实例 applied_template_version，快照按实例有效版本组装天然不包含未升级实例（0024 既有语义，brief 明文「天然正确，gate 不重复处理」）。影响与口径见「四」6。
3. **锁定版本开关 API**：TASK-0016 交付项（0012 只建列不给开关端点）——冒烟「锁一个实例」以 SQL 置位模拟（见「四」7）。
4. **前端**：brief 明文不碰——未动任何 frontend 文件。
5. **菜单种子**：无新增（影响预览复用 iot:template:query），脚本头注已登记。

## 四、偏离计划之处

1. **范围核对基准修正（并行道事实，非本任务改动越界）**：开工 BASE=f195b58 记录在案；TASK-0070 前端批次 d02ac44 在本任务首个提交前落入 main（`git log` 实证 fa064cd parent=d02ac44）。本任务净足迹核对基准相应取 d02ac44..HEAD（29 文件）；`f195b58..HEAD` 中出现的 8 个前端文件全部属 TASK-0070，与本任务无关（本任务未 stage/commit 过任何 frontend 文件）。
2. **IotPublishController javadoc 文案改动已回退（b3d1e0c）**：publish 端点 @Operation 摘要曾随接线更新为「确认门真实现」文案，范围核对发现该文件不在 brief 声明清单内——按「越界即违约」回退至 d02ac44 原文，遗留登记见「五」2。该回退仅注释文案、零行为影响，回退后最终代码上重跑验收 1（2286/0F/0E）以实测背书。
3. **模板升版语义演进（既有行为的定向收窄）**：updateTemplate 原语义「请求携带的 version 按请求值落库」改为「version 不进更新集、由版本历史服务按 diff 自动升版（末版+1）」——依据 REQ-0006 业务能力「模板 version 字段递增留痕」与 brief「模板保存升版时与上一版本 content JSON 对比」的升版留痕要求；请求手填跳号/回退会破坏快照链（uk_template_version 单调）。既有测试 testUpdateTemplate 按新语义改写并在用例 DisplayName 登记依据。insertTemplate 的请求 version 语义不变（首版落请求值，基线快照随其版本落历史行）。
4. **T2 待确认窗口的实现载体（brief 未规定，无 schema 变更前提下的取舍）**：brief 固定 DDL 无「发布位图」列——窗口判定用时间戳（版本历史行 create_time vs 计算域末版 publish_time；行 create_time 晚于末版 publish_time 或计算域从未发布 → 窗口开启）。datetime 秒精度、同秒边界按「已发布」处理——误差窗 1 秒且发布必经确认，对主语义无影响（登记观察项）。
5. **影响预览确认凭据锚定计算域 (scopeType=2, scopeId=0)**：brief 未明示预览（templateId 维度）与 token 键（scope 维度）的映射——按装载口径推定：模板内容（数据项/公式）只经计算域快照进入发布链，网关域快照仅点位/通道（ConfigSnapshotAssembler 装载规则），故确认凭据唯一锚定面=计算域；gate 对网关 scope 恒放行。javadoc 与本报告双登记。
6. **T1/T2「随发布自动生效」在 0015 的语义边界**：本任务交付后，T2 变更的生效路径=「影响预览 → 二次确认 → 发布落地（窗口关闭）」；实例数据面的跟随（按新版本重生成数据项、applied_template_version 推进）归 TASK-0016 承载。该边界下 brief 冒烟序列完全成立（T2 确认发布后窗口关闭 → T1 无 token 直发 200）；REQ-0006 验收 1/2 的实例侧最终生效时点由 0016 收口——已作为 0016 联动注意项登记（「五」1）。
7. **冒烟锁定开关以 SQL 置位**：`UPDATE iot_model_instance SET locked=1 WHERE id=22`（0015 无锁定端点，0016 交付项）——预览分列逻辑对 locked 的读取路径经此实测。
8. **提交粒度**：三个提交（实现 fa064cd / 日志修正 83b99c3 / 范围回退 b3d1e0c），均注明 TASK-0015。

## 五、遗留问题

1. **[0016 联动注意项（重要）]** 本任务确认门的 T2「待确认」窗口以「该变更已经过一次计算域发布」关闭；TASK-0016 实现实例升级链时需评估：若发布后未锁定实例自动跟随（推进 applied_template_version + 重生成数据项），窗口语义无需改动（窗口关闭条件与实例滞后判定二者取交即可）；若跟随发生在发布事务内，注意确认门调用点在组装/落库之前——传播动作应置于版本落库之后同事务或 afterCommit，勿回灌进 gate 判定。
2. **[遗留登记]** IotPublishController publish 端点 @Operation 摘要仍载「确认门 NoOp 占位，TASK-0015 交付后接线」旧文案——文件不在 brief 声明范围（本任务回退未动），建议修订轮一行更新。
3. **[遗留登记]** 状态规则 CRUD 不触发模板升版：statusrule 包不在 brief 声明范围（仅 template/** 可动）。规则状态已纳入内容快照（brief DDL content 注释「数据项/状态规则等」）且 diff 引擎/规则表已映射状态规则档位（conditionExpr/priority/enabled/bindingMap=二档、statusName/color=一档、新增=一档、删除=二档）——规则改动会在下一次模板头/数据项变更升版时以 diff 条目浮出；独立触发钩子建议随状态规则域修订轮或 0016 接线。
4. **[观察项]** T2 窗口时间戳秒精度：save 与计算域 publish 同秒边界按「窗口已关闭」处理（发布必经确认，主语义不受影响）；如需绝对精确可在 0016 给历史行补发布锚定列（需改 DDL，二期评估）。
5. **[观察项]** 存量模板（0015 之前建库）无版本历史基线：影响预览 diff 留空（无从对比、不误计全量 ADDED），首次编辑自动补录基线快照——登记为已知降级。
6. **[观察项]** 冒烟数据留存库内（模板 10 / 实例 20-22 / 发布版本 29-30 / 版本历史行 1-4），对应快照文件（backend/ot-admin/data/snapshots/1/2-0-2.json、2-0-3.json）未清理（.gitignore 覆盖、与 DB 行引用一致性优先，照 0024/0025 冒烟遗留同口径隔离）。
7. **[环境]** 同 0024/0025/0026 报告登记的 WSL2 空闲关停问题（本批以保活循环规避，全程未复发）。

## 六、一句话结论

**通过**——验收 1（最终提交代码 b3d1e0c 全量 mvn：2286 测试 0 失败 0 错误 BUILD SUCCESS，TDD 红 19E→绿 36 目标测试留痕，4 项修复均附根因）、验收 2（升级脚本两次 DONE、uk_template_version 与 brief DDL 逐字一致）、验收 3（dev 冒烟全序列：模板 T2 改公式自动升版 v3/T2:1 → 影响预览 affected=3/diff T2/previewToken/targetVersion=2 → 无 token 发布 **5002** → 带 token 发布 **200**（version 与 targetVersion 一致、Redis 凭据键 ot_framework:iot:pub:confirm:1:2:0:2+4 与哈希实测在位、TTL 生效）→ T1 改名自动升版 v4/T1:1 → 无 token 发布直接 **200** → 锁一台实例预览 lockedInstanceCount=1 分列正确）、验收 4（prod 档干净启动全程 0 ERROR + 正式登录链 + impact-preview/publish/list 全 200）、验收 5（29 文件净足迹全落 brief 声明范围、ot-core 触点仅 ResultCode 一处、yml 仅新增一键）五条全部以原始输出为证通过；偏离均登记（并行道范围基准修正、越界 javadoc 回退、升版语义演进、窗口/锚定口径推定），遗留仅登记级与 0016 联动注意项，无阻断缺陷。

---

---

# 修订轮报告（TASK-0015 修订：diff 评审 I-1/I-2 退回 + M-1/M-2 顺手修 + M-3/M-4 登记）

> 修订执行者：TASK-0015 修订轮；修订日期：2026-11-23；工作区 D:\workspace\WisFactory（main 分支）。
> 输入：`TASK-0015-review.md`（I-1/I-2 完整论证）+ 协调者裁决（修订范围 = 原 brief 范围 + `iot/statusrule/**` 写路径钩子；M-1 连同 I-1 一并修；M-2 顺手修；M-3 登记 deferred；M-4 裁决「模板启停不产生版本行——javadoc/测试固化该语义」）。
> 修订 BASE = **e1b6266**（开工时 `git rev-parse --short HEAD`；TASK-0027 Go 道提交已先期落 main）；修订提交 = **cebf17a**（8 文件，`git diff --name-only e1b6266..cebf17a` 全落授权范围）。并行事实：工作区尚有 ingest/** TASK-0027 在途未提交改动——本批**未触碰**，提交逐文件显式 add（未用 git add -A）。

## 一、改动文件清单（修订轮 8 文件）

`git diff --name-only e1b6266..cebf17a` 原始输出：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/gate/TemplateAwarePublishConfirmGate.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/service/impl/IotStatusRuleServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/IotTemplateVersionService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/impl/IotTemplateVersionServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/tier/TemplateTierRuleTable.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/gate/TemplateAwarePublishConfirmGateTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusrule/service/impl/IotStatusRuleServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/template/service/impl/IotTemplateVersionServiceImplTest.java
```

范围核对：5 个 main 文件 = template/service+impl、template/tier（原 brief 范围内）+ publish/gate（原 brief 范围内）+ statusrule/service/impl（修订授权扩展点——仅 IotStatusRuleServiceImpl 一个文件、纯写路径钩子+autowired+javadoc，未触 controller/dto/mapper）；无 SQL/yml/ot-core 改动。publish 其余端点、前端、ingest、openapi、dataperm 零触碰。

## 二、修复内容与「为什么现在对了」（每条附 TDD 红→绿）

### 1. I-1 内容漂移校验对模板域变更不生效（Important，必须修）——凭据哈希改「计算域+模板域」复合锚定

**修法（裁决选项 b）**：新增 `IotTemplateVersionService.computeConfirmCredentialHash(tenantId, scopeType, scopeId, targetVersion)`——复合哈希 = SHA-256( canonical{ "compute": 计算域快照摘要（原链路不动）, "template": 模板域内容摘要（新：租户内全部模板的 buildContentJson canonical 快照按 templateCode 索引再摘要） } )。预览下发侧（previewImpact 写 Redis）与确认门校验侧（gate）**共用同一方法同参重算比对**。为什么现在对了：旧实现的凭据哈希只锚定计算域快照（装载源=实例行），而模板域编辑（模板头/数据项/状态规则）不触实例行，哈希对模板域变更天然不敏感——预览→再改 T2→TTL 内旧凭据过门；复合锚定后模板域任何内容变更（增删模板/数据项/规则，含逻辑删）都改变模板域摘要 → 哈希漂移 → fail-closed 拒绝（5002），第二个 T2 变更必须重新预览+确认才可发布。取舍：租户级锚定面（任一模板变更即旧凭据失效，宁可多一次预览——与 hasPendingTier2Confirm 的租户粒度判定同口径，fail-closed 优于精准失效）；启停不入模板域快照（M-4 裁决，见 §二.5——启停不使凭据失效）。

**红**（`IotTemplateVersionServiceImplTest.credentialHashChangesWhenTemplateDomainChanges`）：

```
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
org.opentest4j.AssertionFailedError: 模板域内容变更必须使预览凭据哈希漂移（I-1 内容漂移盲区）
  ==> expected: not equal but was: <b49d038497372bd4674a8e6a72a2d3680508d663e8245a9c21e04f520a2ac163>
```

（同一次预览间隔改公式后再预览，两凭据哈希逐字节相同——漂移盲区实证。）**绿**：同测试通过（类 17/17，见 §三.2）。

### 2. M-1 confirmToken 值不被校验（Minor，与 I-1 一并修）——凭据值绑定 token，实质校验

**修法**：凭据值协议 `value = 复合哈希 + ":" + previewToken`（哈希/token 均十六进制，以最后一个 ':' 分段；畸形值——旧协议裸哈希/人工构造——一律 fail-closed 拒绝）。gate 四步校验：token 非空 → 凭据存在 → 复合哈希一致 → **token 匹配**。为什么现在对了：旧实现只判 confirmToken 非空，凭据存在即放行，token 无绑定权重（brief 协议继承弱点）；token 并入凭据值后，发布回传 token 必须与预览下发 token 逐字节一致——同一 targetVersion 的后一次预览覆盖凭据值，旧 token 随之失效，影响预览成为确认凭据的唯一签发点。

**红**（`TemplateAwarePublishConfirmGateTest.pendingTier2ArbitraryTokenRejected`，旧协议下编译可跑）：

```
[ERROR]   TemplateAwarePublishConfirmGateTest.pendingTier2ArbitraryTokenRejected:133 expected: <false> but was: <true>
```

（凭据存在、token="ARBITRARY-TOKEN" 与签发 token 无关，旧 gate 返回 true。）**绿**：gate 测试 9/9（含 token 不匹配拒、畸形凭据拒、复合哈希一致+token 匹配绿，见 §三.2）。

### 3. I-2 状态规则写路径整类绕过版本留痕与确认门（Important，必须修，范围扩展授权）——四写原语钩升版

**修法**：`IotStatusRuleServiceImpl` 注入 `IotTemplateVersionService`，四个写原语**落库后同事务**钩 `recordContentChange(templateId)`（与模板头/模板数据项写路径钩子同事务语义——升版/基线失败整体回滚）：① createRule（新增规则=对象级一档）；② updateRule（conditionExpr 门限/enabled 启停=二档、statusName/color=一档——TemplateTierRuleTable 既有映射）；③ saveBindingMap（bindingMap 直绑=二档）；④ deleteRules（删除规则=二档；批内跨模板 distinct 逐模板留痕，同 deleteItems 范式）。为什么现在对了：规则内容早已纳入 content 快照、diff 引擎与规则表档位映射（0015 交付面），但独立写路径只改 iot_status_rule 不升版——判档/确认门永远看不到；钩上后规则变更产生版本历史行（tier_summary 含 T2），hasPendingTier2Confirm 两条件（T2 行窗口开启 + 滞后跟随实例）链路自通，发布触发确认门——ADR-0008 T2「状态规则门限/优先级」与 REQ-0006 验收 2「覆盖全部 T2」闭环。校验失败路径不钩（未落库无内容变更，测试锁定）。

**红**（`IotStatusRuleServiceImplTest` 新增 5 例；钩子字段尚不存在，该类 31+5=36 例全量 setUp 报错=接线缺失实证）：

```
[ERROR] Tests run: 61, Failures: 5, Errors: 36, Skipped: 0
[ERROR]   IotStatusRuleServiceImplTest.setUp:117 ? IllegalArgument Could not find field
        'templateVersionService' of type [null] on target object
        [net.otlab.admin.module.iot.statusrule.service.impl.IotStatusRuleServiceImpl@...] （36 例同型）
```

**绿**：36/36（新增 5 例：create/update/bindingMap/delete 钩子断言 + 校验失败不钩断言）。

**版本行→确认门链路回归**（`IotTemplateVersionServiceImplTest.statusRuleConditionChangeProducesTier2Row`）：基线含规则（门限 X）→ 改门限（Y）→ recordContentChange 产生版本行 tier_summary=**T2:1**、条目 objectType=statusRule/field=conditionExpr/tier=T2；gate 侧 hasPendingTier2Confirm 对 T2 行+滞后实例返回 true 已由既有 pendingTier2Detected 锁定——「状态规则 T2 变更→版本行产生→发布触发确认门」链路成立。

### 4. M-2 同模板并发保存撞 uk_template_version 裸 500（Minor，顺手修）——映射 4002

**修法**：版本历史行落库收口为 `insertVersionRow` 助手（recordBaseline / 无基线补录 / recordContentChange 主插入三处共用），捕获 `DuplicateKeyException` → `BusinessException(4002, "模板版本并发保存冲突，请重试: templateId=…")`。为什么现在对了：uk_template_version (tenant_id, template_id, version) 物理唯一兜底被并发保存触发时，败者原本以原始 DuplicateKeyException 冒 500（事务回滚数据安全不变，但响应不友好）；映射后与同域 insertTemplate/saveItems 的 4002 兜底范式一致——败者整事务回滚语义不变，仅响应面从 500 变 4002。

**红**：`[ERROR]   IotTemplateVersionServiceImplTest.recordContentChangeDuplicateVersionMapsToDuplicateCode:497 Unexpected exception type thrown, expected: <net.otlab.core.exception.BusinessException> but was: <org.springframework.dao.DuplicateKeyException>` → **绿**（4002+文案断言过）。

### 5. M-4 模板启停（裁决）——启停不产生版本行，javadoc/测试固化语义

**落地**：`status` 移出 buildContentJson 模板头（启停是运营动作非内容变更——内容快照只覆盖内容面，启停轨迹由模板行 status 列承载）；`TemplateTierRuleTable` 保留 template.status→T1 映射并注明兼容用途（0015 首版历史行 content 仍含 status 键——下次编辑 diff 出的 status 条目按一档计，**防误触未登记字段 fail-closed 的二档默认**而误开确认门）；buildContentJson/服务类 javadoc 固化语义。为什么现在对了：启停经 updateTemplate 钩 recordContentChange，旧快照含 status 时每次启停 diff 出 T1 条目产生版本行（噪声+放大 hasPendingTier2Confirm 扫描面）；移出后启停前后内容 JSON 逐字节一致 → 「无变化零版本噪声」既有语义直接兜住——启停不升版、不产行、不触发确认门，也**不使预览凭据失效**（与裁决「运营动作」定性一致）。

**红**（两例）：

```
[ERROR]   IotTemplateVersionServiceImplTest.statusToggleAloneProducesNoVersionRow:460
  启停不入内容快照——启停不产生版本行 ==> expected: <null> but was:
  <[TemplateChangeEntryVO(objectType=template, objectKey=AIR_COMPRESSOR, changeType=CHANGED, field=status, oldValue=1, newValue=0, tier=T1)]>
[ERROR]   IotTemplateVersionServiceImplTest.statusToggleMixedWithFormulaChangeOnlyReportsFormula:483
  启停不入内容快照——混合保存只报公式条目 ==> expected: <1> but was: <2>
```

**绿**：仅启停 → 返回 null/零新行/零升版；启停+改公式混合 → 仅公式条目（T2:1，无 status 条目）。

## 三、验收命令原始输出（修订轮口径=全量 mvn Results 段）

### 3.1 全量回归

```
cd backend && mvn install -pl ot-core -DskipTests    → BUILD SUCCESS
cd backend && mvn test -pl ot-admin                  → Results 段原始输出：

[INFO] Results:
[INFO]
[INFO] Tests run: 2298, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:01 min
[INFO] Finished at: 2026-11-23T02:47:57+08:00
```

2298 = 原基线 2286 + 修订轮净增 12（版本历史服务 +5：漂移锚定/启停×2/4002/状态规则 T2 版本行；状态规则服务 +5：四钩子+失败不钩；gate 净 +2：token 不匹配拒/畸形凭据拒）。0024/0025/0026/0015 既有发布链测试全绿（含 IotPublishServiceImplTest 的 5002 路径、ConfigSnapshotAssembler/SnapshotStorage 面回归）。

### 3.2 目标测试类明细（surefire 实测）

```
IotTemplateVersionServiceImplTest        tests="17" errors="0" failures="0"
IotStatusRuleServiceImplTest             tests="36" errors="0" failures="0"
TemplateAwarePublishConfirmGateTest      tests="9"  errors="0" failures="0"
```

### 3.3 修订提交后差异说明

全量 2298 绿运行于 d55f59c；其后仅一处清理（gate 删除未使用 import，无行为面），gate 类复测 9/9 绿后并入最终提交 **cebf17a**（`git commit --amend`，提交前未推送、无远端影响）。

## 四、未做项与原因

1. **dev/prod 冒烟未重跑**：修订轮契约验收口径=全量 mvn Results 段（未列冒烟）；且凭据协议两端（预览下发/门校验）由**同一方法** computeConfirmCredentialHash 承载、单元测试两侧锁定，集成面无新增分歧点。如 scoped 复评需要 live 冒烟（建议序列：模板 T2 变更+状态规则门限变更 → 预览 → 无 token 发布 5002 → 错误 token 发布 5002 → 正确 token 发布 200 → 预览后改另一模板再发布 5002→重预览→200），按原报告「验收 3」环境口径（WSL mysql/redis、8098 启动前探测用毕即停）可复跑。
2. **遗留登记 #2（IotPublishController 旧 javadoc 文案）未动**：「publish 其余端点不碰」为修订轮明文；仍留原报告遗留清单。
3. **I-1 裁决选项 a（token 校验时复合核对模板头 version/content）未采用**：采用选项 b（复合哈希）——一处实现两侧共用、无凭据比对之外的第二条核对链；理由见 §二.1。

## 五、偏离计划之处

1. **M-4 执行口径说明（需协调者知悉）**：评审 M-4 描述的**现状**是「启停计入内容快照——每次启停产生 T1 版本行」；裁决文字「模板启停不产生版本行（启停是运营动作非内容变更）——javadoc/测试固化该语义」与现状相反。本轮按**裁决语义落地**（status 移出内容快照+测试固化）——将「不修」读作「不按评审建议的表驱动映射方式修，按裁决语义落地」；若裁决本意为「维持现状仅登记」，本轮落地即偏差，回退只需恢复 buildContentJson 的 header.status 一行+两例测试（其余修复不受影响）。
2. **gate 日志 scope 文案**：拒绝日志由「scope={}-{}」泛化格式改为「scope=2-{}」计算域定式（gate 仅计算域走到凭据校验段，网关段已提前返回）——纯日志面，无协议影响。
3. **凭据值协议变更**：Redis value 由裸哈希改为「复合哈希:token」。无迁移需求：TTL 300s 自然过渡，旧格式凭据按畸形 fail-closed 拒绝、重取预览即恢复；发布侧 PublishForm.confirmToken 字段与 5002 语义零变化。

## 六、遗留问题（修订轮登记）

1. **[M-3 deferred 登记]** ADR-0008「显示顺序/可覆盖参数默认值」两类字段暂无 schema 列（iot_data_item_template 现有 13 业务列无 sort/override 列）——规则表对其 T1/T2 映射当前不可达；content JSON 已覆盖全部现有可变列、未知字段 fail-closed 默认 T2 可兜未来新增列。**防遗忘**：未来任一模板域加列时，必须同步纳入 buildContentJson content 快照与 TemplateTierRuleTable 映射（加列忘快照=变更不可见=分档失效），并连带评估确认凭据模板域摘要与影响预览 diff 面。
2. **[M-4 取舍登记]** 模板启停语义固化（javadoc+测试）：启停是运营动作非内容变更——不产生版本行、不触发 diff/确认门、不使预览凭据失效；启停轨迹由 iot_model_template.status 列与操作审计承载。取舍代价：版本历史不记录启停轨迹（查「谁何时停用」走操作审计面，不走版本历史）。
3. **[迁移观察]** 0015 首版已产出的版本历史行（含 dev 冒烟数据版本历史行 1-4）content 含 status 键：其模板下一次内容编辑时 diff 将出现一条 status→缺键 的 T1 条目（一次性噪声；T1 不触发确认门、无功能影响；无历史行的存量模板补录基线即用新形态，无此条目）。
4. **[性能观察]** computeTemplateDomainContentHash 为租户级全量装载（每模板 items+rules 两查，N+1）：预览每次调用 + 确认门仅二档路径调用；租户内模板数量级为十位，一期可接受；如成热点可加租户内缓存（失效点=模板域四写路径钩子处）。
5. **[0016 联动注意项]** 状态规则变更现与模板头/数据项变更同样进入 hasPendingTier2Confirm 判定面（租户粒度、T2 行窗口+滞后实例两条件）——0016 实例升级链实现时「窗口关闭条件与实例滞后判定取交」的原注意项对规则类 T2 行同等适用。
6. 其余遗留（原报告「五」1/4/5/6/7 与遗留登记 #1/#2）不变；本修订未新增环境类遗留。

## 七、修订轮一句话结论

**通过**——评审 I-1（模板域漂移盲区）以「计算域+模板域」复合凭据哈希闭合（红：两预览哈希逐字节相同 b49d0384…→绿：漂移即拒）、M-1 token 从仅判非空变实质校验（红：任意 token 返回 true→绿：不匹配/畸形均拒）、I-2 状态规则四写原语钩升版与确认门接线（红：钩子字段缺失 36 例→绿：36/36，规则 T2 变更→版本行 T2:1→确认门链路成立）、M-2 并发撞版映射 4002（红：裸 DuplicateKeyException→绿）、M-4 裁决语义落地（红：启停产 T1 行→绿：零噪声）；全量 mvn **2298 测试 0 失败 0 错误 BUILD SUCCESS**（原始 Results 段见 §三.1），修订足迹 8 文件全落授权范围（`e1b6266..cebf17a`），偏离与取舍全部登记（含 M-4 执行口径说明，待协调者确认知悉）。
