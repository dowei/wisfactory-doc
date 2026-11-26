# 评审 brief：TASK-0042 diff 评审（代码评审）

> 实现提交 e387c3b（BASE=e49fea8，单提交 24 文件纯 calc/**）——DELTA/INTEGRAL 原生算子与同构存储（解锁 0045 指标聚合→0060 指标看板）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-diff.txt`（e49fea8..e387c3b 限 calc/，24 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-report.md`（含 live 发现 null-metadata NPE 修复 + 真机补验条件登记）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0042.md`
- 决策：FEAT-0013/REQ-0020/REQ-0019/TC-0013/ADR-0010（INTEGRAL 不开放手写）
- 上游：reports\TASK-0041-report.md（执行器/白名单/ResultSink）+ TASK-0049-report.md（st_item_value full_key+六元组 tag 教训）+ calc/src/** 现有代码
- 权威：详细设计 §1.6 + CONV-02 质量码 + st_item_value 契约 + config-snapshot-schema.md:165（full_key 规范）

## 越界反查（协调者已比对，评审者复核）

- 24 文件全落 `calc/**`；区间内 backend 提交 e49fea8（0050）已隔离；backend/frontend/ingest/契约零触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **DELTA 五异常语义**：正常差分/回退打码 code=8/换表 PARAM 分段/溢出归零 code=1/跨周期标记 code=9——与详细设计 §1.6 与 CONV-02 逐条一致；回退不写负值；换表后基线重建正确；质量码不冲突
2. **INTEGRAL 三场景**：真实间隔梯形积分（按相邻样本 ts 而非固定周期）/缺口不积分（quality 非 0/样本缺失间隔跳过）/**停机零增量 vs 断线区分**（两者语义不同、不混为一谈）；INTEGRAL 不开放手写（ADR-0010）——白名单/接口层实证
3. **同构写纪律**：st_item_value + Redis 点位缓存；**full_key={instance}.{data_code} 无 `full:` 前缀**（0049 F-02 教训）；六元组 tag 逐值精确断言（非仅 SQL 形态）；逐语句 REST 下发（0029 多语句信封静默空操作纪律）；status 留痕（0049）与算子结果写（本任务）并存不冲突
4. **派生项换向**：能源计量模板派生（source_type=2）只改绑定方向即换向（正负号随绑定）——契约测试锁定；与 0081 三方合并覆盖口径一致
5. **失败语义**：算子异常不写假 0（CONV-02 打码）、单节点失败隔离（0041 纪律）、TDengine 写失败 WARN 不阻塞计算；live 发现的 null-metadata NPE 修复是否充分（根因与回归测试）
6. **opt-in 接线**：runtime 默认关闭保既有行为（0041/0049 零回归）；开启后与 0041 executor/0049 status 的协同（同批数据既触发算子又触发状态判定是否重复写/顺序）
7. **测试有效性**：194 测试 0F（163 基线+31 新增）；22/22 live 断言与DELTA五/INTEGRAL三/派生换向/tag SQL 对应；红转绿留痕（含 NPE 修复）；**TDengine 真机写入+读回未验真（REST 故障）——契约测试+假适配器捕获 SQL 的证据充分性评估**（与 0049 同口径，列环境恢复后条件不阻断）
8. **零回归与范围**：0041/0049 既有测试零回归；commit 仅 24 文件 calc/**（并行 0050 backend 未触碰）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0042-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。**特别对第 7 点（TDengine 真机未验真的证据充分性）给出明确判定**。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
