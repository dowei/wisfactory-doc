# 实施 brief：TASK-0042（DELTA/INTEGRAL 算子与同构存储）

## 批次

- TASK-0042（本批共 1 个，标准任务 7 人天；计算引擎道 calc/**——前置 0041✅，算子核心，解锁 0045 指标聚合→0060 指标看板）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `calc/**`（DELTA/INTEGRAL 原生算子 + 同构写 st_item_value + Redis 缓存）
  - 不碰 backend/frontend/ingest/契约文件/workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0042.md`
- 决策：FEAT-0013/REQ-0020/REQ-0019/TC-0013/ADR-0010（INTEGRAL 不开放手写）
- 上游交付（开工先读）：reports\TASK-0041-report.md（DualTriggerExecutor+ArithmeticEvaluator 白名单+ResultSink 边界）+ TASK-0049-report.md（st_item_value 留痕形态 full_key 规范+六元组 tag+逐语句 REST）+ calc/src/** 现有执行器/状态代码
- 权威：详细设计 §1.6（DELTA/INTEGRAL 算子口径）+ §1.5B + CONV-02 质量码 + st_item_value 超级表契约
- 范式：calc/README.md + Contract 命名测试

## 接口与决策（逐条结论，不再讨论）

1. **DELTA 算子（原生实现，不走表达式引擎）**：差分=当前值-上周期值；**回退打码**（差值<0 → 质量码按 CONV-02 打码，不写负值）；**换表 PARAM 边界分段**（换表事件触发新基线，跨段不差分）；**溢出归零**（差值超量程→0+质量标记）；**跨周期标记**（采样窗不连续→打标不差分）。算子函数签名与 0041 ArithmeticEvaluator 白名单对齐——DELTA 可引用、INTEGRAL 不开放手写（ADR-0010）
2. **INTEGRAL 算子（原生）**：**真实间隔梯形积分**（按相邻样本真实 ts 间隔而非固定周期）；**缺口不积分**（质量码非 0/样本缺失的间隔跳过）；**停机零增量区分断线**（停机=0 增量有明确原因，断线=不积分+打标，两者区分）；同构结果写 st_item_value
3. **同构存储**：结果按 data_type 写 st_item_value（DIRECT/FORMULA/ROLLUP 同源同构——与 0049 STATUS 留痕并列但不同 full_key）+ Redis 点位缓存（复用 0041 ValueStore/0049 RESP2 客户端形态）；**逐语句 REST 下发**（0029 实证多语句信封静默空操作）、full_key={instance}.{data_code}（0049 F-02 教训——禁止 `full:` 前缀）、六元组 tag 精确断言
4. **能源计量模板派生路径**：派生项（source_type=2 公式派生）只改绑定方向即换向（正负号随绑定方向）——契约测试锁定；模板默认/覆盖口径与 0081 三方合并一致
5. **失败语义**：算子异常不写假 0（CONV-02 打码）、单节点失败隔离（0041 既有纪律）、TDengine 写入失败 WARN 不阻塞计算
6. **TDengine 环境**：REST 6041 当前故障（15s 超时/reset，TD-0053 登记）——本任务写路径做契约测试（假适配器捕获 INSERT SQL+六元组 tag 精确断言）、真机写入+读回列为环境恢复后交付条件，不阻塞代码收尾（与 0049 同口径）
7. **不碰**：0045 指标聚合、0049 状态判定引擎、backend 任何代码、契约冻结

### 测试口径（TDD）

Contract 测试（DELTA 五异常场景：回退/换表/溢出/跨周期/正常差分；INTEGRAL 三场景：真实间隔/缺口/停机断线区分；同构写 full_key+tag 精确断言；派生项换向）；既有 0041/0049 测试零回归；红转绿留痕。

## 验收命令

环境事实（2026-11-24）：Redis 宿主 phpStudy 3.0.504@6379；TDengine REST 故障（真机补验列条件）；8095 归 calc（探测/用毕即停）；并行 backend 道 0050 在途（不碰 backend mvn）。

1. `cd calc && mvn test` → BUILD SUCCESS、0 失败 0 错误（0041/0049 基线零回归+新增，Results 段贴回）
2. live 冒烟（每步原始输出贴回）：calc 起（8095）→ 构造 DELTA/INTEGRAL 输入序列 → 算子输出断言（五异常+三场景）→ 同构写 SQL 契约断言 → 派生项换向 → 全停；真机写入列环境恢复后条件
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 calc/**

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-report.md`。五项齐全 + 环境恢复后补验条件 + 一句话结论。

## 契约

- no-subagents；只改 calc/**；提交只 add calc/**；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）；契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
