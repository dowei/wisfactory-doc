# 实施 brief：TASK-0072（application-prod.yml 顶层 ot 重复键合并修复 · BUG-0001）

## 批次

- TASK-0072（本批共 1 个，BUG 修复微改，BUG 加严条款适用）
- BASE：`98c155e`（主仓库根 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/resources/application-prod.yml`（唯一改动文件）
  - 允许一个 fix commit（main），提交信息注明 TASK-0072 / BUG-0001

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0072.md`
- BUG 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\bugs\BUG-0001.md`
- 复现 TC：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\testcases\TC-0022.md`
- 原始发现证据：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0001-report.md` 的「验收 4」一节

## 修复方案（已裁决，照做）

- 删除 125 行的顶层 `ot:`，其下 redisson/ip/mail 子块**原样**并入 94 行的 `ot:` 块（只调结构不动键值语义）。
- SM4_KEY / CRYPTO_DATA_KEY / CRYPTO_INDEX_KEY 的 fail-fast 约定**不动**（prod 密钥只从环境变量取，不得给默认值）。
- 环境事实：MySQL 127.0.0.1:3307 root/root 库 ot_wisfactory（已初始化含 iot 菜单种子）；Redis localhost:6379 无密码；无 mysql CLI，SQL 走 `bash backend/scripts/sql-exec.sh`（用法见脚本头；多条语句每条一行行尾分号）。

## BUG 修复加严（两项证据，缺一不收）

1. **全仓同契约点扫描**：对 `backend/ot-admin/src/main/resources/application*.yml` 全部文件做 YAML 重复顶层键检测（python yaml 逐文件解析即可），扫描结果贴报告——证明除本缺陷外无同构问题。
2. **回归先红后绿**：
   - 红：修复前按 TC-0022 复现启动失败（`found duplicate key ot`），原始输出贴报告；
   - 绿：修复后同一命令启动成功，原始输出贴报告。

## 验收命令（整体验收）

1. `cd backend && mvn clean package -DskipTests` → BUILD SUCCESS
2. prod 启动（brief 原命令形态，密钥用 application.yml 里 dev 档默认值）：
   `DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root REDIS_HOST=localhost SM4_KEY=<dev默认值> CRYPTO_DATA_KEY=<dev默认值> CRYPTO_INDEX_KEY=<dev默认值> java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod`
   → 期望：日志含 `Started OtAdminApplication`，`grep -c "ERROR"` 启动段为 0
3. 登录（POST /auth/login，图形验证码走 Redis 键 `captcha:code:{uuid}` 预置 + SM4-ECB 加密密码 hex 再 base64——TASK-0001 报告验收 4 有完整可用口径）→ 200 拿 token
4. 带 token `GET /tenant/list` → 200 返回租户列表
5. dev 档回归（确认 prod 文件改动不影响 dev）：`mvn spring-boot:run -pl ot-admin` 启动 + `GET /tenant/list` 200（与 prod 启动串行，先停一个再起另一个，8098 单端口）

**端口纪律**：8098 全仓唯一后端端口，启动前先探测端口空闲（`netstat -ano | grep 8098` 或等效），被占则等待重试，不许强杀他人进程。

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0072-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修，禁止改期望凑输出；修复说明「为什么现在对了」。

## 收尾回写

- 不要回写 TASK 详情（勾选/修改内容/任务结果由协调者按报告收尾）。
