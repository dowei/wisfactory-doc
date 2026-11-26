# TASK-0024 修订轮报告：配置版本快照生成（修 C-1 + I-1）

> **修订轮执行者**（§12/§13 退回一轮：diff 评审不通过，只修 C-1 + I-1 两个发现，Minor 8 条一律不动）。
> 对应评审发现：**C-1（Critical）并发发布竞态摧毁胜者快照文件** + **I-1（Important）并发测试弱化为顺序模拟**，出处 `TASK-0024-review.md`。
- 修订 BASE：**dca3dc9**（开工时 HEAD，git rev-parse 实测 `dca3dc9673ef9a1db02d06d9e7f11fab638c6640`，工作区干净）。
- 工作区 D:\workspace\WisFactory（main）；日期 2026-11-22。
- 修订提交：**4b7bbe3**（I-1 测试改真两线程——TDD 红留痕）→ **da830e4**（C-1 修复——红转绿）。
- 越界核对：`git diff --name-only dca3dc9..HEAD`（连同 4b7bbe3/da830e4 两提交）= **4 文件**，全部落在授权范围「publish 域（service/snapshot/storage 及其测试）」内；SQL/yml/其他域/workitems 库零改动。

---

## 一、改动文件清单（4 文件，均为 publish 域）

| 文件 | 改动 | 对应发现 |
| --- | --- | --- |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/storage/SnapshotStorage.java` | `store()` 一步写正式名拆为 **`stage()` 唯一暂存 → `promote()` 原子落位** 两段式；类/方法 javadoc 全量重写（写入协议、C-1 根因、权衡登记）；`readBytes`/`relativeUri`/`sha256Hex` 不动；`deleteQuietly` javadoc 声明「发布链路仅清理暂存文件、正式名不受任何删除动作作用」 | C-1 |
| `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java` | `publish()` 存储段改写：stage → INSERT（撞 uk → 只删自己的暂存文件 → 5001）→ promote；promote 失败抛错令事务回滚；类 javadoc「版本单调与并发」段重写（两段式 + 为什么现在对了 + 孤儿权衡） | C-1 |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImplTest.java` | 旧顺序模拟用例 `concurrentPublishSecondGets5001` **替换为真两线程用例** `concurrentPublishTwoThreadsOneWinnerWinnerSnapshotIntactLoserNoTrace`（30 轮循环）；类 javadoc 更新；回滚/取回夹具 `storage.store(...)` → `stageAndPromoteFixture()`（stage+promote 最终态） | I-1（+C-1 断言） |
| `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/storage/SnapshotStorageTest.java` | 适配两段式 API：布局/取回/覆盖/删除五用例重写 + **新增同 (tenant,scope,version) 两次 stage 唯一性用例**（5→6 用例） | C-1 |

---

## 二、修复内容与「为什么现在对了」

### C-1：并发发布竞态摧毁胜者快照文件

**根因（评审原文核对属实）**：旧实现 `store()` 在 INSERT **之前**以 REPLACE_EXISTING 写确定性正式名 `<tenant>/<type>-<id>-<version>.json`；两并发线程同读末版算出同一 version，先后覆盖**同一文件**（败者内容覆盖胜者已哈希内容——破坏向量一）；败者 INSERT 撞 `uk_scope_version` 后的补偿 `deleteQuietly(uri)` 作用于**胜者已提交版本行指向的那个共享文件名**（破坏向量二）。每轮并发碰撞必然破坏 REQ-0033 验收 1「哈希可验、可取回」。

**修复方案（brief 建议方向之一：临时文件 + 插入成功后原子落位）**——两段式落位：

```
① stage：快照体写唯一暂存文件  <tenant>/.staging/snapshot-<type>-<id>-<ver>-<随机>.json.tmp
   （随机名一次性——并发发布各自暂存互不相撞，不占正式名）
② 版本行 INSERT（snapshot_uri=确定性正式名，brief D2 文件名契约不变）
   ├── 撞 uk_scope_version（败者）→ 只删自己的暂存文件 → 5001（语义不变，不重试取舍维持）
   └── INSERT 成功（胜者）→ promote：暂存文件 ATOMIC_MOVE 原子落位正式名
```

**为什么现在对了**（验收口径逐条）：

