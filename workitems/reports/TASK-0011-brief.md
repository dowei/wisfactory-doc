# 实施 brief：TASK-0011（iot_model_template 与模板数据项接口）

## 批次

- TASK-0011（本批共 1 个，标准任务 3 人天；backend 串行道）
- BASE：`9a4c6ef`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/**`（模板域新包）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_template_domain.sql`（新增）
  - 不碰 frontend/、ingest/、既有字典域（复用其校验器）与租户/权限组件

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0011.md`
- 关联对象：同目录 `features\FEAT-0004.md`、`requirements\REQ-0004.md`（验收 1/3/4/5 与非目标——平台预置与跨租户复制降级二期）、`testcases\TC-0004.md`
- 决策：`decisions\ADR-0011.md`（TD-0008 强制模板、TD-0011 **default_calc_period 补字段**、TD-0012 属性参数收窄）；`decisions\ADR-0003.md`（字典编码唯一来源）
- 字典校验器（接线对象）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/IotDataDictService.java`（validateForTemplateRef 三态——0010 交付）+ `reports\TASK-0010-review.md` Minor①②（javadoc 保序契约与入参上限随本任务接线消化：校验器 javadoc 补返回 Map 按 dataCode 保序说明、模板保存批量上限调用侧约束）
- 表结构权威（pypdf 自行提取核对）：详细设计 V1.1 PDF §1.5（p10 模板字段表）/ §1.6（p13 数据项模板字段表）/ 附录 A（p31 iot_model_template、p32 iot_data_item_template DDL）；接口行 §3.1（p21-22：/api/v1/iot/template 与 /template/item 各接口权限 iot:template:list/add/edit）
- 既有范式：iot 字典域五接口与 4002 双道闸（`IotDataDictController/Impl`）、字段白名单构建、菜单种子幂等（4018-4023 已占，**你从 4024 起**）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_template_domain.sql，全部 CREATE TABLE IF NOT EXISTS 幂等 + 「与上游差异」头注）

1. `iot_model_template`：附录 A p31 逐字。
2. `iot_data_item_template`：附录 A p32 逐字 **+ ADR-0011(TD-0011) 补字段 `default_calc_period varchar(16) DEFAULT NULL COMMENT '默认计算周期（如 MINUTE/HOUR/DAY，ADR-0011 补字段，上游漏项）'`**（脚本头「与上游差异」注明）。
3. `iot_model_instance`：附录 A p31 逐字**建表先行**（脚本头注明：表结构随本任务建库，实例接口/编码规则/绑定流程全部归 TASK-0012）——删除保护需要引用计数查这张表。
4. 菜单种子：4024-4029 六行（iot:template:list/query/add/edit/remove + iot:template:item:edit 数据项编辑），挂 4009 模板管理页下，幂等 + 超管授权；**权限标识与接口清单一一对应**（后端守卫测试会强制）。

### 代码结构

- 新包 `net.otlab.admin.module.iot.template`：IotModelTemplate / IotDataItemTemplate 实体（继承 BaseEntity）+ Mapper + Service/Impl + Controller。
- 接口（/iot/template 扁平段，照 §3.1 语义）：
  - `GET /iot/template/list`（iot:template:list）分页 + 过滤 templateCode/templateName/nodeType/status
  - `GET /iot/template/{id}`（iot:template:query）详情（含数据项清单可选聚合）
  - `POST /iot/template`（iot:template:add）template_code 唯一双道闸 4002；node_type 必填 1-5；tenant_id 服务层会话覆盖（**平台预置模板 tenant_id=null 的创建入口一期接口降级**——接口仅允许租户模板；平台预置走 SQL 种子，REQ-0004 非目标口径）
  - `PUT /iot/template`（iot:template:edit）template_code 不可改（拒绝）；白名单构建
  - `DELETE /iot/template/{ids}`（iot:template:remove）**删除保护：有实例引用（iot_model_instance.template_id）或被数据项外引用时拒绝**（引用计数查询，提示引用来源）；逻辑删
  - `GET /iot/template/item/list?templateId=`（iot:template:list）按模板查数据项
  - `POST /iot/template/item`（iot:template:item:edit）批量保存数据项（一次调用 N 项——**接线 0010 校验器**：全项 data_code 收集后一次 validateForTemplateRef，NOT_REGISTERED/DISABLED 按文案拒整批不落库，4002 语义映射在调用方）；单批上限 500 项（消化 0010 评审 Minor②）；(template_id, data_code) 批内互撞先判
  - `PUT /iot/template/item`（iot:template:item:edit）编辑：**data_code 不可改**；白名单
  - `DELETE /iot/template/item/{ids}`（iot:template:item:edit）逻辑删
- **default_formula 校验口径（裁决）**：一期本任务只做「非空时长度 ≤2000 与基础格式占位校验」，**Aviator 真实语法校验归 FEAT-0012 校验器交付时接线**（任务正文「公式语法校验调 FEAT-0012 校验器」——该组件尚不存在；0011 遗留登记接线义务）。
- 字段白名单构建 + 4002 双道闸 + `@Transactional(rollbackFor = Exception.class)` 照既有范式；tenant_id 会话覆盖（模板实体 tenant_id 可空——服务层对新建强制写会话租户，null 仅 SQL 种子可产生）。

### 测试口径（TDD）

- 服务层：模板 CRUD 唯一性（同租户撞码 4002/平台级 template_code 全局唯一语义不测——降级口径）、改码拒绝、删除保护（种子实例行引用→拒/无引用→删）、数据项批量保存（字典三态拒绝整批/批内互撞/超 500 项拒/白名单落库 status 与审计字段）、改 data_code 拒绝。
- 控制器：路由/权限注解/参数校验（node_type 1-5 边界）。
- 冒烟（dev，真实库）：admin 登录 → 建模板 → 批量保存数据项（含 1 个未登记码→整批拒 + 文案「未登记」；1 个已停用码→「已停用」）→ 字典补登记后重存成功 → 改码拒 → 删除有引用模板拒（先手工种一行 iot_model_instance）→ 无引用删除成功 → prod 启动回归。

## 验收命令（整体验收）

环境事实同前（MySQL 3307 root/root / Redis 6379 / 8098 端口纪律 / sql-exec.sh 行尾分号纪律）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1260 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对三表在位（SHOW TABLES LIKE 'iot_model%' / 'iot_data_item_template'）+ default_calc_period 列在位 + 4024-4029 六行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + /iot/template/list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0011-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0011）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
