# TASK-0064 实施报告：高危权限点清单注册与菜单种子（机械差集收口）

- BASE：`7771181`（开工时主仓 HEAD，即 TASK-0082 提交）
- 本任务提交：`537366e`（main，注明 TASK-0064；BASE→本提交无交错提交，4 文件线性落库）
- 结论先行：**五条验收全部通过**。官方口径全量 `mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → **2370 = 基线 2366（TASK-0052 报告实测）+ 净增 4，0 失败 0 错误 BUILD SUCCESS**；TDD 红→绿（红相 3 失败全部指向缺失种子/三设计位）；升级脚本二次执行两次 DONE（第二轮 OK(0)、行数恒 3+3）；机械差集比对**零缺失终态**（设计声明 40 项 100% 注册）；dev 403 冒烟 6/6（角色 7 直调三接口 code=403「无权访问该资源」+ 403 审计行带精确缺权原因落库 + admin 导出 200 旁证）；prod 档真实启动 14.5s + **ERROR=0**；提交 4 文件全落声明范围。

---

## 0. 机械差集法四步执行实录

| 步 | 动作 | 结果 |
| --- | --- | --- |
| ① 提取设计清单 | `resources/scripts/export-design-perms.py`（pypdf，§3.1 上=p21/下=p22/续=p23 逐行权限列） | 40 项唯一权限位。两处 PDF 抽取伪影机械归一：连字 `conﬁg`(U+FB01)→`config`；表格换行拆行的共享前缀组合（`iot:instance:add / edit / remove`）按 `(iot:res):(action)((?: / next)+)` 展开逐值 |
| ② 提取注册全集 | 双源：resources/sql 全量种子 grep + 现库实态（sql-exec.sh `SELECT DISTINCT perms FROM sys_menu`） | 种子文件 64 个 iot 值 + 库实态一致（含 4029 `iot:template:item:edit` 四段式——首轮 grep 正则只匹配三段曾误判缺失，修正后排除） |
| ③ 机械差集 | 设计 40 项 − 注册全集 | **恰 3 项缺注册：`iot:point:import` / `iot:point:export` / `iot:item:bind`** |
| ④ 补种子 | `20261123_iot_permissions_complete.sql`：4079/4080 挂 4011 点位管理、4081 挂 4010 实例管理（menu_type=3，现最大 4078 后顺延）+ 超管 role_id=1 授权三行 | 两次执行 DONE，二次幂等 OK(0) |

命名映射登记（比对脚本内置）：设计 `iot:config:{publish,list,rollback}` ≡ 在库 `iot:publish:{publish,list,rollback}`（TASK-0024 既成命名，1:1 三值映射）。设计 p22 回滚行权限列原文即复用 `iot:config:publish`（未单列回滚位），交付侧已分离出更细的 `iot:publish:rollback`（4073）——落在差集脚本「注册超集」信息项，即 brief 决策 2「已分离的不动」。

### 高危单独权限位核验（TASK 明文八项）

| 高危项 | 设计权限列 | 核验结论 |
| --- | --- | --- |
| 公式 | §3.1 p22：`iot:item:edit`（设计未单列） | 注解实态一致（bind/formula/rollup/thresholds 共用）→ 不动 |
| 绑定 | §3.1 p22：`iot:item:bind`（设计单列） | 种子缺失 → **本任务补注册 4081**（注解实态 `iot:item:edit` 粗位复用，登记差异） |
| 部门绑定 | §3.1 p22「绑部门在此维护（deptId）」并入实例 CRUD | 无独立端点 → 不设独立位 |
| 状态规则 | `iot:statusrule:list/add/edit/remove` | 已注册（4063-4066，TASK-0015 交付）→ 已分离，不动 |
| 阈值 | 设计 §3.1 无独立行（0036 扩展端点注解 `iot:item:edit`） | 设计未单列 → 不动 |
| 发布回滚 | 设计原文复用 `iot:config:publish`；交付已分离 `iot:publish:rollback`（4073） | 已分离且已注册 → 不动 |
| 凭据 | `iot:credential:edit`（写/吊销）+ `iot:credential:view`（解密查看）双分离位 | 已注册（4051-4052，TASK-0019 交付）→ 不动 |
| 重算 | §3.4 recalculate 为内部令牌接口，权限列无 perms 值；管理面无对应端点（一期未建） | 无可注册对象 → 不建（登记遗留 §4） |

### 角色初始化段（brief 决策 3）

种子只注册权限点 + 超管授权（role_id=1，照 0024/0052 范式）；**普通角色零反向授权**。守卫测试 `superadminIsTheOnlyGrantedRole` 机械锁定：sys_role_menu 恰三条且全为 role_id=1。普通查看角色 403 由 sa-token 注解天然成立 + dev 冒烟实证（§2.3）。

---

## 1. 改动文件清单（4 文件，全落 brief 声明范围）

| 文件 | 性质 | 说明 |
| --- | --- | --- |
| `resources/sql/upgrade/20261123_iot_permissions_complete.sql` | 新增（brief 点名路径） | 差集三缺项补注册种子：sys_menu 4079-4081 三行（menu_type=3，module_id=4，挂点/号段/八项高危核验/上游差异/幂等口径头注全套）+ sys_role_menu 超管授权三行；INSERT IGNORE 幂等；既有种子行零触碰 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/IotPermissionsCompleteGuardTest.java` | 新增（brief 允许 test/java/**） | 守卫测试 4 用例：①新种子三行一一对应（id/parent/menu_type/perms）②sys_role_menu 恰三条超管授权③**全局**注解 ⊆ 种子登记集（扫描 net.otlab.admin.module 全部 @RestController 类级+方法级 @SaCheckPermission，解析 01_base/02_survey/03_wechat/upgrade 全量 sys_menu INSERT——含 `INSERT...SELECT` 形态与显式列序感知）④三高危端点注解实态锁定 + 三设计位已注册 |
| `resources/scripts/export-design-perms.py` | 新增（brief 允许清单导出脚本放 resources/scripts/） | PDF §3.1/续 权限列机械导出（连字归一 + 斜杠组展开） |
| `resources/scripts/perm-diff-check.sh` | 新增（brief 允许比对脚本放 resources/scripts/） | 设计声明 vs 种子∪库实态双源差集比对，零缺失裁决 + 新种子实态核对 |

不碰：既有种子行（幂等增量）、前端、ingest、ot-core 主码、0051 鉴权链、workitems 库（本报告为唯一例外）。冒烟脚本 `backend/target/t0064_smoke.py` 为 target 临时产物不入库。

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` —— 通过

```
$ mvn install -pl ot-core -DskipTests
[INFO] BUILD SUCCESS（ot-core 安装静默通过）

$ mvn test -pl ot-admin（全量，官方口径终码）
[INFO] Tests run: 2370, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  56.500 s
```

2370 = 基线 2366（TASK-0052 报告实测）+ 净增 4（本守卫测试 4 用例，零改动既有测试）。

### 2.2 验收 2：TDD 红转绿 + 种子二次幂等 + 差集比对零缺失 —— 通过

**红相**（种子未建时，`-Dtest=IotPermissionsCompleteGuardTest`）：

```
[ERROR] Tests run: 4, Failures: 3, Errors: 0, Skipped: 0
[ERROR] IotPermissionsCompleteGuardTest.highRiskEndpointsAnnotatedAndDesignPermsRegistered:149
        设计 §3.1 声明的三个高危细粒度位必须已注册：[iot:item:bind, iot:point:export, iot:point:import]
        缺失=[iot:item:bind, iot:point:export, iot:point:import]
[ERROR] IotPermissionsCompleteGuardTest.newSeedRegistersExactlyTheThreeHighRiskRows:78
        升级脚本不存在：D:\workspace\WisFactory\resources\sql\upgrade\20261123_iot_permissions_complete.sql
[ERROR] IotPermissionsCompleteGuardTest.superadminIsTheOnlyGrantedRole:97
        升级脚本不存在：…（同上）
[INFO] BUILD FAILURE
```

红相同时自证：全局注解 ⊆ 种子集用例（第 4 用例中已通过的部分）在种子缺失前即绿——存量 62 个注解值全部已登记，缺的恰是三个设计位。

**绿相**（种子落库后）：

```
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

**为什么现在对了**：种子以 INSERT IGNORE 注册恰三行（4079 iot:point:import@4011 / 4080 iot:point:export@4011 / 4081 iot:item:bind@4010）+ 超管恰三授权，与守卫断言的机械差集结果逐值相等——红相三失败全部归零，且全局口径（注解⊆种子、注解实态锁定）在种子落库前后语义一致。

**种子二次幂等**（sql-exec.sh 每条语句单独一次调用，两轮）：

```
第一轮：INSERT … sys_menu (4079-4081) → OK(3) / DONE
        INSERT … sys_role_menu (1,4079-4081) → OK(3) / DONE
核对：  SELECT id,menu_name,menu_type,perms FROM sys_menu WHERE id BETWEEN 4079 AND 4081
        → 4079 点位导入 3 iot:point:import / 4080 点位导出 3 iot:point:export / 4081 绑定点位 3 iot:item:bind
        SELECT role_id,menu_id FROM sys_role_menu WHERE menu_id BETWEEN 4079 AND 4081 → 1/4079,1/4080,1/4081
第二轮：INSERT … sys_menu → OK(0) / DONE
        INSERT … sys_role_menu → OK(0) / DONE
        COUNT 复核 → menu_rows=3, grant_rows=3
```

**差集比对零缺失终态**（`bash resources/scripts/perm-diff-check.sh` 输出）：

```
== 设计声明（§3.1/续，归一后）共 40 项 ==
iot:channel:add/edit/list/remove
iot:credential:edit/view
iot:dict:add/edit/import/list/query/remove
iot:gateway:add/edit/list/remove
iot:instance:add/edit/list/remove
iot:item:bind/edit/list
iot:metric:add/edit/list/remove
iot:point:add/edit/export/import/list/query/remove
iot:publish:list/publish
iot:template:add/edit/list/remove

== 缺注册（设计声明 - 已注册全集）——期望空（零缺失） ==
(空) —— 零缺失：设计声明权限位 100% 已注册

== 注册超集（已注册全集 - 设计声明，既有交付扩展位，信息项） ==
iot:channel:query / iot:gateway:query / iot:instance:query / iot:instance:tree /
iot:item:add / iot:item:query / iot:item:remove / iot:metric:query /
iot:publish:query / iot:publish:resend / iot:publish:rollback / iot:statusdict:*（3） /
iot:statusrule:*（4） / iot:template:item:edit / iot:template:query / iot:template:remove /
iot:tenant:*（7） / monitor:actuator:view / monitor:cache:* / monitor:job:* / monitor:online:* /
openapi:client:*（3） / survey:*（20）/ system:*（45）/ wechat:*（21）——既有交付面与框架域
```

（超集清单为节选展示，脚本全量输出；回滚分离位 `iot:publish:rollback` 在超集中——见 §0 命名映射与高危核验表。）

### 2.3 验收 3：dev 403 冒烟 —— 通过

启动前 8098 无 LISTENING；`Started OtAdminApplication`（dev 档）。角色 7 = `smoke_perm_b_t0006`（TASK-0006 冒烟留痕账号，仅持 iot:instance:list/query/tree 三查看位——三接口注解位 point:add/point:list/item:edit 均不在其集内）。`python backend/target/t0064_smoke.py` 原始输出：

```
OK   | S1 admin login | HTTP 200 token=有
OK   | S2 viewer login(角色7) | HTTP 200 token=有
OK   | S3a viewer POST /iot/point/import -> 403 | HTTP 200 code=403 msg=无权访问该资源
OK   | S3b viewer GET /iot/point/export -> 403 | HTTP 200 code=403 msg=无权访问该资源
OK   | S3c viewer PUT /iot/item/bind -> 403 | HTTP 200 code=403 msg=无权访问该资源
OK   | S4 admin GET /iot/point/export -> 200 xlsx(旁证授权链)
     | HTTP 200 content-type=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=utf-8 bytes=4169
TOTAL=6 OK=6 FAIL=0
SMOKE DONE
```

403 审计留痕（PermissionDeniedEvent 链，sql-exec.sh 核对）：

```
SELECT id,user_name,business_type,request_method,url,status,fail_reason FROM sys_operate_log
WHERE user_name='smoke_perm_b_t0006' ORDER BY id DESC LIMIT 6
5610  smoke_perm_b_t0006  7  PUT   /iot/item/bind       0  缺少权限：iot:item:edit；无此权限：iot:item:edit
5609  smoke_perm_b_t0006  7  GET   /iot/point/export    0  缺少权限：iot:point:list；无此权限：iot:point:list
5608  smoke_perm_b_t0006  7  POST  /iot/point/import    0  缺少权限：iot:point:add；无此权限：iot:point:add
5607  smoke_perm_b_t0006  7  POST  /auth/login/test     1  null
```

用毕即停：`dev.cmd stop` → PID=60540 已停止，`netstat :8098` 无 LISTENING。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn package -DskipTests -pl ot-admin`（终码 jar 133.6MB）。启动命令（沿 0052 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

干净单轮（日志 /tmp/t0064_prod.log）：

```
2026-11-23T06:57:47.738+08:00  INFO … net.otlab.admin.OtAdminApplication :
    Started OtAdminApplication in 14.545 seconds (process running for 15.067)
liveness=200
grep -ac " ERROR " /tmp/t0064_prod.log → 0
test 登录 prod 档拒绝旁证：{"code":500,"data":null,"message":"测试登录接口仅限开发环境使用"}
```

- prod 启动 14.5s、**ERROR=0**；liveness 200；test 登录在 prod 被正确禁用（新菜单行随库实态在 prod 同库生效，无额外迁移动作）。
- 用毕即停：`taskkill /F /PID 60544` → `netstat :8098` 仅余 TIME_WAIT 客户端残留、无 LISTENING。

### 2.5 验收 5：改动范围核对 —— 通过

```
$ git diff --name-only 7771181..HEAD
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/config/IotPermissionsCompleteGuardTest.java
resources/scripts/export-design-perms.py
resources/scripts/perm-diff-check.sh
resources/sql/upgrade/20261123_iot_permissions_complete.sql
（4 files changed, 630 insertions）
```

逐一核对：升级脚本 1 = brief 点名路径；测试 1 = `test/java/**` 内新增守卫（既有测试零改动）；脚本 2 = brief 允许的 `resources/scripts/` 临时产物位（清单导出 + 比对）。不碰前端/ingest/ot-core/既有种子行。TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 三端点注解收紧到设计细粒度位（bind→iot:item:bind、import→iot:point:import、export→iot:point:export） | main java 不在 brief 改动范围（仅种子 SQL + 测试）；粗位复用是 TASK-0016/0022 既成交付差异。本任务把设计位**先行注册**（收紧时零 403 窗口）并用守卫测试锁定注解实态防再漂移 |
| 重算权限位（§3.4 recalculate） | 权限列为内部令牌调用方（非 sys_menu perms）；管理面无对应端点（一期未建）→ 无可注册对象 |
| 部门绑定独立权限位 | 设计 §3.1 p22 明文并入实例 CRUD（deptId 在实例维护），无独立端点 |
| 清理既有冒烟账号/角色（TASK-0006 留痕） | 不属本任务范围；账号 7 本次作为冒烟角色复用（零改动） |

## 4. 偏离计划之处

1. **号段**：brief 写「3000-3999 段顺延」——现库 iot 业务按钮段实际在 4000-4999（TASK-0006 起既成号段，3000 以下为上游框架段且 sys_menu AUTO_INCREMENT=3000）。按 brief 同句「**现最大编号后顺延**，头注登记段位」执行：4079-4081（现最大 4078 后），头注登记全号段占用图。
2. **设计回滚权限列原文**：§3.1 p22 回滚行权限列原文为 `iot:config:publish`（设计未单列回滚位）；交付侧已分离 `iot:publish:rollback`（4073）。差集比对以设计原文为准（零缺失成立），分离位落入「注册超集」信息项——即 brief 决策 2「已分离的不动」的实证。
3. **首轮种子提取误报**：早期 grep 正则 `[a-z_]+:[a-z_]+` 漏四段式 `iot:template:item:edit` 一度误判为注解缺口；修正为四段式提取后排除（守卫测试的 SQL 解析器按显式列序感知，无此盲区）。**为什么现在对了**：全局口径断言在修正后双向核对通过（62 注解值全部登记），且 4029 行在库实态可见。
4. **S4 冒烟断言形态**：导出端点返回 xlsx 二进制流而非 JSON，首版按 `code=200` 解析报 UnicodeDecodeError；改按 Content-Type + 字节数断言。**为什么现在对了**：以实测响应形状为准（0051 冒烟同口径先例）。

## 5. 遗留问题

1. **注解收紧的联动 tripwire**：后续若把三端点注解按设计收紧（iot:item:bind / iot:point:import / iot:point:export），`IotPointMenuPermissionGuardTest`（恰五值）与 `IotDataItemMenuPermissionGuardTest` 的注解并集断言会转红——那是刻意的防漂移设计，收紧时需同步更新对应域守卫的预期集（本任务守卫的「注解实态锁定」用例同步改）。
2. **注册超集死位**：`iot:item:remove`、`iot:gateway:query` 两行种子无注解引用（既有交付面：数据项删除走实例 CRUD 级联、网关详情复用 list）——不影响安全（多注册只放宽可见性给被授权角色），登记备查。
3. **重算管理面端点**：一期未建（§3.4 recalculate 仅内部令牌链）。后续若建「运维页重算」端点，按本任务口径需同步补权限位种子（建议 `iot:calc:recalculate`，4000 段顺延）并纳入守卫全局口径。

---

**一句话结论：通过**——五条验收全绿：全量 mvn 2370（0 失败 0 错误 BUILD SUCCESS）、种子二次幂等两次 DONE（二次 OK(0)、行数恒 3+3）+ 差集比对零缺失终态（设计声明 40 项 100% 注册）、dev 403 冒烟 6/6（角色 7 直调三接口 code=403 + 审计行精确缺权原因 + admin 旁证）、prod 启动 0 ERROR、提交 537366e 的 4 文件全落声明范围。
