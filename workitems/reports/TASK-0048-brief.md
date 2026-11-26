# 实施 brief：TASK-0048（iot_status_rule 配置接口与校验 + 状态编码稳定字典）

## 批次

- TASK-0048（本批共 1 个，标准任务 2 人天；backend 串行道——配置链第三环）
- BASE：以开工时主仓 HEAD 为准（记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/**`（状态规则+状态字典域新包）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_status_rule_domain.sql`（新增）
  - 不碰 frontend/、ingest/、模板域（只读 template_id 校验）、数据项域（引用存在性只读查询）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0048.md`
- 关联对象：同目录 `features\FEAT-0016.md`、`requirements\REQ-0007.md`（状态判定/直绑映射/稳定字典验收）、`testcases\TC-0016.md`
- 决策：`decisions\ADR-0009.md`（**TD-0013 状态编码独立稳定字典：只增不删、禁用不复用序号、历史 st_item_value 序号不漂移**；TD-0014 直绑映射）
- 语法口径权威：`RelativeRefExpander.java`（0013——condition_expr 相对引用语法同口径）；规则表 DDL=附录 A p35 逐字（brief 已提取贴下）
- 表结构缺口裁决：**状态编码稳定字典表上游无 DDL**（ADR-0009 TD-0013 只给语义）——本任务定义并登记「与上游差异：新增表，建议回写上游」（照 0021 补 tenant_id 先例）
- 既有范式：iot/template（0011 批量保存）、iot/dataitem（0013 Expander）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_status_rule_domain.sql，幂等 + 头注）

1. `iot_status_rule`：附录 A p35 逐字（uk_template_priority (template_id, priority)——优先级重复物理拒绝；KEY idx_tenant_template）。
2. `iot_status_dict`（**本任务定义，与上游差异登记**——TD-0013 语义落表）：
```sql
CREATE TABLE `iot_status_dict` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '状态字典 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `seq` int NOT NULL COMMENT '稳定序号（只增不复用——历史 st_item_value 序号不漂移的锚）',
  `status_code` varchar(32) NOT NULL COMMENT '状态编码',
  `status_name` varchar(50) NOT NULL COMMENT '状态名称',
  `color` varchar(16) DEFAULT NULL COMMENT '展示颜色',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用（禁用不复用序号）',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_status_code` (`tenant_id`,`status_code`),
  UNIQUE KEY `uk_tenant_seq` (`tenant_id`,`seq`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='状态编码稳定字典表（TD-0013）';
```
   - 序号口径：seq 由服务层取「本租户当前 max(seq)+1」（禁用行序号不复用；物理唯一保漂移不可能）；**不物理删除**（逻辑删也禁用——字典只增不删+禁用，删除接口不存在）。
3. 菜单种子 **4063 起**连续（4062 已占）：iot:statusrule:list/query/add/edit/remove + iot:statusdict:add/disable（状态字典新增/禁用）共七行挂 4009 模板管理页下（状态规则入口挂模板页内——0056 占位按钮对应此页），幂等 + 超管授权。

### 代码结构（/iot/statusrule 段）

- **规则接口**（挂模板）：`GET /iot/statusrule/list?templateId=`（iot:statusrule:list）、`POST /iot/statusrule`（iot:statusrule:add——单条，priority 同模板唯一冲突即 uk 物理+预检 4002 同款）、`PUT /iot/statusrule`（iot:statusrule:edit——condition_expr/status_code/status_name/color/enabled 可改；template_id/priority 不可改——调优先级=删旧建新，报告说明）、`DELETE /iot/statusrule/{ids}`（iot:statusrule:remove——逻辑删）、**启停**（edit 承载 enabled 0/1）。
  - **规则条数上限 ≤10**（任务正文）：同模板活跃规则数 ≥10 时新增拒（400 参数错误文案「状态规则上限 10 条」）。
  - **condition_expr 校验**：Expander 同口径语法（花括号/字符集/前缀白名单/长度 ≤1000/括号配平）+ 相对引用存在性（模板数据项范围——引用不存在 4001 拒存）；**表达式真语义校验归计算引擎**（0036 同款口径登记）。
  - status_code 必须在 iot_status_dict 中存在且启用（未登记 → 拒并提示先建状态编码）。
- **状态字典接口**：`GET /iot/statusdict/list`（iot:statusdict:list 或复用 iot:statusrule:list——**新设 iot:statusdict:list 权限行**照种子裁决）、`POST /iot/statusdict`（iot:statusdict:add——只增）、`PUT /iot/statusdict/{id}/disable`（iot:statusdict:disable——禁用不复用序号；**无删除/无编辑名称接口**——只增不删纪律，名称修正属语义变更须评估漂移，本期不开放，报告登记）。
- **直绑映射配置**（TD-0014 挂模板）：`PUT /iot/statusrule/binding-map`（iot:statusrule:edit）——点位枚举值→状态编码映射（body：pointId + 枚举值映射数组 [{enumValue, statusCode}]；statusCode 校验字典存在启用；映射随规则配置存储（JSON 列？上游无列——**裁决：映射存 iot_status_rule 表新增 binding_map json DEFAULT NULL 列（「与上游差异」登记）或独立小表——选前者（轻量，随规则行），脚本头登记）。
- **规则随配置发布下发**：规则行 config_version 语义与「发布生效」口径照 0021/0023（保存≠生效，登记 0024 发布链）。
- 白名单构建 + `@Transactional` 照范式；tenant_id 会话覆盖；数据权限裁剪（规则挂模板——列表按模板读，模板可见性决定——detail/list 校验模板存在同租户即可，写路径 resolve 防线照 0021 范式对模板判定）。

### 测试口径（TDD）

规则：priority 重复 uk+预检 4002/上限 10 条拒/condition_expr 语法拒绝（Expander 同口径）/引用不存在 4001/status_code 未登记拒/启停/直绑映射保存与校验/随模板读。
字典：新增只增（seq 自增不复用断言——禁用后再增新码 seq 跳号）/禁用不复用序号/status_code 重复拒/名称无编辑接口。
冒烟（dev）：admin 建状态编码（RUNNING/STOPPED/ALARM）→ 模板建规则（condition_expr 引用模板数据项）→ priority 冲突 4002 → 超 10 条拒 → 直绑映射保存 → 禁用编码 → 引用该编码的新规则拒。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线以开工实测为准 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对两表结构 + 4063-4069 七行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + statusrule/list 200
5. 改动范围核对：`git diff --name-only <开工 BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0048-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0048）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
