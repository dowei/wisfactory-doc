# 实施 brief：TASK-0029（攒批写 TDengine、Redis 点位缓存与新值事件——采集接入链收官）

## 批次

- TASK-0029（本批共 1 个，标准任务 4 人天；Go 道 `ingest/**` 纯 Go——前置 0027/0028 已交付统一流水线/校验/幂等）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `ingest/**`（真执行器/Redis 缓存/新值事件/ingestd 接线补全/0027 遗留分流两项）
  - 不碰 backend/**、frontend/**、workitems（报告除外）、契约文件（超级表契约 v1.0.0 冻结）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0029.md`
- 上游交付：`reports\TASK-0028-report.md`（四步校验/时间三分/死信——你的上游）+ `reports\TASK-0027-report.md`（遗留分流：停机排空攒批缓冲/sink 队头约束归本批）+ `reports\TASK-0073-report.md`（Batcher/Writer 抽象/攒批水位 500 或 200ms/写失败整批重排重试受同上限约束/ADR-0014 FIFO）
- 决策：`decisions\ADR-0010.md`（LWT 离线发事件不直写状态 Redis）+ `features\FEAT-0009.md`、`requirements\REQ-0013.md`、`testcases\TC-0009.md`
- 契约权威：`D:\workspace\WisFactory\resources\contract\`（超级表 v1.0.0：st_point_num/st_point_state 四表契约、附录 B DDL、kind/表映射——ingest/internal/tdengine 已有构造器）+ 统一消息模型 v1.1.0
- 服务器事实（协调者 2026-11-23 实测）：TDengine http://192.168.31.145:6041（3.4.3.9，root/taosdata）fd 根治全链毫秒级；**CREATE DATABASE 20-25s 慢特性**（curl -m 60）；iot_default 四张超级表就绪；库名派生唯一入口 DBName()（LOG-0011：附录 B 示例租户名偏离，0027/0029 建库须统一走 DBName() 不得手拼）

## 接口与决策（逐条结论，不再讨论）

1. **真 TDengine 执行器**（零改动替换假执行器）：HTTP SQL 执行器走 taosAdapter REST（`INGEST_TDENGINE_BASE_URL/USER/PASSWORD` env；Basic 认证）；INSERT 构造复用 0073 既有（kind/表映射/附录 B 列序）；**攒批写断言**：单 supertable 一条 INSERT 多 VALUES（禁止逐条写——契约测试锁 INSERT 语句形态）；子表首写自动创建（`CREATE TABLE IF NOT EXISTS ... USING ... TAGS(...)`——标签按配置快照含 tenant_id/kind，与超级表契约一致）；写失败整批重排重试（0073 既有语义，受攒批缓冲同上限约束）；批量写请求并发与失败重试的幂等性靠 0027 已交付 (pointCode,ts) 去重兜底。
2. **Redis 点位缓存**：真客户端替换内存假实现——key `iot:pt:{tenant}:{code}`（§3.5），value v/q/ts；**写穿透**（落库成功后更新缓存）；**缓存失败不阻塞落库**（fail-soft WARN）；**未命中回源**（读路径懒回源 TDengine 最近值——`SELECT ... ORDER BY ts DESC LIMIT 1`，按需实现不预热的读-through）；**Redis 选库口径**：生产 ingest 用 db0（key 前缀 iot:pt: 与 backend 键空间天然隔离），测试用 db1+（Go 道纪律）；`INGEST_REDIS_ADDR/DB` env。
3. **新值事件**：薄接口 ValueEventPublisher + **Redis Pub/Sub 实现**（channel `iot:newvalue:{tenant}`，payload {tenant,pointCode,ts,value,quality} JSON）——计算引擎消费方未建（0040s），本批交付发射侧+契约测试，消费对接登记演进位；**ADR-0010 兑现**：LWT 离线走事件不直写状态 Redis——LWT/status 消息按 data_type 分发路径照流水线既有分发，离线语义只发事件。
4. **0027 遗留分流两项（本批承接）**：①**停机排空攒批缓冲**（SIGTERM 排空：缓冲内合法数据写完再退，超时兜底强制退出——排空失败 WARN 不阻塞退出）；②sink 队头约束（真执行器落位后 sink 的调用形态复核，如为同步队头瓶颈则批量化）。
5. **口径固化（0028 契约留白三项，协调者裁决——仅登记口径，无代码变更）**：①纠偏显式标记形态=结构化 WARN+计数器（不改 CONV-02 码域，维持 0028 裁决）；②被拒点重发计数口径=每次到达都计入对应原因码（拒绝计数=到达次数视角，不去重）；③纠偏计数语义=纠偏动作次数（按点采样 WARN）。
6. **不碰**：校验/死信/时间策略（0028 已交付）；MQTT 消费（0027）；backend；契约冻结。

### 测试口径（TDD）

真执行器：INSERT 语句形态契约测试（多 VALUES 单语句/列序/kind 映射/子表自动创建语句）——假 HTTP server 断言请求体；攒批水位与 200ms 定时双触发；写失败整批重排上限；Redis 缓存：写穿透/失败不阻塞/未命中回源/键形态（真 Redis db1 集成测试可选 env 门控）；新值事件发射 payload 契约；停机排空；ADR-0010 LWT 路径。既有 168 契约测试零回归。红转绿留痕。

## 验收命令

环境事实（协调者 2026-11-23 实测）：WSL docker mysql/redis 在跑；**TDengine 已根治可用**——live 段必须 env 注入 `INGEST_TDENGINE_BASE_URL=http://192.168.31.145:6041`（root/taosdata）；8090/8098 纪律同前；WSL2 空闲关停→长间隙前保活循环。

1. `cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（既有 168 零回归）
2. **live 真库冒烟**（每步原始输出贴回，本任务是 fd 根治后首次真库写入验收）：
   a. scratch 租户库准备：按 DBName() 派生名（如租户码 verify29）→ REST 建库（curl -m 60，慢特性）→ CREATE STABLE 按契约附录 B（scratch 库内，测完清）
   b. ingestd 起（env 注入 TDengine/Redis）→ report 合法批（跨 2 个 supertable 的混合点位）→ 断言：st_point_num/st_point_state 子表自动创建（SHOW STABLES/SHOW TABLES）+ 值可查（SELECT）+ 攒批形态（计数/日志证明非逐条）+ Redis 缓存键在位（v/q/ts）+ 新值事件收到（redis-cli SUBSCRIBE 或 Pub/Sub 计数）
   c. 写失败重试：TDengine 短暂不可达注入（如改 URL 指向坏端口重启 ingestd）→ 整批重排证据 → 恢复
   d. 停机排空：SIGTERM 时缓冲内数据落库证据
   e. 清理：scratch 库 DROP + SHOW DATABASES 终态证明
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 ingest/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0029-report.md`。五项齐全 + TD-0057 延续登记（MQTT 路径）+ 一句话结论。

## 契约

- no-subagents；只改 ingest/**；**提交只 add ingest/****；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）；超级表契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」；scratch 对象测完清理绝不污染共享库。
