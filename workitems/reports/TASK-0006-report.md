# TASK-0006 实施报告：可见集合 Redis 缓存与主动失效

- 执行者：implementer（brief：reports/TASK-0006-brief.md）
- BASE：`ef7a84d`（主仓 HEAD）→ HEAD：`db16079`（main，1 个提交，注明 TASK-0006）
- 总体结论：**4 条验收全部通过**（全量回归 1533 = 基线 1483 + 新增 50，0 失败 0 错误；dev 冒烟全步通过——key 出现/TTL≈600s/角色授权变更即时失效/A 再登录重建/实例绑部门全量失效/绑定变更单用户失效/cache-enabled=false 走 NoOp；prod 档真实启动启动段 0 ERROR + 真实登录 200 + tree 200 + Redis key 出现；改动范围 22 文件全落声明内）
- 修订轮（2026-11-22）：评审 Critical 0 / Important 2 / Minor 6——I-1 接受为残余风险（TD-0055，不改代码）；**I-2 必修已修**（create/delete 补失效挂点，红 2 → 绿，全量复跑 1537 绿，dev 冒烟建/删实例失效复核全过，见文末「修订轮」一节）；Minor 6 条记 deferred

---

## 1. 改动文件清单（22 个，与验收 4 的 git diff 一致）

**新增 main（4 个，全在 dataperm 包——0005 预留接缝的实现）**

| 文件 | 内容 |
| --- | --- |
| `backend/.../iot/dataperm/RedisVisibleScopeCache.java` | 缓存 Redis 实现：key=`iot:dataperm:scope:{userId}:0`（roleVer 占位 0，失效即失效模型）、value=VisibleScope JSON（Jackson 2，宽容未知字段）、TTL=`ot.dataperm.cache-ttl-seconds`（默认 600s，≤0 误配回落 600 并 WARN）；get 损坏/故障降级为未命中，put/evict 故障静默（接口契约：不阻断业务）；evict=SCAN 游标分批 + DEL 分批（SCAN_COUNT_HINT=200，DELETE_BATCH_SIZE=1000，**不用 KEYS**）；键模板 `rawStringRedisTemplate`（组件自拼完整键名，无 key-prefix——验收 redis-cli 命令直查物理键，跨系统隔离靠 database，ADR-1009） |
| `backend/.../iot/dataperm/DataPermCacheEvictEvent.java` | 失效事件 record，payload=userId 可空（null 即全量），工厂 ofUser/all |
| `backend/.../iot/dataperm/DataPermCacheEvictPublisher.java` | 失效点唯一入口（evictUser/evictAll 各一行调用）：有事务注册 afterCommit 发布（照 0003 挂点范式），无事务立即发布（与 0003 的差异及理由见 §4.2）；发布动作 try/catch 收敛，绝不炸穿已提交写事务 |
| `backend/.../iot/dataperm/DataPermCacheEvictListener.java` | `@EventListener` → `cache.evict(event.userId())` 执行 SCAN+DEL；再兜一层 try/catch；单机一期 Spring 应用事件（进程内即达），不引 pub/sub |

**修改 main（7 个）**

