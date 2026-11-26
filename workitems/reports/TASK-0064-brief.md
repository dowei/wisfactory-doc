# 实施 brief：TASK-0064（高危权限点清单注册与菜单种子——机械差集收口）

## 批次

- TASK-0064（本批共 1 个，标准任务 2 人天；backend 串行道——权限种子审计补全）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `resources/sql/upgrade/20261123_iot_permissions_complete.sql`（新增——差集补注册种子）
  - `backend/ot-admin/src/test/java/**`（菜单权限守卫测试扩展）
  - 清单导出脚本/比对脚本（临时产物放 `resources/scripts/` 或报告内嵌，**不入升级脚本**）
  - 不碰既有种子行、前端、ingest、ot-core

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0064.md`
- 权限清单权威：详细设计 §3.1/§3.1续（p22-23）/§3.4/§3.5 接口表——D:\Documents\dowei\2.项目资料\智慧工厂\factory-data-platform-detail-design-v1.1.pdf（**逐接口权限标识列机械提取**；p22-23 文本已由协调者提取过，执行者可用 python pypdf 重提取核对）
- 已交付种子基线（比对目标）：resources/sql/upgrade/ 全量 grep `sys_menu`（0017/0018/0019/0021/0024/0051/0052 等各任务种子——iot:instance/item/metric/point/channel/credential/dict/publish/apiclient/openapi:client 等已在位）+ sys_menu 现库实态（sql-exec.sh 查询）
- 范式：菜单种子幂等+超管授权照 0024/0052 脚本；菜单权限守卫测试照 IotPublishMenuPermissionGuardTest / 0052 菜单守卫测试范式
- 边界：班次日历权限点一期不建（TASK 明文）；状态规则域权限（0015/0016 交付面——statusrule 接口权限位是否已有种子，比对确认）

## 接口与决策（逐条结论）

1. **机械差集法**：①从 PDF §3.1/§3.1续/§3.4/§3.5 提取「路径→权限标识」全清单（含高危项：公式/绑定/部门绑定/状态规则/阈值/发布回滚/凭据/重算）②提取库内 sys_menu 已注册权限标识全集（现库实态+种子文件双源）③差集=缺注册项 → 逐项补种子（menu_type=3，3000-3999 段顺延——现最大编号后顺延，头注登记段位）④比对脚本输出贴报告（防漏防错防「看似有了」）
2. **高危单独权限位核验**：清单中高危操作逐项确认有独立权限位（与普通 CRUD 分离）——已分离的不动，缺失的补
3. **角色初始化段**：普通查看角色默认无配置权——种子不做角色反向授权（只注册权限点+超管授权，超管范式照 0024/0052）；普通角色 403 由守卫测试锁（sa-token 无授权即 403 天然成立，测试锁定防误授权）
4. **接口-权限一一对应断言**：守卫测试逐接口 @SaCheckPermission 注解值 ↔ 种子权限标识比对（机械一一对应——0052/0024 guard 测试范式扩展）
5. **不碰**：既有种子行（幂等增量）、0051 鉴权链、前端（权限位前端已用 v-privilege 消费，无需改）

### 测试口径（TDD）

守卫测试扩展：新增权限点的接口 403 断言（普通角色）；一一对应断言（注解值⊆种子集）；种子幂等两次执行。

## 验收命令

环境事实（协调者 2026-11-23 实测）：WSL docker mysql/redis 在跑；sql-exec.sh 每条语句单独一次调用；8098 prod 启动回归用（探测/用毕即停）；TDengine 本批不涉。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 种子二次幂等 → 两次 DONE；差集比对脚本输出贴报告（清单全量 vs 库内实态——**零缺失**终态）
3. dev 冒烟：普通查看角色（或无授权用户）直调 3 个新补权限接口 → 403 原始输出
4. prod 档真实启动回归 → 0 ERROR
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0064-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；不改范围外文件（既有种子行零触碰）；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0064）。
