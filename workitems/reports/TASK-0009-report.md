# TASK-0009 实施报告（字典批量导入逐行回执 + 横切顺带项 maxLimit）

- 执行者：implementer（subagent）
- BASE：`509a82b`（主仓 HEAD，派发时回填）；提交：`e30de0a`（main，注明 TASK-0009 含横切顺带项，11 files / +1285 −6）
- 总体结论：**5 条验收全部通过**（验收 5 的 BASE..HEAD diff 中含 4 个 ingest/ 文件，属并行 Go 道提交 `2fc80ba`（TASK-0074）在 BASE 之后落库所致，本任务改动严格限于声明范围，证据见验收 5）。

---

## 1. 改动文件清单（提交 e30de0a，11 个，全部落在 brief 声明范围）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/.../iot/entity/dto/DictImportRow.java` | 新增 | 导入行 DTO（Excel 单行 → 服务层原始输入）：仅五个业务列 + Excel 物理行号（表头=1，数据自 2 起）。DTO 结构性不含 status/deleted/id/审计字段——导入侧 mass assignment 无注入通道，落库实体仍由服务层白名单构建 |
| `backend/ot-admin/.../iot/entity/dto/DictImportRowResult.java` | 新增 | 逐行回执行：rowIndex / 原始 dataCode / success / reasonCode / message；失败原因码五常量 MISSING_REQUIRED / TOO_LONG / INVALID_DATA_TYPE / INVALID_CATEGORY / DUPLICATE_CODE（批内互撞与库内已有同报 DUPLICATE_CODE，4002 语义在批量口径内逐行标注） |
| `backend/ot-admin/.../iot/entity/dto/DictImportReceipt.java` | 新增 | 回执结构：totalCount / successCount / failCount / **persisted**（整批是否落库：true=successCount 行已入库；false=整批拒绝任何行未入库，successCount 仅表示「校验通过行数」）/ rows |
| `backend/ot-admin/.../iot/entity/dto/DictImportRejectException.java` | 新增 | 整批拒绝异常（extends BusinessException，code 预置 PARAM_ERROR 400）携带完整回执——BusinessException 本体不携带数据体，控制器捕获后装配 `Result.data` |
| `backend/ot-admin/.../iot/service/IotDataDictService.java` | 修改 | +`importDicts(List<DictImportRow>)` 接口方法与 javadoc 契约 |
| `backend/ot-admin/.../iot/service/impl/IotDataDictServiceImpl.java` | 修改 | `importDicts` 实现：① 批级拒绝（无数据行/超 5000 行 → PARAM_ERROR，无回执）；② 模板说明行（首列「说明：」前缀）与全空行裁剪；③ 逐行校验五分类（必填→超长→类型→值域），回执行号=Excel 物理行号；④ 批内互撞先于落库判定（首现行放行、后续行 DUPLICATE_CODE 指向首现行号）；⑤ 库内已有一次 IN 预检（活跃行，@TableLogic 过滤已删行、已禁用行仍占码，租户条件由插件追加——与单接口 insertDict 预检同口径），避免 5000 次逐行 count；⑥ 任一行失败 → `DictImportRejectException` 整批拒绝（事务内未执行任何 INSERT，rollbackFor 兜底）；⑦ 全部通过 → 白名单实体逐行落库（status=1/deleted=0/tenantId 会话覆盖），DuplicateKeyException 并发兜底 → PARAM_ERROR 整批拒绝（批量口径不用 4002，裁决 2）；写方法 `@Transactional(rollbackFor = Exception.class)` |
| `backend/ot-admin/.../iot/controller/IotDataDictController.java` | 修改 | +三接口（共用 `iot:dict:import`，sys_menu 4023）：`POST /iot/dict/import`（multipart，@OperateLog IMPORT，@RepeatSubmit 3000ms；整批拒绝 catch → body code=400 + data=完整回执清单）；`GET /iot/dict/import/template`（表头五列 + 示例行 + 说明行，值域口径写入说明行文案）；`POST /iot/dict/import/error-report`（前端回传回执 → 失败行渲染 xlsx，无状态设计，无失败行返回仅表头清单）；Excel 解析照 SysUserController 范式（Hutool ExcelReader read(1)，单元格 trim 后按位映射五列，行号保持物理对齐） |
| `backend/ot-core/.../config/MybatisPlusConfig.java` | 修改 | **横切顺带项（裁决 5）**：`PaginationInnerInterceptor` 加 `setMaxLimit(500L)`（详细设计 §3.4「pageSize 上限 500」），超限请求截断到 500 |
| `backend/ot-admin/src/test/.../iot/service/impl/IotDataDictImportServiceTest.java` | 新增 | 16 例（见 §2 验收 1） |
| `backend/ot-admin/src/test/.../iot/controller/IotDataDictImportControllerTest.java` | 新增 | 9 例（见 §2 验收 1），含真实 Hutool 组件内存构造 .xlsx 的 multipart 解析断言与模板/错误清单回读断言 |
| `resources/sql/upgrade/20261122_iot_dict_import.sql` | 新增 | 菜单种子 4023 一行（字典导入，menu_type=3，parent_id=4008，perms=iot:dict:import，sort=6 续号，幂等 INSERT IGNORE）+ sys_role_menu 授 role_id=1；脚本头含用途/接口对应（三接口共用一权限点）/号段占用说明（本脚本占 4023）/幂等口径，语句区无 `--` 注释行（sql-exec.sh 纪律） |

新增测试合计 **25 例**（服务 16 + 控制器 9）。TDD 先红后绿：红 1=两测试类编译失败（DTO/服务方法不存在，实测）；红 2=SaPermissionRegistryConsistencyTest 失败（新注解未登记 sys_menu SQL——守卫按设计工作，SQL 脚本补登记后转绿）。

