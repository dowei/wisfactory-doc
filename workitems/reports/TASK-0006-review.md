# TASK-0006 diff 评审报告（fresh reviewer）

- 评审对象：`reports/TASK-0006-diff.txt`（单提交 db16079，22 文件约 1930 行，已完整读入）
- 依据：brief 契约一致性 + 明显质量缺陷（缓存与失效正确性，从严），口径 L1-execute §13
- 交叉核对：实施 brief / TASK 详情 / 实施报告 / ADR-0016（裁决 5/6 与增补）/ REQ-0002 验收 4 / FEAT-0002（TTL 10 分钟）；仓库 HEAD=db16079 逐点核实（DataPermServiceImpl 接线、VisibleScope record、@Transactional 注解、rawStringRedisTemplate、全仓变更点 grep）
- **结论汇总：Critical 0 / Important 2 / Minor 6**

## 一、brief 契约逐项核对（均通过）

| 契约点 | 核对结论 |
| --- | --- |
| 失效即失效模型 | key=`iot:dataperm:scope:{userId}:0`（roleVer 占位 0），SCAN+DEL 非 KEYS，单用户/全量两粒度均实现且有 SCAN pattern 断言 |
| 四变更点挂 afterCommit | 事务方法（insertRole/updateRole/deleteRoleById、updateDept/deleteDeptById、updateUserRoles/updateUser、updateInstance）注册 afterCommit ✓；非事务方法（updateRoleMenu/updateRoleDept/updateDataScope/绑定三方法/insertDept）立即发布——与 0003 的偏离有成文理由且核实证词属实（注解逐个核实），方向正确（失效是安全语义不允许跳过） |
| 回滚行为 | afterCommit 仅在真实提交后触发（Spring 语义），回滚不发布 → 缓存与回滚后 DB 一致，无幻影失效 |
| cache-enabled 开关与 NoOp 兜底 | 互斥单 Bean，matchIfMissing=true，ApplicationContextRunner 三态断言；误配非 true/false 值时启动快速失败（可接受） |
| resolve 先缓存后计算 | DataPermServiceImpl.resolve：守卫→get→未命中 compute→put（0005 接缝零改动属实）；租户守卫 denied 路径不回填缓存（正确，瞬态不入缓存）；新增回填+二次命中不触库测试 |
| TTL 600s 可配 | 默认 600，≤0 回落 600 并 WARN（防无 TTL 永久键），短 TTL 传递有测试，dev/prod 冒烟 TTL=592/599 实证 |
| JSON VisibleScope 往返 | record 规范构造器对集合做 Set.copyOf 防御性拷贝、null 归一空集——Jackson 往返后经构造器再归一，无 Set 可变性/丢失问题；冒烟 S5c 内容与库内数据逐项核对一致 |
| SCAN 游标完整性 | Cursor 全量排空 + try-with-resources；SCAN 期间 DEL 不漏（删的就是目标键）；分批快照 List.copyOf 后 clear 复用缓冲，无并发修改 |
| 异常不炸主流程 | 监听器与发布器双重 try/catch 收敛，缓存 get 降级为重算、put/evict 静默；Redis 不可达时权限语义保持准确（既不放大也不收缩），与 M-1 租户守卫 fail-closed 语义正交不冲突 |
| 变更点遗漏面（grep 交叉核） | 角色无独立启停方法（状态只经 updateRole）；sys_user_role 写点仅三处服务（initTenantAdmin 只绑新建用户、deleteUserById 死键）均无需失效；sys_role_dept 写点全在 SysRoleServiceImpl 已挂；部门写点全经 SysDeptServiceImpl 已挂；实例 status 不参与可见集求解（selectAllForPerm 仅 deleted=0）——status-only 变更不失效正确；**遗漏：实例 create/delete（见 I-2/M-2）** |

## 二、发现清单

### Critical（0）

无。

### Important（2）

