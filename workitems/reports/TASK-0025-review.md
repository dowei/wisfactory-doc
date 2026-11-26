# TASK-0025 diff 评审报告（代码评审，fresh）

> 评审日期：2026-11-23；评审者：fresh 评审子代理（只依据评审 brief「必读」清单与工作区源码判断，未向协调者追问）。
> 评审对象：实现提交 d08ec02（BASE=da830e4，单提交，24 文件 +2304/-133）；diff 包 `TASK-0025-diff.txt` 与工作区 `git diff da830e4..d08ec02` 逐文件一致（本次评审以工作区源码独立核对，不共享协调者上下文）。

## 一、越界反查复核

`git log` 实证：d08ec02 的父提交即 da830e4，单提交交付；`git diff --name-only da830e4..HEAD` 24 文件，与实施 brief「改动范围」声明（publish 域主代码 / test 域 publish 测试 / application.yml / 升级 SQL 1）逐一比对——**无越界文件**。application.yml 新增仅 4 键（`ot.publish.ack-scan-interval-seconds` / `ack-timeout-seconds` / `target-ingest-id` / `target-calc-id`），全部在 brief 允许键族内，无其他改动行。**复核通过。**

## 二、评审重点逐条核对（brief 11 条 + §13 常规）

1. **落行同事务性 — 通过。** `recordDispatch` 挂 `@Transactional`（join 既有事务），在 `IotPublishServiceImpl.publish`（:202）与 `rollback`（:303）版本落库后同事务调用；测试 `dispatchFailureFailsPublish` 锁定落行异常即发布整体回滚。目标标识来源核对：网关行= `IotGatewayMapper.selectById(scopeId)` 读 gateway_code（会话租户注入、缺失/空白 fail-closed 抛错），接入/计算行= `ot.publish.target-ingest-id`（默认 ingest）/ `target-calc-id`（默认 calc）；网关 scope 两行（1/网关+2/接入）、计算 scope 一行（3/计算），单测+dev 冒烟双向实证。
2. **ACK 语义 — 通过。** 逐条对照：ackStatus=1 任意旧态置 1（更新无 ack_status 前置条件，ack_time=NOW、error_msg 置 NULL）；=2 走「快路径终态检查 + 条件更新 `WHERE ack_status IN (0,2)`」双道保护，已生效不被倒退；=0 拒绝 400；行 (version_id, target_type, target_id) 不存在 400；versionId 与三元组同给时以 versionId 为准做一致性校验（三元组不全也 400）。**200+data=false 忽略语义判定为清晰合理**：过期失败回执是分布式常态竞态，按非错误返回避免内部调用方误重试，且实现者已在报告「偏离 3」与控制器/AckForm javadoc、单测四处留痕登记。
3. **超时扫描 — 通过。** `@Scheduled`（fixedDelayString + SpEL 毫秒换算 + initialDelay 防启动即扫）读 `ot.publish.ack-scan-interval-seconds`；`markAckTimeout` 选 `ack_status=0 AND create_time < now-ack-timeout-seconds`，逐行条件更新 `WHERE ack_status=0`——幂等（已置 2 不重复更新）、防与并发 ACK 生效竞态倒退（生效后 WHERE 不命中即跳过）；error_msg=`ACK 超时（超过 N 秒未回执）`，N 取配置值（dev 冒烟 N=20 实证）。扫描异常仅记 ERROR 不上抛，调度健康优先——合理。并发交互全排列核查：扫描↔生效、扫描↔重发、生效↔重发的条件更新守卫在各交错序下终态均一致，无倒退路径。
4. **重发 — 通过。** 2→0/0→0 重置（ack_status=0、error_msg=NULL、update_time 刷新）+ NotifySender 调用一次；ack_status=1 → 400「已生效无需重发」（快路径拒绝 + 条件更新 `WHERE ack_status != 1` 双道，并发生效竞态下拒绝不倒退）；targetId 多行命中 400。4074 种子 `INSERT IGNORE` 幂等 + 超管授权一行，守卫测试锁定 SQL 行与代码权限码五值集合。
5. **状态查询 — 通过。** `GET /iot/publish/log/list` 复用 `iot:publish:list`（不新增种子，与 brief 一致）；timedOut 派生=待确认且 create_time 已逾窗（覆盖扫描间隙，不改行）；publishState 三态汇总（存在失败优先 > 全部生效 > 下发中，空行集保守归 0 并已登记）——全程零版本行写操作（service 内 versionMapper 仅 select），TD-0034「不改 iot_config_version.publish_status」满足。
6. **租户旁路正当性 — 通过。** 实证 `OtTenantLineHandler` 对 iot_ 前缀表在会话有租户时注入 tenant_id，而本表无 tenant_id 列——不旁路必然 SQL 报错，旁路正当且照 `IotGatewayCodeMapper`/`IotDataItemRefMapper`/`IotChannelCodeMapper` 既有显式范式。服务层租户边界核实：管理面查询/重发先经 `IotConfigVersionMapper`（**未**加 InterceptorIgnore，租户注入生效）解析版本行——跨租户 versionId 解析不到即 400，跨租户三元组经注入天然过滤；再按 version_id 取旁路表行，不构成跨租户读写路径。内部 ACK 无会话租户（无注入）、定位靠内部调用方已知坐标 + 三元组歧义 400 防护；扫描无会话系统级全表（上游 DDL 天然语义）。旁路未引入新跨租户读写路径。
7. **PublishScopeGuard 上提 — 通过。** 归一/写防线/读防线/网关可见性四方法与 0024 原实现逐字一致（本次评审逐行比对 diff 移除段与新增类，发现仅一处文案差异，见发现 1）；0024 侧改纯委托、无逻辑残留（原私有方法连注释一并移除）。**旁证加强：本次评审独立重跑 publish 域测试（非采信报告输出）——IotPublishServiceImplTest 23/23 全绿，0024 既有路径行为不变成立。**
8. **内部端点鉴权 — 无洞。** 实证三件套：① `SaTokenInterceptorConfig` 对方法级 `@Anonymous` 仅跳过 Sa-Token 登录校验（且该端点无 @SaCheckPermission，注解鉴权拦截器无作用对象）；② `GatewayInternalAuthConfig` 独立注册 `InternalAuthInterceptor` 覆盖 `/internal/v1/**`，与 @Anonymous 旁路互不干扰；③ `StaticTokenInternalAuthChecker` fail-closed（令牌未配置一律 401 拒绝）、常量时间比较、IP 白名单 403，控制器内再显式 `check()` 一道（纵深防御）。单测锁定 checker 双调用与令牌拒绝时服务层零触达；dev 冒烟无令牌/错令牌 401、prod 未注入令牌 fail-closed 401 三态实证。与管理面权限（@SaCheckPermission）无混淆——守卫测试明确内部端点不在管理面权限集合口径内。
9. **application.yml 与 javadoc — 通过。** yml 仅 4 个允许键（逐行核对）。javadoc 半成品态已修：`PublishProperties.java` 现文为「扩展 ack- 与 target- 键族」（:8-9），内联 `{@code ot.publish.ack-*}`（:14-15）中 `*` 后随 `}` 不构成 `*/` 提前闭合——编译（2205 测试）与 prod 启动实证无残留语法错误。
10. **升级脚本 — 通过。** 建表语句与 brief 提取的 p33 DDL 逐字一致（列/类型/注释/双索引/表选项逐列比对）；头注两点在位（①不追加 uk_version_target 及其依据——同事务一次性写入+重发不新增行无并发重复路径；②target_id 语义）；4074 `INSERT IGNORE` + role 授权幂等，二次执行零变更实证（验收 2）。头注与「sql-exec.sh 行尾分号切分」兼容纪律（语句区无注释行）核对成立。
11. **5002 口径差异裁决 — 裁决成立。** 事实核验：`ot-core ResultCode` 现仅 200/400/401/403/404/405/500/4002/4003/5001——**确无 5002**，brief「ResultCode 已有」与事实不符（brief 笔误，非实现缺陷）。归属判断：0024 交付件 `PublishConfirmGate.java:17-18` 已明文登记「如需专用错误码 5002『下发未确认』，由 0015 在 ot-core ResultCode 登记扩展」——执行者沿用该在库先例，归属 0015 成立。且 FEAT-0008/REQ-0033 仅以业务规则形式提及「未确认 5002」，未绑定到任何 0025 接口；0025 各 400 路径（ackStatus=0 拒绝、行不存在、定位不一致、已生效重发）语义均为请求形态/状态机拒绝，本就不该用 5002——裁决整体成立。登记途径（报告登记、不动 ot-core 禁区）符合契约。