1. **胜者行哈希与文件内容恒一致**：正式名**只被「INSERT 胜者」调用过一次 `promote`**——落位内容=胜者自己 stage 的字节=胜者计算哈希的原文字节；败者全流程对正式名零写零删（它根本没有正式名的写路径了）。并发下唯一能写正式名 `<type>-<id>-<N>.json` 的线程就是成功插入 version=N 行的那个线程（uk 保证同版本只有一个 INSERT 胜者）。
2. **败者零痕迹**：撞键路径唯一的补偿动作是 `deleteQuietly(暂存uri)`——唯一随机名，不共享、不触碰正式名、不留残 file。
3. **5001 报冲突语义不变**：撞键即报 5001、不重试的取舍（类注释原论证）原样保留；每轮并发断言「恰好一 200 一 5001 + 冲突方 code=5001」。
4. **无新窗口**：promote 失败 → 抛错 → `@Transactional(rollbackFor=Exception)` 回滚 → **不存在「已提交行指向缺失文件」的状态**；「落位后事务回滚（如回写失败）」残留的正式名孤儿无行引用，下个同版本发布 promote 以 REPLACE_EXISTING 覆盖自愈；「暂存后/落位前崩溃」遗留唯一名暂存孤儿（不指向任何行）。三种孤儿均无哈希错位窗口——权衡已在 `SnapshotStorage` 类注释与 `IotPublishServiceImpl` 类注释登记。
5. 附带消除：旧实现两线程并发 `store` 对同一正式名的 REPLACE_EXISTING 竞争（Windows 下可能瞬时 AccessDenied）——新设计下同一版本只有一个线程落位，该窗口不复存在。

### I-1：并发测试按 brief 口径改真两线程

**旧用例缺陷**：两次顺序调用 + mock 第二次 INSERT 抛 `DuplicateKeyException`——对 5001 断言语义等价，但触达不了文件存储交错路径；且旧断言 `assertNull(readBytes("1/1-88-2.json"))` 把「胜者文件被删」当作正确的孤儿清理来断言，恰好把 C-1 固化成测试预期。

**新用例** `concurrentPublishTwoThreadsOneWinnerWinnerSnapshotIntactLoserNoTrace`（brief 测试口径「版本单调——并发发布 5001——两线程同 scope」逐条落实）：

- **真实两工作线程**同 scope(1,88) `CyclicBarrier(2)` 栅栏对齐起跑（先各自注册线程域 static mock——Mockito `MockedStatic` 为线程域、主线程注册不跨线程，经临时探针实验证实后删除探针；工作线程自注册 TenantContext/SecurityUtils），同走真实组装器/凭据终检/确认门/真实文件存储（@TempDir）；
- **uk_scope_version 物理唯一在 mock 层等价仿真**：`synchronized` 判重+占位键集，撞键抛 `DuplicateKeyException`（DB uk 语义）；
- **两线程内容分流**（点位数据按调用序给不同行）→ 两份快照哈希不同——同时覆盖「覆盖」与「删除」两个破坏向量；
- **每轮断言**：恰好一 200 一 5001（冲突方 code=5001）；两线程都真实到达 INSERT 边界（insert 调用数=2——真并发而非顺序）；**胜者版本行指向的文件内容哈希=行哈希**（C-1 验收口径）；全存储恰余胜者一个正式文件（败者零残留，`Files.walk` 清点）；胜者 `retrieveSnapshot` 全链路取回哈希核对一致；
- **30 轮循环**复跑（评审处置建议「多轮重复」）。

**TDD 红转绿留痕**：commit 4b7bbe3（仅测试）对旧实现第 1 轮即红（见下方红证据）→ commit da830e4（修复）绿 + 300 轮零复现。

---

## 三、验收命令原始输出

### 3.1 TDD 红证据（4b7bbe3 状态，对旧实现跑新并发用例）

命令：`cd backend && mvn test -pl ot-admin -Dtest='IotPublishServiceImplTest'`

