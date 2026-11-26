# 评审 brief：TASK-0049 diff 评审（代码评审）

> 实现提交 fded29b（BASE=7628007，单提交 23 文件纯 calc/**）——状态判定执行（优先级求值+Redis 当前状态+st_item_value 留痕+LWT 离线输入）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-diff.txt`（7628007..fded29b，23 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-report.md`（含真 TDengine 写入环境遗留登记）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0049.md`
- 决策：FEAT-0016/REQ-0007/TC-0016/ADR-0010（LWT）/ADR-0009（稳定序号）
- 上游：reports\TASK-0048-report.md（状态规则数据源）+ reports\TASK-0041-report.md（ResultSink 桥接+Redis 订阅）+ calc/src/** 现有代码
- 权威：详细设计 §1.5B/§5 判定链路 + CONV-02 质量码 + st_item_value 契约

## 越界反查（协调者已比对，评审者复核）

- 23 文件全落 `calc/**`；区间仅一提交，无并行交错；backend/frontend/ingest/契约零触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **求值链顺序**：直绑短路 → LWT 离线优先 → priority 升序首命中（故障优先于运行）→ UNKNOWN+质量码 3——顺序是否与 ADR/详细设计 §5 一致；单规则失败隔离（bad rule 不影响其他规则求值）
2. **稳定序号**：状态编码字典单一来源（ADR-0009 防历史漂移）——核实无硬编码散落、序号→状态映射与 0048 规则表口径一致
3. **Redis 当前状态**：iot:st:{tenant}:{instance} key 形态；**未变不重写**与**失败不覆盖旧值**语义（live 3 次同状态零重写实证）；RESP2 客户端零依赖（0041 同款）
4. **st_item_value 留痕**：逐语句 REST INSERT...USING...TAGS（0029 实证的多语句信封静默空操作纪律）；value=稳定序号/quality=0/未变不重写；失败 WARN 不阻塞；**真 TDengine 写入未 live 验真（环境 REST 21s reset）——评估 3 项契约测试+假适配器捕获真实 INSERT SQL 的替代证据是否充分**，若不足判 Important 并说明需补什么
5. **LWT 离线输入**：订阅 iot:gwstatus:{tenant}（0029 发射侧拟定形态）——LWT OFFLINE 优先于规则推导的语义；离线状态是否污染正常判定（离线恢复后如何回到规则求值）
6. **ResultSink 桥接**：复用 0041 执行器，status 默认关闭对既有行为零变化（159 测试中 3 Skip=env 门控合理性）
7. **状态规则源 env 种子 JSON**：StatusRuleProvider 缝——冻结快照 §11 不装载 iot_status_rule（0024 演进位），评估该缝是否清晰、无契约突破、后续接真规则表的接缝可用
8. **测试有效性**：159 测试 0F（140 基线+19 新增）；live 冒烟 RUNNING→FAULT→OFFLINE→UNKNOWN 逐条与断言对应；3 处红转绿留痕

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0049-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。**特别对第 4 点（TDengine 真写入未验真的证据充分性）给出明确判定**。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
