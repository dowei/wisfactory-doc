# TASK-0006 修订轮 scoped 复评报告（fresh reviewer）

- 复评对象：`reports/TASK-0006-fixdiff.txt`（commit 1065f9a，2 文件 70 增 2 删，已完整读入）
- 范围：只判定 I-2 是否修复、修复有无新缺陷；I-1 已裁决为残余风险（TD-0055）不在判定范围；原 6 条 Minor 已 defer 不重开（除非修复使其恶化）
- 交叉核实：仓库 HEAD=1065f9a 与 diff 包逐行一致（`git show --stat` 恰 2 文件）；`IotModelInstanceServiceImpl` 现文件核实 createInstance(L142)/updateInstance(L301)/deleteInstances(L386) 均为 `@Transactional(rollbackFor = Exception.class)`，evictAll 位于 L211/L354/L448；dataperm 域零改动属实

## 结论：I-2 已修复，新发现 0（Critical 0 / Important 0 / Minor 0）

## 判定理由（一句）

createInstance/deleteInstances 各补一行 `evictAll`（均 @Transactional 内 → afterCommit 仅真实提交后发布，写失败/拒绝/回滚路径经测试断言不发布），新建绑部门实例与删除实例提交即全量失效（dev 冒烟 R2/R4 SCAN 为空、R3 重建 key 已含新实例 id=17），缓存与实例状态的 10 分钟错位窗消除，I-2 判定标准达成；修复无新缺陷（详见下）。

## 逐点核验

| 判定点 | 结论 |
| --- | --- |
| 创建绑部门实例即时失效 | ✓ createInstance 尾部（实例行 insert + generateForInstance 同事务全部完成后）evictAll，afterCommit 发布；冒烟 R2：POST /iot/instance → SCAN=[]，R3 重建 key instanceIds 含 id=17 |
| 删除实例即时失效 | ✓ deleteInstances `removed && evictAll`（无行删除不变更状态则不发布，正确收敛），afterCommit；冒烟 R4：DELETE → 落库 deleted=1，SCAN=[] |
| 失效时机/事务边界 | ✓ 两方法 @Transactional（现文件逐个核实），发布语义沿 0003 afterCommit 范式；回滚（含 generateForInstance 整批失败）不发布，无幻影失效；发布点在事务内注册、提交后执行，挂点行本身不产生部分提交 |
| 失败路径不发布 | ✓ 红绿测试：撞码 4002 / 删除被拒（存在数据项）均断言 `never() evictAll`，与实现一致 |
| 粒度 | ✓ evictAll 全量（非 evictUser），测试同时断言 `never() evictUser` 锁定粒度；授权扩展（仅一处 → create/update/delete 三处）经协调者裁决，updateInstance 原挂点保留 |
| 测试断言无放松 | ✓ 4 个新增用例全为加法断言（成功发布 times(1) / 失败不发布），未放松或删除任何既有断言；红转绿记录可信——红 2 失败行号 1254/1284 与 diff 新增文件中两处 `verify(times(1)).evictAll()` 的实际行号逐行核算完全吻合 |
| 账目与范围 | ✓ 1537 = 1533 + 4 正确；改动恰 2 文件（instance impl + 其测试），均在授权范围内，dataperm 域未触碰；报告 §6 已修正原 §3.2/§5.2 指路错误 |
| 原 Minor 是否恶化 | 无——M-4（同步 SCAN+DEL 尾延迟）仅随授权挂点同型扩展两处，单次成本与键量级评估不变，性质未恶化；其余 5 条未涉及 |

## 新发现

无。
