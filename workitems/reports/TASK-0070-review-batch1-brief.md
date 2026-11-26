# 评审 brief：TASK-0070 批次一 diff 评审（代码评审）

> 前端批次一（提交 d02ac44，BASE=f195b58，8 文件——0070 整任务分两批交付，本评审只覆盖批次一）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」；前端三门前置已由执行者实测——build:test/lint/typecheck 0 error + 真登录走查，评审者不复跑构建）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-diff-batch1.txt`（f195b58..d02ac44，8 文件）
- 批次 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-brief.md`（三页组接口决策/本批不做清单）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-report-batch1.md`（含两处 defineExpose 冒烟修复与偏离登记）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0070.md`
- 关联：workitems 库 `features\FEAT-0018.md`、`testcases\TC-0018.md`
- 后端契约源（核对页面调用与 API 形态一致性）：iot/publish/controller/IotPublishController.java（/iot/publish/log/list 响应形态）、iot/credential/controller/**、点位导入导出 controller
- 范式基准（同框架同组件要求）：frontend/src/views/iot/** 与 api/iot/** 的 0056 既有页

## 越界反查（协调者已比对，评审者复核）

- f195b58..d02ac44 共 8 文件全落 `frontend/src/**`，无越界。

## 评审重点（除 §13 常规质量缺陷外）

1. **API 契约一致性**：三页组调用的后端端点/参数/响应字段与 controller 实际形态逐一对齐（publish/log/list 的 ackStatus/timedOut/publishState 字段语义、resend 请求体、credential 掩码字段、导入回执形态）——字段名错位/语义误读是前端评审首要
2. **凭据安全口径**：页面任何形态（列表/详情/日志/错误提示）不出现密文；明文查看入口的权限隐藏（iot:credential:view）与二次确认交互
3. **三态/标红语义**：ack_status 0/1/2 徽标映射、timedOut 标红、publishState 汇总、已生效行重发置灰——与后端查询派生口径一致
4. **同框架同组件**：与 0056 既有页的组件/请求封装/路由挂载一致性（不自造请求层/UI 库）
5. **状态管理/内存**：抽屉组件 defineExpose（执行者冒烟修过 2 处——核修复完整）、组件卸载清理、分页参数
6. **导入向导交互**：逐行回执失败不中断、错误行下载、整批拒绝形态与后端回执对齐
7. 空态/加载态/错误态三态覆盖；乐观锁交互照 0056 范式

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0070-review-batch1.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
