# 评审 brief：TASK-0022 + TASK-0023 微改攒批 diff 评审（两任务联合）

## 角色

- 代码评审（微改攒批的统一 diff 评审——两任务（963dc71=bfdc11b 两提交）联合评审，分节出发现）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-0023-diff.txt`（BASE a961fd1 → bfdc11b，两提交 23 文件约 3900 行；协调者越界反查：全部落在 brief 声明范围 point 域 + redline 测试增补授权例外）
- 实施 brief（两任务契约：导入配额联动不豁免/幂等/10000 行上限/反查/批量启停防线/曲线 fail-soft/发布联动明示）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-0023-brief.md`
- 两 TASK 详情：`tasks\TASK-0022.md`、`tasks\TASK-0023.md`
- 实施报告（分节）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-0023-report.md`
- 范式基准：0018 字典导入（schema 校验/回执/整批拒绝）、0021 点位九接口（裁剪/配额/双道闸）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样（分节）：①TASK-0022——导入校验五项（编码合规/字典 unit/通道存在/min<=max/枚举）、配额联动不豁免（校验通过行数+当前计数>quota 整批拒）、幂等口径（活跃行 DUPLICATE 拒）、10000 行上限先于解析（0013 SAX 教训）、错误清单/导出裁剪同条件；②TASK-0023——反查联表数据权限裁剪（仅可见实例的数据项）、批量启停逐条防线整批 403、停用影响计数、发布联动明示（保存≠生效）、曲线 fail-soft 空集不报错、全限定名查询形态；③联合——明显质量缺陷（SAX 解析复用质量、批量 UPDATE 效率、事务边界、越权回归面）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0022-0023-review.md`。发现清单（按任务分节），每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N，按任务分节）。
