# 实施 brief：TASK-0041（双触发执行器——周期调度+到达驱动+节流幂等）

## 批次

- TASK-0041（本批共 1 个，标准任务 4 人天；**计算引擎道** calc/ 工程内——前置 0040✅/0029✅，双触发计算核心，解锁 0042/0049/0050/0059/0060 的总闸门）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `calc/**`（执行器核心：周期调度/到达驱动/节流幂等/汇总算子/失败隔离）
  - 不碰 backend/frontend/ingest/契约文件/workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0041.md`
- 决策/关联：FEAT-0013/REQ-0020/TC-0013（workitems 库）+ ADR-0014（攒批 FIFO）
- 上游交付（开工先读）：`reports\TASK-0040-report.md`（calc 工程结构/依赖图 GraphHolder/AppliedGraph+ACK 回执+对账循环）+ calc/src/** 现有图代码 + `reports\TASK-0029-report.md`（新值事件 iot:newvalue:{tenant} 发射侧）——**到达驱动的触发源**
- 契约：`resources\contract\`（统一消息模型 v1.1.0 质量码 CONV-02/超级表 v1.0.0）+ 详细设计 §1.6 计算周期/§1.5B 状态规则
- 范式基准：calc/README.md（照 ingest README 体例）+ Contract 命名契约测试

## 接口与决策（逐条结论，不再讨论）

1. **周期调度器**：按数据项 calc_period（HOUR/DAY/MONTH/YEAR，经 0081 已 public 的 mapCalcPeriod 同款口径在 calc 侧独立实现——**calc 不依赖 backend 代码**）整点触发；结果时间戳对齐周期边界（period_start/period_end）；Spring @Scheduled 或自实现调度线程（选简单可靠方案，javadoc 说明取向）
2. **到达驱动**：订阅 Redis Pub/Sub `iot:newvalue:{tenant}`（0029 发射侧真实通道）——新值事件到达触发依赖图中该数据项的下游重算；同秒合并节流（key=数据项全键+周期窗，窗口内多次到达只触发一次，保留最后正确结果）；**幂等**：同项同窗口多次触发只产出一次结果（窗口键去重，进程内有界 TTL，多实例 Redis 化登记演进位）
3. **两通道统一执行引擎**：周期与到达两通道入同一执行器——按依赖图**拓扑顺序**执行（复用 0040 GraphBuilder 的图结构+拓扑序）；单节点失败只影响其下游（标记失败状态+日志），不阻塞整图其他分支；失败结果不发"假 0"（CONV-02 缺失打码语义）
4. **汇总算子**：动态查找子级（按 rollup 配置）、SUM/AVG/MAX/MIN 四法；**缺失打码 7**（quality=7 的子级贡献不纳入但保留打标）、开关关闭不纳入（ADR-0008 T1 保险：新增项默认不参与汇总）
5. **结果落地**：当前 calc 无写结果表（st_metric_value 归 0045、数据项结果表归计算引擎后续）——本任务执行器产出结果通过内部 ResultSink 接口（内存假实现+日志）+ 契约测试锁定结果形态；**不写 TDengine/不写 MySQL 业务表**（避免与 0045 抢表），演进位登记
6. **不碰**：DELTA/INTEGRAL 算子（0042）、指标聚合（0045）、状态判定（0049）、0063 计算状态写、backend 任何代码、契约冻结

### 测试口径（TDD）

Contract 测试（周期边界触发/到达节流同秒合并/幂等同窗只算一次/拓扑顺序/单节点失败隔离/汇总四法+打码 7+开关关闭）；既有 0040 的 115 测试零回归；红转绿留痕。

## 验收命令

环境事实（2026-11-24）：Redis 宿主 phpStudy 3.0.504@6379（Redisson 3.50 实测可连）；8095 归 calc（启动前探测、用毕即停）；TDengine 本任务不写；并行 backend 道可能有 0083 在途（不碰 backend mvn）。

1. `cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误（115 基线 + 新增，Results 段贴回）
2. live 冒烟（每步原始输出贴回）：calc 起（8095）→ Redis pub 新值事件 → 观察执行器触发日志（拓扑顺序+节流合并）→ 周期调度按短周期配置实跑一轮 → 注入单节点失败观察隔离 → 汇总四法结果断言 → 全停
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 calc/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0041-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；只改 calc/**；提交只 add calc/**（绝不 git add -A）；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）；契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
