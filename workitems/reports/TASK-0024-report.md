# TASK-0024 验收报告：配置版本快照生成（SHA-256 + 对象存储）

> **中断恢复接手——前一执行者 S1-S6 已过但无报告，本次完整重跑采集证据。**
> 验收执行：恢复执行者；验收日期：2026-11-22；工作区 D:\workspace\WisFactory（main 分支）。
> 实现提交：ab9481a（28 文件）+ 8566f57（.gitignore）；diff BASE = a1c6649。
> 本次对 brief「验收命令」5 条全部完整重跑，未复用前一执行者任何输出。

---

## 一、改动文件清单（验收 5：改动范围核对）

`git diff --name-only a1c6649..HEAD` 原始输出（29 文件）：

```
.gitignore
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/config/PublishProperties.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/controller/IotPublishController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/PublishForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/RollbackForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/dto/SnapshotVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/entity/IotConfigVersion.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/gate/NoOpPublishConfirmGate.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/gate/PublishConfirmGate.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/mapper/IotConfigVersionMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/IotPublishService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/ConfigSnapshot.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/ConfigSnapshotAssembler.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/SnapshotChannel.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/SnapshotCredentialGuard.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/SnapshotDataItem.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/SnapshotFormula.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/SnapshotMetric.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/snapshot/SnapshotPoint.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/storage/SnapshotStorage.java
backend/ot-admin/src/main/resources/application.yml
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/config/IotPublishMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/controller/IotPublishControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/snapshot/ConfigSnapshotAssemblerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/snapshot/SnapshotCredentialGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/storage/SnapshotStorageTest.java
resources/sql/upgrade/20261122_iot_publish_domain.sql
```

**比对结论：通过。** 28 文件（ab9481a）全落 brief 声明范围：`/iot/publish/**` 新包（22 主代码）、`src/test/.../iot/publish/**`（6 测试）、`application.yml`、升级 SQL。`.gitignore`（8566f57）为协调者已认可的环境产物。application.yml 的 diff 经逐行核对**仅新增 `ot.publish.storage-path` 一个配置键**（brief 允许项「仅新增对象存储配置键」，注释声明本地目录一期 + MinIO 二期登记）。

---

## 二、每条验收命令原始输出

### 验收 1：单测全量（BUILD SUCCESS、0 失败 0 错误）

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 段：

```
[INFO] --- install:3.1.4:install (default-install) @ ot-core ---
[INFO] Installing D:\workspace\WisFactory\backend\ot-core\target\ot-core-1.0.0.jar to C:\Users\dowil\.m2\repository\net\otlab\ot-core\1.0.0\ot-core-1.0.0.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  6.691 s
```

ot-admin 测试 Results 段：

```
[INFO] Results:
[INFO]
[INFO] Tests run: 2165, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:46 min
```

**通过**：BUILD SUCCESS、2165 通过 / 0 失败 / 0 错误 / 0 跳过。

### 验收 2：升级脚本二次幂等 + 表结构 + 菜单种子

命令：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_iot_publish_domain.sql` 连续执行两次。

第一次：

```
===== RUN 1 =====
OK(0)
OK(0)
OK(0)
OK(0)
DONE
EXIT1=0
===== RUN 2 =====
OK(0)
OK(0)
OK(0)
OK(0)
DONE
EXIT2=0
```

（脚本含 SET NAMES + CREATE TABLE IF NOT EXISTS + INSERT IGNORE 菜单 + INSERT IGNORE 授权共 4 条语句；表与种子行在首次建库轮已存在，幂等重跑零变更、不报错、两次 DONE。）

表结构核对（`SHOW CREATE TABLE iot_config_version`）：

```
CREATE TABLE `iot_config_version` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '配置版本 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `scope_type` tinyint NOT NULL COMMENT '范围 1 网关 2 计算',
  `scope_id` bigint NOT NULL COMMENT '范围对象 ID',
  `version` bigint NOT NULL COMMENT '版本号（单调递增）',
  `snapshot_hash` varchar(64) DEFAULT NULL COMMENT '快照哈希',
  `snapshot_uri` varchar(500) DEFAULT NULL COMMENT '快照文件地址',
  `publish_status` tinyint DEFAULT '1' COMMENT '状态 1 已发布 2 已回滚',
  `publish_time` datetime DEFAULT NULL COMMENT '发布时间',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_scope_version` (`tenant_id`,`scope_type`,`scope_id`,`version`),
  KEY `idx_scope_status` (`tenant_id`,`scope_type`,`scope_id`,`publish_status`)
) ENGINE=InnoDB AUTO_INCREMENT=17 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='配置版本表'
```

**uk_scope_version 在位**（brief DDL 逐字一致；COLLATE 为 MySQL 8 对未显式指定排序规则的默认补齐，脚本与 brief 均未写 COLLATE）。菜单种子与授权核对：

