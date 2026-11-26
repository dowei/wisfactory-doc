# TASK-0076 diff 评审（fresh reviewer）

- 评审对象：单提交 `f0f4533`（BASE `d74d56f`，14 文件约 2570 行，全落 `ingest/**`——越界反查：0 越界，与 brief 声明范围一致）
- 评审依据：L1-execute §13 + 实施 brief 三件契约 + 统一消息模型 v1.1.0 §3.2（续）+ 0019 后端锚点（IotCredentialInternalController / CredentialPullVO / StaticTokenInternalAuthChecker，均已对照源码）
- 评审口径：接入服务安全面，从严；**只评审不修改**
- 独立验证（评审者本机重跑）：`go build ./... && go vet ./...` 双零退出；`go test ./... -count=1` 8 包全绿——报告 §2 验收输出属实

## 发现汇总

**Critical 1 / Important 3 / Minor 8**

---

## Critical

### C1 · dedup check-then-act 竞态：Submit 并发面下 QoS1 幂等契约可被打破（实证复现）

- **出处**：`ingest/internal/pipeline/pipeline.go`（本 diff 修改）——`processOne` 内 `SeenBatch`→`MarkBatch`（L329-335）与 `SeenPoint`→`MarkPoint`（L360-365）是**两次独立持锁的复合 check-then-act**；本 diff 新增的 `Submit`（L289-298）以「可与 Run（及多个 Submit）并发调用——流水线组件均并发安全」为文档承诺放开并发调用，但该复合操作不原子。
- **说明**：两个并发 Submit 携同 batchId（QoS1 重报恰在首份处理中到达的正常场景）可双双通过 Seen 检查各自全批处理；(pointCode, ts) 跨批重报同理可双写。**实证**（评审者以 `go test -overlay` 注入探针测试，仓库零改动）：300/800 轮三组重复实验中，同 batchId 并发被当新批处理的轮次 8-11/800（nonDup=2），唯一 (pointCode, ts) 键写出重复行 5-7 行（807 行 / 800 键）。直接违反 brief 决策 1「QoS1 幂等口径=(pointCode, ts) 批量 batchId 整批幂等」与 REQ-0013 验收 3；TDengine 同 ts 覆盖只吸收 DB 层表象，响应计数与去重语义在接入层已错，重报携修正值时落库值非确定。既有并发测试（`TestContractSubmitConcurrentWithRun`）只用互异键，结构性测不到此竞态；`-race` 未跑（无 gcc）且本竞态属逻辑 TOCTOU，race 检测器本也不报。`Submit` 的「并发安全」注释为不实声明。
- **修法方向**：去重挂点增加原子 `MarkIfSeen`（标记并返回是否首见）或 per-key 串行化；同时修正 Submit 并发注释。归本 TASK 修订轮（0027 接真流量前必须闭合）。

## Important

### I1 · 令牌缓存全局锁横跨网络回源：单次慢拉取冻结全部鉴权（吞吐/可用性）

- **出处**：`ingest/internal/gwtoken/gwtoken.go` `credential()`（L159-172）——`v.mu.Lock()` 持锁贯穿 `Puller.Pull`（HTTP 超时默认 10s）。
- **说明**：锁是全局单把，所有网关、两端点（report 401 前置校验 + auth/mqtt 回调）的 Validate 全部串行：一次后端慢拉取/不可达令**所有其它键的缓存命中也无法进行**；后端故障时吞吐坍缩为 ~1 次/10s 且请求堆积——fail-closed 变成放大的自伤 DoS。注释「串行化同一键的并发回源」实际做成了全局串行。修法：per-key singleflight / 双检锁。

### I2 · 401/503 响应回显内部错误细节（含潜在密文材料）给未认证调用方

- **出处**：`ingest/internal/httpapi/report.go` L1206-1208（`"网关令牌校验失败（fail-closed）: " + err.Error()`）、L1214（503 同样拼 sink err）；源头 `gwtoken/puller.go` 错误串含后端 status/业务 message，JSON 解析失败路径还含**响应体前 256 字节原文**（`truncateForLog`——响应体即 `Result<CredentialPullVO>`，含 ciphertext；mock 口径下 ciphertext 就是期望网关令牌）。
- **说明**：未认证客户端可获知后端内部拓扑/响应内容；与后端侧自身的日志卫生纪律（CredentialPullVO `@ToString.Exclude`、密文不进审计）不一致。auth/mqtt 的 deny 路径未回显错误（做得对），仅 report 面泄露。修法：对外固定文案，细节进服务端日志。

### I3 · 认证回调缺「网关 status」校验，代码注释却声称已校验（契约 §3.2 续偏差 + 未登记的缺口）

- **出处**：`ingest/internal/httpapi/authmqtt.go` L643-646 注释「校验网关编码与设备令牌、租户与网关 status（令牌链，fail-closed）」vs 实现（仅 gateway_code+token；status 无处校验）；契约 §3.2（续）「校验 gateway_code 与设备令牌、租户与网关 status」。
- **说明**：通道/凭据禁用经后端 403 覆盖，但**网关级禁用**（凭据仍有效）会被 allow；`Validator` 接口缝合处（GatewayIdentity 无 status 字段）也没有为 0020/0027 留出承载位；实施报告 §3 未做项未登记此偏差。至少须：注释改为实际行为 + 报告登记遗留；0020/0027 定版时 seam 增列。

## Minor（记 TASK 任务结果 deferred，收尾进 LOG）

