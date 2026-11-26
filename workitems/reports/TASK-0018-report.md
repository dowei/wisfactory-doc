# TASK-0018 实施报告：iot_channel 通道 CRUD 与 JSON Schema 校验

- 派发 brief：TASK-0018-brief.md（BASE `042d13f`）
- 提交：`617b884`（main，注明 TASK-0018）
- 总体结论：**5 条验收全部通过**（全量回归 1669 = 基线 1609 + 新增 60，0 失败 0 错误；升级脚本两次执行均 DONE 幂等 + 表结构/4046-4050 核对通过；dev 冒烟 8 步全绿含 DB 核对；prod 档真实登录 200 + channel/list 200 + 全程 0 ERROR；改动 28 文件，25 个落在四条声明路径内，3 个网关域文件为 brief 明文要求的 guard 转正接线——见「偏离计划之处」第 1 条）。

## 1. 改动文件清单

**新增——通道域主代码（backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/）**

| 文件 | 说明 |
| --- | --- |
| `controller/IotChannelController.java` | 五接口 /iot/channel：list（gatewayId 必传）/detail/add/edit/remove；MODBUS/OPCUA 成功文案标注 |
| `domain/ChannelProtocols.java` | 协议字典四值 + 预留协议集（服务层/控制器共用一处，防漂移） |
| `dto/IotChannelCreateForm.java` | 创建表单（connConfig 以 `tools.jackson.databind.JsonNode` 承载——请求即 JSON 对象字面量） |
| `dto/IotChannelUpdateForm.java` | 编辑表单（channel_code/gateway_id 不可变语义注释在列） |
| `entity/IotChannel.java` | iot_channel 实体（extends BaseEntity；conn_config String 承载落库原文） |
| `mapper/IotChannelMapper.java` | 空 BaseMapper（多租户插件注入） |
| `mapper/IotChannelCodeMapper.java` | uk_gateway_channel_code 双道闸第一道预检（旁路 @TableLogic/插件，显式条件含 deleted 标记） |
| `mapper/IotChannelCodeRow.java` | 预检占用行 |
| `schema/ChannelConnConfigValidator.java` | 版本化 JSON Schema 校验器（手写最小校验器——理由见「偏离」第 2 条；凭据关键词递归双保险 + 归一化） |
| `service/IotChannelService.java` / `service/impl/IotChannelServiceImpl.java` | CRUD 编排：双道闸 4002、gateway_id 存在/同租户/未删、协议字典、Schema 校验、可建不可启用、点位 guard、白名单 + @Transactional |
| `service/ChannelReferenceGuard.java` / `service/impl/PlaceholderChannelReferenceGuard.java` | 通道删除保护·点位计数接缝（0021 接线义务，本期恒 0——照 0017 同款结构就位范式） |
| `service/impl/IotChannelGatewayReferenceGuard.java` | **网关删除保护·通道侧实现转正**（countChannels/channelCodes 实查 iot_channel；countPoints 仍 0021 占位） |

**新增——Schema 资源（backend/ot-admin/src/main/resources/schema/channel/）**：`mqtt-v1.json`（broker URI 必填/topic 必填/qos 0-1-2，additionalProperties=false）、`http-v1.json`（reportPath 必填 ^/ 开头/authType 枚举 none|apiKey，additionalProperties=false）、`modbus-v1.json`、`opcua-v1.json`（宽松骨架：仅约束顶层 object，additionalProperties=true——二期登记遗留）。

**新增——测试（backend/ot-admin/src/test/java/net/otlab/admin/module/iot/channel/）**：`schema/ChannelConnConfigValidatorTest`（17）、`service/impl/IotChannelServiceImplTest`（24）、`service/impl/IotChannelGatewayReferenceGuardTest`（3）、`controller/IotChannelControllerTest`（12）、`config/IotChannelMenuPermissionGuardTest`（3）——共 59。

