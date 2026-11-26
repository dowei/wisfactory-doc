# TASK-0044 验证报告（指标定义纳入发布快照与查询接口——交付核实）

> 批次定性：纯验证、零源码改动。BASE=开工 HEAD `73b8587`；本轮只做真机验收，不改任何源码；发现的问题只记录不修。
> 结论先行：**A/B/C/D 四项真机验收全部通过，未发现 TASK-0044 实现缺口，0044 可直接 close**（D 的 value 空集是 FEAT-0013 未落写入方的环境事实，非 0044 缺口，详见 §4/§6）。

## 1 改动文件清单（应为零）

| 类别 | 结果 |
| --- | --- |
| 源码改动 | **零**。全程只执行 SQL 数据种子/还原、curl 只读 API 调用、临时脚本（均在 `D:\workspace\WisFactory\.tmp-t0044v\`，未入库） |
| 仓库 tracked diff | 收尾 `git diff --stat` 输出为空（见 §2.3） |
| 数据改动 | 临时把 `iot_metric` id=5/6/7 的 `deleted` 由 1 改 0、id=5 `period` HOUR→DAY，并新建 id=10；**已全部还原**（id=5 period=HOUR、id=5/6/7 deleted=1、id=10 物理删除），见 §3.2.B5 / §3.2.C8 |
| 端口 | 8098 用毕即停；8095/18098 无监听；宿主 3307/6379 保持不动（phpStudy） |

## 2 验证环境与前置原始输出

### 2.1 开工 HEAD 与工作区

```
$ cd D:\workspace\WisFactory && git rev-parse HEAD && git status --short
73b85872bc9c67e4f37b677be04d360b66f67c1e
 M backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java
 M backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishAutoFollowTest.java
?? .agents/
?? .dev0083r2.log
?? .tmp-0083r2/
```
（开工时 0083 修订 agent 正在同机验证，以上 2 个改动文件与 `.dev0083r2.log`/`.tmp-0083r2/` 均属该 agent，**本轮未触碰**；其随后自行提交 `7628007 TASK-0083 修订轮` 并清理临时文件，HEAD 前移。这是并发 agent 行为，不是本批改动，见 §5。）

### 2.2 端口探测（8098 用前/用后）

```
开工探测：
  TCP 0.0.0.0:3307   LISTENING 65616
  TCP 0.0.0.0:8098   LISTENING 82824   ← 0083 agent 的 spring-boot:run（net.otlab.admin.OtAdminApplication）
  TCP 127.0.0.1:6379 LISTENING 5508
  $ dev.ps1 status → ot-admin 未由本脚本运行，但端口 8098 被占用（PID=82824 java.exe）

等待 0083 agent 收尾（约 02:21 清掉 .dev0083r2.log/.tmp-0083r2、8098 释放）后启动本批实例：
  $ powershell -File dev.ps1 start
  Tomcat started on port 8098
  Started OtAdminApplication in 22.153 seconds (process running for 23.076)
  8098 LISTENING PID=70736

收尾：
  $ dev.ps1 stop → 停止 ot-admin：PID=92096（PID 文件，java.exe）→ 已停止。
  $ netstat -ano | grep LISTENING | grep -E ':(8098|8095|18098)\s'
  (8098/8095/18098 free)
  $ netstat ... :3307/:6379 → 仍 LISTENING（宿主服务未动）
  $ Get-CimInstance Win32_Process -Filter "Name='java.exe'" | Where CommandLine -like '*WisFactory*'
  (无输出)
