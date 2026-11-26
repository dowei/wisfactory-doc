# TASK-0021 实施报告：iot_point 点位 CRUD 与配额校验

执行者：implementer（no-subagents 单体会话）
开工时间：2026-11-22 12:40 前后；报告完成：2026-11-22 13:35 前后
BASE：46d9391（派发时主仓 HEAD，开工实测记录；fix(iot) TASK-0019 修订轮）
本任务提交：9a162a8（main，feat(iot) TASK-0021）
**五条验收全部通过**（细节与原始输出见 §2；结论先行：全量回归 1808 = 开工基线实测 1755 + 本批净增 53，0 失败 0 错误 BUILD SUCCESS；升级脚本二次幂等 + 表结构与菜单 4053-4057 五行核对一致；dev 冒烟全序列绿含配额 4003 接线即生效与裁剪场景；prod 档干净单轮真实登录 200 + point/list 200 + gateway/list 200 全程 0 ERROR；本任务提交 23 文件全落声明范围）。

---

## 1. 改动文件清单（本任务提交 9a162a8，23 文件）

### 1.1 主码（13）

| 文件 | 性质 |
| --- | --- |
| `resources/sql/upgrade/20261122_iot_point_domain.sql` | 新增——iot_point DDL（附录 A p31 逐字，无上游差异登记）+ sys_menu 4053-4057 五行（parent 4011）+ 超管授权；CREATE IF NOT EXISTS + INSERT IGNORE 幂等 |
| `backend/.../iot/point/entity/IotPoint.java` | 新增——实体（protocol_addr JSON 字符串承载；remark 补列与 DDL 一致） |
| `backend/.../iot/point/mapper/IotPointMapper.java` | 新增 |
| `backend/.../iot/point/mapper/IotPointCodeMapper.java` / `IotPointCodeRow.java` | 新增——uk_tenant_point_code 双道闸第一道旁路预检（含已删行、旁路多租户插件显式 (tenant_id, point_code)） |
| `backend/.../iot/point/dto/IotPointCreateForm.java` / `IotPointUpdateForm.java` / `IotPointStatusForm.java` | 新增——三表单（创建/修改/批量启停） |
| `backend/.../iot/point/event/PointEnabledEvent.java` | 新增——建档启用领域事件 record（pointId/tenantId/pointCode/valueType/samplePeriod 五元） |
| `backend/.../iot/point/service/IotPointService.java` / `impl/IotPointServiceImpl.java` | 新增——六接口服务（唯一 4002 双道闸/配额接线/unit 字典/min<=max/枚举/归属校验/启停事件/绑定拒绝删除） |
| `backend/.../iot/point/controller/IotPointController.java` | 新增——六端点（list/detail/add/edit/changeStatus/remove），权限码 4053-4057 语义集 |
| `backend/.../iot/quota/QuotaPointCounterImpl.java` | 新增——PointQuotaCounter 真实实现（iot_point 活跃行 COUNT，tenant_id 显式 eq；0003 预留接口接线） |
| `backend/.../iot/gateway/service/impl/IotGatewayServiceImpl.java` | 修改——selectGatewayList 接 F5 数据权限裁剪（dept_id IN OR instance_id IN）+ 类头 javadoc 登记 |

### 1.2 测试（10）

| 文件 | 性质 |
| --- | --- |
| `backend/test/.../iot/point/service/impl/IotPointServiceImplTest.java` | 新增 25 用例 |
| `backend/test/.../iot/point/controller/IotPointControllerTest.java` | 新增 9 用例（standalone MockMvc） |
| `backend/test/.../iot/point/config/IotPointMenuPermissionGuardTest.java` | 新增 3 用例（SQL 4053-4057 ↔ 控制器注解双向核对，照 gateway 守卫范式） |
| `backend/test/.../iot/quota/QuotaPointCounterImplTest.java` | 新增 3 用例（真实计数语义/tenant_id 显式条件/null 防御） |
| `backend/test/.../iot/redline/PointRedlineTest.java` | 新增 6 用例（授权例外——红线套件增补） |
| `backend/test/.../iot/redline/GatewayRedlineTest.java` | 新增 5 用例（授权例外——红线套件增补） |
| `backend/test/.../iot/redline/RedlineFixture.java` | 修改——夹具扩展：点位/网关行 + Mapper 桩 + 双条件 SQL 模拟器（dept_id IN / bound_instance_id|instance_id IN / FAIL_CLOSED 字面条件求值）+ 服务真实装配 |
| `backend/test/.../iot/gateway/service/impl/IotGatewayServiceImplTest.java` | 修改——setUp 接 dataPermService（默认 allTenant）+ SecurityUtils 静态 mock + 3 个 F5 裁剪用例 |
| `backend/test/.../iot/service/impl/SysTenantPointQuotaTest.java` | 修改——「计数源未装配视为 0」过渡期兜底用例随真实 bean 交付移除（brief 裁决 3），其余 7 用例不变 |