## 2. 验收命令原始输出

### 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

```
mvn install -pl ot-core -DskipTests:
[INFO] BUILD SUCCESS
===INSTALL-DONE===

mvn test -pl ot-admin（Results 段）:
[INFO] Results:
[INFO]
[INFO] Tests run: 1229, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
[INFO] Total time:  01:17 min
[INFO] Finished at: 2026-11-22T01:38:48+08:00
```

逐类（同日志 grep 证据）：

```
[INFO] Tests run: 9,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.controller.IotDataDictImportControllerTest
[INFO] Tests run: 16, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.service.impl.IotDataDictImportServiceTest
[INFO] Tests run: 6,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.system.config.SaPermissionRegistryConsistencyTest
```

判定：**通过**。1229 = 基线 1204（BASE=`509a82b` 派发态）+ 新增 25（16+9；未动任何既有测试文件，git diff 可核）。brief 所写「基线 1200」为约数，实为 1204（证据性更正，见偏离 1）。SaPermissionRegistryConsistencyTest 6 例全绿 = iot:dict:import 注解 ⊆ 本任务新脚本 4023 行登记（守卫闭环）。

### 验收 2：升级脚本二次幂等 + 4023 核对

第 1 次：`bash backend/scripts/sql-exec.sh resources/sql/upgrade/20261122_iot_dict_import.sql`

```
OK(0)   ← SET NAMES utf8mb4
OK(1)   ← INSERT IGNORE sys_menu 4023（首插）
OK(1)   ← INSERT IGNORE sys_role_menu (1,4023)（首插）
DONE
```

第 2 次（幂等）：

```
OK(0)
OK(0)
OK(0)   ← 均已存在 → 0 行
DONE    ← exit 0，无 SQLERROR
```

核对（原始输出）：

```
SELECT id, menu_name, menu_type, perms FROM sys_menu WHERE id = 4023;
id	menu_name	menu_type	perms
4023	字典导入	3	iot:dict:import

SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id = 4023;
role_id	menu_id
1	4023
```

判定：**通过**（两次 DONE 无 SQLERROR；4023 行 perms=iot:dict:import、menu_type=3 挂 4008 页下；(1,4023) 超管授权在位）。

### 验收 3：dev 冒烟（启动：`cd backend && mvn spring-boot:run -pl ot-admin`，日志 backend/target/task0009_dev.log）

启动前端口探测：8098 无 LISTENING（probe exit=1）。就绪：`READY after ~12s (captchaImage HTTP 200)`。启动段：`Started OtAdminApplication in 15.983 seconds (process running for 16.733)`；启动日志 grep ERROR 计数 = **0**。