**新增——升级脚本**：`resources/sql/upgrade/20261122_iot_channel_domain.sql`（附录 A p30 DDL 逐字 + 菜单 4046-4050 + 超管授权，幂等 + 头注全套纪律）。

**修改——网关域 guard 转正接线（brief 明文要求，3 文件）**

| 文件 | 说明 |
| --- | --- |
| `gateway/service/GatewayReferenceGuard.java` | 接口新增 `channelCodes(Long)`（拒绝文案「并提示通道编码」用）；javadoc 同步 0018 转正事实 |
| `gateway/service/impl/IotGatewayServiceImpl.java` | 删除保护拒绝分支文案补通道编码（拒绝语义/调用点结构不变）；javadoc 同步 |
| `gateway/service/impl/PlaceholderGatewayReferenceGuard.java` | **删除**（按其自身契约「0018/0021 建表后替换占位实现」——由通道域 `IotChannelGatewayReferenceGuard` 替代） |
| `gateway/service/impl/IotGatewayServiceImplTest.java`（测试） | 删除分支用例补「提示通道编码」断言 + 空编码表退回纯计数文案防御分支（+1 净新增） |

## 2. 验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0。ot-admin 全量（终码复跑，BASE 基线 1609）：

```
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.channel.schema.ChannelConnConfigValidatorTest
[INFO] Tests run: 24, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.channel.service.impl.IotChannelServiceImplTest
[INFO] Tests run:  3, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.channel.service.impl.IotChannelGatewayReferenceGuardTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.channel.controller.IotChannelControllerTest
[INFO] Tests run:  3, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.channel.config.IotChannelMenuPermissionGuardTest
[INFO] Tests run: 27, Failures: 0, Errors: 0, Skipped: 0 -- net.otlab.admin.module.iot.gateway.service.impl.IotGatewayServiceImplTest（0017 基线 26 + 新增 1）
[INFO] Results:
[WARNING] Tests run: 1669, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS（Total time: 01:00 min）
```

（1669 = 基线 1609 + 通道域 59 + 网关删除用例净增 1；skipped 3 为基线既有。）

### 2.2 验收 2：升级脚本二次幂等 + 表结构 + 4046-4050

第一次执行 `sql-exec.sh ../resources/sql/upgrade/20261122_iot_channel_domain.sql`：

```
OK(0)
OK(0)
OK(5)
OK(5)
DONE
```

第二次执行：

```
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

核对（SHOW CREATE TABLE 摘录 + 菜单/授权）：

```
  `conn_config` json DEFAULT NULL COMMENT '连接配置 JSON（不含凭据）',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用',
  ...
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_gateway_channel_code` (`gateway_id`,`channel_code`),
  KEY `idx_tenant_gateway` (`tenant_id`,`gateway_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='采集通道表'

