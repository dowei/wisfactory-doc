# 复评报告：TASK-0013 修订轮（scoped 复评）

- 评审者：fresh reviewer（brief：reports/TASK-0013-review2-brief.md）
- 对象：修订提交 `4d7246d`（BASE `fb13384`），修复 diff 2 文件 252 行已完整读入；仓库实读核实 `IotDataItemServiceImpl.java` / `RelativeRefExpander.java` / `IotDataItemTemplate.java` / ref DDL
- 范围：仅原 I-1/I-2 与 M-5/M-6 是否修复、修复本身有无新缺陷；不重开全场（Minor #3/#4/#7 deferred 不判）

## 结论：四项全部已修复，新发现 0

| 项 | 判定 | 核实要点 |
| --- | --- | --- |
| I-1 展开终检 | **已修复** | 生成循环内展开+存在性核对之后（Impl L233-236）：expandedFormula 残留 `{`/`}` 即 paramError 整批拒，文案含原始编码与首残留 token（截 40 字符）。对照展开器两正则实核：RELATIVE_REF 只匹配 `{SELF\|PARENT.[A-Za-z0-9_]+}`、BRACE_REF 只匹配 `{[A-Za-z0-9_.]+}`——`{ self.X }`（空格）、`{SELF.ELEC-METER}`（连字符）、未闭合 `{` 均不匹配任一正则、必残留；而正常公式（SELF/PARENT/绝对引用按登记语法）展开剥壳后为裸键形态、天然无花括号，不误伤。畸形 token 不再静默落库，冒烟 R2 实测拒+实例行回滚（0 行落库） |
| I-2 selfScope 收敛 | **已修复** | selfScope = 活跃占用 ∪ **newItems**（L182-188）：已删占用码不在集合，SELF 引用之在展开期 L214-219 即「公式引用的数据项不存在: SELF.X」整批拒，先于任何 insert/ref 写——ref 不落悬空边。回归核对：原放行两情形（活跃既有项、本批将插项）仍在集合内，仅死键情形由静默落悬空边改为 fail-closed，与「已删键永久占用、无复活 API」语义自洽。冒烟 R3e 实测拒+0 落库；R2_DEAD2 存量边登记为删除前历史、不属本轮封口范围，口径一致 |
| M-5 剥壳键限长 | **已修复** | `MAX_LEN_REF_FULL_KEY=180` 与 ref DDL `ref_full_key varchar(180)` 实核一致（20261122_iot_data_item_domain.sql L104）；校验遍历 `outcome.resolvedRefs()`——与落库路径 `ref.setRefFullKey(...)`（L249/L275-279）为同一集合，全部 ref_full_key 写入被覆盖；置于任何 DB 写之前，paramError 文案含键/上限/实际长度。SELF/PARENT 展开键上限 96+1+64=161<180，正常路径零误伤；超长仅可达于绝对引用，冒烟/单测实测 400 形态 |
| M-6 公式项缺公式 | **已修复** | L203-205：`SOURCE_TYPE_FORMULA.equals(getDefaultSourceType()) && isBlank(formula)` 整批拒「公式项缺少公式: <code>」；`defaultSourceType` 为 Integer（实体 L81），equals 空安全；null 兜底走 buildItem 的 SOURCE_TYPE_BIND（L382-383）口径不变。null/空白两形态均有单测固化 |

## 修复本身新缺陷排查：无

- 红转绿闭环可信：4 新测先在 `fb13384` 上跑出恰 4 红（均「nothing was thrown」，佐证修复前静默 fail-open 行为与原发现一致），修复后 25 绿、全量 1449 绿；文案断言与实际消息格式逐一相符（残留花括号/SELF.GONE+不存在/超长+180+204/公式项缺少公式），并 `verify(never()) insert` 锁零副作用。
- 终检 `firstResidualToken` 边界安全（仅 `}` 时取 `}` 起点、Math.min 防越界）；终检仅入非空公式分支（空白公式不可能含花括号）；I-1→M-5→限长检查顺序无交互问题。
- 越界核对：单提交恰 2 文件（服务 impl + 同域测试），未触碰 deferred 三项（事件发布时机/查询先存在性/接口 javadoc 均未见改动，无恶化）；类 javadoc 新增「修订轮防线」段与实现一致。

## 一句判定理由

四项修复均按裁决口径落在生成循环单一收口处、由 red→green 单测与 dev 冒烟负路径+正常模板回归双向实证，且对正常公式路径与 deferred 项零回归——复评通过，无新发现。
