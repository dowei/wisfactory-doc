# Go 道交接包（独立会话启动文件）

> 本文件是「Go 采集接入道」独立会话的唯一入口。按顺序读完即可独立开工，不需要主会话的任何上下文。

## 0. 你的角色

你是 Go 道的**协调者兼执行会话**：按 workitem 协议领任务、写 brief、派 subagent 实现、验收、收尾——与主会话（backend/frontend 道）完全并行、互不等待。

## 1. 环境事实（全部实测过）

- 工作区：`D:\workspace\WisFactory`（Monorepo：backend=Java 管理系统、frontend=Vue3、ingest=**你的 Go 工程**、resources/contract=契约）
- 工作项库：`docs/workitems` 是目录联接，真实路径 `/d/Documents/dowei/2.项目资料/智慧工厂/workitems`；**所有脚本先 `cd` 进真实路径再执行**，或 `export WORKITEMS_ROOT=/d/Documents/dowei/2.项目资料/智慧工厂/workitems`
- 脚本在主仓：`D:\workspace\WisFactory\.agents\scripts\`（sync-index / next-task / set-status / close-task / reserve-id / finalize-object）
- 工具链：go 1.27.2（`go build ./... && go vet ./...` 基线绿）
- Redis localhost:6379：**backend 用 db0，你的测试用 db1+** 隔离；MySQL 127.0.0.1:3307（root/root，库 ot_wisfactory）Go 道只读不写
- **TDengine / EMQX 本地都没有**——mock 口径（contracts-as-tests），真实联调验证登记 TD-0053 挂 TASK-0068
- 端口纪律：8098/8099 归 backend/frontend 道，Go 道不得占用；启动任何实例前先探测、用完必停

## 2. 协议与纪律（与本仓 AGENTS.md L0 一致，重点重申）

- 先读：`.agents/protocol/L1-execute.md`（编排层）+ `.agents/skills/workitem-execute/SKILL.md`；写正文类用 `workitem-authoring` 规范（`reserve-id` → 文件工具写正文 → `finalize-object`，**禁止 sed 拼中文**）
- 铁律：任务未确认不写代码；无实测输出不许写「已完成」；验收不过退回执行者一轮、协调者不自己修
- Git 纪律：**提交只 add `ingest/**` 路径**（主仓同期有 backend 道提交，绝不 `git add -A`）；提交信息注明任务编号
- 验收闭环：实施报告（五项）→ diff 评审（fresh 子代理）→ 修订/复评（如需，上限 1+1）→ `close-task.sh TASK-xxxx --repo D:\workspace\WisFactory` → 回填 `实际耗时`（AI 墙钟小时，一位小数）
- 契约纪律：`resources/contract/` 三份冻结契约（消息模型 v1.1.0 / 快照 Schema v1.0.1 / 超级表 v1.0.0）是唯一权威；发现契约问题**报告并停**，不自行改契约（CONV-03 升版本归协调者裁决）
- 边界：不认领 backend/frontend 任务（0056/0024-0026 等归主会话道）；不碰 workitems 库其他会话在途对象

## 3. 你的任务

**TASK-0073「ingest 流水线内核先行」**（已确认，P0，预估 4 人天）——brief 在 `workitems/reports/TASK-0073-brief.md`，唯一输入，照它执行。
流程：`next-task.sh` 认领（应返回 TASK-0073）→ `set-status.sh TASK-0073 进行中 --expect 已确认` → 按 brief 实施（TDD）→ 验收 → diff 评审 → `close-task.sh`。

TASK-0073 之后：TASK-0027（正式认领）的前置 TASK-0026 由主会话推进中——**等主会话收口 0026 后** 0027/0028/0029 依次解锁，届时回到本文件第 0 节继续；在途期间若无可领任务，向用户报告待命，不要抢 backend 任务。

## 4. 交叉引用

- 关联 REQ/FEAT/TC：`features/FEAT-0009.md`、`requirements/REQ-0013.md`、`testcases/TC-0009.md`
- 边缘网关（TASK-0030/0031）与试点形态 TD-0050/TD-0001 联动——本批不做，形态裁决前不动
- 契约「不定」处（推送频率、window 类型等 5 项）登记于 `resources/contract/push-payload-evaluation.md` 遗留——做 0053 相关任务时闭合，平时不猜

## 5. 汇报

- 交付物：`workitems/reports/TASK-0073-report.md`（+ diff/评审文件按 TASK-xxxx-* 命名）
- 完成后向用户一句话汇报：报告路径 + 验收是否全过；与主会话无通信依赖
