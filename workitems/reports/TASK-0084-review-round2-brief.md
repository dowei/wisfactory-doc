# 评审 brief：TASK-0084 scoped 复评 round2（修订轮）

> 隔离修订 diff：2fc2a4d..3e09775，5 文件全 `ingest/**`（区间夹入的并行 0063 修订 640c9bd 已按路径隔离）。0084 首评 C-1（Critical）+ I-2（Important）退回一轮，本轮为唯一复评。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」；scoped 复评只覆盖修订范围 + 原发现闭合 + 回归确认）

## 必读（只给路径，不粘正文）

- **本轮隔离 diff**：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0084-diff-round2.txt`（2fc2a4d..3e09775，5 文件）
- 首评报告：`reports\TASK-0084-review.md`（C-1 / I-1 / I-2 / M-1 原文与失真论证）
- 退回 brief：`reports\TASK-0084-revision-brief.md`（协调者裁决 + 事实核查口径）
- 修订报告段：`reports\TASK-0084-report.md`（追加的「修订轮」段）
- 原实施 brief + diff：`reports\TASK-0084-brief.md` / `TASK-0084-diff.txt`
- TASK 详情：`tasks\TASK-0084.md`（任务结果段含裁决）
- 权威口径证据（退回 brief 已列路径，逐字为准）：`resources/contract/unified-message-model.md:28`、`config-snapshot-schema.md:67/95/165`、`push-payload-evaluation.md:33`、`ingest/internal/pipeline/pipeline.go:680`、`reports/TASK-0029-report.md:142`、`ingest/internal/snapshot/snapshot.go`（Point/DataItem/Cache.LookupPoint）

## 越界反查

- 修订区间 2fc2a4d..3e09775 实际 6 提交跨度，夹入并行 0063 修订（640c9bd，4 文件 backend/resources）；本 diff 已按路径隔离为 5 文件全 ingest/**（main.go + itembridge/**），与首评声明范围一致；backend/frontend/calc/contract 零触碰。

## 复评重点

1. **C-1（Critical）真修复**：新增 `mapper.go` 的 `SnapshotPointMapper` 是否经配置快照反查 `pointCode → LookupPoint → Point.ID → DataItem(PointID==ID) → FullKey + instance`（instance 从 **DataItem.FullKey** 去末段——此时 `.` 才是契约数据项全键分隔符）；`MapPointCode` 字符串解析是否已删除；未登记/未绑定/nil 缓存是否 fail-closed 丢弃；`main.go` 是否注入已装配 `snapCache`。核实重构**没有**把评审三处失真论证（pointCode 白名单 `[A-Za-z0-9_]`、点号禁用、0034/0036 依据）写进代码/注释。
2. **测试样本真实性**：契约测试（itembridge 14 项）是否改成下划线真样本（`V29_NUM_01`/`F01_WS01_EQ001_M01`/`V29_STATE_01`）且能证明经快照反查命中；点分号假阳性用例是否删除/改语义；live S3 是否真收下划线事件（报告称 `fullKey=F01.WS01.EQ001.NUM_01`、topic instance `F01.WS01.EQ001`），S4 未绑定丢弃、S7 ctx 即时退出是否对应断言。
3. **I-2（Important）闭合**：发布是否用 `context.WithTimeout(runCtx,5s)` 取代 `context.Background()`，取消沿派生链即时打断 Paho `tok.Wait()`；`blockingPub` 契约断言 cancel 后 <300ms `pubCtx.Done` 是否真实有效（非恒真）。
4. **I-1/M-1 处置合理性**：I-1（Redis Password）评估不改码是否与发射侧对称匿名 + 宿主无密码事实一致、报告登记清楚；M-1（WARN 采样）未做是否如实登记 deferred。
5. **回归与范围**：go build/vet/test 12 包全 ok、0F0E，itembridge 契约 14 项全 PASS、既有 211 基线零回归；5 文件全 ingest/**、零新增越界。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0084-review-round2.md`。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。对 C-1/I-2 是否闭合、是否有新 C/I 给出明确判定。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