```
[ERROR] Tests run: 19, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 6.638 s <<< FAILURE! -- in net.otlab.admin.module.iot.publish.service.impl.IotPublishServiceImplTest
[ERROR] net.otlab.admin.module.iot.publish.service.impl.IotPublishServiceImplTest.concurrentPublishTwoThreadsOneWinnerWinnerSnapshotIntactLoserNoTrace -- Time elapsed: 0.095 s <<< FAILURE!
java.lang.AssertionError: 第 1/30 轮: 胜者版本行指向的文件必须存在（败者不得删除/覆盖胜者文件） ==> expected: not <null>
Caused by: org.opentest4j.AssertionFailedError: 胜者版本行指向的文件必须存在（败者不得删除/覆盖胜者文件） ==> expected: not <null>
	at net.otlab.admin.module.iot.publish.service.impl.IotPublishServiceImplTest.concurrentPublishTwoThreadsOneWinnerWinnerSnapshotIntactLoserNoTrace(IotPublishServiceImplTest.java:373)
[ERROR]   IotPublishServiceImplTest.concurrentPublishTwoThreadsOneWinnerWinnerSnapshotIntactLoserNoTrace:392 第 1/30 轮: 胜者版本行指向的文件必须存在（败者不得删除/覆盖胜者文件） ==> expected: not <null>
[ERROR] Tests run: 19, Failures: 1, Errors: 0, Skipped: 0
[INFO] BUILD FAILURE
```

——第 1 轮即复现 C-1：真两线程并发下，胜者已提交版本行指向的文件被败者补偿删除摧毁（readBytes=null）；同轮其余 18 用例全过（红仅由 C-1 缺陷本身触发，非测试夹具问题）。

### 3.2 绿：发布域全类（修复后）

命令：`cd backend && mvn test -pl ot-admin -Dtest='IotPublishServiceImplTest,SnapshotStorageTest'`

```
[INFO] Tests run: 19, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 8.625 s -- in net.otlab.admin.module.iot.publish.service.impl.IotPublishServiceImplTest
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.371 s -- in net.otlab.admin.module.iot.publish.storage.SnapshotStorageTest
[INFO] Tests run: 25, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

（日志中的 hash mismatch / snapshot file missing ERROR 行为「取回-存储损坏探测」「回滚-篡改拒绝」负路径用例的**预期告警输出**。）

**胜者哈希一致性断言证据**（IotPublishServiceImplTest 并发用例逐轮断言体，30 轮×每次执行全过）：

```java
// 胜者版本行指向的文件实际内容哈希与行哈希恒一致（C-1 验收口径）
IotConfigVersion winner = winners.get(0);
assertEquals(2L, winner.getVersion(), "两线程同读末版 1 各算 version=2，胜者落库 2");
assertEquals(SnapshotStorage.relativeUri(TENANT, 1, GATEWAY_ID, 2L),
        winner.getSnapshotUri(), "snapshot_uri=确定性正式名（brief D2）");
byte[] stored = storage.readBytes(winner.getSnapshotUri());
assertNotNull(stored, "胜者版本行指向的文件必须存在（败者不得删除/覆盖胜者文件）");
assertEquals(winner.getSnapshotHash(), SnapshotStorage.sha256Hex(stored),
        "胜者行哈希=文件内容实际哈希");

// 败者零痕迹：全存储恰余胜者一个正式文件（无败者残file、无暂存残留）
try (var walk = Files.walk(tempDir)) {
    allFiles = walk.filter(Files::isRegularFile).toList();
}
assertEquals(1, allFiles.size(), "全存储恰余胜者一个正式文件（败者零残留）: " + allFiles);

// 胜者取回哈希核对一致（retrieveSnapshot 全链路含哈希核对）
when(versionMapper.selectOne(any())).thenReturn(winner);
SnapshotVO vo = publishService.retrieveSnapshot(1, GATEWAY_ID, 2L);
assertEquals(winner.getSnapshotHash(), vo.getSnapshotHash(), "胜者取回哈希核对一致");
```

### 3.3 竞态多轮复跑（零复现）

命令：`cd backend && mvn test -pl ot-admin -Dtest='IotPublishServiceImplTest#concurrentPublishTwoThreadsOneWinnerWinnerSnapshotIntactLoserNoTrace'` 外部连跑 10 次（每次内部 30 轮——合计 **300 竞态轮次**）：

```
RUN1:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (5.126 s)
RUN2:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (8.478 s)
RUN3:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (4.418 s)
RUN4:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (3.669 s)
RUN5:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (3.768 s)
RUN6:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (3.997 s)
RUN7:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (4.389 s)
RUN8:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (5.870 s)
RUN9:  Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (4.401 s)
RUN10: Tests run: 1, Failures: 0, Errors: 0 ... BUILD SUCCESS (4.515 s)
```

**10/10 全绿零复现。**

### 3.4 全量回归

命令：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin`

