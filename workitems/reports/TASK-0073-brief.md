# 实施 brief：TASK-0073（ingest 流水线内核先行，契约驱动 mock TDD）

## 批次

- TASK-0073（Go 道首项；执行环境完全独立于 backend 道——不碰 backend/frontend/resources/contract 除读）
- BASE：主仓当前 HEAD（开工时 `git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `ingest/**`（唯一代码区）
  - 报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0073-report.md`
  - 其余一律不碰（尤其 backend/、frontend/、resources/**、application*.yml、workitems 库其他文件）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0073.md`
- 关联对象：同目录 `features\FEAT-0009.md`、`requirements\REQ-0013.md`、`testcases\TC-0009.md`（本任务只做其内核子集；集成验证归 0027-0029 正式认领时）
- **契约（唯一权威，contracts-as-tests）**：
  - `D:\workspace\WisFactory\resources\contract\unified-message-model.md`（v1.1.0，§3.2 章为消费契约：MQTT 主题、上行消息格式、字段纪律、处理顺序、批量上限、HTTP 上报）
  - `D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md`（v1.0.1：信封 + 快照体字段级 Schema）
  - `D:\workspace\WisFactory\resources\contract\tdengine-super-tables.md`（v1.0.0：四张超级表与标签、建库参数）
  - `D:\Documents\dowei\2.项目资料\智慧工厂\workitems\conventions.md` **CONV-02**（质量码 0-10 唯一存储口径与文本码映射）
- 骨架现状：`ingest/`（go.mod module `net.otlab/ingest`、cmd/ingestd 占位入口、README 边界说明）

## 接口与决策（逐条结论，不再讨论）

1. **contracts-as-tests**：三份契约是解析/写库正确性的唯一基准。解析器单测的样例消息必须来自契约 §3.2 的 JSON 示例（逐字段断言，含 tenant/points[].ts/value/quality/batchId 等信封纪律）；写语句形态断言对照超级表契约（INSERT 语句/标签值/子表命名）。契约说「不定」的字段（如推送频率）不做假设。
2. **管道形态**：`Consumer(假 EMQX/HTTP) → Parser → Validator(质量判定挂点) → Batcher(攒批缓冲/水位/失败重排) → Writer(假 TDengine 执行器)` + `SnapshotFetcher(假端点) → 配置提供` + `PointCache(假 Redis 或 miniredis)`。组件间只经接口交互；0027-0029 接线时替换假实现为零改动组装。质量判定本任务只留挂点与 CONV-02 枚举/映射常量，完整判定逻辑归 0028。
3. **假实现边界（mock 口径，同 TASK-0003 先例）**：假 EMQX=内存 channel 注入消息；假 TDengine 执行器=记录「谁在何时以什么参数调了什么」（SQL 形态断言，不真执行）；假快照端点=本地 httptest 服务回放契约样例。真实联调验证全部挂 TD-0053（TASK-0068），本任务不宣称任何真实连通性。
4. **技术栈约束**：Go 标准库优先；外部依赖仅限测试用途（如 miniredis）——引入任何第三方库需在报告「偏离」中说明理由；module 路径维持 `net.otlab/ingest`。
5. **并发纪律**：消费→攒批路径必须明确并发模型（单消费循环 + 有界队列起步即可，文档写在组件 doc comment）；攒批缓冲必须有上限与丢弃策略（满时丢最旧并计数——对齐 ADR-0014 缓存 FIFO 丢最旧口径），不无界增长。
6. **git 纪律**：提交只 add `ingest/**` 路径，提交信息注明 TASK-0073；主仓同期有 backend 道提交，绝不 `git add -A`/`git add .`。

## 验收命令（整体验收）

1. `cd ingest && go build ./... && go vet ./...` → 双双退出码 0、无输出
2. `cd ingest && go test ./... -count=1` → 全绿（原始输出完整贴回，含 -v 的契约符合性断言清单）
3. `go test ./... -run 'Contract' -v` 单独复跑贴回（契约符合性测试类）
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落 `ingest/**`
5. 依赖清单核对：`cat go.mod`——require 段除测试依赖外为空（或逐项说明）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0073-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- 测试或验收失败先定位根因再修；禁止改测试凑输出；每个修复附「为什么现在对了」。
- 允许 git commit（main，只限 ingest/** 路径，注明 TASK-0073）。

## 收尾回写

- 执行会话自任协调者：按 L1-execute 完成验收（diff 评审）后跑 `close-task.sh TASK-0073 --repo D:\workspace\WisFactory`，回填 `计划子任务`/`修改内容`/`任务结果`/`实际耗时`（墙钟小时一位小数）。
