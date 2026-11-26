# 实施 brief：TASK-0051（对外查询六接口与鉴权——签名+scope+IP 白名单）

## 批次

- TASK-0051（本批共 1 个，标准任务 5 人天；backend 串行道——配置链收官环 + REQ-0035 对外面）
- BASE：以开工时主仓 HEAD 为准（记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/**`（数据服务对外模块新包）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataperm/**`（仅 ClientScope 消费接线——0011 组件已有中台 scope 解析三态，若需微调在此声明）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_openapi_domain.sql`（新增）
  - `backend/ot-admin/src/main/resources/application.yml`（仅允许新增 `ot.openapi.*` 键族）
  - 不碰 frontend/、ingest/、管理面既有域（只读消费）

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0051.md`
- 关联对象：同目录 `features\FEAT-0017.md`、`requirements\REQ-0027.md`（六接口全量+限值契约）、`requirements\REQ-0035.md`（对外鉴权细节）、`testcases\TC-0017.md`
- 决策：`decisions\ADR-0013.md`（A7 appKey 级可选 IP 白名单）；`ADR-0016.md`（裁决 5 中台 scope JSON：{"instanceIds":[...],"subtree":true}——null=无权限默认拒绝）；`ADR-0009.md`（TD-0031 六接口全量；鉴权策略接口预留 OAuth2 位）
- 接口权威：详细设计 V1.1 §3.5 p26（六接口行与响应形状——pypdf 自行提取）；错误码分段 §3.4 p20（6001 签名错误/6002 限流/6003 scope 外——HTTP 403 与业务码 6003 对应：传输层 403、业务码 6003）
- 表结构权威：附录 A p34 iot_api_client/iot_api_call_log DDL（brief 已提取贴下）
- 组件：`iot/dataperm/**`（0011 ClientScope 解析三态+0015 impact 前例——scope 裁剪消费）
- 曲线/历史数据源：**TDengine 192.168.31.145 fd 问题未修复**——历史/实时查询走 fail-soft 空集占位（0023 曲线范式；Redis 实时值本期 Redis 无采集数据→空集——联调归 0068），**本期交付接口契约与鉴权/scope/审计全链，数据空集如实**
- 模型树数据源：iot_model_instance（0012 交付——models/tree 数据权限按 scope 裁剪）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_openapi_domain.sql，p34 逐字 + 幂等 + 头注）

`iot_api_client` + `iot_api_call_log` 两表 p34 逐字（uk_app_key/idx_tenant/idx_app_time）。**调用方开通（最小方式）**：本期无管理界面（0052 交付 CRUD）——脚本内 INSERT 一个测试调用方（app_key=test_client，secret_hash=SHA256(test_secret)——**测试专用**，prod 环境由运营方替换；头注声明）；调用方管理界面归 0052。

### 代码结构（/openapi 段）

1. **鉴权过滤器**（/openapi/v1/** 全覆盖）：
   - appKey+HMAC-SHA256 签名：header `X-App-Key` + `X-Signature` + `X-Timestamp`（防重放 5 分钟窗口——签名串=appKey+timestamp+请求体摘要（SHA256 body），密钥=服务端存的 secret（比对 hash(输入 secret) 与 secret_hash？——裁决：**secret_hash 存 SHA256(明文 secret)**，签名用明文 secret 做 HMAC——服务端无法反查明文，校验时 HMAC(明文？) 不可得——**修正口径：secret 明文可逆存储不可行（不存明文），签名校验改为「调用方持 secret 计算 HMAC(secret, appKey+timestamp+bodyHash)」，服务端用**存储的 secret 本身**？secret_hash 只能验证「提交的 secret 是否正确」——**最终裁决：iot_api_client 增存 `secret_encrypted`（SM4 加密明文，框架 sm4.key）替代/并列 secret_hash（「与上游差异」登记：上游只有 secret_hash 无法支持 HMAC 验签——HMAC 需要服务端持有原文），签名校验=解密得 secret 后重算 HMAC 比对；secret_hash 保留作快速校验（SHA256(提交 secret) 比对通过才走 SM4 解密）**。脚本两列都建（与上游差异登记）。
   - 防重放：timestamp 与服务端时差 >300s → 6001；同 appKey+timestamp+nonce？一期只 timestamp 窗口（nonce 幂等表登记 0052/后续）。
   - appKey 级可选 IP 白名单（ADR-0013 A7）：iot_api_client 增 `ip_whitelist varchar(500) DEFAULT NULL` 列（与上游差异登记——NULL=不限；CSV 形态）。
   - 错误码：签名错/过期 6001；限流 6002；scope 外 6003（HTTP 200 + body code=6003，传输层 403 仅认证失败形态——REQ-0027 口径「HTTP 403 与 6003 对应：传输层 403、业务码 6003」——**裁决：认证类（签名/令牌）=HTTP 403；授权类（scope 外）=HTTP 200+body 6003**；报告里明确映射表）。
   - 鉴权策略接口预留 OAuth2 位（ADR-0009——OpenApiAuthStrategy 接口，HMAC 实现为默认策略）。
2. **限流**：rate_limit 次/分钟（0 不限）——Redis INCR+EXPIRE 滑动/固定窗口固定 60s（简单固定窗）→ 超 6002。
3. **六接口**（/openapi/v1/**，全部过过滤器+scope 裁剪+审计写 iot_api_call_log——403 类全量记录（REQ-0027/§1.10））：
   - `GET /openapi/v1/models/tree`——scope 裁剪后的模型树（复用 0012 树逻辑+ClientScope resolve——scope 展开成可见实例集后同登录用户拼装）
   - `GET /openapi/v1/items/realtime`——实时值（Redis 实时值键本期无采集→空集；契约就位）
   - `GET /openapi/v1/items/history`——历史曲线（TDengine 全限定名 fail-soft 空集——0068 联调）
   - `GET /openapi/v1/metrics/query`——指标查询（FEAT-0013 指标计算未交付→空集占位，契约就位）
   - `GET /openapi/v1/points/history`——点位历史（同 items/history fail-soft）
   - `GET /openapi/v1/dict/sync`——**路由占位**（增量语义归 TASK-0054——返回全量字典空实现+登记）
   - 限值契约（REQ-0027）：分页 pageSize 上限 500、时间跨度上限（历史接口 max 90 天——超 400）——按 §3.5 实际行提取对齐。
4. **审计**：iot_api_call_log 异步写（含 403/6003 类全量记录——参数摘要脱敏）。
5. 数据权限：scope 裁剪走 dataperm 组件 ClientScope（0011 三态：null=无权限默认拒绝；subtree 展开；实例集）——**组件消费，不重写**。

### 测试口径（TDD）

鉴权（签名正确/错签 403/过期 6001/限流 6002/IP 白名单外 403/禁用调用方 403）、scope（null 拒/subtree 展开/精确/混合——红线式断言 scope 外 6003+审计行全量记录）、六接口契约（响应形状/空集如实/限值 400）、审计异步落库断言。
冒烟（dev）：脚本测试调用方 → 正确签名调 models/tree → scope 外实例 → 6003 → 篡改签名 → 403 → 限流触发 → 6002 → 审计表行数核对。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线以开工实测为准 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对两表 + 测试调用方行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + /openapi/v1/dict/sync 200（占位响应）
5. 改动范围核对：`git diff --name-only <开工 BASE>..HEAD` 全落在声明内

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0051-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0051）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
