# 实施 brief：TASK-0008（iot_data_dict 表与字典 CRUD 接口）

## 批次

- TASK-0008（本批共 1 个，标准任务；backend 串行道——前序 TASK-0002 修订轮须已收口）
- BASE：`88caf08`（主仓 HEAD，派发时回填）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/**`（字典域新增 + SysTenantController 路径迁移，见裁决 1）
  - `backend/ot-admin/src/test/java/net/otlab/admin/**`（新增/调整测试）
  - `resources/sql/upgrade/20261121_iot_data_dict.sql`（新增）
  - `frontend/src/api/iot/tenant-api.ts`（仅路径常量一行，见裁决 1）
  - 不碰其他任何文件（ResultCode 无需新增——4002 已存在）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0008.md`
- 关联对象：同目录 `features\FEAT-0003.md`、`requirements\REQ-0003.md`、`testcases\TC-0003.md`
- 约束：`workitems\conventions.md` **CONV-01**（业务数据字典 iot_data_dict 与系统枚举字典 sys_dict 两类不混用——本表是业务字典；引用校验联动归 TASK-0010）
- 表与接口权威（本 brief 已逐字提取贴下，无需再提 PDF；如需上下文可自行提取 p11 §1.5A 与 p23 §3.1 续）
- 前批范式：`SysTenantController/SysTenantServiceImpl/SysTenantServiceImplTest`（CRUD/双道闸/白名单范式）与 `resources/sql/upgrade/20261121_iot_console_menus.sql`（菜单幂等范式）
- 多租户设施：`backend/ot-core/src/main/java/net/otlab/core/tenant/OtTenantLineHandler.java`（iot_ 前缀表自动注入租户条件；tenant_id 服务层从会话覆盖，禁前端传参——TASK-0002 交付，复用不改）

## 接口与决策（逐条结论，不再讨论）

### DDL（详细设计 V1.1 附录 A 六 p35 逐字；脚本里 CREATE TABLE IF NOT EXISTS 幂等落盘）

```sql
CREATE TABLE `iot_data_dict` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '字典 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `data_code` varchar(64) NOT NULL COMMENT '数据编码（租户内唯一）',
  `data_name` varchar(100) NOT NULL COMMENT '数据名称（全厂统一）',
  `category` varchar(32) DEFAULT NULL COMMENT '类别',
  `data_type` varchar(16) NOT NULL COMMENT '数据类型 DOUBLE/INT/BOOL/ENUM',
  `default_unit` varchar(32) DEFAULT NULL COMMENT '默认单位',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_data_code` (`tenant_id`,`data_code`),
  KEY `idx_tenant_category` (`tenant_id`,`category`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='业务数据字典表';
```

### 代码结构

- 新包 `net.otlab.admin.module.iot.dict`（或按现有 iot 包组织，实体 `IotDataDict` 继承 BaseEntity，`@TableName("iot_data_dict")`）；Mapper 空 BaseMapper（租户条件由 TASK-0002 插件自动注入，无需旁路）。
- **路径裁决（本批定版，后续 iot 任务沿用）**：iot 域控制器统一 `/iot/<resource>` 命名空间——详细设计 §3.1 续的 iot 接口本为 `/api/v1/iot/dict/*`（框架无 /api/v1 前缀则取 `/iot/dict`）。**连带迁移：`SysTenantController` 的 `/tenant` → `/iot/tenant`**（消除与框架 `/dict` 等扁平路径的撞名隐患；frontend/src/api/iot/tenant-api.ts 的路径常量同步一行；相关测试路径同步）。前端页面/菜单种子不受影响（走 sys_menu 动态注册与 api 封装）。
- 接口（本任务 5 个；import 接口归 TASK-0009）：
  - `GET /iot/dict/list`（iot:dict:list）分页 + 过滤 dataCode/dataName/category/status
  - `GET /iot/dict/{id}`（iot:dict:query）
  - `POST /iot/dict`（iot:dict:add）data_code 租户内唯一，重复报 **4002**（ResultCode.DUPLICATE_CODE 复用 + DuplicateKeyException 兜底双道闸，照 SysTenantServiceImpl 范式）
  - `PUT /iot/dict`（iot:dict:edit）**data_code 不可改**（编辑请求携带不一致编码 → PARAM_ERROR 拒绝，照租户编码范式）；data_name/category/default_unit 可改；data_type 创建后锁定（同编码拒绝）
  - `DELETE /iot/dict/{id}`（iot:dict:remove）**语义=禁用**（status 置 0，不物理删除、不逻辑删、编码永不复用——详细设计 §1.5A「只增不删」；与 uk_tenant_data_code 的物理唯一天然自洽）
- 写方法 `@Transactional(rollbackFor = Exception.class)`；tenant_id 一律服务层从会话覆盖。
- TDD：先写测试。覆盖：新增成功/重复 4002（活跃+兜底）/编辑改码拒/编辑改 data_type 拒/禁用语义（status=0 且行在且可重复禁用）/列表过滤四维/租户隔离（A 建的 B 看不见——插件注入）。

### SQL 脚本（resources/sql/upgrade/20261121_iot_data_dict.sql）

1. 脚本头：用途/幂等口径/号段占用说明。
2. 建表（上述 DDL + IF NOT EXISTS）。
3. 菜单种子：按钮权限行挂到 0055 已建的字典页菜单（id=4008）之下，**从 4018 起连续取号**：iot:dict:list/query/add/edit/remove 五行（menu_type=3），幂等 INSERT IGNORE，sys_role_menu 授 role_id=1。**不登记 iot:dict:import**（接口归 0009，届时登记，维持「权限标识与接口清单一一对应」）。
4. 执行器注意：语句区不挂 `--` 注释行（sql-exec.sh 已修复支持，但仍保持语句与注释分离纪律）。

## 验收命令（整体验收）

环境事实：MySQL 127.0.0.1:3307 root/root 库 ot_wisfactory；Redis localhost:6379；8098 唯一端口（启动前探测、被占等待、自起实例完成后必须停）。SQL 走 `bash backend/scripts/sql-exec.sh`（多条语句每条一行行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1063 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE 无 SQLERROR；核对 4018-4022 五行 perms 与 (1,4018..4022) 授权
3. dev 冒烟（每步原始输出贴回）：admin 登录（/iot/tenant 路径下 list 回归）→ 建字典 ELEC_CONS/DOUBLE → 重复 4002 → 编辑改 data_code 被拒、改 data_name 成功 → DELETE 禁用后 GET 详情 status=0 且行仍在 → 重建同码 → 4002（编码不复用）→ category/status 过滤 → 建租户 T+init-admin+operator 登录建字典 → admin 会话 list 看不到 T 的字典（租户隔离）→ operator list 只见 T 的
4. prod 档真实启动回归 → Started 无 ERROR + /iot/tenant list 200
5. 前端回归（路径迁移验证）：`cd frontend && pnpm build:test` EXIT=0（tenant-api 路径常量改动编译通过）
6. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0008-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0008）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
