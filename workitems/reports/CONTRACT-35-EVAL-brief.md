# 实施 brief：§3.5 推送载荷契约评估与组稿（读类批次，不改代码）

## 批次

- 契约评估批次（共 1 个 subagent；产物=评估结论 + 视结论的契约增补稿）
- BASE：`04fc3a5`
- 改动范围：
  - `resources/contract/unified-message-model.md`（仅当结论为「并入」时升 v1.1.0 增补；结论为「不并入」则不动）
  - 新增 `resources/contract/push-payload-evaluation.md`（评估记录，无论如何都要写）
  - 报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\CONTRACT-35-EVAL-report.md`
  - 其余一律不碰

## 背景（只给路径，不粘正文）

- 触发：LOG-0004 遗留问题 3——详细设计 §3.5 的「数据项订阅推送」（消息体即统一消息模型）与「指标推送」（在统一消息模型上扩展 9 个字段）是跨系统契约面，前批组稿未纳入；TASK-0027 起（采集接入与数据服务联调）前需要结论。
- 现有冻结契约（v1.0.0，勿破坏既有章节）：`resources/contract/unified-message-model.md`
- 内容来源：`D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-detail-design-v1.1.pdf` §3.5（约 p26；pypdf 提取，字形规范化不改语义——⻅→见、⼊→入 等；提取方法参考 `reports\CONTRACT-FREEZE-brief.md`）
- 约束：`workitems\conventions.md` CONV-03（契约变更必须升版本并通知各系统）

## 任务步骤

1. 提取 §3.5 全文，识别两个推送面的完整定义（主题/方向/载荷/频率/订阅关系）。
2. 对照 unified-message-model.md v1.0.0 判定：数据项订阅推送是否已被统一消息模型完整覆盖（逐字段比对）；指标推送扩展的 9 个字段是否与既有字段冲突（同名异义/类型冲突）。
3. 出结论（三选一并给理由）：A 已完整覆盖、无需增补；B 需在统一消息模型增补一章（升 v1.1.0，附 §3.5 原文与差异声明）；C 需独立契约文件（给出文件名与结构建议，本批只评估不建——若为 C，先停，写评估记录等协调者裁决）。
4. 若结论 B：按 v1.0.0 的头部格式升版本（变更规则：增章=次版本 +1 → v1.1.0；「与上游差异」节注明新增来源章节；正文仅增不改既有章）。
5. 评估记录 push-payload-evaluation.md：结论、比对表（字段级）、理由、对 TASK-0027/0034 等接入与数据服务任务的影响说明。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\CONTRACT-35-EVAL-report.md`。五项：改动文件清单 / 提取核对结果（来源页码、token 回核）/ 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；只写改动范围列出的文件；原文提取优先于改写，拿不准进「与上游差异」不猜。
- 不得降低既有冻结章节的版本或内容（CONV-03）。

## 收尾回写

- 无 TASK 对象（协调动作批次），报告即终稿。
