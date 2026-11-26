# 评审 brief：TASK-0097 点位管理页前端补齐

## 角色

- 代码评审（标准任务，改动 >3 文件、涉及 API 封装与权限，需即时 diff 评审）

## 必读（只给路径，不粘正文）

- 评审 brief：`reports/TASK-0097-review-brief.md`（本文件）
- TASK 详情：`tasks/TASK-0097.md`
- 实施 brief：`reports/TASK-0097-brief.md`
- diff 包：`reports/TASK-0097-diff.txt`（BASE 4e4c1461 → HEAD 449d7eb3，共 7 个前端文件）
- 实施报告：`reports/TASK-0097-report.md`
- 后端接口事实源：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/controller/IotPointController.java` 及同目录 `dto/` 下相关 Form/VO

## 越界反查（已由协调者机械核对，作为必查项）

`git diff --name-only 4e4c1461..449d7eb3` 结果 7 个文件，与实施 brief 改动范围声明逐项比对：

| 文件 | 是否在 brief 声明集内 |
| --- | --- |
| frontend/src/api/iot/point-api.ts | 是 |
| frontend/src/components/iot/iot-consts.ts | 是（brief 允许「复用常量」时改） |
| frontend/src/views/iot/point/index.vue | 是 |
| frontend/src/views/iot/point/components/point-form-modal.vue | 是（brief 允许新建） |
| frontend/src/views/iot/point/components/point-detail-modal.vue | 是（brief 允许新建） |
| frontend/tests/unit/api/iot/point-api.spec.ts | 是 |
| frontend/tests/unit/views/iot/point.spec.ts | 是 |

结论：无声明集之外文件。评审者仍须独立复核 diff 内容是否越界修改上述文件之外的地方。

## 依据

- `../L1-execute.md` 的「diff 评审」口径：看两样——① 与 TASK 验收标准 / brief 接口的一致性；② 明显质量缺陷（错误处理、边界、安全）。发现分级 Critical / Important / Minor。

## 评审重点提示（非结论，仅为定位）

1. point-api 各方法 URL / HTTP 动词 / 参数是否与 IotPointController 端点逐项对齐（尤其 generate 类路径、DELETE 路径参数 ids 的拼接、getDownload/postDownload 用法、silentError）。
2. 表单校验是否与后端 CreateForm/UpdateForm 一致；pointCode 编辑锁定；status 是否误入编辑提交；比值/枚举等 DTO 口径。
3. 权限指令 `iot:point:*` 是否覆盖列表/查询/新增/编辑/启停/删除各按钮；有无越权可见。
4. defineExpose 是否覆盖父级 ref 调用（避免 BUG-0008 复发）。
5. 空态是否诚实（无伪造曲线点/引用记录/字典选项）。
6. 是否有硬编码凭据、appKey/secret、直连 `/openapi/v1/**` HMAC、自造代理等安全红线问题。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0097-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条 / diff 哪一行）
- 一句话说明

无发现也要明确写「无 Critical / Important / Minor 发现」。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
