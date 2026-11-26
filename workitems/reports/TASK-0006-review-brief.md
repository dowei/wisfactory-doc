# 评审 brief：TASK-0006 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；缓存与失效的正确性直接影响权限时效，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-diff.txt`（TASK-0006 单提交 db16079，22 文件约 1930 行；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（失效即失效模型/SCAN+DEL/afterCommit 挂点/cache-enabled 开关/四变更点）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0006.md`
- 实施报告（供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-report.md`
- 组件契约：`workitems\decisions\ADR-0016.md`（裁决 5/6 消费口径）+ REQ-0002 验收「改绑定/角色后权限即时生效」

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：失效即失效模型（SCAN 非 KEYS、按 userId/全量两粒度、roleVer 占位 0）、四变更点挂 afterCommit（事务提交后发布——幻影失效面与漏失效面的边界）、cache-enabled 开关与 NoOp 兜底、resolve 先缓存后计算接线（未命中回填）、TTL 600s 可配；②明显质量缺陷（错误处理、边界、安全——重点：JSON 序列化的 VisibleScope 往返完整性（Set 序列化/反序列化）、SCAN 游标完整性（大键空间遍历不漏）、失效事件在事务回滚时的行为、监听器异常不炸主流程、Redis 不可达时的降级路径（fail-open 还是 fail-closed？与 M-1 TenantContext 语义的一致性）、变更点遗漏面（还有哪些写路径改了授权/部门/绑定却没挂失效——grep 交叉核））

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0006-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
