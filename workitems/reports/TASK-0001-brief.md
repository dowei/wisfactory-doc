# 实施 brief：TASK-0001（sys_tenant 租户表与租户管理接口）

## 批次

- TASK-0001（本批共 1 个，标准任务）
- 验收粒度：完成即测，一个 subagent 跑一次整体验收。
- BASE：`35d22f6`（主仓库根；验收时对比改动范围用）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/**`（新增）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增）
  - `backend/ot-core/src/main/java/net/otlab/core/result/ResultCode.java`（仅追加枚举值）
  - `resources/sql/upgrade/20261121_sys_tenant.sql`（新增）
  - 其余一律不碰（frontend/、ingest/、application*.yml 均不在本批）。

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0001.md`
- 关联对象：同目录 `features\FEAT-0001.md`、`requirements\REQ-0001.md`、`testcases\TC-0001.md`
- 框架约定：`D:\workspace\WisFactory\backend\AGENTS.md`（包结构/Controller/Service/Entity/测试范式全在里面）
- CRUD 模板：`backend\ot-admin\src\main\java\net\otlab\admin\module\system\controller\SysDictController.java` 与对应 Service
- 测试模板：`backend\ot-admin\src\test\java\net\otlab\admin\module\system\controller\` 下任意 *Test.java（standalone MockMvc + Mockito，不加载 Spring 上下文）
- 菜单种子范式：`resources\sql\02_survey.sql`（sys_module+sys_menu 段）与 `resources\sql\upgrade\20260919_sys_menu_role_button_perms.sql`（幂等 INSERT IGNORE + 固定 id + 脚本头说明）
- 登录接口（冒烟用）：`backend\ot-admin\src\main\java\net\otlab\admin\module\system\controller\LoginController.java`

## 接口与决策

### DDL（详细设计 V1.1 附录 A 逐字；脚本里原文落盘，CREATE TABLE 加 IF NOT EXISTS 幂等）

```sql
CREATE TABLE `sys_tenant` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '租户 ID',
  `tenant_code` varchar(32) NOT NULL COMMENT '租户编码（全局唯一，创建后不可改）',
  `tenant_name` varchar(100) NOT NULL COMMENT '租户名称',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用',
  `point_quota` int DEFAULT NULL COMMENT '点位配额（个），空或 0 不限',
  `storage_quota_gb` int DEFAULT NULL COMMENT '存储配额（GB）',
  `expire_time` datetime DEFAULT NULL COMMENT '到期时间，空为长期',
  `contact_name` varchar(50) DEFAULT NULL COMMENT '联系人',
  `contact_phone` varchar(32) DEFAULT NULL COMMENT '联系电话',
  `remark` varchar(500) DEFAULT NULL COMMENT '备注',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_code` (`tenant_code`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='租户表';
```

### 代码结构

- 新包 `net.otlab.admin.module.iot`：controller / entity（dto、vo 子包按需）/ mapper / service / service.impl。`@MapperScan("net.otlab.admin.module.**.mapper")` 已覆盖，无需改配置。
- 实体 `SysTenant` 继承 `net.otlab.core.base.BaseEntity`（审计五字段自动有），`@TableName("sys_tenant")`，`@TableId(type = IdType.AUTO)`。
- Controller：`@RequestMapping("/tenant")` 扁平路径；统一 `Result<T>`；`@SaCheckPermission("iot:tenant:*")`（权限前缀按 TASK 正文用 iot:tenant:*，不用 system:*）；写操作加 `@OperateLog`；分页参数 pageNum/pageSize。
- 接口清单：
  - `GET /tenant/list`（iot:tenant:list）分页 + 条件 tenantCode/tenantName/status
  - `GET /tenant/{id}`（iot:tenant:query）
  - `POST /tenant`（iot:tenant:add）新增；tenant_code 重复 → 业务码 4002
  - `PUT /tenant`（iot:tenant:edit）编辑；配额字段（pointQuota/storageQuotaGb/expireTime）随本接口下发，不单设权限点；请求携带的 tenantCode 与库中不一致 → 拒绝（PARAM_ERROR），创建后编码不可改
  - `PUT /tenant/{id}/status`（iot:tenant:status）启停，body 仅接受 status=0|1
  - `DELETE /tenant/{ids}`（iot:tenant:remove）逻辑删除（@TableLogic）；管理页不暴露删除入口是前端 TASK-0056 的事
- `ResultCode` 追加 `DUPLICATE_CODE(4002, "编码重复")`（详细设计 §3.4 错误码分段：4001-4099 参数与校验，4002=编码重复；此码后续 iot_data_dict 等复用，语义取「编码重复」不取「租户编码重复」）。
- 重复码判定：service 预检活跃行（友好报 4002）+ 捕获 `DuplicateKeyException` 兜底映射 4002（兜底覆盖「已删除行仍占用编码」场景——uk_tenant_code 是物理唯一索引，逻辑删不释放编码，删除后编码不复用是详细设计 §1.1 的既定纪律）。
- 业务异常统一 `throw new BusinessException(4002, "...")` / `BusinessException(ResultCode.PARAM_ERROR.getCode(), "租户编码不允许修改")`，由 GlobalExceptionHandler 映射进 Result.code。
- 写方法加 `@Transactional(rollbackFor = Exception.class)`；方法命名 selectXxxList/insertXxx/updateXxx/deleteXxxById。
- TDD：先写测试再实现。service 单测覆盖：新建成功、重复码 4002（活跃行与已删行占码两场景）、编辑改码被拒、启停仅 0/1、配额字段写入；controller 测试覆盖路由/权限注解/参数校验。

### SQL 脚本（resources/sql/upgrade/20261121_sys_tenant.sql）

1. 脚本头写明：用途、适用环境、幂等口径（CREATE TABLE IF NOT EXISTS + INSERT IGNORE）、占用号段。
2. sys_tenant 建表（上面的 DDL）。
3. `sys_module` 新增一行：id=4、code='iot'、名称'智慧工厂'（已有 1/2/3，先 INSERT IGNORE 占 4）。
4. `sys_menu` 租户管理段：目录+页面+按钮权限点（iot:tenant:list/query/add/edit/remove/status），**显式 id 用 4000-4099 号段**（sys_menu 上游 AUTO_INCREMENT 从 3000 起、survey 已占其段），幂等 INSERT IGNORE，样式照抄 02_survey.sql。
5. `sys_role_menu` 授 role_id=1（超级管理员），照 20260919 脚本范式。
6. 默认租户种子：`INSERT IGNORE INTO sys_tenant (id, tenant_code, tenant_name, status) VALUES (1, 'default', '默认租户', 1)`——私有化单租户初始化约定（REQ-0001 / 详细设计 §0.3），TASK-0002 登录租户注入依赖此行。

## 歧义裁决（逐条结论，不再讨论）

1. TASK 实施步骤 3「联调管理页（FEAT-0018 侧）」**不在本任务**：前端页归 TASK-0056。本任务交付可用接口即视为该步骤在本侧完成，收尾时按此口径勾选。
2. 验收标准里「禁用租户后登录/上报/对外 API 全拒」的端到端验证归 TASK-0002（登录注入）与接入/对外各任务；本任务只交付 status 字段与启停接口。TC-0001 是 FEAT 级联调用例，不作为本任务验收门。
3. 权限前缀用 `iot:tenant:*`（TASK 正文明确），不沿用框架 `system:*` 惯例。
4. 菜单 id 号段 4000-4099、module_id=4、默认租户 id=1：本 brief 定版，后续任务不得再用此号段。
5. 配额设定并入新增/编辑接口；启停独立接口与独立权限点。

## 验收命令（整体验收，五条全过）

环境事实：MySQL 127.0.0.1:3307 root/root，库 `ot_wisfactory` 已由 01/02/03 脚本初始化（43 表）；Redis localhost:6379 无密码；后端端口 8098。**本机没有 mysql CLI**，SQL 一律走 `bash backend/scripts/sql-exec.sh`（用法见脚本头；多条语句必须每条一行、行尾分号）。

1. **单测 + 存量回归**
   - 命令：`cd backend && mvn test -pl ot-admin`
   - 期望：BUILD SUCCESS；`Tests run: N, Failures: 0, Errors: 0`（N 含新增 SysTenant 系列，且存量测试无回归——原始输出整段贴回）
2. **升级脚本二次运行幂等**
   - 命令：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261121_sys_tenant.sql` 连跑两次，再核对：
     `bash backend/scripts/sql-exec.sh "SELECT id,tenant_code,tenant_name,status FROM sys_tenant;"`（另验 SHOW CREATE TABLE 里 uk_tenant_code 存在）
   - 期望：两次均 DONE 无 SQLERROR；核对输出含 id=1 default 行
3. **dev 档接口冒烟**（先起服务再逐条 curl，每条原始输出贴回）
   - 启动：`cd backend && mvn spring-boot:run -pl ot-admin`（后台），就绪判据：8098 有响应
   - 登录 admin/admin123 拿 token（接口形状读 LoginController），后续带 token
   - 序列：新建租户 acme（pointQuota=100）→ 200；同码重建 → body code=4002；list 含 acme；GET /{id} 字段齐全；PUT 改 tenantCode → 被拒（400）；PUT 改 pointQuota=200 → 200 且回读一致；PUT status=0 → 200 回读 status=0；DELETE → 200；同码重建 → 4002（编码不复用）
4. **prod 档真实启动冒烟**（与 3 串行：先停 dev 实例再起 prod，避免 8098 冲突）
   - 命令：`cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root REDIS_HOST=localhost java -jar ot-admin/target/ot-admin-*.jar --spring.profiles.active=prod`（jar 名以 target 实际为准）
   - 期望：启动日志无 ERROR、端口 8098 就绪；带 token `GET /tenant/list` 一次 200。此项不重复整条 CRUD。
5. **改动范围核对**
   - 命令：`git diff --name-only 35d22f6..HEAD`（在主仓库根）
   - 期望：输出文件全部落在「改动范围」清单内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0001-report.md`。必须包含五项：

1. 改动文件清单
2. 每条验收命令的原始输出（不要只写「通过」）
3. 未做项与原因
4. 偏离计划之处
5. 遗留问题

## 契约

- **no-subagents**：你不得再派 subagent。
- 只改本批 TASK 范围内的文件；不得把 TASK `状态` 置为「已完成」，那由协调者收尾。
- 不手写索引行；不得改本批之外的对象（含 workitems 库任何文件——报告写进 reports/ 即可）。
- 测试或验收失败先定位**根因**再修——禁止为了让输出匹配而改测试、改期望或打补丁。
- 每个修复要能说明**为什么现在对了**：附复现测试（先红后绿）或推理链，写进报告。

## 收尾回写

- 不要回写 TASK 详情——勾选 `计划子任务`、`修改内容` / `任务结果` 由协调者按报告收尾。