```
=== S1) admin 登录（dev /auth/login/test，tenantCode 缺省 -> default）
{"code":200,"data":{"nickname":"管理员","token":"007f69ce-62be-481a-ae35-75e3442e8a51","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795282899001}

=== S2) GET /iot/dict/import/template → HTTP_STATUS:200 CONTENT_TYPE:application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=utf-8 SIZE:4037
Content-Disposition: attachment;filename=%E6%95%B0%E6%8D%AE%E5%AD%97%E5%85%B8%E5%AF%BC%E5%85%A5%E6%A8%A1%E6%9D%BF.xlsx
文件头: 50 4b 03 04（xlsx/zip magic）
（模板回读，jshell+Hutool：rows=3
  r1: [0]data_code [1]data_name [2]category [3]data_type [4]default_unit
  r2: [0]ELEC_CONS [1]用电量（示例行，导入前请删除或替换） [2]ELEC [3]DOUBLE [4]kWh
  r3: [0]说明：data_code/data_na...(251) ← 说明行含值域口径，导入时自动跳过）

=== S3) POST /iot/dict/import（5 行拒绝批：1 成功/1 批内重复/1 类型非法/1 库内已有/1 超长，jshell+Hutool 构造 task0009_reject.xlsx）
（HTTP 信封 200）
{"code":400,"data":{"failCount":4,"persisted":false,"rows":[
 {"dataCode":"SMOKE_OK_001","message":"校验通过","reasonCode":null,"rowIndex":2,"success":true},
 {"dataCode":"SMOKE_OK_001","message":"批内重复（与第 2 行撞码）","reasonCode":"DUPLICATE_CODE","rowIndex":3,"success":false},
 {"dataCode":"SMOKE_BADTYPE","message":"数据类型非法: TEXT（仅允许 DOUBLE/INT/BOOL/ENUM）","reasonCode":"INVALID_DATA_TYPE","rowIndex":4,"success":false},
 {"dataCode":"ELEC_CONS","message":"库内已存在同编码条目（含已禁用，编码不复用）","reasonCode":"DUPLICATE_CODE","rowIndex":5,"success":false},
 {"dataCode":"AAAA…(65)","message":"字段超长: data_code(65>64)","reasonCode":"TOO_LONG","rowIndex":6,"success":false}],
 "successCount":1,"totalCount":5},"message":"导入校验未通过，整批未入库（校验通过 1 行 / 失败 4 行），请按回执修正后重新导入"}
→ body code=400（PARAM_ERROR）+ data 逐行回执；五分类各就位；4002 未出现在批量口径（裁决 2）

=== S4) 整批拒绝 DB 层核对（不半落库）
SELECT COUNT(*) AS rows_before FROM iot_data_dict WHERE tenant_id = 1;  → 2
SELECT id, data_code, status FROM iot_data_dict WHERE tenant_id = 1 ORDER BY id;
1	ELEC_CONS	0
3	WATER_CONS	1
（TASK-0008 留痕两行外零新增——整批原子）

=== S5) POST /iot/dict/import（修正批 task0009_fixed.xlsx）
{"code":200,"data":{"failCount":0,"persisted":true,"rows":[…五行全部 {"success":true,"message":"校验通过"} …],"successCount":5,"totalCount":5},"message":"操作成功"}
HTTP_STATUS:200

=== S6) GET /iot/dict/list?pageSize=50 确认落库 → HTTP_STATUS:200, total=7
新落 5 行（id 4-8）：SMOKE_OK_001/冒烟成功编码/ELEC/DOUBLE、SMOKE_NEW_002/修正新编码/WATER/INT、
SMOKE_BADTYPE/类型已修正/ELEC/DOUBLE、SMOKE_NEW_003/库内改码冒烟/GAS/BOOL、SMOKE_SHORT/超长已修正/WATER/INT
（createBy=1, status=1, deleted=0；另含 TASK-0008 留痕 2 行）

=== S6b) 重导同一修正批（成功行不重复创建，TASK 验收标准）
{"code":400,"data":{"failCount":5,"persisted":false,"rows":[…五行全部 reasonCode:"DUPLICATE_CODE","message":"库内已存在同编码条目（含已禁用，编码不复用）" …],"successCount":0,"totalCount":5},…}
SELECT COUNT(*) FROM iot_data_dict WHERE tenant_id=1 → 7（无重复创建）

=== S7) POST /iot/dict/import/error-report（S3 回执 data 体回传）→ HTTP_STATUS:200 xlsx（50 4b 03 04）SIZE:4007
Content-Disposition: attachment;filename=%E5%AF%BC%E5%85%A5%E9%94%99%E8%AF%AF%E8%A1%8C%E6%B8%85%E5%8D%95.xlsx
（清单回读，jshell+Hutool：rows=5 = 表头+4 失败行，成功行不进清单
  [行号][数据编码][原因码][说明]
  [3][SMOKE_OK_001][DUPLICATE_CODE][批内重复（与第 2 行撞码）]
  [4][SMOKE_BADTYPE][INVALID_DATA_TYPE][数据类型非法: TEXT（仅允许 DOUBLE/INT/BOOL/ENUM）]
  [5][ELEC_CONS][DUPLICATE_CODE][库内已存在同编码条目（含已禁用，编码不复用）]
  [6][AAAA…(65)][TOO_LONG][字段超长: data_code(65>64)]）

=== S8) GET /iot/dict/list?pageSize=1000（maxLimit 生效）
code:200 | size:500 | total:7 | pages:1 | records:7 → MAXLIMIT-OK: size 500 <= 500
=== S8b) 对照：pageSize=30 → size:30（未截断，正常回显）
=== S8d) 横切对照：GET /iot/tenant/list?pageSize=1000 → code:200 size:500 total:9（同一拦截器全局生效）
=== S8c) DB 复核 rows_now=7（重导未重复创建）

=== S9) 停止 dev 实例
taskkill /PID 13688 /T /F → 成功；8098 复核：无 LISTENING（probe exit=1）；无残留 java/mvn 进程
```

判定：**通过**（brief 冒烟清单 8 步全覆盖：登录/模板下载头/5 行拒绝批回执五分类/修正重导 200+成功计数/list 落库行数与内容/错误清单 200/pageSize=1000→size≤500；另加 S4 DB 不半落库、S6b 重导不重复创建、S8d 横切对照三步佐证）。

### 验收 4：prod 档真实启动回归

打包：`cd backend && mvn clean package -pl ot-admin -DskipTests` → BUILD SUCCESS，`ot-admin/target/ot-admin-1.0.0.jar`（132,622,241 字节；-DskipTests 因全量测试已在验收 1 同一代码态覆盖，TASK-0008 先例）。

启动（沿 TASK-0008 已确认命令 + prod fail-fast 必需三密钥环境变量，取 dev 默认值非新密钥）：

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
  REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
  java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

```
启动前端口探测：8098 无 LISTENING
READY after ~9s（captchaImage HTTP 200）
2026-11-22T01:57:09.516+08:00  INFO 34924 --- [ot-framework] [main] n.otlab.admin.OtAdminApplication : Started OtAdminApplication in 18.096 seconds (process running for 18.702)
启动段 ERROR 计数（Started 之前）= 0（就绪即采：grep -c ERROR → 0）
```

```
=== P1) prod POST /auth/login（真实口：验证码经 Redis 键 captcha:code:task0009a 预置 JSON 字符串 "AB12"（jshell+jedis 写入并回读验证），
        密码 admin123 经 hutool SM4-ECB 加密 hex 再 base64（jshell 本地往返解密验证 ROUNDTRIP-OK=admin123））
{"code":200,"data":{"nickname":"管理员","token":"5a3e18cd-e6b0-49a4-88b1-02c03461706f","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795283842804}

=== P2) prod GET /iot/dict/list（带 token）→ HTTP_STATUS:200
code:200 | total:7 | size:10，records 含 SMOKE_SHORT/SMOKE_NEW_003/SMOKE_BADTYPE…（default 租户隔离数据完整）

=== P3) prod GET /iot/dict/import/template（新权限点 iot:dict:import prod 鉴权链路）→ HTTP_STATUS:200 xlsx（50 4b 03 04）SIZE:4037

=== P4) prod /auth/login/test 拒绝（dev-only 防线回归）
{"code":500,"data":null,"message":"测试登录接口仅限开发环境使用","timestamp":1795283851677}
```

