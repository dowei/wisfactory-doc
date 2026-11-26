# 实施 brief：TASK-0019（iot_channel_credential 凭据接口与 SM4 加密）

## 批次

- TASK-0019（本批共 1 个，标准任务 3 人天；backend 串行道——凭据是安全敏感面，评审将加严）
- BASE：`fa12df9`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/credential/**`（凭据域新包）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/auth/**`（InternalAuthChecker 完整化——见裁决 5，最小改动）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_credential_domain.sql`（新增）
  - 不碰 frontend/、ingest/、通道域（只读 channel 校验归属）、配置快照链路

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0019.md`
- 关联对象：同目录 `features\FEAT-0006.md`、`requirements\REQ-0011.md`（凭据全生命周期：双权限/掩码/轮换/吊销立即生效/拉取令牌）、`testcases\TC-0006.md`
- 决策：`decisions\ADR-0006.md`（双权限先例 edit/view 分权；凭据不进快照与 config.changed）；接口权威 §3.1 续 p23（四管理接口 + /internal/v1/credential/pull——brief 已转述要点，细字段自行提取 p23）
- 表结构权威：附录 A 凭据表 DDL（pypdf 自行提取——iot_channel_credential，约 p33）
- 既有设施：框架 SM4 工具（sm4.key 环境变量，grep ot-core `Sm4|SM4` 自行定位工具类与用法）；0017 的 InternalAuthChecker（gateway/auth 包）与 /internal/v1 单端点收窄模式；sys_operate_log 审计与 PermissionDeniedAuditListener（0002 交付）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_credential_domain.sql，逐字 + 幂等 + 头注）

`iot_channel_credential`：附录 A 逐字（pypdf 提取；预期含 tenant_id/channel_id/cred_type/credential_cipher({v1} 前缀 SM4 密文)/key_version/status 等——以 PDF 实际为准）。菜单种子 **4051 起**连续（4050 已占）：凭据管理按钮行（iot:credential:edit/view + iot:channel:list 已在 4046-4050 外需复核——凭据列表按 §3.1 续用 iot:channel:list，不重复登记；只登记 edit/view 两行）挂 4013 通道管理页下，幂等 + 超管授权。

### 管理四接口（/iot/channel/credential 段，语义照 §3.1 续 p23）

1. `GET /iot/channel/credential/list?channelId=`（iot:channel:list）——**只返回类型、掩码、key_version 与状态**，永不回密文/明文（掩码格式自定如 `abcd****wxyz` 前四后四）。
2. `POST /iot/channel/credential`（iot:credential:edit）——写入/轮换：明文只走 HTTPS 入参，服务端 SM4 加密落库（`{v1}:<cipher>` 前缀格式）；key_version 递增；同 channel 同 cred_type 重复写入=轮换（更新密文与版本），不新增行。@OperateLog 必加。
3. `GET /iot/channel/credential/view/{id}`（iot:credential:view）——查看明文：**单设权限**（无 view 权限的角色不可见入口+直调 403）；SM4 解密返回明文；@OperateLog 必加（审计：谁看了凭据）。
4. `DELETE /iot/channel/credential/{id}`（iot:credential:edit）——吊销：status 置禁用**立即生效**（拉取接口随即拒）；@OperateLog。
- SM4 工具复用框架（ot-core `Sm4Util` 或等价——grep 定位）；加密失败 fail-closed 拒保存。
- channel_id 归属校验（存在/同租户）；cred_type 枚举照 §3.1（如 username-password/secret 等——以 PDF 与 REQ-0011 实际为准）。

### 内部拉取接口

`GET /internal/v1/credential/pull?channelId=&credType=`（**无 Sa-Token**，照 0017 内部模式）：InternalAuthChecker 静态令牌校验（0017 占位实现**完整化接线**——本任务范围内把 /internal/v1/** 的认证器统一到 credential 域可复用形态，最小改动）+ channelId/credType 归属校验（通道存在/同租户/凭据状态启用）→ 返回密文与 key_version（**调用方内存解密**——服务端不解密传输）；凭据已吊销/不存在 → 403；**不进配置快照与 config.changed 载荷**（ADR-0006——快照链路 0024 交付时排除项，本期登记口径）。网关设备令牌链（校验令牌归属通道）归 0020/0027 接线——本期 InternalAuthChecker 静态令牌即准入，登记遗留。

### 审计与快照无明文（验收硬项）

- 写入/轮换/查看/吊销四动作全部 @OperateLog；**日志与快照无明文**：测试断言 sys_operate_log 的请求参数/返回值字段不含明文与密文（掩码或占位符）——grep 级验证写进单测。
- 无 edit 权限写入被拒**留审计**：PermissionDeniedAuditListener 既有链路断言（0002 交付）。

### 测试口径（TDD）

- 加解密往返（SM4 工具）、{v1} 前缀格式、轮换 key_version 递增、列表掩码（永不回明文/密文断言）、view 双权限（无 view → 403 + 审计留痕）、吊销后拉取 403、拉取归属校验、内部令牌错拒、审计无明文 grep 断言。
- 冒烟（dev）：admin 写凭据 → list 见掩码不见明文 → view 解密展示（operate_log 留痕 DB 核对）→ 无 view 权限测试用户直调 → 403 + 审计 → 吊销 → 拉取 403 → 内部拉取带正确令牌 → 密文+key_version 返回。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号；SM4_KEY=otlab_260829____ 为 dev 默认密钥）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1673 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对表结构 + 4051-4052 两行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + credential/list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0019-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0019）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
