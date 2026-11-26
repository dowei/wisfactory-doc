# 评审 brief：TASK-0084 diff 评审（代码评审）

> 实现提交 2fc2a4d（BASE=fb8f66d，8 文件 ingest/**）——新值事件→MQTT 数据项订阅主题桥接生产者（闭合 FEAT-0017 的 0053 I-2 缺口）。

## 角色

- 代码评审（口径：L1-execute §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0084-diff.txt`（fb8f66d..2fc2a4d，8 文件）
- 实施 brief + 报告：reports\TASK-0084-brief.md / TASK-0084-report.md（含 4 处 TDD 红转绿+live 7 步）
- TASK 详情 + FEAT-0017/REQ-0028/TC-0017
- 契约：`resources/contract/unified-message-model.md` v1.1.0「3.5」推送行——iot/open/{tenant}/{instance}/item，消息体 4 字段 fullKey/ts/value/quality（camelCase/毫秒，0053 定版）
- 上游：reports\TASK-0029-report.md（Redis iot:newvalue 发射侧）+ TASK-0053-report.md（消费侧）+ ingest/internal/event/**

## 越界反查

- 8 文件全落 ingest/**；区间内 0063 backend/SQL 改动已隔离（提交哈希归属）；backend/frontend/calc/契约零触碰。

## 评审重点

1. **4 字段映射精确**：newvalue payload {tenant,pointCode,ts,value,quality} → 发布 4 字段 {fullKey,ts,value,quality}——fullKey 原样 pointCode、ts/value/quality 原样透传、**payload 无 tenant**（tenant 只走主题段，契约逐字）；映射失败/坏 JSON/租户不一致→WARN 丢弃不造假主题
2. **pointCode→instance 映射**：去末段数据码取 instance（如 F01.WS01.EQ001.M01→F01.WS01.EQ001）——与 0034/0036 既定口径一致性核实；主题 `iot/open/{tenant}/{instance}/item` 逐字；映射口径与 ingest/internal/tdengine 既有 childTable/fullKey 映射是否冲突（两套映射不漂移）
3. **租户隔离**：主题含 tenant、payload 无 tenant；订阅 PSUBSCRIBE `iot:newvalue:*` 从主题段提取 tenant 与 payload tenant 一致性校验（不一致丢弃）；与 0053 消费侧按主题+appKey scope 过滤形成端到端双保险
4. **opt-in 与生命周期**：INGEST_ITEM_BRIDGE_ENABLED 默认 false；独立 goroutine+ctx 取消退出；装配/发布失败只 WARN 不阻断 ingest 主流水线（验令牌→查快照→死信→质量→写库）——核实桥接不侵入主链
5. **零新增依赖**：RESP2 PSUBSCRIBE 复用 0029 event 包客户端、Paho 复用 0027 客户端——pom/go.mod 零新增核实
6. **测试有效性**：223 契约测试 0F（211 基线+12）；live 7 步全绿（关开关无发布/4 字段真收/映射失败丢弃/租户隔离/无订阅者不报错/ctx 退出/全停）与断言对应；4 处 TDD 红转绿留痕真实、临时调试日志已清理
7. **零回归与范围**：0027/0029/0041/0042 既有契约测试逐类零回归；commit 仅 8 文件 ingest/**

## 报告

报告写到：D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0084-review.md。发现清单每条带严重级+出处+一句话说明；无 C/I 给「通过」结论。**特别对第 2 点（映射与 0034/0036 口径一致性）给出明确判定**。

## 契约

fresh、no-subagents、只评审不修改（报告除外）。
