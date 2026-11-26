# 实施 brief：TASK-0083（发布时未锁定实例自动跟随推进——TD-0059 处置）

## 批次

- TASK-0083（本批共 1 个，标准任务 2 人天；backend 串行道——FEAT-0005 三档语义收官任务，前置 0081✅）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/publish/**`（发布事务接线——最小改动）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/**`（调用 0081 的 applyToVersion 接缝+未锁定筛选，不改 0081 编排内核）
  - `backend/ot-admin/src/test/java/**`（红线测试）
  - 不碰 frontend/ingest/calc/契约/SQL/yml/ot-core；**0081 已交付的 IotUpgradeApplyServiceImpl 内核零改动**（本任务只是调用方）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0083.md`
- 决策权威：`decisions\ADR-0008.md`（二档政策=随发布生效，未锁定实例自动跟随；T3 永不自动）
- 接缝交付：`reports\TASK-0081-report.md` + `reports\TASK-0081-review-round2.md`（applyToVersion(instance,target) 可复用边界 + **m-1 deferred：applyToVersion 经 regenerate 间接依赖 SecurityUtils/TenantContext——本任务必须显式核实发布事务上下文是否满足**）
- 上游代码（开工先读）：iot/publish/service/impl/IotPublishServiceImpl.java（publish 事务边界+组装读实例行）+ iot/template/service/IotUpgradeApplyService.java + impl（0081 接缝签名与语义）+ IotModelInstance（locked/applied 字段）+ 0016 待升级清单筛选逻辑
- 关联：FEAT-0005/REQ-0006/TC-0005/TD-0059（todo.md）

## 接口与决策（逐条结论，不再讨论）

1. **接线点（2026-11-24 round1 评审 F-1 计划缺陷纠正——以本条为准）**：发布事务内、**快照组装（assemble）之前**调用 `applyToVersion` 为未锁定实例推进——与 0081 确认升级同机制（同步/事务内/逐台留痕）；事务边界以 IotPublishServiceImpl 现有 @Transactional 为准（不另开事务）。**修订理由**：计算引擎运行时**只读落位快照文件、不读实例行**（0040 HttpSnapshotFetcher→SnapshotReconciler→DualTriggerExecutor 实证）——原裁决"快照存储完成后 follow"导致当期快照保留跟随前基线，T2 二档「随发布生效」落空且静默（applied 已推进、确认窗口关闭、无后续发布即永不生效）。**新顺序=确认门 → follow（推进未锁定实例行）→ assemble（读到新行）→ SHA-256/store → 版本落库 → publish_log/notify**。follow 失败整批回滚、零半态；验收须断言**当期发布快照即含跟随后的新值**（不再接受"下一期快照才变"）。
2. **筛选口径**：同租户 + 引用目标模板（scope_type=2 计算域发布才涉及模板实例；scope_type=1 网关域**不触发实例跟随**——网关 scope 与模板实例无关）+ locked=0 + applied_template_version < 模板当前 version；T3 变更**永不自动**（0081 编排内核已按档位区分，本任务不重复判定——调用前若为纯 T3 变更则跳过自动跟随，实例仍走 0016 逐台确认）
3. **上下文核实（0081 m-1）**：applyToVersion 经 regenerate 依赖 SecurityUtils.getUserId()/TenantContext——发布事务运行在管理面登录会话中（publish 端点带 @SaCheckPermission），上下文应齐备；执行者开工实证：在发布场景直接调用 applyToVersion 不抛上下文异常、留痕操作人正确；若不适用→停写报告等裁决
4. **失败语义**：自动跟随失败=发布失败（整批回滚）——与 0081 confirm 同 fail-closed 纪律；不能出现"版本已发布但实例未跟随"半态
5. **不重复**：锁定实例不推进（保持旧版本 + 仍在 0016 清单）；T3 不推进；不改 publish 对外契约（四端点签名不动）；不改 0081 编排内核
6. **测试口径**：红线测试——未锁定实例发布后 applied+数据面生效（snapshotHash 变化实证）；锁定实例发布后保持旧版本且清单可见；网关 scope 发布不触发实例跟随；T3 变更发布不自动跟随；上下文核实断言；失败整批回滚。TC-0005 步骤 3 全绿。

## 验收命令

环境事实（2026-11-24）：MySQL/Redis 宿主 phpStudy（8.0.12@3307 / 3.0.504@6379）；sql-exec.sh 每条语句单独一次调用；8098 启动前探测、用毕即停；TDengine 本批不涉（如需 env 注入 OT_TDENGINE_BASE_URL）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. dev 冒烟（每步原始输出贴回）：T2 模板变更保存 → 发布计算域 scope → 未锁定实例 applied 推进+快照哈希变化实证 → 锁定实例保持旧版且在清单 → 网关 scope 发布不触发 → T3 变更发布不自动跟随 → 失败回滚红线
3. prod 档真实启动回归 → 0 ERROR
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内（0081 内核/SQL/yml/ot-core/前端/calc 零触碰实证）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0083-report.md`。五项齐全 + 上下文核实结论 + 一句话结论。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0083）。
