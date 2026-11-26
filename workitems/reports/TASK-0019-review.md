# TASK-0019 评审报告：iot_channel_credential 凭据接口与 SM4 加密（提交 2ff499b）

- 评审者：fresh reviewer（不共享协调者上下文；no-subagents；只评审不修改）
- 依据：TASK-0019-review-brief.md；L1-execute.md §13（凭据域安全敏感面加严）；实施 brief / TASK 详情 / 实施报告 / ADR-0006 交叉印证
- 输入：diff 包 2566 行逐行读完（22 文件：credential 主代码 12 + gateway/auth 4 + 测试 5 + 升级脚本 1）；关键框架假设回源核实（SmDataCrypto / OtTenantLineHandler / TenantContext / OperateLogAspect / GlobalExceptionHandler / StaticTokenInternalAuthChecker / SaTokenInterceptorConfig / IotGatewayInternalController / ResultCode / BaseEntity / 0018 菜单脚本 / SaPermissionRegistryConsistencyTest 在位）

## 一、契约一致性核验（①项，逐条）

| 契约项 | 结论 | 证据 |
| --- | --- | --- |
| 列表永不回密文/明文 | 通过 | CredentialItemVO 结构无密文字段（反射断言）+ MockMvc jsonPath doesNotExist + 掩码由服务层解密后生成；解密失败降级纯占位 |
| view 单设权限 + @OperateLog | 通过 | @SaCheckPermission("iot:credential:view") + @OperateLog(saveResponse=false)；403 走 PermissionDeniedEvent（BUG-0044 链，GlobalExceptionHandler 实证） |
| 轮换 key_version 递增不新增行 | 通过（happy path） | uk_channel_cred_type upsert + nextKeyVersion 递增；并发缺口见发现 I-1 |
| 吊销立即生效 | 通过 | status 白名单置 0；pullCredential 对 status!=1 统一 403；冒烟 S9b 实证 |
| InternalAuthChecker 收口 /internal/v1/** | 通过 | 功能改动仅 GatewayInternalAuthConfig 1 行（pattern 放宽）；全仓仅心跳 + 凭据拉取两个 /internal/v1 端点，心跳有显式 checker 调用、拦截器加一道同实现校验幂等无害；fail-closed（未配置令牌全拒）+ 常量时间比较未被改动 |
| SM4 {v1} 与 key_version | 通过 | SmDataCrypto 实证：encrypt 对 {v1} 前缀输入幂等原样返回 → 服务的 isCiphertext + equals 双条件 fail-closed 成立；decrypt 异常文案不含输入（无泄密面）；写路径仅密文进 SQL |
| 审计无明文覆盖面 | 通过 | 真实切面织入 + param/response/failReason/content/module/url/method 全列 grep；{v1} 禁令覆盖三个动态列（content 为静态 @Operation 摘要，偏差 5 已登记，可接受）；saveParam/saveResponse=false 在切面序列化前短路（含异常路径），实证成立 |
| 菜单 4051-4052 一一对应 | 通过 | SQL 4051=edit / 4052=view（parent 4013）↔ 控制器三码守卫测试双向核对；iot:channel:list 由 0018 的 4046 承接、跨脚本核对；SaPermissionRegistryConsistencyTest 兜底 |
| 拉取通道归属与租户隔离 | 通过 | 无会话时 OtTenantLineHandler.ignoreTable 实证跳过注入 → 服务层显式 (channel_id, cred_type) 定位 + 通道启用 + 凭据启用 + 租户一致断言；管理面链路插件按会话租户注入 |
| SQL 参数化 | 通过 | 全部 LambdaQueryWrapper eq，无拼接 |

范围核对：22 文件全部落在 brief 四条声明路径内（与协调者结论一致）。dev 冒烟/prod 回归证据与报告第 2 节一致，本评审不再复跑。

## 二、发现清单

### Critical（0）

无。未发现任何明文/密文泄露路径、权限绕过或租户隔离破口。

### Important（1）

- **I-1** / 出处：`IotCredentialServiceImpl.writeCredential` → `rotate()`（main） / **并发轮换竞态：更新路径无乐观锁/原子递增，丢失更新时双 200 单存活**——插入竞态已兜（DuplicateKeyException → 重查转轮换），但两个并发轮换各自基于同一 key_version 计算新版本后裸 `updateById` 后写覆盖先写：双方均收到成功响应，实际只有一个密文落库，且存活行的 (密文, key_version) 配对与真实顺序代次错位、事后无从对账（key_version 是唯一的轮换代史）。brief 评审焦点明列「并发轮换竞态」；一期单密钥下无选钥后果、触发需同毫秒同凭据双写，故 Important 不升级，但属可廉价修复（版本条件更新或 `SET key_version = key_version + 1` + affected rows 核验）的正确性缺陷。

### Minor（7）

- **M-1** / `nextKeyVersion` / key_version 边界未设防：`Long.parseLong` 接受负值（"-5"→"-4"），逼近 Long.MAX_VALUE 时静默溢出为负、且 19 位数字超出 varchar(16) 将 DB 报错——实际不可达，纯理论边界。
- **M-2** / `pullCredential` / 拒绝文案区分「通道不存在、已删除或已禁用」与「凭据不存在或已吊销」，与 javadoc「不区分原因，不给探测面」自相矛盾——持内部令牌者可探测 channelId/credType 存在性（调用方可信且本就知晓 channelId，实际价值趋零，改文案或改注释任一即可）。
- **M-3** / `IotCredentialController.write` / 轮换审计语义弱：轮换按 businessType=INSERT 落审计、POST url 不含凭据 id、param 列整列为空——sys_operate_log 无法区分首写与轮换，法证精度损失（行为人可经凭据行 update_by 关联恢复，报告偏差 2 已登记，但业务类型 UPDATE 可更准确）。
- **M-4** / `IotChannelCredential` 实体 / Lombok @Data 的 toString 含 ciphertext，未像三个 DTO 一样 @ToString.Exclude——当前无任何 stringifies 路径，但最敏感域内唯一携带密文的裸对象缺源端治理，未来任何实体调试日志即泄密文（密文非明文、低敏，纯卫生项）。
- **M-5** / `CredentialMask` / 掩码泄露度：≥12 字符凭据向 iot:channel:list 持有者露出 8 字符（前四后四）——brief 明文授权该口径且已做隐藏段 ≥4 收紧、偏差 3 已登记；对低熵数字类凭据剩余搜索空间收缩明显，建议按 cred_type 差异化全占位策略（已预留单点改动位），记录备查。
- **M-6** / 框架既有路径（不在本 diff，与 REQ-0011 验收 4 相关登记） / `GlobalExceptionHandler.handleHttpMessageNotReadableException` 以 WARN 记 `e.getMessage()`——Jackson「Unrecognized token」类错误会回显畸形请求体中的裸 token 片段，写入接口若收到明文未加引号的畸形 JSON，明文片段可进应用日志；冒烟 grep 未覆盖该形态（未发畸形探针）。非本任务引入、影响所有敏感字符串端点，建议框架级跟进。
- **M-7** / 框架既有配置（不在本 diff） / `application.yml` 仓库内含 dev 默认 data-key（`otlab-data260930`）——prod yml 无默认值 fail-fast 正确，但凭据域 SM4 密钥材料在仓库可见一个 dev 默认值，与升级脚本头注「代码仓库均不得出现密钥明文」的口径存在张力（沿用 sm4.key 框架惯例，仅登记知悉）。

## 三、验证范围限制

- DDL 与《详细设计》附录 A p35 的逐字一致性未独立复核（PDF 不在评审必读清单内，采信实施报告 pypdf 提取声明 + 冒烟 SHOW CREATE TABLE 核对）。
- dev/prod 冒烟为报告原始输出采信，未复跑；单测全量 1723 绿采信。
- 评审结论：**无 Critical，1 Important（I-1 并发轮换丢失更新），7 Minor**；I-1 建议随本任务收尾或并入 0020/0024 邻近任务修复（一处改动），M-1/M-2/M-3/M-4 可择机批量处理，M-5/M-6/M-7 登记知悉。