1. **I-1 / Important / 出处：`DataPermCacheEvictPublisher` afterCommit 挂点（失效模型整体）**——提交后失效不防反方向竞态：变更事务提交前已在途的 resolve（计算读的是旧数据）会在 afterCommit 失效**之后**完成回填，旧可见集以新键写入缓存，权限放大窗口最长 TTL 600s（REQ-0002 验收 4「即时生效」在该窗口不成立）；afterCommit 只解决了「未提交先失效被回填脏数据」单方向，无版本号/延迟双删等反向防护——模型系 brief 裁决内的已知取舍，建议协调者裁决：接受为残余风险登记 TD，或后续任务版本化 key 收敛。
2. **I-2 / Important / 出处：`IotModelInstanceServiceImpl`（createInstance/deleteInstances 未挂失效）+ 实施报告 §3.2**——实例创建即绑部门（IotInstanceCreateForm.deptId）但无失效挂点：新建实例对 dept-scoped 用户最迟 10 分钟不可见，相对 0005 直查（每请求现算）是用户可见的行为回退（fail-closed 方向、TTL 有界）；brief「实例侧仅一处」约束内实现无越界，但报告 §3.2 称「登记遗留 §5.2」指向错误（§5.2 实为冒烟数据清理项），该缺口实际未进遗留清单——需协调者裁决补挂点（create/delete 各一行 evictAll 即可）或明确接受并重登记 TD。

### Minor（6）

1. **M-1 / Minor / 出处：`SysRoleServiceImpl.updateRoleDept`/`updateDataScope`（无 @Transactional）**——delete 成功后 insert 中途失败时 DB 已部分变更而 evict 因异常未发布，旧自定义范围残留 ≤600s（放大方向）；根因是既有缺事务注解（超出本 diff 声明范围），现状下发布位置已是正确选择，updateRoleMenu 不影响 VisibleScope 本就冗余无此风险。
2. **M-2 / Minor / 出处：`IotModelInstanceServiceImpl.deleteInstances`**——实例删除不失效，残留 instanceIds ≤600s；已删实例行被 deleted=0 与 IN 过滤天然排除、TD 侧仅原可见者在窗口内看原本可见的数据（方向无害），报告已登记。
3. **M-3 / Minor / 出处：`DataPermCacheEvictListener`/`DataPermCacheEvictPublisher`/`RedisVisibleScopeCache` 各 catch 块**——log.warn 只带 `e.getMessage()` 丢堆栈，SCAN/DEL/序列化失败根因难定位；建议 warn 附 throwable（与既有 kickOut 的 `log.warn(msg, e)` 范式对齐）。
4. **M-4 / Minor / 出处：`DataPermCacheEvictPublisher.afterCommit` → 监听器同步执行**——全量失效 SCAN+DEL 在已提交写请求的线程内同步执行，键空间增大时拖长该请求尾延迟（COUNT=200 提示与分批 DEL 缓解；单机一期键量=用户数级可接受，大键空间/多实例形态需异步化或 pub/sub）。
5. **M-5 / Minor / 出处：`RedisVisibleScopeCache` key 模板**——键无租户段，正确性依赖「用户终身单租户」不变量（当前由 insert/update 的 tenant_id 服务层会话覆盖保证，REQ-0001/TASK-0002）；该跨域不变量未在键 javadoc/ADR 留痕，若未来引入换租户或多租户成员即缓存跨租户污染。
6. **M-6 / Minor / 出处：实施报告 §2.1**——新增用例加法笔误「12+6+3+3+13+13+5+4+3」=62≠50（13 重复计入、ResolveScope +1 漏计），最终 1483+50=1533 本身正确（§1 表逐项相加为 50 无误）；报告账目勘误，不影响验收结论。

## 三、评审处置建议

- I-1/I-2 进 §12 退回路径的裁决面：两项均为 brief 裁决内取舍/声明的边界缺口而非实现走样，可作为「接受为残余风险 + TD 登记」处理；若选择代码收敛，I-2 修复极小（create/delete 各挂一行 evictAll，在 brief 声明的 instance impl 文件内），I-1 修复需版本化 key（建议独立任务）。
- Minor 6 条按 §13 记 TASK `任务结果` deferred，不进修订轮。
