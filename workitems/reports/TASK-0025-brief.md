# 实施 brief：TASK-0025（下发记录 ACK 回执与超时重发）

## 批次

- TASK-0025（本批共 1 个，标准任务 3 人天；backend 串行道——发布链第二环，前置 TASK-0024 已交付版本落库）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/**`（下发记录/ACK/扫描/重发/状态查询，全部在既有 publish 域内扩展）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_publish_log.sql`（新增）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增 ot.publish.ack-* 与 ot.publish.target-* 配置键）
  - 不碰 frontend/、ingest/、openapi 域、dataperm、credential 域、gateway 域、ot-core（gateway_code 读取复用已有 mapper，不在 gateway 域内改代码）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0025.md`
- 关联对象：同目录 `features\FEAT-0008.md`、`requirements\REQ-0033.md`（验收：ACK 三态可查/超时标红可重发/发布完成以全部 ACK 生效为准）、`testcases\TC-0008.md`
- 决策：`decisions\ADR-0008.md`（快照按实例有效版本组装）；`decisions\ADR-0006.md`（快照不含凭据）
- 上游任务交付：`reports\TASK-0024-brief.md`（发布域既有结构/四端点/确认门 NoOp 范式——0025 的接缝风格照它）
- 表结构权威：详细设计附录 A p33 iot_config_publish_log（brief 已提取贴下）；接口权威：详细设计 §3.1 p22（发布四接口/下发状态查询）+ §3.3 p25（POST /internal/v1/config/ack）——brief 已提取贴下
- 错误码：5002 下发未确认（ResultCode 已有）；400 参数错误；5001 版本冲突（0024 已接）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_publish_log.sql，p33 iot_config_publish_log 逐字 + 幂等 + 头注）

```sql
CREATE TABLE `iot_config_publish_log` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '下发记录 ID',
  `version_id` bigint NOT NULL COMMENT '配置版本 ID',
  `target_type` tinyint NOT NULL COMMENT '目标 1 网关 2 接入 3 计算',
  `target_id` varchar(64) NOT NULL COMMENT '目标标识',
  `ack_status` tinyint DEFAULT '0' COMMENT '回执 0 待确认 1 生效 2 失败',
  `ack_time` datetime DEFAULT NULL COMMENT '回执时间',
  `error_msg` varchar(1000) DEFAULT NULL COMMENT '失败原因',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  KEY `idx_version` (`version_id`),
  KEY `idx_ack` (`ack_status`,`ack_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='配置下发记录表';
```

（与上游无列差异，逐字；头注登记两点：①不追加 uk_version_target——版本行由 uk_scope_version 单调保证，log 行与版本同事务一次性写入、重发不新增行，无并发重复路径；②目标标识语义：target_id 网关= gateway_code，接入/计算=配置化服务标识。）菜单种子 **4074**：iot:publish:resend 挂 4015 配置发布页下，幂等 + 超管授权（照 0024 的 4070-4073 范式；只读状态查询复用 iot:publish:list 不新增种子）。

### 代码结构（publish 域内扩展）

1. **下发记录落行**：发布成功（版本落库后同事务）按 scope 派生目标并插入 iot_config_publish_log（ack_status=0）：scope_type=1 → 两行〔1/网关 gatewayCode（读 iot_gateway）+ 2/接入服务标识〕；scope_type=2 → 一行〔3/计算引擎标识〕。接入/计算标识取配置 `ot.publish.target-ingest-id`（默认 ingest）/ `ot.publish.target-calc-id`（默认 calc）。回滚生成新版本走同一下发落行逻辑（回滚=重新下发）。
2. **ACK 回执接口**：`POST /internal/v1/config/ack`——挂既有 `/internal/v1/**` 内部拦截器（StaticTokenInternalAuthChecker，零注册成本，GatewayInternalAuthConfig 已覆盖）；body：`{versionId 或 scopeType+scopeId+version 二选一, targetType(1/2/3 必填), targetId(必填), ackStatus(0/1/2), errorMsg}`；行定位 (version_id, target_type, target_id)，找不到 → 400。语义：ackStatus=1 → ack_status=1、ack_time=NOW、error_msg 清空（任意旧态可置 1）；ackStatus=2 → error_msg 必填、ack_status=2（**仅可从 0/2 覆盖，已生效 1 为终态不被失败回执倒退**——防过期失败回执回归）；ackStatus=0 不接受（回执只有生效/失败两种），400。versionId 与三元组都给时以 versionId 为准校验一致性。
3. **超时扫描定时任务**：Spring `@Scheduled`（无 quartz/xxl 依赖，工程内也无既有先例——`@EnableScheduling` 加在 publish 域 config 类上）+ 配置 `ot.publish.ack-scan-interval-seconds`（默认 60，fixedDelayString）；扫描 `ack_status=0 AND create_time < now - ot.publish.ack-timeout-seconds`（默认 300）→ 置 ack_status=2、error_msg='ACK 超时（超过 N 秒未回执）'。扫描幂等（已置 2 的不重复更新）。
4. **重发接口**：`POST /iot/publish/resend`（iot:publish:resend 4074）——body：`{versionId 或三元组, targetId}`；前置：行 ack_status=2（含超时标红行）或 0（未回执但想重推）→ 重置 ack_status=0、error_msg=NULL、update_time 刷新，并调用下发接缝（第 5 条）；ack_status=1 → 400「已生效无需重发」。
5. **下发接缝 NotifySender**：接口 `NotifySender`（方法：notify(versionId, targetType, targetId) / 语义=触发 config.changed 下发）+ NoOp 实现（javadoc 声明 TASK-0026 交付后接线真实通知——照 0024 PublishConfirmGate NoOp 范式）；发布落行后、重发重置后各调用一次。本期不阻断。
6. **下发状态查询**：`GET /iot/publish/log/list?versionId= 或 scopeType+scopeId+version`（iot:publish:list 复用）——返回行列表（每版本目标数≤3，不分页）+ 每行派生字段 timedOut（ack_status=0 且已超超时窗，覆盖扫描间隔间隙）+ 版本级汇总 publishState（全部行 ack_status=1 → 已生效；存在 2 → 存在失败；否则 → 下发中）——「发布完成以全部 ACK 生效为准」为查询派生口径，不改 iot_config_version.publish_status（TD-0034 生效以发布为准）。
7. 数据权限防线：状态查询/重发为管理面操作——照 0024 范式（网关 scope 校验网关可见；scope_type=2 仅管理员语义）；/internal/v1/config/ack 无管理面数据权限（内部令牌已收口）。
8. 不做前端页面（FEAT-0008 触点=后台）；config.changed 真实通知与 /internal/v1/config/snapshot 下游拉取归 TASK-0026，本任务只留 NotifySender 接缝。

### 测试口径（TDD）

落行：网关 scope 发布→两行（1/网关码+2/接入标识）、计算 scope 发布→一行（3/计算标识）、回滚→新版本同样落行、同事务性；ACK：生效置 1+ack_time+error_msg 清空 / 失败置 2+error_msg 必填 / 已生效不被失败倒退 / 找不到行 400 / ackStatus=0 拒绝 / versionId 与三元组一致性校验；超时扫描：0→2 + error_msg 超时原因 + 幂等；重发：2→0/0→0 重置+NotifySender 调用 / 1 → 400；状态查询：三态汇总口径（全部生效/存在失败/下发中）+ timedOut 派生；内部令牌拦截（无令牌拒绝——既有拦截器行为）；菜单权限 guard 测试 4074。红转绿留痕。

## 验收命令（整体验收）

环境事实（协调者 2026-11-22 实测）：WSL docker wisfactory-mysql/wisfactory-redis 已自启恢复；MySQL 宿主侧 sql-exec.sh 探针 DONE；Redis PONG；8098 空闲（启动前再探、用毕即停）；SQL 无 mysql CLI，bash backend/scripts/sql-exec.sh（多条语句每条一行行尾分号）；TDengine fd 问题未修复——**不碰 TDengine 写路径**。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对 publish_log 表结构（idx_version/idx_ack）+ 4074 一行
3. dev 冒烟（每步原始输出贴回）：发布网关 scope → publish_log 两行（待确认）→ 状态查询三态可查 → 带 token curl /internal/v1/config/ack 置生效 → 行变已生效+ack_time → 再造一行失败回执 → 调短 ack-timeout-seconds 验证扫描置 2（error_msg 含超时）→ 重发 → 重置待确认 → 计算 scope 发布 → 一行 → 重发已生效行 → 400
4. prod 档真实启动回归 → 0 ERROR + /iot/publish/log/list 200（部署/配置类加严：触及 application.yml + SQL）
5. 改动范围核对：`git diff --name-only <开工 BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0025-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0025）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