ERROR 说明（诚实口径）：启动段（Started 之前）**0 ERROR**（就绪即采，原始证据在上）；P4 步骤之后日志出现 2 条 ERROR（01:57:31，均为该探测步自身触发、启动后、与本任务改动无关的既有行为）：
1. `业务异常：auth.testLoginDevOnly`——GlobalExceptionHandler 对业务异常的 ERROR 级日志（P4 故意触发的 dev-only 拒绝，TASK-0008 冒烟同款步骤）；
2. `操作日志写入失败 … Field 'user_id' doesn't have a default value`——既有缺陷：失败登录（未认证态）触发 @OperateLog 写 `sys_operate_log`，user_id 为 null 而列为 NOT NULL → 写入失败。与本任务改动无关（本任务 diff 不触 operate log/login 域），记遗留问题 2。

判定：**通过**（启动段 0 ERROR + 真实登录 200 + list 200 + 模板下载 200 经新权限点 + dev-only 防线拒绝）。prod 实例用毕即停（taskkill /PID 34924 /T /F），8098 复核 0 监听。

### 验收 5：改动范围核对

命令：`git diff --name-only 509a82b..HEAD`（HEAD=e30de0a）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/controller/IotDataDictController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/dto/DictImportReceipt.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/dto/DictImportRejectException.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/dto/DictImportRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/entity/dto/DictImportRowResult.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/IotDataDictService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/controller/IotDataDictImportControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/IotDataDictImportServiceTest.java
backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java
ingest/internal/pipeline/pipeline_contract_test.go          ← 并行 Go 道
ingest/internal/tdengine/tdengine.go                        ← 并行 Go 道
ingest/internal/tdengine/tdengine_contract_test.go          ← 并行 Go 道
ingest/internal/tdengine/tdengine_injective_contract_test.go ← 并行 Go 道
resources/sql/upgrade/20261122_iot_dict_import.sql
```

归因（原始输出）：

```
git log --oneline 509a82b..HEAD
e30de0a feat(iot): 字典批量导入（…）（TASK-0009，含横切顺带项）
2fc80ba TASK-0074: 子表名/租户库名转义映射定长化修复（单射性，BUG-0002/复评 N1）

git show e30de0a --name-only → 恰 11 文件（本表去 ingest 四行），全部落在 brief 声明内
git show 2fc80ba --name-only → 恰上述 4 个 ingest 文件（并行 Go 道任务，非本任务改动）
```

核对结论：**本任务改动（e30de0a）11 文件全部落在 brief 声明范围**——iot 域 9 个（dto 4 新增 + service 2 + controller 1 + 测试 2）、ot-core 1（MybatisPlusConfig 仅 setMaxLimit(500L) 一行改动）、SQL 1 新增；未碰 frontend/、ingest/、既有字典/租户实现、application*.yml、workitems 库其他文件。BASE..HEAD 因并行 Go 道提交 `2fc80ba` 落在中间而多出 4 个 ingest 文件（`git show` 归因可核）。**通过**。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 5000 行真实文件的 dev 冒烟 | 行数上限（5001 拒 / 5000 放行）已在服务单测边界值覆盖（testImport_OverLimitRejected / testImport_ExactlyAtLimitAccepted），真实 5000 行文件构造与传输对验收断言无增量信息；brief 冒烟清单未含此项 |
| 导入逐行回执的「必填缺失/值域非法」两类 dev 冒烟 | 冒烟按 brief 的 5 行配方执行（1 成功/1 批内重复/1 类型非法/1 库内已有/1 超长——即五分类结果的完整配方）；必填缺失与值域非法已在服务单测逐例覆盖（testImport_MissingRequired / testImport_InvalidCategory 等） |
| 4002 出现在批量导入任何路径 | 非未做而是裁决 2 的直接落实：整批拒绝统一 PARAM_ERROR(400)+data 回执，行内重复以 DUPLICATE_CODE 原因码逐行标注；单点 POST /iot/dict 的 4002 语义不受影响（TASK-0008 既有行为，回归全绿） |

## 4. 偏离计划之处

1. **基线数字更正（证据性）**：brief 验收 1 写「基线 1200 + 新增」，实测 BASE 派发态基线 1204（= 1229 − 新增 25；本任务未改任何既有测试文件）。判定与结论不受影响。
2. **任务中途环境事故与恢复**：01:44 前后 Windows 进程派生全面瞬时失败（0xC0000142，Git Bash/mvn/java 均受影响，约 3 分钟后自愈）；该事故同时导致本机 MySQL(3307)/Redis(6379)（phpStudy 面板托管的普通进程，非 Windows 服务，非 WSL docker——WSL Ubuntu 容器为 13+ 月前的陈旧退出态）失联。处置：以 phpStudy 既有二进制按其常规方式拉起——`mysqld.exe --defaults-file=D:/phpstudy_pro/Extensions/MySQL8.0.12/my.ini`（my.ini 端口本就是 3307）与 `redis-server.exe --port 6379 --bind 127.0.0.1`；数据完好（iot_data_dict 8 行含 TASK-0008 留痕、sql-exec 探针 +PONG），两个服务恢复后保持运行（即 brief 所述环境事实的原状）。未改任何仓库文件/配置。
3. **prod 首次启动尝试被工具超时回收**：bash `nohup java -jar &` + 等待循环超时被移入后台后，随调用终止被回收（日志 0 字节）；改为 PowerShell Start-Process 经 cmd 全脱链启动后成功（Started 18.096s）。属执行器机械问题，非应用问题。
4. **回执结构增加 `persisted` 字段（brief 字面未列）**：裁决 2 下整批拒绝时 successCount 行并未落库，仅以成败计数会误导用户；`persisted=false` 显式标记「任何行未入库」，200 路径恒 true。单测断言两路径。
5. **错误清单下载接口形态自定**：brief 只要求「回执可下载」，未定接口。实现为 `POST /iot/dict/import/error-report`（无状态：前端回传回执 JSON，服务端过滤失败行渲染 xlsx，不持久化）。**前端契约注意：请求体是回执本身（导入响应的 `data` 字段），不是整个 Result 信封**（冒烟中误传信封即得「请求参数格式错误」400，属预期）。
6. **模板示例行不自动跳过**：仅「说明：」前缀行（说明行）在导入时自动跳过（单测 testImport_NoteRowsAndBlankRowsSkipped 佐证行号物理对齐）；示例行为真实数据格式，若不删会被当数据行校验（与框架 SysUser 导入模板同口径），其处置指引写入示例行 data_name 与说明行文案（「示例行导入前请删除或替换」）。理由：示例行若也用魔法前缀跳过，用户改写残留前缀会静默造脏码，而字典编码只增不删。
7. **TDD 红转绿中两处测试侧修正**（实现零改动，见附 1 R2/R3）。
8. **验收命令形态**（沿前序先例）：prod 打包 -DskipTests（全量测试同代码态覆盖）；prod 启动补三密钥环境变量（fail-fast 必需，dev 默认值非新密钥）；验收 5 先 commit 再 diff（提交严格限 11 文件，任务前已存在的未跟踪 `.agents/` 未纳入）。

## 5. 遗留问题

1. **冒烟数据留痕**：default 租户新增 5 行 SMOKE_* 字典（id 4-8，仅增不删语义下不可物理清理）+ TASK-0008 留痕 2 行；Redis 验证码键 5 分钟自然过期、token 会话过期。重复验收可直接重跑（重导同文件整批 DUPLICATE_CODE 属预期行为）。
2. **既有缺陷（建议立项 BUG，非本任务范围）**：未认证态触发带 @OperateLog 的失败请求（如 prod /auth/login/test）时，`sys_operate_log` INSERT 缺 user_id 报 `Field 'user_id' doesn't have a default value`，异步写日志失败仅 ERROR 留痕不影响主流程（P4 实测复现）。与本任务改动无关，本报告如实记录供协调者归口。
3. **环境脆弱性提示（运维口径）**：本机 MySQL/Redis 由 phpStudy 面板托管为普通进程，无 Windows 服务自启/守护；一次系统级进程派生事故即可使其失联。建议后续以 docker compose 或 Windows 服务方式固化（影响所有 backend 任务的验收稳定性，非本任务代码问题）。
4. **行号对齐前提**：回执行号=Excel 物理行号，依赖 Hutool `read(1)` 对每物理行产出一条记录（中间空行以空行占位）——控制器测试以真实 Hutool 组件构造含中间空行的 .xlsx 实证对齐（testImport_ParsesExcelRows + 服务侧 testImport_NoteRowsAndBlankRowsSkipped 行号 3/5 断言）；表尾最后一个有值行之后的行不产出（无影响）。
5. **TASK-0010 联动提示**：TASK-0008 报告遗留 4 项仍有效；本任务新增可用语义——`IotDataDictService.importDicts` 供前端/后续批量场景复用；错误清单接口为无状态回传设计，前端需保存导入响应的 `data` 用于 POST 下载。