```
id    menu_name      menu_type  perms
4070  发布版本查询    3          iot:publish:list
4071  快照取回       3          iot:publish:query
4072  配置发布       3          iot:publish:publish
4073  版本回滚       3          iot:publish:rollback
role_id  menu_id
1        4070
1        4071
1        4072
1        4073
```

**通过**：两次 DONE；uk_scope_version 存在；4070-4073 四行 + 超管（role_id=1）授权四行。（AUTO_INCREMENT=17 说明表内有前一执行者冒烟遗留版本行，与本次验收隔离。）

### 验收 3：dev 冒烟全序列

dev 档启动（mvn spring-boot:run，默认 profile dev）：`Started OtAdminApplication in 16.678 seconds`，启动日志 **0 ERROR**，Tomcat 8098。登录走 `/auth/login/test`（dev 专用）。

**步骤 1-4 建链（0017-0023 链：网关+通道+点位+数据项）**

```
STEP1 create gateway
{"code":200,...}   —— 网关 SMOKE0024_GW → id=9109
STEP2 create channel
{"code":200,...}   —— 通道 SMOKE0024_CH1（MQTT，connConfig={broker,topic}）→ id=19，gatewayId=9109
STEP3 create point
{"code":200,...}   —— 点位 SMOKE0024_P1（DOUBLE，min=0 max=100，channelId=19）→ id=12020
STEP4 regenerate data items (instance 13)
{"code":200,"data":0,...}  —— 实例 13（templateId=9）regenerate 200；数据项在位：
{"code":200,"data":[{"id":13,...,"dataCode":"ELEC_CONS","fullKey":"F01.WS13.EQ14A.ELEC_CONS","instanceId":13,...}]}
```

**步骤 5：发布网关 scope → 版本 1**

```
POST /iot/publish/publish {"scopeType":1,"scopeId":9109}
{"code":200,"data":{"id":17,"tenantId":1,"scopeType":1,"scopeId":9109,"version":1,
 "snapshotHash":"c42826551b697c471310b7bfc1f53f81ae6aa191a5bb2f43576f79e88d534934",
 "snapshotUri":"1/1-9109-1.json","publishStatus":1,"publishTime":"2026-11-22 22:18:35",...}}
```

文件存在 + 哈希可验：

```
$ ls backend/ot-admin/data/snapshots/1/
-rw-r--r-- 1 dowil 197609 549 Nov 22 22:18 1-9109-1.json
$ sha256sum backend/ot-admin/data/snapshots/1/1-9109-1.json
c42826551b697c471310b7bfc1f53f81ae6aa191a5bb2f43576f79e88d534934
= 落库 snapshotHash 完全一致
$ cat 快照体（关键段）：
{"schemaVersion":"1.0.1","tenantId":1,"scopeType":1,"scopeId":9109,"version":1,
 "points":[{"id":12020,...,"pointCode":"SMOKE0024_P1",...,"minValue":0.000000,"maxValue":100.000000,...,"configVersion":1}],
 "channels":[{"id":19,...,"channelCode":"SMOKE0024_CH1","protocolType":"MQTT","connConfig":{"broker":"tcp://127.0.0.1:1883","topic":"smoke0024/up"},"status":1,"hasCredential":false}]}
```

（schemaVersion 1.0.1 一并下发；点位/通道按 Schema 装载；凭据派生标记 hasCredential=false；configVersion=1 随发布递增。）

**步骤 6：变更点位 → 再发布 → 版本 2**

```
PUT /iot/point（minValue 0→5，白名单字段）{"code":200,...}
POST /iot/publish/publish {"scopeType":1,"scopeId":9109}
{"code":200,"data":{"id":18,...,"version":2,
 "snapshotHash":"c7bf95f28a0b9d887db37684208db7206e8d1d5ff6fbfce60a1c9afbc7971dd3",
 "snapshotUri":"1/1-9109-2.json",...}}
—— 版本 2 单调生成、哈希随内容变化
```

**步骤 7：回滚到 1 → 版本 3（复用旧快照）**

```
POST /iot/publish/rollback {"scopeType":1,"scopeId":9109,"targetVersion":1}
{"code":200,"data":{"id":19,"version":3,
 "snapshotHash":"c42826551b697c471310b7bfc1f53f81ae6aa191a5bb2f43576f79e88d534934",
 "snapshotUri":"1/1-9109-1.json","publishStatus":1,...}}
—— 新版本行 3 的 hash/uri 与版本 1 完全相同（brief 裁决「新版本行复用旧快照文件」）；
   版本列表 3 条（4/3/2/1 中的 1、2、3——版本 1、2 行未被改写，历史保留）
```

**步骤 8：快照取回与版本 1 哈希一致**

