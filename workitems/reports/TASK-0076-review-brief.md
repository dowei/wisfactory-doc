# 评审 brief：TASK-0076 diff 评审

## 角色

- 代码评审（标准任务验收的 diff 评审环节；上报端点与令牌链是接入服务安全面，评审从严）

## 必读（只给路径，不粘正文）

- diff 包：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-diff.txt`（TASK-0076 单提交 f0f4533，14 文件约 2570 行，全落 ingest/**；协调者越界反查：全部落在 brief 声明范围）
- 实施 brief（三件契约：上报端点/认证回调/令牌链 fail-closed/TDengine 写客户端全限定名与超时/0076 实测教训固化）：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-brief.md`
- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0076.md`
- 实施报告：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-report.md`
- 契约权威：`resources\contract\unified-message-model.md` v1.1.0（§3.2）+ 0073 既有内核（复用不重写声明核对）
- 后端对接锚点：`backend/ot-admin/src/main/java/net/otlab/admin/module/iot/credential/controller/IotCredentialInternalController.java`（0019 令牌拉取响应形状）

## 依据

- 代码评审：`D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13
- 评审两样：①与 brief 契约一致性——重点：上报端点五字段组/批量上限 500/死信路由（0073 复用边界）、认证回调 allow/deny+主题范围、令牌链 fail-closed（后端不可达 deny）+缓存 TTL、写客户端全限定名/Basic 认证头/超时 30s/code 24 与 918 环境级识别 fail-soft、0073 内核零重写声明核对（pipeline/message/quality 包是否真复用）；②明显质量缺陷（错误处理、边界、并发安全——重点：HTTP 端点并发吞吐（pipeline 并发模型契约）、令牌缓存并发读写、超大 body 防线、JSON 解析错误处理、HMAC/令牌比对常量时间、context 超时传递）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0076-review.md`。发现清单，每条带：严重级（Critical/Important/Minor）/ 出处 / 一句话说明。

## 契约

- **fresh**：不共享协调者上下文；**no-subagents**；**只评审不修改**——所有产出进报告。

完成后，你的最终回复只需一句话：评审完成 + 发现分级汇总（Critical N / Important N / Minor N）。