```

### 2.3 收尾 git 状态

```
$ git diff --stat
(空)
$ git status --short
?? .agents/
?? .tmp-t0044v/
$ git log --oneline -5
7628007 TASK-0083 修订轮: F-1 follow 前置 assemble（当期快照含跟随新值）+ F-2 status=1 筛选
73b8587 TASK-0041 修订轮: ...
4d4012e TASK-0083: 发布时未锁定实例自动跟随（复用 0081 applyToVersion）
...
```
（`.tmp-t0044v/` 为本批临时脚本/原始输出目录，未入库；`.agents/` 为开工既存未跟踪目录。）

## 3 逐项验证：命令、原始输出、结论

统一入口：临时 `api.sh`（token 从 `.tmp-t0044v/token-admin.txt` 读取）、`sql-exec.sh` 每条语句单独一次调用、Git Bash 全路径 `C:\Program Files\Git\bin\bash.exe`。

### 3.A 快照纳入实证 —— 通过

**A0 基线（未造数据时）**：DB 中 6 条 `iot_metric` 全为逻辑删（`deleted=1`），最新已发布版 v14 的 `metrics=[]`；直接拉最新快照一度报 500“快照文件缺失”——定位为**配置**而非实现缺口：dev.ps1 工作目录是 `backend/`，默认 `./data/snapshots` 解析到 `backend/data/snapshots`（只到 v5），而 v14 文件在 `backend/ot-admin/data/snapshots/`。重启时注入 `OT_PUBLISH_STORAGE_PATH=D:\workspace\WisFactory\backend\ot-admin\data\snapshots` 后取回正常（见 §5）。

```
$ bash sql-exec.sh "SELECT id,... FROM iot_metric ORDER BY id;"
id  deleted period ...
4   1       DAY
5   1       HOUR
6   1       DAY
7   1       DAY
8   1       DAY
9   1       DAY

$ bash api.sh GET '/iot/publish/snapshot?scopeType=2&scopeId=0'   # 重启前
{"code":500,"data":null,"message":"快照文件缺失（存储损坏探测，见服务端告警日志）",...}
__HTTP__200
$ 日志：[SNAPSHOT] snapshot file missing tenant=1 scope=2-0 version=14 uri=1/2-0-14.json

$ 注入 OT_PUBLISH_STORAGE_PATH 重启并重新登录后重试：
$ bash api.sh GET '/iot/publish/snapshot?scopeType=2&scopeId=0'
code 200 message 操作成功
version 14 hash f8a1633255b8b117b6c8928975cf095acd2b5364459e61680ba2a32f5f467da7
schemaVersion 1.0.1 metrics []
```

**A1 造活数据**：复用既有软删行 id=5（HOUR/instance3）、id=7（DAY/instance12），`deleted 1→0`（新建会占用唯一编码归档，复用更安全）：

```
$ bash sql-exec.sh "UPDATE iot_metric SET deleted=0 WHERE id IN (5,7); SELECT ..."
OK(2)
id  metric_code      period  agg_type  instance_id  dept_id  status deleted
5   T43_SMOKE_AGG_H  HOUR    1         3            4        1      0
7   GEN_T43_M1       DAY     2         12           null     1      0
DONE
```

**A2 发布并拉快照**：

```
$ bash api.sh POST '/iot/publish/publish' '{"scopeType":2,"scopeId":0}'
{"code":200,"data":{"id":54,"version":15,"snapshotHash":"72c44f20ae30dd27c60bfaec4b29e61ba109c32d103eee96ff7d4ac869aa3987","snapshotUri":"1/2-0-15.json","publishStatus":1,...},"message":"操作成功"}
__HTTP__200

