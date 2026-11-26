# TASK-0002 diff 评审报告（fresh reviewer）

- 评审对象：`TASK-0002-diff.txt`（BASE 04fc3a5 → 3f3b3c7，33 文件）
- 评审依据：brief「依据」节 + `L1-execute.md` §13（与 TASK 验收标准 / brief 接口契约的一致性 + 明显质量缺陷）
- 评审输入：diff 包（完整读入）、实施 brief、TASK 详情、实施报告（含根因链 R1-R6 与第二执行者复核附录，仅作交叉印证）；超出 diff 的仓库交叉核对仅限读（SaTokenInterceptorConfig / ActuatorAccessPolicy / SaTokenConfig / 01_base.sql / 各 ServiceImpl 周边代码），未修改任何文件。

---

## 一、brief 重点核验项结论（先核验，发现清单见第二节）

| 核验项 | 结论 | 依据 |
| --- | --- | --- |
| 白名单式 ignoreTable 是否真正 fail-closed | **通过** | `OtTenantLineHandler`：清单（六表）+ `iot_` 前缀之外一律 ignore，方向为白名单；`normalize` 覆盖反引号/双引号剥离 + 小写归一（单测覆盖 `SYS_USER`/`` `sys_user` ``）；无会话经 ignoreTable 短路（R5 的 MP 3.5.17 `tenant_id = NULL` 陷阱已规避）；getTenantId 若意外可达返回 null → 生成恒假条件，方向仍是 fail-closed |
| 旁路是否仅限三处 Mapper 方法且显式条件完整 | **通过** | 全仓 grep `@InterceptorIgnore` 仅 3 处方法级：`SysUserMapper.selectByTenantAndUsername` / `countByTenantAndUsername`、`SysRoleMapper.selectByTenantAndCode`，三条 SQL 均自带 `tenant_id = #{}` + 业务键 + `deleted = 0`，全部 `#{}` 参数化，无注入面 |
| tenant_id 服务层覆盖是否覆盖全部写入口 | **通过** | 全仓 grep 关联行插入点共 6 处（init-admin 1 处走路径租户、其余 5 处走会话）；user（insert/update/import/roles）、dept（insert/update）、role（insert/update + 三类关联行）、dict/config 写侧置空——均覆盖；UPDATE/DELETE 由插件追加 WHERE 租户条件，跨租户写匹配不到行 |
| 登录租户解析顺序与禁用拒登 | **通过** | `resolveLoginTenant`：显式 tenantCode（trim）→ 配置 `ot.tenant.default-code`；`selectEnabledByCode` 仅放行 status=1，null 一律 400「租户不存在或已禁用」（不区分存在/禁用，防枚举）；用户查询显式 (tenant_id, username)；login 与 login/test 两口同口径；会话写 `tenant_id`/`tenant_code` |
| init-admin 路径租户例外与 4002 | **通过** | tenant_id 仅取路径变量（表单无 tenantId 字段）；目标租户存在且 status=1；(tenant_id,username) 双道闸（旁路预检 4002 + `uk_tenant_username` 兜底 4002）；关联行/角色 tenant_id=路径租户；重复报 `ResultCode.DUPLICATE_CODE(4002)` 有单测断言 + 冒烟 S3c 实证 |
| 升级脚本幂等与存量回填 | **通过** | `information_schema` 判存的条件化存储过程（列/索引），`INSERT IGNORE` 依赖 01_base 实测在位的 PK（`sys_role_menu(role_id,menu_id)`、`sys_menu(id)`）；`uk_username`→`uk_tenant_username` 双向判存；六表 NOT NULL DEFAULT 1 即存量回填；验收 2 两次 DONE 无 SQLERROR + SHOW CREATE TABLE 佐证 |
| 注入面（tenant_code 直拼 SQL？） | **通过** | tenantCode 仅进 `LambdaQueryWrapper.eq`（参数化）；三条新原生 SQL 全 `#{}`；无 `${}`、无字符串拼接 |
| 会话伪造面 | **通过** | tenant_id/tenant_code 只由服务端在登录成功后写入 Sa-Token 会话，无 header/参数读取路径；`TenantContext` 为唯一读口 |
| 事务边界 | **通过** | initTenantAdmin 单事务（rollbackFor=Exception），角色/授权行/用户/绑定同生共死；insertTenantUser 的 DuplicateKey→BusinessException 在代理边界内抛出，整事务回滚，无半提交残留 |
| JSqlParser 覆盖盲区 | **通过（留遗留）** | 全仓无 XML mapper、无 JdbcTemplate；原生 @Select/@Update 全部命中 survey_/wechat_ 非白名单表（插件不触）；报告遗留 4 已如实登记复杂 SQL 限制 |
| 并发 init-admin | **通过（有边界缺陷，见 Minor-5）** | 用户侧并发由 uk_tenant_username 兜底 → 4002；角色侧 DuplicateKey 恢复 + 复用，存在 REPEATABLE READ 快照边界（Minor-5） |
| MP INSERT 列已存在时插件行为 | **通过（实证）** | 冒烟 S3/S3b：会话租户 1 下建出的用户/角色/关联行 tenant_id=5/6（路径租户），证明 3.5.17 对已含 tenant_id 列的 INSERT 跳过注入，实体显式值生效 |
| 越界反查 | **通过** | 33 文件均在 brief 声明范围（iot 模块的 service/dto 属裁决 4 端点契约的必要实现）；application.yml 仅授权的一个键 |

