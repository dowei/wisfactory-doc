# 评审 brief：TASK-0052 diff 评审（代码评审）

> 实现提交 1494e1e（BASE=e492973，单提交，21 文件）——调用方管理/双密钥轮换/吊销立即生效（openapi 管理面）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0052-diff.txt`（e492973..1494e1e，21 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0052-brief.md`（DDL/管理接口四组/轮换协议/吊销/权限码种子/测试口径）
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0052-report.md`（含预案 A 裁决落地与 yml 偏离登记）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0052.md`
- 关联：workitems 库 `features\FEAT-0017.md`、`requirements\REQ-0035.md`、`testcases\TC-0017.md`、`decisions\ADR-0006.md`
- 上游代码（0051 地基，需要时直接读）：backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**（auth/audit/config）+ resources/sql/upgrade/20261122_iot_openapi_domain.sql（双列 secret 差异登记原文）

## 越界反查（协调者已比对，评审者复核）

- 21 文件全落 backend/ot-admin + resources/sql（声明范围内）；yml 的 rotation-window-days 一键为 brief 条目 7 明令但改动范围清单未点名——执行者已登记偏离，评审者核实为唯一 yml 触点且属 brief 明令项（追认评估）。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **明文一次性纪律**：开通/轮换响应含明文 secret；全 diff 扫描确认列表/详情/日志/@OperateLog response 列/审计行**无明文无密文泄漏**（执行者称审计无明文测试为真实切面织入——核实）
2. **轮换协议**：当前值移入 prev 三列+新值入主列的原子性（单条 UPDATE 原子换装——中间态评估）；验签顺序扩展（当前→prev→过窗惰性清理）与 0051 HmacAuthStrategy 原验签逻辑的兼容（不破坏既有单密钥验签路径）；rotated_at 缺失 fail-closed 视同过窗
3. **并行窗**：7 天常量 ot.openapi.rotation-window-days（yml 唯一新键）；过窗惰性清理的触发点与并发安全（多请求同时触发清理）
4. **吊销立即 403**：预案 A（每请求查库无缓存）——核实 0051 filter 现状确实无缓存（若评审者发现实际有缓存则本项失效需升级）；无 TTL 空窗实证口径
5. **管理面权限**：openapi:client:list/add/edit/remove 四码 + 菜单种子 4075-4078 幂等+超管授权；管理面与 0051 鉴权面（filter）的权限边界不混淆
6. **4002 唯一性**：appKey 撞键映射 4002 口径（0015 M-2 范式）；scope JSON 校验（空=默认拒绝透传）
7. **SQL**：加列幂等范式（information_schema+PREPARE）；test_client 种子零改动；4075-4078 挂点合理性
8. **ResultCode 零改动**声明核实（4002 既有复用）
9. 0051 既有测试零回归（2366 全量内；auth/filter 既有用例不受验签顺序扩展影响——单密钥路径）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0052-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处（文件:行）、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
