# 实施 brief：TASK-0076（ingest HTTP 上报端点 + 网关令牌链 + TDengine 写客户端，0027 前端半 mock 先行）

## 批次

- TASK-0076（本批共 1 个，标准任务 3 人天；**Go 独立道**——与 backend 0051（mvn）并行零冲突，本会话内调度）
- BASE：主仓当前 HEAD（开工时 git rev-parse --short HEAD 记录进报告）
- 改动范围（越界即违约）：
  - `ingest/**`（唯一代码区）
  - 报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-report.md`
  - 其余一律不碰

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0076.md`
- 关联对象：`features\FEAT-0009.md`、`requirements\REQ-0013.md`（令牌/幂等/死信）、`requirements\REQ-0011.md`（凭据/令牌口径）、`testcases\TC-0009.md`
- 契约权威：`resources\contract\unified-message-model.md` v1.1.0（§3.2 上报格式/批量上限 500/处理顺序）+ `resources\contract\tdengine-super-tables.md` v1.0.0（附录 B DDL）
- 0073 既有内核（复用不重写）：`ingest/internal/`（message/pipeline/quality/cache/snapshot/tdengine 六包）+ `ingest/README.md` 边界说明
- 后端对接锚点：0019 `/internal/v1/credential/pull`（backend/ot-admin/.../iot/credential/controller/IotCredentialInternalController.java——响应形状以代码为准）；TD-0053 实测教训（2026-11-22 会话）：**REST /rest/sql/<db> 库上下文在 3.4.3.9 返回 9750——必须全限定表名（db.stable）或 SQL 内 USE 多语句；DDL 响应延迟可达秒级以上（读超时调大）；fd 不足时 code 24 Too many open files / 918 Database in creating status 识别为环境级失败 fail-soft 重试**
- EMQX 真订阅/快照拉取/真库写入/端到端归 0027 正式认领（0026 收口后）+ 0068 联调——本任务 mock 先行

## 接口与决策（逐条结论，不再讨论）

1. **HTTP 上报端点** `POST /ingest/v1/report`：契约 §3.2（gateway/points[].ts/value/quality 信封）；验网关令牌（裁决 2）→ 统一流水线入口（0073 pipeline）→ 未知编码死信（0073 已有）；单消息 ≤500、批量上限 500（超限 400 契约文案）；QoS1 幂等口径=(pointCode, ts) 批量 batchId 整批幂等（0073 cache 包复用）。
2. **EMQX 认证回调** `POST /ingest/v1/auth/mqtt`：入参 username/password（EMQX HTTP 认证回调形态——username=gatewayCode，password=网关令牌）；allow/deny + 主题范围（允许该网关的 data/status 主题通配）——响应形状按 EMQX HTTP 认证文档惯例（{result: "allow"|"deny"}），测试以自定契约锁形状（真 EMQX 联调归 0068）。
3. **网关令牌链**：凭据拉取客户端（调后端 `/internal/v1/credential/pull?channelId=&credType=`，InternalAuthChecker 静态令牌 header X-Internal-Token——配置化）+ 本地缓存（TTL 配置化）+ 校验器（上报/认证回调携带的令牌 vs 拉取的凭据比对——比对形态 mock 口径：凭据明文比对占位，**真网关令牌体系归 0020/0027 正式认领时定版**，接口化留位）；**拉取失败 fail-closed**（deny/拒上报——与后端不可达时不放行）。
4. **TDengine 写客户端真实形态**：REST 客户端（Basic 认证）——**全限定表名**（iot_default.t_xxx USING iot_default.st_xxx——9750 教训）；读超时配置化（默认 30s——DDL 秒级延迟实测）；fail-soft（code 24/918 环境级失败识别 + WARN + 重试计数，不炸管道）；子表命名/超级表契约照超级表契约附录 B；**真库写入归 0068（fd 待修复），本任务以 httptest 假服务断言请求形态（全限定名/认证头/超时参数）**。
5. 契约纪律照 0073（contracts-as-tests）；组件间接口交互；外部依赖仅测试用途（httptest 标准库即可），新增第三方需报告声明。

### 测试口径（TDD）

上报端点：契约样例批量上报→流水线调用断言（mock pipeline）/未知编码死信/批量上限 500/超限 400/令牌失败拒；认证回调：allow/deny/主题范围/令牌错 deny；令牌链：拉取成功缓存/拉取失败 fail-closed/缓存 TTL；写客户端：全限定名断言/Basic 认证头/超时参数/code 24 与 918 环境级识别/fail-soft 重试计数。

## 验收命令（整体验收）

1. `cd ingest && go build ./... && go vet ./...` → 双零退出
2. `cd ingest && go test ./... -count=1` → 全绿（原始输出贴回）
3. `cd ingest && go test ./... -run 'Contract' -v` → 契约符合性清单
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落 `ingest/**`

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；禁止改测试凑输出。允许 git commit（main，仅 ingest/** 路径，注明 TASK-0076）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
