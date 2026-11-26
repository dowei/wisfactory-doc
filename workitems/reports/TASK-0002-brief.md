# 实施 brief：TASK-0002（sys_user/dept/role 补 tenant_id 与登录租户注入）

## 批次

- TASK-0002（本批共 1 个，标准任务；与读类 doc 道任务并行——文件集不相交：你只碰 backend/ 与 resources/sql/）
- BASE：`04fc3a5`（主仓库根 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-core/src/main/java/net/otlab/core/**`（多租户插件配置/会话工具，仅新增或最小改动）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/system/**`（user/dept/role/dict/config 实体与登录链路）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/**`（仅 SysTenantController 追加 init-admin 端点，见裁决 5）
  - `backend/ot-admin/src/test/java/**`（新增/调整测试）
  - `resources/sql/upgrade/20261121_tenant_id_columns.sql`（新增）
  - 不碰 frontend/、ingest/、application*.yml（除非新增配置键确需——见裁决 4，允许在 application.yml 增 `ot.tenant.default-code` 一个键）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0002.md`
- 关联对象：同目录 `features\FEAT-0001.md`、`requirements\REQ-0001.md`、`testcases\TC-0001.md`
- 决策：`decisions\ADR-0011.md`（TD-0040 登录租户字段定版）、`decisions\ADR-0002.md`（标签双保险，本任务无关但防混）
- 多租户改造权威：详细设计 V1.1 PDF §0.3「改造三：云端模式下 sys_ 核心表补 tenant_id」表格——用 `python -c` + pypdf 提取 p5（提取口径见 `reports\TASK-0001-brief.md` 环境事实节）
- 登录现状：`backend/ot-admin/src/main/java/net/otlab/admin/module/system/controller/LoginController.java` 及其 service
- MyBatis-Plus 3.5.17 内置多租户插件：`com.baomidou.mybatisplus.extension.plugins.inner.TenantLineInnerInterceptor` + `TenantLineHandler`（读官方文档用法，配置落 `MybatisPlusConfig`）
- 前批交付：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/SysTenantController.java`、`resources/sql/upgrade/20261121_sys_tenant.sql`

## 接口与决策（逐条结论，不再讨论）

1. **表结构增量**（20261121_tenant_id_columns.sql，幂等口径：列存在则跳过——用 information_schema 判存或 MySQL 8 IF NOT EXISTS 语法）：
   - sys_user / sys_dept / sys_role / sys_role_dept / sys_user_role / sys_role_menu 各加 `tenant_id bigint NOT NULL DEFAULT 1`（详细设计 §0.3 推荐直接加字段；存量行回填 default 租户 id=1，DEFAULT 1 兼顾存量兼容）
   - sys_user 唯一约束：删旧 username 唯一键（名以 01_base.sql 实际为准），建 `uk_tenant_username (tenant_id, username)`
   - sys_dict / sys_config 加 `tenant_id bigint NULL`（仅预留列，一期不过滤不启用覆盖，见裁决 3）
   - 不加外键（框架全库无外键风格，详细设计 §0.3）
2. **登录先定租户**（ADR-0011/TD-0040 定版口径）：
   - 登录请求增加可选 `tenantCode` 参数；解析顺序：显式 tenantCode → 否则取配置 `ot.tenant.default-code`（application.yml 新增，默认 `default`——私有化包注入默认租户后用户无感；云端前端传字段，前端字段归 TASK-0056，本任务只交付 API）
   - 按编码查 sys_tenant（interceptor 白名单外，见裁决 3）：不存在或 status=0 → 拒绝登录（BusinessException(400, "租户不存在或已禁用")，不新增枚举）
   - 用户查询显式带 (tenant_id, username)（登录上下文无会话，interceptor 不生效，服务层显式条件）；密码/状态校验沿用现有链路
   - 登录成功：Sa-Token 会话写入 `tenant_id`（与 tenant_code），后续链路统一从会话取
3. **tenant_id 注入拦截器（MyBatis-Plus TenantLineInnerInterceptor）**：
   - `TenantLineHandler.getTenantId()` 从 Sa-Token 会话读 tenant_id；无会话（登录/匿名上下文）返回 null → 插件跳过追加条件（这是登录路径能显式查询的前提，非漏洞——未登录本就无租户数据可查）
   - **白名单式过滤（fail-closed）**：`ignoreTable()` 对**不在**租户清单内的表一律忽略——当前允许清单：sys_user、sys_dept、sys_role、sys_role_dept、sys_user_role、sys_role_menu；表名以 `iot_` 开头的业务表（未来任务交付时自动纳入）；其余表（sys_tenant/sys_menu/sys_module/sys_dict/sys_config/sys_file/sys_operate_log/登录日志/survey_/wechat_ 等全部框架表）一律忽略——绝不用黑名单式（排除法漏一张表就全库污染）
   - 禁止前端传参指定 tenant_id（REQ-0001 核心规则）：user/dept/role 各写接口的实体字段 tenant_id 一律由服务层从会话覆盖，不信任请求体
4. **运营方初始化租户管理员端点（本任务新增，裁决依据见下）**：`POST /tenant/{id}/init-admin`（权限点 `iot:tenant:admin`，菜单行 menu_type=3 登记进 upgrade 脚本 4008 段之后顺延取号——注意 TASK-0055 已占 4008-4016，你从 **4017** 起），body：username/nickname/password（复用现有用户创建服务，tenant_id 强制取路径租户 id 而非会话租户——管理面例外，路径即授权范围；校验 (tenant_id, username) 唯一，重复报 4002）。新增裁决依据：REQ-0001 验收 1「运营方新建租户并初始化管理员后，该管理员可登录且只见本租户数据」要求此流程存在，但现无任何 TASK 承载；本任务 owns user-tenant 写入链路，随任务交付。协调者将留痕 FEAT-0001 关联任务，若你要改接口形态先停下写报告。
   - 越权防护：目标租户必须存在且 status=1；本端点仅运营方可用（权限点已控）。
5. **越租户回归测试**（TDD 先红后绿）：
   - 单测：同名用户两租户并存（(tenant_id,username) 唯一约束路径）、登录显式 tenantCode A/B 各自命中、禁用租户拒登录、租户不存在拒登录
   - 冒烟（dev 档，真实库）：用 init-admin 给租户 A/B 各建同名管理员 operator → operatorA 登录只看 A 数据、operatorB 只看 B（user/list 按 tenant 过滤实证：各只回自己租户的 operator）→ 运营方 admin 仍登录 default → 用 operatorA token 直调 init-admin/其他租户数据 → 403 → 禁用租户 A 后 operatorA 再登录被拒（TC-0001 步骤 2 与「禁用登录拒」在本任务落地）
6. **回归风险**：存量 survey/wechat/system 全部单测必须全绿（1033 例基线）——白名单式 ignoreTable 保证老表零感知；dev 库存量 admin/ry 数据回填 tenant_id=1 后行为不变。

## 验收命令（整体验收）

环境事实：MySQL 127.0.0.1:3307 root/root 库 ot_wisfactory；Redis localhost:6379 无密码；8098 唯一后端端口（启动前探测，被占等待重试，不许杀他人进程；**你自己起的实例完成后必须停掉**）。SQL 走 `bash backend/scripts/sql-exec.sh`（多条语句每条一行行尾分号）。

1. `cd backend && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1033 例 + 新增，Results 段完整贴回）
2. 升级脚本二次运行幂等 → 两次 DONE 无 SQLERROR；核对 `SHOW CREATE TABLE sys_user`（uk_tenant_username 在位）+ `SELECT tenant_id,COUNT(*) FROM sys_user GROUP BY tenant_id`
3. dev 冒烟序列（每步原始输出贴回）：admin 登录 → 建租户 A/B → init-admin 各建同名 operator → operatorA/operatorB 分别登录（tenantCode=A/B）→ 各自 user/list 仅见本租户 → operatorA 直调 `POST /tenant/{B}/init-admin` → 403 → admin 禁用租户 A → operatorA 再登录 → 拒（含 tenantCode 缺省走 default 的 admin 登录回归）
4. prod 档真实启动冒烟（BUG-0001 已修，按原命令）：打包 + prod 启动无 ERROR + admin 登录 + list 200
5. 改动范围核对：`git diff --name-only 04fc3a5..HEAD` 全落在改动范围声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0002-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0002）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