净增用例 = 新增 54（25+9+3+3+6+5+3 网关裁剪用例）− 移除 1 = **+53**。

---

## 2. 验收命令原始输出

### 2.1 验收 1：全量回归 —— 通过

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[INFO] Tests run: 1808, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

基线以开工实测为准（brief 口径）：worktree 检出 BASE=46d9391 实测：

```
$ git worktree add D:\workspace\WisFactory-baseline0021 46d9391
$ cd backend && mvn test -pl ot-admin
[WARNING] Tests run: 1755, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

1808 = 1755 + 53（净增，明细见 §1.2），0 失败 0 错误（3 skipped 为红线二-中台 scope 占位——TASK-0051 接线后启用，基线同值）。新增测试类逐个红转绿过程：QuotaPointCounterImpl/配额接线用例首跑红（2 处用例自身缺陷：dict 种子缺失、updateById 桩缺失），定位根因为测试夹具而非主码后修正转绿——「为什么现在对了」：配额校验链路本身在 0003 mock 验收已固化，本批新增的是真实计数 bean 的装配路径与 add 调用方接线，主码首次全绿。

### 2.2 验收 2：升级脚本二次幂等 + 表结构/菜单核对 —— 通过

```
$ bash sql-exec.sh resources/sql/upgrade/20261122_iot_point_domain.sql   # 第一次
OK(0)          # SET NAMES
OK(0)          # CREATE TABLE IF NOT EXISTS（判存跳过建表分支或建表，返回受影响 0）
OK(5)          # sys_menu 4053-4057 五行插入
OK(5)          # sys_role_menu 超管五行授权
DONE
$ bash sql-exec.sh resources/sql/upgrade/20261122_iot_point_domain.sql   # 第二次
OK(0)
OK(0)
OK(0)
OK(0)
DONE           # 两次 DONE；第二次全部 OK(0) 不报错不重复插入
```

表结构与菜单核对（三次核对语句）：

```
$ bash sql-exec.sh "SHOW CREATE TABLE iot_point; ..."   （尾部节选）
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_point_code` (`tenant_id`,`point_code`),
  KEY `idx_tenant_channel` (`tenant_id`,`channel_id`),
  KEY `idx_tenant_dept` (`tenant_id`,`dept_id`),
  KEY `idx_tenant_instance` (`tenant_id`,`bound_instance_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='点位主表'
id    parent_id  menu_name  menu_type  perms
4053  4011       点位查询    3          iot:point:list
4054  4011       点位详情    3          iot:point:query
4055  4011       点位新增    3          iot:point:add
4056  4011       点位修改    3          iot:point:edit
4057  4011       点位删除    3          iot:point:remove
role_id  menu_id
1        4053
1        4054
1        4055
1        4056
1        4057
DONE
```

逐字核对：与 pypdf 提取的附录 A p31 iot_point DDL 列序/列型/注释/索引完全一致（`COLLATE=utf8mb4_0900_ai_ci` 为 MySQL 8 默认排序规则随库继承，脚本未显式指定 COLLATE——与通道/网关域脚本同口径）；菜单五行与 4053-4057 语义一一对应（4052 确为凭据域占用，号段连续取号无冲突）。

### 2.3 验收 3：dev 冒烟 —— 通过

启动：dev.ps1（8098，dev 档）→ 冒烟脚本 `backend/target/t0021_smoke.py` 原始输出（完整贴回）：

```
=== S1) admin dev login === S1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S1 token= a3b423c7-0...
    PUT /iot/tenant pointQuota=None -> HTTP 200 code 200 | 操作成功  [期望200] OK
    库内回读 point_quota = None
=== S2) 建点位：unit 用字典外值 -> 拒（TD-0045 强制单位字典校验） ===
S2a -> HTTP 200 code 400 | 单位「℃冒烟字典外」不可用：单位字典（iot_unit）尚未登记任何条目，单位从字典选取禁手填，请联系平台运营方在系统字典维护后重试  [期望400] OK
S2a 断言: 拒绝文案含「平台运营方」归口提示 OK
=== S3) 字典补登记（归口运营方流程：iot_unit 类型 + ℃ 条目） ===
S3a dict type -> HTTP 200 code 200 | 操作成功  [期望200] OK
S3b dict entry -> HTTP 200 code 200 | 操作成功  [期望200] OK
S3 断言: iot_unit 类型与条目登记完成 OK
=== S3c) 字典补登记后再建点位 -> 成功（建档启用） ===
S3c -> HTTP 200 code 200 | 操作成功  [期望200] OK
S3c 断言: 点位 id=9010 落库 status=1（启用） unit=字典值 OK
=== S4) 配额场景（运营方 API 改 point_quota：quota=当前活跃数 N → N/N 达上限拒） ===
    当前活跃点位数 N = 3
    PUT /iot/tenant pointQuota=3 -> HTTP 200 code 200 | 操作成功  [期望200] OK
    库内回读 point_quota = 3
