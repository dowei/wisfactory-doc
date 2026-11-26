# 评审 brief：TASK-0051 diff 评审（对外面——全系统暴露面之首，评审最严）

## 角色

- 代码评审（标准任务验收的 diff 评审环节；**/openapi/v1 是全系统唯一对外暴露面**——签名/scope/限流/审计四链一体，评审最严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-diff.txt`（TASK-0051 单提交 8348047，46 文件约 4810 行；协调者越界反查：全部落在 brief 声明范围 openapi 域 + ClientScope 接线 + redline 转正授权例外）
- 实施 brief（鉴权过滤器契约/secret_encrypted 与上游差异裁决/错误码映射裁决/限流/六接口/审计）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0051.md`
- 实施报告（含两条上游差异列登记与 BASE 漂移切分核对，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-report.md`
- 决策：`workitems\decisions\ADR-0013.md`（A7 IP 白名单）、`ADR-0016.md`（裁决 5 scope JSON）、`ADR-0009.md`（TD-0031）、组件 `iot/dataperm/**`（ClientScope 三态）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13（对外面最严）
- 评审两样：①与 brief 契约一致性——重点：签名链完整性（HMAC-SHA256 串形态/防重放 300s/bodyHash/常量时间比对）、secret_encrypted 双列口径（与上游差异登记的必要性——secret_hash 不可逆无法验签的论证）、错误码映射（认证类 HTTP 403 vs 授权类 body 6003 的边界一致性——每个 403/6003 出口归类正确）、限流 Redis 固定窗（键形态/并发）、scope 裁剪走组件（三态消费不重写）、六接口响应形状与 §3.5 逐行对照、dict/sync 占位、审计异步写与 403 类全量（含失败也记？）、限值契约（pageSize 500/跨度 90 天）；②明显质量缺陷（错误处理、边界、安全——重点：**泄露面全扫**（错误回显/日志/审计脱敏）、鉴权过滤器路径覆盖（/openapi/v1/** 通配 vs 精确——openapi 内有无未设防路径）、时间戳校验的时钟偏移容忍、Redis 不可达时限流 fail-open/closed、审计异步丢失面、SQL 参数化、测试断言真实性）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
