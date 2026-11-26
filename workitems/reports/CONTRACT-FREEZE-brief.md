# 实施 brief：CONV-03 三契约冻结组稿（读类批次，不改代码）

## 批次

- 契约组稿批次（共 1 个 subagent；产物为三份契约文档初稿 + 组稿报告）
- BASE：`35d22f6`（主仓库根）
- 改动范围：
  - `resources/contract/unified-message-model.md`（新增）
  - `resources/contract/config-snapshot-schema.md`（新增）
  - `resources/contract/tdengine-super-tables.md`（新增）
  - 报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\CONTRACT-FREEZE-report.md`
  - 其余一律不碰（不建目录之外的任何文件，不改 backend/frontend/ingest/workitems 库）。

## 背景（只给路径，不粘正文）

- 约束来源：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\conventions.md` 的 **CONV-03**——三份契约冻结后第一批接入方可进入联调；契约变更必须升版本并通知各系统；冻结前不联调。
- 内容来源（唯一权威）：`D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-detail-design-v1.1.pdf`（36 页）：
  - 统一消息模型 → §3.2
  - 配置快照 Schema → §3.3
  - TDengine 超级表 → §2.2 与附录 B（约 p35-36）
- 修订叠加（冻结稿必须反映，PDF 原文可能尚未更新）：
  - **ADR-0002**（`workitems\decisions\ADR-0002.md`）：四张超级表 TAGS 统一补 `tenant_id BIGINT`——附录 B 原文没有，组稿时加上并在脚注标明「ADR-0002 修订叠加」。
  - **CONV-02**（conventions.md）：质量码 0-10 数值枚举为唯一存储口径——超级表契约中质量码字段注释必须带 0-10 全表。

## 提取方法

本机有 python + pypdf（无 pdftotext）。先用全文扫描定位章节页码，再逐页提取：

```bash
python - <<'EOF'
from pypdf import PdfReader
r = PdfReader(r"D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-detail-design-v1.1.pdf")
for i, p in enumerate(r.pages):
    t = p.extract_text() or ""
    if any(k in t for k in ("3.2 ", "3.3 ", "超级表", "消息模型", "快照")):
        print(f"p{i+1}: {t.strip().splitlines()[0][:60]}")
EOF
```

注意：PDF 文本含旧字形（⼊/⽹/⼾ 等部首变形），提取后**不要机改正文语义**，仅把明显乱码字形规范为标准字（⼊→入、⽹→网、⼾→户），DDL/字段名/JSON 示例逐字保留。

## 产物要求（三份契约文档统一格式）

每份头部：

```
# <契约名> · 冻结稿 v1.0.0
- 冻结日期：2026-11-21
- 内容来源：工厂数据采集系统详细设计 V1.1 <章节号>（逐字提取）
- 修订叠加：<ADR/CONV 编号与内容，没有写「无」>
- 变更规则：本契约变更必须升版本（语义化：字段/语义不兼容升次版本，措辞勘误升修订号）并通知各系统（CONV-03）
- 状态：冻结（冻结后接入侧方可联调）
```

正文 = 章节原文逐字（表格转 Markdown 表格，DDL 代码块保留，JSON Schema 代码块保留）。文末加「与上游差异」小节：列出冻结稿相对 PDF 原文的全部增改点（目前预期只有 ADR-0002 的 TAGS tenant_id；若组稿中发现其它不一致，如实列出并标记「待协调者裁决」，不要自行定夺）。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\CONTRACT-FREEZE-report.md`，含五项：

1. 改动文件清单（三份契约 + 本报告）
2. 每份契约的来源页码与提取核对结果（原文章节起止页）
3. 未做项与原因
4. 偏离计划之处（含 PDF 中发现的与 REQ/ADR 冲突点）
5. 遗留问题

## 契约

- no-subagents：不得再派 subagent。
- 只写上述三个新文件与一份报告；不碰任何其它文件。
- 原文提取优先于改写：拿不准的字段/语句原样保留并进「与上游差异」，不猜。
