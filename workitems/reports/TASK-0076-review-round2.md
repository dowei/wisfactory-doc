# TASK-0076 修订轮 scoped 复评（fresh reviewer）

- 评审对象：修订轮提交 `4b2d59d`（仅 ingest/** 7 文件；`git diff --stat f0f4533..HEAD` 实测 446+/53-，与 brief 声明范围一致）
- 评审口径：只判 C1/I1/I2/I3 四项是否修复、修复本身有无新缺陷；不重开全场；原 8 条 Minor 已 defer 出范围（核对未被修复恶化——recover 双写/panic 回显、双 flusher、RejectedWhole→503 等均未被本轮触碰）
- 独立验证（评审者本机重跑）：`go build ./... && go vet ./...` 双零退出；`go test ./... -count=1` 8 包全绿；`go test ./internal/pipeline/ -run 'Race' -count=10` 全绿；`go test -race` 仍不可用（cgo 缺失，与报告 §6.4 登记一致）

## 结论：C1 / I1 / I2 / I3 四项全部已修复

| 项 | 判定 | 依据 |
| --- | --- | --- |
| C1 | 已消除 | `BatchDeduper`/`PointDeduper` 收敛为原子 `MarkIfSeen`（首见判定+标记+FIFO 淘汰单锁内一次完成），Memory 两实现同步收敛；全仓 grep 旧 `Seen*/Mark*` 零残留、实现者仅 Memory 一对（无漏改）；`processOne` 两调用点均改原子口径；`Submit` 注释改为真实承诺（原子性前提逐项列明，TASK-0028「不得降级为先查后写两段式」已写入接口契约）；竞态回归转正 2 项（30 轮×12 goroutine 同 batchId 恰一 non-dup / 同 (pointCode, ts) 跨批恰一写出），本机 `-count=10` 复跑零复现——满足 brief 判定标准 |
| I1 | 已落地 | `credential()` 改 per-key singleflight（`inflight` 表 + Pull 移出元数据锁）：同键并发收敛一次 Pull、异键并行互不阻塞、拉取失败不缓存口径不变；正确性成立——等待者经 `close(call.done)` 的 happens-before 读到结果、无嵌套锁、等待不持锁（无死锁面）；发起者 ctx 取消错误复用为已知取舍已注明（singleflight 惯例）；`PerKeyParallelism`（A 悬挂期间 B 命中）与 `SameKeySingleflight`（8 并发恰 1 次 Pull）两测试锁定 |
| I2 | 已落地 | 401/503 改固定文案常量（`msgTokenRejected`/`msgSinkUnavailable`），err 细节仅进可注入 Logger（nil → log.Default()）；复核 report.go 全部错误路径：其余 400 路径只回显客户端自提供输入的纪律文案，不涉后端内部；测试断言响应体零密文材料（SECRET-MATERIAL/internal-detail-X）且日志承载细节——「细节进服务端日志」为裁决明确口径，非泄露面 |
| I3 | 已落地 | `authmqtt.go` 注释改为实际行为（仅 gateway_code+设备令牌，租户来自令牌链；通道/凭据禁用经后端 403 覆盖、网关级禁用会被 allow 明示）；`GatewayIdentity` 扩展位注释到位；报告 §6.4 登记 0027（扩 credential/pull 响应字段）→ 0020/0027（承载+实校验）义务链——与裁决口径（status 实校验不强制本期）一致 |

## 新发现：3 条（全部 Minor，不动摇本轮闭合）

1. **Minor · gwtoken `credential()` inflight 通道 panic 中毒**（`ingest/internal/gwtoken/gwtoken.go`）：`Puller.Pull` 若 panic，`inflight[key]` 残留且 `call.done` 永不 close——该键后续所有 Validate 永久阻塞在 `<-call.done`（goroutine 无界堆积直至重启）。singleflight 标配的 defer 清理缺失；触发面为 Pull 内 panic（net/http 常规路径返回错误不 panic，属异常路径），爆炸半径单键。建议：发起者 defer 内完成 close+delete（对齐 x/sync/singleflight 的 panic 防护）。
2. **Minor · 竞态回归测试 2 末尾断言读陈旧快照**（`ingest/internal/pipeline/pipeline_submit_race_contract_test.go` `TestContractSubmitRaceSamePointTsExactlyOnce`）：`st := h.pipe.Stats()` 在 `eventually`（等 `WrittenPoints` 收敛）**之前**取样，末尾 `if st.WrittenPoints != rounds` 却以该陈旧值判「写出后」——`WrittenPoints` 由异步 flusher 递增（BatchSize=1，flushCh 信号+10ms ticker 双触发），快照时可能 <30 而数毫秒后才收敛，存在 sporadic 假失败（flaky）面。应改为收敛后重读 `h.pipe.Stats().WrittenPoints`。测试质量缺陷，非产品缺陷。
3. **Minor · I1 测试假拉取器数据竞争**（`ingest/internal/gwtoken/gwtoken_contract_test.go` `gatedPuller`）：`calls` 为无同步 int，A goroutine 写（`p.calls++`）与主 goroutine 轮询读（`puller.calls >= 2`）并发——真实 data race，报告中已登记的 `-race` 补跑面（§6.4）将被该测试污染（每次必报）。建议 mutex/atomic。（`TestContractTokenChainSameKeySingleflight` 所用 `countingPuller` 因 singleflight 收敛实际单次调用，当前不受影响。）

## 判定理由（一句）

四项修复全部落地且经本机独立复跑验证（build/vet 双零、全量 -count=1 绿、竞态回归 -count=10 零复现），修复实现无并发正确性缺陷（原子性、可见性、死锁面均成立），3 条新发现皆 Minor 级（panic 中毒面 / 测试 flake / 测试内数据竞争），defer 的 8 条 Minor 未被恶化——本轮 scoped 复评通过。
