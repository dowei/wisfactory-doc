# 评审 brief：TASK-0095 指标管理页与 metric-api 前端补齐

## 角色

- 代码评审（标准任务，改动 >3 文件、涉及 API 封装、表单校验与权限，需即时 diff 评审）

## 必读（只给路径，不粘正文）

- 评审 brief：`reports/TASK-0095-review-brief.md`（本文件）
- TASK 详情：`tasks/TASK-0095.md`
- 实施 brief：`reports/TASK-0095-brief.md`
- diff 包：`reports/TASK-0095-diff.txt`（BASE 449d7eb3 → HEAD 09ae85cc，共 7 个前端文件）
- 实施报告：`reports/TASK-0095-report.md`
- 后端接口事实源：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/controller/IotMetricController.java` 及 `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/metric/dto/` 下相关 Form

## 越界反查（已由协调者机械核对，作为必查项）

`git diff --name-only 449d7eb3..09ae85cc` 结果 7 个文件，与实施 brief 改动范围声明逐项比对：

| 文件 | 是否在 brief 声明集内 |
| --- | --- |
| frontend/src/api/iot/metric-api.ts | 是 |
| frontend/src/components/iot/iot-consts.ts | 是（brief 允许「复用常量」时改） |
| frontend/src/views/iot/metric/index.vue | 是 |
| frontend/src/views/iot/metric/components/metric-form-modal.vue | 是（brief 允许新建） |
| frontend/src/views/iot/metric/components/metric-detail-modal.vue | 是（brief 允许「按需拆 detail modal」） |
| frontend/tests/unit/api/iot/metric-api.spec.ts | 是 |
| frontend/tests/unit/views/iot/metric.spec.ts | 是 |

结论：无声明集之外文件。评审者仍须独立复核 diff 内容是否越界修改上述文件之外的地方。

## 依据

- `.agents/protocol/L1-execute.md` 的「diff 评审」口径：看两样——① 与 TASK 验收标准 / brief 接口的一致性；② 明显质量缺陷（错误处理、边界、安全）。发现分级 Critical / Important / Minor。

## 评审重点提示（非结论，仅为定位）

1. metric-api 各方法 URL / HTTP 动词 / 参数是否与 IotMetricController 端点逐项对齐（尤其 `generate` 的 `instanceId` query 参数 + body、`changeStatus` 的 form、`remove` 的 ids 路径拼接）。
2. 表单校验是否与后端 CreateForm/UpdateForm 注解一致：aggType 1-5、period HOUR/DAY/MONTH/YEAR、calendarType 锁 1、metricCode 字符集 `[A-Za-z0-9_]` 且创建后禁改、比值型 aggType=5 分子分母齐备、非比值型 ratio 字段置 NULL、status 是否误入编辑提交。
3. 权限指令 `iot:metric:*` 是否覆盖列表/查询/新增/批量生成/编辑/启停/删除各按钮；有无越权可见。
4. defineExpose 是否覆盖父级 ref 调用（避免 BUG-0008 复发）。
5. 空态是否诚实（无伪造指标/来源项列表）；批量生成 JSON 清单输入是否有前置字段口径提示、不伪造模板。
6. 是否有硬编码凭据、appKey/secret、直连 `/openapi/v1/**` HMAC、自造代理等安全红线问题。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0095-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条 / diff 哪一行）
- 一句话说明

无发现也要明确写「无 Critical / Important / Minor 发现」。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