## 三、发现清单

无 Critical、无 Important。以下 Minor 6 条，按 §13 记收口 deferred，不进修订轮：

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1 | Minor | `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/support/PublishScopeGuard.java:82`（diff 中 IotPublishServiceImpl 移除段对应行） | 读路径计算 scope 的 403 文案由「计算范围版本/快照仅限全租户…」改为「计算范围版本/快照/下发状态仅限全租户…」——共享组件连带改了 0024 既有端点（list/snapshot）的用户可见错误文案，与报告/guard javadoc「行为逐字平移、行为不变」的表述不完全一致（行为语义不变，仅文案扩写；0024 测试未断言该文案故全绿） |
| 2 | Minor | `publish/config/PublishProperties.java:40` + `publish/notify/PublishAckTimeoutScanner.java:38` | `ackScanIntervalSeconds` 字段绑定后无任何读取点（扫描周期实际经注解 SpEL 直读原始键），周期默认值 60 在字段默认值与 SpEL 默认值两处重复——同义双源，改动其一即静默漂移（brief 只要求 fixedDelayString 读键，现状可用，属一致性债） |
| 3 | Minor | `publish/service/impl/IotPublishServiceImpl.java:202` + `IotPublishLogServiceImpl.recordDispatch` | 发布事务内 promote 已落正式名快照文件后若 `recordDispatch` 抛错（如网关 gateway_code 空白的 fail-closed 边界），版本行回滚而正式名文件残留为孤儿——同版本号下次发布按确定性文件名覆盖自愈，属文件/DB 非原子性的既有特征在 0025 新增的一个失败点（0024 两段式只保证并发竞态零残留，不覆盖事务回滚场景） |
| 4 | Minor | `publish/service/impl/IotPublishLogServiceImpl.java:97`、`publish/notify/NotifySender.java` | `notifySender.notify` 在发布事务提交前调用（NoOp 现状无害）——TASK-0026 接真实通知时若在事务内直发，下游收到信号即拉快照可能与未提交版本竞态（404/哈希不符），接缝 javadoc 未登记「须提交后生效（如 after-commit 事件）」的接线约束，建议 0026 brief 显式承接 |
| 5 | Minor | `publish/controller/IotPublishController.java:128` | resend 返回裸实体 `IotConfigPublishLog`（继承 BaseEntity，序列化含 createBy/updateBy/deleted 等内部审计字段），与同任务 log/list 的 VO 收口口径不一致（无敏感字段、无害，属响应形态一致性债） |
| 6 | Minor | `publish/service/impl/IotPublishLogServiceImpl.java:259、341` | `ack-timeout-seconds` 未做取值校验：配置 0/负值时 deadline 落在当前/未来，扫描会把全部待确认行立即置 2（操作员误配置的放大面）；负扫描周期则 Spring 启动即失败——均无 brief 义务，登记为配置健壮性观察项 |

