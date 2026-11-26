# 评审 brief：TASK-0026 diff 评审（代码评审）

> 实现提交 2e4a1b9（BASE=d08ec02，单提交，21 文件 +2054/-41）——发布链收官环。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0026-diff.txt`
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0026-brief.md`（含 §3.2/§3.3 上游规范提取与接口决策 7 条——决策 4 有 0025 评审 Minor ④ 接线约束）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0026-report.md`（含 Jackson 3 发现与 TD-0056 登记文案）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0026.md`
- 关联：workitems 库 `requirements\REQ-0033.md`、`testcases\TC-0008.md`、`features\FEAT-0008.md`、`decisions\ADR-0006.md`、`decisions\ADR-0008.md`
- 上游报告：`reports\TASK-0024-report.md`、`reports\TASK-0025-report.md`；0025 评审 `reports\TASK-0025-review.md`（Minor ④ 原文）
- 需要上下文时直接读工作区源文件核对

## 越界反查（协调者已比对，评审者复核）

- `git diff --name-only d08ec02..HEAD` 共 21 文件，全部落 brief「改动范围」声明内（pom.xml 1 + publish/main 13 + application.yml + test 5 + 无 SQL——与「无新 SQL」声明一致），无越界文件。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **afterCommit 裁决落地正确性**（0025 Minor ④ 承接，brief 决策 4）：事务激活时延迟到 afterCommit 发送、无事务立即发送——核实 TransactionSynchronization 注册正确（同步状态、异常路径不吞不重复）、javadoc 双处登记存在
2. **MqttNotifySender**：topic 映射（网关→`iot/{tenantCode}/{gatewayCode}/config`、接入/计算→共享 `iot/config.changed`）与 gatewayCode/tenantCode 取数正确；载荷 §3.3 五键逐字；QoS 1/retain=false；fail-soft（port 异常→WARN 不阻断发布事务）；懒建立+重连+句柄复用无连接泄漏
3. **条件装配边界**：`ot.publish.notify.enabled=true` 才替换 NoOp——false 路径行为与 0025 完全一致（回归）
4. **snapshot 端点**：最新/指定版/不存在 400；哈希核对复用 0024 路径（不一致→500 告警）；**出站凭据终检对 Jackson 3 节点生效**（命中转 500 告警 fail-closed）；跨租户可选 tenant 参数、缺省歧义 400 fail-closed
5. **changes 端点**：五段 diff 摘要正确性（added/removed/changed 按对象对齐）；changes-max-span 超限/中间快照缺失/基线缺失→needFullSync=true 只给 latest；since≥latest 空数组
6. **Jackson 3 修复的范围控制**：执行者报告的「出站 JsonNode 须 Jackson 3 节点」修复——核实改动是否被正确限定在出站快照组装处（未引入全局序列化行为变化、未影响其他端点的既有 JSON 契约）；这是框架级发现，评估是否值得登记 TD/ADR
7. pom.xml 仅新增 paho 1.2.5 显式钉死；yml 仅 changes-max-span+notify.* 8 键
8. 回滚链：未新增代码路径（仅冒烟证据）——核实 service impl 改动未动回滚语义
9. TD-0056 登记文案与「通知载荷 version=业务版本号非 versionId」口径登记是否已在报告/Javadoc 留痕清晰

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0026-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行 或 diff 位置）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