---

## 附 1：根因链（TDD 红→绿）

| # | 现象（红） | 根因 | 修复 | 为什么现在对了 |
| --- | --- | --- | --- | --- |
| R1 | 两个新测试类编译失败：`DictImportRow/DictImportRowResult/DictImportReceipt/DictImportRejectException` 找不到符号、`importDicts` 方法不存在 | TDD 红灯预期：实现未写 | 实现 4 个 DTO/异常 + 服务方法 + 3 接口 | 红→绿符合 TDD 流程，断言逻辑未放松 |
| R2 | `testImport_AnyFailureRejectsWholeBatch` 断言 successCount=1 实得 2 | **测试侧误算**：第 3 行 WATER_CONS 是批内首现，按「首现放行、后续撞码」口径本就校验通过——实现行为正确，期望值算错（把 5 行配方误当 4 失败） | 修正期望为 2（并补断言「首现行 reasonCode=null」） | 与 testImport_BatchInternalCollision 的首现/次现口径一致；整批原子断言（0 次 insert）不变 |
| R3 | `testTemplate_Download` 断言 `getContentLength()>0` 失败（实为 0） | MockHttpServletResponse 只有在显式 setContentLength 时才有该值；ExcelWriter 直接 flush 到响应输出流不设置 Content-Length（HTTP 层为 chunked）——实现正确，断言取错了字段 | 改用 `getContentAsByteArray().length` | 字节体真实存在于响应缓冲，download 正常（S2/P3 冒烟 SIZE 4037 佐证） |
| R4 | `SaPermissionRegistryConsistencyTest` 红：iot:dict:import 未登记 | 守卫测试按设计工作：新 @SaCheckPermission 注解必须 ⊆ resources/sql 的 sys_menu INSERT 登记集 | 新增 `20261122_iot_dict_import.sql` 登记 4023 行 | 守卫转绿=「权限标识与接口清单一一对应」闭环（验收 2 核对 4023 行在库） |
| R5 | dev 冒烟 S2 首次 GET 模板返回 401「未按照指定前缀提交 token，prefix=Bearer」 | sa-token 配置 token-name=Authorization 且要求 Bearer 前缀，冒烟脚本裸传 token | 统一 `Authorization: Bearer <token>` | 请求头口径问题，非接口缺陷；S2 起全部 200 |
| R6 | dev 冒烟 S7 首次错误清单下载得 400「请求参数格式错误」 | 误把整个 Result 信封当回执 POST（端点契约是回执 `data` 体） | 用 python 提取 `.data` 后重传 | S7-retry2 得 200 xlsx 且回读 4 失败行；契约已在偏离 5/遗留 5 写明 |
| R7 | 任务中途 sql-exec/Redis 全部 Connection refused；netstat 无 3307/6379 | 0xC0000142 进程派生事故连带 phpStudy 托管的 MySQL/Redis 普通进程失联（非本任务代码问题） | 按偏离 2 以 phpStudy 二进制原状拉起，数据完好 | 环境回到 brief 声明事实；后续验收全部在此恢复态上完成 |
| R8 | prod 首启日志 0 字节、无监听 | bash 后台 java 随超时调用被回收（执行器机械问题） | PowerShell Start-Process 全脱链重启 | Started 18.096s + 启动段 0 ERROR + P1-P4 全过 |

