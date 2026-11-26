# 实施 brief：TASK-0022 + TASK-0023 微改攒批（点位导入导出 + 反查/批量启停/曲线占位，同一 subagent 统一测一次）

## 批次

- **微改攒批**：TASK-0022（3 人天）+ TASK-0023（2 人天）共 2 个任务给同一个 subagent，全部做完统一测一次（L1 §2 微改攒批）
- BASE：`a961fd1`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/**`（导入导出/反查/批量启停）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/channel/service/**`（ChannelReferenceGuard 点位计数转正——0018 遗留「点位→通道→网关」间接计数的裁决落点，见裁决 4）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `backend/ot-admin/src/main/resources/schema/point/`（导入模板 Schema 或校验常量，如需）
  - 不碰 frontend/、ingest/、模板/实例/字典域（只读）、dataperm

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks\TASK-0022.md`（导入导出）、`tasks\TASK-0023.md`（反查/批量启停/曲线占位）
- 关联对象：`features\FEAT-0007.md`、`requirements\REQ-0008.md`、`testcases\TC-0007.md`
- 范式权威：`iot/dataitem`（0013 导入三态回执模式——本批导入校验项不同但结构同源）、`iot/channel/schema/ChannelConnConfigValidator`（0018）、`iot/point/**`（0021 九接口与双道闸/裁剪/配额接线全部已就位）
- 决策：`decisions\ADR-0011.md`（TD-0045 单位字典）、`decisions\ADR-0009.md`（TD-0015/0016）

## 接口与决策（逐条结论，不再讨论）

### TASK-0022 导入导出（照 0018 字典导入范式，校验项按点位域语义）

1. `GET /iot/point/import/template`（iot:point:add 权限复用）+ `POST /iot/point/import`（iot:point:add）+ `POST /iot/point/import/error-report`（同 0018 无状态回传渲染）——模板列按点位域字段（pointCode/pointName/dataType 1-4/valueType DOUBLE|INT|BOOL|ENUM/unit/channelId 或 channelCode/boundInstanceId 可空/deptId 可空/minValue/maxValue/samplePeriod/deadband/remark）。
2. 逐行校验项（任务正文）：编码合规（[A-Za-z0-9_.] 与长度 96、租户内唯一活跃行预检 + 已删占用同口径拒绝）、unit 在 sys_dict iot_unit 字典、通道存在（channelCode→id 解析，不存在拒）、min<=max、dataType/valueType 枚举合法；**整批不半落库**；单批上限 **10000 行**（任务正文）。
3. 按编码幂等：重导同文件 → 已存在活跃行整批 DUPLICATE_CODE 类拒绝（照 0018 口径：批量拒绝 body code=400 + data 逐行回执，4002 留单点语义——0019/0021 已确立先例）。
4. 配额联动：**导入不触发配额校验豁免**——导入也受配额强控（校验通过行数 + 当前计数 > quota → 整批拒并提示配额上限；豁免会造成「界面 99/100 拒、导入塞 1000 行」绕过）；配额计入 import 成功落库行。
5. 导出：`GET /iot/point/export`（iot:point:list 权限复用）——按查询条件导出 xlsx（与模板列同构），数据权限裁剪照 list 同条件（裁剪后的可见集导出）。
6. 事件语义：导入落库的启用点位**不发 PointEnabledEvent**（批量导入视为配置迁移，建子表由重生成/启用动作触发——报告登记口径留痕，0029 消费方联调对齐）。

### TASK-0023 反查/批量启停/曲线占位

1. **反查接口**：`GET /iot/point/{id}/references`（iot:point:query）——被哪些数据项引用：查 iot_data_item.idx_tenant_point（point_id），返回数据项清单（full_key/instance 编码/数据项名——联表裁剪数据权限：仅可见实例的数据项）；返回空=无引用。
2. **详情反查聚合**：`GET /iot/point/{id}` 详情响应聚合 references 计数 + 近 24h 曲线占位字段。
3. **近 24h 曲线**：`GET /iot/point/{id}/curve`（iot:point:query）——**数据源 TDengine st_point_num 全限定名查询（本期 fail-soft 空集占位）**：实现走 0003 HttpTdengineAdmin 形态（base-url 配置，192.168.31.145 环境联调归 0068），查询失败/无数据返回空数组不报错；曲线端点契约就位、真实数据归 0029 写库后联调。
4. **批量启停**：`PUT /iot/point/status`（iot:point:edit）——批量 ids + status 0/1；**每条过数据权限防线**（resolve——不可见即整批拒，fail-closed 照 0021 修订轮范式）；**停用影响清单**：status=0 时响应携带受影响数据项计数（idx_tenant_point 引用计数）——前端二次确认语义由该计数承载；**发布联动占位**：批量启停属于「影响接入的变更」——变更生效走 FEAT-0008 发布链（0024 未交付），本期变更即时落库但**不产生接入生效**，接口响应明示「变更已保存，重新发布后对接入生效」（REQ-0010 保存≠生效同款口径）；批量启停启用侧发 PointEnabledEvent（逐条，0029 消费）。
5. ChannelReferenceGuard 点位计数转正（裁决 4）：删通道时其下有点位 → 拒（提示点位编码）——「点位→通道→网关」传递性补全；网关侧间接计数（点位→网关直接计数）仍占位登记（0024 裁决）。

### 统一验收（攒批测一次）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1822 + 新增，Results 段贴回）
2. dev 冒烟（两任务串一起，每步原始输出贴回）：admin 建网关+通道 → 导入 1000 行点位文件（含 5 类错误行配方）→ 整批拒+逐行回执 → 修正重导 → 200+配额计入 → 重导幂等拒 → 导出 xlsx 与导入内容一致 → 删通道拒（提示点位编码）→ 反查接口返回引用数据项 → 批量停用含引用点位 → 响应携带影响计数 + 「重新发布后生效」明示 → 受限用户批量启停不可见点位 → 整批 403 → 24h 曲线端点空集 200
3. prod 档真实启动回归 → 0 ERROR + point/list 200
4. 改动范围核对：`git diff --name-only a961fd1..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-0023-report.md`（两任务合并报告，分节对应）。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；**不得把两个 TASK 置「已完成」**；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许两个 git commit（main，分别注明 TASK-0022 / TASK-0023——两任务提交分离便于追溯）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
