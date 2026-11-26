# 实施 brief：TASK-0006（可见集合 Redis 缓存与主动失效）

## 批次

- TASK-0006（本批共 1 个，标准任务 2 人天；backend 串行道——数据权限链第二环 0005→0006→0007）
- BASE：`ef7a84d`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/**`（缓存实现与失效监听——0005 交付的 VisibleScopeCache 接缝）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/system/**`（仅失效事件发布点：role/dept/user-role 变更处最小侵入）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImpl.java`（绑部门变更点失效，仅一处）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/**`（新增/调整测试）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增 `ot.dataperm.cache-*` 键族）
  - 不碰 frontend/、ingest/、dataperm 组件的翻译逻辑本体（缓存挂接缝不重算）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0006.md`
- 关联对象：同目录 `features\FEAT-0002.md`（业务规则：TTL 10 分钟/主动失效）、`requirements\REQ-0002.md`（验收「改绑定/角色后权限即时生效」）、`testcases\TC-0002.md`
- 接缝（消费对象）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/**` 的 VisibleScopeCache 接口与 NoOp 实现（0005 交付）+ resolve 调用路径（先缓存后计算已按接缝写好）
- 失效来源盘点：`system/service/impl/SysRoleServiceImpl`（角色增改/授权变更）、`SysDeptServiceImpl`（部门变更）、`SysUserServiceImpl`（用户角色绑定 updateUserRoles）与 `iot/instance` 绑部门（0012 updateInstance deptId）——grep 自行核实全部变更点
- 基础设施：`ot-core/module/redis/**`（框架 Redis 服务范式）

## 接口与决策（逐条结论，不再讨论）

1. **Redis 实现**：`RedisVisibleScopeCache implements VisibleScopeCache`——key=`iot:dataperm:scope:{userId}:{roleVer}`，value=VisibleScope JSON 序列化（Jackson），TTL 600s（`ot.dataperm.cache-ttl-seconds` 可配，默认 600=10 分钟）。**「角色版本」载体裁决**：sys_role 表无版本列——用**失效即失效**模型替代版本号：key 不含版本，改为失效时按 userId 前缀删除（`iot:dataperm:scope:{userId}:*`）；roleVer 段保留为占位 0（未来引入版本列时的演进位，本期恒 0）。理由：改角色授权影响的是「授权过该角色的所有用户」，逐用户失效需要反查用户集（成本高）；**裁决失效粒度=按变更类型**：①用户角色绑定变更→失效该 userId；②角色本身/授权变更→失效**全量**（`iot:dataperm:scope:*`——低频操作，全量失效简单正确）；③部门变更→全量；④实例绑部门变更→全量（实例可见集含 dept 推导）。全部用 Redis SCAN+DEL（非 KEYS——生产安全）。
2. **失效机制**：Spring 应用事件（`DataPermCacheEvictEvent`，payload=userId 可空→null 即全量）+ `@EventListener`（transact 语义：变更点在事务提交后发布——TransactionSynchronization.afterCommit，照 0003 挂点范式）→ 监听器执行 SCAN+DEL。**单机一期口径**：不引 Redis pub/sub（单实例内事件即可；多实例部署的跨节点失效登记 TD/遗留，随集群形态任务裁决）。
3. **resolve 接线**：DataPermService.resolve 改为——cache.get(userId) 命中→返回；未命中→计算→cache.put。**NoOp 实现保留**（`ot.dataperm.cache-enabled` 开关，默认 true；关闭时走 NoOp——测试与故障兜底）。
4. **失效点接入（最小侵入）**：SysRoleServiceImpl（insert/update/updateRoleMenu 等写方法 afterCommit 发全量失效事件）、SysDeptServiceImpl（insert/update/delete 同）、SysUserServiceImpl（updateUserRoles 发单用户失效）、IotModelInstanceServiceImpl.updateInstance（deptId 变更分支发全量）。每处一行事件发布（复用 afterCommit 助手），测试断言事件发布与监听器行为。
5. **测试口径（TDD）**：缓存命中/未命中/TTL 过期（可用短 TTL 配置模拟）/失效单用户/失效全量（SCAN 语义）/角色变更→全量失效/绑定变更→单用户失效/实例绑部门→全量/cache-enabled=false 走 NoOp/resolve 未命中计算后回填。**失效即时性验收**（TC-0002 口径）：单测级模拟即可（mock cache 直查），端到端归 0007 红线套件。

## 验收命令（整体验收）

环境事实同前（WSL docker Redis 容器 wisfactory-redis；**测试用 db1+ 隔离 backend 的 db0**——brief 早期约定；8098 端口纪律）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1483 + 新增，Results 段贴回）
2. dev 冒烟（每步原始输出贴回）：admin 登录建两角色账号（A=本部门及以下、B=自定义窄范围）→ A 登录触发可见集计算 → Redis 查 key 存在（`wsl docker exec wisfactory-redis redis-cli --scan --pattern 'iot:dataperm:scope:*'`）→ admin 改 A 的角色授权 → A 的缓存 key 消失（即时失效）→ A 再登录重建 → B 改绑部门 → 全量失效核对 → cache-enabled=false 回归 NoOp
3. prod 档真实启动回归 → 0 ERROR + tree 200 + Redis key 出现
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0006）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
