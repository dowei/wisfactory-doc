# 实施 brief：TASK-0052（iot_api_client 调用方管理与调用审计）

## 批次

- TASK-0052（本批共 1 个，标准任务 3 人天；backend 串行道——前置 0051 已交付对外六接口与 HMAC 鉴权链）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**`（调用方管理接口/轮换/缓存失效钩子——**注意：openapi 是独立模块，不在 iot/ 下**）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/openapi/**`（对应测试）
  - `resources/sql/upgrade/20261123_iot_api_client_rotation.sql`（新增）
  - `backend/ot-core/src/main/java/**/ResultCode.java`（如需新码——最小化）
  - 不碰 0051 交付的六接口行为、前端、ingest、dataperm

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0052.md`
- 关联：workitems 库 `features\FEAT-0017.md`、`requirements\REQ-0035.md`、`testcases\TC-0017.md`、`decisions\ADR-0006.md`（缓存主动失效思路）
- 上游交付（0051——本任务的地基，开工先读）：`reports\TASK-0051-report.md` + 源码 net/otlab/admin/module/openapi/**（auth/OpenApiAuthStrategy·OpenApiClientIdentity·OpenApiCredentials、config/OpenApiFilterConfig·OpenApiProperties、audit/OpenApiAuditWriter）+ `resources/sql/upgrade/20261122_iot_openapi_domain.sql`（**已交付双列 secret**：secret_hash=SHA256(明文) 供 0052 快速校验 + secret_encrypted=SM4(明文) 供 HMAC 验签——上游仅 secret_hash 无法支持 HMAC 的登记差异；test_client 冒烟种子）
- 表结构：附录 A p34 iot_api_client（0051 脚本头注已登记差异）——本任务只加列不改既有列

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261123_iot_api_client_rotation.sql，加列 + 幂等 + 头注）

```sql
ALTER TABLE `iot_api_client`
  ADD COLUMN `secret_encrypted_prev` varchar(512) DEFAULT NULL COMMENT '轮换前密文（SM4，并行窗内仍可验签）',
  ADD COLUMN `secret_hash_prev` varchar(128) DEFAULT NULL COMMENT '轮换前哈希（SHA256，快速校验）',
  ADD COLUMN `secret_rotated_at` datetime DEFAULT NULL COMMENT '轮换时间（并行窗口起点）';
```

（头注登记：上游无双密钥列——「双密钥轮换并行一周」（TASK-0052 要点）需物理承载；加列不动既有列与种子行。）

### 管理接口（management 面，照 0056/0016 范式）

1. **开通**：`POST /openapi/client`（管理面）——服务端生成随机 secret（≥32 字符安全随机）→ SM4 落 secret_encrypted + SHA256 落 secret_hash → **明文仅本次响应体返回一次**（响应含 appKey+secret 明文+「仅此一次」提示），任何后续接口/列表不回明文；appKey 全局唯一（uk_app_key 撞键 → 业务码 4002 口径照 0015 M-2 范式）；scope 字段 JSON（空=默认拒绝——0051 filter 已强制，管理侧透传校验 JSON 合法性）；rate_limit 编辑同接口。
2. **列表/详情**：`GET /openapi/client/list`、`GET /openapi/client/{id}`——只含掩码（secret 展示为前 4+****+后 4 或纯 ****——照 0019 凭据掩码范式）、scope/rate_limit/status/轮换时间；无任何密文/明文列出。
3. **轮换**：`POST /openapi/client/rotate`（body：id）→ 当前值移入 prev 三列（secret_encrypted_prev/secret_hash_prev/secret_rotated_at=NOW）→ 生成新 secret 入主列 → **响应一次性返回新明文**；**验签顺序**（OpenApiAuthStrategy 扩展）：先当前列验 → 失败再 prev 列验（且 now-secret_rotated_at ≤ 7d 并行窗）→ 过窗后惰性清理 prev（验签路径判定时清理，不建扫描任务）。
4. **吊销/禁用**：`PUT /openapi/client/status`（status 置 0）→ **立即 403 无 TTL 空窗**——执行者实证 0051 filter 的客户端状态读取现状：若每请求查库则天然立即（报告说明）；若存在缓存（scope/status 缓存）则加主动失效钩子（ADR-0006 思路，0025 publish-log 同款范式）。
5. **审计**：iot_api_call_log 已由 0051 交付（403 全量异步审计）——本任务不重复建设；管理动作（开通/轮换/吊销）走 @OperateLog 留痕（0079/0080 同款）。
6. **权限码**：openapi:client:list/add/edit/remove 四码 + 菜单种子（新页「调用方管理」挂系统管理下合适父节点——照 0024/0016 种子范式，挂点在头注登记）+ 超管授权；幂等。
7. **双密钥并行窗**：7 天常量 `ot.openapi.rotation-window-days`（默认 7，yml 新键）；并行窗内新旧 secret 均可验签（业务连续性）；过窗旧 secret 失效。
8. **不碰**：0051 六接口行为/scope 鉴权语义/限流计数逻辑（filter 内核不动，只加验签顺序扩展与状态读取核查）。

### 测试口径（TDD）

开通（appKey 唯一 4002/明文仅一次——列表与详情断言无明文/scope 空默认拒绝落库）；轮换（prev 三列带走旧值/新明文一次性/并行窗内双验签/过窗旧失效惰性清理）；吊销立即 403（无 TTL 空窗——若 filter 有缓存则失效钩子红转绿）；审计 403 行与 @OperateLog 留痕；菜单种子 4075-4078 幂等+超管授权；0051 既有鉴权测试零回归。红转绿留痕。

## 验收命令

环境事实（协调者 2026-11-23 实测）：WSL docker mysql/redis 在跑；sql-exec.sh 每条语句单独一次调用；8098 prod 启动回归用（探测/用毕即停）；TDengine 本批不涉；WSL2 保活。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对三新列 + 菜单种子行
3. dev 冒烟（每步原始输出贴回）：开通（明文一次性+唯一性 4002）→ 新 appKey 真打六接口（HMAC 验签过+scope 外 4003 口径照 0051）→ 轮换（新旧并行验签都过）→ 模拟过窗（SQL 改 rotated_at=-8d）→ 旧 secret 403 → 吊销 → 立即 403 → 审计行核对 → 全停
4. prod 档真实启动回归 → 0 ERROR
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0052-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0052）。
