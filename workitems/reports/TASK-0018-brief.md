# 实施 brief：TASK-0018（iot_channel 通道 CRUD 与 JSON Schema 校验）

## 批次

- TASK-0018（本批共 1 个，标准任务 2 人天；backend 串行道——采集域第二环）
- BASE：`042d13f`（主仓 HEAD，派发时回填）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/**`（通道域新包）
  - `backend/ot-admin/src/main/resources/schema/**`（协议 conn_config JSON Schema 文件，新增——classpath 资源）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_channel_domain.sql`（新增）
  - 不碰 frontend/、ingest/、网关域（复用 GatewayReferenceGuard 接口实现接线）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0018.md`
- 关联对象：同目录 `features\FEAT-0006.md`、`requirements\REQ-0010.md`（业务能力/核心规则/验收 1-4）、`testcases\TC-0006.md`
- 决策：`decisions\ADR-0006.md`（通道凭据不进 conn_config 明文——独立存 0019）；通道表 DDL 权威=附录 A p30（iot_channel——brief 已提取逐字贴下）；接口行 §3.1 p21（channel/list 按网关查）
- 接线对象：`iot/gateway/service/GatewayReferenceGuard.java`（0017 占位接口——本任务实现通道侧引用计数，网关删除保护生效）
- 既有范式：iot/gateway（0017）、iot/template（0011）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_channel_domain.sql，逐字 + 幂等 + 头注）

```sql
CREATE TABLE `iot_channel` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '通道 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `gateway_id` bigint NOT NULL COMMENT '所属网关 ID',
  `channel_code` varchar(64) NOT NULL COMMENT '通道编码（网关内唯一）',
  `channel_name` varchar(100) NOT NULL COMMENT '通道名称',
  `protocol_type` varchar(16) NOT NULL COMMENT '协议 MQTT/HTTP/MODBUS/OPCUA',
  `conn_config` json DEFAULT NULL COMMENT '连接配置 JSON（不含凭据）',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_gateway_channel_code` (`gateway_id`,`channel_code`),
  KEY `idx_tenant_gateway` (`tenant_id`,`gateway_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='采集通道表';
```

菜单种子 **4046 起**连续（4045 已占）：iot:channel:list/query/add/edit/remove 五行挂 4013 通道管理页下，幂等 + 超管授权。

### 代码结构（/iot/channel 段，语义照 REQ-0010）

- CRUD 五接口（/iot/channel）：list（按 gatewayId 必传查通道列表——§3.1 口径，另有分页过滤 channelCode/protocolType/status）、add（channel_code 网关内唯一双道闸 4002；gateway_id 校验存在/同租户/未删；protocol_type ∈ 字典 MQTT/HTTP/MODBUS/OPCUA）、edit（channel_code 不可改；conn_config 变更重新校验；gateway_id 不可改——通道属网关固定）、remove（软删；**其下有点位则拒绝**——点位表未建，guard 结构就位登记 0021 接线；同时实现 GatewayReferenceGuard 的通道侧实现——网关删除保护中通道引用计数生效，网关删除 guard 从占位转正）、detail。
- **conn_config JSON Schema 校验（版本化）**：
  - Schema 文件存 `resources/schema/channel/`（classpath）：`mqtt-v1.json`、`http-v1.json`、`modbus-v1.json`、`opcua-v1.json`——一期按 REQ-0010 业务能力最小字段集定义：MQTT={broker(URI 必填), topic(必填), qos(0/1/2 默认 0)}；HTTP={上报路径(必填), 鉴权方式(枚举 none/apiKey 默认 none)}；MODBUS/OPCUA=宽松骨架（结构字段暂空 + additionalProperties 允许——二期定义，登记遗留）。Schema 版本号入文件名与 `$id`，变更升版本。
  - 校验库：**networknt json-schema-validator**（小依赖，报告偏离节声明理由）或手写最小校验器（二选一，理由写报告）。
  - 凭据字段防线（ADR-0006）：Schema 定义 `additionalProperties: false` 且顶层禁 `credential`/`password`/`secret`/`token` 等键名（显式 forbidden 关键词校验双保险）——含凭据字段的 conn_config 保存被拒。
  - 协议-Schema 匹配：protocol_type 决定加载哪个 schema；未知协议拒。
- **MODBUS/OPCUA 可建不可启用（REQ-0010 验收 2）**：保存成功但 **status 强制 0**（请求携带 status=1 一律落库 0 + 响应/文档标注「未启用适配器，一期不产生接入行为」）；MQTT/HTTP 正常启停。
- **变更须重新发布才生效（REQ-0010 验收 4）**：通道行加 `config_version` 语义？——配置快照链路（FEAT-0008/TASK-0024）承载版本；本期通道行**不含** config_version 列（上游 DDL 无此列——通道生效状态由快照对账，登记口径即可）；保存≠生效的语义=本期无即时下发动作（无推送链路），报告中说明「生效走 TASK-0024 配置快照发布链路」。
- 白名单构建 + 双道闸 4002 + `@Transactional` 照范式；tenant_id 会话覆盖。

### 测试口径（TDD）

- Schema：MQTT 合法/缺 broker 拒/qos 非法拒；HTTP 合法/缺路径拒；含 credential 键拒（双保险）；MODBUS/OPCUA 保存 status 强制 0；未知协议拒。
- CRUD：网关内唯一 4002（活跃+兜底）/改码拒/gateway_id 不可改/网关删除保护转正（种子通道行 → 删网关拒并提示通道编码）/点位引用 guard 占位（0021 接线）。
- 控制器：路由/权限注解/按 gatewayId 必传校验。
- 冒烟（dev）：admin 建网关 → 建 MQTT 通道（合法 conn_config）→ 重复码 4002 → 建 MODBUS 通道 → status=0 标注核验 → 含凭据键 conn_config 拒 → 删网关拒（有通道）→ 删通道 → 删网关成功。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1609 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对表结构 + 4046-4050 五行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + channel/list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0018）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
