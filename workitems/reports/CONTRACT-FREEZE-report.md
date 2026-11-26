# 组稿报告：CONV-03 三契约冻结组稿（读类批次）

- 执行者：documenter（subagent，按 CONTRACT-FREEZE-brief.md 执行）
- BASE：`35d22f6`（主仓库根）
- 完成时间：2026-11-21
- 结论：三份契约初稿全部写完，均冻结于 v1.0.0；未改任何代码与 workitems 库文件。

## 1. 改动文件清单

| 文件 | 动作 | 说明 |
| --- | --- | --- |
| `D:\workspace\WisFactory\resources\contract\unified-message-model.md` | 新增 | 统一消息模型契约冻结稿 v1.0.0 |
| `D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md` | 新增 | 配置快照 Schema 契约冻结稿 v1.0.0 |
| `D:\workspace\WisFactory\resources\contract\tdengine-super-tables.md` | 新增 | TDengine 超级表契约冻结稿 v1.0.0 |
| `D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\CONTRACT-FREEZE-report.md` | 新增 | 本报告 |

未创建/未修改上述之外的任何文件（未触碰 backend / frontend / ingest / workitems 库其他文件）；`resources/contract/` 目录为本次新增目录。

## 2. 来源页码与提取核对结果

提取方法：pypdf 全文扫描定位章节 → 逐页提取 → Kangxi 部首变形字形规范化（⼊→入、⽹→网、⼾→户、⻓→长、⻔→门、⻅→见；连字 ﬁ→fi，如 conﬁg→config）→ 表格按版面模式重建为 Markdown → 关键 token 与原文回核对账。

| 契约 | 来源章节 | PDF 起止页 | 提取核对结果 |
| --- | --- | --- | --- |
| 统一消息模型 | §3.2「采集接入：MQTT 主题与上行消息格式」（含 §3.2 续 HTTP 上报与 EMQX 认证） | p24 起、p25 至「3.3 配置同步」标题前止 | 主题 4 行、上行 JSON 示例、字段纪律/处理顺序/批量上限 3 段、HTTP 接口 2 行，逐字入稿；回核 11 个主题/路径/示例 token 全部命中原文 |
| 配置快照 Schema | §3.3「配置同步（系统间接接口）」 | p25（自「3.3 配置同步（系统间接接口）」标题起至页末） | 接口表 4 行（2 GET + 通知 + POST ack）、三类消费方 3 段逐字入稿；回核 4 个路径/主题 token 命中（其中 X-Gateway-Token、snapshot/changes 查询串在 PDF 中被换行拆开，去空白后命中） |
| TDengine 超级表 | §2.2「超级表设计（四张）」+ 附录 B「TDengine 建库建表」 | §2.2：p19（2.2 标题至 2.3 标题前）；附录 B：p36（「附录 B TDengine 建库建表」小节） | §2.2 五列表 4 行 + 说明 3 段逐字入稿；附录 B 建库建超级表 DDL 逐字入稿（见下「DDL 断行整理」）；回核 19 个表名/类型/长度 token 全部命中 |

核对口径说明：

- PDF 提取文本存在排版断行（如 `BIGI`/`NT` 拆断 BIGINT、`USE iot_group_a;` 与 CREATE 同行、TAGS 行内多空格），属提取伪影而非原文内容；冻结稿按语句整理换行与空格，标识符/类型/长度/语句次序逐字未改，已在各契约「与上游差异」第 3/4 条声明。
- 全文字形规范仅限部首变形与连字映射，未做任何语义机改。

## 3. 未做项与原因

