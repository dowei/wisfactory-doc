# diff 评审：TASK-0017（iot_gateway 网关 CRUD 与在线状态回写）

- 评审者：fresh reviewer（brief：reports/TASK-0017-review-brief.md；no-subagents；只评审不修改）
- 评审对象：TASK-0017-diff.txt（单提交 042d13f，26 文件 3154 行，已完整读入）
- 依据：L1-execute.md §13 + 实施 brief（TASK-0017-brief.md）+ REQ-0009 验收 1-5 + TASK 详情
- 交叉印证：实施报告（TASK-0017-report.md）；关键声明均经评审者独立复核（PDF 提取、代码库框架机制、守卫测试语义），不采信报告自述

## 一、契约一致性核对（brief 重点项，全部通过）

| # | 核对项 | 结论 | 证据 |
| --- | --- | --- | --- |
| 1 | DDL 与附录 A p30 逐字性 | **通过** | 评审者自行 pdftotext 提取 detail-design-v1.1.pdf p30：iot_gateway 十三业务列（名称/类型/默认值/次序）+ uk_tenant_gateway_code/idx_tenant_instance/idx_online 三索引与升级脚本逐字一致；MySQL 8 键长可行（64×4+8=264 字节 < 3072），头注「与上游差异：无」如实 |
| 2 | §3.1 p21 接口行数 | **通过** | PDF p21 实测 4 行（list/add/edit/remove，无 detail）——brief 记「五行」与 PDF 不符，按 PDF 落地 4 端点，菜单五行照 brief、query 行（4042）预登记不挂端点，差异已在报告 §0/§4.1 留痕 |
| 3 | gateway_code 字符集与唯一双道闸 | **通过** | @Pattern `[A-Za-z0-9_]+` + @Size 64 + 服务层 validateGatewayCode 双保险；双道闸 = 旁路预检（IotGatewayCodeMapper，@InterceptorIgnore + 显式 tenant_id，含已删行 + deleted 文案区分）+ DuplicateKeyException 兜底，均 4002——与 0012 IotInstanceCodeMapper 范式同构（已核对实例域源码，实现顺序「归属校验先于预检」的声明属实） |
| 4 | 回写字段组白名单强制 | **通过** | 三道闸齐备：控制器原始 JSON 层（顶层键 tenantCode/items、条目键 gatewayCode+五字段，越界键点名拒 PARAM_ERROR）→ typed 载荷（GatewayHeartbeatItem 仅六键）→ 服务层更新集仅五字段（单测逐字段 assertNull 固化）；REQ-0009 验收 4「越界 UPDATE 被拒」在三层均有断言 |
| 5 | InternalAuthChecker 实现 | **通过** | MessageDigest.isEqual 常量时间比对（现代 JDK 实现对不等长也不提前退出，无长度泄露）；配置空值 fail-closed 拒绝全部（单测固化）；IP 白名单默认 127.0.0.1 走代码默认值（brief「yml 仅新增一个键」约束被正确调和，报告 §4.2 留痕）；OtIpUtils.getClientIp 受 behind-proxy 开关控制（防伪造转发头口径核对 OtIpUtils 源码属实） |
| 6 | /internal/** 暴露面收窄 | **通过** | 全仓 grep：/internal/v1 下仅此一个端点（POST /internal/v1/gateway/heartbeat）；拦截器注册仅 /internal/v1/gateway/**；@Anonymous 方法级注解经 SaTokenInterceptorConfig 核实为框架认可的跳过口径（handlerMethod.hasMethodAnnotation）；拦截器 + 控制器双层显式 check（纵深防御，报告 §2.6-4 修复后认证先于载荷解析，顺序正确） |
| 7 | 删除 guard 扩展点 | **通过** | GatewayReferenceGuard 接口（countChannels/countPoints，注释即 0018/0021 接线依据：通道按 gateway_id 直引——与 PDF p30 iot_channel DDL 的 gateway_id 列吻合；点位归属列留给 0021 裁决）+ 占位恒零实现（TODO 登记）+ 调用点拒绝分支被单测固化（计数>0 即拒、不触 deleteById） |
| 8 | query 预登记的守卫影响 | **通过** | 评审者通读 SaPermissionRegistryConsistencyTest：断言为单向「注解 ⊆ SQL 登记」（且注册表含 upgrade/ 全部脚本）——预登记行无注解不触发守卫；域内 IotGatewayMenuPermissionGuardTest 双向核对并将「query 行预留不挂端点」显式断言固化（防误删），形态自洽 |
| 9 | 改动范围 | **通过** | git diff --name-only eb01ba2..042d13f 实测 25 文件（diff 包 26 项含 diff.txt 头部误计，实核 25）全落声明内；application.yml 仅 +2 行（唯一授权键及行内注释），无密钥入库（env 占位 `${OT_GATEWAY_INTERNAL_TOKEN:}`） |
| 10 | 多租户与定位语义 | **通过** | OtTenantLineHandler 核实：白名单式、iot_ 前缀自动纳入、无会话经 ignoreTable 短路跳过注入、sys_tenant 明确在忽略清单——服务实现「回写链路显式 (tenant_id, gateway_code) 条件 + 插件条件叠加只会更严」的防御纵深声明与框架机制吻合；回写租户按 body tenantCode 解析，不存在/禁用拒（单测+冒烟 S7e） |
| 11 | 其余 brief 项 | **通过** | heartbeat_period 默认 30 + 1-86400 边界（实测）；编辑改码拒（同值 no-op 放行）；删除软删；status 创建强制 1、online_status 创建强制 0；回写不挂 @OperateLog（单测断言注解缺席）、管理面三写操作挂 @OperateLog（单测断言）；菜单 SQL 列清单与 20261122_iot_template_domain.sql 逐字一致、4041-4045 与 0013 已占的 4040 无冲突、INSERT IGNORE 幂等 + role 1 授权；LWT/graceful 正确归 0020 |

框架机制侧证（评审者独立核实）：MetaObjectHandler（MybatisPlusConfig）无会话路径 updateFill 安全（getCurrentUserId 捕获异常返 null，prod 冒烟心跳 200 佐证）；分页插件 maxLimit 500 与回写批量上限 500 同口径。

## 二、发现清单

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| F1 | Minor | `IotGatewayInternalController.parseItem` | 数值字段类型归一化用 `intValue()/longValue()` 静默截断：`onlineStatus: 1.5` 落 1、`configVersion: 17.9` 落 17——值域 0/1 虽守住了，但畸形小数被静默纠偏而非拒绝，与「越界即拒」的接口从严口径不完全一致（建议非整数直接 PARAM_ERROR） |
| F2 | Minor | `StaticTokenInternalAuthChecker.checkIpWhitelist` | IP 白名单比对不对称：白名单条目小写归一、`OtIpUtils.getClientIp` 返回值未归一——IPv6 十六进制大写来源（behind-proxy 下经 XFF）对小写配置必不命中，方向是 fail-closed 403（可用性缺陷非安全漏洞）；clientIp 同样 toLowerCase 即对称 |
| F3 | Minor | `IotGatewayInternalController.heartbeat` | 批量体大小防线位置偏晚：单批 500 条上限在服务层、且在 Jackson 全量解析 body 之后生效，HTTP 层无体积上限——该面已过令牌+IP 双闸（暴露面为已认证内部调用方），风险可控，登记备忘（如需彻底封死可在 Filter 层按 Content-Length 预拒） |
| F4 | Minor | `IotGatewayServiceImpl.writeHeartbeats` | 心跳更新为逐条 UPDATE 循环（最多 500 条/事务，单条条件 id+tenant_id+gateway_code 走主键索引）而非批量形态——当前 30s 心跳节奏与 500 上限下可接受，规模上来后可改 CASE WHEN / JDBC batch 降往返；效率备忘不阻塞 |
| F5 | Minor | `IotGatewayServiceImpl.selectGatewayList` | 网关列表未接数据权限裁剪（同域实例列表有 VisibleScope/requireDataPerm，设计 §3 通用规范称未标注查询接口同样走数据权限）——租户隔离由插件保证、缺的是租户内部门级可见域；brief 决策清单未含此项且网关可见域语义无 ADR，报告 §4.5/§5.2 已登记待协调者裁决：**网关管理页对受限范围用户放行前必须先裁决此项** |
| F6 | Minor | `IotGatewayUpdateForm` / 报告 §1 | 编辑白名单含 status，超出 brief 枚举（gatewayName/instanceId/deptId/heartbeatPeriod/remark 类）——属合理扩展：REQ-0009 验收 3 的启停必须有写路径、§3.1 无 changeStatus 行，报告已留痕；登记为 brief-vs-实现差异供验收追溯 |
| F7 | Minor | `IotGatewayServiceImpl.writeHeartbeats` + `IotGatewayService` javadoc | 返回语义小偏差：返回 `items.size()`（条目数），javadoc 称「实际回写的网关数」——同码重复条目场景两者不等（接口文档口径与实现不一致，纯文案级，改 javadoc 即可） |

## 三、评审结论

- **Critical 0 / Important 0 / Minor 7**——无退回项（§12 修订轮不触发）；F1-F4、F7 为可选的后续微修（记 TASK `任务结果` deferred，随 0019/0020 邻域任务顺带处理即可），F5、F6 需协调者裁决/知悉（F5 有放行前置条件含义，见上表）。
- 五项验收的实测输出（1609=1551+58 全绿、脚本二次幂等、dev/prod 冒烟、范围核对）与 diff 实物相互印证，未发现报告失实声明；报告 §2.6 六红的根因修复（items 初始化、用例 stub、LIKE 断言、check 先于解析）逻辑自洽且均有单测固化。
