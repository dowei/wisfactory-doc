# TASK-0053 代码评审报告 · scoped 复评 round2（终审）

- 评审角色：代码评审子代理（fresh，口径 L1-execute §13「diff 评审」复评条款）
- 评审范围：仅修复 diff `a230f91..e9cc251`（限 `backend/`，7 文件）+ 原评审 F-01/I-1、F-02/I-2；**不重开**原评审已通过的 9 个评审重点
- 依据：brief 指定必读文件（`TASK-0053-diff-round2.txt` / `TASK-0053-review.md` / `TASK-0053-report.md` 修订轮段 / `TASK-0053-brief.md` / `resources/contract/unified-message-model.md` v1.1.0 §3.5 推送行）+ 工作区源码；fresh 独立判断，未向协调者追问，未派 subagent，只评审未改文件（本报告除外）
- 评审日期：2026-11-24

## 评审前独立核验（非仅采信报告）

| 核验项 | 方法 | 结果 |
| --- | --- | --- |
| 修订提交范围 | `git show --name-only e9cc251` + `git log a230f91..e9cc251 -- backend/` | 区间两提交：`ea34d02`（0047 修订，全 calc）+ `e9cc251`（本任务，7 文件全 `backend/ot-admin/**` openapi 模块+测试）；backend 触达仅 1 提交、7 文件，与 diff 包逐行一致。✓ |
| 4 项红线测试真实绿 | 实跑 `mvn test -pl ot-admin -Dmaven.compiler.proc=full -DargLine="-Djdk.attach.allowAttachSelf=true" -Dtest=SubscriptionReadIsolationContractTest` | **Tests run: 4, Failures: 0, Errors: 0 / BUILD SUCCESS**（源码逐行核对 4 个断言：租户 B 读不到 A 指标/数据项、无订阅空集、同租户 F01/F02 隔离）。✓ |
| 全量测试 | 同上全量跑（工作区含 0063 在途未跟踪 calcread 测试） | `Tests run: 2457, Failures: 0, Errors: 1` —— 2457 = 报告 2449 + 8 个未跟踪 calcread 用例；唯一 error 是 `LocalStorageServiceTest.storeWithoutExtensionKeepsFileNameWithoutDot` 的临时目录清理竞态（`NoSuchFileException: target/upload-test/files/2026`），单独重跑 6/6 绿，**与 openapi/修订 diff 无关**；openapi 全部类逐类绿（详见问 3）。✓ |
| 环境注意（非发现） | 本环境默认 `mvn test` 触发 `-proc:none`，Lombok 未处理致无关模块编译失败；Mockito inline mock 默认无法自附加致 mock 类失败 | 均以编译选项/argLine 绕过；失败面在 system/survey 等无关模块，**不是本修订引入**，按环境证据记录。 |
| 原评审通过项未被改写 | diff 逐 hunk 对照 | 修改面仅：控制器读口 2 处、Consumer 落缓冲 1 行、Store 增归属 Map + 过滤读口、Registry 增 2 个授权集方法、Service 接口/实现读口签名、新增 1 测试类；VO/codec/桥/配置/契约零触碰。✓ |

---

## 复评三问逐项结论

### 问 1：I-1（安全越权）是否真修复 —— 【明确判定：是，真修复；通过】

- **结论**：I-1 跨租户读取面已从根因消除——身份贯穿控制器→服务→注册表授权集→缓冲过滤，4 项红线测试独立复跑全绿；摄入侧双校验与「非 hash 幂等」语义零改动。**这是本轮安全关键的明确判定：通过。**
- **出处（工作区绝对路径 + 关键行）**：
  - `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\controller\SubscriptionController.java:81-93`（两个读口 `requireIdentity(request)` 结果不再丢弃，传入 `metricEvents(identity)` / `itemEvents(identity)`）
  - `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\subscribe\SubscriptionServiceImpl.java:63-73`（以 `identity.getAppKey()` 取注册表授权集）
  - `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\subscribe\SubscriptionRegistry.java:126-153`（`authorizedMetricTenants` / `authorizedItemProvenance`：从该 appKey **当前有效订阅**派生 tenantCode 集 / `(tenantCode,instanceCode)` 归属键集；appKey 为 null 返回空集）
  - `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\subscribe\SubscriptionEventStore.java:30-33,48-73,122-156`（`itemProvenance` 独立 Map；`metricEventsFor` 按 `e.getTenant()` 过滤、`itemEventsFor` 按归属键过滤；授权集空/租户不匹配→空 `ArrayDeque`）
  - `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\subscribe\SubscriptionConsumer.java:38-56`（摄入侧主题 tenant + 订阅实例 + fullKey 命名空间三重判定原样保留，仅末尾 `acceptItem(event, tenantCode, instanceCode)` 多传归属）
  - `D:\workspace\WisFactory\backend\ot-admin\src\test\java\net\otlab\admin\module\openapi\subscribe\SubscriptionReadIsolationContractTest.java:118-171`（4 项红线断言）
- **一句话说明**：旧实现「验签后丢弃身份 + 直接返回全局 `metricRing()/itemRing()`」的根因被身份感知读口替换：指标按 `tenantCode`、数据项按 `(tenantCode,instanceCode)` 授权集过滤，无订阅/租户不匹配 fail-closed 返空；4 项红线的断言（B 读不到 A、无订阅读不到、同租户跨实例隔离）对旧实现必红（报告修订轮二 RED 为编译红+行为红双重锁定），本人独立复跑 4/4 绿；`acceptItem` 仅多写一个不参与幂等键的 `itemProvenance`，摄入侧双校验与 `fullKey|ts|quality` / 全键+窗口（非 hash）幂等键逐字未改，评审重点 3 结论不动。

