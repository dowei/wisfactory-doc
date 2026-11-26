# 评审 brief：TASK-0040 diff 评审（代码评审）

> 中断恢复收尾提交 04e6acd（BASE=537366e，49 文件全部 calc/**）——计算引擎服务引导：新顶层 Maven 工程 + 快照消费 + 依赖图 + 新图原子切换 + ACK 回执闭环。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0040-diff.txt`（537366e..04e6acd，49 文件——新工程全量首评）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0040-brief.md`（目录规划裁决/快照消费/依赖图/ACK/切换语义/测试口径）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0040-report.md`（含中断恢复说明/live 冒烟全序列/关键发现）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0040.md`
- 决策：workitems 库 `features\FEAT-0013.md`、`requirements\REQ-0020.md`、`testcases\TC-0013.md`、`decisions\ADR-0008.md`
- 契约：`D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md` v1.0.1
- 上游契约源（需要时读）：backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/controller/IotPublishInternalController.java（snapshot/ack 两端点）

## 越界反查（协调者已比对，评审者复核）

- 49 文件全落 `calc/**`（含 .gitignore/README/pom/src）——新顶层工程授权范围；未触碰 backend/frontend/ingest（执行者报告实证）。

## 评审重点（除 §13 常规质量缺陷外）

1. **快照消费与哈希**：HTTP 拉取+Basic 内部令牌+**原始字节哈希重算校验**（BUG-0005 修复后出站体=原始字节，重算 == snapshotHash）——fail-closed 路径；错形态/缺字段的防御
2. **依赖图构建**：节点/边构建（数据项/公式/引用边/指标五对象消费）；**失败主因全覆盖**：悬空引用/循环/实例化失败（白名单语义：DELTA 可引用/INTEGRAL 不开放）；空快照处理；NodeKind/EdgeType 映射正确性
3. **新图原子切换**：GraphHolder/AppliedGraph 的引用替换语义——构建成功才切换、失败保持旧图持续可用（FAIL 路径不得清空旧图引用）；并发读图与切换的线程安全
4. **ACK 回执闭环**：POST /internal/v1/config/ack 的 targetType=3/targetId=calc（与 0025 派生口径一致）+ ackStatus=1/2+errorMsg；**构建成功/失败两条路径均发 ACK**；失败时 errorMsg 含可诊断信息；ACK 自身失败的处理（fail-soft 不应影响图状态）
5. **配置纪律**：端口 8095（不占 8098/8099/8090）；env 键完整；config.changed 订阅缝占位（TD 登记）与定时对账循环；优雅停机
6. **测试有效性**：115 测试零回归；Contract 命名契约测试（fixture 取自 v1.0.1）；live 冒烟证据（ACK id=21 成功/id=22 失败双落库）与单测断言的对应关系
7. **新工程质量**：pom 依赖最小化（Jackson 3 纪律 TD-0058/Java 21）；README 体例（照 ingest）；.gitignore 覆盖 target/；配置默认值安全性
8. **执行者登记项评估**：悬空引用 R2_DEAD2→R2_DEAD1 遗留快照 fail-closed（正确行为）；v41 ACK 首次未受理疑为 backend 超时扫描并发窗口——评估是否需要新增缺陷档案或登记观察即可

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0040-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