---

## 二、发现清单

### Important（1 条）

**I-1 租户管理员角色自动供给时越权超授：拿到平台级监控权限，违背本端点自述契约「只见本租户数据」**

- 出处：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/SysTenantServiceImpl.java` — `ensureTenantAdminRole` / `isOperatorOnlyMenu`（授予集=全量菜单仅排除 4001 与 `iot:tenant:*` 前缀）
- 说明：01_base 的 `sys_menu` 含「系统监控」子树（110-150：`monitor:online:list` / `monitor:online:forceLogout` / `monitor:cache:list/remove` / `monitor:job:add/edit/remove/run/changeStatus` / `monitor:actuator:view` 等），这些行既非 4001 也不带 `iot:tenant:` 前缀，全部落入租户管理员角色的 146 条授权；`SaTokenConfig.getPermissionList` 按 角色→role_menu→menu.perms 直通生效。后果：①`/monitor/online/list` 走 Sa-Token 会话清单（无 DB 租户维），租户管理员可枚举**全平台**在线用户（含他租户用户名/id）——直接违背 REQ-0001 验收 1「只见本租户数据」；②`/monitor/online/{userId}` 可强制下线**任意**用户（含运营方 admin，Redis 按 loginId 无租户校验）；③`/monitor/cache/remove` 可删任意 Redis 键（验证码、登录失败计数、他人会话）；④`monitor:job:*` 可新增/立即执行定时任务（invokeTarget 有 bean 白名单+方法黑名单兜底，反射面已被收窄，故不到 Critical）；⑤`monitor:actuator:view` 放开 actuator 分级管控闸门（当前仅 health 对外暴露，属潜在面）。实施报告 R2 只验证了 `iot:tenant:*` 授权为 0，未发现 monitor 侧超授。建议修复方向：`ensureTenantAdminRole` 排除「系统监控」子树（或改为显式菜单白名单），并对存量 `tenant_admin_*` 角色做一次授权订正；修复后补断言（monitor:* = 0 行）。

### Minor（8 条，记 deferred，不进修订轮）

**M-1** TenantContext 异常全吞 → 基础设施故障时 fail-open。出处：`backend/ot-core/.../tenant/TenantContext.java`（getTenantId/getTenantCode 的 `catch (Exception e) { return null; }`）。说明：Redis 抖动等异常被归约为「无租户上下文」，白名单表查询静默不过滤而非失败；当前因鉴权拦截器同样依赖 Redis 而无实际可达路径，但方向应改为「区分未登录/非Web 与基础设施异常，后者不静默放行」。

**M-2** 禁用租户不切断存量会话。出处：`SysTenantServiceImpl.updateTenantStatus`（仅改 status；对照用户停用已有 kickOut 即时生效链路）。说明：被禁租户的在线用户最长 24h 内可继续以原会话正常操作，禁用仅在再登录时生效；brief 只约定「禁用后拒登」，契约未破，但禁用作为运营方管控手段偏弱，建议后续任务补踢会话或 AccountStatusGuard 式逐请求租户状态复核。

**M-3** 存量旧会话窗口与无上下文写落的静默 DEFAULT 1。出处：`TenantContext`（旧会话无 tenant_id → 全表忽略，报告遗留 1 已列）+ 插件对无上下文 INSERT 跳过注入 → 白名单表落 `DEFAULT 1`。说明：部署后旧 token 最长 24h 不过滤（受影响者仅存量平台用户，爆炸半径有限，报告已给清会话/强制重登建议）；非 Web 线程写白名单表会静默归入租户 1——均属既定设计代价，提醒随遗留 1 一并收口。

**M-4** 并发 init-admin 撞 `uk_role_code` 的恢复路径受事务快照限制。出处：`SysTenantServiceImpl.ensureTenantAdminRole` 的 DuplicateKeyException 恢复分支。说明：REPEATABLE READ 下竞争方提交晚于本事务快照时，恢复重查仍为 null → 原始 DuplicateKeyException 以 500 透出（数据一致、重试即成功）；并发同租户双 init-admin 属小概率路径，属健壮性而非正确性缺陷。

**M-5** 运营方菜单排除依赖数据约定而非子树关系。出处：`SysTenantServiceImpl.isOperatorOnlyMenu`（仅 id=4001 + perms 前缀 `iot:tenant:`）。说明：未来挂在 4001 下的新按钮若未遵守 `iot:tenant:` 命名约定，会被静默授予租户管理员；建议补 parent 子树排除或菜单数据断言（与 I-1 的白名单化修复可一并处理）。

**M-6** 死代码旁路口保留。出处：`SysUserService`/`SysUserServiceImpl.selectUserByUsername`（全仓零调用）。说明：仅按 username 的查询在无会话上下文下即跨租户全表语义，留着是一个未来误用的旁路脚枪，建议删除或改名标注仅供内部无租户场景。

**M-7** `addUsersToRole` 未校验目标用户的租户归属。出处：`SysRoleServiceImpl.addUsersToRole`（只验角色存在，未 getById 用户）。说明：传入他租户 userId 可写入 (他租户用户, 本租户角色, 本租户 tenant_id) 的幽灵关联行——因权限按会话租户过滤，受害方看不到该行、无提权，但产生跨租户脏数据；建议插入前校验用户属于会话租户（`updateUserRoles` 已有此校验，两口径应对齐）。

**M-8** 角色编码预检域与物理唯一域不一致。出处：`SysRoleServiceImpl.insertRole/updateRole`（预检被插件收敛为租户域，`uk_role_code` 仍全局唯一）。说明：他租户已占用的编码会通过预检、靠 DuplicateKeyException 兜底转友好报错——最终用户体验正确，但「预检说可用、落库报已存在」的语义错位随多租户化新引入（遗留 3 已登记全局唯一决策，可维持现状留痕）。

---

## 三、汇总

- **Critical 0 / Important 1 / Minor 8**
- Important 1 条（I-1 租户管理员角色超授平台级监控权限）进 §13 退回路径；Minor 8 条记 TASK `任务结果` deferred 收尾进 LOG。
- brief 重点核验项（fail-closed 白名单、三处旁路显式条件、全写入口覆盖、登录租户解析与禁用拒登、init-admin 4002、脚本幂等与回填）全部通过；无注入面、无会话伪造面、事务边界正确。
