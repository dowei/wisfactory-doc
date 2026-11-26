# 评审 brief：TASK-0054 diff 评审（代码评审）

> 实现提交 3cdde58（BASE=5758d5f，隔离区间 9aed9a4..3cdde58——剔除并行道 0058 前端交错提交；8 文件）——dict/sync 字典同步占位转正。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0054-diff.txt`（9aed9a4..3cdde58，8 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0054-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0054-report.md`（含 status=0 未删行归类偏离说明）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0054.md`
- 关联：FEAT-0017/REQ-0028/TC-0017/ADR-0010（workitems 库）——中台建点回传一期不做
- 权威：详细设计 §3.5 dict/sync（PDF pypdf 可提取）；上游 0051 占位说明（`reports\TASK-0051-report.md`）

## 越界反查（协调者已比对，评审者复核）

- 8 文件全落 backend openapi 模块+测试；并行道 0058 前端提交（9aed9a4）已被区间隔离剔除；create/dataperm/前端/ingest/calc 零触碰；无新 SQL。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **增量口径**：`update_time > FROM_UNIXTIME(since/1000)` 边界正确性（严格大于——恰好等于 since 的行应排除）；since=0 全量；**deleted=1 逻辑删行必须返回**（三态中的禁用态不能因物理过滤丢失）；跨租户 tenant_id 显式条件
2. **三态分类**：执行者偏离=3→deleted=1 禁用 / 2→status=0 未删变更 / 1→窗内新建——评估该分类是否自洽（brief 未定义 status=0 归类，执行者按冒烟缺"变更"态细化）；status=0 行若同时 update_time 在窗内是否同时满足新增与变更条件（CASE 顺序）
3. **VO 具体化**：保留 since/instances/items 既有字段（占位兼容）；InstanceRow/ItemRow 含资产=模型对应字段（instanceId/templateId/modelCode/deptId 与 itemId/dataCode/instanceId/fullKey/status）；Schema description 说明中台资产/平台模型命名差异
4. **权限**：appKey 有效即可、**不走 scope/dataperm 裁剪**（§3.5 原文）——Service 用验签 tenantId 直查实证；identity 取 audited 链；无效 key 403+6001 既有链路不改
5. **SQL 注入/性能**：since 参数化（Mapper XML/注解 SQL 无字符串拼接）；FROM_UNIXTIME 索引友好性（可用索引扫描 vs 全表——数据量小可接受，但评估说明）
6. **测试有效性**：6 服务单测+3 Mapper SQL 解析回归——三态/边界/跨租户/禁用态是否真断言（非快照）；2395 测试零回归（基线 2392+3？以报告基线为准核实）
7. **执行者登记项**：status=0 归类偏离（报告 §2/§4）——合理性裁决；"中台建点回传一期不做"边界（无写接口）实证

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0054-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
