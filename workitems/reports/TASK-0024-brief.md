# 实施 brief：TASK-0024（配置版本快照生成——SHA-256 + 对象存储）

## 批次

- TASK-0024（本批共 1 个，标准任务 4 人天；backend 串行道——发布链第一环，0025/0026 的地基）
- BASE：以开工时主仓 HEAD 为准（记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/**`（发布域新包）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/credential/**`（凭据排除校验若需触达——最小改动，否则不碰）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_publish_domain.sql`（新增）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增对象存储配置键）
  - 不碰 frontend/、ingest/、openapi 域（0051 已交付）、dataperm

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0024.md`
- 关联对象：同目录 `features\FEAT-0008.md`、`requirements\REQ-0033.md`（验收：发布生成单调新版本/哈希可验可取回/快照无凭据字段/并发发布 5001）、`testcases\TC-0008.md`
- 决策：`decisions\ADR-0008.md`（快照按实例有效版本组装——锁定实例按其版本；模板三档变更确认门）；`decisions\ADR-0006.md`（快照不含凭据——0019 已登记凭据表为显式排除项）
- 表结构权威：附录 A p33 iot_config_version（scope_type 1 网关 2 计算/scope_id/version 单调递增/snapshot_hash/snapshot_uri/publish_status 1 已发布 2 已回滚——brief 已提取贴下）
- 数据源（快照组装的表）：iot_point（0021）/iot_channel（0018）/iot_data_item（0013）/iot_metric（0043）+ iot_status_rule/binding_map（0048）+ iot_status_dict（0048）——全部已交付；对象存储：框架 SysFile/存储服务（grep ot-core storage LocalStorageService/MinioStorageService——0011 范式）或本地目录配置
- 口径基准：快照 Schema 契约 `resources/contract/config-snapshot-schema.md` v1.0.1（**字段级 Schema 已派生定版——快照体组装按它实现**）；错误码 5001 版本冲突（ResultCode 已有）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_publish_domain.sql，p33 iot_config_version 逐字 + 幂等 + 头注）

```sql
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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='配置版本表';
```
（与上游差异：上游仅列部分索引——uk_scope_version 为单调递增并发防护的物理保证，登记头注。）菜单种子 **4070 起**连续（4069 已占）：iot:publish:list/query/publish/rollback 挂 4015 配置发布页下，幂等 + 超管授权。

### 代码结构（/iot/publish 段）

1. **快照组装器**：按 scope（网关：该网关下点位/通道/数据项全链；计算：全租户指标/数据项）组装 JSON 快照体——**严格按 config-snapshot-schema.md v1.0.1 字段级 Schema**（camelCase 命名/类型映射/schema_version 一并下发）；组装来源=实例有效版本（锁定实例按其版本，ADR-0008——实例 locked 字段与 applied_template_version 已由 0012/0011 建列）。
2. **SHA-256 哈希 + 对象存储**：快照体序列化（确定性序列化——键序稳定）→ SHA-256 → 存对象存储（**存储裁决：本地目录（ot.publish.storage-path 配置，默认 ./data/snapshots）一期——MinIO 对象存储二期登记**；文件名 `<scope_type>-<scope_id>-<version>.json`）；snapshot_uri 存相对路径。
3. **版本单调递增**：同 (tenant, scope_type, scope_id) 并发发布防护——**INSERT 后 SELECT 末版**竞态用 uk_scope_version 物理唯一兜底（version=last+1 计算后 INSERT 撞 uk → 5001 重试一次或报冲突，报告说明取舍）。
4. **发布接口**：`POST /iot/publish/publish`（iot:publish:publish）——body: scopeType/scopeId；流程：组装 → 哈希 → 存储 → 版本落库（publish_status=1, publish_time=NOW）；**并发发布 5001**（验收项）；scope 校验（scope_type=1 时 scope_id 必须是存在启用的网关；2 时无实体校验）。
5. **回滚接口**：`POST /iot/publish/rollback`（iot:publish:rollback）——body: scopeType/scopeId/targetVersion → 新版本行 publish_status=1 指向 targetVersion 快照（版本单调递增但快照复用——快照内容按 targetVersion 取回重存 or uri 指向旧快照——**裁决：新版本行 snapshot_uri 复用旧快照文件**，哈希同——报告说明）；回滚目标版本不存在 → 400。
6. **快照取回**：`GET /iot/publish/snapshot?scopeType=&scopeId=&version=`（iot:publish:query）——从对象存储取回快照体 + 哈希核对（取回哈希与落库哈希不一致 → 500 告警日志——存储损坏探测）。
7. **影响预览接入**：发布接口入参 `confirmToken`？——**裁决：TASK-0015（模板变更强制影响预览+二次确认）未交付——本任务在发布流程中预留确认门接口位（PublishConfirmGate 接口 + NoOp 实现，javadoc 声明 0015 交付后接线），本期发布不阻断**；报告登记接线义务。
8. **快照无凭据**：组装器白名单字段（严格按 Schema v1.0.1——凭据表不在数据源清单）+ 组装后终检（快照体 grep 凭据关键词——credential/password/secret/token 非法键即组装失败 fail-closed，照 0018 双保险）。
9. 数据权限裁剪：发布/回滚为写操作（运营方动作）——resolve 防线照范式（scope 对象可见性：网关 scope 校验网关可见；计算 scope=全租户仅管理员语义——scope_type=2 校验调用方为管理员角色，报告说明）；列表/详情裁剪照 0021 范式。

### 测试口径（TDD）

组装器：按 Schema v1.0.1 字段完整性（点位/通道/数据项/公式/指标/状态规则与映射逐段断言）/锁定实例按版本/凭据排除终检/确定性序列化（两次组装哈希一致）；版本单调（并发发布 5001——两线程同 scope）/回滚复用快照/取回哈希核对/存储损坏探测；确认门 NoOp；数据权限防线红转绿。
冒烟（dev）：建网关+通道+点位+数据项（0017-0023 链）→ 发布网关 scope → 版本 1 落库+文件存在+哈希可验 → 变更点位 → 再发布 → 版本 2 → 回滚到 1 → 快照取回与版本 1 哈希一致 → 并发发布 → 5001 → 凭据关键词终检（手工塞含凭据行 → 发布组装失败）。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线以开工实测为准 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对表结构（uk_scope_version）+ 4070-4073 四行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + publish/list 200
5. 改动范围核对：`git diff --name-only <开工 BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0024）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
