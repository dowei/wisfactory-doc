# 实施 brief：TASK-0028（校验流水线、死信计数与质量判定）

## 批次

- TASK-0028（本批共 1 个，标准任务 3 人天；Go 道 `ingest/**` 纯 Go——前置 TASK-0027 已交付统一流水线/幂等/未知编码死信骨架）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `ingest/**`（quality 判定逻辑、死信计数与告警、时间语义、既有挂点接线）
  - 不碰 backend/**、frontend/**、workitems（报告除外）、契约文件（CONV-02 质量码 0-10 枚举冻结，发现问题报告并停）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0028.md`
- 上游任务交付：`reports\TASK-0027-report.md`（流水线接线/幂等/未知编码死信骨架/遗留分流——其中「sink 同步调用队头约束」「停机排空」归 0029 不在本批）、`reports\TASK-0073-report.md`（校验挂点位置）
- 决策：workitems 库 `decisions\ADR-0009.md`（人工补录不走 ingest——边界注记）；`features\FEAT-0009.md`、`requirements\REQ-0013.md`、`testcases\TC-0009.md`
- 契约权威：`D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md`（点位有效范围 min/max 字段）+ 统一消息模型 v1.1.0（quality 语义/ts 毫秒）+ CONV-02 质量码枚举（ingest/internal/quality）
- 详细设计锚点：§3.2 p24（ts 设备侧采样时间与 sentAt 分开/迟到按 ts 落库/时间戳越界为拒绝明细；quality 网关先填、超限类接入按 min/max 二次判定覆盖写）+ §1.9A p17（断网补传按原始 ts 落库——计算引擎按补传窗重算的锚点，**原始 ts 不可被纠偏破坏**）

## 接口与决策（逐条结论，不再讨论）

1. **量程质量判定（质量挂点真实现）**：按快照点位有效范围 min/max 二次判定——value<min → quality=1（下限越界）、value>max → quality=2（上限越界）、**原值保留不丢弃不假 0**（验收锚点：超限打码原值保留）；gateway 已填 quality 与接入判定的合并规则=§3.2 口径（超限类接入覆盖写，非超限类保留网关值）；min/max 缺失的点位跳过量程判定（原 quality 透传）；边界值（==min/==max）不算越界。
2. **死信计数与告警**：0027 的未知编码死信扩展为统一死信通道——原因码枚举（unknown_code/ts_overflow/ts_too_old/format_invalid 等，与 HTTP report 拒绝明细原因码对齐）；**按租户隔离计数**（进程内有界 map，key=tenant+reason，多实例 Redis 化登记演进位）；告警=WARN 日志（固定格式含租户/原因/计数，防风暴：首次+每 N 条采样，N 配置化 `INGEST_DEADLETTER_WARN_EVERY` 默认 100）；「异常队列单独存储」一期落地为进程内有界环形缓冲（最近 N 条明细，供排查读口/日志），Redis 化登记同上。
3. **时间语义三分（协调者裁决——设计文档无独立章节，以 TASK 要点+§1.9A 锚点为口径）**：
   - **ts 越界拒绝**：ts > now+`INGEST_TS_FUTURE_REJECT`（默认 5m）→ 拒绝死信 reason=ts_overflow；ts < now-`INGEST_TS_MAX_AGE`（默认 30 天）→ 拒绝死信 reason=ts_too_old（脏数据防线；MQTT 路径进死信计数，HTTP 路径进拒绝明细）
   - **未来窄带纠偏+打标**：now < ts ≤ now+5m（未来时钟漂移窄带）→ **ts 纠偏为接收时间并打标**——「纠偏仅对超补传窗口时间戳生效并打标」的落地；打标=结构化 WARN 日志（采样）+独立计数器，**不改 CONV-02 quality 码域**（枚举冻结，不自造「已纠偏」码——如产品要显式标记走契约变更另议）
   - **补传豁免**：now-30 天 ≤ ts < now → **原始 ts 落库，不纠偏不打标**——§1.9A 断网补传锚点（计算引擎按补传时间窗识别重算，原始 ts 是重算窗口的依据，不可破坏）；`INGEST_TS_MAX_AGE` 内的迟到数据同理豁免
   - 所有阈值 env 配置化；javadoc 登记三条路径与依据
4. **去重复核**：0027 幂等已交付——本批补齐校验视角测试（去重发生在校验流水线哪一步的口径确认：四步校验「格式→量程→时间→去重」顺序中，去重靠后——重复上报不产生重复死信计数）
5. **人工补录不走 ingest（ADR-0009）**：边界注记——ingest 不提供任何人工补录入口（无新增代码；文档/javadoc 登记）。
6. **不碰**：攒批写入（0029）、Redis 点位缓存（0029）、新值事件（0029）；0027 遗留的「sink 同步调用队头约束/停机排空」归 0029（brief 已分流）。

### 测试口径（TDD）

量程边界用例（min/max 开闭区间/缺失跳过/原值保留断言/无假 0）；quality 合并规则（网关先填+超限覆盖）；死信（未知编码/时间越界/格式非法原因码+租户隔离计数+采样告警格式）；时间三路径（越界拒绝/窄带纠偏打标/补传豁免原始 ts）+阈值配置化；四步顺序与去重位置口径；既有 135 契约测试零回归。红转绿留痕。

## 验收命令

环境事实（协调者 2026-11-23 实测）：WSL docker mysql/redis 在跑（Go 测试 Redis db1+）；TDengine 不涉（writer 假执行器）；8090/8098 纪律同前；WSL2 保活。

1. `cd ingest && go build ./... && go vet ./... && go test ./...` → 全绿（既有 135 零回归）
2. live 冒烟（若 8098 空闲；忙则跳过登记待补）：ingestd 起 → report 混合批（合法+超限值+未知编码+未来漂移 ts+陈旧 ts）→ 断言各路径分流正确（accepted/rejected/死信计数/纠偏打标日志）→ 全停
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 ingest/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0028-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；只改 ingest/**；**提交只 add ingest/****；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）；CONV-02 冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
