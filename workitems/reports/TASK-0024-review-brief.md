# 评审 brief：TASK-0024 diff 评审（代码评审）

> 中断恢复接手轮：实现 ab9481a 已提交，本次评审为验收收尾的 diff 评审环节。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-diff.txt`（BASE=a1c6649，HEAD=8566f57；ab9481a=实现 28 文件 + 8566f57=gitignore，共 29 文件 +3780 行）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-brief.md`（接口与决策 9 条 + DDL + 测试口径都在其中）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0024.md`
- 关联（同库 workitems 下）：`features\FEAT-0008.md`、`requirements\REQ-0033.md`、`testcases\TC-0008.md`、`decisions\ADR-0008.md`、`decisions\ADR-0006.md`
- 快照 Schema 契约（字段级权威）：`D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md`（v1.0.1）
- 需要上下文时可直接读工作区源文件核对（`D:\workspace\WisFactory\backend\...`）

## 越界反查（协调者已比对，评审者复核）

- `git diff --name-only a1c6649..HEAD` 共 29 文件：28 文件全部落在 brief「改动范围」声明内（publish/main 20 + application.yml 1 + test 6 + upgrade SQL 1）。
- 声明范围外仅 1 个：`.gitignore`（提交 8566f57，TASK-0024 环境产物——backend/data 快照运行时目录入 gitignore，协调者已认可）——请核实其仅含 gitignore 增项。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. 快照组装严格按 config-snapshot-schema.md v1.0.1 字段级 Schema（camelCase / schema_version 一并下发 / 审计列剔除 / 五对象按 scope 装载）
2. 凭据排除双保险：白名单字段 + 组装后关键词终检 fail-closed（credential/password/secret/token 等，ADR-0006）
3. 版本单调：uk_scope_version 物理兜底 + 撞 uk 后的 5001 处理路径正确性（brief 允许重试一次或报冲突，评估所选路径）
4. 回滚=新版本行复用旧快照（先验后插、历史不改写；目标版本不存在 → 400）
5. 取回哈希核对（不一致 → 500 告警日志——存储损坏探测）
6. 确定性序列化（键序稳定、两次组装哈希一致）
7. PublishConfirmGate 接口 + NoOp 实现（javadoc 声明 TASK-0015 交付后接线义务）
8. 数据权限防线：网关 scope 可见性校验；scope_type=2 仅管理员语义
9. application.yml 仅新增对象存储配置键（ot.publish.* 命名空间），不动其他键
10. 升级脚本：p33 逐字 + uk_scope_version 差异登记头注 + 菜单种子 4070-4073 幂等 + 幂等二次运行

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行 或 diff 位置）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」「依据」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
