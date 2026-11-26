# TASK-0087 diff 评审退回修订 brief（一轮退回：I-1 allTenant scope 判定）

> 主体提交 578753f（10 文件），验收全过但评审发现 1 Important。本轮只修 I-1。评审 agent 因 API 频率限制中断未写完整报告，核心发现由协调者据源码独立核实确认（证据见下）。续用原执行者上下文（agent_7ec733be）。

## 必读（只给路径，不粘正文）

- TASK 详情：`tasks\TASK-0087.md`（任务结果段含协调者裁决）
- 实施 brief + 报告：`reports\TASK-0087-brief.md` / `TASK-0087-report.md`
- diff：`reports\TASK-0087-diff.txt`
- **缺陷证据（协调者独立核实）**：
  - `backend/.../openapi/service/impl/OpenApiQueryServiceImpl.java` `:182`（itemsRealtime）、`:285`（metricsQuery）、`:395`（requireInstanceVisible）——三处 `!scope.instanceIds().contains(...)` 缺 allTenant 检查
  - 同文件 `:118` 正确范式：`!scope.allTenant() && scope.instanceIds().isEmpty()`
  - `backend/.../iot/dataperm/VisibleScope.java:36-38`——`allTenant()` 构造的 instanceIds 恒 `Set.of()`（空集）
  - `backend/.../iot/displayread/impl/IotDisplayReadServiceImpl.java:66/125`——管理面用 `sessionScope()` = `dataPermService.resolve(userId)`，超管/运营方可返回 allTenant=true

## I-1（Important，必须修）：共享核心 scope 判定未处理 allTenant

**问题**：共享核心 `OpenApiQueryServiceImpl` 三处越权判定用 `!scope.instanceIds().contains(...)` 但没先看 `allTenant`。HMAC 路径的 scope 恒 `resolveClientScope`（allTenant 永远 false）所以一直正常；但本任务把**管理面 `resolve(userId)` 会话 scope** 接入了同一核心，超管/运营方的 `resolve()` 返回 allTenant=true、instanceIds 空集 → `:182/:285/:395` 会把所有 item/metric/instance 都判为 scope 外 → 这些用户经管理面四读口取不到任何数据（fail-closed 不泄露，但功能失效，与数据权限语义矛盾）。评审中断前的线索"null-scope 差异 / allTenant 断言缺失"指向此。

**修复要求（协调者裁决）**：
1. `OpenApiQueryServiceImpl` 的 `:182`、`:285`、`:395` 三处判定改为 allTenant 感知——`allTenant=true` 时视为可见（跳过 instanceIds 集合检查），逻辑与同文件 `:118` 的正确范式一致。具体可用 `if (!scope.allTenant() && !scope.instanceIds().contains(x))` 形态（`:182/:285`）与 `if (!scope.allTenant() && (instanceId == null || !scope.instanceIds().contains(instanceId)))`（`:395` 保留 instanceId==null 校验）。
2. **不要改** `resolveScope`/HMAC 身份翻译路径，HMAC 行为零回归。
3. 补一条管理面 allTenant=true 契约测试：mock `DataPermService.resolve(userId)` 返回 `VisibleScope.allTenant(false)`，经 displayread 端点取数 → 能取到数据（200 + 非空的 realtime/metric values）；在修复前代码上该测试必红（空集/403），修后绿。TDD 红转绿留痕。
4. 保留原有 scope 外/denied/跨租户隔离测试零回归；null-scope（resolve 返回 null → denied）路径保持。

**附带核查（评审线索，顺手确认非缺陷）**：`distinct` 参数在共享核心的用法、HMAC 六接口既有测试——确认本次改动未引入新行为差异即可；无问题在报告说明。

## 验收命令（沿用原始）

1. `cd backend && mvn -pl ot-admin test` → 0 失败 0 错误（含新增 allTenant 管理面读口测试 + 既有 2576 基线零回归），贴 Results 段
2. 范围核对：`git diff --name-only 578753f..HEAD` 只落 openapi service impl + displayread 测试（最小修复），frontend/calc/ingest/contract/HMAC 鉴权零触碰

## 交付

- 修订提交（注明 TASK-0087 修订轮 I-1）；报告追加「修订轮」段：I-1 三处修复 + allTenant 测试红转绿 + 为什么现在对了 + HMAC 零回归证据。
- no-subagents；只改最小范围；不置 TASK 已完成；报告写到 `reports\TASK-0087-report.md`（追加段）。
