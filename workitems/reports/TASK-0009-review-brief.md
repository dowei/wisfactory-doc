# 评审 brief：TASK-0009 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-diff.txt`（TASK-0009 单提交 e30de0a，11 文件约 1450 行，已隔离同期 Go 道提交 2fc80ba；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（含导入契约、回执分类、整批原子性、maxLimit 顺带项）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0009.md`
- 实施报告（含根因链 R1-R8 与环境事故恢复记录，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-report.md`
- 前批范式：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/service/impl/IotDataDictServiceImpl.java` 既有部分（4002 双道闸/白名单/值域 @Pattern——本 diff 的增量方法须与其同口径）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` 的「diff 评审」口径（§13）
- 评审两样：①与 brief 契约的一致性——重点：整批原子性（任一失败 0 INSERT）、回执五分类与行号物理对齐、批内互撞先于落库、库内预检一次 IN（含已禁用行占码语义）、4002/400 口径区分、maxLimit 仅一行且全量回归、菜单 4023 与守卫闭环、Excel 解析对 SysUserController 范式的对齐；②明显质量缺陷（错误处理、边界、安全——重点：上传文件的大小/类型防线、Excel 解析的内存放大面（超大文件/恶意构造）、模板下载与错误清单的响应流处理、@RepeatSubmit 的语义、导入并发的 DuplicateKey 兜底口径、multipart 配置缺省是否够 5000 行文件）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0009-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条）
- 一句话说明

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
