# 复评报告：TASK-0024 scoped 复评 round2（代码评审）

- 复评者：fresh 代码评审子代理（不共享协调者上下文，未派 subagent，未修改任何工作区文件）
- 复评对象：`TASK-0024-diff-round2.txt`（BASE=dca3dc9，HEAD=da830e4；4b7bbe3=I-1 测试 TDD 红 / da830e4=C-1 修复绿；4 文件 +354/-82）
- 复评口径：L1-execute §13 复评条款——只答三问（C-1 / I-1 / 修复 diff 新增 C/I），不重开原评审已通过的 8 项与 Minor 8 条（M-1~M-8 记 deferred）
- 依据物：diff 包、原评审报告 `TASK-0024-review.md`、修订轮报告 `TASK-0024-revision-report.md`、实施 brief `TASK-0024-brief.md`、工作区源码核对（backend/ot-admin publish 域）+ 独立复跑测试
- 日期：2026-11-22

## 结论

**通过**——三问全过，无新 Critical/Important。

## 越界反查（复核）

`git diff --name-only dca3dc9..HEAD` 实测 4 文件（publish service impl / SnapshotStorage / 两测试类），全部落在修订授权范围「publish 域 service/snapshot/storage 及其测试」内；SQL/yml/frontend/其他域零改动。✓ 与协调者比对一致。

另核提交拆分：`git show --stat 4b7bbe3` 实测仅 1 个测试文件（+166/-19），与「TDD 红留痕」自述一致（红先于修复提交）。✓

## 三问逐条

### 问 1：C-1 是否真修复 —— **是（结构性消除，非补偿式打补丁）**

- 结论：**过**。
- 出处：`IotPublishServiceImpl.publish()`（`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java:153-197`，stage→INSERT→promote 流程）、`SnapshotStorage.java:80-115`（stage/promote 本体）、全仓调用点 grep。
- 说明：
  1. **两个破坏向量同时消除**：向量一（败者覆盖正式名）——新流程在 INSERT 之前只写 `<tenant>/.staging/` 下的唯一暂存名（`Files.createTempFile`，随机名 + O_EXCL 原子创建，并发/跨进程均不相撞），正式名在 INSERT 前完全不被触碰；向量二（败者删除正式名）——`DuplicateKeyException` 分支唯一补偿动作是 `deleteQuietly(stagingUri)`（IotPublishServiceImpl.java:180，实参为暂存 uri）。全仓 grep 实证：main 代码 `deleteQuietly` 仅 2 个调用点、实参均为 `stagingUri`；`promote` 仅 1 个调用点（INSERT 胜者路径）；`readBytes` 2 个调用点均只读；回滚路径零文件写删。「正式名只被 INSERT 胜者 promote 写一次」为结构性事实而非约定。
  2. **promote 失败路径时序**：promote 在 `@Transactional(rollbackFor = Exception.class)`（IotPublishServiceImpl.java:150-151，已核实注解）方法体内、事务提交前执行；promote 抛 IOException → 删暂存 → 抛 BusinessException → 整事务回滚（版本行一并回滚）→ 不存在「已提交行指向缺失文件」。且 promote 先于 commit 的顺序在进程崩溃场景同样成立（崩溃于 promote 前后均不会产生「已提交行 + 未落位文件」组合）。
  3. **孤儿权衡已登记**：`SnapshotStorage` 类 javadoc（:41-47）登记两种孤儿——①「暂存后/落位前」崩溃遗留唯一名暂存孤儿（无行引用）；②落位后事务回滚（如回写失败）残留正式名孤儿（无行引用，下个同版本 promote REPLACE_EXISTING 自愈）；`IotPublishServiceImpl` 类 javadoc 与修订轮报告「遗留问题 2」同口径登记（清扫工具建议挂 M-5/MinIO 二期，本期不做）。登记完整，权衡成立。
  4. 附带核对：正式名命名空间（`<type>-<id>-<version>.json`）与暂存子目录（`.staging`）无交叠；跨租户目录隔离不变（brief D2 文件名契约不变，测试断言 `snapshot_uri=确定性正式名`）。非原子 move 降级路径（ATOMIC_MOVE 不支持时）沿旧实现模式，失败落点归入已登记孤儿类，无新窗口。

### 问 2：I-1 是否真修复 —— **是（真两线程 × 真文件存储，断言覆盖验收口径全项）**

