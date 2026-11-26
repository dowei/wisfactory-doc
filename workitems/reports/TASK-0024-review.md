# 评审报告：TASK-0024 diff 评审（代码评审）

- 评审者：fresh 代码评审子代理（不共享协调者上下文）
- 评审对象：`TASK-0024-diff.txt`（BASE=a1c6649，HEAD=8566f57；ab9481a=实现 28 文件 + 8566f57=gitignore，共 29 文件 +3780 行）
- 评审口径：L1-execute §13「diff 评审」+ 实施 brief 接口与决策 9 条 + 评审重点 10 条
- 依据物：diff 包、实施 brief、TASK 详情、FEAT-0008 / REQ-0033 / TC-0008 / ADR-0008 / ADR-0006、config-snapshot-schema.md v1.0.1、工作区源码核对（backend/ot-admin、ot-core、resources/sql）
- 日期：2026-11-22

## 结论

**不通过（1 Critical + 1 Important，进 §12 退回路径）。**

主体质量高：Schema v1.0.1 字段级一致性、凭据双保险、确定性序列化、取回哈希核对、回滚先验后插、升级脚本幂等、application.yml 键纪律逐项核对均过（见下表）。但**并发发布竞态下的孤儿补偿删除会摧毁胜者已提交版本的快照文件**——恰发生在验收明文的并发场景（TC-0008 步骤 5），使 REQ-0033 验收 1「哈希可验、可取回」对胜者版本失效；配套的并发测试按 brief 口径应为「两线程同 scope」，实际弱化为顺序 mock 模拟，未能暴露该缺陷。

## 越界反查（复核）

- `git diff --name-only a1c6649..HEAD` 实测 29 文件：publish/main 20 + application.yml 1 + test 6 + upgrade SQL 1，全部落在 brief「改动范围」声明内。✓
- 声明范围外仅 `.gitignore`（提交 8566f57）：实测该提交仅 `.gitignore` 1 文件 3 行新增（空行 + 注释 + `backend/data/`），无其他内容。✓ 予以认可。

## 评审重点 10 条逐项核对

| # | 重点 | 结论 | 依据 |
| --- | --- | --- | --- |
| 1 | 快照组装严格按 Schema v1.0.1 | **过** | 五对象 POJO 与契约字段表逐字段比对一致（point 18 / channel 10 / dataItem 20 / formula 6 / metric 15），camelCase、审计五列剔除、additionalProperties=false 键集复核、装载口径（网关=points+channels、计算=dataItems+formulas+metrics、未装载键省略）、顶层键与 schemaVersion 一并下发均符合；ConfigSnapshotAssemblerTest 断言字段完整性与键省略 |
| 2 | 凭据排除双保险 | **过（附 M-1/M-2）** | 保险一=组装器白名单（凭据表不在数据源，仅派生 hasCredential/credentialTypes 标记）；保险二=SnapshotCredentialGuard 顶层键集 + 五对象键集 + 八词键名全深度递归 fail-closed，与 0018 ChannelConnConfigValidator 同款词集（已核对源码一致） |
| 3 | 版本单调 + 撞 uk 5001 路径 | **不过（C-1）** | uk_scope_version 物理兜底正确；「撞键即报 5001 不重试」取舍论证充分（重试会把两人冲突静默变先后两版，违背 REQ-0033/TC-0008）——但 5001 路径的孤儿补偿删除作用在确定性共享文件名上，摧毁胜者快照（见 C-1） |
| 4 | 回滚=新版本行复用旧快照 | **过（附 M-7）** | 先取回目标快照并核对哈希（损坏/缺失 fail-closed 拒绝回滚、不产生指向坏文件的行）后插新行；uri/哈希复用、历史行不改写、目标不存在 400、并发回滚撞 uk 5001 且无文件操作（无 C-1 同类风险）；brief D5 复用裁决与「报告说明」义务在类注释落实 |
| 5 | 取回哈希核对 | **过** | retrieveSnapshot 取回后 sha256Hex 与落库比对，不一致/文件缺失均 log.error 告警 + 500；SnapshotStorageTest 断言 SHA-256 标准向量与格式；回滚路径同款校验 |
| 6 | 确定性序列化 | **过** | @JsonPropertyOrder 全类钉死 + ORDER_MAP_ENTRIES_BY_KEYS（对任意深度 Map 生效，含 conn_config/protocol_addr 嵌套）+ JSON 列解析 TreeMap + 行序 id 升序双保险 + NON_NULL；测试以乱序行序两次组装断言逐字节一致 |
| 7 | PublishConfirmGate + NoOp | **过** | 接口 javadoc 载明 0015 接线义务三步（含 5002 登记归属 0015、本任务不动 ot-core）；NoOp 恒放行；服务层落库前调用；测试覆盖 false 分支拒绝路径 |
| 8 | 数据权限防线 | **过** | 写路径：网关 scope 存在+启用+可见（dept/instance 锚点，NULL 不特判 fail-closed）、计算 scope 仅 allTenant（管理员语义）；读路径同防线（不要求启用）；测试红转绿覆盖 403/400 各分支 |
| 9 | application.yml 仅新增 ot.publish.* | **过** | diff 仅 +2 行（ot.publish.storage-path 一键），其余键未动；@ConfigurationProperties(prefix="ot.publish") 对应 |
| 10 | 升级脚本 | **过** | p33 DDL 与 brief 提取逐字一致 + uk_scope_version 差异头注（含 5001 语义与回写上游建议）；4070-4073 INSERT IGNORE 显式 id 幂等（sys_menu PK=id 已核、sys_role_menu 复合 PK (role_id,menu_id) 已核——二次运行不重复插）；4015 在位（20261121_iot_console_menus.sql）、4069 已被 0048 占用、4070-4073 全库无冲突；sys_menu 列清单与 01_base.sql 及既有脚本范式一致 |