```
GET /iot/publish/snapshot?scopeType=1&scopeId=9109&version=3
{"code":200,"data":{"version":3,"snapshotHash":"c42826551b697c471310b7bfc1f53f81ae6aa191a5bb2f43576f79e88d534934",
 "snapshotBody":"{\"schemaVersion\":\"1.0.1\",...,\"version\":1,\"points\":[...SMOKE0024_P1...minValue\":0.000000...]...}"}}
GET /iot/publish/snapshot?scopeType=1&scopeId=9109&version=1
{"code":200,"data":{"version":1,"snapshotHash":"c42826551b697c471310b7bfc1f53f81ae6aa191a5bb2f43576f79e88d534934","snapshotBody":"...（同上同一快照体）"}}
—— 取回哈希核对通过（无 500 存储损坏告警）；版本 3 取回体 = 版本 1 快照（回滚复用语义正确）
```

**步骤 9：并发发布 → 5001**

```
两线程同 scope(1,9109) 并发 POST /iot/publish/publish：
THREAD1: {"code":200,"data":{"id":20,"version":4,"snapshotHash":"38c486bb93f1398ac8991ced6969898c4306abb89c22ef97407557080b724d64","snapshotUri":"1/1-9109-4.json",...}}
THREAD2: {"code":5001,"data":null,"message":"版本冲突：该范围存在并发发布，请刷新后重试",...}
版本列表 total=4——仅生成一个新版本，uk_scope_version 物理兜底 + 5001 转换生效
```

**步骤 10：凭据关键词终检（塞含凭据行 → 发布组装失败）**