S4a N/N 达上限 -> HTTP 200 code 4003 | 点位配额不足（上限 3）  [期望4003] OK
S4a 断言: 配额计数器真实计数（QuotaPointCounterImpl 活跃行 COUNT）→ 接线即生效，未落库 OK
=== S4b) 放宽到 N+1（N/N+1 放行口径）→ 第二个点位建成 ===
    PUT /iot/tenant pointQuota=4 -> HTTP 200 code 200 | 操作成功  [期望200] OK
    库内回读 point_quota = 4
S4b N/(N+1) 放行 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S4b 断言: 第二个点位 id=9011 落库 OK
=== S5) 裁剪场景（受限用户 smoke_perm_a_t0006 list：只见 dept 4 行） ===
S5a login -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5b point list -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5b 断言: restricted 只见 dept4 行、dept2 行不返回 -> ['T21_SMOKE_DEPT4']
S5c admin point list -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5c 断言: admin 可见 dept2+dept4（scope=1 对照面） OK
S5d gateway list -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5d 断言: 网关只见 dept4 行 -> ['T21_SMOKE_GW4']
S5e detail 越权 -> HTTP 200 code 403 | 无权访问该点位（数据权限不命中）  [期望403] OK
S5e 断言: 受限用户查不可见点位详情 -> 403（数据权限不命中） OK
=== S6) 启停事件留痕（置 0 再置 1，事件无消费方发布即留痕日志） ===
S6a disable -> HTTP 200 code 200 | 操作成功  [期望200] OK
S6b enable -> HTTP 200 code 200 | 操作成功  [期望200] OK
S6 断言: changeStatus 落库 status=1 OK（事件日志留痕核对见报告 grep 输出）
=== S7) 收尾：删除冒烟点位（point_quota 还原 NULL 走收尾 SQL——MP null 不进 SET，见报告） ===
S7 -> HTTP 200 code 200 | 操作成功  [期望200] OK
SMOKE DONE
```

启停事件日志留痕核对（事件无消费方，发布即留痕——`logs/ot-admin-dev.log`）：

```
$ grep -c "点位启用事件已发布" backend/logs/ot-admin-dev.log
11
$ grep "点位启用事件已发布" backend/logs/ot-admin-dev.log | tail -6   （节选，UTF-8 原文）
2026-11-22 13:19:56.836 ... 点位启用事件已发布（建子表触发，写入方 TASK-0029）: pointId=9010, tenantId=1, pointCode=T21_SMOKE_P1, valueType=DOUBLE, samplePeriod=30
2026-11-22 13:19:57.639 ... 点位启用事件已发布（建子表触发，写入方 TASK-0029）: pointId=9011, tenantId=1, pointCode=T21_SMOKE_P2, valueType=INT, samplePeriod=null
2026-11-22 13:19:58.596 ... 点位启用事件已发布（建子表触发，写入方 TASK-0029）: pointId=9010, tenantId=1, pointCode=T21_SMOKE_P1, valueType=DOUBLE, samplePeriod=30
```

11 条 = 三轮冒烟的建档启用事件（add 落 1）+ S6 的 changeStatus 置 1 事件（同 pointId=9010 复现发布）——建档启用与启停操作置 1 两条触发路径均留痕；停用（S6a）无事件行（置 0 不发事件，符合裁决）。

冒烟 SQL 原始输出（配额还原收尾 + 场景种子）：

```
$ bash sql-exec.sh "UPDATE sys_tenant SET point_quota=NULL WHERE id=1;
SELECT tenant_code, point_quota FROM sys_tenant WHERE id=1;"
OK(1)
tenant_code  point_quota
default      null
DONE
```

冒烟场景种子（受限用户裁剪用，T21_SMOKE_DEPT4/DEPT2 点位与 T21_SMOKE_GW4/GW2 网关，INSERT IGNORE 两次 OK(2)/OK(2)，行清单见 §4 遗留 3）。受限用户账号：dev 库既有冒烟账号 smoke_perm_a_t0006（user 6，dept 4，role 6 data_scope=2）补授 iot:point:list/query + iot:gateway:list 菜单（dev 库内操作，不入库——§3 偏离 6）。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn package -pl ot-admin -am -DskipTests` → ot-admin-1.0.0.jar。启动命令（沿 0017/0019/0020 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

