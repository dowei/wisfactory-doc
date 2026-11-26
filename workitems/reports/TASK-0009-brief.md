# 实施 brief：TASK-0009（字典批量导入与逐行回执 + 横切顺带项）

## 批次

- TASK-0009（本批共 1 个，标准任务 + 1 项横切顺带修复，backend 串行道）
- BASE：`509a82b`（主仓 HEAD，派发时回填）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/**`（字典导入域）
  - `backend/ot-core/src/main/java/net/otlab/core/config/MybatisPlusConfig.java`（横切顺带项，见裁决 5，仅一行）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_dict_import.sql`（新增：iot:dict:import 权限行）
  - 不碰 frontend/、ingest/（Go 道在跑）、既有字典/租户实现（复用不修改）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0009.md`
- 关联对象：同目录 `features\FEAT-0003.md`、`requirements\REQ-0003.md`、`testcases\TC-0003.md`
- 既有实现（复用范式）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java`（4002 双道闸/白名单构建/值域 @Pattern）、`IotDataDictController.java`（/iot/dict 五接口）、`IotDataDictServiceImplTest`
- 契约行：详细设计 §3.1 续 p23「POST /api/v1/iot/dict/import iot:dict:import 批量导入（Excel 模板 + 逐行校验回执，重复编码单独返回）」（brief 已转述，无需再提 PDF）
- 框架 Excel 能力：主仓 grep 现有 Excel/POI 组件（survey 模块导入导出应有先例——`grep -rn "Excel\|EasyExcel\|POI" backend/ot-admin/src/main/java --include="*.java" -l` 自行定位范式后复用）

## 接口与决策（逐条结论，不再讨论）

1. **导入接口**：`POST /iot/dict/import`（权限 `iot:dict:import`），multipart 上传 Excel（.xlsx）；单批上限 **5000 行**（超限整批拒绝，PARAM_ERROR 提示）；**失败整批拒绝不半落库**（任一行失败即全批回滚——TASK 正文明确，与逐行回执不矛盾：回执是给用户看的校验结果，落库原子性照旧整批）；空文件/无数据行拒绝。
2. **逐行回执**：响应体为逐行结果清单——行号、原始 data_code、结果（成功/失败+原因码与文案）；失败原因分类：**重复编码（4002 语义，批内互撞与库内已有均报）、类型非法（data_type 不在 DOUBLE/INT/BOOL/ENUM）、值域非法（category/data_type 不合 @Pattern 口径）、必填缺失（data_code/data_name/data_type）、超长（varchar 限长）**；回执可下载（错误行清单 .xlsx，复用同一 Excel 组件）。HTTP 200 + body code 200，data 携带回执结构（成败计数 + 行清单）——整批失败时 body code=4002 系？不：整批拒绝也是「校验不通过」语义，body code 用 PARAM_ERROR(400) 且 data 携带完整回执清单（与单接口 4002 语义区分开——4002 留给「编码重复」单点语义，批量口径在 data 内逐行标注）。
3. **模板下载**：`GET /iot/dict/import/template`（同一权限点）——单 sheet 表头（data_code/data_name/category/data_type/default_unit）+ 一行示例 + 说明行（值域口径写进批注或说明行）。
4. **批内互撞判定**：同一批内重复 data_code 直接标失败行（不必等落库才撞），与库内已有编码判定同口径（活跃行预检）。
5. **横切顺带项（两次评审点名后的升级处置）**：`MybatisPlusConfig` 的 `PaginationInnerInterceptor` 加 `setMaxLimit(500L)`（详细设计 §3.4「pageSize 上限 500」）——一行改动 + 全量回归确认无破坏；提交与本任务同 commit（信息注明含横切项）。
6. **事务与权限**：导入写方法 `@Transactional(rollbackFor = Exception.class)`；菜单种子 4023 一行（iot:dict:import，menu_type=3）挂 4008 页下 + 超管授权，幂等 INSERT IGNORE；TDD：先写测试（模板下载头/行数上限/逐行回执分类/批内互撞/整批回滚/错误清单下载）。

## 验收命令（整体验收）

环境事实：MySQL 127.0.0.1:3307 root/root 库 ot_wisfactory；Redis 6379；8098 端口纪律同前（探测/等待/用毕即停）。SQL 走 `bash backend/scripts/sql-exec.sh`（多条语句每条一行行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1200 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对 4023 行 perms=iot:dict:import 与 (1,4023) 授权
3. dev 冒烟（每步原始输出贴回）：admin 登录 → GET 模板下载（HTTP 200，文件头正确）→ 构造含 5 行的 Excel（1 成功/1 批内重复/1 类型非法/1 库内已有重复/1 超长）POST 导入 → 整批拒绝 + 回执逐行标注（五分类各就位）→ 修正后重导 → 200 + 成功计数 → GET /iot/dict/list 确认落库行数与内容 → 错误清单下载 200 → 分页 pageSize=1000 请求任意 list → 响应 size ≤500（maxLimit 生效）
4. prod 档真实启动回归 → 0 ERROR + list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0009）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