id	parent_id	menu_name	menu_type	perms
4046	4013	通道查询	3	iot:channel:list
4047	4013	通道详情	3	iot:channel:query
4048	4013	通道新增	3	iot:channel:add
4049	4013	通道修改	3	iot:channel:edit
4050	4013	通道删除	3	iot:channel:remove
role_id	menu_id
1	4046
1	4047
1	4048
1	4049
1	4050
```

### 2.3 验收 3：dev 冒烟（mvn spring-boot:run；日志 backend/target/task0018_dev.log）

启动前 8098 无 LISTENING；`Started OtAdminApplication in 10.745 seconds`；启动段 ERROR 计数 = 0。冒烟脚本 backend/target/t0018_smoke.py 原始输出（2026-11-22 09:59 轮，最终构建）：

```
=== S1) admin dev login -> 200 token= 9da58a45...
=== S2) 建网关 GW_T0018_01（仅必填） ===
S2a POST gateway GW_T0018_01 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2b 网关 id = 6
=== S3) 建 MQTT 通道（合法 conn_config） ===
S3a POST MQTT CH_T0018_M1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S3b POST HTTP CH_T0018_H1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
=== S4) 重复码 4002（同网关同码） ===
S4a duplicate code -> HTTP 200 code 4002 | 通道编码已存在: CH_T0018_M1  [期望4002] OK
=== S5) 建 MODBUS 通道（status 强制 0 + 标注核验） ===
S5a POST MODBUS CH_T0018_MB1 携带status=1 -> HTTP 200 code 200 | 通道保存成功：该协议一期未启用适配器，status 已置 0（不产生接入行为，二期适配器上线后可启用）  [期望200] OK
S5b 成功文案标注: 通道保存成功：该协议一期未启用适配器，status 已置 0（不产生接入行为，二期适配器上线后可启用）
S5c list codes: ['CH_T0018_M1', 'CH_T0018_H1', 'CH_T0018_MB1'] | MODBUS status = 0 MQTT status = 1
=== S6) 负路径（Schema/字典/不可变字段） ===
S6a 未知协议 COAP -> 期望400 -> code 400 | 协议类型仅允许 MQTT/HTTP/MODBUS/OPCUA: COAP
S6b MQTT 缺 broker -> 期望400 -> code 400 | conn_config 缺少必填字段: [broker]
S6c 含凭据键 conn_config -> 期望400 -> code 400 | conn_config 不允许携带凭据字段（ADR-0006：凭据独立存凭据管理，不入 conn_config 明文）: password
S6d 未知键（additionalProperties=false） -> 期望400 -> code 400 | conn_config 不符合 HTTP v1 Schema: $.clientId 为未知字段（additionalProperties=false）
S6e 改通道编码 -> 期望400 -> code 400 | 通道编码不允许修改: CH_T0018_M1
S6f 换所属网关 -> 期望400 -> code 400 | 通道不允许更换所属网关: 6
S6g list 缺 gatewayId -> 期望400 -> code 400 | 按网关查通道：gatewayId 必传
S6h MODBUS 编辑携带 status=1 -> 期望200 -> code 200 | 操作成功
S6i MODBUS status after edit: 0 (期望 0)
=== S7) 删网关拒（有通道，提示通道编码） ===
S7a DELETE gateway GW_T0018_01 -> 期望400 -> code 400 | 网关存在采集通道，禁止删除（通道 3 条: CH_T0018_H1、CH_T0018_M1、CH_T0018_MB1，先清理通道）
=== S8) 删通道 → 删网关成功 ===
S8a DELETE CH_T0018_M1 / CH_T0018_H1 / CH_T0018_MB1 -> 三条均 code 200 | 操作成功
S8b DELETE gateway GW_T0018_01 -> 期望200 -> code 200 | 操作成功
```

DB 核对（S9，sql-exec.sh）：

```
id	gateway_id	channel_code	protocol_type	conn_config	status	deleted
4	6	CH_T0018_M1	MQTT	{"qos": 1, "topic": "factory/gw01/up", "broker": "tcp://192.168.1.10:1883"}	1	1
5	6	CH_T0018_H1	HTTP	{"authType": "apiKey", "reportPath": "/api/v1/report"}	1	1
6	6	CH_T0018_MB1	MODBUS	{"slaveId": 1}	0	1
id	gateway_code	deleted
6	GW_T0018_01	1
```

（conn_config json 列内键序由 MySQL JSON 类型归一，与落库前紧凑序列化语义等价；MODBUS status=0、MQTT/HTTP status=1、三通道+网关均 deleted=1 软删。用毕即停：taskkill → 8098 复核无 LISTENING。判定：**通过**。）

### 2.4 验收 4：prod 档真实启动回归

打包 `mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（ot-admin-1.0.0.jar）。启动（沿 0017 确认命令 + 三密钥 + 内部令牌 env）：

