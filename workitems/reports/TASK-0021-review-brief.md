# 评审 brief：TASK-0021 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；配额强控接线与数据权限裁剪同批，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-diff.txt`（BASE 46d9391 → a61fd24 合并包：主体 9a162a8（23 文件）+ 修订轮 a61fd24（0018 遗留 guard 接线授权）；协调者越界反查：全部落在 brief 声明范围 + redline 测试增补授权例外）
- 实施 brief（DDL 逐字/九接口/配额接线/unit 字典校验/F5 裁剪/建子表事件/TOCTOU 残余）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0021.md`
- 实施报告（含修订轮节——旧 jar 行为红实证，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-report.md`
- 决策：`workitems\decisions\ADR-0011.md`（TD-0045 单位字典）、`decisions\ADR-0016.md`（裁决 6/资产类 create_by 口径）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：DDL 与 p31 逐字性、九接口与 §3.1 点位段对齐、unit 强制字典校验（iot_unit 条目校验+归口提示）、**配额接线正确性**（PointQuotaCounter 活跃行 COUNT + add 前调用 + 4003；TOCTOU 残余已登记）、F5 裁剪同批（点位/网关列表条件形态、资产类 create_by 不适用口径、红线套件增补断言）、PointEnabledEvent 只发布不建监听、channel/point 双 guard 转正的调用面完整；②明显质量缺陷（错误处理、边界、安全——重点：双唯一并发兜底、启停事件与事务边界（afterCommit or 同事务）、裁剪条件的 NULL fail-closed、SQL 参数化、守卫测试覆盖 4053-4057）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