### 问 2：I-2 处置确认（桥接归 TASK-0084）—— 通过

- **结论**：登记清晰、边界无隐藏自造桥接，符合 brief「不自造新事件、不碰 ingest、判断不了标无法判定」口径。
- **出处**：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-report.md:370-375`（修订轮四完整登记：主题差异/桥接缺口事实、归属裁决、live S3 为协议层模拟而非端到端、0084 后补验条件）；`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0084.md:1-25`（已建档：状态「已确认」、名称明示承接「TASK-0053 评审 I-2」、范围 ingest Go 侧、含桥接实现+真协议冒烟+验收标准）；`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-review-round2-brief.md:12`（brief 明示 I-2 已拆 TASK-0084）。
- **一句话说明**：报告把「0029 `iot:newvalue:{tenant}` → `iot/open/{t}/{i}/item` 缺桥接生产者」定性为归属边界而非本任务代码缺口，逐项写明归 0084、live S3 性质、端到端补验条件；全仓 backend 检索无任何 `iot:newvalue` 订阅/桥接发布者（只有订阅主题构造与 Paho 通配符订阅常量），**无隐藏自造桥接**。补验条件依赖 0084 尚未交付（实际端到端验证结果：无法判定，按 brief 属 0084 后续，不影响本轮登记判定）。

### 问 3：无新 C/I（2449 零回归、归属记录无语义分叉、重点 3/7 不变）—— 通过

- **结论**：三问全项通过；无新增 Critical/Important。
- **出处**：
  - 全量/逐类：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0053-report.md:333-358`（2449 = 2445+4 原始输出）；本人独立复跑的 openapi 逐类绿——`SubscriptionReadIsolationContractTest` 4/4、`SubscriptionIsolationContractTest` 9/9、`EventRingBufferContractTest` 5/5、`ItemChangeEventContractTest` 2/2、`MetricProducedEventContractTest` 4/4、`OpenApiYmlStructureTest` 2/2、`HmacAuthStrategyTest` 22/22、`OpenApiQueryServiceImplTest` 31/31、`OpenApiDictSyncMapperSqlTest` 3/3、`OpenApiAuthFilterTest` 7/7。
  - 归属记录无语义分叉：`SubscriptionEventStore.java:48-73`（`acceptItem` 原无参重载保留，幂等键 `itemKey` 逐字未变）、`SubscriptionConsumer.java:42-56`（`hasItemSubscription`/fullKey 命名空间判定先于记录且未改）、`SubscriptionRegistry.java:65-124`（订阅注册/命中判定零改）。
  - 重点 3/7 未变：`SubscriptionEventStore.java:40-46`（`acceptMetric` 仍走 `event.idempotencyKey()`，非 hash）；diff 未触碰 `PahoSubscriptionBridge` / `KafkaSubscriptionBridge` / `application.yml` / 契约文件/`SubscriptionObserveController`（仅注入消费入口，无裸读口）。
- **一句话说明**：2449 = 2445 基线 + 4 新增，本人在含 0063 在途文件的工作区复跑得 2457（+8 未跟踪 calcread）且唯一 error 为无关 `LocalStorageServiceTest` 清理竞态（单跑全绿），openapi 全类逐类零回归；`itemProvenance` 只在 `acceptItem` 落缓冲时写、不进入订阅注册/摄入校验/幂等键，`metricEventsFor` 仍按指标全键+窗口去重后的缓冲读；重点 3（幂等非 hash）与重点 7（环境替代证据充分、非真机）结论不因本修订改变。

---

## 新发现（按 C/I/Minor 分级）

| 级别 | 出处 | 一句话说明 |
| --- | --- | --- |
| **Minor** | `D:\workspace\WisFactory\backend\ot-admin\src\main\java\net\otlab\admin\module\openapi\subscribe\SubscriptionEventStore.java:33,57-65,75-82` | `itemProvenance` 无界增长：`upsert(items, ...)` 按 `capacity`（默认 256）在 `items` 超限时淘汰最旧键，但 `itemProvenance.put(key, ...)` 对每条不同 fullKey|ts|quality 事件都写入、从无对应 remove；长跑高频数据项流下该 Map 会随时间单调增长（只存小字符串、不影响读隔离正确性），建议在 `upsert` 淘汰分支同步移除归属键，或让 `items`/`itemProvenance` 共用淘汰逻辑。不阻断本轮结论。 |

除上述 Minor 外，未发现新增 Critical / Important。

---

## 评审结论

**通过（scoped 复评 round2，终审）** —— 三问全过且无新 C/I：

1. **重点 1（安全越权修复）明确判定：真修复、通过**——事件读口按验签身份有效订阅派生授权集过滤环形缓冲（指标按 tenantCode、数据项按 (tenantCode,instanceCode)），无订阅/租户不匹配 fail-closed 空集；4 项越权红线测试对旧实现必红、本人独立复跑 4/4 绿；摄入侧双校验与非 hash 幂等语义零改动。
2. I-2 按协调者裁决登记为 TASK-0084 边界，无隐藏桥接，登记清晰。
3. 全量 2449（复核环境 +8 在途用例=2457）openapi 逐类零回归，归属记录未引入订阅/消费语义分叉，原重点 3/7 结论不变。

唯一新增为 1 条 Minor（`itemProvenance` 无界增长，建议随淘汰清理），不构成退回理由。按「修订上限 1 退回+1 复评，本轮为终审」口径，本任务 scoped 复评以通过结束；TASK 状态仍按约定保持「进行中」由协调者收尾。
