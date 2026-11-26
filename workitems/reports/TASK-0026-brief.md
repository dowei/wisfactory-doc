# 实施 brief：TASK-0026（config.changed 通知、快照拉取接口与回滚——发布链收官）

## 批次

- TASK-0026（本批共 1 个，标准任务 3 人天；backend 串行道——发布链第三环，前置 TASK-0024/0025 已交付）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告；0025 收尾后应为 d08ec02 或其修订轮结果）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/**`（notify 真实现 + internal 拉取端点 + changes 摘要 + service 扩展）
  - `backend/ot-admin/pom.xml`（**仅新增 MQTT 客户端依赖**：org.eclipse.paho.client.mqttv3，版本显式钉死）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增 ot.publish.notify.* 与 ot.publish.changes-* 配置键）
  - **无新 SQL**（无新表；如实现中发现必须建表/加列，停下写报告等裁决，不得自行加）
  - 不碰 frontend/、ingest/、openapi 域、dataperm、credential/gateway/channel 域

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0026.md`
- 关联对象：同库 `features\FEAT-0008.md`、`requirements\REQ-0033.md`（验收：config.changed 丢失靠拉取对账哈希兜底/回滚=旧快照新版本重新下发）、`testcases\TC-0008.md`
- 决策：`decisions\ADR-0008.md`、`decisions\ADR-0006.md`（凭据不进快照与通知载荷）
- 上游交付：`reports\TASK-0024-brief.md` + `reports\TASK-0024-report.md`、`reports\TASK-0025-brief.md` + `reports\TASK-0025-report.md`（NotifySender 接缝/落行语义/内部端点范式/冒烟登录链范式全在前两份报告）
- 上游代码（开工先读）：iot/publish/notify/**（NotifySender 接口+NoOp——你要替换的真实现落点）、publish/support/PublishScopeGuard.java、publish/controller/IotPublishInternalController.java（@Anonymous+显式 checker 范式）、publish/storage/SnapshotStorage.java（取回路径）、gateway/auth/GatewayInternalAuthConfig.java（/internal/v1/** 零注册）
- 接口权威：详细设计 §3.2 p24（下行配置主题）+ §3.3 p25（快照拉取/增量查询/通知载荷）——brief 已提取贴下

## 接口与决策（逐条结论，不再讨论）

### 上游规范提取（p24/p25）

- 下行·配置主题：`iot/{tenant_code}/{gateway_code}/config` QoS 1——配置变更通知**只带版本号与哈希**，快照本体走 HTTP 拉取，避免大消息压垮 Broker
- 快照拉取：`GET /internal/v1/config/snapshot?scopeType=&scopeId=&version=`——接入服务、计算引擎、网关（经接入转发）；不带 version 返回最新版，带则指定版；响应含 version、snapshotHash 与快照体（Schema 版本号一并下发）
- 增量查询：`GET /internal/v1/config/changes?sinceVersion=`——返回之后变更摘要（变更对象与类型），供下游判断增量应用还是全量重拉；跨度过大（如中间版本快照已清理）时返回 needFullSync=true
- 通知载荷（§3.3 逐字）：`{"tenant": "group_a", "scopeType": 1, "scopeId": 88, "version": 17, "hash": "..."}`——通知只作触发信号，下游必须再拉快照比对哈希，不得直接信任通知内容

### 代码结构

1. **MqttNotifySender（NotifySender 真实现，替换 NoOp 注册）**：
   - 依赖：org.eclipse.paho.client.mqttv3（pom 显式版本）
   - topic 映射（按 publish_log 目标逐行发）：target_type=1 网关 → `iot/{tenantCode}/{gatewayCode}/config`（gatewayCode 读 iot_gateway）；target_type=2 接入 / 3 计算 → 共享内部主题 `iot/config.changed`（接入订阅后按 scopeType=1 过滤、计算按 scopeType=2 过滤——单一共享主题，不按目标拆分）
   - 载荷 §3.3 逐字五键（tenant=租户编码；hash=快照 SHA-256）；QoS 1、retain=false
   - **fail-soft 纪律**：broker 未配置/连接失败/发布异常 → WARN 日志、**发布流程不阻断不回滚**（REQ-0033 验收 2：通知丢失靠拉取对账兜底）；连接懒建立 + 自动重连；客户端句柄复用不逐发新建
   - 配置键：`ot.publish.notify.enabled`（**默认 false**——未启用时保持 NoOp 语义路径）、`broker-url/username/password/client-id/keep-alive-seconds/timeout-seconds`
2. **GET /internal/v1/config/snapshot**（iot/publish 域内新 internal controller 方法，照 0025 的 @Anonymous+显式 checker 范式）：scopeType/scopeId 必填、version 可选（缺省=该 scope 最新已发布版）→ 响应 `{version, snapshotHash, snapshotUri, schemaVersion, snapshot:<快照体对象>}`；快照体从 SnapshotStorage 取回 + 哈希核对（不一致 500 告警，复用 0024 路径）；scope 无版本/指定版本不存在 → 400；**凭据终检对出站快照体同样生效**（复用 0024 组装终检，防存储被篡改后凭据外泄）
3. **GET /internal/v1/config/changes**：sinceVersion 必填 → `{needFullSync: bool, latestVersion, changes:[{version, hash, publishTime, summary:{added:[], removed:[], changed:[]}}]}`；summary 由相邻版本快照体 diff 生成（按对象 code/id 对齐，分点位/通道/数据项/公式/指标五段计变更对象与类型）；**跨度超限或中间快照文件缺失 → needFullSync=true**（只给 latest 摘要，不给逐版 diff）；`ot.publish.changes-max-span` 默认 10；sinceVersion ≥ 最新 → changes 空数组
4. **回滚接线确认**：0025 已让回滚=新版本+落行+NotifySender 逐行触发——真 sender 上线后回滚即自动重新下发；**本任务不新增回滚代码路径**，冒烟做 E2E 证据（发布→通知尝试→拉取→ACK→变更→再发布→回滚→通知→拉取新指向）。**0025 评审 Minor ④ 交接约束（必须承接）**：现 NotifySender 在**事务提交前**被调用——真实现须二选一并 javadoc 登记取舍：a) 发送移至 afterCommit（推荐：杜绝「通知已发、行已回滚」的伪通知）；b) 维持提交前调用并 justify（fail-soft 下伪通知可容忍——下游拉取对账兜底）。不得无裁决沿用现状
5. **EMQX 真联调口径**：本地无 EMQX（GO-LANE-HANDOFF 实测）——本任务 mock 口径：NotifySender 依赖薄接口（如 MqttPublisherPort）隔离 paho，单测 mock port 验 topic 映射/载荷/fail-soft；真 broker 验证登记 TD-0056（若用户当期补 wisfactory-emqx 容器，升级为真联调，派发时协调者另行说明）
6. 无新菜单种子（/internal/v1 无管理面权限；无管理面新端点）；数据权限：PublishScopeGuard 复用于 changes 摘要的 scope 校验（如有管理面入口）/internal 端点由内部令牌收口
7. 0025 的 5002 登记义务不在本任务（归 TASK-0015 确认门）；不实现灰度/审批流（REQ-0033 非目标）

### 测试口径（TDD）

topic 映射（网关目标→专属 topic、接入/计算→共享 topic、tenantCode/gatewayCode 取数）；载荷五键逐字+哈希值正确；fail-soft（port 抛异常→WARN 不阻断发布事务）；enabled=false 走 NoOp；snapshot 端点（最新/指定/不存在 400/哈希核对/凭据终检出站生效）；changes 端点（diff 五段摘要正确/跨度超限 needFullSync/中间缺失 needFullSync/空数组）；回滚 E2E 链。红转绿留痕。

## 验收命令（整体验收）

环境事实同前（协调者 2026-11-22 实测）：WSL docker wisfactory-mysql（**MySQL 8.0.42**@3307 root/root 库 ot_wisfactory）/wisfactory-redis（7.2.4）；宿主机 phpStudy mysqld 空转进程无监听——忽略；sql-exec.sh **每条语句单独一次调用**；TDengine REST 已恢复但本任务不涉；8098 启动前探测、用毕即停；WSL2 空闲关停→长间隙前保活循环；backend 前置 mvn install -pl ot-core -DskipTests。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 无新 SQL——核对声明：`git diff --name-only <BASE>..HEAD` 无 resources/sql/** 文件（如出现即违约）；application.yml 仅新增 ot.publish.notify.* / ot.publish.changes-* 键（逐行核对贴回）
3. dev 冒烟（每步原始输出贴回）：dev 起服（notify.enabled=false 默认）→ 发布网关 scope → NoOp 路径零异常 → 置 enabled=true+假 broker 地址重启 → 再发布 → WARN fail-soft 证据（发布仍 200、版本落库）→ curl 内部令牌拉 snapshot（不带 version=最新/带 version=指定/坏 version=400）→ changes 查询（diff 摘要/超限 needFullSync）→ ACK 生效 → 变更点位再发布 → 回滚 → 拉取新版本快照与旧内容一致
4. prod 档真实启动回归 → 0 ERROR + snapshot 拉取 200（触及 application.yml，部署/配置类加严）
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0026-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0026）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