```
Started OtAdminApplication in 18.095 seconds (process running for 18.878)
=== prod 全程 ERROR 计数 ===
0
=== prod 真实登录（Redis 预置 captcha:code:task0018a="AB12"（JSON 带引号，EX 600）+ SM4 admin123 密文，jshell ROUNDTRIP="admin123" 往返验证） ===
LOGIN HTTP 200 code 200 message 操作成功
LOGIN token 0db93a63...
=== prod GET /iot/channel/list?pageNum=1&pageSize=10&gatewayId=6 ===
LIST HTTP 200 code 200 total 0
（gatewayId=6 的冒烟通道已软删——@TableLogic 过滤后 total=0，接口 200 可用）
=== prod GET /iot/gateway/list（旁证） ===
LIST HTTP 200 code 200 total 2
gateway codes: ['GW_ASCII_T1', 'GW_T0017_01']（软删行 GW_T0017_02 / GW_T0018_01 正确过滤）
```

用毕即停：taskkill → 8098 复核无 LISTENING。判定：**通过**（启动+全程 0 ERROR、真实登录 200、channel/list 200）。

### 2.5 验收 5：改动范围核对 `git diff --name-only 042d13f..HEAD`

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/controller/IotChannelController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/domain/ChannelProtocols.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/dto/IotChannelCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/dto/IotChannelUpdateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/entity/IotChannel.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/mapper/IotChannelCodeMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/mapper/IotChannelCodeRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/mapper/IotChannelMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/schema/ChannelConnConfigValidator.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/service/ChannelReferenceGuard.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/service/IotChannelService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/service/impl/IotChannelGatewayReferenceGuard.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/service/impl/IotChannelServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/service/impl/PlaceholderChannelReferenceGuard.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/GatewayReferenceGuard.java          ← 偏离 1
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/impl/IotGatewayServiceImpl.java     ← 偏离 1
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/impl/PlaceholderGatewayReferenceGuard.java（删除）← 偏离 1
backend/ot-admin/src/main/resources/schema/channel/http-v1.json
backend/ot-admin/src/main/resources/schema/channel/modbus-v1.json
backend/ot-admin/src/main/resources/schema/channel/mqtt-v1.json
backend/ot-admin/src/main/resources/schema/channel/opcua-v1.json
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/channel/config/IotChannelMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/channel/controller/IotChannelControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/channel/schema/ChannelConnConfigValidatorTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/channel/service/impl/IotChannelGatewayReferenceGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/channel/service/impl/IotChannelServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/gateway/service/impl/IotGatewayServiceImplTest.java  ← 偏离 1（测试同口径调整）
resources/sql/upgrade/20261122_iot_channel_domain.sql
```

共 28 文件：25 个落在四条声明路径内；3 个网关域文件为 guard 转正接线（偏离 1，见下）。frontend/、ingest/ 零触碰；workitems 库零改动（本报告文件除外）；TASK 状态未改。

## 3. 未做项与原因

1. **点位引用计数未实现**（brief 明文）：iot_point 表归 TASK-0021 建表——通道删除保护的点位计数以 `ChannelReferenceGuard` 接缝结构就位（占位恒 0，拒绝分支被 `IotChannelServiceImplTest.testDelete_PointReferenceRejected` 固化），`IotChannelGatewayReferenceGuard.countPoints` 同样占位；接线义务登记 0021 brief。
2. **通道行不含 config_version 列**：上游附录 A p30 DDL 无此列（brief 裁决）——通道生效状态由配置快照对账（FEAT-0008/TASK-0024）。
3. **无任何下发/生效动作**（brief 裁决「保存≠生效语义=本期无即时下发动作」）：本期管理面 CRUD 不触发推送——REQ-0010 验收 4「变更未经发布对接入不生效，发布后生效」由 TASK-0024 配置快照发布链路承载（通道行变更落入下一次快照）；本服务无推送依赖、无消息发送代码。
4. **MODBUS/OPCUA conn_config 完整字段集未定义**（brief 明文「宽松骨架……二期定义，登记遗留」）：v1 仅约束顶层 object + additionalProperties 允许，凭据双保险对这两协议同样生效。
5. **网关删除后通道编码不入复用池之类关联语义**：channel_code 唯一作用域是 (gateway_id, channel_code)，网关软删不影响其下通道行（仍占用编码；引用计数保护使其不可达删除路径——先清通道才能删网关）。

## 4. 偏离计划之处

1. **网关域 3 文件改动（含 1 删除）——brief 明文要求所致，非越界**：brief 接口与决策节要求「实现 GatewayReferenceGuard 的通道侧实现——网关删除 guard 从占位转正」、测试口径要求「种子通道行 → 删网关拒并提示通道编码」。为此：(a) 按占位实现自身契约（「0018/0021 建表后替换占位实现，不改调用点」）删除 `PlaceholderGatewayReferenceGuard`，通道域新增 `IotChannelGatewayReferenceGuard`（实现接口，countChannels 实查 iot_channel）；(b) 接口加 `channelCodes(Long)` 方法、`deleteGateway` 拒绝分支文案补通道编码——拒绝语义（计数>0 即拒）与调用点结构未变，仅文案按 §3.1 p21「拒绝并提示」增补（冒烟 S7a 固化）。`IotGatewayServiceImplTest` 同口径调整（+1 用例）。验收 5 清单中如实标注。
2. **校验库选型：手写最小校验器（brief 二选一）**：一期 Schema 词表固定且极小（type/required/properties/additionalProperties/enum/minLength/maxLength/pattern/format:uri），networknt 的全量 draft 支持（$ref/assertion/远端格式等）对一期是过剩能力，且新依赖引入传递依赖面与版本管理成本——架构选型「组件能晚引入就晚引入」同口径（ADR-0006 放弃外部密钥服务同理）。Schema 文件本身为标准 draft-07 子集，二期词表扩张（MODBUS/OPCUA 完整字段集）可直接切换标准库实现而不改 Schema 文件。
3. **表单 connConfig 以 `tools.jackson.databind.JsonNode` 承载（而非 String）**：首轮 dev 冒烟 S3a 实测——前端自然以 JSON 对象字面量提交 conn_config，String 字段致反序列化 400（「请求参数格式错误」）。改为 JsonNode 后请求即对象、服务层 toString() 转紧凑文本再入校验器；同时修正了 Spring Boot 4 运行时 Jackson 3（tools.jackson）与 Jackson 2 注解/实体的类型边界（DTO 用 tools.jackson，校验器内部解析保持 Jackson 2——其仅处理内部文本，不触 HTTP 转换器）。附带强化：凭据关键词扫描按 brief「顶层禁」升级为**递归全层级**（宽松骨架协议允许嵌套对象，仅查顶层会漏 `{"tcp":{"password":...}}`）。
4. **detail 端点挂 iot:channel:query**：§3.1 p21 通道段五接口与菜单五行（brief 明文 4046-4050 五行）一一对应——与网关域「query 预留不挂端点」不同，通道有 detail 行，`IotChannelMenuPermissionGuardTest` 断言五端点 ↔ 五菜单行精确双向对应。
5. **冒烟首轮两次未全绿（脚本/代码缺陷，均已修复并复跑）**：第一轮 String connConfig 400（偏离 3）；第二轮冒烟脚本 list GET 漏拼 GID 导致 S5c KeyError——均为脚本/接口形态问题，第三轮（终码）全绿。失败轮原始输出留档 backend/target/（首轮 task0018_dev.log 已被复跑覆盖，偏差在偏离 3/4 说明中如实记录）。
6. **conn_config 入参长度上限 8192 字符**（brief 未列）：服务层防线（校验器内），防误传大负载；超限拒并提示。单测 `testOverlongRejected` 固化。

## 5. 遗留问题

1. **iot_point 引用计数接线（TASK-0021）**：`ChannelReferenceGuard.countPoints` 与 `IotChannelGatewayReferenceGuard.countPoints` 双占位——0021 建表后按其 brief 裁决的点位归属列（gateway_id 直引或经通道间接）扩展实现，不改调用点（0018 已实践同款转正路径，可照抄）。
2. **MODBUS/OPCUA conn_config v1 宽松骨架**（REQ-0010 非目标）：二期随适配器框架 REQ-0012 定义完整字段集后升 v2（新增 v2 文件 + 调 `ChannelConnConfigValidator.SCHEMA_VERSION` 映射；v1 文件不动）。
3. **qos/authType 缺省不落库补省**：Schema `default` 仅作客户端提示语义，服务端不注入缺省值（MQTT qos 缺省按 0 处理、HTTP authType 按 none——运行语义由 0022 适配器按缺省解释）。若二期要求数据库显式落省值，需在 0022 裁决。
4. **dev 库冒烟留痕**（是否清理由协调者裁决）：iot_channel 三行（CH_T0018_M1/CH_T0018_H1/CH_T0018_MB1，id 4-6，均 deleted=1）与 iot_gateway 一行（GW_T0018_01，id=6，deleted=1）。重跑冒烟需换码（同码撞 4002/已被删除记录占用属预期）。prod Redis 种子 captcha:code:task0018a 已 EX 600 自动清理。
5. **网关删除文案含通道编码的分隔符**：多通道以「、」连接；若二期要求可点击定位（前端跳转通道列表带 gatewayId 过滤），归前端交互任务，后端接口已提供 list 按 gatewayId 过滤能力。

（报告完。五项验收证据原始输出见第 2 节；本地留档：backend/target/task0018_fulltest.log、task0018_smoke_out.log、task0018_dev.log、task0018_prod.log、task0018_prod_out.log。）

---

## 修订轮（TASK-0018 验收退回一轮，§13 diff 评审）

- 评审报告：TASK-0018-review.md（Critical 0 / Important 2 / Minor 5；Minor 5 条不动记 deferred）
- 修复范围：仅 `channel/schema/ChannelConnConfigValidator.java` 与其测试（契约不变）；提交注明 TASK-0018 修订轮。

### R-1 必修项修复内容

| 项 | 修复 | 实现 |
| --- | --- | --- |
| I-1 required 可被显式 null 绕过 | `requireNoMissing` 改为**值感知**判定（键存在且值非 null 且字符串非空白——显式 null / 空白必填串一律按缺失报「缺少必填字段」文案），并**前移到结构校验之前**（membership 语义先于 type/format——否则 `{"broker":null}` 会被「必须是绝对 URI」格式错误掩盖，报不出缺字段文案） | `ChannelConnConfigValidator.requireNoMissing(JsonSchema, JsonNode)` 重写 + 校验编排顺序调整 |
| I-2 凭据递归扫描不进数组 | `collectCredentialKeys` 重写为**全深度递归**：对象逐层扫键名 + 递归成员值，**数组元素逐个递归**（路径可读 `creds[0].password` / `signals[0].group[0].secret`）；标量递归终止 | `ChannelConnConfigValidator.collectCredentialKeys` 重写 + 类 javadoc 同步 |

### R-2 红转绿记录（TDD）

新增 4 用例（`ChannelConnConfigValidatorTest`），**修复前先跑捕获 RED**：

```
[ERROR] Tests run: 21, Failures: 4, Errors: 0, Skipped: 0 -- ChannelConnConfigValidatorTest
[ERROR]   testMqttExplicitNullBrokerRejected      （I-1：{"broker":null} 修复前静默通过并落库）
[ERROR]   testMqttBlankBrokerRejected             （I-1：{"broker":"   "} 空白必填串）
[ERROR]   testCredentialInsideArrayRejected       （I-2：{"creds":[{"password":"x"}]} 修复前静默通过——MODBUS 宽松骨架唯一可落库形状）
[ERROR]   testCredentialDeepNestedArrayRejected   （I-2：{"signals":[{"group":[{"secret":"x"}]}]}）
```

修复后全绿（同套件复跑）：

```
[INFO] Tests run: 21, Failures: 0, Errors: 0, Skipped: 0 -- ChannelConnConfigValidatorTest
```

另同口径调整 2 个既有断言（排序变化的连带，拒绝语义不变）：`testMqttBrokerNotUri`（空串 broker 现按缺失报，非 URI 用例仍报格式）与 `IotChannelServiceImplTest.testUpdate_ProtocolChangeRevalidatesConnConfig`（MQTT→HTTP 旧配置缺 reportPath 现报「缺少必填字段」而非 unknown-key 聚合——仍拒且更直指语义）。

### R-3 补充验收 1：全量复跑 `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 退出码 0；ot-admin 全量：

