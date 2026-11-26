# TASK-0055 实施报告（控制台框架、菜单种子与权限点注册 · 前端）

- 报告时间：2026-11-21
- 执行者：implementer（brief 唯一输入：TASK-0055-brief.md）
- BASE：`98c155e`（主仓库根 HEAD，开工时已核对一致）
- 结论：**5 条验收命令全部通过**（第 5 条按 brief 主选项完成：浏览器真实登录 + 截图 + DOM 断言证据，未走退化路径）

---

## 一、改动文件清单（全部为新增，除最后一项为 3 行追加）

| 文件 | 说明 |
| --- | --- |
| `frontend/src/api/iot/tenant-api.ts` | 租户 API 封装（对接 TASK-0001 已交付的 SysTenantController：/tenant list/{id}/POST/PUT/status/delete） |
| `frontend/src/views/iot/components/iot-placeholder.vue` | 占位页公共骨架（空态卡片 + 归属任务说明） |
| `frontend/src/views/iot/tenant/index.vue` | 租户列表页最小可用骨架：查询表单 + 列表 + 新建/编辑/启停/删除，按钮全部 `v-privilege` 绑定 `iot:tenant:*`（权限显隐演示页） |
| `frontend/src/views/iot/tenant/components/tenant-form-modal.vue` | 租户新增/编辑弹窗（编辑态编码禁改；二次确认在列表页） |
| `frontend/src/views/iot/dict/index.vue`（4008） | 字典管理占位骨架 |
| `frontend/src/views/iot/template/index.vue`（4009） | 模板管理占位骨架 |
| `frontend/src/views/iot/instance/index.vue`（4010） | 实例管理（模型）占位骨架 |
| `frontend/src/views/iot/point/index.vue`（4011） | 点位管理占位骨架 |
| `frontend/src/views/iot/gateway/index.vue`（4012） | 网关管理占位骨架 |
| `frontend/src/views/iot/channel/index.vue`（4013） | 通道管理占位骨架 |
| `frontend/src/views/iot/metric/index.vue`（4014） | 指标管理占位骨架 |
| `frontend/src/views/iot/publish/index.vue`（4015） | 配置发布占位骨架 |
| `frontend/src/views/iot/import/index.vue`（4016） | 导入向导占位骨架 |
| `frontend/src/router/iot.ts` | iot 静态路由段（当前为空，注释说明菜单驱动机制；照 survey.ts 预留样式） |
| `frontend/src/router/routers.ts` | **唯一修改的既有文件**：+3 行（import iotRouters 并展开进 routerArray） |
| `resources/sql/upgrade/20261121_iot_console_menus.sql` | 菜单种子：sys_menu 4008-4016（九个页面，menu_type=2，parent=4000，module_id=4）+ sys_role_menu 授 role_id=1 |

未触碰：backend 全部代码与 application*.yml、directives/store/layout 既有文件、workitems 库（TASK 详情未回写，按契约留给协调者）。

另注：`git status` 显示 `backend/ot-admin/src/main/resources/application-prod.yml` 有改动——**非本任务改动**，属并行的 backend 道 TASK-0072（文件集不相交），本任务未触碰。

---

## 二、验收命令原始输出

### 验收 1：前端构建 `cd frontend && pnpm build:test`

```
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. ...(既有提示，与本任务无关)
vite v5.2.12 building for test...
transforming...
✓ 5871 modules transformed.
rendering chunks...
warnings when minifying css:
▲ [WARNING] Unexpected "2of" [css-syntax-error]
    <stdin>:1:99250:
      1 │ ...ntents}.tox .tox-ai :nth-child(2of.tox-ai__response-status-icon)...
        ╵                                   ~~~
（上面这条为存量 tinymce .tox-ai 样式的压缩告警，与本任务无关）
...（chunk 列表，含本次新增 iot 产物）dist/js/iot-placeholder-CtfY8MRV.js、dist/js/tenant-form-modal-*.js
(!) Some chunks are larger than 500 kB after minification. ...(chunk 大小 warning，brief 明示可忽略)
✓ built in 1m 20s
```

- 首次构建与最终构建（见下）EXIT 均为 0，无 error。构建中途因 lint --fix 修了 2 处 prettier 换行格式，故重跑一次最终态构建：
```
$ pnpm build:test > /tmp/task0055-build-final.log 2>&1; echo "EXIT=$?"
EXIT=0
✓ built in 1m 5s
```
- 附加门禁（工程约定 CI 全量闸，非 brief 必做）：
  - `npx eslint src/api/iot/** src/views/iot/** src/router/iot.ts src/router/routers.ts --fix` 后复检：**LINT_CLEAN（0 error 0 warning）**
  - `pnpm typecheck`（vue-tsc --noEmit）：通过（无输出退出）

### 验收 2：菜单种子幂等（连跑两次 + 核对）