## 发现清单

### Critical

**C-1 并发发布竞态：撞 uk 后的孤儿补偿删除摧毁胜者已提交版本的快照文件**
- 出处：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImpl.java:164-171`（diff 包行 723-730，publish() 的 DuplicateKeyException 分支）与 `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/storage/SnapshotStorage.java:62,102-104`（store 的 REPLACE_EXISTING + relativeUri 确定性文件名）、`:93-101`（deleteQuietly）
- 说明：snapshot_uri 文件名由 (tenant, scope_type, scope_id, version) 确定性生成，两并发发布线程同读末版后必计算同一 version、`store` 先后以 REPLACE_EXISTING 覆盖**同一文件**（后写者内容覆盖先写者，先写者落库哈希与文件已不符）；败者 INSERT 撞 uk 后 `deleteQuietly(uri)` 删除的正是**胜者版本行指向的那个共享文件名**——故 TC-0008 步骤 5 的每一次并发碰撞都必然使胜者已提交版本「哈希可验可取回」被破坏（取回/回滚转 500「快照文件缺失」），REQ-0033 验收 1 在并发场景失效；5001 报错本身正确，破坏在补偿动作作用于共享文件名（修复方向建议：文件名掺内容哈希或插入成功后再落正式名，或撞键路径不做删除而交由取回侧哈希探测兜底）。

### Important

**I-1 并发测试未按 brief 测试口径「两线程同 scope」，弱化为顺序 mock 模拟，恰好掩盖 C-1**
- 出处：`backend/ot-admin/src/test/java/net/otlab/admin/module/iot/publish/service/impl/IotPublishServiceImplTest.java:269-287`（diff 包行 2860-2878），类注释 `:73` 自认「顺序模拟」
- 说明：brief 测试口径明写「版本单调（并发发布 5001——**两线程同 scope**）」；现实现为两次顺序调用 + mock 第二次 insert 抛 DuplicateKeyException——对 5001 断言语义等价，但触达不了文件存储交错路径，胜者文件被删的破坏路径因此不可见。

### Minor（记收口 LOG 的 deferred，不进修订轮）

- **M-1** `SnapshotCredentialGuard.java:98-99,182-196`（diff 包行 1664、1750-1770）——凭据豁免两键（hasCredential/credentialTypes）按「键名任意深度」生效，包括 connConfig/protocolAddr 自由对象内部；藏于该两键名下的值不参与值扫描。写入侧 0018 校验器已拒含 credential 子串键（已核），残余路径极窄，建议豁免收敛到 channel 对象层。
- **M-2** `SnapshotCredentialGuard.java:94-95`（diff 包行 1657-1661）——值扫描仅三词（password/passwd/secret），token/apikey 仅键名扫描：brief D8 字面「非法键」已满足（键扫描全八词），收窄有 0018 authType=apiKey 先例支撑且 javadoc 登记；仅提示与 brief「credential/password/secret/token 等」宽读的差距，属已登记的设计取舍。
- **M-3** `SnapshotCredentialGuard.java:140-162`（checkObjectArray）——对象键白名单只做「键 ⊆ Schema」（additionalProperties=false 方向），未复核各对象 required 字段方向；DB NOT NULL 约束兜底实际不可达，提示契约冻结口径的 required 侧未机械复核。
- **M-4** `IotPublishServiceImpl.java:460-465`（diff 包行 1019-1026）——`pointIdsOf(byte[])`/`dataItemIdsOf(byte[])` 无调用方（回滚路径经 verifyTargetSnapshotAndExtractIds→idArrayOf），死代码。
- **M-5** `SnapshotStorage.java:107-110`（resolveAbsolutePath）——readBytes/deleteQuietly 对 DB 中 snapshot_uri 无根内归一化校验（理论上 `../` 可逃逸存储根）；该值仅由发布链路写入、攻击向量需先破库，建议加根内校验硬化。
- **M-6** `IotPublishMenuPermissionGuardTest.java:61-69`——守卫测试先按预期键过滤再断言相等，同脚本内 4015 下多出的非预期行（如未来 4074）不会红；「恰好四行」语义打折（控制器侧 exact 集合测试有补偿）。
- **M-7** `IotPublishServiceImpl.java:619-624`（类注释权衡登记）——回滚复用旧快照体导致快照体顶层 version=目标版本原值 ≠ 行信封 version，契约 §3.3「顶层 version 与响应信封 version 同值」对回滚产生的版本不成立（网关本地缓存自校验语义待对齐）；brief D5 已裁决复用且 javadoc 已登记，建议进契约差异登记 / 上游补充稿对账清单。
- **M-8** 组装器/存储未做万级点位分片/压缩——FEAT-0008 异常与边界自标「次要」，brief 未要求，本期不阻塞；登记规模上限观察项。

## 通过项与核对记录（要点）

- ResultCode.VERSION_CONFLICT=5001 / PARAM_ERROR=400 / FORBIDDEN=403 / NOT_FOUND=404 / ERROR 均已在 ot-core 核对存在；@OperateLog 审计 + 测试断言（REQ-0033「单独权限+审计」）。
- BaseEntity（@TableLogic deleted）+ OtTenantLineHandler iot_ 前缀自动租户注入核实；IotConfigVersionMapper 裸 @Select 显式 tenant_id+deleted=0 为防御纵深，口径正确。
- ServiceImpl 导入（com.baomidou.mybatisplus.spring.service.impl.ServiceImpl）与全仓既有服务一致。
- IotGateway.instanceId/deptId、IotPoint/IotDataItem.configVersion 列（0021/0013 建列承接）核实存在；writeBack 以快照内容 id 精确回写（回滚取复用快照解析，不误标期间新增行）。
- SnapshotStorage 原子落位（临时文件 + ATOMIC_MOVE + 降级路径）、deleteQuietly 容错、sha256Hex 64 位小写十六进制符合 varchar(64)。
- 测试面：组装器字段完整性/键省略/确定性/凭据终检/JSON 列 fail-closed、guard 八词与豁免、存储布局/哈希/覆盖/删除、控制器四端点与权限码集合、菜单守卫三测、确认门 false 分支、数据权限防线——覆盖面与 brief 测试口径对照，除 I-1（两线程）外均落实。
- .gitignore（8566f57）仅含 gitignore 增项核实。

## 处置建议

- C-1 + I-1 同轮修复：改并发发布撞键路径的文件处理（消除共享文件名上的破坏性补偿），并将并发测试补为真实两线程（或至少在并发用例后补「胜者版本取回哈希可验」断言）；修复后按 §13 做 scoped 复评（只看修复 diff 与本清单）。
- Minor 8 条记 TASK `任务结果` deferred，收尾进 LOG。
