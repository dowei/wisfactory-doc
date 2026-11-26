# 实施 brief：TASK-0055（控制台框架、菜单种子与权限点注册 · 前端）

## 批次

- TASK-0055（本批共 1 个，标准任务；与 backend 道 TASK-0072 并行——文件集不相交：你只碰 frontend/ 与新增的 iot 菜单种子 SQL）
- BASE：`98c155e`（主仓库根 HEAD）
- 改动范围（越界即违约）：
  - `frontend/src/views/iot/**`（新增）
  - `frontend/src/api/iot/**`（新增）
  - `frontend/src/router/**`（仅追加 iot 路由段）
  - `resources/sql/upgrade/20261121_iot_console_menus.sql`（新增）
  - 其余一律不碰（不改 directives/store/layout 既有文件、不碰 backend 任何代码；若路由/菜单机制确需改既有文件，停下写进报告「偏离」并说明原因，等协调者裁决）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0055.md`
- 关联对象：同目录 `features\FEAT-0018.md`、`requirements\REQ-0034.md`、`testcases\TC-0018.md`
- 前端工程约定：`D:\workspace\WisFactory\frontend\AGENTS.md`（若存在）与 `frontend/src/` 现有结构——视图范式抄 `src/views/system/`（如 dict 页），权限指令= `src/directives/privilege.ts`，路由在 `src/router/`
- 后端已交付的租户域（本任务演示权限挂载的真实接口）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/SysTenantController.java`
- 菜单种子范式：`resources/sql/02_survey.sql`（菜单段）与 `resources/sql/upgrade/20261121_sys_tenant.sql`（幂等 INSERT IGNORE + 固定号段，TASK-0001 已交付 4000-4007）

## 接口与决策

1. **菜单号段裁决（覆盖任务正文与 FEAT-0018 的「3000-3999 段」规划文本）**：iot 菜单一律用 **4000 段（4000-4999）**。TASK-0001 已占 4000-4007（智慧工厂目录 4000 + 租户管理页 4001 + 六权限按钮 4002-4007，module_id=4 已建）。本任务从 **4008 起**连续取号；3000-3999 实测空闲但不再启用（避免 iot 两段并存）。此为计划文本过时的纠偏，随收尾留痕，不回改 FEAT/TASK 正文。
2. **菜单种子范围**：只建「控制台树骨架」——模块目录已存在（4000），本任务补：字典管理/模板管理/实例管理（模型）/点位管理/网关管理/通道管理/指标管理/配置发布/导入向导 各目录或页面菜单（menu_type 1/2，parent 挂 4000 下，具体层级照 system/survey 现有树样式）。**不建按钮权限行（menu_type=3）**：原则「权限标识与接口清单一一对应」——后端目前只有租户域有真实接口（其按钮行 4002-4007 已登记），其余页面的按钮权限行由各自后端任务交付时登记（后端有守卫测试强制：SaPermissionRegistryConsistencyTest）。
3. **前端骨架范围**：`src/views/iot/` 下按上述页面建占位视图（空态列表骨架即可，真实表单/表格归 TASK-0056/0070）；`src/api/iot/` 建 API 封装（现阶段只有 tenant 是真实的，照 `src/api/system/` 现有封装样式）；路由注册进 iot 段（懒加载、与 system 路由样式一致）；**租户列表页做成可用的最小骨架**（列表 + 新增/编辑/启停/删除按钮，按钮全部用 privilege 指令绑定 iot:tenant:* 权限位）作为权限显隐的演示页——完整租户页归 TASK-0056。
4. **权限机制**：权限由后端下发前端只做显隐（REQ-0034）；前端拦截不做、越权全靠后端 403。
5. 菜单种子脚本幂等：INSERT IGNORE + 显式 id（4008 起连续），脚本头写明用途/号段占用/幂等口径，样式照 20261121_sys_tenant.sql；sys_role_menu 授 role_id=1 照同范式。

## 验收命令（整体验收）

环境事实：后端 dev 档启动后 8098；MySQL 127.0.0.1:3307 root/root 库 ot_wisfactory；无 mysql CLI，SQL 走 `bash backend/scripts/sql-exec.sh`（多条语句每条一行行尾分号）。

1. **前端构建**：`cd frontend && pnpm build:test` → 期望 built 成功无 error（chunk 大小 warning 可忽略）
2. **菜单种子幂等**：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261121_iot_console_menus.sql` 连跑两次均 DONE 无 SQLERROR；核对 `SELECT id,menu_name,menu_type,module_id FROM sys_menu WHERE id>=4008;` 输出与设计一致
3. **权限标识一一对应核对**：`grep -rn "SaCheckPermission" backend/ot-admin/src/main/java --include="*.java" | grep -o 'iot:[^"]*'` 与 SQL 中 menu_type=3 的 perms 全集比对 → 两侧集合相等（当前应恰好是 iot:tenant:* 六个）
4. **普通角色按钮不可见 + 直调 403**：起 dev 后端（`cd backend && mvn spring-boot:run -pl ot-admin`，8098 端口纪律见下）→ 用 admin 在界面/接口上建一个「iot 查看角色」（不授任何 iot:* 权限）+ 一个测试用户挂该角色 → 该用户登录（POST /auth/login/test，dev 档免验证码）→ 拿其菜单列表接口响应（无 iot 菜单或仅有目录）→ 用其 token 直调 `GET /tenant/list` → **403**；再用 admin token 直调同接口 → 200。原始输出贴回。
5. **前端显隐演示**：dev 起 frontend（`pnpm dev`，8099）用浏览器以测试用户登录，进入智慧工厂菜单 → 租户列表页按钮不可见；admin 登录则按钮可见。截图或 DOM 断言证据贴报告。（若浏览器自动化受阻，退化为：以测试用户 token 调后端菜单接口证明其无 iot:tenant:* 权限位 + privilege 指令实现代码引用核对，如实写明证据形态。）

**端口纪律**：8098（后端）与 8099（前端 dev）全仓唯一；backend 道另有 TASK-0072 在做打包与 prod 启动冒烟——你启动 8098 前先探测端口空闲，被占则等待重试，不许强杀他人进程；与 TASK-0072 无文件交集，无需等它。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0055-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修，禁止改期望凑输出。

## 收尾回写

- 不要回写 TASK 详情（由协调者按报告收尾）。