前置核对（执行前 4008+ 为空、4000-4007 在位）：
```
$ bash backend/scripts/sql-exec.sh "SELECT id, menu_name, menu_type, module_id FROM sys_menu WHERE id >= 4008;"
id	menu_name	menu_type	module_id
DONE
$ bash backend/scripts/sql-exec.sh "SELECT id, menu_name, menu_type, module_id FROM sys_menu WHERE id BETWEEN 4000 AND 4007;"
id	menu_name	menu_type	module_id
4000	智慧工厂	1	4
4001	租户管理	2	4
4002	租户查询	3	4
...（4003-4007 六按钮行，略）
DONE
```

连跑两次：
```
$ bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261121_iot_console_menus.sql
===== RUN 1 =====
OK(0)      ← SET NAMES
OK(9)      ← sys_menu 4008-4016
OK(9)      ← sys_role_menu
DONE
EXIT1=0
===== RUN 2 (idempotency) =====
OK(0)
OK(0)
OK(0)
DONE
EXIT2=0
```
两次均 DONE、均无 SQLERROR；第二次三语句全部 OK(0)（不重复插入、不改已有行）。

核对查询（与设计一致：9 行页面、menu_type=2、module_id=4、role_id=1 全授）：
```
$ bash backend/scripts/sql-exec.sh "SELECT id, menu_name, menu_type, module_id FROM sys_menu WHERE id>=4008;"
id	menu_name	menu_type	module_id
4008	字典管理	2	4
4009	模板管理	2	4
4010	实例管理	2	4
4011	点位管理	2	4
4012	网关管理	2	4
4013	通道管理	2	4
4014	指标管理	2	4
4015	配置发布	2	4
4016	导入向导	2	4
DONE
$ bash backend/scripts/sql-exec.sh "SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id>=4008;"
role_id	menu_id
1	4008
...（4009-4016，共 9 行，略）
DONE
```

### 验收 3：权限标识一一对应核对

```
$ grep -rn "SaCheckPermission" backend/ot-admin/src/main/java --include="*.java" | grep -o 'iot:[^"]*' | sort
iot:tenant:add
iot:tenant:edit
iot:tenant:list
iot:tenant:query
iot:tenant:remove
iot:tenant:status
（unique 计数 = 6）
```

SQL 侧（resources/sql 全目录，含 upgrade/）：
```
$ grep -rniE "iot:[a-z]+:[a-z]+" resources/sql --include="*.sql" -o | sort -u
resources/sql\upgrade\20261121_sys_tenant.sql:67:iot:tenant:list
resources/sql\upgrade\20261121_sys_tenant.sql:68:iot:tenant:query
resources/sql\upgrade\20261121_sys_tenant.sql:69:iot:tenant:add
resources/sql\upgrade\20261121_sys_tenant.sql:70:iot:tenant:edit
resources/sql\upgrade\20261121_sys_tenant.sql:71:iot:tenant:remove
resources/sql\upgrade\20261121_sys_tenant.sql:72:iot:tenant:status
```

- 两侧集合相等：**{iot:tenant:list, query, add, edit, remove, status}，各 6 个**（当前应恰好是 iot:tenant:* 六个 ✓）。
- 本任务新增的菜单种子 SQL 未引入任何三段式权限串（九行全部 perms=NULL、menu_type=2），未破坏 `SaPermissionRegistryConsistencyTest` 固化的「注解 ⊆ sys_menu SQL 登记」不变量。

### 验收 4：普通角色按钮不可见 + 直调 403

环境说明（端口纪律执行记录）：起 8098 前探测发现端口已被一个 java 进程占用（他人 dev 后端，探测 `/auth/captchaImage` 200、`/auth/login/test` 可用）；按「等待重试、不强杀」处理。该进程随后自行消失（ECONNREFUSED），端口转空后按 brief 命令自行拉起：`cd backend && mvn spring-boot:run -pl ot-admin`（约 20s 起来，HTTP 200）。全程未强杀任何进程。

建夹具（用 admin 在接口上建：角色/用户走后端接口；用户密码与前端 `src/lib/encrypt.ts` 同构 SM4 加密后提交，非明文直插）：
```
--- 1.admin login/test ---           {"code":200,"username":"admin"}
--- 2.POST /role (iot 查看角色) ---   {"code":200,"message":"操作成功","httpStatus":200}
--- 3.role created ---               {"id":3,"roleName":"iot 查看角色","roleCode":"iot_viewer_t0055"}（menuIds 空，不授任何权限）
--- 4.POST /user ---                 {"code":200,"message":"操作成功","httpStatus":200}
--- 5.user created ---               {"id":3,"username":"iot_view_t0055","status":1}
--- 6.PUT /user/{id}/role ---        {"code":200,"message":"操作成功"}   ← 测试用户挂上该角色
```

