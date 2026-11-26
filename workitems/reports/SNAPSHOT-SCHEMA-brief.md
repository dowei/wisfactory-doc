# 实施 brief：快照体字段级 Schema 派生（TD-0052 裁决 B · ADR-0015）

## 批次

- 契约组稿批次（共 1 个 subagent，读类；与 backend 道并行，不碰代码）
- BASE：`04fc3a5`
- 改动范围：
  - `resources/contract/config-snapshot-schema.md`（升 v1.0.1：新增字段级 Schema 章；既有协议信封章逐字不动）
  - 报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\SNAPSHOT-SCHEMA-report.md`
  - 其余一律不碰

## 必读（只给路径，不粘正文）

- 裁决依据：`workitems\decisions\ADR-0015.md`（决定/理由/影响范围全在其中）
- 被升版契约：`resources\contract\config-snapshot-schema.md`（现 v1.0.0，协议信封 + 快照体枚举承诺）
- 唯一映射源：详细设计 V1.1 PDF `D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-design-v2.2.pdf` 不用——用 **factory-data-platform-detail-design-v1.1.pdf** 的 §1.4-1.8 字段表与附录 A DDL（pypdf 提取，方法见 `reports\CONTRACT-FREEZE-brief.md`；§1.4 网关/通道、§1.5 点位、§1.6 模型域、§1.7 指标、§1.8 配置发布域——以 PDF 实际章节号为准，先扫目录再取页）
- 一致性参照：`resources\contract\unified-message-model.md` v1.1.0（命名风格先例）、`workitems\conventions.md`（CONV-01 凭据不进快照等）

## 产物要求

1. config-snapshot-schema.md 头部升 v1.0.1（修订记录注明 ADR-0015 + 派生来源章节；变更规则/状态行保留；既有「3.3 协议信封」章逐字不动）。
2. 新增「快照体字段级 Schema（v1.0.1 派生）」章：
   - 按快照体枚举（点位/通道/数据项/公式/指标——v1.0.0 信封承诺的对象集）逐对象给出字段级定义：来源表、字段名（快照 JSON 内 camelCase）、类型、来源列名（snake_case）、可空性、语义备注；JSON Schema 形态（每对象一个 schema 代码块）
   - 凭据类字段（iot_channel_credential 相关）不得进快照（CONV-01 口径与 §3.3 凭据例外段一致；引用不携带）
   - 逻辑删/审计字段（create_by 等）不进快照——快照是配置面不是审计面；tenant_id 必须在快照体内（配置按租户隔离的根）
   - 每个对象标注来源章节页码；字段命名/类型映射与 MySQL 原文的差异逐条进「与上游差异」
3. 「与上游差异」节追加：v1.0.1 派生章的全部映射差异 + 声明「上游 §3.3 补充稿到位后按演进式变更对齐（ADR-0015）」。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\SNAPSHOT-SCHEMA-report.md`。五项：改动文件清单 / 提取核对结果（每对象来源页码与 token 回核）/ 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；只写改动范围列出的文件；拿不准的字段映射进「与上游差异」并标注，不猜；不得改动既有协议信封章任何字（CONV-03）。

## 收尾回写

- 无 TASK 对象（协调动作批次），报告即终稿。