## 附 2：契约自查

- 未派 subagent；未把 TASK 置「已完成」（详情留协调者收尾）；未改 workitems 库其他文件（仅本报告）。
- 改动 11 文件全部在 brief 声明内（验收 5 归因可核）；frontend/、ingest/ 零接触；既有字典/租户实现零修改（纯增量方法）；application*.yml 未动；ResultCode 未动（批量路径 0 处新增 4002）。
- 全量 1229 例 0 失败 0 错误（验收 1 原始输出贴回）；5 条验收命令全部执行且原始输出贴回；TDD 先红后绿（附 1 R1/R4）。
- 整批原子性与逐行回执并存实证：S3 拒绝批 0 行落库（DB 计数 2 不变）+ 回执逐行五分类；S5 修正批 5 行全落库。
- 4002 语义口径照裁决 2：批量整批拒绝 body code=PARAM_ERROR(400)+data 回执；行级重复用 DUPLICATE_CODE 原因码；单点接口 4002 不受影响（既有测试回归全绿）。
- 批内互撞先于落库判定（内存判定，0 次 insert 即拒绝）；单批上限 5000 行边界双测。
- maxLimit 横切项照裁决 5：仅 MybatisPlusConfig 一处 `setMaxLimit(500L)`，与本任务同 commit（提交信息注明含横切项）；全量回归 + dev/prod 双实证（dict list 与 tenant list 的 pageSize=1000 均截断 size=500，pageSize=30 正常）。
- 事务与权限照裁决 6：`@Transactional(rollbackFor = Exception.class)`；菜单种子 4023 幂等 INSERT IGNORE 挂 4008 + (1,4023) 超管授权；守卫测试闭环。
- 8098 纪律照做：dev/prod 两实例启动前探测（均无监听）、用毕即停、复核 0 监听；无残留 java/mvn 进程。
- 环境事故（R7）如实报告：MySQL/Redis 为 phpStudy 托管普通进程，事故中失联，已按原方式恢复并保持运行，数据完好；未写入任何仓库配置（遗留 3 记运维建议）。

---

# 7. 修订轮（验收退回 §13：评审 I-1/I-2/I-3 修复与补充验收）

- 评审出处：`reports/TASK-0009-review.md`——Critical 0 / Important 3 / Minor 6；Important 3 项全部进修订轮修复，Minor 6 项按裁决不动。
- 修订轮提交：`9045f38`（main，注明 TASK-0009 修订轮，8 files / +762 −71）。

## 7.1 修复内容（为什么现在对了）