测试用户登录与直调（补授页面菜单前——角色无任何菜单/权限）：
```
--- 7.test user login/test ---       {"code":200,"username":"iot_view_t0055"}
--- 8.test user /user/info ---       {"menuTotal":0,"allMenuNames":[],"iotMenus":[],"permissionCount":0,"iotPermissions":[]}
--- 9.test user GET /tenant/list --- {"httpStatus":200,"body":{"code":403,"data":null,"message":"无权访问该资源","timestamp":1795237548084}}
--- 10.admin GET /tenant/list ---    {"httpStatus":200,"code":200,"total":1,"records":[{"id":1,"tenantCode":"default","tenantName":"默认租户"}]}
```

- 测试用户菜单列表接口响应：**无 iot 菜单（0 条菜单）、无任何权限点** ✓（brief 允许「无 iot 菜单或仅有目录」两者）
- 测试用户 token 直调 `GET /tenant/list` → 业务码 **403 无权访问该资源** ✓（框架全局包装：HTTP 200 + body code=403，与前端 axios 拦截器「code 403 → 弹无权限」契约一致）
- admin token 直调同接口 → **code 200，total=1** ✓

补充（为验收 5 的页面可达性，给角色 3 补授 4000+4001 目录+页面菜单、仍无任何权限点后复测）：
```
updateRoleMenu: {"code":200,"message":"操作成功"}  selectedMenuId:[4000,4001]
test user menus: [{"id":4000,"name":"智慧工厂","type":1,"children":["4001:租户管理"]}]
test user permissions: []
test user GET /tenant/list: {"httpStatus":200,"code":403,"message":"无权访问该资源"}   ← 403 依然成立
```

### 验收 5：前端显隐演示（浏览器真实登录，截图 + DOM 断言）

dev 前端：`pnpm dev`（8099，探测空闲后启动，`vite ready in 3391 ms`）。两位用户均**走真实登录页**（账号/密码/验证码输入后点登录；验证码为图片字符码，由截图人工识读）。

**（a）测试用户 iot_view_t0055（角色=「iot 查看角色」，仅授 4000+4001 菜单、0 个权限点）**

登录成功 → 右侧模块天然只有「智慧工厂」→ 展开菜单只有「租户管理」→ 进入 `#/iot/tenant`：

- 截图证据（浏览器工件，登录页/租户页全图）：
  - `C:\Users\dowil\.zcode\cli\artifacts\...\call_5afb9c21e0f04ec9a6f20f3c-tool-result-9e83644b-...png`（测试用户登录页）
  - `C:\Users\dowil\.zcode\cli\artifacts\...\call_ae1382b4b1fb4cbf9fb0b2e4-tool-result-e6f3df8f-...png`（**测试用户的租户页**：查询表单 + 空表「暂无数据」+ 右下角「无权访问该资源」toast + 工具栏无「新建」按钮；侧边栏仅 智慧工厂→租户管理）
- DOM 断言（测试用户租户页，main 区域）：
```
{"pageButtons":["查询","重置"],
 "hasXinjian":false,   ← 新建（iot:tenant:add）
 "hasBianji":false,    ← 编辑（iot:tenant:edit）
 "hasTingyong":false,  ← 停用（iot:tenant:status）
 "hasQiyong":false,    ← 启用（iot:tenant:status）
 "hasShanchu":false}   ← 删除（iot:tenant:remove）
```
  高危操作按钮全部不可见 ✓（privilege 指令对无权限元素整节点移除，DOM 中不存在）；列表接口 403 → 表格空态 + 全局「无权访问该资源」提示 ✓

**（b）管理员 admin（userType=3 超管）**

退出登录 → 真实登录页登录 admin → 头部「切换模块」→ 选「智慧工厂」→ 落到 `#/iot/tenant`：

- 截图证据：
  - `C:\Users\dowil\.zcode\cli\artifacts\...\call_a7243a5b41b6401594b16564-tool-result-dd3c54cf-...png`（**管理员租户页**：工具栏「+ 新建」按钮可见；列表 1 行 default 租户，行内「编辑 停用 删除」可见；侧边栏 10 个页面菜单齐全）
- DOM 断言（管理员租户页）：
```
{"hasXinjian":true,
 "rowActionTexts":["编辑","停用","删除"],
 "sidebarItems":["智慧工厂","租户管理","字典管理","模板管理","实例管理","点位管理",
                 "网关管理","通道管理","指标管理","配置发布","导入向导"]}
```
  管理员按钮可见 ✓、九个新菜单全部出现在侧边栏 ✓（4008-4016 与 views/iot 组件一一对应，无 404/组件缺失告警）

**（c）占位页抽样（字典管理 4008）**

