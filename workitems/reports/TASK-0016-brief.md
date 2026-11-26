# 实施 brief：TASK-0016（实例版本锁定与待升级清单——后端接口组）

> 0070 的「待升级清单页」前端在等本任务的 API；锁定两列（applied_template_version/locked）已由 0012 建列（p31 无此两列差异已登记），快照按实例有效版本组装已由 0024 兑现——本任务是 ADR-0008 落地件的最后一环。

## 批次

- TASK-0016（本批共 1 个，标准任务 3 人天；backend 串行道）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/template/**`（实例锁定开关/待升级清单/确认升级接口）
  - `backend/ot-admin/src/test/java/**`（对应测试）
  - **无新 SQL**（两列已在位，版本历史表 0015 已建——如实现中发现缺表/列，停下写报告）
  - 不碰 publish 域（快照组装已按 applied_template_version，不动）、前端、ingest、ot-core、yml

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0016.md`
- 决策权威：`decisions\ADR-0008.md`（锁定/待升级/逐台迁移三段语义）
- 关联：`features\FEAT-0005.md`、`requirements\REQ-0006.md`、`testcases\TC-0005.md`
- 上游交付（开工先读）：`reports\TASK-0015-report.md`（版本历史/分档引擎/复用面：IotTemplateVersionService + TemplateContentDiffEngine + tier 表）+ `reports\TASK-0024-report.md`（快照按 applied_template_version 组装——锁定语义的下游兑现）+ 0012 的实例域数据权限范式
- 上游代码（开工先读）：iot/template/**（IotModelInstance.appliedTemplateVersion/locked、IotModelTemplate.version、IotTemplateVersionService、TemplateContentDiffEngine、TemplateTierRuleTable）+ 0012 实例树接口的权限裁剪写法

## 接口与决策（逐条结论，不再讨论）

1. **锁定开关**：`PUT /iot/instance/lock`（body：instanceId + locked）——复用既有 `iot:instance:edit` 权限，不新增种子；locked=1 时实例编辑仅允许 status/remark（0012 既有口径，接口层校验确认）；数据权限照 0012 实例域范式（部门裁剪）。
2. **待升级清单接口**：`GET /iot/template/upgrade/list?templateId=`（复用 `iot:template:query`）——行=引用该模板且 appliedTemplateVersion < 模板当前 version 的实例：{instanceId, instanceName, appliedTemplateVersion, currentVersion, locked, tierSummary, diff}；diff 用 0015 的 TemplateContentDiffEngine 对「实例 applied 版本 content vs 当前版本 content」生成（逐条目带档位——T2 段=锁定实例待确认项，T3 段=逐台迁移项）；分页照 0056 范式；数据权限裁剪照实例域范式。
3. **逐台/批量确认升级**：`POST /iot/template/upgrade/confirm`（body：instanceIds[] + targetVersion）——校验 targetVersion == 模板当前 version（不允许任意目标版）；逐台 UPDATE applied_template_version（乐观锁，冲突返回 4002 口径照 0015 修订轮范式）；**确认动作逐台写操作日志**（框架 @OperateLog 范式，0079/0080 同款）；locked 实例同样可确认（这正是锁定实例的升级通道）；确认后实例按新版本进下次发布快照（0024 组装天然兑现，本任务不改 publish）。
4. **权限**：全接口复用既有 iot:instance:edit / iot:template:query，不新增菜单种子。
5. **边界**：无版本差异的实例不出现在清单；模板回退（current < applied，理论上不发生——版本单调）防御性跳过并 WARN；确认请求含无效 instanceId → 400 且整批拒绝（不半成功）。
6. **不碰**：publish 域、前端（0070 批二接本 API）、ingest、ot-core。

### 测试口径（TDD）

锁定开关（置位/解除/locked=1 编辑受限）；清单（applied<current 筛选/无差异不出现/diff 复用 0015 引擎逐条目档位/分页）；确认（单台/批量/applied 更新+operate log 留痕/乐观锁冲突 4002/无效 id 整批拒绝/locked 实例确认通过）；快照组装不受影响（0024 既有测试回归——锁定实例未确认前保持旧口径）；数据权限裁剪。红转绿留痕。

## 验收命令

环境事实（协调者 2026-11-23 实测）：WSL docker mysql/redis 在跑；sql-exec.sh 每条语句单独一次调用；8098 prod 启动回归用（探测/用毕即停）；TDengine 本批不涉；WSL2 保活。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（Results 段贴回）
2. dev 冒烟（每步原始输出贴回）：模板 T2 变更升版 → 清单出该模板实例行（applied<current，diff 带 T2 条目）→ 锁一台 → 锁定实例保持旧口径（预览 affected/locked 分列照 0015 冒烟）→ 单台确认 → applied=当前版 → 批量确认余下 → 清单清空 → operate log 留痕核对
3. prod 档真实启动回归 → 0 ERROR
4. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落声明内（无 SQL/yml/ot-core 改动）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0016-report.md`。五项齐全 + 一句话结论。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）。
- TDD 红转绿，每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0016）。