改动文件（严格限「字典导入域文件与测试」；maxLimit 与 4023 零改动）：

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/.../iot/service/support/DictImportExcelSaxParser.java` | 新增 | SAX 流式解析器（POI `XSSFReader` + `org.apache.poi.util.XMLHelper.newXMLReader()` 安全 XMLReader，XXE 防护口径不变，poi-ooxml 既有依赖零新增）。行号取自 `<row r="N">` 的 r 属性（1-based 物理行号）；逐单元格取值无任何 alias 填充（缺格=null）；逐行回调维护数据行计数，超过上限立即抛 `DictImportRowLimitException` 中止解析（RuntimeException 经 JAXP XMLReader.parse 原样透传，实测）；sharedStrings.xml 以 SAX 展平为纯 `List<String>`（避免 JAXB 元素树二次放大）；整数值单元格归一（1002.0→"1002"，与 DOM 口径一致）；空串单元格归一 null（hutool 对 null 写空 inlineStr，与「缺格」在导入语义等价）；仅首个 sheet（与原 getReader 默认口径一致） |
| `backend/ot-admin/.../iot/entity/dto/DictImportRowLimitException.java` | 新增 | 流式中止标记异常（携带中止时数据行数），控制器从异常链识别后映射 PARAM_ERROR(400) 整批拒绝 |
| `backend/ot-admin/.../iot/entity/dto/DictImportRow.java` | 修改 | ① `NOTE_ROW_PREFIX` 常量自 ServiceImpl 上移至此；② 新增 `isSkippable()` 实例判定（全空行或说明行）——控制器流式计数与服务层裁剪共用同一口径，杜绝漂移 |
| `backend/ot-admin/.../iot/service/IotDataDictService.java` | 修改 | `IMPORT_MAX_ROWS=5000` 常量上移至接口（控制器解析层与服务层共用） |
| `backend/ot-admin/.../iot/service/impl/IotDataDictServiceImpl.java` | 修改 | 删除私有 isSkippableRow/NOTE_ROW_PREFIX（改用 `DictImportRow.isSkippable()` 与接口常量）；服务层 5000 行防线保留（程序化调用方兜底）；导入校验/回执/落库逻辑零变化 |
| `backend/ot-admin/.../iot/controller/IotDataDictController.java` | 修改 | `parseImportRows` 由 Hutool DOM `read(1)` 改为 SAX 解析器调用 + `DictImportRowLimitException` → PARAM_ERROR(400)「单批导入上限 5000 行，当前 N 行」；TEMPLATE_NOTE 前缀与上限文案引用共享常量；javadoc 记录三项修复口径 |
| `backend/ot-admin/src/test/.../iot/controller/IotDataDictImportControllerTest.java` | 修改 | 名实不符的两测试改为真实含空行夹具（`testImport_ParsesExcelRows_PhysicalAlignmentWithEmptyRow` / `testImport_TrimsCells_KeepsPhysicalAlignmentWithGap`——header+数据+空行+数据，断言 rowIndex=物理行号）；新增说明行透传行号保真、首行缺格保持 null（I-2）、超 5000 行流式中止（I-3，断言 400 + 服务层零调用 + 打印耗时）共 12 例 |
| `backend/ot-admin/src/test/.../iot/service/support/DictImportExcelSaxParserTest.java` | 新增 | 7 例：POI XSSF（sharedStrings 编码路径）与 Hutool BigWriter（inlineStr 编码路径）双通道夹具——空行后物理行号 4 不前移、空行条目若透传必为 skippable、首数据行缺格保持 null（无 alias 'B'）、数值单元格归一 1001/1002、5001 行流式中止（中止点=5001 + 耗时打印）、恰 5000 行放行、说明行透传行号保真 |

**为什么现在对了**：① 行号不再由「返回序」推导而由 `<row r>` 属性保真携带——空行（无论 XML 缺席还是占位）都不再使后续行号前移，回执与错误清单指行与 Excel 一致；② SAX 无 aliasFirstLine 机制且空串归一 null——首数据行缺格与后续行同口径触发 MISSING_REQUIRED，"A"/"B" 列名字母无落库通道；③ 上限判定移入逐行流式回调——解析在第 5001 数据行当场中止，内存 O(共享字符串文本+单行)，50MB DOM 放大面消除。

## 7.2 关键技术决策：Hutool SAX 在本工程 classpath 上不可用（偏离评审字面口径的实证依据）

评审 I-3 处方「Hutool ExcelUtil.getBySax / Excel07SaxListener」在本工程依赖组合（hutool 5.8.25 + POI 5.2.5）下**静默失效**——jshell 探针（与工程依赖完全一致）实证：

```
== SAX probe（ExcelUtil.readBySax(InputStream,0,RowHandler)）→ 零回调、零输出
== A) Excel07SaxReader.read(File, int 0)      → done, no error（零回调）
== B) Excel07SaxReader.read(File, String rId1) → done, no error（零回调）
== C) ExcelUtil.readBySax(File, int 0)         → done, no error（零回调）
```

根因（字节码考古）：hutool `SheetDataSaxHandler.startElement` 以 **qname** 匹配（`"sheetData".equals(qname)` → `ElementName.of(qname)`），而其 `ExcelSaxUtil.readFrom` 底层用 POI `XMLHelper.newXMLReader()`——namespace-aware 安全口径下 default-namespace 元素 qname 为空串，行元素永不匹配，`isInSheetData` 永不置位 → 全部元素静默跳过且不报错。升级 hutool 超出本任务改动范围（「改动限字典导入域文件与测试」），故按评审 I-3 的**意图**（SAX 流式、逐行回调计数、超限即止、内存有界）改用 POI 原生 XSSFReader + 同一 POI 安全 XMLReader 自实现行级回调。解析行为等价性由双通道夹具（POI sharedStrings 编码 + Hutool inlineStr 编码）7+12 例单测覆盖。

## 7.3 TDD 红转绿记录

红 1（实现前，编译失败）：

```
[ERROR] 找不到符号 类 DictImportRowLimitException / 类 DictImportExcelSaxParser（两新测试类编译失败）
[INFO] BUILD FAILURE
```

红 2（解析器初版，6 断言红——两处真实缺陷在测试驱动下显形）：

```
DictImportExcelSaxParserTest.testParse_PhysicalRowNumbersWithEmptyRowGap:101 只有两个数据行（空行不算数据行） ==> expected: <2> but was: <0>
DictImportExcelSaxParserTest.testParse_EmptyRowEntryIfPresentIsSkippable:122 expected: <2> but was: <0>
DictImportExcelSaxParserTest.testParse_FirstRowMissingCellsStayNull:143 ==> expected: <null> but was: <>
IotDataDictImportControllerTest.testImport_ParsesExcelRows_PhysicalAlignmentWithEmptyRow:148 ==> expected: <2> but was: <3>
IotDataDictImportControllerTest.testImport_TrimsCells_KeepsPhysicalAlignmentWithGap:235 空行后行号保真 ==> expected: <4> but was: <3>
IotDataDictImportControllerTest.testImport_FirstRowMissingCells_StayNull:207 ==> expected: <null> but was: <>
```

根因与修复（为什么现在对了）：
1. POI 夹具 0 数据行——sharedStrings 关系挂在 **workbook 部件**而非包根（我初版从包根取关系静默取空清单 → 所有 t="s" 单元格 null → 全行 skippable）。修复：包根 --CORE_DOCUMENT--> workbook 部件 --sharedStrings--> sharedStrings 部件的关系链解析；测试断言（POI 路径取值正确）即回归防线。
2. hutool 夹具缺格得到 "" 而非 null——hutool 对 null 值写空 inlineStr `<t></t>`，空串行既非全空行（isSkippable 失效 → 行计数虚增 3）也绕过必填语义。修复：解析层空串归一 null（「显式空串」与「缺格」在五列均无空串合法值的导入语义下等价）。

绿（修复后）：

```
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.controller.IotDataDictImportControllerTest
[INFO] Tests run: 7,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.service.support.DictImportExcelSaxParserTest
[INFO] Tests run: 19, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