ot-core install 段：

```
[INFO] Installing D:\workspace\WisFactory\backend\ot-core\target\ot-core-1.0.0.jar to C:\Users\dowil\.m2\repository\net\otlab\ot-core\1.0.0\ot-core-1.0.0.jar
[INFO] BUILD SUCCESS
[INFO] Total time:  4.745 s
```

ot-admin 测试 Results 段：

```
[INFO] Results:
[INFO]
[INFO] Tests run: 2166, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:06 min
```

**2166 通过 / 0 失败 / 0 错误 / 0 跳过**（基线 2165 + 净增 1：并发用例 1:1 替换、SnapshotStorageTest 新增 stage 唯一性用例）。

### 3.5 改动范围核对

`git diff --name-only dca3dc9..HEAD` = 4 文件（见「一」），全落授权范围；未触碰 SQL/yml/frontend/其他域/workitems 库（唯一新写 workitems 文件为本报告）。

---

## 四、未做项与原因

1. **Minor 8 条（M-1~M-8）一律未动**——修订轮指令明确只修 C-1 + I-1；Minor 按评审处置建议记 TASK deferred 收口 LOG。其中 `deleteQuietly` javadoc 中对 M-5（根内校验硬化）只作「另行登记，不混入本修复」的指引性提及，未做任何 M-5 行为改动（`readBytes`/`deleteQuietly` 方法体与旧实现逐字一致）。
2. **未启动 8098 冒烟**——修订轮验收命令仅含单测全量 + 竞态多轮复跑；发布链 HTTP 层行为（端点/权限/响应信封）未改动，验收报告 §二-3 的 dev 冒烟结论不受本修订影响。8098 纪律自然满足（未启动）。
3. **未改 workitems 库任何文件**（含 TASK-0024 详情状态）——按指令留给协调者收尾。

## 五、偏离计划之处

1. **临时探针实验**：因 Mockito `MockedStatic` 线程域特性（主线程注册的 static mock 对工作线程不可见——实测 worker 读到真实 TenantContext 返回 null），先写临时探针测试实证「工作线程各自注册互不干扰、主线程注册不受影响」后再落并发用例；探针文件已删除、未入库。该实验决定了新用例「工作线程自注册 static mock」的结构（已在测试 javadoc 登记）。
2. **夹具方式**：回滚/取回夹具原用一步 `storage.store()`；`store()` 随修复移除后改 `stageAndPromoteFixture()`（stage+promote 两步等效最终态）。快照内容/哈希断言逻辑零变化。
3. **SnapshotStorageTest 净增 1 用例**（同版本两次 stage 唯一性）——超出「只改不增」的最小读法，但该用例直接固化 C-1 修复的结构前提（并发同版本暂存互不相撞），属修复的可测性组成，予以保留。

## 六、遗留问题

1. **Minor 8 条照旧**（评审清单 M-1~M-8，含 M-7 契约差异登记、M-5 硬化建议）——未动，待收口 LOG。
2. **暂存孤儿文件的清理工具**：进程在「stage 后/promote 前」崩溃会遗留唯一名暂存孤儿（不指向任何行、不影响取回，量级=单次崩溃一个文件）；当前靠 @TempDir 场景外无自动清扫——量级极低，建议与 M-5 硬化或 MinIO 二期一并考虑（如 promote 时顺手清扫同租户 .staging 下超龄文件），本期不做。
3. **并发发布语义边界**（非缺陷，登记备查）：本测试的 uk 仿真为内存键集；真实 DB 下 uk_scope_version 撞键由 MySQL 物理保证，5001 语义已在验收轮 dev 冒烟（200+5001）实测通过，两轮证据互为印证。

---

## 七、一句话结论

**修订轮通过**——C-1 以「stage 唯一暂存 → INSERT 胜者后 promote 原子落位」结构性消除（正式名只被胜者写一次，胜者行哈希=文件内容哈希恒成立、败者零残留、5001 语义不变），I-1 以真两线程栅栏对齐×30 轮用例红转绿留痕（4b7bbe3 红：第 1 轮复现胜者文件被毁；da830e4 绿：300 竞态轮零复现），全量回归 2166 测试 0 失败 0 错误，4 文件改动全落 publish 域授权范围。
