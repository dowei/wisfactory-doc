# 实施 brief：TASK-0059（设备详情页——状态区/实时/曲线/来源追溯）

## 批次

- TASK-0059（本批共 1 个，标准任务 5 人天；前端道——前置 0058 批一已交付（展示端工程+模型树骨架），状态/实时/曲线数据源现已全部就位：0050 状态读口✅/0051 对外接口✅/0023 点位反查✅）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `frontend/src/**`（展示端设备详情页 + 复用 0058 批一展示端视图区的组件/API 客户端）
  - 路由配置（/display/** 下新增设备详情路由）
  - 不碰 backend/calc/ingest/契约文件/workitems（报告除外）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0059.md`
- 关联：FEAT-0019/REQ-0029/TC-0019/ADR-0010（曲线降采样留痕）/ADR-0011（看板同源）
- 上游交付（开工先读）：`reports\TASK-0058-report-batch1.md`（展示端工程 src/views/display/** + display-api.ts + router/display.ts + 三页骨架）+ `reports\TASK-0050-report.md`（状态读口四端点：current/history，动态翻译+UNKNOWN 口径）+ `reports\TASK-0051-report.md`（对外六接口：items/realtime、points/history、metrics/query——**注意 openapi/v1 需 appKey+HMAC，浏览器不可直连；展示端应走管理面同源接口**）
- 范式基准：frontend/src/views/display/**（0058 批一）+ frontend/src/views/iot/**（0056/0070 范式）
- 数据源核对（开工 grep）：管理面是否已有 items/realtime 与 points/history 的等价端点（0023 点位反查/0051 管理侧封装）；若只有 openapi/v1 带 HMAC，则展示端曲线/实时走**后端代理或管理面已有接口**——不把 appKey/secret 暴露到前端（安全红线）

## 接口与决策（逐条结论，不再讨论）

1. **设备详情页路由**：/display/device?instanceId=（从 0058 车间页设备色块点击进入，下钻计数不变）；复用 0058 批一 OtLayout + 展示端组件体系
2. **状态区**：大色块（statusName/color 来自 0050 GET /api/v1/iot/status/current 的动态翻译，不硬编码）+ 持续时长（current 返回 changeTime 计算）+ **历史时间线**（0050 GET /api/v1/iot/status/history，逐条状态变化+时间）；LWT 离线态与 UNKNOWN 按 0050 口径展示（不现算）
3. **实时数据区**：当前值/单位/质量标记/超限标红/离线灰显——走管理面已有的点位实时值接口（grep 确认 0023/0051 管理侧形态）；**禁止前端直连 openapi/v1 HMAC 接口**（安全红线）；若管理面无等价端点→报告登记缺口等裁决，不自造代理
4. **历史曲线**：多项叠加 + 时间范围选择 + 超限断线时段标注；数据走管理面点位历史接口；**降采样标注**（ADR-0010）：曲线须明示"当前展示为降采样结果"+降采样方式（如 60s/300s 分档），不静默降采样
5. **来源追溯**：汇总类数据项展开子级贡献（instance/dataCode/value）+ 公式类展示输入项（formula refs + 输入值）——从 0036 impact/ref 反查或 0013 引用边读口取；留痕读计算留痕（iot_calc_log 归 0063 未交付——本批展示**已有数据源可得的部分**，缺数据源的入口不写死链、报告登记）
6. **组件列表**：按 data_type 分组的设备组件清单（点位/数据项/指标），点击可展开详情
7. **数据纪律**：不造假数据（与 0058 批一同款——无数据源的槽位显示「数据接入中」空态）；卡片状态数与 0050 判定一致（同源）；重算后自动刷新（轮询或手动刷新按钮，选简单方案 javadoc 说明）
8. **不碰**：0060 指标看板（另有任务）、0058 批二（卡片数据，另批）、backend/calc 任何代码

### 测试口径

设备路由可达+下钻计数；状态区三来源（current/history/UNKNOWN）与 0050 同源断言；实时值质量标记渲染（超限红/离线灰）；曲线降采样标注可见；来源追溯展开；空态无假数据；pnpm 三门 + live 走查（8098 探测，忙则跳过登记）。

## 验收命令

1. `pnpm build:test` / `pnpm lint` / `pnpm typecheck` → 全 0 error
2. 联调冒烟（视 8098 空闲）：登录→展示端车间页点设备→详情页状态/实时/曲线/追溯四区走查→全停
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 frontend/src/** 与路由配置

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0059-report.md`。五项齐全 + 数据源缺口登记（若有）+ 一句话结论。

## 契约

- no-subagents；**提交只 add frontend/**；提交注明 TASK-0059；不改 workitems 库（唯一例外：写报告）；不把 TASK 置「已完成」。
- 同框架同组件；不造假数据；**前端不持有 appKey/secret**。