关联回归（既有字典域 + 守卫）：52 例全绿（IotDataDictImportServiceTest 16 / IotDataDictServiceImplTest 17 / IotDataDictControllerTest 13 / SaPermissionRegistryConsistencyTest 6）。

## 7.4 补充验收 1：全量复跑

`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`：

```
mvn install -pl ot-core -DskipTests → BUILD SUCCESS
mvn test -pl ot-admin（Results 段）:
[INFO] Tests run: 1243, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  01:08 min
[INFO] Finished at: 2026-11-22T02:38:41+08:00
```

数学闭环：1243 = 1229（首轮）+ 3（控制器类净增：+3 新增/重命名改造）+ 7（解析器新测试类）+ 4（`OperateLogAspectTest`，系并行 Java 道提交 `9754e0e`（TASK-0075，sys_operate_log 身份三列可空化）落在两轮之间，与本任务无关、本任务零改动）。逐类 diff 证据：`IotDataDictImportControllerTest 9→12`、`+DictImportExcelSaxParserTest 7`、`+OperateLogAspectTest 4`（git show 9754e0e 归因可核）。附注：TASK-0075 恰好修复了本报告首轮遗留 2（operate-log user_id NOT NULL）——遗留 2 已由并行道闭环，后续 prod 冒烟不再产生该 ERROR。

## 7.5 补充验收 2：dev 冒烟复核三步（启动前探测 8098 无监听；`Started OtAdminApplication in 15.968 seconds`；启动日志 ERROR 计数=0；admin 登录 200）

```
=== R1) 含空行 5 行文件（r1 表头 / r2 数据 / r3 空行 / r4 类型非法 / r5 数据）→ 回执行号=物理行号
code: 400 | message: 导入校验未通过，整批未入库（校验通过 2 行 / 失败 1 行）…
  rowIndex=2 code=REV_G1      success=True  reason=None
  rowIndex=4 code=REV_BADTYPE success=False reason=INVALID_DATA_TYPE
  rowIndex=5 code=REV_G3      success=True  reason=None
persisted: False | total: 3
PHYSICAL-ALIGNMENT-OK（断言 [2,4,5] 逐字通过——修复前该文件回执行号为 [2,3,4]，空行后整体前移）

=== R2) 首行缺名称文件（r2=首数据行缺 data_name + r3 正常行）→ MISSING_REQUIRED 且 0 行落库
code: 400 | message: 导入校验未通过，整批未入库（校验通过 1 行 / 失败 1 行）…
  rowIndex=2 code=REV_MISS_CODE success=False reason=MISSING_REQUIRED msg=必填字段缺失: data_name
  rowIndex=3 code=REV_OK_1      success=True  reason=None
persisted: False
DB 复核：iot_data_dict tenant_id=1 仍 7 行（0 行落库）；data_code IN ('REV_MISS_CODE','B','A') → 0 行（垃圾码无落库）
FIRST-ROW-MISSING-REQUIRED-OK

=== R3) 超 5000 行文件（5001 数据行，114,988 字节）→ 快速整批拒绝
CURL_TIME_TOTAL:0.144s SIZE_UPLOAD:115213
{"code":400,"data":null,"message":"单批导入上限 5000 行，当前 5001 行","timestamp":1795286457900}
DB 复核：data_code LIKE 'REV_BIG%' → 0 行
（上传+解析中止全程 0.144s——全量 DOM 解析完成后再计数的旧口径下，该文件会先完整展开 5000+ 行再被服务层拒绝）
```

dev 实例用毕即停（taskkill /PID 25876 /T /F），8098 复核 0 监听。

## 7.6 遗留 4 更正（如实声明）

首轮报告遗留 4 声称「控制器测试以真实 Hutool 组件构造含中间空行的 .xlsx 实证对齐」——**不成立**：首轮 `testImport_ParsesExcelRows` 与 `testImport_TrimsCellsAndKeepsBlankRows` 两个夹具均**不含空行**，首轮解析为 Hutool DOM `read(1)`（ignoreEmptyRow=true 默认空行整行跳过），含空行文件中后续行回执行号整体前移（评审 I-1 实证）。修订轮已按裁决更正：解析改为 SAX 物理行号保真，两个测试改造为真实含空行夹具并断言 rowIndex=物理行号（7.3 红→绿与 7.5 R1 为修复前后行为对照）。首轮遗留 4 的其余内容（表尾行不产出，无影响）维持有效。

## 7.7 修订轮契约自查

- 改动 8 文件全部落在「字典导入域文件与测试」（controller / impl / DTO / 服务接口 / 新增解析器与两测试类）；MybatisPlusConfig、20261122_iot_dict_import.sql、其余 DTO（Receipt/RowResult/RejectException）、frontend/、ingest/ 零改动。
- Minor 6 条按裁决未动；maxLimit(500L) 与 4023 权限行原样（全量 1243 例含守卫 6 例回归为证）。
- 一个 commit（9045f38）注明 TASK-0009 修订轮；报告仅追加本节（含遗留 4 更正），不改首轮记录。
- 自起 dev 实例 1 次，用毕即停，8098 复核 0 监听；未派 subagent；未把 TASK 置「已完成」。