$ bash api.sh GET '/iot/publish/snapshot?scopeType=2&scopeId=0' > A2-snapshot-v15.out
HTTP 200 code 200 message 操作成功
version 15 schemaVersion 1.0.1 hash 72c44f20...aa3987
metrics count= 2
{"aggType": 1, "calendarType": 1, "deptId": 4, "id": 5, "instanceId": 3, "metricCode": "T43_SMOKE_AGG_H", "metricName": "冒烟小时平均电耗", "momFlag": 0, "period": "HOUR", "status": 1, "tenantId": 1, "yoyFlag": 0}
{"aggType": 2, "calendarType": 1, "id": 7, "instanceId": 12, "metricCode": "GEN_T43_M1", "metricName": "批量生成-日累计电耗", "momFlag": 0, "period": "DAY", "status": 1, "tenantId": 1, "yoyFlag": 0}
```

**A3 逐字段比对**（Python 解析快照 vs `iot_metric` 活行，GBK 解码 DB 中文列）：

```
$ bash sql-exec.sh "SELECT ... FROM iot_metric WHERE deleted=0 ORDER BY id;"
id  tenant_id metric_code      instance_id agg_type period ... status deleted
5   1         T43_SMOKE_AGG_H  3           1        HOUR   ... 1      0
7   1         GEN_T43_M1       12          2        DAY    ... 1      0
DONE
snapshot metric ids= [5, 7] DB active ids= [5, 7]
id=5 metricCode=T43_SMOKE_AGG_H period=HOUR aggType=1 fullKey(instanceId)=3  => MATCH
id=7 metricCode=GEN_T43_M1      period=DAY  aggType=2 fullKey(instanceId)=12 => MATCH
A RESULT: PASS — snapshot.metrics every field == active iot_metric row; metricCode/instanceId/aggType/period identical
```

**A 结论：通过**。快照体含非空 `metrics` 数组，Schema v1.0.1 逐字段与 `iot_metric` 活行一致（含 fullKey `metricCode`/`instanceId`、`aggType`、`period`）。

### 3.B “保存≠生效”实证 —— 通过

**B1 改库不发布**：

```
$ bash sql-exec.sh "UPDATE iot_metric SET period='DAY' WHERE id=5; SELECT ..."
OK(1)
id  metric_code      period agg_type instance_id
5   T43_SMOKE_AGG_H  DAY    1        3
DONE
```

**B2 再拉快照（未发布）**：

```
$ bash api.sh GET '/iot/publish/snapshot?scopeType=2&scopeId=0'
snapshot version= 15 id=5 period= HOUR aggType= 1 metricCode= T43_SMOKE_AGG_H
DB current period=DAY (changed, not published)
B2 RESULT: PASS — snapshot still HOUR, save != effective
```

**B3 发布 + B4 回读**：

```
$ bash api.sh POST '/iot/publish/publish' '{"scopeType":2,"scopeId":0}'
{"code":200,"data":{"id":55,"version":16,"snapshotHash":"a7c5db020b828bcf57aead61cab23a5b9333456d9804893d2dd157ea73d4b5e7","snapshotUri":"1/2-0-16.json","publishStatus":1,...}}
__HTTP__200

$ bash api.sh GET '/iot/publish/snapshot?scopeType=2&scopeId=0'
snapshot version= 16 hash= a7c5db...73d4b5e7
id=5 metricCode= T43_SMOKE_AGG_H period= DAY aggType= 1
B4 RESULT: PASS — snapshot reflects new period=DAY after publish
```

**B5 测完还原（id=5 period 回 HOUR）**：见 §3.C8 的合并还原输出；最终 `id=5 period=HOUR deleted=1`。

**B 结论：通过**。未发布变更不进快照；发布后新值进快照，完全符合验收标准“指标变更未发布不生效”。

### 3.C 管理面列表数据权限 —— 通过

**C1 普通角色（无 `iot:metric:list`）直调 → 403**：沿用现成 `ry`（user 2，role 2=普通用户，SQL 核对 role 2 无任何 `iot:metric:%` 菜单）。

```
$ SQL: SELECT m.id,m.perms FROM sys_role_menu rm JOIN sys_menu m ... WHERE rm.role_id=2 AND m.perms LIKE 'iot:metric:%';
(无行)
$ curl -X POST /auth/login/test '{"tenantCode":"default","username":"ry","password":"admin123"}'
{"code":200,"data":{"userId":2,"username":"ry","token":"ac101ea4-...","tokenTimeout":86400},"message":"操作成功"}
$ TOKEN=<ry> bash api.sh GET '/iot/metric/list?pageNum=1&pageSize=10'
{"code":403,"data":null,"message":"无权访问该资源","timestamp":1795458504536}
__HTTP__200
```

**C2 admin 200**：

```
$ bash api.sh GET '/iot/metric/list?pageNum=1&pageSize=50'
HTTP 200 code 200 total 2
  5 T43_SMOKE_AGG_H DAY 3 4
  7 GEN_T43_M1      DAY 12 None
