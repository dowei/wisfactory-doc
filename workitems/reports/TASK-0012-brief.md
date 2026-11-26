# 实施 brief：TASK-0012（iot_model_instance 与实例树接口：绑部门、权限裁剪）

## 批次

- TASK-0012（本批共 1 个，标准任务 4 人天；backend 串行道；**数据权限组件首个真实消费方**）
- BASE：`c94c9b7`（主仓 HEAD，派发时回填）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/**`（实例域新包）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/service/IotModelTemplateServiceImpl.java`（**仅一处**：updateTemplate 补「有实例引用禁改 node_type」守卫——0011 评审 M-6 裁决随本任务落地）+ 其测试类
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_instance_columns.sql`（新增）
  - 不碰 frontend/、ingest/、dataperm 组件本体（复用）、字典域

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0012.md`
- 关联对象：同目录 `features\FEAT-0004.md`、`requirements\REQ-0004.md`（验收 2/5）、`requirements\REQ-0005.md`（实例树权限裁剪验收）、`testcases\TC-0004.md`
- 决策：`decisions\ADR-0011.md`（TD-0008 强制模板、TD-0010 挪树一期禁止）、`decisions\ADR-0016.md`（裁决 3/6：未绑部门 fail-closed、组件消费口径）；`reports\TASK-0005-inputs.md`（§4 步骤 3-5 与点位/实例过滤语义）
- 组件（消费对象）：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/**`——DataPermService.resolve(userId) → VisibleScope{allTenant/deptIds/instanceIds/createByOnly}；javadoc 与 `reports\TASK-0010-review.md` 的保序契约（如调校验器）参照
- 既有范式：0011 模板域（`iot/template/**`——uk 双道闸/白名单/删除保护/守卫测试）；iot_model_instance 表已由 0011 建库（**本任务不重建**，实体已存在 `iot/template/entity/IotModelInstance.java`——按需扩展，javadoc 声明在案）

## 接口与决策（逐条结论，不再讨论）

### 增量脚本（20261122_iot_instance_columns.sql，information_schema 判存幂等 + 「与上游差异」头注）

1. iot_model_instance 补两列（TASK 正文指定，PDF 无出处——「与上游差异」头注登记）：`applied_template_version int DEFAULT NULL COMMENT '已应用的模板版本（实例版本锁定，REQ-0005）'`、`locked tinyint DEFAULT 0 COMMENT '实例锁定 0 未锁 1 已锁（锁定后配置变更受限，REQ-0005）'`。
2. 菜单种子 **4030 起**连续（4018-4029 已占）：iot:instance:list/query/add/edit/remove + iot:instance:tree（树接口单设权限点），挂 4010 实例管理页下，幂等 + 超管授权。权限与接口一一对应（守卫测试强制）。

### 代码结构（/iot/instance 段，语义照 §3.1 p22）

- 实体扩展：IotModelInstance + 两新字段（0011 已建实体 javadoc 授权扩展）。
- **创建** `POST /iot/instance`（iot:instance:add）：
  - 选模板语义（TD-0008）：node_type=4 设备/5 组件时 **template_id 必填**且模板存在、status=1、模板 node_type 与实例 node_type 一致（REQ-0004 验收 5「空压机模板只能建设备级实例」）；node_type=1 工厂/2 车间时 template_id 可空（纯组织节点，§1.5 口径）；3 产线可选。
  - 路径式编码：请求只传本级短码（shortCode），服务层拼 `parent.instance_code + "." + shortCode`；顶级实例直接用 shortCode 为全码；shortCode 字符集 `[A-Za-z0-9_]`（防路径分隔符注入编码体系）、长度约束使全码 ≤96；uk_tenant_instance_code 双道闸 4002（预检活跃行+已删行均占码——0011 修订轮口径同款）。
  - ancestors 维护：parent.ancestors + "," + parentId（顶级="0"）；parent 存在性/同租户/status=1 校验。
  - 绑部门：node_type=1/2 时 **dept_id 必填**（服务层防线——ADR-0016 开放点 10 口径；dept 存在性/同租户/status 校验）；设备/组件级 dept_id 可空（可从模板实例生成链继承，归 0013）。
  - 字段白名单（tenant_id 会话覆盖、applied_template_version 创建时=模板当前 version（选了模板时）、locked 创建时=0、status 强制 1、审计字段框架填充）。
- **编辑** `PUT /iot/instance`（iot:instance:edit）：**instance_code/parent_id/ancestors/node_type/template_id 不可改**（TD-0010 挪树一期禁止——改父即改全键前缀；请求携带不一致一律拒绝）；白名单仅 instanceName/deptId/sort/status/remark 类；**锁定语义**：locked=1 时编辑仅允许 status/remark（其余拒，REQ-0005 锁定口径）；status 0/1。
- **删除** `DELETE /iot/instance/{ids}`（iot:instance:remove）：有子实例（parent_id 引用）→ 拒（提示子实例编码）；有数据项引用 → 拒（0013 交付后自然生效，本期该计数恒 0 不影响）；批内缺失 fail-closed；逻辑删。
- **树查询** `GET /iot/instance/tree`（iot:instance:tree）：**权限裁剪接线（本任务核心）**——调 DataPermService.resolve(当前用户) → VisibleScope；allTenant=true 返回全树；否则**只保留 instanceIds 集合内的节点及其到根的路径节点**（路径节点仅作骨架展示、无权兄弟节点不返回——REQ-0002 验收「实例树接口无权节点不返回」）；createByOnly=true 时数据查询类场景按空树返回（实例树属数据查询语义，ADR-0016 裁决 3/4：仅本人对数据类=空集）。**实现在服务层做内存裁剪**（全树一次查出后按可见集过滤），排序 sort 升序；禁用节点照常返回（status 字段携带）。
- **列表** `GET /iot/instance/list`（iot:instance:list）：分页 + 过滤 instanceCode/instanceName/nodeType/status/deptId；**数据权限裁剪同样生效**（§3.1 未标注接口走 §4 过滤——instanceIds IN 条件拼装，照 ADR-0016 裁决 6 点位类形态的实例版：`id ∈ 可见集`；createByOnly → 空集）。
- **详情** `GET /iot/instance/{id}`（iot:instance:query）：详情类 403 语义（ADR-0016 裁决 2——SCOPE_FORBIDDEN 常量首个真实消费点）。
- **模板 node_type 守卫（M-6）**：IotModelTemplateServiceImpl.updateTemplate 增「有实例引用（iot_model_instance.template_id）时 node_type 不可改」守卫（拒绝文案含引用计数）——一行防线，测试补红转绿。

### 测试口径（TDD）

- 服务层：创建（四类 node_type 模板约束/短码拼接全链=F01→F01.WS01→F01.WS01.EQ001/ancestors 正确/同码 4002 含已删行/dept 必填边界/locked 编辑限制）、编辑不可改五字段、树裁剪（allTenant 全树/受限用户只见可见集+根路径/无权兄弟不返回/createByOnly 空树）、列表裁剪、详情 403、删除子实例保护、模板 node_type 守卫红转绿。
- 控制器：路由/权限注解/参数校验。
- 冒烟（dev）：admin 建工厂（绑 dept 1）→ 建车间（父=工厂，绑 dept）→ 建设备（选模板）→ 全码/ancestors DB 核对 → 造受限角色（照 0055 夹具范式）授权 iot:instance:tree 但数据权限仅车间级 → 该用户树只见车间子树+根骨架 → admin 全树 → 编辑改 parent 拒 → 锁定后编辑受限 → 删除有子实例的工厂拒。

## 验收命令（整体验收）

环境事实（**新**）：MySQL/Redis 由 WSL docker 容器承载（wisfactory-mysql/wisfactory-redis，restart=unless-stopped；`wsl docker ps` 查看；sql-exec.sh 直连 127.0.0.1:3307 正常）。其余同前（8098 端口纪律、sql-exec 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1331 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对两新列 + 4030-4035 六行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + /iot/instance/tree 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内（template 服务一处守卫为授权例外）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0012-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0012）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
