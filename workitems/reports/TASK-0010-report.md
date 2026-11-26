# TASK-0010 实施报告（模板数据项字典校验联动——公共校验器先行）

- 执行者：implementer（subagent，no-subagents 契约遵守——未再派生）
- BASE：`9045f38`（主仓 HEAD，派发时声明）；提交：`9a4c6ef`（main，注明 TASK-0010，4 files / +437 −0）
- 总体结论：**2 条验收全部通过**（全量回归 1260 = 基线 1243 + 新增 17，0 失败 0 错误；改动范围 4 文件全落声明内）。

---

## 1. 改动文件清单（4 个，全部落在 brief 声明范围）

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/DictCodeStatus.java` | 新增 | 三态枚举 `OK / NOT_REGISTERED / DISABLED`（brief 裁决 1 状态口径）；文案常量 `MSG_NOT_REGISTERED = "数据编码未在字典登记: "` 与 `MSG_DISABLED = "数据编码已停用: "`（任务正文硬要求的「未登记/已停用」文案区分，供 0011 报错直接复用，格式在本枚举唯一维护防漂移）+ `messageFor(code)` 拼完整文案辅助方法；javadoc 注明 deleted 行按未登记口径、异常与 4002 映射归调用方 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/IotDataDictService.java` | 修改（纯增量） | 新增接口方法 `Map<String, DictCodeStatus> validateForTemplateRef(Long tenantId, Collection<String> dataCodes)`（brief 裁决 1 指定形态逐字采纳）；javadoc 载明纯查询语义 / 一次 IN 预检 / 显式租户入参双保险 / fail-closed 边界；既有四方法零改动 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java` | 修改（纯增量） | 实现校验器：null/空集合→空 Map 不查库；tenantId null 或批内 null/空串编码 fail-closed 判 NOT_REGISTERED（不进 IN 条件、null 租户不查库）；去重（LinkedHashSet）后**一次 IN 预检**（`select(data_code, status).eq(tenant_id, 入参).in(data_code, codes)`，不逐个 count）；uk_tenant_data_code 保证逐码至多一行，status=1→OK、status=0→DISABLED、查无→NOT_REGISTERED（deleted 行被 @TableLogic 运行时过滤即此口径）；status null 防御性兜底 DISABLED（理论不可达）；返回 LinkedHashMap 保序（调用方可按模板数据项顺序取首个失败项报错）。既有方法零改动（类 javadoc 仅追加 TASK-0010 段落） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImplValidateForTemplateRefTest.java` | 新增 | 17 例独立测试类（standalone Mockito 口径与既有 `IotDataDictServiceImplTest` 同款）：空集合/null 集合返回空不查库、三态各断言（OK/NOT_REGISTERED/DISABLED）、deleted 行=NOT_REGISTERED、批内混合（1 OK + 1 NOT_REGISTERED + 1 DISABLED）+ 恰一次 selectList（单次 IN 佐证）、wrapper 显式携带 tenant_id=入参值 + data_code IN、跨租户编码不可见（显式条件口径）、null/空串编码 fail-closed、tenantId null fail-closed 不查库、文案常量逐字断言 + messageFor 拼接 + 三态枚举存在性、重复入参去重、status null 兜底 DISABLED、纯查询不写库（never insert/update/delete） |

新增测试 **17 例**，TDD 先红（编译失败：`DictCodeStatus` 找不到符号，实测输出见附 1）后绿。

## 2. 验收命令原始输出

### 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

基线（BASE=`9045f38` 改动前同命令，基线证据）：

```
[INFO] Tests run: 1243, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  50.575 s
[INFO] Finished at: 2026-11-22T02:53:11+08:00
```

改动后（验收本体，同命令）：

```
mvn install -pl ot-core -DskipTests:
[INFO] BUILD SUCCESS
[INFO] Total time:  4.373 s
[INFO] Finished at: 2026-11-22T02:57:03+08:00

mvn test -pl ot-admin（Results 段）:
[INFO] Results:
[INFO]
[INFO] Tests run: 1260, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] Total time:  51.044 s
[INFO] Finished at: 2026-11-22T02:57:57+08:00
```

判定：**通过**。1260 = 基线 1243 + 新增 17（数学闭环；brief 预告基线 1243 与实测逐字一致）。全绿 0 失败 0 错误——既有 1243 例无一被接口纯增量改动波及（IotDataDictControllerTest / IotDataDictServiceImplTest 等原量原样通过）。commit 后于 HEAD（`9a4c6ef`）复跑一次同结果（`Tests run: 1260, Failures: 0, Errors: 0, Skipped: 0 / BUILD SUCCESS`，2026-11-22T02:58:57+08:00）。注：输出中 wechat 域两行 ERROR 为既有测试故意注入的失败场景日志（WxMpTemplateSendServiceImplTest 内部模拟），非测试失败。

### 验收 2：改动范围核对 `git diff --name-only 9045f38..HEAD`