```

**C3–C7 数据范围过滤（沿用现成 smoke 角色 `smoke_perm_a_t0006`，user 6 / role 6 / data_scope=2 / dept 4，已授 4058-4062 五菜单——SQL 核对）**：

```
$ SQL: SELECT ur.user_id,ur.role_id,r.data_scope ... → 6  6  smoke_perm_a_t0006  2
$ SQL: role 6 菜单 → 4058/4059/4060/4061/4062（iot:metric:* 五项全在）
$ TOKEN=<smoke> bash api.sh GET '/iot/metric/list?pageNum=1&pageSize=50'
HTTP 200 code 200 total 2
  5 T43_SMOKE_AGG_H period= DAY instanceId= 3 deptId= 4
  7 GEN_T43_M1      period= DAY instanceId= 12 deptId= None
（此时 admin 同集 total 2）

$ 为制造不可见对照面，临时恢复 id=6（instance 4=dept 2，双分支皆不命中 dept4 角色）：
$ bash sql-exec.sh "UPDATE iot_metric SET deleted=0 WHERE id=6; ..."
OK(1)
id  metric_code     instance_id dept_id period
6   T43_SMOKE_INV   4           2       DAY
$ bash api.sh GET '/iot/metric/list?pageNum=1&pageSize=50'        # admin
admin total 3 ids [5, 6, 7]
$ TOKEN=<smoke> bash api.sh GET '/iot/metric/list?pageNum=1&pageSize=50'
smoke total 2 ids [5, 7]
C7 RESULT: PASS — data-scope filtering works (id6 invisible; admin sees all 3)
```

**C8 合并还原（id5/6/7 deleted=1、id5 period=HOUR、临时 id10 物理删）**：

```
$ bash sql-exec.sh "DELETE FROM iot_metric WHERE id=10; UPDATE iot_metric SET period='HOUR' WHERE id=5; UPDATE iot_metric SET deleted=1 WHERE id IN (5,6,7); SELECT ..."
OK(1)
OK(1)
OK(3)
id  metric_code      period deleted
5   T43_SMOKE_AGG_H  HOUR   1
6   T43_SMOKE_INV    DAY    1
7   GEN_T43_M1       DAY    1
DONE
```

**C 结论：通过**。无权限角色 403、admin 200、受限角色按 `dept_id IN 部门集 OR instance_id IN 可见集` 过滤（不可见指标不返回），完全符合验收标准“列表按数据范围过滤”。

### 3.D 对外 metrics/query 同源 —— 通过（value 空集为环境事实，见 §4）

**D1 test_client 签名真调**（`test_client`/`test_secret`，签名串 `appKey+timestamp+SHA256(空体)`，HMAC-SHA256，与 `OpenApiCodes`/`HmacAuthStrategy` 同口径）：

```
$ python signed_query.py DAY T43_SMOKE_AGG_H
URL= http://127.0.0.1:8098/openapi/v1/metrics/query?metricCodes=T43_SMOKE_AGG_H&period=DAY&start=1795300000000&end=1795450000000
ts= 1795458551760 bodyHash= e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
sig= 445169022ee4774d43817801a095bdfe5ae07e642294f59e50e63532bf932ea1
HTTP 200
headers {'Content-Type': 'application/json', 'Content-Length': '73', ...}
{"code":200,"data":[],"message":"操作成功","timestamp":1795458552020}
```

**D2 同源实证（SQL 日志）**：服务端日志逐次显示 `/openapi/v1/metrics/query` 走的是**同一张 `iot_metric` 活行表**，并做 scope 裁剪与审计落库：

```
$ grep -n 'openapi/v1/metrics/query\|selectByCodesForTenant' backend/logs/ot-admin-dev.log（节选）
GET "/openapi/v1/metrics/query?metricCodes=T43_SMOKE_AGG_H&period=DAY&start=...", parameters={masked}
DEBUG n.o.a.m.o.m.O.selectByCodesForTenant - ==> Preparing: SELECT id, tenant_id, metric_code, metric_name, instance_id, period, status FROM iot_metric WHERE tenant_id = ? AND deleted = 0 AND metric_code IN ( ? )
DEBUG n.o.a.m.o.m.O.selectByCodesForTenant - ==> Parameters: 1(Long), T43_SMOKE_AGG_H(String)
DEBUG n.o.a.m.o.m.O.selectByCodesForTenant - <== Total: 1
DEBUG n.o.a.m.i.d.m.D.selectAllForPermForTenant - ==> Preparing: SELECT id, instance_code, dept_id, ancestors FROM iot_model_instance WHERE deleted = 0 AND tenant_id = ?
DEBUG n.o.a.m.o.m.O.selectTenantCodeById - ... SELECT tenant_code FROM sys_tenant ...
DEBUG n.o.a.m.o.m.O.selectByIdsForTenant - ... WHERE tenant_id = ? AND deleted = 0 AND id IN ( ? )
DEBUG org.springframework.web.servlet.DispatcherServlet - Completed 200 OK
DEBUG n.o.a.m.openapi.mapper.OpenApiCallLogMapper.insert - ==> Parameters: 1(Long), test_client(String), /openapi/v1/metrics/query(String), qs=metricCodes=T43_SMOKE_AGG_H&period=DAY&start=..., 200(Integer), 263(Long)
```

**D3 scope 裁剪对照**：test_client scope=`{"subtree":true,"instanceIds":[2]}`（F01 子树，覆盖 instance 3/4）。造一条 scope 外指标（instance 5=F02，不在子树）：

```
$ bash sql-exec.sh "INSERT INTO iot_metric (... instance_id=5 ...) VALUES (...,'T44_SCOPE_OUT',...); ..."
OK(1)
id  metric_code    instance_id period deleted
10  T44_SCOPE_OUT  5           DAY    0

