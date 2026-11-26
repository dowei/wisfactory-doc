# 会话交接文件（2026-11-22 主会话 → 后续会话）

> 本文件是主调度会话的中断交接。按顺序读完即可继续流水线，无需历史上下文。

## 0. 你的角色与当前状态

你是主调度会话的接续者。当前**唯一在途任务 = TASK-0024（配置版本快照生成）**：实现已提交（ab9481a，28 文件），验收未跑完（前一执行者完成实现+通过 S1-S6 后被额度中断，无报告）。TASK-0024 状态=进行中，LOG-0037 未建。

## 1. 环境事实（全部实测）

- 工作区 D:\workspace\WisFactory；工作项库真实路径 /d/Documents/dowei/2.项目资料/智慧工厂/workitems（脚本先 cd 进去或设 WORKITEMS_ROOT）
- backend：Maven 多模块（ot-core/ot-admin），Java 21，mvn install -pl ot-core -DskipTests 前置惯例
- MySQL/Redis：WSL docker 容器 wisfactory-mysql/wisfactory-redis（unless-stopped 自启），127.0.0.1:3307 root/root 库 ot_wisfactory / 6379；重启电脑后验证 `wsl docker ps` + sql-exec.sh 探针
- TDengine 192.168.31.145:6041 root/taosdata：**fd 问题未修复**（taosd ulimit nofile 不足——DDL 挂起/code 24/918/iot_default 卡 creating），等用户修复后重跑 TD-0053 核验（不要在修复前碰 TDengine 写路径验收）
- 前端 pnpm（build:test/lint/typecheck 三门）；Go 道（ingest/，独立）
- SQL 无 mysql CLI：bash backend/scripts/sql-exec.sh（多条语句每条一行行尾分号）
- 端口纪律：8098 归 backend，启动前探测、用毕即停；派发后验证 agent 目录出现（agent_<id> 目录在 agents/sess 目录下）

## 2. 协议与纪律（AGENTS.md L0 + .agents/protocol/L1-execute.md）

- 流程：认领（set-status --expect）→ brief（reports/TASK-xxxx-brief.md）→ 后台派 subagent → 验证启动 → 验收（报告五项+diff 评审从严）→ 修订/复评（上限 1+1）→ close-task（--repo D:\workspace\WisFactory）→ LOG 骨架特补/指针化 → sync-index
- 派发后验证存活；协调者不自己修；评审 Critical/Important 退回一轮；diff 包按任务单提交隔离（并行道 BASE 漂移用 git show <commit> 隔离）
- 跨任务 brief 引用义务要显式携带（0018 遗漏 guard 接线的教训）

## 3. 立即要做的事

**TASK-0024 恢复验收**（新 subagent，报告 reports/TASK-0024-report.md 注明中断恢复接手）：
- 实现 ab9481a 已提交（28 文件：快照组装器严格按 resources/contract/config-snapshot-schema.md v1.0.1/SHA-256/本地目录存储 backend/data/snapshots（已入 gitignore）/单调版本 uk_scope_version/回滚复用快照/PublishConfirmGate NoOp/凭据排除终检/数据权限防线）
- 验收 5 条见 brief：reports/TASK-0024-brief.md（S1-S6 中断前已过，必须由你完整重跑采集证据）
- 若盘点发现实现缺陷：停下写报告等裁决，不自行修
- 验收过 → close-task（LOG-0037 骨架生成后填写）→ FEAT-0008 剩 0025/0026

**然后按序推进**（brief 现写）：0025（下发记录 ACK 回执与超时重发，3 人天）→ 0026（config.changed 通知、快照拉取接口与回滚，3 人天——发布链收官）→ 0027 正式认领解锁（Go 道真集成：EMQX 订阅/快照拉取/真库写入——TASK-0076 mock 前端半已就绪）→ 0048/0051 已完成、0052-0054/0055-0057/0021-0023 已完成

## 4. 台账快照（截至交接）

- 已完成 38/76（0001-0005,0007-0023 中除 0024/0025/0026,0043,0048,0051,0055-0057,0072-0076）；FEAT-0002/0003/0004/0006/0007 已完成
- 关键裁决留痕：ADR-0015（快照 Schema 派生）、ADR-0016+增补 8/9（数据权限口径）、TD-0055（缓存竞态残余风险）、TD-0054（Go 道已占）
- 契约冻结 v1.1.0/v1.0.1/v1.0.0（CONV-03 已满足）
- 上游修订三项待登记：data_item_template 补 tenant_id/idx_ancestors 前缀/default_calc_period（LOG-0015）
- 并行纪律：backend 串行、frontend/pnpm 道可并行、Go 道 ingest/** 独立、只读评审随时插空；并发额度超限会静默失败——派发后验证存活
- 待用户决策：TD-0001 选厂；TDengine fd 修复
