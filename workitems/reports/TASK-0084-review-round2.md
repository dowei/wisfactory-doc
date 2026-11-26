# TASK-0084 复评报告（修订轮 · 终审 round2 · diff 评审 L1-execute §13）

- 评审对象：修订隔离 diff `2fc2a4d..3e09775`（5 文件全 `ingest/**`：main.go + itembridge/**），主体提交 2fc2a4d、修订提交 3e09775
- 评审角色：fresh、no-subagents、只评审不修改（唯一例外：本报告）
- 评审日期：2026-11-24
- 评审口径：`reports/TASK-0084-review-round2-brief.md`「复评重点 / 越界反查 / 报告 / 契约」逐项执行
- 终审性质：0084 已用尽「1 退回 + 1 复评」上限，本轮为终审；判据＝C-1 与 I-2 是否真闭合且无新 C/I → 通过，否则如实不通过
- 独立验证手段（只读）：`git log/diff/show` 核提交范围与归属；`go build/vet/test ./...`；`go test ./... -run Contract -v` 计数；`-tags live` 真 Redis 复跑 7 步；源码 / 权威契约 / 0029 报告逐文件比对；`grep` 反查失真论证残留

## 总判定

**通过**——首评 C-1（Critical）与 I-2（Important）均真实闭合，无新增 C/I。C-1 由「pointCode 字符串点号去末段」改为「配置快照反查 pointCode→Point.ID→DataItem.FullKey+instance」，下划线真样本契约命中 + live S3 真收双实证，主路径不再空转；I-2 由 `context.Background()+5s` 改为 `context.WithTimeout(runCtx,5s)`，取消沿派生链即时打断在途 Paho 发布，`blockingPub` 契约断言非恒真。I-1 评估不改码、M-1 deferred 均按 brief 如实登记。回归与范围全绿。

---

## 闭合判定（终审核心）

### C-1（Critical）——已闭合

逐条复评重点 1 核实（均基于 HEAD=3e09775 源码，非报告自述）：

- **反查链正确**：`ingest/internal/itembridge/mapper.go:39-68` `Map()` 经 `m.cache.LookupPoint(pointCode)`（:43）→ `Point.ID` → 遍历 `snap.DataItems` 找 `di.PointID != nil && *di.PointID == p.ID`（:51-58）→ `full = item.FullKey` 原样发布（:62）→ `idx := strings.LastIndex(full, ".")` 从**数据项全键 FullKey** 去末段取 instance（:63-67）。与退回 brief 裁决的权威路径逐字一致：`.` 此时是契约 `fullKey = instance_code + "." + data_code` 的分隔符，解析合法；pointCode 原文只作 `LookupPoint` 键，不再做任何字符串结构假设。
- **字符串解析已删除**：全 `ingest/**` `grep MapPointCode` = 0 命中（bridge.go 中 `MapPointCode` 函数体及 `fmt`/点号去末段逻辑整体移除），`onEvent` 改调 `b.mapper.Map(src.PointCode)`（bridge.go:172）。
- **fail-closed 四路径**：`m==nil || m.cache==nil`（:40）、`LookupPoint` 未命中或 p==nil（:44）、`Get()` 快照未装载（:47-50）、点位未绑 DataItem（:59-61）、FullKey 无实例段（`idx<=0 || idx==len-1`，:64-66）均返回 error → `onEvent` WARN + `dropped++` 丢弃（bridge.go:172-177），不造假主题。
- **snapCache 注入正确**：`main.go:340` `startItemBridge(pctx, cfg, logger, snapCache)` → `main.go:448` `NewBridge(src, pub, itembridge.NewSnapshotPointMapper(snapCache), logger)`；该 `snapCache`（`main.go:277` `snapshot.NewCache()`）与快照 reconciler 共享同一原子替换指针（`main.go:353 Cache: snapCache`），因此生产环境 mapper 命中真实装载的配置快照，而非空缓存。
- **三处失真论证未进代码/注释**：全 `ingest/internal/itembridge/**` 与 main.go `grep -E "A-Za-z0-9_|白名单|禁用|0034|0036|字符集"` = 0 命中。mapper.go / bridge.go 注释明确写「pointCode 是点位编码、fullKey 是数据项全键，两个编码空间」（mapper.go:4-6、bridge.go:7-8），采用退回 brief 的事实核查口径。
- **权威口径交叉核对**（独立读取原文，非采信评审或报告）：`unified-message-model.md:28` pointCode 示例 `F01_WS01_EQ001_M01`（下划线）；`config-snapshot-schema.md:67/95` pointCode 仅 `string(96)` 无字符集约束、`:165` fullKey=instance_code+"."+data_code；`push-payload-evaluation.md:33` pointCode 与 fullKey 是两个编码空间；`pipeline.go:680` newvalue 的 pointCode=`PointRow.PointCode` 原样；`TASK-0029-report.md:142-153` live 真样本全下划线 `V29_NUM_01`/`V29_STATE_01`。`snapshot.go:86-107` DataItem 确有 `FullKey`/`PointID *int64`，`:525-562` Cache 确有 `NewCache/Replace/Get/LookupPoint`——实现所依赖的 API 均真实存在。

**判定：C-1 闭合。**

### I-2（Important）——已闭合

- **超时 ctx 派生**：`bridge.go:186-193` 发布用 `pubCtx, cancel := context.WithTimeout(runCtx, 5*time.Second)`，`b.pub.Publish(pubCtx, topic, body)`，取代原 `context.Background()+5s`。`runCtx` 由 `Run` 经闭包 `func(channel, payload string){ b.onEvent(ctx, ...) }`（bridge.go:121）透传；仅当 runCtx 为 nil 才回退 Background（:188-190，防御性兜底，生产路径不触发）。
- **取消真实打断在途发布**：`paho_publisher.go:70-86` `Publish` 真实 `select` 于 `tok.Wait()` 完成与 `<-ctx.Done()` 之间，派生 pubCtx 取消即上返 `ctx.Err()`，非仅测试假象。全 `itembridge/**` 的 `context.Background()` 仅存于测试 scaffold 与 bridge.go:189 兜底，生产发布路径无 Background 悬挂。
- **契约断言非恒真**：`bridge_contract_test.go` `TestContractPublishUsesRunCtxCancelInterruptsInflight` 用 `blockingPub` 在 `Publish` 内真实阻塞于 `<-ctx.Done()`；先断言 `pubCtx != ctx`（证明是 `WithTimeout` 派生 ctx 而非 Run ctx 本体），`cancel()` 后 `<300ms` 内必须 `<-pubCtx.Done()` 否则 FAIL——非恒真。独立复跑 PASS。

**判定：I-2 闭合。**

### I-1（Important）/ M-1（Minor）处置合理性——核实无误

- **I-1**：报告 §修订轮五.2 如实登记「Redis Password 未接线——与发射侧（`event/redis.go`/`cache.RedisOptions`）对称匿名形态，宿主 Redis 6379 无密码、0029 实测匿名可用，按 brief 不改代码，需密码环境由后续 `REDIS_PASSWORD` 统一 env 扩展」，与首评 I-1「既有边界、非本任务硬要求」及返工 brief 第 3 条一致。评审独立核对宿主 `redis-cli PING=PONG`（匿名可用）。处置合理。
- **M-1**：报告 §修订轮五.4 如实登记「brief 可选、不强制——未做 WARN 采样，保持主体原子计数+WARN 形态，登记演进位，不阻断」。与返工 brief 第 5 条一致。处置合理。

---

## 测试样本真实性（复评重点 2）

- **契约 14 项全改下划线真样本且证明经快照反查命中**：`bridge_contract_test.go` 夹具 `testSnapshotCache()`（:23 附近）装载 `V29_NUM_01→F01.WS01.EQ001.NUM_01`（instance `F01.WS01.EQ001`）、`F01_WS01_EQ001_M01→F01.M01`（instance `F01`）、`V29_STATE_01→F01.WS01.EQ002.STATE_01`。`TestContractSnapshotMapperUnderscorePointCodeResolvesDataItemFullKey` 对三条 `m.Map(point)` 断言真反查结果；注释明确「原字符串解析实现会 no-dot 丢弃，快照反查命中」——能区分新旧实现。
- **点分号假阳性用例已删除/改语义**：原 `TestContractMapPointCodeFailureDrops` 改为 `TestContractSnapshotMapperUnboundOrMissingDrops`，点分号形态 `F01.WS01.EQ001.M01` 现作为「快照无绑定的原始 pointCode」应被丢弃；`TestContractSnapshotMapperNilCacheFailsClosed` 覆盖 nil 缓存 fail-closed。无残留纯字符串解析用例。
- **桥接 6 项改下划线样本**：map→发布、映射失败丢弃、租户隔离、发布失败 fail-soft、坏 JSON、ctx 取消全改 `V29_NUM_01`/`V29_STATE_01`/`F01_WS01_EQ001_M01`，主题实例段断言随之改为 `F01.WS01.EQ001` 等快照数据项实例。
- **live S3 真收下划线事件**：`bridge_live_test.go:398` PUBLISH `V29_NUM_01`、:408-410 逐字断言 topic `iot/open/{tenant}/F01.WS01.EQ001/item`、:423 断言 fullKey `F01.WS01.EQ001.NUM_01` + 4 字段、:426 禁 tenant 字段；S4（:432-436）未绑定 `V29_UNBOUND_99` 500ms 无消息；S7（:481）终态 published=3/dropped=1/publishFailed=0。**评审独立复跑 live 全序列**（真 Redis 6379 PONG + 真 Paho + 内嵌 MQTT 3.1.1 夹具 broker + 装载下划线绑定快照），S1–S7 `LIVE-ALL-DONE`、EXIT=0，S3 输出 `topic=.../F01.WS01.EQ001/item payload={"fullKey":"F01.WS01.EQ001.NUM_01",...}`，与报告逐字一致。

---

## 回归与范围（复评重点 5）

- **构建/测试**：`cd ingest && go build ./... && go vet ./... && go test ./... -count=1` 独立复跑 → BUILD_EXIT=0 / VET_EXIT=0 / TEST_EXIT=0，12 包全 `ok`、0 FAIL 0 ERROR。
- **契约计数**：`go test ./... -run Contract -v -count=1` 独立统计 PASS=225 / FAIL=0（=211 既有基线 + 14 itembridge 修订后契约，修订前 12 → 修订后 14，差额 +2 与新增 `...NilCacheFailsClosed`、`...PublishUsesRunCtx...` 两 deny 项吻合），零回归。
- **范围隔离**：`git log 2fc2a4d..3e09775` 含 3e09775（0084）与 640c9bd（并行 0063）。`git show --name-only 3e09775` = 5 文件（`main.go` + itembridge 的 `bridge.go`/`bridge_contract_test.go`/`bridge_live_test.go`/`mapper.go`），全 `ingest/**`，越界 0；640c9bd 的 4 文件（3 个 backend + `resources/sql/upgrade/20261124_iot_calc_schema.sql`）为 0063 修订，按路径隔离不在 0084 范围，backend/frontend/calc/contract 零触碰。与首评声明范围一致。

---

## 新发现清单

**无新增 C（Critical）/ I（Important）/ M（Minor）发现。**

补充说明（非发现、供记录）：`SnapshotPointMapper.Map` 对 `LookupPoint`（一次 `cur.Load`）与 `Get`（又一次 `cur.Load`）分两次原子读，极端并发下 reconciler 原子替换快照可能使 `p`（旧快照 Point）与 `snap`（新快照 DataItems）版本错开；由于 Point.ID 为稳定主键、`PointID==ID` 绑定即便变化也只会命中当前有效 DataItem 或未命中 fail-closed 丢弃，不产生伪造主题/错误数据，属配置快照缓存固有读法、非本轮修订引入，不构缺陷，仅作观察记录。

---

## 说明

- 评审为 fresh 视角，仅依据 brief 指到的隔离 diff / 首评 / 返工 brief / 实施报告 / 工作项 / 权威契约原文与工作区 HEAD=3e09775 源码判断；未采信首评三处失真论证，亦未仅凭修订报告自述，关键结论均回到源码与契约原文独立核实。
- 未向协调者追问、未派 subagent、未修改任何源码或 workitems 状态（唯一例外：本报告）。
- 终审结论：**通过**——可进入后续收尾；本评审不改 TASK 状态。

## 一句话结论

**通过（终审）**——C-1 真修复（快照反查 `pointCode→Point.ID→DataItem.FullKey+instance`，下划线真样本契约命中 + live S3 真收 `fullKey=F01.WS01.EQ001.NUM_01`，主路径不再空转，三处失真论证零残留）、I-2 真修复（`WithTimeout(runCtx,5s)` + Paho ctx select 即时打断，`blockingPub` 断言非恒真且实测 <300ms）、I-1/M-1 处置如实；`go build/vet/test` 12 包全 ok、契约 225 PASS/0 FAIL、live 7 步 `LIVE-ALL-DONE`、5 文件全 `ingest/**` 越界 0，无新增 C/I。
