# 评审 brief：TASK-0015 scoped 复评 round2（代码评审）

> 修订轮复评（§13：复评是 scoped 的——只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为最终评审。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-diff-round2.txt`（e1b6266..cebf17a，单提交 8 文件）
- 原评审发现（复评基线）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-review.md`（I-1 模板域漂移盲区 / I-2 状态规则绕过留痕 / Minor ①token 不校验 ②撞版裸 500 ③ADR 列不可达 ④启停版本噪声）
- 修订轮报告段：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-report.md`「修订轮」段（修复清单+红绿证据+M-4 口径说明）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-brief.md`
- 需要上下文时直接读工作区源文件（D:\workspace\WisFactory\backend\...）

## 越界反查（协调者已比对，评审者复核）

- 修复 diff 8 文件：publish/gate 1 + statusrule/service 1（修订轮授权扩展）+ template/service·tier 3 + test 3——全落授权范围（原 brief 范围+statusrule 扩展）。

## 复评要点（只答三问）

1. **I-1 是否真修复**：复合凭据哈希（计算域快照摘要+模板域内容摘要，computeConfirmCredentialHash 预览侧/校验侧共用同参重算）——模板域任何变更（含逻辑删）是否必然哈希漂移→fail-closed 5002；有无残留「模板变更不改哈希」路径
2. **I-2 是否真修复**：statusrule 四写原语（create/update/saveBindingMap/deleteRules）同事务钩 recordContentChange——T2 状态规则变更是否必然产生版本行并触发确认门；deleteRules 按 distinct templateId 的正确性；钩子遗漏面复核（statusrule 域还有没有其他写路径）
3. **修复 diff 是否引入新 Critical/Important**：复合哈希的性能/正确性（同参重算两侧一致性）、M-1 四步校验畸形值 fail-closed、M-2 4002 映射、M-4 status 移出快照的语义一致性（启停零噪声且规则表 T1 映射防旧历史行误触二档）——只针对这 8 文件 diff 本身，不重开原评审已过部分

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0015-review-round2.md`。逐条回答三问（每条：结论+出处+一句话说明）；新发现按 Critical/Important/Minor 分级；三问全过且无新 C/I 时明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