| 文件 | 内容 |
| --- | --- |
| `backend/.../iot/dataperm/NoOpVisibleScopeCache.java` | 加 `@ConditionalOnProperty(name="ot.dataperm.cache-enabled", havingValue="false")`——与 Redis 实现（matchIfMissing=true）互斥注册单 Bean，关闭开关走 NoOp 直查（0005 兜底保留） |
| `backend/.../iot/dataperm/DataPermProperties.java` | 新增 `cacheTtlSeconds`（默认 600） |
| `backend/.../system/service/impl/SysRoleServiceImpl.java` | 9 处一行发布：insertRole/updateRole/deleteRoleById/updateRoleMenu/updateRoleDept/updateDataScope → 全量；addUsersToRole/removeUserFromRole/removeUsersFromRole → 逐用户（裁决 1 失效粒度①②；brief 只点名 insert/update/updateRoleMenu，「等写方法」按 grep 核实全挂） |
| `backend/.../system/service/impl/SysDeptServiceImpl.java` | 3 处：insertDept/updateDept/deleteDeptById → 全量（裁决 3，仅在写成功后发布） |
| `backend/.../system/service/impl/SysUserServiceImpl.java` | 2 处：updateUserRoles → evictUser(userId)（brief 指定）；updateUser 携带 deptId → evictUser(userId)（grep 核实补充变更点，见 §4.1） |
| `backend/.../iot/instance/service/impl/IotModelInstanceServiceImpl.java` | 1 处（brief「仅一处」）：updateInstance 中 `updated && form.getDeptId() != null && !form.getDeptId().equals(existing.getDeptId())` → 全量 |
| `backend/ot-admin/src/main/resources/application.yml` | 仅新增 `ot.dataperm.cache-enabled`（默认 true）与 `ot.dataperm.cache-ttl-seconds`（默认 600）两键，带注释 |

**测试（11 个：新增 6 + 调整 5，全在声明内）**

| 文件 | 内容 |
| --- | --- |
| `RedisVisibleScopeCacheTest.java`（新，12 用例） | 命中/未命中/损坏 JSON 降级/Redis 故障降级；put 默认 TTL 与短 TTL 配置传递（brief「可用短 TTL 配置模拟」）、误配 ≤0 回落；evict 单用户 pattern=`iot:dataperm:scope:7:*` 与全量 pattern=`iot:dataperm:scope:*`（SCAN 语义断言）+DEL、无命中不发 DEL、故障静默 |
| `DataPermCacheEvictPublisherTest.java`（新，6 用例） | 无事务立即发布 / 有事务提交前不发布 / afterCommit 发布 / afterCommit 与无事务路径发布异常均收敛 |
| `DataPermCacheEvictListenerTest.java`（新，3 用例） | 事件 payload 直通 cache.evict（null=全量）；缓存实现违约抛出被兜住 |
| `DataPermCacheWiringTest.java`（新，3 用例，ApplicationContextRunner） | 默认/true → 仅 Redis 实现；false → 仅 NoOp（互斥单 Bean） |
| `SysRoleServiceImplTest.java`（新，13 用例） | 角色 7 个全量失效点 + 3 个逐用户失效点 + 写失败/空入参不发布 |
| `SysUserDataPermEvictTest.java`（新，5 用例） | updateUserRoles 单用户失效（含无增删幂等场景）；updateUser 带/不带 deptId |
| `DataPermResolveScopeTest.java`（调，+1） | 未命中计算后回填：首次 resolve 计算+put、二次 resolve 命中不再触库 |
| `SysDeptServiceImplTest.java`（调，+4） | insert/update/delete 全量失效 + 删除被拒不发布 |
| `IotModelInstanceServiceImplTest.java`（调，+3） | 改绑部门全量失效 / 未改部门不发布 / 同值 no-op 不发布 |
| `SysDeptRoleTenantInjectionTest.java`、`SysUserTenantInjectionTest.java`（调，0 净增） | 既有测试因服务新增 publisher 字段补充 mock 注入（非行为改动） |

## 2. 验收命令原始输出

### 2.1 验收 1：全量测试

```
mvn install -pl ot-core -DskipTests
[INFO] BUILD SUCCESS（Total time: 3.686 s）

mvn test -pl ot-admin
[INFO] Tests run: 1533, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

1533 = 基线 1483 + 新增 50（12+6+3+3+13+13+5+4+3，调整类净增 0）。Results 段全绿，0 失败 0 错误。

### 2.2 验收 2：dev 冒烟（8098 纪律：启动前探测无 LISTENING；`Started OtAdminApplication in 17.69 seconds`；启动段 ERROR=0）

```
=== S1) admin 登录（dev /auth/login/test） ===
{"code":200,"data":{"nickname":"管理员","token":"163f27e1-6bda-495f-b09d-2ff3031f3fca","userId":1,"username":"admin"},"message":"操作成功"}