**干净单轮（日志 /tmp/task0021/prod_clean.log，Redis 预置 captcha:code:t0021p="AB21" JSON 字符串 EX 600 + SM4 admin123 密文）**：

```
Started OtAdminApplication in 16.43 seconds (process running for 16.987)
=== ERROR count ===
0
LOGIN HTTP 200 code 200 message 操作成功
LOGIN token 448e99b3...
LIST /iot/point/list HTTP 200 code 200 total 2
POINT-LIST rows: [{"pointCode": "T21_SMOKE_DEPT4", "status": 1, "deptId": 4}, {"pointCode": "T21_SMOKE_DEPT2", "status": 1, "deptId": 2}]
LIST /iot/gateway/list HTTP 200 code 200 total 12
GW-LIST rows: [{"gatewayCode": "GW_ASCII_T1", ...}, {"gatewayCode": "GW_T0017_01", ...}, ...]   （10 行节选）
PROD PROBE DONE
```

真实登录 200 + point/list 200 + gateway/list 200（F5 裁剪接线在 prod 档装配无缺 bean——admin 全量可见对照），启动段与全程 **ERROR=0**。用毕即停：杀进程 → `netstat :8098 LISTENING` 复核无输出，Redis 验证码键清除（DEL → 0）。

首轮（非干净轮）说明：Redis 验证码预置值未带 JSON 引号（"AB21" 裸值）→ 登录时框架 JSON 反序列化 1 条预期 ERROR（SerializationException）→ 按 0019/0020 先例重启干净单轮交付字面 0 ERROR。该 ERROR 为冒烟预置数据形态问题，非应用缺陷；修正形态 `'"AB21"'` 后登录即通。

### 2.5 验收 5：改动范围核对 —— 通过

