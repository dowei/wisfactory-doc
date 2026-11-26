# 实施 brief：TASK-0036（绑定/公式/汇总/阈值配置接口与引用边维护）

## 批次

- TASK-0036（本批共 1 个，标准任务 4 人天；backend 串行道——数据项配置链核心，影响预览的接线对象）
- BASE：以开工时主仓 HEAD 为准（backend 道串行，开工时 git rev-parse --short HEAD 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/**`（配置接口扩展——0013/0014 域内）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/**`（如状态规则门限覆盖需触达实例项读取——最小改动）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - 不碰 frontend/、ingest/、模板域（模板默认值只读）、metric 域（0043 另行）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0036.md`
- 关联对象：同目录 `features\FEAT-0012.md`、`requirements\REQ-0015.md`（valid/alarm 分工）、`requirements\REQ-0016.md`（公式语法/循环依赖 4001）、`requirements\REQ-0017.md`（汇总口径）、`testcases\TC-0012.md`
- 决策：`decisions\ADR-0008.md`（模板变更三档——配置变更对已建实例的分档生效）、`decisions\ADR-0009.md`（TD-0013/0014 状态稳定字典+直绑映射、TD-0021 质量码 0-10）、`decisions\ADR-0010.md`（TD-0017/0038 JUMP 关、死区不启用；TD-0019/0020 换表选 b、双给出口码位 10）
- 既有实现（口径基准）：`iot/dataitem/**`（0013 生成链：RelativeRefExpander 同口径校验、ref 依赖图、InstanceItemsGeneratedEvent；0014 参数覆盖与时间线钩子）
- 影响预览接线对象：0057 公式编辑器前端已交付（影响预览占位按钮）——本任务的引用图反查 API 即其接线对象（**brief 裁决 6**）
- 表结构：iot_data_item（0013 建库：source_type/formula/rollup_method/calc_period/valid_min/max/alarm_low/high/enabled）+ iot_data_item_ref（依赖图）

## 接口与决策（逐条结论，不再讨论）

1. **三类取值配置接口**（数据项级 PUT，或按 §3.1 实际行——pypdf 提取 §3.1 数据项配置段对齐）：
   - `PUT /iot/dataitem/{itemId}/binding`（source_type=1 绑定点位）：point_id 必填、点位存在/同租户/status=1；绑定即写 ref（ref_type=2 点位）；换绑覆盖旧边
   - `PUT /iot/dataitem/{itemId}/formula`（source_type=2 公式）：formula 必填非空；**校验链照 0013 Expander 同口径**（{TOKEN} 字符集/前缀白名单/括号配平/长度 ≤2000）+ **引用存在性**（SELF./PARENT./绝对键逐个核对本实例与父实例数据项——引用不存在 4001 拒存）+ **循环依赖检测**（重建 ref 边后 DFS 本项可达自身 → 4001「循环依赖」——这是 0057 占位项的接线点之一）；通过则落 formula + 事务内重建 ref（先删后插，ref_type=1）
   - `PUT /iot/dataitem/{itemId}/rollup`（source_type=3 汇总）：rollup_method SUM/AVG；汇总范围=子级数据项动态纳入（引用边在计算期展开——本期落「汇总标识+范围规则」存储，ref 不展开子级全量边，报告说明口径）
   - **阈值四字段**：`PUT /iot/dataitem/{itemId}/thresholds`（valid_min/max/alarm_low/high）——min<=max 配对校验「结果行四列」口径照 0014 修订轮；覆盖写实例行（0014 已交付 effective-params 读取——本接口是覆盖写入的另一入口，语义一致）
   - 语法校验的 Aviator 真语法仍归 FEAT-0012（占位口径照 0011/0036 任务正文）——本任务交付的是**引用结构与循环依赖**层
2. **校验通过落库并事务内重建 ref（先删后插）**——任务正文硬要求；formula 换绑/清除时旧边清理。
3. **继承链**：实例未配置（source_type/formula/阈值空）→ 运行时读模板默认（0014 effective-params 范式——本任务把 source_type/formula/rollup_method/calc_period 也纳入 effective 读取口径，阈值部分 0014 已交付，报告合并说明）。
4. **汇总动态纳入子级+缺失打码 7**（任务正文/ADR-0009）：汇总项计算语义归计算引擎——本任务落「汇总项标识+子级范围规则存储 + 缺失打码 7 的配置声明位」；计算行为归 FEAT-0012（登记口径）。
5. **DELTA 可引用、INTEGRAL 不开放**（任务正文/ADR-0010）：公式内函数白名单校验占位（函数名单常量——真实 Aviator 校验归 FEAT-0012）。
6. **影响预览 API（0057 接线点）**：`GET /iot/dataitem/{itemId}/impact`（iot:item:query）——引用图反查：**正向**=本项引用了谁（ref 边出向）+ **反向**=谁引用了本项（ref 边入向，即「受影响上级项」）+ 指标关联（本期 ref 图不含指标边——返回空集占位，指标引用边归 0043/计算引擎链路登记）；响应含 full_key/instance 编码/名称，数据权限裁剪（仅可见实例的项）。
7. **变更事件**：三类配置变更落库后发 `DataItemConfigChangedEvent`（itemId/tenantId/变更类型——照 0013 事件范式只发布不建监听；配置快照联动归 0024）。
8. 循环依赖检测范围：单实例内 SELF/绝对键闭环 + 跨实例 PARENT 边闭环（ancestors 链向上）——DFS 实现，环上项 id 列表进 4001 文案。

### 测试口径（TDD）

三类配置接口逐个：绑定成功/换绑覆盖/绑定点位不存在拒；公式展开校验（引用不存在 4001/循环依赖 4001 含环列表/畸形 token 4001）/落库+ref 重建正确（先删后插断言）；rollup 汇总标识存储；阈值四列配对结果行口径；继承链三态；impact 正反向+裁剪；事件发布断言。
冒烟（dev）：0013 生成链建实例数据项 → formula 配置（SELF/绝对键混合）→ impact 反查 → 制造循环（A 引 B 引 A）→ 4001 环列表 → 阈值覆盖 → effective-params 合并核对。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线以开工实测为准 + 新增，Results 段贴回）
2. dev 冒烟（每步原始输出贴回，序列见测试口径）
3. prod 档真实启动回归 → 0 ERROR + impact 200
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0036-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0036）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
