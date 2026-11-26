# 实施 brief：TASK-0054（dict/sync 字典同步接口——占位转正）

## 批次

- TASK-0054（本批共 1 个，标准任务 2 人天；backend 串行道——数据服务增量同步首任务，前置 0051 已交付占位骨架）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**`（dict/sync 占位转正：queryService impl + VO 结构 + 增量查询 mapper）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/**`（对应测试）
  - **无新 SQL**（复用既有 iot_model_instance / iot_data_item 表，增量靠 update_time/deleted）
  - 不碰 frontend/、ingest/、calc/、dataperm、create 域

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0054.md`
- 上游交付：`reports\TASK-0051-report.md`（0018 六接口+鉴权链；**dict/sync 占位实现说明**）+ 源码 net/otlab/admin/module/openapi/**（OpenApiController:122 / OpenApiQueryService:72 / OpenApiQueryServiceImpl:294 / OpenApiDictSyncVO——开工先读占位全文）
- 关联：workitems 库 `features\FEAT-0017.md`、`requirements\REQ-0028.md`、`testcases\TC-0017.md`、`decisions\ADR-0010.md`（中台建点回传一期不做 PB-10）
- 权威：详细设计 §3.5 dict/sync 接口（D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-detail-design-v1.1.pdf——python pypdf 可提取；参数 since=增量游标）

## 接口与决策（逐条结论，不再讨论）

1. **增量口径**：`GET /openapi/v1/dict/sync?since=<epoch millis>` —查询 `update_time > from_unixtime(since/1000)` 的**实例 + 数据项**两类行（含逻辑删：deleted=1 的行以「禁用」态返回，不能因物理过滤丢失禁用通知）；since 缺省=0（全量）；响应 since 原样回显（调用方游标习惯）
2. **三类清单**：每行 `{id, code, name?, changeType(1新增/2变更/3禁用), updateTime, assetModelRelation}` —**模型/资产对应关系在载荷明确**（中台叫资产、平台叫模型）：实例行 `{instanceId, templateId, modelCode, deptId, status}`；数据项行 `{itemId, dataCode, instanceId, fullKey, sourceType, status}`；中台资产平台模型的命名差异在 javadoc 与 Schema description 说明
3. **权限**：appKey 有效即可（0051 鉴权链已覆盖）——**不走 scope 裁剪**（§3.5 原文）；identity 仍由 requireIdentity(request) 取（审计用）
4. **不扩展 VO 之外的 DTO**（VO 内嵌行 DTO 可新增）；不建新表；不发事件
5. **一期不做**：中台建点回传（ADR-0010 PB-10）——不改范围、不预留写接口
6. **测试口径**：增量边界（since=0/精确边界/未来游标空清单）；三态（新增/变更/禁用含 deleted=1 行）；跨租户隔离（只返回本租户）；appKey 无效 401/6001（0051 既有）；VO 结构与占位兼容（既有字段 since/instances/items 不删，行元素由 Object 具体化）

## 验收命令

环境事实（2026-11-23 实测）：WSL docker 容器已停——MySQL/Redis 在**宿主 phpStudy**：MySQL 8.0.12@3307（root/root）/Redis 3.0.504@6379；sql-exec.sh 每条语句单独一次调用；8098 prod 启动回归用（探测/用毕即停）；TDengine 本批不涉。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. dev 冒烟（每步原始输出贴回）：test_client appKey+HMAC 真调 `/openapi/v1/dict/sync?since=0` → 实例与数据项清单非空且三态齐全 → 造一条禁用（deleted=1）断言以 changeType=3 返回 → since 推进到 mock 边界 → 只返回变更后行 → 跨租户隔离断言
3. prod 档真实启动回归 → 0 ERROR
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0054-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；不改范围外文件（无新 SQL/create 域零触碰）；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0054）。