## 四、无法判定项

- 升级脚本 DDL 与《详细设计》附录 A p33 **上游原文**的逐字一致性：上游文档不在本评审可达范围（brief 只贴了提取件），本次核对基准为「脚本与 brief 提取的 DDL 逐字一致」（已核对通过）——与上游原文是否有字形级出入**无法判定**，与验收 2 的表结构实测（idx_version/idx_ack 在位）不冲突。
- 2205 全量测试数：报告以原始输出为证，本次评审独立重跑范围为 publish 域 67 用例（全绿，见上）；全量 2205 未复跑，采信报告输出。

## 五、评审性验证（fresh 自做，非采信报告）

- git：d08ec02 父=da830e4 单提交；24 文件清单与 diff 包一致，全落 brief 声明范围。
- 独立重跑：`mvn test -pl ot-admin -Dtest='IotPublish*'` → guard 4 / controller 11 / internal 6 / logService 23 / publishService（0024）23，**67/67 全绿 0 失败 0 错误**（surefire 报告时间戳 2026-11-23 00:00 为本次运行）。
- 源码核对：SaTokenInterceptorConfig（@Anonymous 语义）、GatewayInternalAuthConfig（/internal/v1/** 拦截）、StaticTokenInternalAuthChecker（fail-closed/常量时间/IP 白名单）、OtTenantLineHandler（iot_ 前缀注入+无会话跳过）、IotConfigVersionMapper（无旁路→租户注入成立）、ResultCode（无 5002）、BaseEntity（审计填充/@TableLogic）、MybatisPlusConfig（update(null,wrapper) 不触填充、调度线程无 Sa-Token 风险）。

## 六、结论

**通过。** 评审重点 11 条全部核对成立，无 Critical / Important；Minor 6 条均为一致性债/边界观察项，不阻断收尾，按 L1-execute §13 记 TASK `任务结果` deferred（收口进 LOG）。5002 口径差异裁决（归属 TASK-0015、0025 各 400 路径本就不该用 5002）经 FEAT-0008 / REQ-0033 / 0024 交付件 javadoc 在库先例对照后**维持执行者裁决**；brief「ResultCode 已有 5002」确认为笔误。
