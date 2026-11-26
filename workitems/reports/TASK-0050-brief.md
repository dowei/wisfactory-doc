# 实施 brief：TASK-0050（状态展示读口——数据服务统一状态查询）

## 批次

- TASK-0050（本批共 1 个，标准任务 2 人天；backend 数据服务道——前置 0049✅，解锁前端 0059 设备详情/0058 批二状态卡片）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**`（对外状态查询 + 数据服务读口）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/status*/**` 或数据服务状态模块（当前状态 Redis 读+历史时间线读）
  - `backend/ot-admin/src/test/java/**`
  - 不碰 calc/**（0049 状态判定引擎零改动）、frontend/**、ingest/**、契约文件、status 规则表配置域（0048 已交付）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0050.md`
- 决策：FEAT-0016/REQ-0007/TC-0016/ADR-0004（展示端不做状态现算）/ADR-0009（稳定序号）
- 上游交付（开工先读）：`reports\TASK-0049-report.md`（状态写入侧：Redis iot:st:{tenant}:{instance} + st_item_value 留痕 full_key 规范+quality=0+未变不重写）+ `reports\TASK-0048-report.md`（iot_status_rule 规则表+字典——翻译数据源）+ `reports\TASK-0051-report.md`（对外 API 六接口+6001-6099 错误段+HMAC 鉴权链）+ reports\TASK-0054-report.md（dict/sync 增量口径）
- 关联代码：openapi/**（OpenApiController/QueryService/审计链）+ statusrule 域（0048）
- 权威：详细设计 §1.5B/§5（状态链路）+ §3.1 对外状态查询接口

## 接口与决策（逐条结论，不再讨论）

1. **当前状态读口**：`GET /api/v1/iot/status/current?instanceId=`（管理面展示用）+ `GET /openapi/v1/status/current?instanceId=`（对外）——从 Redis `iot:st:{tenant}:{instance}` 读（0049 写入形态，值=稳定序号+时间），**不现算**（ADR-0004）；无该设备状态→返回 UNKNOWN（seq=null/quality=3 口径与 0049 一致），**不报错**；走数据权限拦截（管理面 dept 裁剪；对外走 0051 appKey+scope）
2. **状态翻译**：稳定序号→名称/颜色，以规则表 iot_status_rule + 字典为准（0048 交付面）；**翻译随规则发布变化**——不硬编码状态名/颜色字典（0049 F-06 deferred 的 Redis statusName 硬编码就是反例，本读口从规则表动态取）；对外载荷含 statusCode/statusName/color/changeTime
3. **历史时间线读口**：`GET /api/v1/iot/status/history?instanceId=&from=&to=` + 对外对应端点——读 TDengine st_item_value（0049 写入形态：full_key={instance}.STATUS、value=稳定序号、quality=0）；**TDengine REST 当前故障（15s 超时/reset，TD-0053 登记）**——本任务实现读口+契约测试（假适配器捕获 SELECT SQL 精确断言 full_key 过滤/时间范围/序列），真机读回列为环境恢复后交付条件，不阻塞代码收尾（与 0049 同口径）
4. **对外载荷与错误段**：状态查询载荷字段与 0051 对外六接口风格一致；错误用 6001-6099 段（无权限 6002/越 scope 6003/参数错 6004 等，照 0051 既有口径不新增码）；scope 裁剪（对外 status 查询须 scope 含目标实例，否则 6003）
5. **三处一致性**：详情页（0059 后续）/色块（0058 批二后续）/对外 API 均经本读口——同一翻译逻辑单一来源（本任务交付后前端只消费不改口径）
6. **不碰**：0049 calc 状态判定引擎、0048 规则配置域、前端（0059/0058 批二后续消费）、TDengine 写（只读）

### 测试口径（TDD）

当前状态读（有/无 Redis 状态→UNKNOWN/数据权限裁剪/对外 scope）；翻译动态性（规则表改后翻译随之变，非硬编码）；历史 SQL 契约（full_key 精确过滤/时间范围/序列）；对外载荷+错误段；既有 0051/0048 测试零回归。红转绿留痕。

## 验收命令

环境事实（2026-11-24）：MySQL/Redis 宿主 phpStudy（8.0.12@3307/3.0.504@6379）实测正常；**TDengine REST 故障（6041 TCP 通但 15s 超时/reset，服务器侧，TD-0053 登记）**——历史真机读回列为环境恢复后条件；8098 启动前探测、用毕即停；sql-exec.sh 每条语句单独一次调用。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. dev 冒烟（每步原始输出贴回）：造 Redis 状态行 → 管理面/对外当前状态查询（含 UNKNOWN 空态）→ 翻译随规则发布变化 → 对外 scope 外 6003 → 历史 SQL 契约断言；TDengine 真机读回登记"待环境恢复补验"
3. prod 档真实启动回归 → 0 ERROR
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内（calc/frontend 零触碰实证）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0050-report.md`。五项齐全 + 环境恢复后补验条件登记 + 一句话结论。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0050）。
