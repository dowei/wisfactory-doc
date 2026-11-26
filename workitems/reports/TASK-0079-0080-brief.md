# 实施 brief：微改批 TASK-0079 + TASK-0080（两任务一个执行者，统一测一次）

## 批次

- 微改批（N=2 → 同一 subagent，全部做完统一测一次；backend 道排队——前置 TASK-0015 收尾后由协调者派发）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/credential/**`（0079：CredentialPullVO + service 填充）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/**`（0080：internal 端点出站组装一处）
  - `backend/ot-admin/src/test/java/**`（对应测试）
  - 不碰其他域、SQL、yml、workitems（报告除外）

## 任务一：TASK-0079（凭据拉取响应扩 gatewayStatus）

- 背景：0076 修订轮 I3 登记缺口的后端承接——Go 侧 authmqtt 需要网关级禁用校验的数据源。TASK-0027 已按契约字段名 `gatewayStatus` 实现 Go 侧消费与 fake puller 红线测试（禁用 deny）。
- 修复口径：`CredentialPullVO` 扩 `gatewayStatus` 字段（值语义=iot_gateway.status 原值，禁用=0——以实体字段口径为准）；service 填充一处（pull 时按通道所属网关带出）；红线测试：禁用网关的 pull 响应携带 gatewayStatus=禁用值 + 既有凭据测试全绿。
- 最小触点纪律：不改既有凭据拉取语义；响应其余字段不动。

## 任务二：TASK-0080（BUG-0005 修复——内部快照拉取原始字节透传）

- 背景：GET /internal/v1/config/snapshot 出站快照体经 Jackson 再序列化（5.000000→5.0），下游从收到体重算 SHA-256 与 snapshotHash（=原始存储字节哈希）恒不一致——「拉快照比对哈希」兜底（REQ-0013/REQ-0033 验收 2）被削弱。证据：BUG-0005 档案 + TASK-0027-report §5.2（live 实证）。
- 修复口径：出站快照体改为**原始存储字节透传**（Jackson 3 RawValue/原样内嵌；响应其余字段——version/snapshotHash/snapshotUri/schemaVersion——不动；存储文件读取→原样嵌入响应，不经过 JsonNode 解析）。
- **BUG 修复类加严（§13）**：
  1. **全仓同契约点扫描**：grep 出站组装链路中其他「解析 JsonNode 后再序列化」的出站快照路径（如管理面 /iot/publish/snapshot、changes 端点摘要源）——扫描结果贴报告；管理面端点若同样丢失原始字节，列出待裁决（不自行扩散修复）
  2. **可重算断言先红后绿**：集成断言——对出站快照体（原样 JSON）重算 SHA-256 == snapshotHash，先对旧代码跑红、修复后绿；0026 既有 snapshot 端点测试回归
- **Jackson 3 纪律**（TD-0058）：RawValue 形态须用 Jackson 3 API（tools.jackson）

## 统一验收（两任务全做完测一次）

环境事实（协调者 2026-11-23 实测）：WSL docker wisfactory-mysql（8.0.42@3307）/wisfactory-redis 正常；8098 prod 启动回归用（若活体联调：backend dev 起→curl 拉快照→对出站体重算哈希核对→还原；启动前探测用毕即停）；TDengine 本批不涉（无需 TDengine env 注入）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 0080 可重算断言红→绿证据 + live curl 复核（对出站体重算 == snapshotHash）
3. 0079 红线测试证据（禁用网关响应携带 gatewayStatus）
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0079-0080-report.md`（一批一报告）。五项：改动文件清单 / 统一验收原始输出（含先红后绿与扫描证据）/ 未做项与原因 / 偏离计划之处 / 遗留问题。末尾一句话结论。

## 契约

- no-subagents；不改范围外文件；扫描发现其他命中→列出待裁决不自行修；不把 TASK 置「已完成」（协调者收尾）；不改 workitems 库其他文件。
- 允许 git commit（main，注明 TASK-0079/0080）。