=== S2) admin 建两角色（POST /role，A=dataScope 2 本部门及以下菜单 4030/4031/4035；B=dataScope 5 自定义窄范围 deptIds=[2]） ===
{"code":200,...} ×2 → DB：角色 6 smoke_perm_a_t0006(2)、7 smoke_perm_b_t0006(5)

=== S3) admin 建两账号（POST /user，密码 SM4 密文提交，jshell ROUNDTRIP-OK=Perm@2006a）+ PUT /user/{id}/role 绑定 ===
用户 6 smoke_perm_a_t0006(dept=4→角色6)、7 smoke_perm_b_t0006(dept=1→角色7)，各 {"code":200}

=== S4) A 登录 ===
{"code":200,"data":{"nickname":"smokePermA-T0006","token":"069c2c78-...","userId":6,...}}

=== S5) A 调 GET /iot/instance/tree 触发可见集计算 → HTTP=200 → Redis 查 key 存在 ===
$ wsl docker exec wisfactory-redis redis-cli --scan --pattern 'iot:dataperm:scope:*'
iot:dataperm:scope:6:0

=== S5c) key 内容与 TTL ===
GET iot:dataperm:scope:6:0
{"allTenant":false,"deptIds":[4],"instanceIds":[12,13,14,3,6,8,11],"createByOnly":false}
TTL iot:dataperm:scope:6:0
592
（正确性注记：A=部门 4 本部门及以下 → 部门锚点实例 3/8 + 子树展开 6/11/12/13/14；9 已逻辑删故不在——与库内数据逐项核对一致）

=== S6) admin 改 A 的角色授权（POST /role/menu/updateRoleMenu → updateRoleMenu 挂点） → A 的 key 即时消失 ===
{"code":200,...}
SCAN结果=[]

=== S7) A 再登录并调 tree → key 重建 ===
tree HTTP=200
iot:dataperm:scope:6:0

=== S8) B 登录并调 tree（第二 key，为全量核对准备） ===
tree HTTP=200；B 的可见集 {"allTenant":false,"deptIds":[2],"instanceIds":[15,16,4,5],"createByOnly":false}（自定义窄范围=部门 2 及其子树实例）
SCAN：iot:dataperm:scope:6:0 / iot:dataperm:scope:7:0

=== S9) admin PUT /iot/instance 实例 F02(id=5) 改绑部门 2→4（brief 裁决 4 唯一实例侧挂点） → 全量失效核对 ===
{"code":200,...}
SCAN结果=[]
日志：[DATAPERM] visible-scope cache evicted pattern=iot:dataperm:scope:* keys=3
      [DATAPERM] visible-scope cache evicted ALL
（keys=3={1,6,7}：admin 自身在 requireDataPerm 中 resolve 也已回填，一并被全量失效——符合裁决②④）

=== S10) 重建双 key（A/B 各登录 + tree） → SCAN：6:0 / 7:0 ===

=== S11) admin 改用户A角色绑定（PUT /user/6/role → updateUserRoles 挂点，裁决 1 失效粒度①） ===
{"code":200,...}
SCAN结果=[iot:dataperm:scope:7:0]
（仅 userId=6 的 key 消失，userId=7 保留——单用户失效即失效模型验证）

=== S13) cache-enabled=false 回归 NoOp（OT_DATAPERM_CACHE_ENABLED=false 重启 dev） ===
Started OtAdminApplication in 17.301 seconds
=== S14) NoOp 下 B/admin 登录 + tree → HTTP=200 ×2，业务正常直查 ===
SCAN结果=[]
日志：无任何 [DATAPERM] cache put/evicted（NoOp 不写缓存不发布失效动作）
```

### 2.3 验收 3：prod 档真实启动回归

打包：`mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS（jar 07:35，含本任务全部改动）。

