# 评审 brief：TASK-0059 diff 评审（代码评审）

> 实现提交 86a1c09（BASE=e387c3b，隔离区间剔除 0042 修订 01236fd；13 文件 frontend/**）——设备详情页（状态区/实时/曲线/来源追溯，展示端推进）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0059-diff.txt`（e387c3b..86a1c09 限 frontend/，13 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0059-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0059-report.md`（含管理面 API 缺口登记：无按设备取数据项实时/历史端点）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0059.md`
- 关联：FEAT-0019/REQ-0029/TC-0019/ADR-0010（降采样留痕）/ADR-0011（同源）
- 上游：reports\TASK-0058-report-batch1.md（展示端骨架）+ TASK-0050-report.md（状态读口四端点）+ TASK-0051-report.md（对外 HMAC 接口——浏览器不可直连）
- 范式：frontend/src/views/display/**（0058 批一）

## 越界反查（协调者已比对，评审者复核）

- 13 文件全落 `frontend/**`；区间内 0042 calc 修订（01236fd）已隔离；backend/calc/ingest/契约零触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **安全红线（首要）**：全 diff 扫描确认**前端零 appKey/secret、零 HMAC 直连**——所有数据走管理面登录态端点；无 fetch 到 /openapi/v1/** 的代码
2. **状态区同源**：大色块/持续时长/历史时间线是否真走 0050 读口（current/history），无硬编码状态名/颜色、无现算；UNKNOWN/LWT 离线态按 0050 口径
3. **实时区与缺口处置**：管理面无"按设备取数据项实时/历史值"端点——执行者用点位近 24h 曲线端点派生绑定类、公式/汇总类走"数据接入中"空态。**评估该处置是否诚实且合理**（不造假数据、不写死链），缺口登记是否清晰到可执行（需要什么端点/归哪个任务）
4. **历史曲线**：多选叠加/时间范围/超限断线标注 + **降采样明示标注**（ADR-0010）——不静默降采样；TDengine 故障下 fail-soft 空集是否正确处理（前端空态验证）
5. **来源追溯**：引用边+公式/汇总配置展开逻辑；子级贡献值/输入值真值读口缺口的登记（0063 未交付）——无死链、空态诚实
6. **下钻与空态**：/display/device?instanceId= 第三次下钻，车间页入口接通；无数据源槽位统一"数据接入中"空态、零假数据（承接 0058 批一纪律）
7. **测试有效性**：591 用例（583 基线+8）三门口全绿；live 10/10+曲线端点补测 200 与断言对应；组件卸载清理/内存泄漏

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0059-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。**特别对第 3 点（API 缺口处置合理性）给出明确判定**——若判 Important 说明需要什么后端端点补位。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
