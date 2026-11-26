# TASK-0018 diff 评审报告（fresh reviewer）

- 评审对象：TASK-0018（iot_channel 通道 CRUD 与 JSON Schema 校验），提交 `617b884`，diff 包 28 文件约 3000 行（25 文件落 brief 声明路径 + gateway 域 3 主文件为 brief 明文要求的 guard 转正接线 + 1 个 gateway 测试同口径调整，越界反查通过）
- 评审依据：实施 brief 契约逐条核对 + L1-execute §13 两样（契约一致性 / 明显质量缺陷）；口径从严（conn_config Schema 校验为配置正确性与凭据防线关口）
- 交叉印证材料：TASK-0018-report.md（验收输出）、ADR-0006、TASK-0018.md

## 一、brief 契约一致性核对（结论）

| 契约项 | 结论 |
| --- | --- |
| DDL 与附录 A p30 逐字 | 通过——列/索引/引擎/注释逐字一致，`CREATE TABLE IF NOT EXISTS` 为 brief 自身要求的幂等适配 |
| 四个 Schema 文件字段集 + additionalProperties:false + 凭据双保险 | 基本通过——mqtt/http v1 字段集与 brief 一致且 additionalProperties=false；modbus/opcua 宽松骨架照 brief 登记；凭据关键词双保险实现为递归全层级（强于 brief 顶层口径）。**但双保险存在数组形状漏洞（I-2）、required 判定存在显式 null 漏洞（I-1），见发现清单** |
| protocol-Schema 匹配与未知协议拒 | 通过——protocol_type 决定加载 `<协议>-v1.json`，资源缺失即拒；服务层字典闸（大小写敏感四值）先行（见 M-2 大小写接缝） |
| MODBUS/OPCUA 可建不可启用（status 强制 0） | 通过——创建/编辑双路径均强制落 0，单测与冒烟 S5/S6h-S6i 固化；响应标注存在覆盖缺口（M-1） |
| channel_code 网关内唯一双道闸 4002 | 通过——含已删行旁路预检（显式条件 tenant/gateway/code，文案区分活跃/已删占用）+ DuplicateKey 兜底，冒烟 S4a 实证 4002 |
| gateway_id / channel_code 不可改 | 通过——携带不一致即拒、同值 no-op 不拦，单测固化 |
| 删除保护双层 | 通过——网关删除拒（活跃行计数 + 文案提示通道编码 + 空编码表防御分支）；通道删除点位 guard 占位就位且拒绝分支被单测固化（0021 接线义务登记在案） |
| 菜单 4046-4050 一一对应 | 通过——SQL 五行（parent 4013，幂等 + 超管授权）与五端点权限码双向核对由守卫测试锁定 |
| GatewayReferenceGuard 转正调用面完整性 | 通过——全仓反查：网关删除仅 `IotGatewayController → deleteGateway` 一条路径且经 guard；占位实现按自身契约删除，新实现 countChannels/channelCodes 实查活跃行（@TableLogic + 多租户插件），拒绝语义与调用点未变 |
| 校验库二选一 | 通过——手写最小校验器，理由在报告偏离 2（Schema 文件为标准 draft-07 子集，二期可切标准库）；但手写实现的两个漏洞（I-1/I-2）即该选型的质量代价，须修复 |
| 未做项与偏离 | 通过——5 条未做项与 6 条偏离均有 brief/报告依据，无隐瞒（conn_config 8192 上限属新增防线，报告偏离 6 如实登记） |

## 二、质量面复核（均通过）

- SQL 参数化：全部经 MP Lambda wrapper 或 `#{}` 显式 SQL（IN 用 foreach `#{}`），LIKE 经 SqlLikeUtils.escape，无拼接注入面。
- 事务边界：create/update/delete 均 `@Transactional(rollbackFor=Exception.class)`；DuplicateKey 兜底转 4002 后随事务回滚，无半状态。
- guard 引用计数口径：活跃行（@TableLogic）+ 同租户（插件注入），符合 brief「活跃行」口径；countChannels 与 channelCodes 两查非原子已有空表防御分支。
- 错误信息质量：校验失败均指向具体字段与路径（`$.clientId`、`tcp.password`、缺失字段列表、枚举/长度/格式值），达 brief 要求。
- 白名单/防 mass assignment：create/update 显式字段白名单，tenant_id 会话覆盖，deleted/审计字段不进请求面；list 的实体绑定参数在服务层白名单过滤，无 tenantId 旁路。
- 资源消耗（校验器自身）：8192 上限先于校验器 readTree（本 diff 自面已封）；HTTP 层面见 M-3。

