# 评审 brief：TASK-0036 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；循环依赖检测与引用边维护是计算正确性核心，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0036-diff.txt`（TASK-0036 单提交 9e73d3e，19 文件约 2840 行；协调者越界反查：全部落在 brief 声明范围 dataitem 域）
- 实施 brief（三类配置契约/校验链/ref 先删后插/继承链扩展/impact 反查/汇总与 DELTA 占位口径）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0036-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0036.md`
- 实施报告（含 WSL 空闲终止事件偏离登记，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0036-report.md`
- 语法口径权威：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/RelativeRefExpander.java`（0013）；决策 `workitems\decisions\ADR-0008.md`/`ADR-0009.md`/`ADR-0010.md`

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：三类配置接口（binding/formula/rollup/thresholds）语义与 §3.1 对齐、公式校验链完整（Expander 同口径 + 引用存在性 + **循环依赖 DFS 跨实例正确性**——PARENT 边向上+绝对键，环上 id 列表进 4001 文案）、ref 先删后插事务内重建（换绑/清除清旧边）、继承链扩展纳入 source_type/formula/rollup/calc_period、impact 正反向+指标边空集占位+裁剪、汇总动态纳入口径与缺失打码 7 声明位、DELTA/INTEGRAL 函数白名单占位、DataItemConfigChangedEvent 只发布；②明显质量缺陷（错误处理、边界、安全——重点：DFS 的环检测完备性（自环/父环/绝对键跨层环/深图栈溢出）、并发配置变更竞态（两人同时改同一项 formula 的 ref 重建交错——先删后插的原子性）、SQL 参数化、0057 前端契约（impact 响应形状与前端占位按钮期望的匹配））

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0036-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
