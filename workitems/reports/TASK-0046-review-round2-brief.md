# 评审 brief：TASK-0046 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-diff-round2.txt`（f8421c7..ebd443d 限 calc/，9 文件）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-review.md`（F-01 级联未复用拓扑序 + F-02 载荷字段错〔我的 brief 笔误已纠正〕+ F-03~07 Minor）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-report.md`「修订轮」段
- 实施 brief（**决策 4 已纠正为冻结推送契约 9 扩展字段**）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-brief.md`
- 契约权威：`resources/contract/unified-message-model.md` v1.1.0「3.5」推送行（9 扩展字段：metric_code/instance_key/period/window_start/window_end/value/yoy/mom/quality）
- 需要上下文时读工作区源码（D:\workspace\WisFactory\calc\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 9 文件全落 calc/**（graph 公开化两静态方法零行为改动 + metric 域 + event + 测试）；区间仅一提交无交错；backend/frontend/ingest/契约零触碰。

## 复评要点（只答三问）

1. **F-01 是否真修复**：MetricCascade 把指标定义映射为 0040 图（METRIC 节点+METRIC_SOURCE 边）、拓扑序复用 GraphBuilder.topoSort、从下级 materialized 读值——三级链 M_LOW←M_HIGH←M_TOP 端到端测试对旧实现必红（上级 null）→ 绿（均 7）；**GraphBuilder 两静态方法公开化是否真零行为改动**（逐行核对）；每层 try/catch 失败隔离不破坏既有语义
2. **F-02 是否真修复**：MetricProducedEvent 补齐 9 扩展字段（quality 正常 0/不完整 7、instanceKey=full_key、instanceCode 保留过渡）——**逐字段对照 unified-message-model.md v1.1.0 §3.5 推送行**是否齐备（含 quality）；javadoc 的"camelCase 最终归 0053 定版"与"0053 按指标全键+窗口幂等非 hash"是否如实登记；旧载荷对 9 字段断言必红成立
3. **修复 diff 是否引入新 Critical/Important**：244 测试零回归（240 基线+4）；F-03 注释订正/F-04~07 deferred 留痕真实；EMQX 环境替代证据结论不因本修订改变；级联图构造对 0040 图结构零破坏

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0046-review-round2.md`。逐条回答三问（结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
