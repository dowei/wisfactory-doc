# 评审 brief：TASK-0053 scoped 复评 round2（代码评审）

> 修订轮复评（§13：只看修复 diff 与原发现，不重开全场）。修订上限 1 退回+1 复评，本轮为终审。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」复评条款）

## 必读（只给路径，不粘正文）

- 修复 diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-diff-round2.txt`（a230f91..e9cc251 限 backend/，7 文件）
- 原评审发现（基线）：reports\TASK-0053-review.md（I-1 读口越权 + I-2 桥接缺口〔已拆 TASK-0084〕）
- 修订轮报告段：reports\TASK-0053-report.md「修订轮」段
- 实施 brief：reports\TASK-0053-brief.md
- 契约：`resources/contract/unified-message-model.md` v1.1.0「3.5」推送行
- 需要上下文时读工作区源码

## 越界反查

- 7 文件全落 backend openapi 模块；区间仅一提交；0063/0047 在途改动已隔离；calc/frontend/ingest/契约零触碰。

## 复评要点（只答三问）

1. **I-1 是否真修复（安全关键）**：事件读口是否真按验签身份的有效订阅派生授权集过滤环形缓冲（指标按 tenantCode、数据项按 (tenantCode,instanceCode) 归属）；无订阅/租户不匹配→空集 fail-closed；4 项越权红线测试（租户 B 读不到 A、无订阅读不到、同租户跨实例隔离）对旧实现必红；**摄入侧双校验与"非 hash 幂等"语义零改动**核实
2. **I-2 处置确认**：桥接生产者归 TASK-0084（已建档），本任务报告修订轮显式登记边界+补验条件，不改代码——核实登记清晰、无隐藏自造桥接
3. **无新 C/I**：2449 测试零回归（2445 基线+4）；主题归属记录的引入是否影响订阅注册/消费既有语义（全量测试佐证）；评审重点 3（幂等非 hash）与重点 7（环境证据）结论不因本修订改变

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-review-round2.md。逐条回答三问（结论+出处+一句话说明）；新发现按 C/I/Minor 分级；三问全过且无新 C/I 时明确「通过」结论。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