```
[INFO] Results:
[WARNING] Tests run: 1673, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS（Total time: 01:02 min）
```

（1673 = 修订前 1669 + 新增 4；0 失败 0 错误。）

### R-4 补充验收 2：dev 冒烟复核（backend/target/t0018_rev_smoke.py；日志 task0018_rev_dev.log）

启动前 8098 无 LISTENING；`Started OtAdminApplication in 12.264 seconds`；启动段 ERROR 计数 = 0。原始输出：

```
=== R1) admin dev login -> 200 token= 1f4edde2...
=== R2) 建网关 GW_T0018R1（修订轮复核专用码） ===
R2a POST gateway GW_T0018R1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
R2b 网关 id = 7
=== R3) I-1 复核：显式 null broker 保存拒（期望400 缺少必填字段） ===
R3a MQTT broker=null -> HTTP 200 code 400 | conn_config 缺少必填字段: [broker]  [期望400] OK
=== R4) I-2 复核：数组嵌套凭据 conn_config 拒（期望400 凭据字段） ===
R4a MODBUS creds 数组内 password -> HTTP 200 code 400 | conn_config 不允许携带凭据字段（ADR-0006：凭据独立存凭据管理，不入 conn_config 明文）: creds[0].password  [期望400] OK
R4b OPCUA 深层嵌套 secret -> HTTP 200 code 400 | conn_config 不允许携带凭据字段（ADR-0006：凭据独立存凭据管理，不入 conn_config 明文）: signals[0].group[0].secret  [期望400] OK
=== R5) 正常 MQTT 通道回归成功（期望200） ===
R5a POST MQTT CH_T0018R_M1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
R5b list codes: ['CH_T0018R_M1'] | MQTT status = 1 | connConfig = {"qos": 1, "topic": "factory/gw01/up", "broker": "tcp://192.168.1.10:1883"}
=== R6) 清理：删通道 + 删网关（期望200） ===
R6a DELETE CH_T0018R_M1 -> code 200 | 操作成功
R6b DELETE gateway GW_T0018R1 -> code 200 | 操作成功
```

