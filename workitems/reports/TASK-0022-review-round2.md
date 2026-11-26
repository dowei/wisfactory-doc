# TASK-0022 修订轮 scoped 复评报告（I-1 导入引用存在性校验）

- 评审者：fresh reviewer（不共享协调者上下文，no-subagents，只评审不修改）
- 复评日期：2026-11-22
- 范围：仅判定修订轮 I-1（导入 deptId/boundInstanceId 缺存在性/同租户校验）是否修复、修复有无新缺陷；不重开全场评审（0023 节不在范围，原 Minor 7 条已 defer 不在判定范围）
- 输入：评审 brief / 原评审报告 TASK-0022-0023-review.md / 修复 diff TASK-0022-fixdiff.txt（3 文件 213 行逐行读完）/ 协调者裁决口径 / 实施报告 §6 修订轮节；修复关键断言已对当前工作区源码（HEAD=071dd1f）逐点实证

## 结论

**I-1 已修复。新发现：0（Critical 0 / Important 0 / Minor 0）。**

判定理由：悬空引用通道已封死——deptId/boundInstanceId 携带即受「格式→存在性」双闸约束（格式非法由既有 INVALID_NUMBER 承载，存在性/同租户/未删由新增 INVALID_REF 承载），未命中行整批拒且拒发生在任何 INSERT 之前，校验语义经源码实证与单点建档 validateDept/validateInstance 完全同口径。

## 逐点实证（对工作区源码，非仅 diff 文字）

1. **校验链完整**：第一遍 `validateImportFields`（IotPointServiceImpl L776-781）对携带但不可解析的 deptId/boundInstanceId 判 INVALID_NUMBER；第二遍 `markReferenceViolations`（L880-917）对携带且可解析的 id 去重后批内一次 IN 预检（`deptMapper.selectByIds` / `instanceMapper.selectByIds`），未命中标 `INVALID_REF`，文案逐行标注缺失列名与 id（如 `deptId(777)`）。两遍共用同一 `parseLong`（L1018-1027，trim+异常吞返回 null）——首轮放行的行到引用校验必然可解析，无解析口径漂移；空白=未携带不触发校验，与 DDL 可空及单点建档口径一致。
2. **同租户/未删保证属实**：`sys_dept` 在 OtTenantLineHandler fail-closed 白名单内（TENANT_TABLES L38-44），`iot_model_instance` 经 `iot_` 前缀规则自动纳入（L46-47、L64）；`selectByIds` 与单点建档 `selectById`（validateDept L1253 / validateInstance L1242 用同两个 mapper）走同一拦截器链——多租户插件注入会话租户条件 + BaseEntity `@TableLogic`（L38-39）过滤已删行，契约完全对齐。validateInstance 注释「不校验 status」的口径在导入侧同样成立（selectByIds 不加 status 条件），一致。
3. **整批拒语义未被破坏**：`failRow` 置 success=false → `failCount > 0` → L670-678 抛 PointImportRejectException，该点在配额预检与落库循环（L700 起）之前，事务内未执行任何 INSERT；新测试断言 `verify(baseMapper, never()).insert(...)`，dev 冒烟 S2 实证坏引用行混批整批拒（persisted=false、逐行 INVALID_REF 标注）、S3 修正重导 200。
4. **IN 批查性能符合裁决口径**：去重后每个 mapper 至多一次查询（与批大小无关，测试 `times(1)` 实证），上限受 IMPORT_MAX_ROWS=10000 约束，无逐行往返放大。
5. **原因码一致性**：`REASON_INVALID_REF` 为 PointImportRowResult 常量体系新增项，命名/形态与既有 REASON_ 系列一致；原因码为纯 String 常量经回执透出，无其他枚举/switch 需联动。
6. **断言无放松**：存量断言仅追加 message 参数（诊断强化）；新增 3 用例覆盖「deptId 悬空拒」「boundInstanceId 悬空拒」「命中放行 + 批查次数」正反两向。
7. **改动范围收敛**：3 文件 +144/−6 全落 point 域导入校验器与测试，0023 已验收部分零触碰，与实施报告 §6.4 git diff 核对一致。
8. **TDD 与回归**：红转绿记录（3 用例红 → 21/21 绿）+ 全量 1866 通过（§6.2）+ dev 冒烟两轮复核（§6.3）齐备。

## 新发现

无。（观察项，不计级：`failRow` 后写覆盖先写标签，故同现「引用悬空 + 编码占用」的行最终标注 DUPLICATE_CODE 而非 INVALID_REF——该覆盖行为系修复前 failRow 既有设计（unit/channel/occupied 同此），整批拒结论不受影响，非本轮修复引入，不恶化任何 deferred Minor。）
