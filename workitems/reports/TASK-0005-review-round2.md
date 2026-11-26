# 复评报告：TASK-0005 修订轮 scoped 复评（fresh reviewer）

- 复评对象：修订轮提交 `509a82b`（6 文件，+249/-20，全部在 dataperm 包与测试内；`c2ff33e..509a82b` 反查一致，application.yml 未触碰）
- 输入：评审 brief、原评审报告（I-1/I-2 + Minor 6）、修复 diff 包（516 行完整读入）、ADR-0016 增补（2026-11-22）第 8/9 条、TASK-0005-report 六、修订轮一节
- 交叉验证（代码仓）：`DataPermServiceImpl`（守卫位置与 clientScope 构造）、`VisibleScope.denied()`（allTenant=false、集合全空）、`TenantContext.getTenantId()`（无登录态/会话未写租户/非 Web 上下文返回 null，与守卫判定形态吻合）、`DataPermConditionAssembler.pointCondition` 与 `TdengineTagConditionRenderer`（deptIds 空集时部门分支按 `!isEmpty()` 干净省略）、`exactScope` 全仓 grep 无残留引用

## 结论

**原两条 Important 均已修复。** 新发现 1 条（Minor，纯注释失准，无功能影响）。

### 原 I-1（中台 deptIds 从实例行推导 → 授权放大）——已消除

- `exactScope(rows)` 已删除（全仓无残留引用），subtree=false 与 subtree=true 两条路径统一经 `clientScope(visibleIds)` 构造，`deptIds` 恒为 `Set.of()`（增补第 8 条口径）。
- 中台 VisibleScope 唯一来源是 `resolveClientScope`，其任何分支（精确/展开/锚点空/denied/守卫拒绝）都不再产出非空 deptIds；点位条件（MySQL 组装器 L92、TDengine 渲染器 L80）均以 `!deptIds().isEmpty()` 守门，空集时部门分支整体省略、仅剩实例分支——不产生空 IN 或悬空 OR，fail-closed 方向正确。
- 用户路径 `computeUserScope` 的 deptIds（角色授权本体）未被触碰，不受影响。
- `DataPermClientScopeTest` 被修正的两条断言（deptIds={10,11}/{10,20,30} → 空集）与增补第 8 条逐字一致；diff 反查仅此两处断言及 DisplayName 变更，其余既有断言（instanceIds 集合、allTenant/createByOnly、captor、截断 WARN 逐字符）均未放松。

### 原 I-2（无租户上下文静默跨租户全量）——已消除

- `resolve` / `resolveClientScope` 入口均有 `TenantContext.getTenantId()==null → VisibleScope.denied() + WARN [DATAPERM] no-tenant-context user=<id|unknown>` 守卫；守卫置于缓存读取之前（无上下文连缓存都不触达，`verifyNoInteractions(cache, ...)` 锁定），语义最严。
- 返回值 `VisibleScope.denied()` 实测为 allTenant=false、deptIds/instanceIds 全空、createByOnly=false——与增补第 9 条「空 VisibleScope（allTenant=false、集合全空）」一致；WARN 格式与第 9 条 `[DATAPERM] no-tenant-context` 一致，user 后缀为 javadoc 契约并由守卫测试逐字符锁定。
- 不误伤合法路径：真实 `TenantContext.getTenantId()` 仅在无登录态/会话未写租户/非 Web 上下文返回 null（组件消费形态精确匹配）；守卫测试第 4 例（有上下文 → 正常进入六步计算、roleMapper 被调用、allTenant=true）锁定放行行为。组件当前未接线，无存量调用被误伤；且无上下文拒绝系增补第 9 条明文裁决。

## 新发现

1. **Minor · 新增守卫测试类内注释与自身实证事实矛盾**——出处：`DataPermTenantContextGuardTest.java` 类 javadoc（「未打桩时 getTenantId() 返回 null = 无上下文形态」）与 `setUp` 注释（「默认不打桩：getTenantId() 返回 null」）。mockStatic 未打桩时 Mockito 默认答案对包装 Long 返回 0L 而非 null——这正是修订报告自己记录的调试发现，也是同文件 `noTenantContext()` 助手注释所写的正确事实；同文件两处说法相互矛盾。功能上无影响（4 个用例全部显式调用 `noTenantContext()` 置 null，且依赖错误默认的用例会在 WARN 断言上响亮失败），仅注释失准可能误导后续维护者。建议收尾随 LOG 顺手改注释即可，不必进修订轮。

无其他新发现：测试隔离完整（4 个测试类 mockStatic 均在 tearDown 关闭、appender/mocks 释放）；无断言放松；空值语义（denied 形态、subjectLabel null→unknown）均有锁定；修复范围未触碰 Minor 6 条所涉代码，不使其恶化。

## 判定理由（一句话）

原 I-1 中台路径任何分支均不再产出非空 deptIds 且部门分支被消费端干净省略、被修正断言与增补第 8 条逐字一致且未放松其他断言；原 I-2 双入口守卫返回与增补第 9 条一致的空 VisibleScope + 可 grep WARN、真实 TenantContext null 形态精确匹配且合法路径有放行锁定；唯一新发现为新增测试类内一处与自身实证事实矛盾的注释（Minor，无功能影响）。
