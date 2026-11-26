# 实施 brief：TASK-0049（状态判定执行——优先级求值+Redis+留痕）

## 批次

- TASK-0049（本批共 1 个，标准任务 4 人天；**计算引擎道 calc/**——前置 0048✅/0041✅，状态判定核心，解锁 0050 状态读口→前端 0059/0058 批二）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `calc/**`（状态判定模块：求值引擎 + Redis 当前状态 + st_item_value 留痕，接入 0041 执行器）
  - 不碰 backend/frontend/ingest/契约文件/workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0049.md`
- 决策/关联：FEAT-0016/REQ-0007/TC-0016/ADR-0010（LWT 离线作判定输入）/ADR-0009（状态编码稳定序号）
- 上游交付（开工先读）：`reports\TASK-0048-report.md`（iot_status_rule 配置接口+绑定，状态规则数据源）+ `reports\TASK-0041-report.md`（DualTriggerExecutor 执行器+ResultSink 边界+Redis 订阅）+ calc/src/** 现有 executor/* 代码
- 权威：详细设计 §1.5B 状态判定规则 + §5 判定链路（D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-design-v2.2.pdf，pypdf 可提取）
- 契约：CONV-02 质量码 + st_item_value 超级表契约（resources/contract/）
- 范式：calc/README.md + Contract 命名契约测试

## 接口与决策（逐条结论，不再讨论）

1. **求值引擎**：实例被引用数据项更新时触发（复用 0041 DualTriggerExecutor 的到达驱动通道，或在本模块内订阅同一 ResultSink 输出——选最小改动方案 javadoc 说明）→ 按 iot_status_rule.priority **升序逐条求值、首个命中即当前状态**；**直绑状态优先于规则推导**（实例/数据项直绑分支短路）；全不命中→UNKNOWN + 质量标记（CONV-02，不硬套默认状态）；状态编码用**稳定序号**（ADR-0009，防历史漂移，代码映射集中单一来源）
2. **Redis 当前状态**：`iot:st:{tenant}:{instance}` key，value=状态稳定序号+时间；Redis 客户端复用 calc 既有 RESP2（0041 RedisArrivalSubscriber 同款，零新增依赖）；**只写状态变化**（未变不重写 key——与 st_item_value 未变不重写同纪律）
3. **st_item_value 留痕**：状态变化以 STATUS 数据项写 st_item_value（稳定序号、quality=0、**未变不重写**）——注意 0041 ResultSink 边界当前不写业务表，本任务状态留痕是**首个真实 ResultSink 实现**（写 TDengine st_item_value）——但 TDengine 写入需谨慎：本任务 writer 走 0040 既有 TdengineAdmin HTTP REST 风格（逐语句独立下发——0029 实证多语句信封静默空操作），子表首写 INSERT...USING 标签含 tenant_id/kind
4. **LWT 离线输入（ADR-0010）**：EMQX LWT/网关离线事件作为判定输入（离线状态优先/触发重判）——本期 calc 侧订阅 Redis 事件通道（如 iot:gwstatus:{tenant}，0029 发射侧拟定形态）或直接消费 0027 已落库的状态消息；**不直写状态 Redis**（状态判定是本任务职责）
5. **失败语义**：判定失败只影响该实例状态（不发假状态、不覆盖旧 Redis 值）；留痕写失败 WARN 不阻塞求值
6. **不碰**：0050 状态读口（后续道）、0042 算子、0045 指标聚合、backend 任何代码、契约冻结

### 测试口径（TDD）

Contract 测试（优先级升序首命中/直绑短路/UNKNOWN 不硬套/稳定序号/未变不重写 Redis 与 st_item_value/LWT 离线触发/失败隔离）；既有 0041 的 140 测试零回归；red→green 留痕。

## 验收命令

环境事实（2026-11-24）：Redis 宿主 phpStudy 3.0.504@6379（Redisson/RESP2 实测可连）；TDengine 192.168.31.145:6041（root/taosdata，fd 根治可用、CREATE DATABASE 20-25s 慢特性）；8095 归 calc（探测/用毕即停）；scratch 对象测完清理；并行 backend 道可能有任务（不碰 backend mvn）。

1. `cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误（140 基线+新增，Results 段贴回）
2. live 冒烟（每步原始输出贴回）：calc 起（8095）→ 造状态规则+数据项更新事件 → 断言求值链（故障优先于运行）+ Redis iot:st:{tenant}:{instance} 写+st_item_value 留痕→ 重复同状态断言未重写 → LWT 离线触发重判 → 全停+TDengine scratch 清理（如建）
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 calc/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；只改 calc/**；提交只 add calc/**（绝不 git add -A）；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）；契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