启动（沿 TASK-0012/0013/0014 已确认命令 + 三密钥环境变量）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
启动前端口探测：8098 无 LISTENING
Started OtAdminApplication in 17.476 seconds (process running for 18.236)
=== prod 启动段（Started 之前）ERROR 计数 === 0
=== prod 全程 ERROR 计数 === 1（唯一一条为负路径业务异常留痕：
  GlobalExceptionHandler : 业务异常：auth.captchaBlank——首次真实登录取验证码时
  未剥 RedisService JSON 序列化的引号所致，属测试者操作失误非系统缺陷，修正取码后成功）

=== prod 真实登录（GET /auth/captchaImage → Redis 读 captcha:code:{uuid} → POST /auth/login，SM4 密文密码） ===
{"code":200,"data":{"nickname":"smokePermA-T0006","token":"a46402e6-0b83-44f2-a1bb-ae010afb23a9","userId":6,...}}

=== prod tree 调用 ===
tree HTTP=200（数据正常裁剪返回）

=== prod Redis key 出现核对 ===
iot:dataperm:scope:6:0
TTL=599
```

用毕即停（task 已终止），8098 释放，冒烟产生的 dataperm key 已删除。

### 2.4 验收 4：改动范围核对

```
git diff --name-only ef7a84d..HEAD → 22 个文件（清单见 §1），全部落在 brief 声明内：
dataperm/**（6）+ system/**（8，main 仅失效事件发布点）+ IotModelInstanceServiceImpl.java（1，仅一处挂点）
+ application.yml（仅 ot.dataperm.cache-* 键族）+ src/test/**（11）
未触碰 frontend/、ingest/、dataperm 翻译逻辑本体（DataPermServiceImpl 零改动——0005 接缝先缓存后计算已按语义写好）。
提交：db16079（main，注明 TASK-0006）。
```

## 3. 未做项与原因

1. **多实例跨节点失效（Redis pub/sub）未实现**：brief 裁决 2 单机一期口径（进程内 Spring 事件即可），跨节点失效登记 TD 随集群形态任务裁决。
2. **实例创建/删除未挂失效**：~~brief 明确「绑部门变更点失效，仅一处」= updateInstance~~——**修订轮 I-2 已修**（评审退回必修项，协调者授权把 brief「实例侧仅一处」扩展为 create/update/delete 三处挂点全覆盖，见文末「修订轮」一节）。原条目指路错误（误写「登记遗留 §5.2」——§5.2 实为 dev 库冒烟数据清理项，TD 类登记在 §5.1）随本条闭环一并修正。
3. **用户创建/删除未挂失效**：新用户无缓存可失效；删除用户已被 kickOut（BUG-0043 既有逻辑）且不能再登录，其缓存键残留仅是死数据（TTL 兜底）。brief 失效点清单亦未列。
4. **resolveClientScope（中台路径）不缓存**：brief 接线仅覆盖 resolve(userId)；中台 scope 由 appKey 静态配置（iot_api_client.scope）驱动，且 deptIds 恒空集、无 (user_id, 版本) 语义。如需缓存归中台域后续任务。

## 4. 偏离计划之处

1. **SysUserServiceImpl.updateUser 补充一处单用户失效**（`updated && user.getDeptId() != null → evictUser(user.getId())`，同值失效为无害多余）：brief 失效点清单只列了 updateUserRoles，但用户自身部门变更直接改变 scope∈{2,3} 的部门求解输入（`userMapper.selectDeptIdByUserId`），不失效则部门搬迁后权限最长 10 分钟不更新，违背 REQ-0002 验收 4 方向。属 grep 核实发现的补充变更点（brief「grep 自行核实全部变更点」授权范围内），不扩文件范围。
2. **无事务上下文时立即发布而非照 0003 跳过**：0003（租户开通建库触发）跳过无害是因为有开通扫描兜底；本组件的兜底只有 TTL（10 分钟权限放大窗口）。且 brief 指定的挂点 updateRoleMenu/updateRoleDept/updateDataScope/addUsersToRole 等本就无 @Transactional 注解，「跳过」会漏掉这些点的失效。已在 Publisher javadoc 说明。
3. **键模板用 rawStringRedisTemplate（无 key-prefix）而非 @Primary 的 stringRedisTemplate**：验收命令 `redis-cli --scan --pattern 'iot:dataperm:scope:*'` 严格要求物理键形态；带前缀会写成 `ot_framework:iot:dataperm:scope:*` 导致验收命令查不到。rawStringRedisTemplate 的设计用途即「自拼完整键名的组件」（当前使用方微信公众号票据同类）；dataperm 键命名空间 `iot:dataperm:scope:*` 全局唯一，跨系统隔离以 database 分库为准（ADR-1009）。
4. **SysRoleServiceImpl 失效点按 grep 核实全挂 9 处**（brief 字面只点名 3 处）：裁决 1 失效粒度 ①② 本就要求绑定变更单用户、授权变更全量；updateRoleDept/updateDataScope/deleteRoleById 是授权/角色本体变更点，addUsersToRole/removeUserFromRole/removeUsersFromRole 是绑定变更点，漏挂任意一处即留 10 分钟权限放大窗口。

## 5. 遗留问题

1. **TD 登记（建议）**：多实例部署的跨节点缓存失效需 Redis pub/sub 或版本化 key（失效即失效模型在单实例内自洽，跨节点失效事件不通）；随集群形态任务裁决。
2. **dev 库冒烟留痕（是否清理由协调者裁决）**：角色 6/7（smoke_perm_a_t0006/smoke_perm_b_t0006，各含菜单 4030/4031/4035；角色 7 绑 sys_role_dept 部门 2）；用户 6/7（smoke_perm_a_t0006/smoke_perm_b_t0006，密码 Perm@2006a）；sys_user_role 绑定（6→6、7→7）；**实例 F02(id=5) dept_id 已由 2 改绑 4**（S9 全量失效核对的操作留痕，会影响既有 smoke 场景中 F02 归属）。重跑冒烟可复用同批账号。
3. **失效失败仅 TTL 兜底**：Redis 故障时 evict 静默（WARN 留痕），权限最长 10 分钟不更新；单机一期可接受（与缓存故障降级为重算同方向），多实例/高安全形态下宜配合 TD-1 的跨节点失效方案收敛。
4. **TTL 过期的真实时钟行为**：单测覆盖 TTL 配置传递与误配回落；真实 600s 过期在 dev/prod 冒烟以 TTL=592/599（随写随查递减）实证，未等待完整 600s 自然过期（等价成立：TTL 由 Redis 服务端原子执行，SET 时已带）。

## 6. 修订轮（2026-11-22，评审退回 §13）

- 评审：`reports/TASK-0006-review.md`——Critical 0 / Important 2 / Minor 6
- 处置裁决（协调者）：**I-1 接受为残余风险，登记 TD-0055**（todo.md；在途 resolve 竞态窗口毫秒级、TTL 600s 有界、方向为放大而非泄露，与 brief「TTL 仅兜底」模型一致；版本化 key 留演进位，多实例/强即时场景时收敛）——**本项不改代码**；**I-2 必修**（createInstance/deleteInstances 补失效挂点，授权把 brief「实例侧仅一处」扩展为 create/update/delete 三处全覆盖）；Minor 6 条不动记 deferred（含 M-6 报告账目勘误——以 §1 清单逐项相加 50 为准；M-3 丢堆栈/M-4 同步失效尾延迟/M-5 键无租户段等留待后续）。
- 修订轮改动（commit `1065f9a`，2 文件，均在授权范围内——dataperm 域零改动）：

| 文件 | 内容 |
| --- | --- |
| `backend/.../iot/instance/service/impl/IotModelInstanceServiceImpl.java` | createInstance/deleteInstances 各补一行 `dataPermCacheEvictPublisher.evictAll()`（两方法均 @Transactional，afterCommit 随提交发布；写失败/拒绝路径不发布） |
| `backend/.../iot/instance/service/impl/IotModelInstanceServiceImplTest.java` | 新增 4 用例（红转绿）：create 成功发布 / 撞码失败不发布 / delete 成功发布 / 删除被拒不发布 |

### 6.1 补充验收 1：全量复跑（含红转绿记录）

```
红（实现前）：mvn test -pl ot-admin -Dtest="IotModelInstanceServiceImplTest"
[ERROR] Tests run: 60, Failures: 2, Errors: 0  <<< FAILURE!
  IotModelInstanceServiceImplTest.testCreate_publishesEvictAll:1254
  IotModelInstanceServiceImplTest.testDelete_publishesEvictAll:1284
（恰为两个行为用例失败：create/delete 未发布失效；两个「失败不发布」用例先行通过）

实现（各一行 evictAll）后绿：
[INFO] Tests run: 60, Failures: 0, Errors: 0, Skipped: 0  -- IotModelInstanceServiceImplTest
[INFO] BUILD SUCCESS

mvn install -pl ot-core -DskipTests
[INFO] BUILD SUCCESS
mvn test -pl ot-admin
[INFO] Tests run: 1537, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

1537 = 修订轮前 1533 + 新增 4；红转绿 2（占位行为缺失）→ 实现后全绿。

### 6.2 补充验收 2：dev 冒烟复核（8098 纪律：启动前无 LISTENING；`Started OtAdminApplication in 17.433 seconds`；自起实例用毕即停，停后 8098 已释放）

```
=== R1) admin / A（smoke_perm_a_t0006，既有冒烟账号）登录；A 调 tree 制造缓存 key ===
A tree HTTP=200
SCAN：iot:dataperm:scope:6:0

=== R2) admin 新建绑部门实例 T06R2（POST /iot/instance，工厂级 dept=4） → 全量失效 key 消失 ===
{"code":200,"data":null,"message":"操作成功"}
SCAN结果=[]（A 的 key 即时消失）
落库：id=17 T06R2 dept_id=4 deleted=0
日志：[DATAPERM] visible-scope cache evicted pattern=iot:dataperm:scope:* keys=1 / evicted ALL

=== R3) A 再登录调 tree → key 重建且新建实例即时可见（REQ-0002 验收 4 方向补强） ===
A tree HTTP=200；tree 响应含 T06R2
重建 key：{"allTenant":false,"deptIds":[4],"instanceIds":[3,5,6,8,11,12,13,14,15,16,17],"createByOnly":false}
（17 已入可见集；5=F02 因首轮冒烟改绑部门 4 亦在——库内现值口径）

=== R4) admin 删除实例 id=17（DELETE /iot/instance/17） → key 消失 ===
（留痕：首次误用 DELETE /iot/instance 无路径参数 → 405 方法不支持，系测试者端点笔误，重试正确端点）
{"code":200,"data":null,"message":"操作成功"}
SCAN结果=[]
落库：id=17 deleted=1
日志：[DATAPERM] visible-scope cache evicted pattern=iot:dataperm:scope:* keys=2 / evicted ALL
（keys=2={1,6}：admin 在 deleteInstances 数据权限防线中 resolve 回填的 key 与 A 的 key 一并失效）

=== 用毕即停 ===
停止 ot-admin：PID=15144（PID 文件，java.exe）→ 已停止；8098 已释放；dataperm 冒烟键已清空
```

### 6.3 修订轮遗留

- I-1 残余风险随 TD-0055 跟踪（本报告 §5.1 为初版表述，TD-0055 为权威登记）。
- 首轮冒烟库内留痕（§5.2）不变：角色 6/7、用户 6/7、F02(id=5) dept 2→4；本轮自建实例 T06R2 已删（deleted=1 逻辑删行，编码永久占用 uk——复跑冒烟换码）。