```
SQL 直插毒通道（绕过 API 层 0018 Schema 校验）：
INSERT INTO iot_channel (tenant_id,gateway_id,channel_code,channel_name,protocol_type,conn_config,status,deleted)
VALUES (1,9109,'SMOKE0024_CHPOISON','poison-channel-with-credential','MQTT',
        '{"broker":"tcp://127.0.0.1:1883","topic":"poison/x","mqttPassword":"hunter2secret"}',1,0);  → OK(1)（id=20）
POST /iot/publish/publish {"scopeType":1,"scopeId":9109}
{"code":400,"data":null,"message":"快照终检失败：快照体含凭据关键词（ADR-0006：凭据独立存凭据管理，不入配置快照）:
 channels[1].connConfig.mqttPasswordchannels[1].connConfig.mqttPassword",...}
版本列表 total=4 —— 组装失败 fail-closed，未落库新版本
（毒行验后已 DELETE 清理，gateway 9109 下仅剩正常通道 19）
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

启动日志：

```
2026-11-22T22:28:27.487  INFO ... The following 1 profile is active: "prod"
2026-11-22T22:28:39.308  INFO ... Started OtAdminApplication in 12.654 seconds (process running for 13.462)
ERROR_COUNT=0 （`grep -c ' ERROR '` = 0）
```

prod 下认证走正式链（`GET /auth/captchaImage` → Redis 读码 → SM4 传输密文 → `POST /auth/login`）登录成功（code 200）。

publish/list 断言：

```
GET /iot/publish/list?scopeType=1&scopeId=9109&pageNum=1&pageSize=10
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":20,...,"version":4,...},{"id":19,...,"version":3,...},...],...}}
—— 业务 code 200 + 版本列表数据返回
```

快照取回（补充断言）：

```
GET /iot/publish/snapshot?scopeType=1&scopeId=9109&version=3
{"code":200,"data":{"version":3,"snapshotHash":"c42826551b697c471310b7bfc1f53f81ae6aa191a5bb2f43576f79e88d534934","snapshotBody":"{...}",...}}
```

**通过**：prod 档真实启动 0 ERROR + publish/list 200。部署/配置类加严条款满足：application.yml（ot.publish.storage-path 新键在 prod 生效——快照取回从存储取回成功即为证）与升级 SQL（验收 2 已在库验证）均被触及。验毕后 backend 已停止（8098 释放，纪律保持）。

### 验收 5：改动范围核对

见「一、改动文件清单」——29 文件（git diff --name-only a1c6649..HEAD）全部落在 brief 声明范围内，**通过**。

---

## 三、未做项与原因

1. **TDengine 写路径验收**：按协调者指令不做（192.168.31.145 fd 问题未修复）。本任务验收序列（发布链）不涉及任何 TDengine 写入——快照组装数据源为 MySQL 业务表（iot_point/iot_channel/iot_data_item/iot_metric 等），冒烟与 prod 回归均未触 TDengine，故无因该限制被跳过的本任务步骤。
2. **冒烟步骤 4「建数据项」以 regenerate 对齐口径完成**：实例 13 的模板清单在库内已全部生成（前一执行者冒烟遗留），regenerate 返回 `data:0`（无缺失需新增）且数据项 ELEC_CONS（id=13）在位可查——0013 链接口与数据均在位。网关 scope 快照按 Schema 不装载数据项（数据项归计算 scope），故未再新建重复数据项；计算 scope 发布不在 brief 冒烟序列内（发布网关 scope 为序列指定步骤），未展开。

## 四、偏离计划之处

1. **快照文件落盘位置**：本次冒烟产物在 `backend/ot-admin/data/snapshots/`（前一执行者的在 `backend/data/snapshots/`）。`ot.publish.storage-path` 默认值 `./data/snapshots` 按**进程工作目录**解析——mvn spring-boot:run 的 cwd 是 ot-admin 故落 ot-admin/data/。非代码缺陷（snapshot_uri 存相对路径、取回经同配置解析，自洽）；部署时以绝对路径注入 `OT_PUBLISH_STORAGE_PATH` 即归一。由此产生的遗留问题见「五」。
2. **prod 回归的密钥注入**：首轮以示例密钥（DataKey16BytesXY）启动——启动本身成功，但 publish/list 返回 code 500「敏感数据解密失败」（库存量密文由 dev 默认密钥 otlab-data260930 对加密，示例密钥不匹配）。属本地回归环境口径问题、非实现缺陷；改用与库内数据匹配的密钥（otlab-data260930 / otlab-idx-260930，即 application.yml dev 默认值对）重启后通过。真实生产部署按约定经环境变量注入现网密钥即可。
3. **环境侧：WSL VM 空闲关停引发的容器「重启循环」假象**：验收前段 MySQL/Redis 间歇性拒连，排查确认 WSL2 VM 空闲 60 秒即整体关停（dockerd NRestarts=0、journal 无死亡记录、容器 ExitCode=0），唤醒后容器由 unless-stopped 策略拉起。以 run_in_background 保活循环（每 10 秒 wsl 心跳）维持 VM 常驻后全程稳定。属本机 WSL 空闲策略，非项目问题；协调者探活时「自启恢复」即同一机制的表现。
4. **冒烟 API 用 ASCII 载荷**：Git Bash 下 curl 传中文 JSON 触发编码异常（400「JSON格式错误」），改用 ASCII 名称完成建链；不影响验收有效性。

## 五、遗留问题

1. **[gitignore 盲区，建议补一行]** `.gitignore`（8566f57）只加 `backend/data/`，而运行时工作目录不同（如 mvn spring-boot:run 的 cwd=ot-admin）时快照产物落在 `backend/ot-admin/data/`——该位置**未被 ignore**（本次冒烟实测 git status 出现 `?? backend/ot-admin/data/`，产物已清理）。建议追加 `backend/*/data/`（或改用绝对路径约定）交协调者裁决；本次未改动（只验收不修复 + .gitignore 不在验收改动范围）。
2. **[minor，凭据终检告警文案]** 终检失败消息中命中键重复拼接：`...channels[1].connConfig.mqttPasswordchannels[1].connConfig.mqttPassword`（键清单 join 无分隔符）。功能不受影响（fail-closed 正确），属提示文案瑕疵，可在后续修订轮归入（与 5001 消息等文案统一处理）。
3. **[接线义务登记（brief D7 声明）]** 发布确认门 PublishConfirmGate 为 NoOp 占位，TASK-0015 交付后接线强制影响预览+二次确认——本期发布不阻断，属计划内占位，非缺陷。
4. **[MinIO 二期登记（brief D2 声明）]** 快照存储一期本地目录，MinIO 对象存储二期——计划内。
5. **[上游回写建议（脚本头注已登记）]** uk_scope_version 为本任务增补的物理唯一键（上游附录 A p33 无此约束），建议照 TASK-0021 补 tenant_id 先例「攒下一版小修订」回写上游。
6. **[环境]** 本机 WSL2 空闲 60 秒关停导致 docker 容器（host 网络）周期性不可达，长会话验收/开发需保活或调 vmIdleTimeout——建议登记本机开发环境注意事项。

## 六、一句话结论

**通过**——验收 1（2165 测试 0 失败 0 错误 BUILD SUCCESS）、验收 2（升级脚本两次 DONE、uk_scope_version 与 4070-4073 种子在位）、验收 3（发布 v1→文件落盘哈希可验→变更再发布 v2→回滚 v3 复用旧快照→取回哈希与 v1 一致→并发发布一 200 一 5001→含凭据行发布被终检 400 拦截 fail-closed）、验收 4（prod 档真实启动 0 ERROR + publish/list 200）、验收 5（29 文件改动全落 brief 声明范围）五条全部以原始输出为证通过；遗留仅 minor 级（gitignore 盲区、终检文案重复拼接）与计划内占位（确认门 NoOp、MinIO 二期），无阻断缺陷。
