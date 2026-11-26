# 评审 brief：TASK-0015 diff 评审（代码评审）

> 实现三提交 fa064cd/83b99c3/b3d1e0c（BASE=f195b58，前端批 d02ac44 在两 BASE 之间——diff 包已按净足迹隔离），29 文件 +903 行级规模，发布确认门真接线。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-diff.txt`（隔离区间 d02ac44..HEAD，29 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-brief.md`（DDL/分档逐档结论/确认协议/测试口径）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-report.md`（含三提交拆分、T2「待确认」窗口关闭语义登记、偏离回退记录）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0015.md`
- 决策权威：workitems 库 `decisions\ADR-0008.md`（三档定义逐条）；关联 `features\FEAT-0005.md`、`requirements\REQ-0006.md`、`testcases\TC-0005.md`、`features\FEAT-0008.md`
- 接缝与上游源码（需要时直接读）：iot/publish/gate/PublishConfirmGate.java（签名契约）、iot/publish/service/impl/IotPublishServiceImpl.java（gate 调用点）、iot/template/**
- 上游报告：`reports\TASK-0024-report.md`（gate 占位语义/5002 登记义务原文）

## 越界反查（协调者已比对，评审者复核）

- 隔离区间 29 文件全落 brief「改动范围」声明内（template/** + publish/gate + IotPublishServiceImpl 最小适配 + ot-core ResultCode 一处 + application.yml 一键 + SQL 一份 + 测试）；执行者曾误改 IotPublishController javadoc 并**自行回退**（b3d1e0c）——复核回退干净、无残留。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **分档引擎 vs ADR-0008 逐条一致性**：T1/T2/T3/FORBIDDEN 的字段级映射是否与 ADR 三档定义逐条吻合（新增/展示属性=T1；公式/聚合/周期/有效范围/告警阈值/单位=T2；删项/改类型/改取值方式=T3；data_code=禁止 400）；表驱动可演进性
2. **逐条目判档**：diff 引擎增/删/改三形态、不取最高档合并——混合保存的档位独立性
3. **版本历史钩子**：模板头/数据项写路径同事务自动升版——钩子覆盖面是否完整（列出所有模板写路径并核对有无漏钩：创建/更新/数据项增删改/删除模板等）、基线/补录/无变化零噪声语义、与 uk_template_version 的并发安全（同模板并发保存撞键处理）
4. **影响预览**：affected/locked 分列口径（引用+启用+锁定语义 vs ADR-0008 锁定不跟随）；previewToken 锚定（Redis 键/TTL/内容 SHA-256 一致性校验）
5. **gate 真实现**：TemplateAwarePublishConfirmGate 替换 NoOp 的装配方式（NoOp 删除还是条件化——false 路径行为是否保持）；签名不动+五参重载的兼容性；**网关 scope 恒放行**语义评估（scope_type=1 发布不涉模板档位——该放行理由是否成立）；仅二档触发、三档不经本门（快照按 applied_template_version 组装天然不包含未升级实例——与 0024 组装逻辑交叉核实）
6. **T2「待确认」窗口关闭语义**（执行者登记取舍：「已经过一次计算域发布」关闭、无 schema 变更前提）——评估该取舍的自洽性与漏确认风险（发布后模板又改但未再发布期间，下一次发布是否正确要求确认）
7. **5002 登记**：ot-core ResultCode 仅此一处触点；5002 与确认门拒绝路径的完整闭环（BusinessException→响应码）
8. 回归面：0024/0025/0026 既有发布测试全绿（2286 全量内）；NoOp 替换对一档/无模板变更路径零行为变化
9. SQL：iot_model_template_version 与 brief DDL 逐字 + uk_template_version + 头注差异登记；yml 仅 confirm-token-ttl-seconds 一键

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行 或 diff 位置）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
