# 实施 brief：TASK-0014（实例参数覆盖与阈值继承链）

## 批次

- TASK-0014（本批共 1 个，标准任务 2 人天小体量；backend 串行道）
- BASE：`4d7246d`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/**`（覆盖接口与继承链，数据项域内）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - 不碰 frontend/、ingest/、模板域、instance 域（只读）、dataperm

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0014.md`
- 关联对象：同目录 `features\FEAT-0004.md`、`requirements\REQ-0005.md`（继承链验收）、`testcases\TC-0004.md`
- 决策：`decisions\ADR-0011.md`（TD-0022 valid/alarm 分工：valid→质量码 1/2、alarm→展示标红；TD-0012 属性参数一期收窄为阈值类+状态规则门限）；`decisions\ADR-0008.md`（模板变更三档——覆盖与模板默认值变更的关系）
- 时序倒挂消解口径（任务正文）：时间线表 iot_data_item_history 由 TASK-0039 交付——**本任务覆盖动作的时间线 PARAM 记录以接口钩子形式预留**（服务方法占位 + javadoc 声明接线点 + 遗留登记），不建表不写历史行
- 既有实现：`iot/dataitem/**`（0013 全部交付：实体含 valid_min/valid_max/alarm_low/alarm_high 四列——实例行已可存覆盖值）、`iot/template/**`（default_ 四字段只读源）

## 接口与决策（逐条结论，不再讨论）

1. **覆盖接口**：`PUT /iot/instance/{instanceId}/items/{itemId}/params`（权限 `iot:item:edit`——复用 4040 数据项编辑权限点，不新增）——body 仅接受 valid_min/valid_max/alarm_low/alarm_high 四字段（partial：null=清除覆盖回归继承；数值=覆盖）。**覆盖语义=仅写实例行四列**（0013 已交付的列），不改模板、不复制规则行（任务正文：状态规则门限覆盖值随实例数据项保存，规则行不复制——状态规则门限载体归 FEAT-0016/模板配置，本期覆盖接口只处理 valid/alarm 四字段）。
2. **继承链读取（运行时继承，非复制落库——任务正文硬要求）**：`GET /iot/instance/{instanceId}/items/{itemId}/effective-params`（权限 `iot:item:query`）——返回有效参数：实例行有值→实例值；否则→模板 default_ 同名值；再否则→null（不判定）。响应体区分来源（INSTANCE/TEMPLATE/NONE 三态枚举）供前端展示继承来源。
3. **校验**：min<=max（valid 对与 alarm 对各自成立）；负值/超 decimal(20,6) 范围拒绝（PARAM_ERROR）；四字段全 null=清除覆盖（合法，回归继承）。
4. **时间线钩子**：覆盖成功后调用 `IotDataItemHistoryService.recordParamChange(...)` 占位接口（本任务新建接口与 NoOp 实现——0039 交付真实实现后零改动接线）；javadoc 声明接线点与 change_type=PARAM 语义。
5. **数据权限**：覆盖与读取两接口均为写/读单实例资源——**接线 0012 写路径范式**：resolve 校验目标实例 ∈ 可见集（403 照 0012 修订轮口径）；批量覆盖不做（一期单项）。
6. **TDD**：覆盖生效仅本实例（兄弟实例不受影响）、清除覆盖回归继承、继承链三态、min>max 拒、时间线钩子被调用（NoOp 断言）、数据权限 403。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。无新表无菜单种子（复用权限点）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1449 + 新增，Results 段贴回）
2. dev 冒烟（每步原始输出贴回）：admin 登录 → 取 0013 冒烟数据（或新建链）→ 覆盖某项 valid_min/max → effective-params 显示 INSTANCE 来源与值 → 兄弟实例同码项不受影响 → 清除覆盖 → 回归 TEMPLATE 来源 → min>max → 400 → 受限用户（0012 夹具范式）覆盖不可见实例项 → 403
3. prod 档真实启动回归 → 0 ERROR + effective-params 200
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0014-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0014）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