`git diff --name-only 587d5e6..HEAD`（本任务提交 9a162a8 相对其父提交；23 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/impl/IotGatewayServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/controller/IotPointController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/IotPointCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/IotPointStatusForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/dto/IotPointUpdateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/entity/IotPoint.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/event/PointEnabledEvent.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/mapper/IotPointCodeMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/mapper/IotPointCodeRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/mapper/IotPointMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/IotPointService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/service/impl/IotPointServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/quota/QuotaPointCounterImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/gateway/service/impl/IotGatewayServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/config/IotPointMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/controller/IotPointControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/point/service/impl/IotPointServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/quota/QuotaPointCounterImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/GatewayRedlineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/PointRedlineTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/RedlineFixture.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/SysTenantPointQuotaTest.java
resources/sql/upgrade/20261122_iot_point_domain.sql
```

全落声明范围（point/** 新包 11 文件 + quota/** 1 文件 + gateway/service/impl/IotGatewayServiceImpl.java + test/** 8 文件 + 升级脚本 1 文件）；redline 测试包增补为 brief 明文授权例外。未触碰 frontend/、ingest/、模板/实例/字典域主码、dataperm 组件本体、workitems 库（本报告除外）、TASK 详情。

---

## 3. 偏离计划之处

1. **「九接口」计数对齐**：pypdf 提取 §3.1 p21 点位段实际 8 行接口（list/detail/add/edit/remove/import/export/changeStatus）。brief 枚举本批交付六端点（list/detail/add/edit/批量启停/remove），import/export 归 TASK-0022、详情反查与发布联动归 TASK-0023（FEAT-0007 任务拆解如此）——「九接口」按点位域总计口径理解，交付严格按 brief 枚举与 §3.1 行对齐；changeStatus 因 §3.1 有独立行故为独立端点（权限 iot:point:edit 与 §3.1 权限列一致）。
2. **BASE 漂移**：工作期间另一会话向 main 提交了 TASK-0057（587d5e6，frontend 公式编辑器 6 文件）——与本任务无交集、未触碰。验收命令 5 的字面形式 `46d9391..HEAD` 会混入 0057 文件，故范围核对按本任务提交相对其父提交（587d5e6..HEAD）执行；两文件集不相交。
3. **wsl docker MySQL 形态**：容器 host 网络模式无端口映射，mysqld 直监听 WSL 侧 3307（sql-exec.sh 默认端口吻合）；WSL VM 闲置自动关机会连杀容器（XA crash recovery 循环），执行期间以常驻 sleep 会话保活。环境事实记录，非本任务改动。
4. **prod 首轮 1 条预期 ERROR**：见 §2.4（Redis 验证码值须为 JSON 字符串），按 0019/0020 先例干净单轮交付 0 ERROR。
5. **SysTenantServiceImpl 零改动**：配额接线未改校验器文件（其包 service/impl 不在 brief 声明范围）——真实 bean 装配经既有 `@Autowired(required=false)` 自动完成，校验器零改动即生效（brief「接线即生效」字面兑现）；「未装配视为 0」兜底分支成为不可达代码（保留，无害），对应过渡期单测按 brief 裁决移除。
6. **dev 库冒烟配置**：受限账号 smoke_perm_a_t0006 补授 iot:point:list/query、iot:gateway:list 三菜单行并同步密码哈希（与 admin 同值）；配额临时调整走真实 API PUT /iot/tenant（运营方改配额语义）、还原 NULL 用 SQL（MP null 不进 SET）——均为 dev 库冒烟配置，不入库不进提交。
7. **冒烟轮次修正**：S4 首版脚本把「当前活跃 1 个」写死，实际库内含裁剪种子共 3 个活跃点位 → 改为动态读 total 后按 N/N 拒、N+1 放行执行；该轮修正过程的两条 MISMATCH 原始输出未采信（脚本缺陷），最终轮全绿输出已贴回。

## 4. 未做项与原因

1. **§3.1 点位段 import/export**：归 TASK-0022（点位批量导入导出）；**详情反向绑定查询与批量启停发布联动**：归 TASK-0023——FEAT-0007 拆解如此，本批不越界。
2. **GatewayReferenceGuard.countPoints 占位转正**（0018 登记的接线义务，网关删除保护点位计数仍恒 0）：brief 改动范围未含 `channel/service/impl/IotChannelGatewayReferenceGuard.java` 与 `gateway/service/GatewayReferenceGuard.java`（越界即违约）；且 iot_point 无 gateway_id 直引列（点位归属经 channel 间接——p31 DDL 无该列），转正需按「点位→通道→网关」间接计数扩展实现。**登记请协调者裁决**：建议随 TASK-0022/0024 或修订轮处理，届时按间接计数实现、调用点与拒绝分支不变。
3. **点位写路径数据权限防线**（edit/changeStatus/remove 的越权 403）：brief F5 段只声明「点位 list + 网关 list 接裁剪 + 详情 403」，未含写防线；本批照 brief 交付读裁剪（与 0012 实例域「写防线后补修订轮」同形态）。**登记遗留**：建议修订轮按实例域 0012 修订轮 I-1 口径补（写原语目标 ∈ 可见集，403 文案不泄露标识）。
4. **建档启用后首条上报免建表入库的端到端验证**：写入方 TASK-0029 未交付，本批只发布事件（brief 裁决：判据按业务事件，端到端入库验证归 TC-0009/联调）。
5. **protocol_addr JSON 应用层 Schema**：一期预留字段仅限长 8192，OPC UA 发现字段结构随二期定义（REQ-0008 非目标一致），不做前置校验。

## 5. 遗留问题

1. **配额 TOCTOU 残余**：计数预检与落库之间的并发窗口（极端并发下可短暂超配额）——0003 评审 M-7 裁决接受并登记；兜底语义（校验器对 101/100 同样拒绝）已有单测固化。
2. **删除绑定保护当前恒放行**：iot_data_item.point_id 本期生成恒 NULL（TASK-0013 口径），批量删除的绑定计数实际恒 0——保护分支与拒绝文案已就位并被单测固化，绑定产生（0023/0029 链路交付后写入 point_id）即自动生效。
3. **dev 库冒烟残留**（不影响仓库与提交）：T21_SMOKE_DEPT4/DEPT2 点位（id 9001/9002）、T21_SMOKE_GW4/GW2 网关（9101/9102）、iot_unit 字典类型与「℃冒烟字典外」条目、role 6 的 4053/4054/4041 授权、user 6 密码哈希与 admin 同值——后续任务冒烟可复用，不需要时手工清理。
4. **「归口运营方维护单位字典」流程文档化**（TD-0045 附条件）：拒绝文案已带明确引导（「请联系平台运营方在系统字典维护后重试」），运营侧 SOP 文档待落地（非代码项）。
5. **数据权限缓存失效挂点**：点位/网关无改绑部门事件源（0012 实例改绑才发 DataPermCacheEvictEvent）——点位 dept_id 由本服务维护但缓存 TTL 10 分钟（TASK-0006 交付后）内改绑不即时失效；与实例域同款窗口口径，TASK-0006 缓存设计已覆盖（key=(user_id, 角色版本)），改绑部门粒度的失效挂点登记待 0006 评审确认。

## 6. 结论

五条验收全部通过；TASK-0021 brief 全部交付项完成（DDL+实体、六接口 CRUD、配额计数器真实实现与 add 接线即生效、PointEnabledEvent 只发布不建监听、F5 点位/网关列表裁剪同批落地 + 详情 403、红线套件增补、菜单 4053-4057、单测/冒烟/prod 回归）。TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。

---

## 7. 修订轮（验收退回一轮 §13——补 0018 报告遗留接线，主代码改动经协调者授权）

退回裁决要点：0018 报告遗留明确要求「点位引用 guard 占位：0021 brief 必带接线」，预制 brief 时遗漏携带；首轮执行者拒绝越界（正确），本修订轮获授权补上。**修订轮 commit：a61fd24**（fix(iot) 通道删除保护·点位侧 guard 转正接线，7 文件——均在授权范围「channel 域 guard 实现与测试 + point 域测试（如需）」内，point 域主码零改动）。

### 7.1 改动文件（7）

| 文件 | 性质 |
| --- | --- |
| `channel/service/ChannelReferenceGuard.java` | 修改——接口增 `pointCodes(channelId)`（前 5 条口径 javadoc）+ 转正登记 |
| `channel/service/impl/IotPointChannelReferenceGuard.java` | 新增——实查 iot_point.channel_id 活跃行（@TableLogic 过滤逻辑删行、插件注入租户条件即同租户保证）；pointCodes id 升序取前 5 条；照 0018 IotChannelGatewayReferenceGuard 网关侧转正同款对偶范式（表归点位域、实现置于通道域消费其 Mapper） |
| `channel/service/impl/PlaceholderChannelReferenceGuard.java` | 删除——0018 占位按其自身契约替换（调用点与拒绝语义不变） |
| `channel/service/impl/IotChannelServiceImpl.java` | 修改——deleteChannel 拒绝文案按 §3.1 p21「拒绝并提示」补点位编码（前 5 条，照 0017/0018 网关删除拒绝提示通道编码范式）+ 类头 javadoc 转正登记 |
| `channel/controller/IotChannelController.java` | 修改——remove 端点 @Operation 摘要同步（「其下有点位则拒绝并提示点位编码前 5 条」） |
| `test/.../IotPointChannelReferenceGuardTest.java` | 新增 5 用例（countPoints 条件形态/null 防御、pointCodes 列表/前 5 条截断/null 防御） |
| `test/.../IotChannelServiceImplTest.java` | 修改——删除段更新：零引用放行（断言 pointCodes 仅拒绝分支调用）、拒绝文案含编码、**新增转正 guard 整链用例**（真实 guard + 点位 Mapper 桩：有点位行拒并提示编码 → 清点位 → 删通道成功） |

调用面核对（必修项 2）：iot_channel 删除路径唯一入口 `IotChannelServiceImpl.deleteChannel` 全部经 guard（`channelReferenceGuard.countPoints(id)` 在存在性校验之后、removeById 之前，无旁路计数路径）；测试同步断言 pointCodes 仅在 countPoints>0 拒绝分支调用。网关侧 guard（GatewayReferenceGuard.countPoints）占位**不在本授权范围**，维持占位并保留登记（见 §7.4）。

### 7.2 红转绿记录

- **红（编译层）**：接口加 pointCodes 后先跑新测试 → `BUILD FAILURE：PlaceholderChannelReferenceGuard 不是抽象的, 并且未覆盖 ChannelReferenceGuard 中的抽象方法 pointCodes(java.lang.Long)`（占位实现无法编译通过，真实实现尚未交付——TDD 红留痕）。
- **红（行为层，dev 实证）**：修订轮代码编译进测试但 bootable jar 尚为旧构建（首轮 `mvn package` 因运行中 dev 进程锁 jar 失败，后重建）——旧 jar 下 dev 冒烟 S4a：挂点位通道 DELETE 返回 **code 200 删除成功**（占位行为：guard 恒 0 引用，删除保护实际失效）。
- **绿**：重建 jar 重启 dev → S4a 拒绝（400 并提示点位编码）；单测 45/45（含新 guard 5 用例 + 服务整链用例）。

### 7.3 补充验收原始输出

**补充验收 1：全量复跑 —— 通过**

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[INFO] Tests run: 1814, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

1814 = 修订轮前 1808 + 6 净增（IotPointChannelReferenceGuardTest 新 5 + 服务测试删除段净增 1：新增整链用例、零引用放行用例重写仍计 1）。（3 skipped 与基线同值——红线二-中台 scope 占位。）

**补充验收 2：dev 冒烟复核 —— 通过**

启动：dev.ps1（8098，dev 档，修订轮 jar）；脚本 `backend/target/t0021rev_smoke.py` 原始输出：

```
=== S1) admin dev login === S1 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S1 token= 36ac66df-0...
S1b 网关定位: GW_T0020_01 id = 14
=== S2) 建通道（MQTT，挂网关） ===
S2a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2b 断言: 通道 id=12 落库 OK
=== S3) 建点位挂该通道（channel_id 直引） ===
S3a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S3b 断言: 点位 id=9013 挂通道 12 OK
=== S4) 删通道 -> 拒并提示点位编码（0018 遗留接线转正后的行为） ===
S4a -> HTTP 200 code 400 | 通道存在点位，禁止删除（点位 1 个: T21REV_CH_M01，先清理点位）  [期望400] OK
S4a 断言: 删通道被拒且文案含点位编码 T21REV_CH_M01 OK
=== S5) 清点位 -> 删通道成功 ===
S5a 删点位 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5b 删通道 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5c 断言: 通道已软删（列表不再返回） OK
SMOKE DONE
```

红态对照输出（旧 jar 行为，§7.2 行为层红留痕，未采信为验收）：

```
S4a -> HTTP 200 code 200 | 操作成功  [期望400] !!MISMATCH    ← 修订轮接线前：挂点位通道可删
```

用毕即停：dev.ps1 stop → PID 42352 已停 → `netstat :8098 LISTENING` 复核无输出。

### 7.4 修订轮范围与遗留更新

- 本轮改动限授权范围（channel 域 guard 实现与测试），point 域主码零改动（point 域测试不需要——guard 消费点位 Mapper 的契约由 guard 测试与服务整链用例双向锁定）。
- **网关侧 guard 点位计数仍占位**（GatewayReferenceGuard.countPoints，0017 结构/0018 登记）：本轮授权只覆盖通道侧；网关删除保护的点位计数需按「点位→通道→网关」间接计数扩展（或经通道聚合），**登记留待协调者授权后续处理**。
- §4 遗留 2（删除绑定保护恒放行）与 §5 遗留清单其余项不受本轮影响；通道删除点位引用保护自本轮起实际生效（逻辑删行不计、编码不复用语义由既有双道闸预检覆盖）。
- dev 库冒烟残留新增：CH_T21REV_01 逻辑删行（编码归档占用）+ T21REV_CH_M01 已删点位行（ §5 遗留 3 同性质，不影响功能）。

### 7.5 修订轮结论

两项补充验收全部通过；0018 报告遗留「点位引用 guard 占位」接线义务在本轮落地闭环（通道删除保护实际生效）。TASK 状态维持「进行中」，未回写任务详情。

---

## 8. 修订轮（评审退回一轮——diff 评审 I-1/I-2 必修项落地）

评审结论：Critical 0 / Important 2 / Minor 8（评审报告 reports/TASK-0021-review.md）。必修两项落地，**Minor 8 条按裁决不动记 deferred**。**修订轮 commit：a961fd1**（fix(iot) 点位建档 remark 白名单补齐 + 点位三写路径数据权限防线，4 文件——均在授权范围「point 域 + redline 测试包（授权例外）」内）。

### 8.1 改动文件（4）

| 文件 | 性质 |
| --- | --- |
| `point/service/impl/IotPointServiceImpl.java` | 修改——I-1：createPoint 白名单补 `setRemark`（表单字段与落库白名单一一对应）；I-2：updatePoint/changePointStatus/deletePoints 三写原语补 `requireWritePerm` 数据权限防线（类头 javadoc 增「写路径数据权限防线」段登记口径） |
| `test/.../point/service/impl/IotPointServiceImplTest.java` | 修改——新增 5 用例（remark 白名单落库 + 写防线编辑/批量启停/批量删除三路径 403 与可见回归） |
| `test/.../redline/PointRedlineTest.java` | 修改——新增场景 6 写越权 3 用例（直改/启停/删除不可见点位三路径全 403 文案不泄露编码 + 可见点位写操作回归正常） |
| `test/.../redline/RedlineFixture.java` | 修改——点位 Mapper 补写桩（selectByIds 批量定位/updateById/deleteByIds）+ itemMapper 访问器（可见删除用例重桩绑定计数查询） |

### 8.2 两项必修实现口径

- **I-1（建档 remark 静默丢失）**：createPoint 字段白名单构建块补 `entity.setRemark(form.getRemark())`——表单 remark（@Size(max=500)）与落库白名单一一对应；updatePoint 原已承载 remark 不受影响。
- **I-2（点位写路径无数据权限——brief 层缺口，协调者认账并授权修齐）**：三写原语共用 `requireWritePerm`（单目标 + 批量两个重载），照 0012 修订轮 I-1 同型范式——存在性 → 数据权限 → 业务校验（删除防线先于绑定校验，不向无权者泄露绑定状态）。可见口径与列表裁剪同条件：allTenant，或 dept_id ∈ 部门集，或 bound_instance_id ∈ 可见集（NULL 双条件行 fail-closed）；**无 create_by 分支**（点位为资产类非个人创建物——纯 scope=4 用户无可见行保持 fail-closed，与评审裁决一致）。不可见 → 403「无权操作该点位（数据权限不命中）」文案不泄露点位编码；批量原语批内任一不可见即整批拒绝（fail-closed 不半批变更）。

### 8.3 红转绿记录

- **红**：先落 4 用例跑红——`testCreate_RemarkWhitelisted`（remark 落库 null）+ `testEdit_DataPermForbiddenOnInvisible` / `testChangeStatus_DataPermGuard` / `testDelete_DataPermGuard`（Expected BusinessException to be thrown, but nothing was thrown——三写路径对不可见点位无防线）。
- **中间修正**（测试夹具，非主码）：RedlineFixture 点位 Mapper 缺 selectByIds 桩（Mockito 默认空集合 → listByIds 走「点位不存在」400 而非 403）→ 补批量定位桩后转绿——「为什么现在对了」：批量写原语的存在性校验入口是 selectByIds（MP 3.5.17 listByIds 委托），夹具此前只覆盖 selectById 单查。
- **绿**：IotPointServiceImplTest 30/30、PointRedlineTest 9/9。

### 8.4 补充验收原始输出

**补充验收 1：全量复跑 —— 通过**

```
$ cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin
[INFO] Tests run: 1822, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