- 结论：**过**。
- 出处：`IotPublishServiceImplTest.concurrentPublishTwoThreadsOneWinnerWinnerSnapshotIntactLoserNoTrace`（`backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImplTest.java:295-398`）与 `publishInWorker`（:410-424）。
- 说明：
  1. **真并发非顺序模拟**：`newFixedThreadPool(2)` 两真实工作线程 `CyclicBarrier(2)` 栅栏对齐起跑，均走真实组装器/凭据终检/NoOp 门/真实 `SnapshotStorage`（@TempDir）；`insertCalls==2` 断言强制「两线程都真实到达 INSERT 边界」——真并发到达文件存储交错路径。30 轮循环复跑。
  2. **断言覆盖验收口径全项**：每轮断言恰好一 200（winners==1）+ 一 5001（conflicts==1 且 code=VERSION_CONFLICT，TC-0008 步骤 5）；胜者行哈希=文件内容实际哈希（readBytes + sha256Hex，C-1 验收口径）；胜者 `snapshot_uri`=确定性正式名（brief D2）；`Files.walk` 全存储清点恰余胜者一个正式文件（败者零残留——旧实现下该断言会同时抓到「覆盖」与「删除」两个向量）；胜者 `retrieveSnapshot` 全链路（读防线→selectOne→readBytes→哈希核对）取回一致。两线程点位内容按调用序分流（pointCode 不同→哈希不同），使两个破坏向量均可被哈希/存在性断言捕获。
  3. **uk 仿真等价、不放水**：`synchronized(committedKeys){ if(!add(uk)) throw DuplicateKeyException }` 为原子「判重+占位」，对「同键并发 INSERT 恰一胜者」的已提交结局与物理唯一键语义一致；被仿真的仅是 DB 约束本身（单测无真实 DB 的必然选择），文件层交错全部真实。该边界在修订轮报告「遗留问题 3」如实登记，并与验收轮 dev 冒烟（200+5001）互为印证——非弱化。
  4. **红转绿有效**：4b7bbe3 仅测试（已核 stat），记录的红证据在第 1 轮即于「胜者文件必须存在」断言复现 C-1——证明本用例对 C-1 破坏向量真实敏感；da830e4 转绿。MockedStatic 线程域按工作线程自注册处理（主线程注册不跨线程），`ExecutionException`/`TimeoutException`/意外返回值均转轮次标注的失败，无挂起/吞异常通道。
  5. **独立复跑**：本复评在 工作区 HEAD=da830e4 实跑 `mvn test -pl ot-admin -Dtest='IotPublishServiceImplTest,SnapshotStorageTest'` → **Tests run: 25, Failures: 0, Errors: 0, BUILD SUCCESS**（日志中 hash mismatch/file missing ERROR 行为负路径用例预期告警）；并发用例单独再跑 1 次 → 绿。与修订报告自述一致。

### 问 3：修复 diff 是否引入新 Critical/Important —— **否**

- 结论：**过（无新 C/I）**。
- 出处：diff 全量（4 文件）逐段核对 + 工作区源码。
- 说明（候选点逐一处置）：
  - **事务边界**：promote 先于 commit（见问 1 第 2 点），关闭而非打开「已提交行指向缺失文件」窗口；败者 5001 路径同样经 rollbackFor 回滚，无半提交状态。
  - **资源清理**：stage 写失败即删暂存（SnapshotStorage.java:86-91）；promote 失败/撞键失败均删自己的暂存（IotPublishServiceImpl.java:180,188）；胜者 promote 成功即暂存移走。暂存孤儿仅在进程崩溃场景遗留，量级单次崩溃一个文件，已登记（非新发现）。
  - **并发正确性**：同一正式名的唯一写者=同版本唯一 INSERT 胜者（uk 保证）；不同版本正式名互异；回滚/取回路径零文件写删，与并发发布无文件级竞争。
  - **测试有效性**：round 间 stub 泄漏核查——`selectOne` 桩仅主线程 retrieveSnapshot 用（publish 流程只走 selectMaxVersion+insert），工作线程不受跨轮桩影响；pool 于 finally shutdownNow；`committedKeys`/计数器每轮重置。无放水通道。
  - **非原子 move 降级**：与旧实现同模式（原评审已过项的等价保留），失败落点归已登记孤儿类，无新窗口。
  - **无 fsync**：本地目录一期既有取舍（旧 store 同样无 fsync，取回侧哈希核对为最后防线），非本 diff 引入的新增风险类，不重开。

## 新发现清单

无 Critical / 无 Important。

- **Minor（不阻塞，仅登记）**：并发用例的「两线程内容分流」依赖 `pointReads` 调用序（隐含假设每次 publish 恰读一次点位——当前组装器成立且确定性成立）。若未来发布管线同线程二次读点位，分流内容可能同化、覆盖向量灵敏度退化（删除向量仍被「文件存在+恰余一文件」断言捕获）。现状正确，仅建议后续如改动组装管线读次序时同步复核该假设。出处：IotPublishServiceImplTest.java:323-326。
- （非新发现，已被执行者登记为遗留：暂存孤儿无自动清扫——修订报告「遗留问题 2」；M-5 根内校验硬化照旧 deferred——deleteQuietly javadoc 已指引「另行登记，不混入本修复」，方法体与旧实现逐字一致，未夹带。）

## 复评裁定

- 问 1（C-1 修复）：**过**
- 问 2（I-1 修复）：**过**
- 问 3（新增 C/I）：**过（无）**

**结论：通过。** 建议按 §12/§13 流程收尾：Minor 8 条（M-1~M-8）+ 本轮 1 条 Minor 登记记 TASK `任务结果` deferred，收尾进 LOG。
