# 实施 brief：TASK-0017（iot_gateway 网关 CRUD 与在线状态回写）

## 批次

- TASK-0017（本批共 1 个，标准任务 2 人天；backend 串行道——采集域起点）
- BASE：`eb01ba2`（主仓 HEAD，派发时回填）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/**`（网关域新包）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_gateway_domain.sql`（新增）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增 `ot.gateway.internal-token` 一个键）
  - 不碰 frontend/、ingest/、instance/template 域（只读）、字典域

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0017.md`
- 关联对象：同目录 `features\FEAT-0006.md`、`requirements\REQ-0009.md`（验收 1-5——回写字段组固定/LWT 与 graceful 可区分归 0020）、`testcases\TC-0006.md`
- 表结构权威（pypdf 自行提取核对）：详细设计 V1.1 PDF 附录 A p30（iot_gateway DDL 逐字——tenant_id/gateway_code/instance_id/dept_id/online_status/last_heartbeat_time/heartbeat_period/version/config_version/ip_addr/status）；接口行 §3.1 p21（/api/v1/iot/gateway 五行）与内部回写接口（p21-22 内部段——自行提取）
- 既有范式：iot/template 批量保存与守卫（0011）、iot/instance 创建与删除保护（0012）、字典域 4002 双道闸

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_gateway_domain.sql，CREATE TABLE IF NOT EXISTS 幂等 + 「与上游差异」头注）

1. `iot_gateway`：附录 A p30 逐字（uk_tenant_gateway_code + idx_tenant_instance + idx_online）。若 MySQL 8 对某索引有长度问题照 0011 idx_ancestors 前缀先例处理并登记头注。
2. 菜单种子 **4041 起**连续（4040 已占）：iot:gateway:list/query/add/edit/remove 五行挂 4012 网关管理页下，幂等 + 超管授权。
3. **不建其他表**（通道/凭据/点位归 0018/0019/0021）。

### 代码结构（/iot/gateway 段，语义照 §3.1 p21）

- CRUD 五接口：list（分页+过滤 gatewayCode/gatewayName/status/onlineStatus——含在线状态与心跳时间列）、add（gateway_code 租户内唯一双道闸 4002；字符集 `[A-Za-z0-9_]`（作 MQTT 主题与设备身份段——REQ-0009 业务能力）+ 长度；instance_id 归属校验（存在/同租户/status=1）+ dept_id 校验（存在/同租户）；heartbeat_period 默认 30）、edit（gateway_code 不可改；白名单 gatewayName/instanceId/deptId/heartbeatPeriod/remark 类按库内实际列）、remove（软删；**其下有点位/通道时拒绝**——两表尚未建，guard 结构就位（引用计数调用点按表扩展），本期实际恒零引用，接线义务登记 0018/0021 brief）、detail（如 §3.1 有行则建，无则 list/detail 合并按 list 权限）。
- **在线状态回写内部接口**：`POST /internal/v1/gateway/heartbeat`（**无 Sa-Token**——内部白名单+令牌体系）：body 批量（网关编码列表+五字段组：onlineStatus/lastHeartbeatTime/ipAddr/version/configVersion）；**字段组白名单强制**（REQ-0009 验收 4「回写仅限既定字段组（越界 UPDATE 被拒）」——服务层构建更新集只含五字段，请求体多余字段直接拒绝）；按 gateway_code+tenant 上下文定位网关（接入服务侧租户由令牌/主题解析——本期接口按 body 携带 tenantCode 解析，租户不存在/禁用拒）。
- **内部认证裁决（占位）**：`InternalAuthChecker` 接口（本任务新建于 gateway 域）+ 配置静态令牌实现（`ot.gateway.internal-token` env 注入，header `X-Internal-Token` 比对）——完整网关令牌链归 0019（凭据）与 0020（LWT），接线义务登记遗留；IP 白名单：`ot.gateway.internal-ip-whitelist` 配置项（默认 127.0.0.1），校验在过滤器/拦截器层。
- 回写不写操作日志（高频心跳）——@OperateLog 不加，报告说明。
- LWT 与 graceful 区分：状态区分字段若上游 DDL 有则实现，无则 0020 裁决（本任务只保证回写字段组透传正确）。
- 字段白名单 + 双道闸 4002 + `@Transactional` 照既有范式；tenant_id 会话覆盖。

### 测试口径（TDD）

- CRUD：唯一性双道闸/改码拒/字符集与长度拒/instance 与 dept 校验/删除 guard 结构（本期恒放行+登记）/白名单落库。
- 回写：字段组白名单（越界字段拒）/批量多网关/租户禁用拒/内部令牌错拒/IP 白名单外拒/心跳更新落库（online_status/last_heartbeat_time/ip_addr/version/config_version 五列断言）。
- 控制器：路由/权限注解（管理面五接口）；内部接口无 Sa-Token 注解但有 InternalAuthChecker 调用断言。
- 冒烟（dev）：admin 建网关（挂实例+部门）→ list 含在线状态列 → 内部接口带正确令牌回写心跳（五字段）→ DB 核对 → 越界字段拒 → 错令牌拒 → 4041-4045 核对。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1551 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对表结构（SHOW CREATE TABLE iot_gateway）+ 4041-4045 五行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + /iot/gateway/list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0017-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0017）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