1822 = 评审轮前 1814 + 8 净增（服务测试 +5、红线点位 +3）。（3 skipped 与基线同值——红线二-中台 scope 占位。）

**补充验收 2：*RedlineTest 全绿 —— 通过**

```
$ mvn surefire:test -pl ot-admin -Dtest='*RedlineTest'
[INFO] Tests run: 8,  ... -- in net.otlab.admin.module.iot.redline.InstanceRedlineTest
[INFO] Tests run: 9,  ... -- in net.otlab.admin.module.iot.redline.PointRedlineTest     ← 含新增写越权三用例
[WARNING] Tests run: 28, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

（28 = Instance 8 + Point 9 + Gateway 5 + DataItem 3 + ClientScope 3 占位 skipped；新增点位写越权用例全绿。）

**补充验收 3：dev 冒烟复核 —— 通过**

启动：dev.ps1（8098，dev 档，修订轮 jar）；受限账号 smoke_perm_a_t0006（dev 冒烟配置：role 6 本轮补授 iot:point:edit(4056)/remove(4057) 菜单——dev 库操作不入库）；脚本 `backend/target/t0021rev2_smoke.py` 原始输出：

```
=== S1) 双账号登录 === S1a admin -> HTTP 200 code 200 | 操作成功  [期望200] OK
S1b restricted -> HTTP 200 code 200 | 操作成功  [期望200] OK
S1c 定位: 不可见点位 id=9002（dept 2）/ 可见点位 id=9001（dept 4）
=== S2) 受限用户直改不可见点位 -> 403（评审 I-2 编辑防线） ===
S2a -> HTTP 200 code 403 | 无权操作该点位（数据权限不命中）  [期望403] OK
S2a 断言: 403 且文案不泄露点位编码 OK
=== S3) 受限用户启停不可见点位 -> 403（批量启停防线，整批 fail-closed） ===
S3a -> HTTP 200 code 403 | 无权操作该点位（数据权限不命中）  [期望403] OK
S3a 断言: 403 OK
=== S4) 受限用户删除不可见点位 -> 403（删除防线先于绑定校验） ===
S4a -> HTTP 200 code 403 | 无权操作该点位（数据权限不命中）  [期望403] OK
S4a 断言: 403 OK
S4b 断言: 不可见点位三路径后原样未被改动（admin 可读） OK
=== S5) 受限用户编辑可见点位 -> 200（防线不误伤） ===
S5a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5b 恢复点名 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S5 断言: 可见点位编辑落库成功并已恢复 OK
=== S6) 建档携带 remark -> 回读一致（评审 I-1） ===
S6a 建档 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S6b 断言: remark 建档落库回读一致 OK
S6c 清理 -> HTTP 200 code 200 | 操作成功  [期望200] OK
SMOKE DONE
```

用毕即停：dev.ps1 stop → PID 38080 已停 → `netstat :8098 LISTENING` 复核无输出。

### 8.5 deferred（Minor 8 条不动）

Minor 8 条按评审裁决不动、记 deferred（清单见 reports/TASK-0021-review.md Minor 段；如后续需要处理，随修订轮或后续任务批评估议）。

### 8.6 修订轮结论

两项必修（I-1 remark 白名单 / I-2 三写路径数据权限防线）按裁决口径落地闭环，三项补充验收全部通过。累计三个 commit：9a162a8（主批）→ a61fd24（0018 遗留 guard 接线）→ a961fd1（评审 I-1/I-2）。TASK 状态维持「进行中」，未回写任务详情。
