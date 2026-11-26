# 组稿报告：快照体字段级 Schema 派生（SNAPSHOT-SCHEMA，TD-0052 裁决 B · ADR-0015）

- 批次：契约组稿批次（读类，不碰代码）；BASE `04fc3a5`
- 日期：2026-11-21
- 执行者：documenter subagent（no-subagents 契约遵守：未再派生下级）

## 1. 改动文件清单

| 文件 | 改动 |
| --- | --- |
| `resources/contract/config-snapshot-schema.md` | v1.0.0 → v1.0.1：头部升版并加修订记录行（ADR-0015 + 派生来源章节）；新增「快照体字段级 Schema（v1.0.1 派生）」章（派生规则 8 条 + 顶层结构 + 五对象字段表与 JSON Schema 代码块）；「与上游差异」追加 v1.0.1 派生声明（第 4-12 条）与 v1.0.0 备注现状更新。既有「3.3」协议信封章与 v1.0.0 差异条目 1-3、备注段逐字未动 |
| `workitems/reports/SNAPSHOT-SCHEMA-report.md` | 本报告（即终稿，无 TASK 对象） |

其余未动：不碰 backend / frontend / ingest / workitems 库对象 / 其它契约文件；无代码改动。

## 2. 提取核对结果（每对象来源页码与 token 回核）

提取方法：pypdf（py -3.14 -I，pypdf 6.19.0）逐页提取；先扫 36 页目录定位章节实际页码，再逐页取字段表与 DDL。字形处理仅限明显乱码（部首变形 ⼊/⽹/⼾/⻔ 等、连字 ﬁ→fi，如 p7 正文 conn_conﬁg），DDL / 列名 / 语义未机改。

目录定位（brief 预估章节号与 PDF 实际不同，以实际为准）：

| 对象 | 来源章（PDF 实际） | 字段表页 | DDL 回核页 |
| --- | --- | --- | --- |
| 通道 channel ← iot_channel | §1.3 采集域：网关与通道 | p7 | p30（附录 A 一） |
| 点位 point ← iot_point | §1.4 点位域 | p9 | p31（附录 A 二） |
| 数据项 dataItem ← iot_data_item | §1.6（续）实例数据项 | p14 | p32（附录 A 三） |
| 公式 formula ← iot_data_item_ref | §1.7 引用关系与指标定义 | p15 | p32（附录 A 三） |
| 指标 metric ← iot_metric | §1.7 引用关系与指标定义 | p15 | p33（附录 A 四） |
| 凭据排除依据 ← iot_channel_credential | §1.3A 通道凭据（V1.1 新增） | p8 | p35（附录 A 六） |
| 信封语境（快照体 / Schema 版本号承诺） | §3.3 | p25（回扫确认「快照体」「Schema 版本号」两词在页） | — |
| 装载口径依据（scope_type 1 网关 / 2 计算） | §1.8 配置发布域 | p16 | p33 |

token 回核（脚本对每对象全部来源列名 + 唯一约束名做子串核对，字段表页与 DDL 页双回核）：

- channel：9/9 命中（id、tenant_id、gateway_id、channel_code、channel_name、protocol_type、conn_config、status、uk_gateway_channel_code），p7 与 p30 双页全中。
- point：19/19 命中（id、tenant_id、point_code、point_name、data_type、value_type、unit、min_value、max_value、sample_period、deadband、channel_id、protocol_addr、bound_instance_id、dept_id、status、config_version、remark、uk_tenant_point_code），p9 与 p31 双页全中。
- dataItem：21/21 命中（20 列 + uk_tenant_full_key），p14 与 p32 双页全中。
- formula：7/7 命中（6 列 + uk_item_ref），p15 与 p32 双页全中。
- metric：16/16 命中（15 列 + uk_tenant_metric_code），p15 与 p33 双页全中。
- 凭据表（排除核对）：cred_type、ciphertext、key_version 在 p8 / p35 存在（证明映射源可核对），在 v1.0.1 派生章的 schema 块与字段表中 0 出现（脚本回核），仅 cred_type 作为 §1.3A 要求的「凭据类型」标记取值域出现在描述文本。
- 合计：72/72 命中，缺失 0。

产物自检（脚本回核）：

- 「3.3」协议信封章与 v1.0.0 基准逐字节一致（1162 字符比对通过）；变更规则 / 状态行、v1.0.0 差异条目 1-3 与备注段逐字保留。
- 6 个 json 代码块全部解析通过（顶层结构 1 + 五对象 schema 各 1）；五 schema `$id` 为 config-snapshot/{point,channel,dataItem,formula,metric}@1.0.1。
- 顶层键含 tenantId / scopeType / scopeId / version / schemaVersion 与五个对象数组；五对象均携带 tenantId。
- ciphertext、key_version、create_time、create_by、update_time、update_by、deleted 不出现在任何 schema 块与字段表行。

