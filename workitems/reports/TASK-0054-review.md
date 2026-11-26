# 评审报告：TASK-0054 diff 评审（代码评审）

- 评审对象：提交 `3cdde58`（BASE `5758d5f`，隔离区间 `9aed9a4..3cdde58`，8 文件）——dict/sync 字典同步占位转正
- 评审口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」
- 评审者：fresh 评审子代理（仅依据 brief 必读文件 + 工作区源码；未向协调者追问；未再派 subagent）
- 日期：2026-11-24
- **结论：通过**（Critical 0 / Important 0 / Minor 5）

评审者独立实测：在 `backend/` 执行 `mvn test -pl ot-admin`，汇总 `ot-admin/target/surefire-reports/*.txt` 得 **Tests run: 2395, Failures: 0, Errors: 0, Skipped: 0**，与执行者报告一致。

---

## 一、越界反查复核

- `git diff --name-only 9aed9a4..3cdde58` 与 `git diff-tree --no-commit-id --name-only -r 3cdde58` 均为同一组 8 文件，全部落在 `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**` 与 `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/**` 声明范围内。
- 并行道 TASK-0058 前端提交 `9aed9a4` 已被区间隔离剔除；create / dataperm / frontend / ingest / calc 零触碰；无新 SQL / 新表。
- 「中台建点回传一期不做」实证：`OpenApiController.java` 仅有 `@GetMapping("/dict/sync")`，全控制器无 `@PostMapping/@PutMapping/@DeleteMapping/@PatchMapping`——无写接口、无预留，符合 ADR-0010 TD-0032 与 brief 决策 5。

## 二、评审重点逐条核对

1. **增量口径**：两个 Mapper 均为 `WHERE tenant_id = #{tenantId} AND update_time > FROM_UNIXTIME(#{since}/1000)`（严格大于），无 `deleted = 0` 物理过滤；CASE 首分支 `deleted = 1 THEN 3` 保证逻辑删行以禁用态返回；since=0 → `FROM_UNIXTIME(0)` 含全部历史行。`since` 用 `#{since}` 绑定、无字符串拼接，无注入面。✅
2. **三态分类**：CASE 顺序 `deleted=1→3 / status=0→2 / create_time>=since→1 / 余→2`，对 status=0 未删行归类确定、自洽（执行者报告 §4 的偏离说明合理，brief 未定义 status=0 归类，按冒烟缺「变更」态细化可接受）。但类级 javadoc 与 SQL 不一致——见发现 F-1。⚠️（Minor）
3. **VO 具体化**：`since/instances/items` 三字段保留；`InstanceRow` 含 instanceId/templateId/modelCode/deptId/status，`ItemRow` 含 itemId/instanceId/fullKey/sourceType/status，均继承 id/code/name/changeType/updateTime；中台资产 vs 平台模型命名差异在 javadoc 与 @Schema 已说明。`ItemRow` 无显式 `dataCode` 字段——见发现 F-2。⚠️（Minor）
4. **权限**：`OpenApiQueryServiceImpl.dictSync` 用验签产物 `identity.getTenantId()` 直查，不调用 `dataPermService`；控制器走 `requireIdentity(request)` 审计链；无效 key 403+6001 沿用 0051 过滤器未改。6 个服务单测均 `verifyNoInteractions(dataPermService)`。✅
5. **SQL 注入/性能**：参数化通过。FROM_UNIXTIME 作用于参数侧而非 `update_time` 列侧，谓词形状本身索引友好；但两表 DDL 均无 `update_time` 索引，实际走全表扫描——当前数据量小可接受（brief 口径），报告未单列评估——见发现 F-3。⚠️（Minor）
6. **测试有效性**：6 服务单测（三态/缺省 since/边界/未来游标/deleted=1/跨租户）+ 3 MapperAnnotationBuilder SQL 解析回归（真解析注解、渲染 BoundSql 后断言 SQL 片段，非快照）。精确边界的「严格大于」由 SQL 层 `sql.contains("update_time > FROM_UNIXTIME(?/1000)")` 断言钉住，服务层测试诚实注明「本层原样传游标」。跨租户用 `verify(..., never())` 真断言。判定：有效。✅
7. **执行者登记项**：status=0 归类偏离裁决合理（见上）；「一期不做」实证成立（见上）。

**测试计数核实**：brief 提出「基线 2392+3？」——实测口径为 **基线 2387（TASK-0081 修订轮终值，见 `TASK-0081-review-round2.md`）+ 本任务净增 8 = 2395**：服务单测 26→31（原占位 1 例删除、6 例新增，净 +5）+ 新增 MapperSqlTest 3 例。执行者报告 §2「原 25，净 +6」略有出入（实为原 26、净 +5），但总数 2395 已独立复跑确认。此为台账表述问题，非代码缺陷——见发现 F-5。

## 三、发现清单

| 编号 | 严重级 | 出处（文件:行） | 说明 |
| --- | --- | --- | --- |
| F-1 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/dto/OpenApiDictSyncVO.java:24-25` | 类级 javadoc 三态定义与实现矛盾：写「2 变更（status 由启用转停）」「3 禁用（deleted=1 逻辑删**或 status=0 停用**）」，但两 Mapper 的 CASE 将 `status=0` 未删行归为 `2 变更`、仅 `deleted=1` 归 `3`；javadoc 自相矛盾且与运行时不符（字段级 @Schema:60 正确）。建议后续修正该句，避免中台读类级文档被误导。 |
| F-2 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/dto/OpenApiDictSyncVO.java:109,110-135` | `ItemRow` 无显式 `dataCode` 字段，但 brief §2 载荷清单与该类自身 @Schema 描述均列 `dataCode`；实际值由继承的 `code`（映射 data_code）承载，而 `InstanceRow` 却有显式 `modelCode`——两行键名不对称。值未缺失，但字面键名是否符合中台契约**无法判定**（报告 §5 遗留 2 已登记载荷待中台确认）。 |
| F-3 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/mapper/OpenApiInstanceMapper.java:79-81`；`OpenApiItemMapper.java:50-52`；DDL `resources/sql/upgrade/20261122_iot_template_domain.sql:97-119`、`20261122_iot_data_item_domain.sql:67-96` | 增量查询谓词 `update_time > ...` 在两表均无 `update_time` 索引，走全表扫描；数据量小可接受（brief 明确），但报告未单列性能评估。 |
| F-4 | Minor | `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/service/impl/OpenApiQueryServiceImplTest.java:530` | 常量注释 `DICT_T0_MS = 1763968800000L; // 2026-11-24 10:00:00 +08:00` 与事实不符：该值实为 2025-11-24 15:20 +08；所述时刻应为 1795485600000。断言只做游标透传、值本身任意，功能无损，仅注释误导。 |
| F-5 | Minor | `D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0054-report.md:21` | 报告称服务单测「原 25，净 +6」，实测父提交为 26 个 `@Test`、现 31 个（净 +5）；本任务全量净增为 +8（含 3 个 MapperSqlTest），基线 2387、总数 2395。台账表述偏差，不影响实测结论。 |

## 四、结论

- Critical 0 / Important 0 → 按 §13 **通过**，不需退回修订轮。
- 5 条 Minor 均不进修订轮：F-1/F-2/F-4 建议随中台联调/收口顺带修正（与报告 §5 遗留 2 同批）；F-3 数据量增长后再评估索引；F-5 记入收尾台账。
- 越界反查、增量口径（严格大于/since=0/逻辑删不丢/显式租户）、权限（不消费 dataperm、identity 取审计链）、无写接口（一期不做）、SQL 参数化，以及 2395/0/0/0 全量零回归均复核成立。
