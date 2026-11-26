# 实施 brief：TASK-0005（数据权限翻译组件（六步）实现）

## 批次

- TASK-0005（本批共 1 个，标准任务；backend 串行道，派发时前序任务须已收口）
- BASE：`d971c10`（主仓 HEAD，派发时回填）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/**`（新增，组件包）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataperm/**`（新增）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增 `ot.dataperm.subtree-max` 一个配置键）
  - 不碰 frontend/、ingest/、既有登录链路与拦截器

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0005.md`
- 关联对象：同目录 `features\FEAT-0002.md`、`requirements\REQ-0002.md`、`testcases\TC-0002.md`
- **实现口径权威：`decisions\ADR-0016.md`（七项裁决逐条照做，含配置键名、日志格式、scope JSON 结构、NULL 归类、标签名）**
- 设计素材（§4 全文 + 表结构摘要 + 五种 scope 语义）：`reports\TASK-0005-inputs.md`（p27 §4；p4 五种 data_scope 顺序以 DataScopeEnum 为准：1全部/2本部门及下属/3本部门/4仅本人/5自定义）
- 多租户设施：`backend/ot-core/src/main/java/net/otlab/core/tenant/OtTenantLineHandler.java` 与 `TenantContext`（组件消费会话租户上下文，不重复实现）
- 边界参考：`backend/ot-admin/src/main/java/net/otlab/admin/module/system/service/DataScopeService.java`（框架既有部门范围服务——语义对齐但组件自包含，见裁决 3）

## 接口与决策（逐条结论，不再讨论）

1. **组件落位**：`net.otlab.admin.module.iot.dataperm` 新包（iot 公共权限组件）；**自包含设计**——组件内部自带所需只读 Mapper（sys_dept/sys_role/sys_role_dept/iot_model_instance 的 select，全部参数化 SQL），不调用框架 DataScopeService，不依赖 iot 域业务服务。理由与迁移口径：数据服务工程尚未立项，组件以「只依赖只读 Mapper 与配置」为边界保证可整体迁出；TASK-0051 前数据服务工程立项时迁为共享 Maven 模块（届时 @MapperScan 相应放宽），单实现原则由「未来迁出」而非「现在拆模块」保证。
2. **六步实现**（步骤语义照 ADR-0016 七项裁决执行，此处只列组件形态）：
   - 入口 API：`DataPermService.resolve(Long userId)` → `VisibleScope { boolean allTenant; Set<Long> deptIds; Set<Long> instanceIds; boolean createByOnly; }`（含 ADR-0016 第 4 条的并集合并语义）
   - 条件拼装 API：按表类别三态——配置类（instance_id IN 或 create_by 分支）/ 点位类（dept_id IN OR bound_instance_id IN，NULL 归类按第 6 条）/ 时序类（instance_id IN；scope=4 无贡献）；TDengine 下推提供标签条件渲染器（标签名按第 7 条：instance_key/dept_id），渲染结果为条件片段供消费方拼 SQL
   - 策略常量：SCOPE_EMPTY / SCOPE_FORBIDDEN（第 2 条判定规则；组件提供判定辅助 `isDetailAccess(resource)` 按接口形态，最终选择权在接口实现——本任务不接线）
   - 子树展开上限：`ot.dataperm.subtree-max` 默认 5000，截断 + WARN（第 1 条日志格式）
   - 步骤 6 缓存：只定义 `VisibleScopeCache` 接口 + NoOp 实现（本期直查），Redis 实现与主动失效归 TASK-0006——resolve 调用路径已按「先缓存后计算」写好接缝
3. **未绑部门 fail-closed（TD-0043）**：按 ADR-0016 第 6 条结构性满足，不特判 NULL。
4. **测试口径（TDD）**：组件级单测覆盖——五种 data_scope 各自可见集合正确；多角色并集（含 4 与部门角色混合的 OR 分支）；scope=1 直接全租户（跳过部门求解）；自定义 scope 查 sys_role_dept；ancestors 前缀展开（含多级）；子树上限截断；中台 scope JSON 三态（null=拒绝/subtree=false 精确/subtree=true 展开）；点位 NULL 行对非 scope=1 不可见。测试数据自建（内存 H2？不——沿用 standalone Mockito 范式 mock Mapper； ancest或前缀逻辑用真实字符串断言）。
5. **本任务不接线任何 HTTP 端点**（T15 无独立页面、以接口测试验收——接线归 TASK-0006/0007/0012/0051）；因此无菜单种子、无权限点、无 dev 冒烟，验收=单测+全量回归。TC-0002 红线的集成验证归 TASK-0007（组件级等价断言已含在单测）。

## 验收命令（整体验收）

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 + 新增，Results 段贴回）
2. `mvn test -pl ot-admin -Dtest='DataPerm*'` 单独复跑贴回（组件测试类数与用例数）
3. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0005-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0005）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