## 三、发现清单

### Important（2）

- **I-1 | Important | `channel/schema/ChannelConnConfigValidator.java` L186-188 + L117（diff 行 621-644/683-694）**：required 判定可被显式 null 绕过——`validateAgainstSchema` 对值为 NullNode 的已知属性直接 `continue`，而 `requireNoMissing` 用 `keySet(root)`（键名存在即算在）判定，`{"broker":null,"topic":"t"}` 能通过 MQTT v1 校验并把 `"broker":null` 落库，违 brief「缺 broker 拒」与「配置正确性关口」定位；单测只覆盖键缺失、未覆盖显式 null。修复建议：键存在但值为 null 的已知属性按「缺失」或「类型不符」拒绝。
- **I-2 | Important | `channel/schema/ChannelConnConfigValidator.java` L254-267（diff 行 707-720）**：凭据关键词递归扫描不进数组——`collectCredentialKeys` 仅对 `child.isObject()` 递归，ArrayNode 内对象携带的 `password`/`secret`/`token` 等逃逸双保险；MODBUS/OPCUA 宽松骨架（additionalProperties=true）可落 `{"signals":[{"password":"x"}]}` 类凭据明文进 iot_channel.conn_config，违 ADR-0006 / brief「含凭据字段的 conn_config 保存被拒」；MQTT/HTTP 因 strict Schema 结构性拒绝数组不受影响，但落库的凭据明文会随通道行进入 TASK-0024 配置快照链路（ADR-0006 明文快照不含凭据）。javadoc 声称「全部层级」与实现不符；无数组形状用例。修复建议：对 isArray 的每个元素同样递归。

### Minor（5）

- **M-1 | Minor | `channel/controller/IotChannelController.java` L96/L110-113**：MODBUS/OPCUA 编辑不携带 protocolType 时成功响应不标注——`reservedProtocol` 只看 `form.getProtocolType()`，编辑既有 MODBUS 通道仅改 status 返回通用「操作成功」（冒烟 S6h 实录），brief「响应/文档标注」未全覆盖（status 强制 0 行为本身正确，S6i 核验）。
- **M-2 | Minor | `channel/schema/ChannelConnConfigValidator.java` L123-127**：校验器协议大小写口径与服务层不一致——`loadSchema` 小写归一加载（"mqtt" 命中 mqtt-v1.json），「未知协议拒」实际靠服务层 `resolveProtocolType` 大小写敏感字典闸兜底；校验器被独立复用即存在接缝，且单测 `testUnknownProtocolRejected` 用 `"mqtt "`（带空格）回避了该真实变体而非锁定行为。
- **M-3 | Minor | `backend/ot-admin/src/main/resources/application.yml` L139-140**：JSON 请求体无全局上限——50MB/100MB 为 multipart 限制，不约束普通 JSON body；conn_config 的 8192 上限在 HTTP 反序列化之后才生效（校验器自身 readTree 前已有长度闸，本 diff 自面已封）。框架级遗留面，建议立独立任务统一请求体上限。
- **M-4 | Minor | `channel/schema/ChannelConnConfigValidator.java` L123-141**：`loadSchema` 每次校验重读 classpath 资源并重新 `Pattern.compile`，无缓存——管理面低频 CRUD 可接受，登记不处理（二期换标准库时一并解决）。
- **M-5 | Minor | `gateway/service/impl/IotGatewayServiceImpl.java` L62-66**：guard 转正段 javadoc 缩进错位（续行多 4 空格前缀）——纯外观，不涉语义。

## 四、结论

- Critical 0 / Important 2 / Minor 5。
- 按 L1-execute §13：I-1/I-2 进 §12 退回路径（两处同在 ChannelConnConfigValidator.java，一轮修订可清：补 null-已知属性拒 + 数组递归 + 各补形状用例）；Minor 记 TASK `任务结果` deferred 收尾进 LOG。
- 其余 brief 契约项（DDL 逐字、菜单五行、双道闸、不可变字段、删除保护双层、guard 转正调用面、可建不可启用、改动范围）全部核对通过，实施报告验收输出与 diff 相互印证无矛盾。

（评审完。）
