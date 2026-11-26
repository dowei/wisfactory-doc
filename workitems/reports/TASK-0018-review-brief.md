# 评审 brief：TASK-0018 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；conn_config Schema 校验是配置正确性与凭据防线的关口，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-diff.txt`（TASK-0018 单提交 617b884，28 文件约 3000 行；协调者越界反查：25 文件落声明路径 + gateway 域 3 文件为 brief 明文要求的 guard 转正接线）
- 实施 brief（DDL 逐字/Schema 版本化校验裁决/凭据双保险/MODBUS-OPCUA 可建不可启用/Guard 转正）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0018.md`
- 实施报告（含校验库选型理由，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-report.md`
- 决策：`workitems\decisions\ADR-0006.md`（凭据独立存不进 conn_config）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：DDL 与 p30 逐字性、四个 Schema 文件的字段集与 additionalProperties:false + 凭据关键词双保险、protocol-Schema 匹配与未知协议拒、MODBUS/OPCUA status 强制 0、channel_code 网关内唯一双道闸 4002、gateway_id 不可改、删除保护双层（通道引用使删网关拒 + 点位引用占位 0021 接线）、菜单 4046-4050 一一对应、GatewayReferenceGuard 转正的调用面完整性（网关删除路径全部经 guard？）；②明显质量缺陷（错误处理、边界、安全——重点：Schema 校验的资源消耗（超大 conn_config 的 DoS 面——multipart/请求体上限）、校验失败的错误信息质量（指向具体字段）、guard 转正后网关删除的引用计数口径（活跃行）、SQL 参数化、事务边界）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0018-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
