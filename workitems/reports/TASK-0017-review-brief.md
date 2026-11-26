# 评审 brief：TASK-0017 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；内部回写接口是接入服务的写入面，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0017-diff.txt`（TASK-0017 单提交 042d13f，26 文件约 3150 行；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（DDL 权威/回写字段组白名单/InternalAuthChecker 占位/删除 guard 结构）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0017-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0017.md`
- 实施报告（含 §3.1 四行/五行差异登记，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0017-report.md`
- 契约：REQ-0009 验收 1-5（回写字段组固定/越界拒/LWT 与 graceful 区分归 0020）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：DDL 与附录 A p30 逐字性、gateway_code 字符集（MQTT 主题段）与唯一双道闸、回写字段组白名单强制（五字段外请求体直接拒）、InternalAuthChecker 拦截器实现（静态令牌比对常量时间？、IP 白名单语义、无 Sa-Token 暴露面的路径收窄——/internal/** 是否只注册该一个端点）、删除 guard 占位接口的扩展点清晰度（0018/0021 接线依据）、query 权限行预登记未挂端点的守卫影响（SaPermissionRegistryConsistencyTest 对注解⊆SQL 单向——预登记行无注解是否影响守卫断言）；②明显质量缺陷（错误处理、边界、安全——重点：内部接口的批量体大小防线、心跳更新的 SQL 效率（批量 update 形态）、tenantCode 解析的注入面、auth 拦截器的路径覆盖（/internal/** 通配 vs 精确）、HTTP 状态码语义（401/403 分野））

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0017-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