$ python signed_query.py DAY 'T43_SMOKE_AGG_H,T44_SCOPE_OUT' → HTTP 200 {"code":200,"data":[],...}
$ python signed_query.py DAY 'T44_SCOPE_OUT'                → HTTP 200 {"code":200,"data":[],...}
$ bash api.sh GET '/iot/metric/list?pageNum=1&pageSize=50'  # 管理面对照，四条都在
admin total 4
  5 T43_SMOKE_AGG_H instance 3 period DAY
  6 T43_SMOKE_INV   instance 4 period DAY
  7 GEN_T43_M1      instance 12 period DAY
  10 T44_SCOPE_OUT  instance 5 period DAY
```
（对外与管理面读同一 `iot_metric` 定义集，对外按调用方 scope 裁剪后仅实例分支进入查询；值空原因见下。）

**D4 负向对照**：

```
$ python (tampered signature) → HTTP 403 {"code":6001,"message":"签名不匹配",...}
$ python signed_query with period=WEEK → HTTP 200 {"code":400,"message":"period 必填且取值 HOUR/DAY/MONTH/YEAR",...}
```

**D 结论：通过**。`test_client` 真签名调用 `/openapi/v1/metrics/query` 全链（验签→scope→指标定义→审计）走通，读的正是管理面同一张 `iot_metric`（SQL 日志实证）；返回 `data=[]` 是 `st_metric_value` 无写入方（0051 报告已登记、计算引擎 FEAT-0013 未交付）导致的**值层面环境事实**，不是 0044 定义同源缺口。

## 4 未验证项与原因

| 项 | 原因 |
| --- | --- |
| D 的“返回值 ⇔ 管理面值逐条相等” | 值源 `st_metric_value` 无写入方（FEAT-0013 未交付，0051 报告 §304 已明示“空集占位，契约就位”），TDengine REST 本机 6041 未起。已退而验证“定义同源”：签名链通、`iot_metric` 同表 SQL 实证、scope 裁剪与审计落库；值相等待 0068 联调/计算引擎落地后补验。 |
| 前端页面交互层 | brief 未要求；本轮只做后端 API 真机验收。 |
| prod 档 | brief 指定 dev；prod 启动回归归 0051 已验。 |
| 快照 Schema 全字段（points/dataItems/formulas）逐列复核 | 非 0044 范围；本轮只核对 `metrics` 数组字段与 DB 一致。 |

## 5 偏离计划之处

1. **8098 并发占用**：开工时 8098 被 0083 修订 agent 的 `spring-boot:run` 占用（其日志 `.dev0083r2.log` 正在写、DB 里刚写入 v14）。未 kill、未抢端口，等其自行收尾释放后再启动本批实例。
2. **快照存储路径配置问题（配置非实现）**：首启 dev.ps1 后 `GET /iot/publish/snapshot` 报 500“快照文件缺失”——根因是 dev.ps1 工作目录 `backend/` 使默认 `./data/snapshots` 指向 `backend/data/snapshots`（只到 v5），而已发布 v14 的快照文件在 `backend/ot-admin/data/snapshots/`。用启动环境变量 `OT_PUBLISH_STORAGE_PATH` 指向既有存储目录后取回正常，**未改源码**。建议后续统一 dev 启动的工作目录/存储路径口径（登记为遗留问题，不属 0044）。
3. **数据种子策略**：因 6 条 `iot_metric` 全部逻辑删、唯一键 `uk_tenant_metric_code` 不包 `deleted`（编码归档不复用），A/B/C 采用**复用既有软删行** `deleted 1→0` 造活数据，而非新建指标；D 的范围外对照行 id=10 用后物理删除。所有改动已按 §3.C8 还原。
4. **HEAD 前移（并发 agent）**：本批开工 HEAD=`73b8587`，期间 0083 agent 在另一线程提交 `7628007`（含其 2 个 tracked 文件改动），本批未参与该提交；收尾 tracked diff 为空。
5. **临时文件**：验证脚本/原始输出存放 `D:\workspace\WisFactory\.tmp-t0044v\`（未跟踪、未入库）；期间误在 `backend/scripts/` 生成一份 raw 输出已即时删除；收尾工作区只剩开工既存 `.agents/` 与本批 `.tmp-t0044v/`（可安全清理）。

## 6 遗留问题

1. **dev 快照存储路径口径不统一**：dev.ps1 以 `backend/` 为 cwd，而历史已发布快照文件在 `backend/ot-admin/data/snapshots/`，默认配置下直接拉最新快照会 500。建议运维/后续任务统一 `OT_PUBLISH_STORAGE_PATH` 或 dev 启动 cwd（非 0044 实现缺口）。
2. **metrics/query 值空集**：`st_metric_value` 无写入方（FEAT-0013/0068 联调范围）；本轮已验定义同源，值同源待计算引擎落地后补验。
3. **test_client 种子行仍在**：scope=`instanceIds [2]`、rate_limit=8（0051 既有），本轮未改其配置；本轮调用产生的 `iot_api_call_log` 审计行属正常业务数据，未清理。
4. **dev 库 iot_metric 归档行**：id=4/5/6/7/8/9 仍为逻辑删（编码占用，本轮已还原到开工状态），后续冒烟如需同名编码请用新编码。

## 7 最终判定

**一句话：A（快照纳入+字段一致）、B（保存≠生效）、C（403/200+数据范围过滤）、D（test_client 签名同源）四项真机验收全部通过、零实现缺口，TASK-0044 可直接 close**（D 的值空集属 FEAT-0013 未交付的环境事实，按 0051 既定“空集占位”口径登记遗留，不构成本任务补实现条件）。
