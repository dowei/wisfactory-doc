# 实施 brief：TASK-0007（越权红线自动化测试——本期管理面红线固化 + 中台部分挂接线义务）

## 批次

- TASK-0007（本批共 1 个，标准任务 3 人天；backend 串行道——数据权限链收官环/架构验收级）
- BASE：`1065f9a`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/redline/**`（红线测试套件新包——夹具+用例集）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（既有测试如需复用夹具的最小调整）
  - 不碰 main 代码（本任务原则上零主代码改动——若红线测试暴露主代码缺陷，停下写报告并标注「疑似主代码缺陷+复现路径」，等协调者裁决，不自行修）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0007.md`
- 关联对象：同目录 `features\FEAT-0002.md`、`requirements\REQ-0002.md`（验收 5 条——红线一/红线二/五种 scope 各建账号验证/改绑定即时生效/点位原始数据查询受控）、`testcases\TC-0002.md`（六步骤）
- 决策：`decisions\ADR-0016.md`（七项裁决+增补 8/9——判定规则 SCOPE_EMPTY/SCOPE_FORBIDDEN、中台 scope JSON、混合角色合并）
- 被测面（当前已交付的全部查询/写接口）：`iot/instance/**`（tree/list/detail/params——0012/0014）、`iot/dataperm/**`（组件）、失效链（0006）
- 夹具先例：`frontend` 道 0055 与 0012 冒烟的受限角色范式（dev 库已有 iot_view_t0055 等夹具——测试内自建夹具不依赖 dev 库手工数据，standalone/集成口径自选并在报告声明）

## 接口与决策（逐条结论，不再讨论）

1. **本期可固化红线（管理面，实例域全覆盖）**：
   - 红线一-A：工厂 A 账号（scope=本部门及以下，绑 A 厂 dept）调 `GET /iot/instance/tree` → 不含 B 厂节点；`GET /iot/instance/list` → 空/仅 A；`GET /iot/instance/{B 厂 id}` → 403
   - 红线一-B：写路径同口径（0012 修订轮已封）——A 账号 PUT B 厂 deptId/DELETE B 厂叶子/在 B 厂父下 create → 全 403（写原语穿透回归防线）
   - 未绑部门 fail-closed：非 scope=1 用户查 dept_id 为空的实例 → 不可见（实例树/列表均不返回）
   - 仅本人语义：纯 scope=4 用户 instance 树/列表空集（数据查询语义，ADR-0016 裁决 3）
   - 混合角色：scope=3+scope=4 并存 → 部门集合可见 + 列表含 create_by=me 自建（0012 修订轮口径回归固化）
   - 失效即时性：改绑定/角色后权限变更即时生效（0006 缓存链——缓存失效后立即重查）
2. **中台 scope 红线（红线二）本期不可集成自动化**——依赖 iot_api_client 表与 /openapi/v1 拉树接口（TASK-0051/0052 未交付）。**接线义务裁决**：0005 组件的中台 scope 解析三态已有单测等价断言（DataPermClientScopeTest）——本任务在红线套件中登记「红线二-中台部分」占位测试类（@Disabled 或显式 skip + javadoc 指向 0051 接线时启用），TASK-0051 brief 将带红线二集成用例义务。验收口径相应调整：**红线一（管理面）+ 附加场景全绿 = 本任务验收**；红线二标记「待 0051 接线后转正」。
3. **夹具设计**：`RedlineFixture` 独立夹具类——两厂实例树（A 厂 dept1/B 厂 dept2，各自子节点）+ 三账号（A 厂长 scope=2、B 厂长 scope=2、纯 scope=4 用户）+ Sa-Token 会话模拟（照 0012 测试范式）；夹具与用例全部 standalone/集成口径**不依赖 dev 库手工数据**（自建数据、用毕清理）。
4. **CI 接入口径**：用例进标准 `mvn test` 套件即随 CI（仓库现状无独立 CI 配置文件——「CI 接入」实现为：红线套件命名规范 `*RedlineTest` + surefire 默认包含 + 在套件包 javadoc 声明「上线前越权专项=本套件全绿」；若发现 surefire 配置排除该包名再调整配置）。
5. **架构验收登记**：验收标准「上线前越权专项测试通过」——本任务交付套件；上线前专项执行归 REQ-0032 收口核对项（登记即可）。

### 测试清单（红线套件，全部先构造越权尝试再断言拒绝）

照「测试清单」八个场景（上面第 1 条逐项）各建独立测试方法；每个查询接口（tree/list/detail/params）至少一条越权用例——架构风险防法「每个查询接口必须有越权用例」。

## 验收命令（整体验收）

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1537 + 新增，Results 段贴回）
2. `mvn test -pl ot-admin -Dtest='*RedlineTest'` 单独复跑 → 全绿（套件类数与用例数贴回）
3. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内（**原则上零主代码改动**——若出现主代码 diff 逐文件说明理由）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0007-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 若红线测试暴露主代码缺陷：**停下报告**（缺陷+复现路径），由协调者裁决立 BUG 走独立修复——禁止顺手修主代码。
- 允许 git commit（main，注明 TASK-0007）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