## 3. 未做项与原因

1. 未派生第六类对象（网关表、模板数据项、模型域模板 / 实例、数据字典 iot_data_dict、状态判定规则 iot_status_rule）：不在 v1.0.0 信封承诺的五对象集内，brief 亦限定对象集；其中 §1.5B 状态规则「随配置发布下发计算引擎」与五对象枚举的张力如实登记进差异第 11 条与遗留问题，不自行扩集。
2. conn_config / protocol_addr 内层键名未定义：上游只给成分描述（MQTT 含 broker、topic、QoS；HTTP 含上报路径与鉴权方式）且自述「结构由 Schema 约束」但未定版内层键名；按「不猜」原则留自由 object + 禁含凭据约束，差异第 6 条留痕。
3. 未做模板阈值继承展开：实例数据项四阈值字段未填时继承模板 default_ 四字段属管理系统生成口径，上游未定快照是否代做展开；快照如实携带实例行原值（可空即空），表备注留痕。
4. 未回写任何 workitems 对象：brief 收尾条款「无 TASK 对象，报告即终稿」；TD-0052 已由 ADR-0015 标记裁决完毕（自 todo.md 删除的动作属协调侧，未代做）。
5. 未通知各系统：CONV-03 要求契约变更须通知各系统，属协调动作，不在本批执行范围（列入遗留）。

## 4. 偏离计划之处

1. 章节号对位与 brief 预估不同：brief 预估「§1.4 网关/通道、§1.5 点位、§1.6 模型域、§1.7 指标、§1.8 配置发布域」；PDF 实际为 §1.3 网关与通道、§1.4 点位域、§1.5 模型域、§1.6 数据项、§1.7 引用关系与指标定义、§1.8 配置发布域。brief 已授权「以 PDF 实际章节号为准，先扫目录再取页」，按实际执行；契约头部与派生章按实际章节号标注。
2. brief 写「CONV-01 凭据不进快照」：conventions.md 现行 CONV-01 实为「业务数据字典与系统枚举字典分野」（ADR-0003），凭据排除口径的实际出处是 §1.3A「密钥与下发口径」四条 + §3.3 凭据例外段 + §1.3「通道凭据不入 conn_config 明文」。契约按实际出处引用（未引 CONV-01）；数据项 dataCode 的字典分野备注处如实引用了 CONV-01 本义。
3. 凭据标记字段名为拟定：§1.3A 只给文字口径「快照里通道只带『是否有凭据、凭据类型』标记」，未给字段名；hasCredential / credentialTypes 为 v1.0.1 拟定并在差异第 9 条留痕（属派生授权范围内的必要落点，非弃权）。
4. 提取环境：CONTRACT-FREEZE brief 的提取方法（python + pypdf）在本机 `python`（3.8，pypdf 5.9.0）位于用户 site-packages，`-I` 隔离模式下不可见；改用 `py -3.14 -I`（pypdf 6.19.0）并因 GBK 控制台在脚本内 reconfigure stdout 为 UTF-8。提取方法本身未变。
5. 附录 A DDL 实际在 p30-p36，本批回核用 p30-p33（五表 + 凭据表 p35）；p34 / p36（对外调用方、计算日志等）与本批对象集无关，未核对。

## 5. 遗留问题

1. 状态判定规则的快照承载对象未定：§1.5B 明言规则集「随配置发布下发计算引擎（走 3.3 配置同步）」，但 v1.0.0 信封只承诺五对象，状态规则无处安放（差异第 11 条留观察）。TASK-0024（快照生成）派发前需协调者或上游补充稿定承载对象（第六对象或并入既有对象），否则计算引擎无法按 §1.5B 判定链路拿到规则。
2. 快照体顶层键名与装载口径为 v1.0.1 拟定（schemaVersion / tenantId / scopeType / scopeId / version / 五数组名；网关范围装载通道+点位、计算范围装载数据项+公式+指标）：上游 §3.3 未逐字定义，TASK-0024 落地时若调整须升版本留痕（差异第 10 条）。
3. conn_config / protocol_addr 内层键名、bigint 超 2^53 的字符串化策略：均待上游 §3.3 补充稿定版（差异第 6 条）；补充稿到位后按演进式变更逐条机械对账、冲突升版本以补充稿为准（ADR-0015）。
4. 公式对象落位（映射为 iot_data_item_ref 引用边、表达式随数据项 formula 字段）为拟定口径：§3.3「数据项、公式、引用边、指标」并提与信封五对象枚举之间的对应待补充稿确认（差异第 8 条）。
5. CONV-03 通知动作：config-snapshot-schema.md 升 v1.0.1 后须通知各系统（接入、计算引擎、网关侧），本批未执行，请协调侧纳入收口清单。