1. **recover 双写头部 + panic 值外泄**（`httpapi/report.go` L1172-1177）：defer 内 writeJSON 在响应已部分写出时二次 WriteHeader（superfluous header）；500 响应体拼入 `%v` panic 值（panic 内部信息直达客户端）。
2. **RejectedWhole→503 语义错位**（`httpapi/pipeline_sink.go` L1046-1049）：理论上不可达（两端同一 Parser），但一旦触发即把客户端信封错误错分类为服务端不可用。
3. **注释与行为不符**（`gwtoken/puller.go` L383 注释「空 = 不带头」vs L434 `Header.Set` 空值头照发；fail-closed 结果不变）。
4. **鉴权前的全量解析 + 双重解析**（`httpapi/report.go` L1179-1196 + pipeline 二次 Parse）：无认证请求可消耗 ≤2MiB 解析 CPU（gateway 取自信封所致的顺序设计）；同一请求体入口与流水线各解析一次——可先轻量取 gateway 验令牌再单次解析。
5. **双 flusher 仅注释防守**（`pipeline/pipeline.go` L300-304）：Run 与 StartFlusher 互斥无 once/flag 保障，0027 接线失误即双 flusher（requeue 交错、缓存乱序更新）。
6. **主题范围无通配符防线**（`httpapi/authmqtt.go` L728-730）：tenant/gateway 编码直接内插 MQTT 主题串，`+`/`#`/`/` 未校验——静态映射期安全，0020/0027 接真注册表后成 ACL 范围伪造位，应留校验缝合。
7. **accepted≠落库的静默缺口**（0073 缓冲丢最旧语义经 HTTP 暴露）：行进有界缓冲后 BufferCap 溢出丢最旧，200「accepted」已发出、QoS1 HTTP 客户端不会重投——受理承诺与最终落库之间无对账信号。
8. **REST 客户端分类与部署硬化注记**（`tdengine/rest.go`）：code=0 但 HTTP 非 200 时 `SQLError{Code: httpStatus}` 把 HTTP 状态码混入 TDengine 错误码命名空间；默认 `root/taosdata` + 明文 HTTP 传输为部署面硬化事项（0068 前登记）。

## 契约符合性核对（依据 ①逐项）

| 项 | 结论 |
| --- | --- |
| 上报端点五字段组 | ✅ 信封 tenant/gateway/batchId/sentAt/points 齐全；tenant/batchId/points 纪律入口 400，gateway 经令牌链 fail-closed 兜底 |
| 批量上限 500 | ✅ 入口整批 400 契约文案，不进流水线（测试锁定）；2MiB body 防线在位 |
| 死信路由（0073 复用边界） | ✅ 复用 `DeadLetterSink`/冻结五步，未复制第二份流水线逻辑；unknown_code 响应明细+死信出口双验证 |
| 认证回调 allow/deny+主题范围 | ✅ 形态照 brief 决策 2（自定契约授权）；deny 路径全覆盖（内部令牌缺失/错 401+deny 双信号、空三要素、令牌错、后端不可达）；⚠ 网关 status 缺失见 I3 |
| 令牌链 fail-closed + 缓存 TTL | ✅ 五闸全拒、拉取失败不缓存、TTL 配置化默认 30s、常量时间比对（subtle.ConstantTimeCompare 两处）；⚠ 并发面见 C1/I1 |
| 写客户端（9750 教训固化） | ✅ 固定 POST /rest/sql 无库后缀、EnsureSchema 走 USE 多语句、Basic 头、默认 30s 配置化、code 24/918→EnvError WARN fail-soft（重试计数经 0073 攒批重排，9750 不误判）；⚠ 见 M8 |
| 0019 后端锚点 | ✅ GET /internal/v1/credential/pull?channelId=&credType=、X-Internal-Token（与 StaticTokenInternalAuthChecker.TOKEN_HEADER 同名）、Result<CredentialPullVO>{channelId,credType,ciphertext,keyVersion} 逐字段对齐 |
| 0073 内核零重写声明 | ✅ 属实：diff 中内核仅 pipeline.go 增量（Submit/StartFlusher/SubmitResult/返回值），message/quality/cache/snapshot/tdengine(0073 部分) 零触碰；0073 既有 55 项契约测试零改动全绿（评审者独立重跑确认）；⚠ 但 Submit 并发安全声明不实（C1） |

## 复现证据（C1，仓库零改动）

探针以 `go test -overlay` 虚拟注入 `internal/pipeline`（复用 0073 测试 harness，FakeExecutor 计行），三组重复实验：

| 轮数/goroutine | 同 batchId 被当新批轮次 | 唯一键重复写出行 |
| --- | --- | --- |
| 300 × 8 | 0（未现） | 1（301 行/300 键） |
| 800 × 24 | 11 | 7（807/800） |
| 800 × 24 | 8 | 5（805/800） |

探针文件仅存于临时目录（`/tmp/gw76probe/`），`git status` 确认 ingest/ 工作区零改动。

## 结论

验收命令实测属实（评审者独立重跑通过）；契约面五字段组/批量上限/死信路由/认证回调形状/令牌链 fail-closed/TD-0053 教训固化全部落实，0073 复用声明诚实。**C1 为并发面实证的幂等契约破坏，进 §12 退回路径**（修订轮上限内）；I1-I3 安全/可用面同轮处理；Minor 8 条记 deferred。建议修订轮修复后按 §13 复评（scoped：修复 diff + 原发现），并在有 gcc 的环境补 `go test -race ./...`（C1 修复的回归面）。
