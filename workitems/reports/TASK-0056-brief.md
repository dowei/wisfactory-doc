# 实施 brief：TASK-0056（管理页第一组——租户/字典/模板/实例/网关/通道基础 CRUD 页）

## 批次

- TASK-0056（本批共 1 个，标准任务 5 人天；**frontend 独立道**——与 backend 0019 并行，pnpm 道与 mvn 道物理不相交）
- BASE：`fa12df9`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `frontend/src/views/iot/**`（既有占位页升级为完整页）
  - `frontend/src/api/iot/**`（API 封装扩展）
  - `frontend/src/components/iot/**`（如需公共组件，新增）
  - 不碰 backend/、ingest/、router 既有文件（菜单驱动动态注册已就绪）、views 非iot 目录

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0056.md`
- 关联对象：同目录 `features\FEAT-0018.md`、`requirements\REQ-0034.md`、`testcases\TC-0018.md`
- 已交付后端接口（对接权威，controller 为准）：`iot/tenant`（0001，/tenant→注意 0018 已迁移 /iot/tenant）、`iot/dict`（0008-0010）、`iot/template + /iot/template/item`（0011）、`iot/instance`（0012：tree/list/detail/params 覆盖与 effective-params 0014）、`iot/gateway`（0017）、`iot/channel`（0018）——**逐个读 controller 确认路径/参数/响应形状再写 api 封装，不猜**
- 既有前端资产：`views/iot/tenant/index.vue`（0055 可用骨架——本任务升级为完整页范本）、`api/iot/tenant-api.ts`、`views/iot/` 九个占位页、`views/system/dict` 等范式页、`directives/privilege.ts`
- 权限位现状：iot:tenant:*（4002-4007）/iot:dict:*（4018-4022+4023 import 未开放）/iot:template:*（4024-4029 含 item:edit）/iot:gateway:*（4041-4045）/iot:channel:*（4046-4050）/iot:instance:*（4030-4035）——按钮 v-privilege 照 0055 范式

## 接口与决策（逐条结论，不再讨论）

1. **页面范围（本期交付完整 CRUD 的六域）**：租户、字典（iot_data_dict——注意与框架 sys_dict 页并存，本页是业务数据字典）、模板（含模板数据项子表管理 + 状态规则配置入口占位——iot_status_rule 后端未交付，入口按钮 v-privilege 占位点击提示「待 TASK-0048 交付」）、实例（树 + 列表 + 详情 + 数据项清单 + 参数覆盖表单调 effective-params/params）、网关、通道（含 conn_config JSON 编辑 + 协议类型联动 Schema 前置提示——后端校验为准，前端做基础格式提示）。点位/指标两域后端未交付——**占位页维持**（0021/0043 交付后再升级）。
2. **后端联调不在本轮**（关键裁决）：backend 串行道正被 0019 占用，**8098 本轮不归前端**——本轮验收=构建/静态检查/组件级证据；真实接口联调归集成轮（FEAT-0006 全齐后的 0068 或专项联调）。API 封装以 controller 代码为准（静态正确性），运行期验证留集成轮。
3. **通用交互范式（任务正文）**：表单校验与后端一致前置提示（编码字符集/必填/长度——从 controller DTO 校验注解对齐）；停用/删除二次确认；乐观锁冲突提示=预留（iot 域无 @Version 字段——提示逻辑组件化但登记不触发）；空态/分页上限对齐后端 maxLimit=500。
4. **v-privilege 权限位**：每页按钮绑定对应 iot:*:* 权限位（对照菜单种子表）。
5. **测试/验收口径**：`pnpm build:test` EXIT=0、`pnpm lint` 0 error、`pnpm typecheck` 过——三条静态门；另交付「页面-接口映射核对表」（每页调用的 API 清单 vs controller 注解逐条对齐，贴报告）作为运行期联调前的静态证据。**不要求**起 8098/8099 冒烟。

## 验收命令（整体验收）

1. `cd frontend && pnpm build:test` → EXIT=0（built 成功，chunk warning 可忽略）
2. `cd frontend && pnpm lint` → 0 error
3. `cd frontend && pnpm typecheck` → 通过
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0056-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题（含页面-接口映射核对表）。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件；不碰 backend/ 与 ingest/。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0056）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