DB 留痕核对（sql-exec.sh）：

```
id	gateway_id	channel_code	status	deleted
7	7	CH_T0018R_M1	1	1
id	gateway_code	deleted
7	GW_T0018R1	1
```

用毕即停：taskkill → 8098 复核无 LISTENING。判定：**补充验收两条全部通过**。

### R-5 修订轮改动清单与遗留

- 改动文件（2）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/schema/ChannelConnConfigValidator.java`（requireNoMissing 值感知 + 前移；collectCredentialKeys 全深度递归；类 javadoc 同步）、`backend/ot-admin/src/test/java/net/otlab/admin/module/iot/channel/schema/ChannelConnConfigValidatorTest.java`（+4 用例）；连带断言对齐 1 处（`IotChannelServiceImplTest`，注释注明修订轮排序）。未触碰其他文件。
- Minor 5 条不动记 deferred：M-1（编辑预留协议不携带 protocolType 时成功文案不标注）/ M-2（校验器协议大小写接缝，服务层字典闸兜底）/ M-3（框架级 JSON body 无全局上限，建议立独立任务）/ M-4（Schema 资源无缓存，二期换库一并解决）/ M-5（网关 javadoc 缩进错位，纯外观）——收尾时按评审裁决进 LOG。
- 本地留档新增：backend/target/task0018_fulltest_rev.log、task0018_rev_smoke_out.log、task0018_rev_dev.log、t0018_rev_smoke.py、t18rev.log / t18rev2.log（红转绿中间记录）。