| 未做项 | 原因 |
| --- | --- |
| 未把 §3.5 的数据项订阅推送与指标推送载荷并入消息模型契约 | brief 指定唯一内容来源为 §3.2；§3.5（p26）只在推送中引用统一消息模型，属另一章节，纳入即超出改动范围。已在消息模型契约「与上游差异」备注中指路 §3.5 |
| 未给快照体扩写字段级 JSON Schema | PDF 全篇未见快照体字段级定义（§3.3 只到响应信封；§1.8 只有 iot_config_version 存储列），按「原文提取优先于改写、不猜」纪律如实冻结协议层表述，见遗留问题 1 |
| 未改写附录 B DDL 为 quality 列加 COMMENT | brief 要求 DDL 逐字保留；CONV-02 的 0-10 全表以独立「质量码枚举注释（CONV-02 修订叠加）」小节承载并显式标注非 PDF 原文，是否回写进 DDL 注释待裁决（见遗留问题 2） |
| 未执行 brief 预期的「附录 B 补 tenant_id + ADR-0002 脚注」 | PDF V1.1 原文已含 tenant_id（见偏离 1），无需叠加，避免重复 |

## 4. 偏离计划之处

1. **ADR-0002 叠加未执行——brief 预期过时，PDF 原文已含 tenant_id。** brief 写明「附录 B 原文没有 tenant_id，组稿时加上并加脚注」；经核对 PDF V1.1：p3 修订说明第 1 项「超级表补租户标签（ADR-002）……落地位置 2.2、附录 B」，§2.2 表格与正文段落、附录 B 四张 STABLE 的 TAGS 均已带 `tenant_id BIGINT`。冻结稿如实沿用原文、未重复叠加。此非内容冲突（契约与 ADR-0002 决定 B 一致），仅 brief 的前置假设与上游现状不符；已在 TDengine 契约「与上游差异」如实列出。
2. **ADR 编号写法不一致：PDF 正文引「（ADR-002）」，workitems 登记为 ADR-0002（4 位补零）。** 指同一 ADR；冻结稿保留 PDF 原文写法不改，作为观察项记录。
3. **「配置快照 Schema」在 PDF 中无字段级内容可冻结。** brief 命名该契约为「配置快照 Schema」，但 PDF §3.3 只定义同步协议与响应信封（version、snapshotHash、快照体枚举「点位/通道/数据项/公式/指标」及「Schema 版本号一并下发」的承诺），快照体内部结构全文留白。冻结稿按 §3.3 原文冻结协议层，字段级 Schema 缺位如实标注（见遗留问题 1）。
4. **PDF 与 REQ/ADR 冲突点排查：未发现新的实质冲突。** CONV-02（质量码 0-10、文本码映射、CALC_ERR 不落时序行）与 §3.2 的 quality 处理（网关先填、接入按 min/max 二次判定覆盖）及 §2.2/附录 B 的 `quality INT` 列相容；ADR-0002 与 PDF V1.1 现文一致；ADR-0003（通道凭据不进快照）与 §3.3「凭据例外」段落一致。唯一不一致为上述第 2 条编号写法。

## 5. 遗留问题

1. **快照体字段级 Schema 缺失（上游留白，联调前必须补齐）。** 第一批接入要按快照体建缓存与依赖图，仅有信封不够。建议协调者决定：由上游出 §3.3 补充稿（含 Schema 版本号初值），或授权在冻结稿上补字段级 Schema 并升版本。
2. **quality 0-10 枚举是否回写进 DDL 列 COMMENT。** 当前以契约独立小节承载；若要求 DDL 自带注释（TDengine 支持列 COMMENT），属对附录 B 原文的实质改动，需升版本并裁决。
3. **§3.5 推送载荷是否单独立契约。** 数据项订阅推送「消息体即统一消息模型」、指标推送「在统一消息模型上扩展 9 个字段」，均为跨系统契约面；本次未纳入，建议下批组稿评估。
4. **ADR-002 / ADR-0002 写法统一。** PDF 后续修订时建议统一为 4 位补零口径（AGENTS.md 铁律 6）。
5. **brief 假设复核建议。** 本批发现的「附录 B 已含 tenant_id」说明上游 PDF 与 workitems 决策记录的同步状态需复核；后续组稿批次引用 brief 前建议先对 PDF 现文做一次增量核对（本批已在契约内以「与上游差异」留痕）。
