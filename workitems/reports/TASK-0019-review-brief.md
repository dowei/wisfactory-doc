# 评审 brief：TASK-0019 diff 评审（安全敏感面加严）

## 角色

- 代码评审（标准任务验收的 diff 评审环节；**凭据域是全系统安全敏感面之首**，评审最严——重点明文泄露面与加密链完整性）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-diff.txt`（TASK-0019 单提交 2ff499b，22 文件约 2570 行；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（四管理接口契约/掩码/轮换/吊销/内部拉取/审计无明文硬验收）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0019.md`
- 实施报告（供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-report.md`
- 决策：`workitems\decisions\ADR-0006.md`（双权限先例/凭据不进快照）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13（安全敏感面从严）
- 评审两样：①与 brief 契约一致性——重点：列表永不回密文/明文（掩码实现）、view 单设权限+@OperateLog、轮换 key_version 递增不新增行、吊销立即生效（拉取即 403）、InternalAuthChecker 统一收口 /internal/v1/**（0017 占位完整化的正确性与既有调用面回归）、SM4 {v1} 前缀与 key_version、审计无明文 grep 断言的覆盖面（请求参数/响应/异常消息全查？）、菜单 4051-4052 双权限一一对应；②明显质量缺陷（错误处理、边界、安全——重点：**全泄露面扫描**：view/列表/拉取/审计/异常消息/日志每一处可能泄明文或密文的路径、SM4 加密失败的 fail-closed、掩码实现的信息泄露度（前四后四是否足够）、拉取接口的通道归属校验与租户隔离、key_version 溢出、并发轮换竞态、SQL 参数化、/internal 认证收口后 0017 既有心跳端点的回归影响）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
