# 实施 brief：TASK-0015（模板变更分档判定与影响预览——发布确认门真接线）

## 批次

- TASK-0015（本批共 1 个，标准任务 3 人天；backend 串行道）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/**`（分档引擎/diff/版本历史/影响预览）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/gate/**`（NoOp→真实现替换）与 `publish/service/impl/IotPublishServiceImpl.java`（仅 gate 调用点如需适配，最小改动）
  - `backend/ot-core/src/main/java/**/ResultCode.java`（**新增 5002**——PublishConfirmGate javadoc 声明归 0015 的唯一 ot-core 触点）
  - `backend/ot-admin/src/main/resources/application.yml`（仅新增 ot.publish.confirm-token-ttl-seconds 一键）
  - `resources/sql/upgrade/20261123_iot_template_version.sql`（新增）
  - `backend/ot-admin/src/test/java/**`、`backend/ot-core/src/test/java/**`（对应测试）
  - 不碰 publish 其余端点、前端、ingest、openapi、dataperm

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0015.md`
- 决策权威：`decisions\ADR-0008.md`（三档定义与政策——brief 已消化，逐条结论见下）；`decisions\ADR-0006.md`
- 关联：`features\FEAT-0005.md`、`requirements\REQ-0006.md`、`testcases\TC-0005.md`、`features\FEAT-0008.md`（5002 语境）
- 接缝：`D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\publish\gate\PublishConfirmGate.java`（javadoc 三条接线义务=你的任务清单）+ `NoOpPublishConfirmGate.java`
- 上游代码（开工先读）：iot/template/**（IotModelTemplate.version / IotModelInstance.appliedTemplateVersion+locked 两列已在位——p31 无此两列的登记差异）、iot/publish/service/impl/IotPublishServiceImpl.java（gate 调用点+组装按实例有效版本）、iot/publish/dto/PublishForm.java
- 上游报告：`reports\TASK-0024-report.md`（gate 占位语义/5002 登记）、`reports\TASK-0026-report.md`（afterCommit 纪律）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261123_iot_template_version.sql，新增模板版本历史表 + 幂等 + 头注）

```sql
CREATE TABLE `iot_model_template_version` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '版本快照 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `template_id` bigint NOT NULL COMMENT '模板 ID',
  `version` int NOT NULL COMMENT '模板版本号（模板内单调递增）',
  `content` json NOT NULL COMMENT '该版本模板全量内容快照（数据项/状态规则等）',
  `tier_summary` varchar(200) DEFAULT NULL COMMENT '相对上一版本的档位摘要（如 T2:2,T3:1）',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_template_version` (`tenant_id`,`template_id`,`version`),
  KEY `idx_template` (`tenant_id`,`template_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='模板版本历史表（ADR-0008 落地件）';
```

（头注登记：附录 A 无此表——ADR-0008 落地件新增，登记差异；content 形态=模板域既有结构的 JSON 序列化，键序确定性。菜单种子：无新增——影响预览复用既有模板查询权限。）

### 分档规则引擎（表驱动，ADR-0008 逐条落地）

- **T1（随发布自动生效）**：新增模板数据项；改名称/说明/显示顺序/状态颜色等展示属性；新增只兜底的状态规则
- **T2（随发布生效，但强制影响预览+二次确认）**：改默认公式；改有效范围默认值；改计算周期；改状态规则门限/优先级；改可覆盖参数默认值；改告警阈值默认值
- **T3（永不自动，逐台确认迁移——不经发布门）**：删除模板数据项；改数据类型（瞬时↔累计）；改取值方式（直绑↔公式↔汇总）
- **data_code 修改：直接禁止**（400，与点位/网关编码不可改同原则；换编码=新增一项+停用旧项）
- **逐条目判档**：一次保存含多类变更时每条 diff 独立判档（不取最高档合并）；发布确认门按「本次待发布内容相对实例有效版本的变更中存在 T2 条目」触发
- 规则实现为**表驱动**（变更字段→档位映射表，便于演进，TASK 技术要点明文）

### diff 与影响预览

- **diff 生成**：模板保存升版时与上一版本 content JSON 对比 → 变更条目列表（对象类型/dataCode/字段/旧值/新值/档位），随版本历史行存储摘要（tier_summary）
- **影响预览接口**：`GET /iot/template/impact-preview?templateId=`（复用既有模板查询权限，iot:template:* 不新增种子）→ `{targetVersion, diff:[...], affectedInstanceCount, lockedInstanceCount}`——受影响实例数=同租户引用该模板且启用中的实例（未锁定实例跟随发布、locked=1 实例单列不跟随——ADR-0008 锁定口径）
- **二次确认协议**：预览响应携带 `previewToken`（Redis key `iot:pub:confirm:{tenantId}:{scopeType}:{scopeId}:{targetVersion}`，value=待发布内容摘要哈希，TTL `ot.publish.confirm-token-ttl-seconds` 默认 300）→ 发布请求 `PublishForm` 扩 `confirmToken` 字段 → gate 真实现校验：token 存在+内容摘要一致 → true；缺失/过期/内容漂移 → false（发布流程抛 BusinessException）→ **ot-core ResultCode 新增 5002 PUBLISH_NOT_CONFIRMED（下发未确认）**
- **gate 真实现替换 NoOp**（保持接口签名不动）：实现内部判档——仅二档触发确认；一档放行；三档不经本门（逐台迁移走 0016 待升级清单，快照按实例 applied_template_version 组装天然不包含未升级实例——0024 已按实例有效版本组装，天然正确，gate 不重复处理）
- 数据权限：预览走模板域既有范式；gate 在 publish 域内不新增面

### 测试口径（TDD）

分档规则表逐档用例（T1/T2/T3 各代表性变更+混合保存逐条目判档+data_code 禁止 400）；diff 生成正确性（增/删/改三形态+展示属性归 T1）；影响预览计数（未锁定/locked 分列）；确认门：二档变更发布无 token→5002 红→带 token 绿；token 过期/内容漂移拒绝；一档/三档放行；NoOp→真实现替换后 0024/0025/0026 既有发布测试回归全绿（一档路径不回归）；版本历史表幂等二次执行。

## 验收命令（整体验收）

环境事实（协调者 2026-11-23 实测）：WSL docker wisfactory-mysql（8.0.42@3307 root/root ot_wisfactory）/wisfactory-redis（6379）；Redis backend 用 db0；sql-exec.sh 每条语句单独一次调用；8098 启动前探测、用毕即停；TDengine 本批不涉。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对 iot_model_template_version 表结构（uk_template_version）
3. dev 冒烟（每步原始输出贴回）：模板改一条公式（T2）保存升版 → 影响预览（实例数+diff+previewToken）→ 不带 token 发布 → **5002** → 带 token 发布 → 200；模板改展示属性（T1）→ 发布无 token 直接 200；锁一个实例 → 预览 lockedInstanceCount 分列正确
4. prod 档真实启动回归 → 0 ERROR（触 yml+SQL，部署/配置类加严）
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内（ot-core 触点仅 ResultCode 一处）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0015）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
