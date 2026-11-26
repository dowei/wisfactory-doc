# 实施 brief：TASK-0065（操作审计接线与时间线分工核查——机械覆盖审计）

## 批次

- TASK-0065（本批共 1 个，标准任务 2 人天；backend 串行道——前置 0064 已闭环）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/**`（**仅限补 @OperateLog 注解与既有切面适配**——各写接口缺口补接线，不改业务逻辑）
  - `backend/ot-admin/src/test/java/**`（覆盖断言守卫测试扩展）
  - 清单比对脚本（临时产物落 resources/scripts/ 或报告内嵌）
  - 不碰 ot-core、前端、ingest、既有切面核心（OperateLogAspect 行为不动）、**不建 iot_data_item_history**（0039 未交付——只做分工核查登记）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0065.md`
- 权威：详细设计 §3.1/续/§3.4/§3.5 写操作接口表（p21-23——pypdf 提取，0064 脚本 export-design-perms.py 可复用）+ ADR-0006（凭据读写全量审计）
- 上游交付（开工先读）：`reports\TASK-0064-report.md`（机械差集法+守卫测试全局口径范式——本任务直接复用）+ OperateLogAspect 现状（grep 定位）+ 0051/0052（openapi 域 @OperateLog 实例）+ 0024/0015/0016（publish/template 域 @OperateLog 实例）+ 0019（凭据审计交付面）
- 平台现状：sys_operate_log 表与 OperateLogAspect（框架件——saveResponse 短路口径见 0052 评审实证）

## 接口与决策（逐条结论）

1. **机械覆盖审计**（照 0064 差集法）：①提取设计写操作接口全清单（POST/PUT/DELETE 类）②grep 现库各 controller 写端点的 @OperateLog 标注集 ③差集=缺审计项 → 逐项补 @OperateLog（既有切面直接挂，不改切面本身）④比对输出贴报告
2. **前后值口径**：sys_operate_log 记录操作人/时间/请求参数/响应（框架既有能力）——**不扩展 schema 加前后值列**；「前后值」的业务语义归 iot_data_item_history 时间线（0039 未交付）——分工核查结论登记：操作日志=谁调接口+何时+成败，时间线=业务值变化（边界清晰不双写），0039 交付前时间线缺位为已知状态（登记不阻塞）
3. **凭据读写全量审计（ADR-0006）**：0019 交付面核查——凭据写入/轮换/吊销/查看明文四类操作是否全量 @OperateLog+审计行（grep+冒烟双证）；缺则补
4. **高风险范围界定**：机械清单以设计接口表为准（模板/实例/数据项/点位/通道/凭据/网关/指标/状态规则/发布回滚/调用方管理）——只读接口不审；内部令牌端点（/internal/v1/**、/openapi/v1/**）不挂管理面操作日志（各有自己的审计链——0051 403 全量）
5. **守卫测试**：全局口径扩展（照 0064 范式）：高风险写端点 ⊆ @OperateLog 标注集（机械断言防回归）
6. **不碰**：OperateLogAspect 核心、iot_data_item_history（0039）、sys_operate_log schema、前端

### 测试口径（TDD）

覆盖守卫测试扩展（写端点⊆标注集）；补接线端点的审计行断言（操作人/参数/响应留存）；凭据四类操作全量审计断言；既有测试零回归。

## 验收命令

环境事实（协调者 2026-11-23 实测）：WSL docker mysql/redis 在跑；sql-exec.sh 每条语句单独一次调用；8098 prod 启动回归用（探测/用毕即停）；TDengine 本批不涉。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. 覆盖比对脚本输出贴报告（写端点全量 vs @OperateLog 标注集——**零缺失**终态）
3. dev 冒烟：抽 3 个补接线端点真调 → sys_operate_log 行实查（操作人/参数/响应留存）+ 凭据查看明文审计行实查
4. prod 档真实启动回归 → 0 ERROR
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0065-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；不改范围外文件（OperateLogAspect 核心/iot_data_item_history 零触碰）；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0065）。
