# TASK-0090 执行报告（BUG-0007 微改修复）

- BASE：8660ee0d1e394681c3e280cb4dd1c99de0b16860
- 提交哈希：abad464e2e4c34f17eae62654fa756005462b169
- 执行时间：2026-11-25
- 状态：代码已提交；TASK 保持原状态（未置「已完成」）

## 修复内容

1. `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/scheduler/AlarmTimeoutScanner.java`
   - 构造器第三参数改为 `@Value("${ot.alarm.data-timeout-scan-seconds:300}") int timeoutSeconds`
   - 补充 `import org.springframework.beans.factory.annotation.Value`
   - 默认值 300 不变，扫描行为不变。
2. 新增 `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/alarm/scheduler/AlarmTimeoutScannerContextTest.java`
   - 使用 `ApplicationContextRunner` 仅注册 `AlarmTimeoutScanner` + mock 依赖（`AlarmRecordService` / `JdbcTemplate`，其中 JdbcTemplate 以 `calcReadJdbcTemplate` bean 名注册以匹配 `@Qualifier`）
   - 断言 context 启动无失败、`getBean(AlarmTimeoutScanner.class)` 成功
   - 旧版（无 `@Value`）上下文中 `int` 参数无 bean 可注入 → 断言必红；修后绿
   - 未使用 `@SpringBootTest`，避免全量上下文的外部依赖

## 验收 1：单元测试（原样贴回输出）

命令：`cd backend && mvn -pl ot-admin test`

```
[INFO] Results:
[INFO] 
[INFO] Tests run: 2614, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:05 min
[INFO] Finished at: 2026-11-25T17:20:28+08:00
[INFO] ------------------------------------------------------------------------
```

其中新装配测试单独确认（surefire 报告）：

```
Test set: net.otlab.admin.module.iot.alarm.scheduler.AlarmTimeoutScannerContextTest
-------------------------------------------------------------------------------
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.354 s -- in net.otlab.admin.module.iot.alarm.scheduler.AlarmTimeoutScannerContextTest
```

共 2614 = 既有 2613 零回归 + 新装配测试 1。

## 验收 2：启动实测（原样贴回输出）

命令：`cd backend && mvn spring-boot:run -pl ot-admin`（dev profile，默认 active）

启动日志关键行（`/tmp/task0090-springboot.log`）：

```
2026-11-25 17:22:10.475 [main] INFO  org.springframework.boot.tomcat.TomcatWebServer - Tomcat started on port 8098 (http) with context path '/'
2026-11-25 17:22:10.487 [main] INFO  net.otlab.admin.OtAdminApplication - Started OtAdminApplication in 12.166 seconds (process running for 12.727)
```

curl 实测：

```
$ curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8098/
200
```

HTTP 状态码：**200**（非 000）。

后端进程已保留运行：PID 128268（java），端口 8098 处于 LISTENING，可供界面测试。

## 验收 3：范围核对（原样贴回输出）

命令：`git diff --name-only 8660ee0d..HEAD`

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/alarm/scheduler/AlarmTimeoutScanner.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/alarm/scheduler/AlarmTimeoutScannerContextTest.java
```

仅落 2 个授权文件，无额外改动；未 git add -A，未 push。

## 结论

修复完成：BUG-0007 启动阻断已消除（8098 实测 HTTP 200），上下文装配回归测试纳入即可防止旧缺陷复发；2613 既有测试零回归；改动范围与契约一致。TASK 状态未变更，待调用方按流程验收/收尾。
