# 实施 brief：TASK-0010（模板数据项字典校验联动——公共校验器先行）

## 批次

- TASK-0010（本批共 1 个，标准任务小体量；与 0009 复评（只读）并行——文件集不相交）
- BASE：`9045f38`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dict/**`（校验器落位，或按既有 iot 包组织——实体/服务所在包）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增测试）
  - 不碰 frontend/、ingest/、模板域（模板域尚不存在——校验执行点归 TASK-0011）、既有字典 CRUD（复用不修改）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0010.md`
- 关联对象：同目录 `features\FEAT-0003.md`、`requirements\REQ-0003.md`、`testcases\TC-0003.md`
- 上下游契约：`tasks\TASK-0011.md`（模板域——校验执行点将在其模板保存流程接线，读本任务了解调用方需求即可，不改它）；`reports\TASK-0008-report.md` 遗留 5（0008 对 0010 的联动提示）
- 既有实现：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java`（IN 预检/状态语义范式——校验器与它同口径）、`IotDataDict.java`（status 0 禁用 1 启用、@TableLogic deleted）

## 接口与决策（逐条结论，不再讨论）

1. **公共校验器**：`IotDataDictService` 新增 `Map<String, DictCodeStatus> validateForTemplateRef(Long tenantId, Collection<String> dataCodes)`（或等价形态：入参编码集合，返回每个编码的核对状态）——状态三值：**OK / NOT_REGISTERED（未登记）/ DISABLED（已停用）**；deleted 行按未登记口径（@TableLogic 自动过滤）。批量入口设计：0011 的模板保存是「一次保存 N 个数据项」，一次 IN 预检回填，不逐个 count。
2. **文案区分（任务正文明确）**：报错文案区分「未登记」（`数据编码未在字典登记: <code>`）与「已停用」（`数据编码已停用: <code>`）——供 0011 模板保存报错直接复用；错误码口径：**校验器只返回状态不抛异常**，异常与 4002 映射由调用方（0011）决定——校验器保持纯查询语义，便于测试与多场景复用。
3. **租户口径**：tenant_id 显式入参（0011 调用点在模板保存流程，届时租户上下文已定——校验器不做会话假设，保持组件纯函数风格）；租户隔离由查询条件显式携带（与 0002 旁路方法同款显式条件风格——注意：本查询走白名单表 iot_data_dict，插件会自动注入，但显式入参 tenantId 仍保留并在查询中体现，双保险且语义自洽；如插件注入导致条件重复以实测为准，报告里说明）。
4. **测试口径（TDD）**：空集合返回空；未登记/已停用/OK 三态各断言；批内混合（1 OK + 1 NOT_REGISTERED + 1 DISABLED）；deleted 行=NOT_REGISTERED；跨租户编码不可见（插件注入 or 显式条件——以实现为准断言）；文案常量与三态枚举的存在性。
5. **本任务不接线**：无 HTTP 端点/菜单种子/冒烟（校验执行点归 0011）；验收=单测+全量回归。

## 验收命令（整体验收）

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1243 + 新增，Results 段贴回）
2. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0010-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0010）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
