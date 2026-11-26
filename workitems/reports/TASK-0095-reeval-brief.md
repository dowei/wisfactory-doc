# scoped 复评 brief：TASK-0095 评审退回修复

## 角色

- 代码评审（scoped 复评：只看修复增量与原 Important 发现，不重开全场）

## 背景

- 首轮 fresh 评审（`reports/TASK-0095-review.md`）发现 1 项 Important：新增态 `metric-form-modal.vue` 的 `loadItemOptions()` 只在 `if (rowData)` 编辑分支调用，新增态无法加载「来源数据项 / 比值分子 / 比值分母」下拉，aggType=5 比值型指标无法直接新建。
- 协调者按协议原样退回实现者修复一轮。实现者已提交修复 `57804b47`（HEAD），增量仅 2 文件。

## 必读（只给路径，不粘正文）

- 本 brief：`reports/TASK-0095-reeval-brief.md`
- 原评审发现：`reports/TASK-0095-review.md`（Important 段）
- 修复增量 diff：`reports/TASK-0095-fix-diff.txt`（09ae85cc → 57804b47，2 文件）
- 修复后报告：`reports/TASK-0095-report.md`（“评审退回修复”小节 + 修复后四门原始输出）

## 复评范围（只看这两点）

1. **修复是否真正解决原 Important**：
   - 新增态（无 rowData）是否现在能加载数据项下拉选项；
   - 是否有 instanceId 变化时的重载/清空失效引用（`onInstanceChange` / watch 等）；
   - aggType=5 时用户能否在新建表单中选择分子/分母并成功提交，不再需要“先建非比值型再编辑”。
2. **修复是否引入新问题**（仅限本增量 2 文件）：
   - 是否造假数据 / 调用错误端点；
   - 是否破坏编辑态原有加载路径；
   - defineExpose、权限、校验有无回退。

## 依据

- `.agents/protocol/L1-execute.md` §13：复评是 scoped 的，只看修复 diff 与原发现，不重开全场；1 轮退回 + 1 轮复评为上限。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0095-reeval.md`。给出：
- 原 Important 是否 resolved（resolved / unresolved + 证据）
- 新引入问题（Critical / Important / Minor，或明确无）
- 一句话结论：可收尾 / 需进一步处理

## 契约

- **fresh**：只依据上面「必读」路径判断，不向任何人追问；判断不了标「无法判定」。
- **no-subagents**：不得再派 subagent。
- **只评审不修改**：不改任何文件。
