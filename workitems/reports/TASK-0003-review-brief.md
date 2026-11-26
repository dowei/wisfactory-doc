# 评审 brief：TASK-0003 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0003-diff.txt`（BASE 17e09b1 → 88caf08，22 文件约 2100 行；协调者越界反查：21 文件在声明内，SysJobServiceImpl 2 行白名单登记为已申报偏离、协调者已追认——评审仍须核其改动质量与安全影响）
- 实施 brief（含接口契约与 mock 验收裁决）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0003-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0003.md`
- 实施报告（含根因链 R1-R8，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0003-report.md`
- 建库参数契约：`resources\contract\tdengine-super-tables.md`（附录 B 章——建库语句与超级表 DDL 应逐字对应）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` 的「diff 评审」口径（§13）
- 评审两样：①与 TASK 验收标准 / brief 接口契约的一致性——重点：库名派生唯一入口与安全校验、建库语句与契约附录 B 逐字性（KEEP 1825d/DURATION 10d/BUFFER 256、四张超级表 TAGS 含 tenant_id）、fail-soft 语义（失败映射业务异常、WARN 留 0、单租户失败不中断、防风暴节流）、开通挂点（db_created 强制 0 防.mass assignment、afterCommit 即时触发、status=null 视同启用）、配额校验边界（空/0 不限、4003 语义全局）、sys_job 白名单改动的最小性与契约测试；②明显质量缺陷（错误处理、边界、安全——重点：REST 实现的注入面（tenantCode 拼进 URL/DDL 的路径）、HttpClient 超时与资源泄漏、事务边界（afterCommit 与事务回滚交互）、白名单扩展是否破坏框架既有安全设计）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0003-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条）
- 一句话说明

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