- 截图：`C:\Users\dowil\.zcode\cli\artifacts\...\call_db8265c494f144448a8528bd-tool-result-ac0eda33-...png`（「字典管理页面骨架（空态）——完整功能由 TASK-0056 / TASK-0070 交付」空态卡片）
- 其余 8 个占位页为同一组件不同 props，不逐一截图。

---

## 三、未做项与原因

| 项 | 原因 |
| --- | --- |
| `src/api/iot/` 其余页面的 API 封装（字典/模板/实例/点位/网关/通道/指标/发布/导入） | 后端目前仅租户域有真实接口（brief 决策 3 明示）。预写空壳封装必然是猜接口；按 brief「照 src/api/system 现有封装样式」的要求，随各页面对应后端任务交付时补建 |
| menu_type=3 按钮权限行（4008 起各页） | 非「未做」而是 brief 决策 2 的裁决要求：原则「权限标识与接口清单一一对应」，其余页面按钮权限行由各自后端任务交付接口时登记（守卫测试强制）；当前 SQL 侧与注解侧集合已相等（验收 3） |
| 完整租户页（配额/到期提醒、编码唯一性前置提示、编码字符集校验） | brief 明示归 TASK-0056；本任务仅交付最小可用骨架 |

## 四、偏离计划之处

1. **菜单号段（brief 决策 1 的既定纠偏，留痕）**：任务正文与 FEAT-0018 规划「3000-3999 段」为过时文本；按 brief 裁决实取 **4000 段 4008-4016**（3000-3999 不启用）。未回改 FEAT/TASK 正文，以此处与本报告留痕。
2. **router 静态路由段为空**：brief 允许 `frontend/src/router/**（仅追加 iot 路由段）`。实测本工程 iot 页面与 system 各页（字典/角色等）完全同构——全部由 sys_menu 下发、经 `buildRoutes()` 的 `import.meta.glob` 懒加载动态注册，静态注册反而绕开菜单权限模型。故 `router/iot.ts` 以空数组落地（照 survey.ts 预留样式、注释说明机制）并接线进 `routers.ts`（+3 行）；浏览器验收证明 10 个页面经菜单动态注册全部可达、无组件缺失告警。若协调者要求真实的静态路由内容，请裁决。
3. **验收 4 的「起 dev 后端」**：探测时 8098 已有他人 dev 后端（未强杀，先复用其做了 admin 冒烟），该进程中途自行退出后按 brief 原命令自行拉起 dev 后端并完成全部验证。
4. **测试角色从「无任何菜单」补授到「4000+4001 目录+页面」**：验收 4 先按 brief 原样（不授任何 iot 菜单/权限）取证（菜单 0 条 + 403）；验收 5 需要进入页面演示按钮显隐，故给角色补授了目录+页面菜单（仍 0 个权限点，403 依旧成立，补授后已复测留证）。brief 验收 4 的口径「无 iot 菜单或仅有目录」在两个阶段分别满足。
5. **验收 5 证据形态细节**：验证码为图片字符码（如 AP7X/S9HQ/BXLM），由浏览器截图识读后键入，第一次提交因验证码过期/误读报「验证码错误」，刷新后成功——真实登录路径未被绕过（未使用 token 注入）。另：本环境 Playwright 高层 click 对该页面多次超时，改用页面内 evaluate 对快照定位到的真实元素执行 DOM click（事件冒泡至 Vue @click，与真人点击同路径）；按钮可见性判定以 DOM 断言 + 截图双重证据为准。

## 五、遗留问题

1. **dev 库测试夹具**：sys_role id=3（iot 查看角色 / iot_viewer_t0055，菜单 4000+4001）、sys_user id=3（iot_view_t0055，密码 Iot@view2026）及关联行留在 dev 库中作为权限显隐演示夹具；是否清理由协调者裁决（清理两行 sys_user_role / sys_role_menu 关联即可，注意租户表未动）。
2. **运行中的服务**：8098 dev 后端（本任务拉起，`mvn spring-boot:run -pl ot-admin`，日志 `/tmp/task0055-backend.log`）与 8099 前端 dev server（日志 `/tmp/task0055-frontend-dev.log`）仍在后台运行，供复验与后续任务使用；若 TASK-0072 需要独占 8098，请先停掉本任务这份（本任务进程可安全终止，不存在他人进程）。
3. **并行任务的越界提示**：工作区中 `backend/ot-admin/src/main/resources/application-prod.yml` 出现未提交改动，属并行 backend 道 TASK-0072 的文件集，本任务未触碰也未纳入本报告清单。
4. **租户骨架的已知简化**（均归 TASK-0056 收口）：tenantCode 无唯一性/字符集前置提示（后端 4002 业务码兜底）；列表无批量删除；表单无乐观锁冲突提示。
5. **占位页空态文案**：9 个占位页共用 iot-placeholder 组件，描述统一为「完整功能由 TASK-0056 / TASK-0070 交付」；若后续任务编号调整需同步改文案（单文件改动）。
