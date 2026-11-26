# 评审 brief：TASK-0025 diff 评审（代码评审）

> 实现提交 d08ec02（BASE=da830e4，单提交，24 文件 +2304/-133）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0025-diff.txt`
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0025-brief.md`（接口与决策 8 条 + DDL + 测试口径）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0025-report.md`（含其登记的 5002 口径差异——见评审重点 11）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0025.md`
- 关联：workitems 库 `requirements\REQ-0033.md`、`testcases\TC-0008.md`、`features\FEAT-0008.md`、`decisions\ADR-0006.md`
- 上游交付参照：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0024-report.md`；上游代码 D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\iot\publish\**（0024 部分按需对照）
- 需要上下文时直接读工作区源文件核对

## 越界反查（协调者已比对，评审者复核）

- `git diff --name-only da830e4..HEAD` 共 24 文件，全部落在 brief「改动范围」声明内（publish/main 17 + application.yml + test 6 + upgrade SQL 1），无越界文件。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. 落行同事务性：发布/回滚版本落库后同事务插 publish_log（网关 scope 两行〔1/gatewayCode+2/ingest〕、计算 scope 一行〔3/calc〕）；目标标识来源与配置键
2. ACK 语义与 brief 逐条一致：ackStatus=1 任意旧态可置 1（ack_time=NOW+error_msg 清空）；=2 仅可从 0/2 覆盖且已生效终态不倒退（200+data=false 忽略语义是否清晰合理）；=0 拒绝 400；行定位 (version_id,target_type,target_id) 不存在 400；versionId 与三元组同给时一致性校验
3. 超时扫描：@Scheduled（ot.publish.ack-scan-interval-seconds）+ 逾窗 0→2 + error_msg 超时文案 + 按行条件更新幂等/防竞态（并发 ACK 与扫描竞争时的正确性）
4. 重发：2→0/0→0 重置 + NotifySender 调用 + ack_status=1 → 400；4074 权限码与种子幂等
5. 状态查询：GET /iot/publish/log/list 复用 iot:publish:list；timedOut 派生覆盖扫描间隙；publishState 三态汇总（全部生效/存在失败/下发中）不改版本行（TD-0034）
6. **租户旁路正当性**：上游 DDL 无 tenant_id → mapper @InterceptorIgnore(tenantLine) ——核实服务层是否先按会话租户解析 version 行（旁路不得引入跨租户读写路径）
7. **PublishScopeGuard 上提**：0024 的 scope 防线提取为共享类——核实对 0024 既有路径零行为变更（0024 测试仍在 2205 全量内通过为旁证）
8. **内部端点鉴权无洞**：/internal/v1/config/ack @Anonymous + 显式 checker 双调用——核实 @Anonymous 只绕 sa-token 会话鉴权、静态令牌校验确实生效（无令牌 401/403 实证），不与管理面权限混淆
9. application.yml 仅新增 ot.publish.ack-*/target-* 键；javadoc 无残留语法错误（核验 agent 曾观测 `ack-*/target-*` 提前终止注释的半成品态——确认已修）
10. 升级脚本：p33 iot_config_publish_log 逐字 + 头注两点（不追加 uk 的论证/目标标识语义）+ 4074 幂等
11. **5002 口径差异裁决复核**：执行者登记「brief 称 ResultCode 5002 已有，实测无；本任务接口全走 400；5002 义务归 TASK-0015」——对照 FEAT-0008 业务规则「未确认 5002」与 ADR-0008 确认门语境，判断 5002 归属 0015 是否成立、0025 各 400 路径是否本就不该用 5002

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0025-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行 或 diff 位置）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
