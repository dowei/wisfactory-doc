# 实施 brief：TASK-0057（公式编辑器组件——本期核心件 + 后端依赖项挂接线义务）

## 批次

- TASK-0057（本批共 1 个，标准任务 6 人天中的本期核心部分；**frontend 独立道**）
- BASE：`4d7246d` 之后以派发时主仓 HEAD 为准（开工时 git rev-parse --short HEAD 记录进报告——backend 道可能并行推进）
- 改动范围（越界即违约）：
  - `frontend/src/components/iot/formula/**`（公式编辑器组件新包）
  - `frontend/src/views/iot/template/components/template-item-modal.vue`（接入编辑器——替换纯 textarea）
  - `frontend/src/api/iot/**`（如需引用数据源 API 封装扩展）
  - 不碰 backend/、ingest/、其他域页面

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0057.md`
- 关联对象：同目录 `features\FEAT-0018.md`、`requirements\REQ-0016.md`（公式语法/相对引用/循环依赖）、`testcases\TC-0012.md`
- **语法权威（客户端校验规则来源）**：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/RelativeRefExpander.java`（0013 交付——{SELF.X}/{PARENT.X}/{绝对键} 语法与字符集规则，前端校验必须与其同口径）+ 0011 落地的占位校验（括号配平/长度 ≤2000）
- 数据源：instance/template 数据项清单 API（0012/0013 已交付——引用选择器选码数据源）；`views/iot/template/components/template-item-modal.vue`（现 textarea 形态）
- 决策：`decisions\ADR-0011.md`（TD-0012 属性参数收窄）；REQ-0016（循环公式 4001）

## 接口与决策（逐条结论，不再讨论）

1. **本期交付（编辑器核心三件）**：
   - **编辑器组件**：多行公式输入（monospace）、语法高亮可后置（一期 textarea+覆盖层提示即可，不做 CodeMirror 重依赖——报告偏离节声明）、实时校验提示（客户端规则：花括号配对/嵌套合法、{TOKEN} 字符集 `[A-Za-z0-9_.]`、相对引用前缀白名单 SELF./PARENT.、长度 ≤2000、括号配平——全部与 0013 Expander 同口径，规则表抽成共享常量文件）
   - **引用选择器**：按唯一键选数据项**禁手输**——数据源=当前实例/模板数据项清单（既有 API），插入时按上下文自动包装 {SELF.code} 或 {PARENT.code}（父级选择）；选择器分组按 instance/数据项两级
   - **校验错误文案与后端 4001 语义一致**：前端文案常量与后端错误分段对齐（4001 公式语法错误族），报告列映射表
2. **后端依赖项（登记接线义务，占位实现）**：
   - 影响预览（受影响上级项与指标）：需引用图反查 API（0013 建了 iot_data_item_ref 表但无查询端点）——**接线义务归 TASK-0036**（绑定/公式/汇总配置接口与引用边维护）；前端占位按钮「影响预览」点击提示「待 TASK-0036 交付」
   - 试算入口（单点+窗口曲线）：需 Aviator 试算端点（FEAT-0012）——占位同上，登记归 FEAT-0012/0045
   - 循环依赖即时反馈：后端生成链 fail-closed（0013 终检）+ 4001（FEAT-0012）——前端仅做语法级提示，语义级循环检测登记后端义务
3. **接入**：template-item-modal 的默认公式字段替换为编辑器组件（props：数据源清单/初始公式/禁手输模式）；导出组件供 0036/0016 后续复用。
4. **验收口径**：`pnpm build:test`/`pnpm lint`/`pnpm typecheck` 三门 + 组件级测试（vitest 已在 devDependencies——校验规则表单测：合法/畸形 token/嵌套/长度/括号配平/相对引用前缀，逐条断言）+ 编辑器接入后 template-item-modal 构建通过。**不起 8098 联调**（同 0056 裁决）。

## 验收命令（整体验收）

1. `cd frontend && pnpm build:test` → EXIT=0
2. `cd frontend && pnpm lint` → 0 error
3. `cd frontend && pnpm typecheck` → 通过
4. `cd frontend && pnpm test -- --run` （vitest run，仅跑校验规则与编辑器组件测试）→ 全绿（若工程无 vitest 配置则报告说明并以 build 门替代）
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0057-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因（影响预览/试算/循环检测三项接线义务明细）/ 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件；不碰 backend/ 与 ingest/。
- 测试失败先定位根因再修。允许 git commit（main，注明 TASK-0057）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
