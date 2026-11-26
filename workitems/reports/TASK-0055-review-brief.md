# 评审 brief：TASK-0055 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0055-diff.txt`（BASE 98c155e 之上的暂存 diff，16 文件 630 行；已由协调者完成越界反查：全部落在 brief 声明范围，routers.ts +3 行属「仅追加 iot 路由段」）
- 实施 brief（含接口契约与裁决）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0055-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0055.md`
- 实施报告（验收输出，供交叉印证，不替代你自己的 diff 判断）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0055-report.md`

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` 的「diff 评审」口径（§13）
- 评审两样：①与 TASK 验收标准 / brief 接口契约的一致性（菜单号段 4008-4016 连续、menu_type=2 无按钮行、module_id=4、幂等口径、privilege 指令绑定 iot:tenant:*、权限由后端下发前端只做显隐、无越权前端拦截逻辑）；②明显质量缺陷（错误处理、边界、安全——如 XSS/注入面、按钮权限位漏绑、API 封装错误处理与现有 api/system 范式一致性、占位页可维护性）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0055-review.md`。发现清单，每条带：

- 严重级（Critical / Important / Minor）
- 出处（哪个文件哪一条）
- 一句话说明

## 契约

- **fresh**：不共享协调者上下文——只依据「必读」与「依据」点到的东西判断，不向协调者追问；判断不了的标「无法判定」。
- **no-subagents**：你不得再派 subagent。
- **只评审不修改**：不改任何文件，所有产出进报告。
