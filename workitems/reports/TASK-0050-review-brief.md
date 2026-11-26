# 评审 brief：TASK-0050 diff 评审（代码评审）

> 实现提交 e49fea8（BASE=331f860，单提交 15 文件 backend/**）——状态展示读口（数据服务统一状态查询，解锁前端 0059/0058 批二）。

## 角色

- 代码评审（口径：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0050-diff.txt`（331f860..e49fea8，15 文件）
- 实施 brief：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0050-brief.md`
- 执行者报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0050-report.md`（含动态翻译实证/TDengine 真机读回未验真登记）
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0050.md`
- 决策：FEAT-0016/REQ-0007/TC-0016/ADR-0004（展示端不现算）/ADR-0009（稳定序号）
- 上游：reports\TASK-0049-report.md（状态写入形态：Redis iot:st + st_item_value full_key 规范）+ TASK-0048-report.md（iot_status_dict/rule 翻译源）+ TASK-0051-report.md（对外 API+6001-6099 错误段）
- 需要上下文时读工作区源码（D:\workspace\WisFactory\backend\...）

## 越界反查（协调者已比对，评审者复核）

- 15 文件全落 backend（iot/statusread 新模块 7 + openapi 4 + 测试 4）；区间仅一提交（0042 calc 改动未提交、未混入）；calc/frontend/ingest/契约零触碰。

## 评审重点（除 §13 常规质量缺陷外，按 brief 逐条核对）

1. **不现算纪律（ADR-0004）**：current 读口是否真直读 Redis `iot:st:{tenant}:{instance}`（0049 写入形态），无任何状态重新求值逻辑；无状态→UNKNOWN/seq=null/quality=3 两面一致（管理面+对外）
2. **动态翻译**：statusCode→name/color **只从 iot_status_dict + iot_status_rule 取**（0049 F-06 硬编码反例规避）——核实无硬编码字典散落；翻译随字典改名改色即时跟随（live 实证）的机制正确（无缓存或缓存失效口径明确）
3. **历史时间线 SQL**：queryStatusHistory 的 full_key=`{instance}.STATUS` 精确过滤（**无 `full:` 前缀**——0049 F-02 教训）、时间范围/ORDER BY ts ASC、库名形态；假 taosAdapter 精确断言是否真锁 SQL 形态；**TDengine REST 故障下真机读回未验真——评估契约测试证据充分性**（与 0049 同口径：代码路径+SQL 形态_tests即可收尾，真机列为环境恢复条件）
4. **对外 scope/错误段**：对外 status 查询 scope 外 6003、参数错 400（复用 0051 既有码无新增）——与 0051 六接口错误段口径一致；appKey+HMAC 鉴权链不改
5. **数据权限**：管理面 current/history 走 dept 实例裁剪（数据权限拦截）——核实无越权读取他人实例状态
6. **测试有效性**：2423 测试 0F（2403 基线+20）；live 10/10（动态翻译改名改色/UNKNOWN 两面/scope 6003/参数 400/历史空集）与断言对应；TDD 红转绿（编译红→构造不匹配红→绿）留痕真实
7. **零回归与范围**：0051/0048 既有测试零回归；commit 仅本任务 15 文件（并发 0042 calc 未触碰实证）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0050-review.md`。发现清单，每条带：严重级（Critical / Important / Minor）、出处、一句话说明。若整体无 Critical/Important，明确给出「通过」结论。**特别对第 3 点（TDengine 真机读回未验真的证据充分性）给出明确判定**。

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与工作区源码判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
