# TASK-0095 代码评审报告（fresh diff 评审）

- 评审对象：`reports/TASK-0095-diff.txt`（BASE `449d7eb3` → HEAD `09ae85cc`，7 个前端文件）
- 依据：`reports/TASK-0095-review-brief.md`、`tasks/TASK-0095.md`、`reports/TASK-0095-brief.md`、`reports/TASK-0095-report.md`、`.agents/protocol/L1-execute.md` §13，以及 `IotMetricController` 与 4 个 Form DTO / `IotMetric` 实体（事实源）。
- 契约：fresh、no-subagents、只评审不修改。

## 越界复核

实测 `git -C D:/workspace/WisFactory diff --name-only 449d7eb3..09ae85cc` 共 7 个文件，与实施 brief 声明集逐项一致：`frontend/src/api/iot/metric-api.ts`、`frontend/src/components/iot/iot-consts.ts`、`frontend/src/views/iot/metric/index.vue`、`frontend/src/views/iot/metric/components/metric-form-modal.vue`、`frontend/src/views/iot/metric/components/metric-detail-modal.vue`、`frontend/tests/unit/api/iot/metric-api.spec.ts`、`frontend/tests/unit/views/iot/metric.spec.ts`。无声明集之外文件，无后端/点位页改动。**无越界发现。**

## 评审重点逐条结论（简述）

1. **API 对齐**：7 端点 URL/动词/参数与 `IotMetricController` 逐项一致——`generate` 的 `instanceId` 走 query、`form` 走 body；`changeStatus` 传 `{ids,status}`；`remove` 数组逗号拼接路径参数，均匹配 `@RequestParam` / `@RequestBody` / `@PathVariable List<Long>` 签名。lint/typecheck 未见空参数漏传。无发现。
2. **表单校验**：metricCode(必填/≤96/`[A-Za-z0-9_]+`)、metricName(必填/≤100)、aggType(1-5)、period(HOUR/DAY/MONTH/YEAR)、calendarType 锁 1、比值型分子分母齐备、非比值型 ratio_* 置 undefined（JSON 序列化后为后端强制 NULL）、status 未入编辑集、`metricCode` 编辑态禁输且携带原值——与 CreateForm/UpdateForm 注解及 brief 口径一致。无发现。
3. **权限**：查询/重置=`iot:metric:list`，详情=`query`，新建/批量生成=`add`，编辑/启停/批量启停=`edit`，删除/批量删除=`remove`，逐按钮覆盖且无越权可见。无发现。
4. **defineExpose**：form modal 暴露 `{showModal, validateRatio}`、detail modal 暴露 `{showModal}`，父级 ref 调用路径均有单测覆盖，BUG-0008 未复发。无发现。
5. **空态与批量生成**：详情空值显示「-」、列表空数据空表、不造假日数据；批量生成有前置 alert 说明「模板载体未定义、按清单生成、instanceId 覆盖」且未伪造模板。无发现（但见下方 Important 关于新增态下拉的说明）。
6. **安全红线**：仅走 `/@/lib/axios` 管理面；无 appKey/secret/HMAC、无 `/openapi/v1/**`、无硬编码凭据、无自造代理。无发现。

## 发现清单

### Important

- **新增指标表单从不加载「来源数据项 / 比值分子 / 比值分母」下拉选项，比值型（aggType=5）指标无法通过新建表单提交**
  - 出处：`frontend/src/views/iot/metric/components/metric-form-modal.vue`，diff **第 393–406 行**（`showModal` 中 `loadItemOptions(form.instanceId)` 仅在 `if (rowData)` 编辑分支调用；对应模板 diff 第 216–236、255–274 行的三个数据项下拉，以及 diff 第 372–387 行 `loadItemOptions` 的定义）。新增态传 `undefined`，`itemOptions` 被置空且无任何实例选择 change/watch 触发重加载；`a-select` 无 `allowCreate`，用户无法录入 `sourceItemId`、`ratioNumItemId`、`ratioDenItemId`。后端 `validateAndBuild` / `validateRatio` 要求 aggType=5 时分子分母必填，因此「新建比值型指标」这一验收要求（TASK 验收标准「可在界面新建/编辑…指标」「比值型分子分母齐备」）在直接新建路径上不可达（仅能先建非比值型再编辑补救）。

## 结论汇总

- Critical：**无发现**
- Important：**1 项**（见上）
- Minor：**无发现**

按 L1-execute §13，Important 需退回实现方修复后做 scoped 复评。