（沿前序任务先例：验收前先 commit `9a4c6ef`——brief 契约允许 git commit main 注明 TASK-0010；工作区改动仅 4 文件 + 任务前已存在的未跟踪 `.agents/`（未纳入提交），再比较。）

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/DictCodeStatus.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/IotDataDictService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImplValidateForTemplateRefTest.java
```

核对结论：4 个文件全部落在声明内——`backend/ot-admin/.../module/iot/**` 主代码 3 个（校验器落位于既有 iot 包 service/impl，与实体/服务同包组织）+ `backend/ot-admin/src/test/.../module/iot/**` 新增测试 1 个。未碰 frontend/、ingest/、模板域（未新建任何文件）、既有字典 CRUD 方法体（两处「修改」均为纯增量：接口 +1 方法、实现 +1 方法与 javadoc 追加，`git diff --stat` = +437 −0）、ResultCode / application*.yml / resources/sql / workitems 库其他文件。**通过**。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| HTTP 端点 / 菜单种子 / 冒烟 | brief 裁决 5 明文：本任务不接线，校验执行点归 TASK-0011 模板保存流程；验收=单测+全量回归（已做） |
| 模板域任何文件 | brief 改动范围明文：模板域尚不存在，不许建；0011 的调用点在其模板保存流程接线 |
| 4002 异常映射与文案抛错 | brief 裁决 2 明文：校验器只返回状态不抛异常，异常与 4002 映射由调用方（0011）决定——文案已备好（常量 + messageFor），0011 取状态直接拼用 |
| 运行时会话态「插件注入与显式条件重复」的实测观察 | standalone 单测无多租户插件（无 Sa-Token 会话），本任务无 HTTP 端点无冒烟，运行时观察不可达；语义分析：插件对 iot_ 白名单表追加 `tenant_id=<会话租户>`，与 wrapper 显式 `tenant_id=<入参>` 同值 AND，重复无害；0011 调用点租户上下文已定、两值恒等——留 0011 联测时一并以实测确认（见遗留 1） |

## 4. 偏离计划之处

1. **无计划偏离**。接口形态、三态语义、文案、批量 IN、纯查询口径、测试口径、验收命令均按 brief 逐条落地；基线数字（1243）与 brief 预告逐字一致，无需更正。以下为 brief 未细化处的自定边界（均 fail-closed 方向，测试锁定）：
   - **fail-closed 边界补充**（brief 口径 4 未列，自加 3 例锁定）：tenantId=null → 全判 NOT_REGISTERED 且不查库；批内 null/空串编码 → NOT_REGISTERED 且不进 IN 条件；库行 status=null → DISABLED（DDL `status tinyint DEFAULT '1'` + 服务层白名单恒写 1，理论不可达，防御性兜底）。理由：校验器是 0011 落库前的最后闸门，任何「无法确认登记」的输入都不许放行；且这些口径不依赖会话假设，与纯函数风格自洽。
   - **测试落位为独立新类**而非追加进既有 `IotDataDictServiceImplTest`（两者均在 brief 声明的测试范围内）：独立成类使本任务 diff 边界清晰、不触 0008 测试文件。
   - **返回 Map 用 LinkedHashMap 保序**（brief 未指定实现形态）：调用方按模板数据项顺序取首个失败项报错时顺序确定。

## 5. 遗留问题

1. **插件注入与显式条件重复的运行时实测**：语义上双保险无害（同值 AND，见未做项 4），0011 接线模板保存流程后有真实会话链路，建议其联测/冒烟时对 `validateForTemplateRef` 路径做一次实测确认并记录。
2. **TASK-0008 报告遗留 4（联动提示）就此闭环**：其建议「按 data_code+status=1 校验（禁用行不在可选集）」即本校验器 OK/DISABLED 分野——0011 接线后 TC-0003 步骤 4/5（模板选码校验、禁用后新引用被拒）即可端到端联测；本任务已交付三态本体与文案。
3. **文案复用提醒（非缺陷）**：`messageFor(code)` 对 OK 返回 null——0011 只需对非 OK 状态取文案，无需判空文案字符串。

---

## 附 1：TDD 红→绿记录

红（实现前，仅跑新测试类）：

```
[ERROR] COMPILATION ERROR :
[ERROR] .../IotDataDictServiceImplValidateForTemplateRefTest.java:[8,42] 找不到符号
[ERROR]   符号:   类 DictCodeStatus
[ERROR]   位置: 程序包 net.otlab.admin.module.iot.service
[INFO] BUILD FAILURE
```

（`validateForTemplateRef` 同步缺失，同为编译期红灯；无运行期失败用例——红灯即编译失败，无断言放松问题。）

绿（实现后，仅跑新测试类）：

```
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.service.impl.IotDataDictServiceImplValidateForTemplateRefTest
[INFO] Tests run: 17, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

## 附 2：契约自查

- 未派 subagent；未把 TASK 置「已完成」（详情状态未回写，留协调者收尾，brief 明文不回写 TASK 详情）；未改 workitems 库其他文件（仅本报告）。
- 改动 4 文件全部在 brief 声明内（验收 2 原始输出贴回）；未碰 frontend/、ingest/、模板域、既有 CRUD 方法体、ResultCode（4002 未动）、application*.yml、resources/sql。
- 2 条验收命令全部执行且原始输出贴回；基线 1243 实测确认后 1260 = 1243 + 17 闭环；TDD 先红后绿（附 1）。
- 纯查询语义达成：校验器无 throw 语句、无事务注解、测试断言 never insert/update/delete；状态三值与文案常量逐字与 brief/任务正文一致（`数据编码未在字典登记: <code>` / `数据编码已停用: <code>`）。
- 批量入口一次 IN 预检达成：去重后单次 `selectList`（测试断言 times(1)），不逐个 count。
- 租户口径达成：tenant_id 显式入参且体现在查询条件（测试断言 wrapper 含 tenant_id 条件且参数值=入参）；校验器不做任何 TenantContext/会话读取（与 0008 既有服务「会话覆盖」风格刻意区分——本方法是纯函数）。
