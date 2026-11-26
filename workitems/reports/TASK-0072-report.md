# TASK-0072 实施报告（application-prod.yml 顶层 `ot` 重复键合并修复 · BUG-0001）

- 执行者：implementer（subagent，无 subagent 派发）
- BASE：`98c155e`（主仓库根 HEAD，开工时 `git rev-parse HEAD` 核实一致）
- 时间：2026-11-21 12:48 开工，14:35 完成
- 复现/回归口径：TC-0022（修复前 prod 启动必败，修复后必须成功，先红后绿）

## 0. 结论概览

**全部验收通过**：五条验收命令（打包 / prod 启动 / 登录 / GET /tenant/list / dev 回归）全部达标；两项 BUG 加严证据（全仓 yml 扫描、先红后绿）齐备并附原始输出。修复提交 `27268a0`（main，单文件 +22/-23）。

## 1. 改动文件清单

| 文件 | 改动 |
| --- | --- |
| `backend/ot-admin/src/main/resources/application-prod.yml` | 唯一改动文件：删除 125 行顶层 `ot:`，其 redisson/ip/mail 子块原样并入 94 行 `ot:` 块；+22/-23 行（净 -1 行 = 消失的重复顶层键） |

- git 提交：`27268a0 fix(config): application-prod.yml 顶层 ot 重复键合并，redisson/ip/mail 并入唯一 ot 块（TASK-0072 / BUG-0001）`（main，单文件，见 §6）
- SM4_KEY / CRYPTO_DATA_KEY / CRYPTO_INDEX_KEY 的 fail-fast 约定未动：`sm4.key` 仍为 `${SM4_KEY}`，`ot.crypto.*` 仍为 `${CRYPTO_DATA_KEY}` / `${CRYPTO_INDEX_KEY}`，无默认值（§5 语义等价比对可证）。
- 注释处理说明：随子块移动的 3 组注释原样保留；其中「客户端IP解析」两条注释原挂在第二个 `ot:` 之上、内容实为描述 `ot.ip.behind-proxy`，随 `ip:` 子块并入块内并统一为 2 空格缩进（纯注释排版，键值未动）。

## 2. BUG 加严证据 1：全仓同契约点扫描（YAML 重复键检测）

方法：Python 3.8.6 + PyYAML 5.3.1，`yaml.compose` 逐文件解析，递归检测**所有层级**重复键并区分顶层——比 brief 要求的「顶层键检测」更严（嵌套层重复键同样会令 snakeyaml 严格模式抛错，一并扫描）。

范围：`find D:/workspace/WisFactory -name "application*.yml" -not -path "*/target/*"` 全仓仅 3 个源码文件（`ot-admin/src/main/resources/` 下 application.yml / application-dev.yml / application-prod.yml）；`target/classes/` 下 3 个为 Maven 打包副本（构建产物随源文件同步），不属源码契约点；另对打包 jar 内解出的 `BOOT-INF/classes/application-prod.yml` 做了独立复核（修复前 `ot`×2，修复后 `ot`×1，见 §3.1/§3.3）。

**修复前（红前基线）扫描原始输出：**

```
application-dev.yml | OK
    top-level keys: ['spring', 'wechat', 'druid', 'ot', 'logging']
    nested-level duplicates: 0
application-prod.yml | TOP-LEVEL-DUP
    top-level keys: ['mybatis-plus', 'spring', 'wechat', 'sm4', 'ot', 'springdoc', 'knife4j', 'druid', 'ot']
    [TOP-LEVEL] dup key='ot' first@line94 duplicate@line125 col1
    nested-level duplicates: 0
application.yml | OK
    top-level keys: ['server', 'spring', 'sm4', 'ot', 'spring.servlet.multipart', 'mybatis-plus', 'sa-token', 'knife4j', 'management', 'project']
    nested-level duplicates: 0
```

**修复后扫描原始输出：**

```
application-dev.yml | OK
    top-level keys: ['spring', 'wechat', 'druid', 'ot', 'logging']
    nested-level duplicates: 0
application-prod.yml | OK
    top-level keys: ['mybatis-plus', 'spring', 'wechat', 'sm4', 'ot', 'springdoc', 'knife4j', 'druid']
    nested-level duplicates: 0
application.yml | OK
    top-level keys: ['server', 'spring', 'sm4', 'ot', 'spring.servlet.multipart', 'mybatis-plus', 'sa-token', 'knife4j', 'management', 'project']
    nested-level duplicates: 0
```

结论：除本缺陷（application-prod.yml 顶层 `ot` 94/125 重复）外，全仓无同构问题（含全部嵌套层级）。application.yml 顶层同时存在 `spring` 与字面键 `spring.servlet.multipart` 属不同键名（YAML 合法、Spring Boot 支持点号键），非重复键。

## 3. BUG 加严证据 2：回归先红后绿（TC-0022）

### 3.1 红（修复前复现启动失败）

打包（修复前）：`cd backend && mvn clean package -DskipTests` → BUILD SUCCESS（37.681 s，2026-11-21T12:53:49）；并从打包 jar 解出 `BOOT-INF/classes/application-prod.yml` 复核：顶层键 `['mybatis-plus', 'spring', 'wechat', 'sm4', 'ot', 'springdoc', 'knife4j', 'druid', 'ot']`，`ot` 出现 2 次——缺陷确已随 jar 打包。

按 TC-0022 / BUG-0001 复现命令（brief 原命令形态，密钥取 application.yml dev 档默认值）：

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

**原始输出（完整，EXIT_CODE=1，共 61 行）：**

```
12:54:01.812 [main] ERROR org.springframework.boot.SpringApplication -- Application run failed
while constructing a mapping
 in 'reader', line 1, column 1:
    mybatis-plus:
    ^
found duplicate key ot
 in 'reader', line 125, column 1:
    ot:
    ^

	at org.yaml.snakeyaml.constructor.SafeConstructor.processDuplicateKeys(SafeConstructor.java:119)
	at org.yaml.snakeyaml.constructor.SafeConstructor.flattenMapping(SafeConstructor.java:71)
	at org.yaml.snakeyaml.constructor.SafeConstructor.flattenMapping(SafeConstructor.java:66)
	at org.yaml.snakeyaml.constructor.SafeConstructor.constructMapping2ndStep(SafeConstructor.java:204)
	at org.yaml.snakeyaml.constructor.BaseConstructor.constructMapping(BaseConstructor.java:553)
	at org.yaml.snakeyaml.constructor.SafeConstructor$ConstructYamlMap.construct(SafeConstructor.java:592)
	at org.yaml.snakeyaml.constructor.BaseConstructor.constructObjectNoCheck(BaseConstructor.java:265)
	at org.yaml.snakeyaml.constructor.BaseConstructor.constructObject(BaseConstructor.java:248)
	at org.springframework.boot.env.OriginTrackedYamlLoader$OriginTrackingConstructor.constructObject(OriginTrackedYamlLoader.java:120)
	at org.yaml.snakeyaml.constructor.BaseConstructor.constructDocument(BaseConstructor.java:202)
	at org.yaml.snakeyaml.constructor.BaseConstructor.getData(BaseConstructor.java:167)
	at org.springframework.boot.env.OriginTrackedYamlLoader$OriginTrackingConstructor.getData(OriginTrackedYamlLoader.java:100)
	at org.yaml.snakeyaml.Yaml$1.next(Yaml.java:530)
	at org.springframework.beans.factory.config.YamlProcessor.process(YamlProcessor.java:203)
	at org.springframework.beans.factory.config.YamlProcessor.process(YamlProcessor.java:169)
	at org.springframework.boot.env.OriginTrackedYamlLoader.load(OriginTrackedYamlLoader.java:85)
	at org.springframework.boot.env.YamlPropertySourceLoader.load(YamlPropertySourceLoader.java:50)
	at org.springframework.boot.env.PropertySourceLoader.load(PropertySourceLoader.java:71)
	at org.springframework.boot.context.config.StandardConfigDataLoader.load(StandardConfigDataLoader.java:55)
	at org.springframework.boot.context.config.StandardConfigDataLoader.load(StandardConfigDataLoader.java:36)
	at org.springframework.boot.context.config.ConfigDataLoaders.load(ConfigDataLoaders.java:100)
	at org.springframework.boot.context.config.ConfigDataImporter.load(ConfigDataImporter.java:133)
	at org.springframework.boot.context.config.ConfigDataImporter.resolveAndLoad(ConfigDataImporter.java:88)
	at org.springframework.boot.context.config.ConfigDataEnvironmentContributors.withProcessedImports(ConfigDataEnvironmentContributors.java:130)
	at org.springframework.boot.context.config.ConfigDataEnvironment.processWithProfiles(ConfigDataEnvironment.java:326)
	at org.springframework.boot.context.config.ConfigDataEnvironment.processAndApply(ConfigDataEnvironment.java:245)
	at org.springframework.boot.context.config.ConfigDataEnvironmentPostProcessor.postProcessEnvironment(ConfigDataEnvironmentPostProcessor.java:97)
	at org.springframework.boot.context.config.ConfigDataEnvironmentPostProcessor.postProcessEnvironment(ConfigDataEnvironment.java:82)
	at org.springframework.boot.support.EnvironmentPostProcessorApplicationListener.onApplicationEnvironmentPreparedEvent(EnvironmentPostProcessorApplicationListener.java:137)
	at org.springframework.boot.support.EnvironmentPostProcessorApplicationListener.onApplicationEvent(EnvironmentPostProcessorApplicationListener.java:120)
	at org.springframework.context.event.SimpleApplicationEventMulticaster.doInvokeListener(SimpleApplicationEventMulticaster.java:180)
	at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:133)
	at org.springframework.boot.context.event.EventPublishingRunListener.multicastInitialEvent(EventPublishingRunListener.java:137)
	at org.springframework.boot.context.event.EventPublishingRunListener.environmentPrepared(EventPublishingRunListener.java:82)
	at org.springframework.boot.SpringApplicationRunListeners.lambda$environmentPrepared$0(SpringApplicationRunListeners.java:123)
	at org.springframework.boot.SpringApplicationRunListeners.doWithListeners(SpringApplicationRunListeners.java:123)
	at org.springframework.boot.SpringApplicationRunListeners.doWithListeners(SpringApplicationRunListeners.java:117)
	at org.springframework.boot.SpringApplicationRunListeners.environmentPrepared(SpringApplicationRunListeners.java:65)
	at org.springframework.boot.SpringApplication.prepareEnvironment(SpringApplication.java:356)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:316)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1365)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1354)
	at net.otlab.admin.OtAdminApplication.main(OtAdminApplication.java:19)
	at java.base/jdk.internal.reflect.DirectMethodHandleAccessor.invoke(DirectMethodHandleAccessor.java:103)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at org.springframework.boot.loader.launch.Launcher.launch(Launcher.java:106)
	at org.springframework.boot.loader.launch.Launcher.launch(Launcher.java:64)
	at org.springframework.boot.loader.launch.JarLauncher.main(JarLauncher.java:40)
```

（留档：`C:\Users\dowil\AppData\Local\Temp\task0072\red-run.log`；进程退出码 1，失败发生在配置解析阶段，早于端口绑定。）

### 3.2 修复说明（为什么现在对了）

- 根因：顶层键 `ot:` 在同一 YAML mapping 中出现两次，Spring Boot 4 携带的 snakeyaml 严格模式（`SafeConstructor.processDuplicateKeys`）对任何层级重复键直接抛异常 → Application run failed，prod 档任何实例无法启动。
- 修复：删除 125 行顶层 `ot:`，其 redisson/ip/mail 子块原样并入 94 行 `ot:` 块（只调结构不动键值语义）；`sm4` 与 `ot.crypto.*` 的「仅环境变量、无默认值、缺失即 fail-fast」约定不动。
- 为什么现在对了：a) 修复后扫描（§2）证实 `ot` 顶层单键、全文件（含嵌套层）零重复键，snakeyaml 重复键检测无触发点；b) 语义等价比对（§5）证实合并前后键值集合完全一致（73 个扁平键零差异），Spring 绑定路径（ot.redisson.*、ot.ip.*、ot.mail.*、ot.crypto.*、ot.desensitize.*）不变；c) 绿验（§3.3/§4-2）同命令启动成功且启动段 ERROR=0。

### 3.3 绿（修复后同一命令启动成功）

同命令重跑（修复后 jar，2026-11-21 14:31:53 启动）：

```
cd backend && DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

**原始输出（完整，共 55 行，grep -c "ERROR" = 0，EXIT_CODE=0 为本人主动 taskkill 收尾前的正常运行）：**

```

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v4.1.0)

2026-11-21T14:31:53.013+08:00  INFO 48108 --- [ot-framework] [           main] net.otlab.admin.OtAdminApplication       : Starting OtAdminApplication using Java 21.0.12 with PID 48108 (D:\workspace\WisFactory\backend\ot-admin\target\ot-admin-1.0.0.jar started by dowil in D:\workspace\WisFactory\backend)
2026-11-21T14:31:53.023+08:00  INFO 48108 --- [ot-framework] [           main] net.otlab.admin.OtAdminApplication       : The following 1 profile is active: "prod"
2026-11-21T14:31:57.190+08:00  INFO 48108 --- [ot-framework] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Multiple Spring Data modules found, entering strict repository configuration mode
2026-11-21T14:31:57.209+08:00  INFO 48108 --- [ot-framework] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data Redis repositories in DEFAULT mode.
2026-11-21T14:31:57.354+08:00  INFO 48108 --- [ot-framework] [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 67 ms. Found 0 Redis repository interfaces.
2026-11-21T14:32:01.346+08:00  INFO 48108 --- [ot-framework] [           main] o.s.boot.tomcat.TomcatWebServer          : Tomcat initialized with port 8098 (http)
2026-11-21T14:32:01.399+08:00  INFO 48108 --- [ot-framework] [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2026-11-21T14:32:01.400+08:00  INFO 48108 --- [ot-framework] [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/11.0.22]
2026-11-21T14:32:01.499+08:00  INFO 48108 --- [ot-framework] [           main] b.w.c.s.WebApplicationContextInitializer : Root WebApplicationContext: initialization completed in 8282 ms
2026-11-21T14:32:01.598+08:00  INFO 48108 --- [ot-framework] [           main] n.o.c.c.d.DynamicDataSourceConfig        : Default datasource: master
2026-11-21T14:32:01.602+08:00  INFO 48108 --- [ot-framework] [           main] n.o.c.c.d.DynamicDataSourceConfig        : Datasource names: master
2026-11-21T14:32:01.715+08:00  INFO 48108 --- [ot-framework] [           main] n.o.c.c.d.DynamicDataSourceConfig        : Creating datasource [master] - url: jdbc:mysql://localhost:3307/ot_wisfactory?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8&allowPublicKeyRetrieval=true, username: root
2026-11-21T14:32:01.781+08:00  INFO 48108 --- [ot-framework] [           main] n.o.c.c.d.DynamicDataSourceConfig        : Created datasource: master
2026-11-21T14:32:02.005+08:00  WARN 48108 --- [ot-framework] [           main] c.b.m.s.MybatisSqlSessionFactoryBean     : Property 'mapperLocations' was specified but matching resources are not found.
 _ _   |_  _ _|_. ___ _ |    _ 
| | |\/|_)(_| | |_\  |_)||_|_\ 
     /               |         
                        null 
2026-11-21T14:32:02.923+08:00  WARN 48108 --- [ot-framework] [           main] c.b.m.core.metadata.TableInfoHelper      : Can not find table primary key in Class: "net.otlab.admin.module.system.entity.SysUserRole".
2026-11-21T14:32:02.924+08:00  WARN 48108 --- [ot-framework] [           main] c.b.m.core.injector.DefaultSqlInjector   : class net.otlab.admin.module.system.entity.SysUserRole Not found @TableId annotation, Cannot use Mybatis-Plus 'xxById' Method.
2026-11-21T14:32:02.995+08:00  WARN 48108 --- [ot-framework] [           main] c.b.m.core.metadata.TableInfoHelper      : Can not find table primary key in Class: "net.otlab.admin.module.system.entity.SysRoleMenu".
2026-11-21T14:32:02.996+08:00  WARN 48108 --- [ot-framework] [           main] c.b.m.core.injector.DefaultSqlInjector   : class net.otlab.admin.module.system.entity.SysRoleMenu Not found @TableId annotation, Cannot use Mybatis-Plus 'xxById' Method.
2026-11-21T14:32:03.548+08:00  WARN 48108 --- [ot-framework] [           main] c.b.m.core.metadata.TableInfoHelper      : Can not find table primary key in Class: "net.otlab.admin.module.system.entity.SysRoleDept".
2026-11-21T14:32:03.549+08:00  WARN 48108 --- [ot-framework] [           main] c.b.m.core.injector.DefaultSqlInjector   : class net.otlab.admin.module.system.entity.SysRoleDept Not found @TableId annotation, Cannot use Mybatis-Plus 'xxById' Method.
2026-11-21T14:32:06.955+08:00  INFO 48108 --- [ot-framework] [           main] net.otlab.core.config.RedisConfig        : Redis 隔离信息：database=0，key-prefix=''（跨系统隔离以 database 为准；前缀仅作用于本项目 RedisTemplate）
2026-11-21T14:32:08.867+08:00  INFO 48108 --- [ot-framework] [           main] n.o.a.m.wechat.config.WxMpServiceConfig  : 微信公众号模块已启用，appId: ，票据键前缀: ot_framework:wx:mp:
2026-11-21T14:32:09.249+08:00  INFO 48108 --- [ot-framework] [           main] org.redisson.Version                     : Redisson 3.50.0
2026-11-21T14:32:10.029+08:00  INFO 48108 --- [ot-framework] [isson-netty-4-2] o.redisson.connection.ConnectionsHolder  : 1 connections initialized for localhost/127.0.0.1:6379
2026-11-21T14:32:10.029+08:00  INFO 48108 --- [ot-framework] [isson-netty-4-1] o.redisson.connection.ConnectionsHolder  : 1 connections initialized for localhost/127.0.0.1:6379
2026-11-21T14:32:15.055+08:00  INFO 48108 --- [ot-framework] [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2026-11-21T14:32:15.784+08:00  INFO 48108 --- [ot-framework] [           main] n.o.a.m.s.s.impl.SysJobServiceImpl       : 定时任务初始化完成，已注册 0 个任务
____ ____    ___ ____ _  _ ____ _  _ 
[__  |__| __  |  |  | |_/  |___ |\ | 
___] |  |     |  |__| | \_ |___ | \| 
https://sa-token.cc (v1.45.0)
2026-11-21T14:32:19.162+08:00  INFO 48108 --- [ot-framework] [           main] b.m.s.MybatisPlusApplicationContextAware : Register ApplicationContext instances org.springframework.boot.web.server.servlet.context.AnnotationConfigServletWebServerApplicationContext@30ea8c23
2026-11-21T14:32:20.204+08:00  INFO 48108 --- [ot-framework] [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 4 endpoints beneath base path '/actuator'
2026-11-21T14:32:20.491+08:00  INFO 48108 --- [ot-framework] [           main] o.s.boot.tomcat.TomcatWebServer          : Tomcat started on port 8098 (http) with context path '/'
2026-11-21T14:32:20.522+08:00  INFO 48108 --- [ot-framework] [           main] net.otlab.admin.OtAdminApplication       : Started OtAdminApplication in 29.599 seconds (process running for 31.825)
2026-11-21T14:32:20.600+08:00  INFO 48108 --- [ot-framework] [           main] n.o.a.m.s.r.DataCryptoMigrateRunner      : 敏感数据存量迁移检查：无待迁移明文行
2026-11-21T14:32:30.736+08:00  INFO 48108 --- [ot-framework] [nio-8098-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2026-11-21T14:32:30.737+08:00  INFO 48108 --- [ot-framework] [nio-8098-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2026-11-21T14:32:30.739+08:00  INFO 48108 --- [ot-framework] [nio-8098-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 2 ms
2026-11-21T14:32:33.956+08:00  WARN 48108 --- [ot-framework] [nio-8098-exec-2] n.o.c.exception.GlobalExceptionHandler   : 登录认证异常：未按照指定前缀提交 token，prefix=Bearer
```

- `grep -c "ERROR"` 启动段与全程均为 **0**；末行 WARN 是本人首次鉴权请求未带 `Bearer` 前缀所致（客户端口径错误，见偏离 3），非启动问题。
- 打包 jar 内配置独立复核：解出 `BOOT-INF/classes/application-prod.yml` 顶层键 `['mybatis-plus', 'spring', 'wechat', 'sm4', 'ot', 'springdoc', 'knife4j', 'druid']`，`ot` 出现 1 次。
- 实例为本任务所起（PID 48108），验收完成后已 taskkill 停止并确认 8098 释放（无孤儿进程）。

## 4. 整体验收命令原始输出（五条）

### 4-1. 打包

`cd backend && mvn clean package -DskipTests`（修复后，2026-11-21T12:56:25）：

```
[INFO] Reactor Summary for otlab Framework 1.0.0:
[INFO] 
[INFO] otlab Framework .................................... SUCCESS [  0.218 s]
[INFO] OT Core ............................................ SUCCESS [ 13.152 s]
[INFO] OT Admin ........................................... SUCCESS [ 22.581 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  36.382 s
[INFO] Finished at: 2026-11-21T12:56:25+08:00
```

### 4-2. prod 启动

命令：brief 原命令形态（见 §3.3）。结果：`Started OtAdminApplication in 29.599 seconds`，`Tomcat started on port 8098`，`netstat` 确认 8098 LISTENING（PID 48108），`grep -c "ERROR"` = **0**。原始输出全文见 §3.3。

### 4-3. 登录（POST /auth/login，图形验证码 + SM4 密文全流程）

验证码经 Redis 键预置（prod 档 `key-prefix=''`，物理键即 `captcha:code:{uuid}`；值按应用侧 JacksonJsonRedisSerializer 的 JSON 字符串格式写入，如 `"AB12"`——TASK-0001 报告曾实测非 JSON 格式值会被 Jackson 反序列化拒绝）；密码为 SM4-ECB(admin123, `otlab_260829____`) 密文 hex 后 base64（`NmRhM2Q3NDU1OTU4Mjk3ZTc0M2ZkNTkxYWY2YmMwNWY=`）。环境无 hutool / 可用 SM4 库（pycryptodome 3.9.8 无 SM4），采用纯 Python SM4-ECB 实现，先以 GB/T 32907-2016 标准测试向量自校验（加密/解密均 PASS）再生成密文：

```
SELF-TEST encrypt: PASS (cipher=681edf34d206965e86b3e94f536e4246)
SELF-TEST decrypt: PASS
cipher hex : 6da3d7455958297e743fd591af6bc05f
login password field (b64 of hex): NmRhM2Q3NDU1OTU4Mjk3ZTc0M2ZkNTkxYWY2YmMwNWY=
roundtrip: PASS
```

登录原始输出：

```
captcha preset: key=captcha:code:task0072-d60b9a5db138 value="AB12"
=== POST /auth/login
{"code":200,"data":{"nickname":"管理员","token":"f42a5d30-e78e-494c-8068-a784562d8210","tokenTimeout":86399,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795242753753}
HTTP_STATUS:200
TOKEN:f42a5d30-e78e-494c-8068-a784562d8210
```

### 4-4. 带 token GET /tenant/list

注意 sa-token 配置 `token-prefix: Bearer`（application.yml line 161，前端 `frontend/src/utils/file-url.ts` 同口径 `Authorization: 'Bearer ' + token`）。首次请求漏带前缀被拒（业务码 401，属客户端口径错误，见偏离 3），按 `Authorization: Bearer <token>` 重试：

```
curl -s -m 10 -H "Authorization: Bearer f42a5d30-e78e-494c-8068-a784562d8210" http://localhost:8098/tenant/list

{"code":200,"data":{"current":1,"pages":1,"records":[{"id":1,"tenantCode":"default","tenantName":"默认租户","status":1,"pointQuota":null,"storageQuotaGb":null,"expireTime":null,"contactName":null,"contactPhone":null,"remark":null,"createBy":null,"createTime":"2026-11-21 12:04:13","deleted":0,"updateBy":null,"updateTime":"2026-11-21 12:04:13"}],"size":10,"total":1},"message":"操作成功","timestamp":1795242770290}
HTTP_STATUS:200
```

### 4-5. dev 档回归（确认 prod 文件改动不影响 dev；与 prod 启动串行，8098 单端口）

prod 实例停止、8098 释放确认后，`cd backend && mvn spring-boot:run -pl ot-admin`（后台）启动；就绪判据 `GET /auth/captchaImage` HTTP 200：

```
READY after ~35s (captchaImage HTTP 200)
```

dev 启动日志关键行（`dev-run.log` 共 192 行）：

```
2026-11-21 14:33:33.745 [main] INFO  net.otlab.admin.OtAdminApplication - The following 1 profile is active: "dev"
2026-11-21 14:33:55.685 [main] INFO  org.springframework.boot.tomcat.TomcatWebServer - Tomcat started on port 8098 (http) with context path '/'
2026-11-21 14:33:55.742 [main] INFO  net.otlab.admin.OtAdminApplication - Started OtAdminApplication in 23.041 seconds (process running for 23.906)
```

`grep -c "ERROR"`（全程）= **0**。dev 档免验证码测试登录（LoginController 的 `/auth/login/test`，prod 档拒绝、dev 专用）+ `GET /tenant/list`：

```
=== POST /auth/login/test
{"code":200,"data":{"nickname":"管理员","token":"307d8eab-7712-45f4-a922-5b7f78107e48","tokenTimeout":86400,"userId":1,"username":"admin"},"message":"操作成功","timestamp":1795242864422}
HTTP_STATUS:200
=== GET /tenant/list (Authorization header)
{"code":200,"data":{"current":1,"pages":1,"records":[{"id":1,"tenantCode":"default","tenantName":"默认租户","status":1,"pointQuota":null,"storageQuotaGb":null,"expireTime":null,"contactName":null,"contactPhone":null,"remark":null,"createBy":null,"createTime":"2026-11-21 12:04:13","deleted":0,"updateBy":null,"updateTime":"2026-11-21 12:04:13"}],"size":10,"total":1},"message":"操作成功","timestamp":1795242864782}
HTTP_STATUS:200
```

dev 实例为本任务所起（进程树 sh.exe 18664 → mvn java 80332 → forked java 48916），验收完成后 `taskkill //T` 整树停止，三个 PID 确认消失、8098 释放（无孤儿进程）。

## 5. 语义等价比对（合并前后键值零差异）

方法：python yaml 解析；修复前文件取「重复键=并集」语义（两块子键 crypto/desensitize 与 ip/mail/redisson **不相交**，并集良定义），展平为 73 个键值对与修复后文件逐一比对：

```
BEFORE ot 块1 子键: ['crypto', 'desensitize']
BEFORE ot 块2 子键: ['ip', 'mail', 'redisson']
子键重叠: NONE
before(union) keys: 73, after keys: 73
keys only in BEFORE: NONE
keys only in AFTER : NONE
VALUE DIFFS: NONE
RESULT: SEMANTICALLY-EQUAL
ot 子键(after): ['crypto', 'desensitize', 'ip', 'mail', 'redisson']
```

附注：若用 PyYAML 默认 load 读修复前文件（重复键静默 last-wins），会丢掉第一块的 3 个键（ot.crypto.*、ot.desensitize.enabled）只剩 70 键——这正是重复键的另一重危害：即便解析器宽容也会静默丢配置。

## 6. git 提交

- 提交：`27268a0 fix(config): application-prod.yml 顶层 ot 重复键合并，redisson/ip/mail 并入唯一 ot 块（TASK-0072 / BUG-0001）`（main 分支，父提交 `98c155e`，单文件 `backend/ot-admin/src/main/resources/application-prod.yml`，+22/-23）。
- 提交事故与纠正（见偏离 4）：首次 commit 误将并发会话已暂存的 16 个无关文件一并带入，立即 `git reset --soft HEAD~1` 撤销，改用 pathspec 只提交本任务文件；并发会话的暂存状态原样恢复（其 16 个文件仍处 staged，未提交、未推送、未改动内容）。

## 7. 未做项与原因

无——brief 五条验收命令与两项 BUG 加严证据全部执行并通过；无越界文件改动。

## 8. 偏离计划之处

1. **终止 TASK-0001 会话遗留 dev 实例（12:58，进程处置，无仓库文件改动）。** 开工时 8098 被占：PID 41612（`mvn spring-boot:run` 父 Maven JVM PID 48756 fork 的 OtAdminApplication）均 12:32:52-58 启动，与 TASK-0001 报告末次 dev 回归步骤吻合，属上一任务会话收尾后未停止的遗留孤儿进程（无连接、无活动）。等待重试 45s 不释放后，经 AskUserQuestion 未获回复，按最佳判断 taskkill；**协调者已追认该处置有效**。
2. **绿验首跑（13:05:27）失败：端口竞争，非修复缺陷。** 我释放端口后另一会话 13:05 启动 dev 实例（PID 61032）抢占 8098，我的实例在 YAML 解析、数据源、Redis、Redisson 等全部初始化完成后于 Tomcat 绑定阶段失败（`Web server failed to start. Port 8098 was already in use.`）。该实例长期零连接、零活动；**经协调者确认其为 TASK-0055 执行者完成验收后遗留的孤儿进程，并已由协调者结束释放**。此后按原命令重跑绿验成功（§3.3）。
3. **登录/鉴权请求首次未按 sa-token `token-prefix: Bearer` 口径提交 token**，`GET /tenant/list` 返回业务码 401（未登录或登录已失效）；核对 application.yml（line 161 `token-prefix: Bearer`）与前端 `file-url.ts` 口径后改用 `Authorization: Bearer <token>` 即 200。属客户端请求格式错误，非应用缺陷；两次原始输出均已贴 §4-4 上下文。
4. **git 提交事故（已纠正）。** 首次 commit 因并发会话预先 stage 的 16 个文件已在 index 中而被一并提交（17 files changed）；立即 `git reset --soft HEAD~1` 撤销（该提交从未推送、从历史消失），改用 `git commit -- <路径>` 仅提交本任务文件，并发会话暂存状态原样恢复。最终提交 `27268a0` 仅 1 文件。
5. **SM4 加密工具替换。** 环境无 hutool，pycryptodome 3.9.8 不含 SM4；改用纯 Python SM4-ECB 实现，先过 GB/T 32907-2016 官方测试向量自校验（PASS）再生成登录密文，口径与前端/后端 hutool SM4-ECB/PKCS5 一致（hex 后 base64）。
6. **报告起草节奏。** 端口阻塞等待期间先行落盘已完成部分的证据章节（§1/§2/§3.1/§3.2/§4-1/§5），绿验与 dev 回归完成后补录其余章节；不影响证据内容。

## 9. 遗留问题

1. **8098 单端口与多会话并行执行冲突（本次两度发生）。** 遗留实例（TASK-0001、TASK-0055 各一例）与并发会话实例均曾占用端口阻塞验收。建议：a) 协调者排程时串行化需要 8098 的任务；b) 执行类 brief/skill 固化「收尾停掉自己起的实例」要求（本次协调者已对本人明确该要求并已执行）。
2. **并发会话工作树遗留状态**：TASK-0055/0056 相关的 16 个文件（frontend iot 页面/路由 + iot 控制台菜单 SQL）仍处 staged 未提交，本人未动，归其任务收尾处理。
3. **重复键防护缺位**：本次缺陷能长期潜伏（先于 BASE 35d22f6 存在）说明 CI/构建链无 YAML 重复键守卫。本报告 §2 的 python 扫描脚本可作为守卫原型，建议后续固化为构建期检查（防回归），归 TASK/工程化排期。
4. 本次冒烟数据副作用：Redis 中两条验证码键（`captcha:code:task0072-*`，5 分钟 TTL 自动过期）与 prod/dev 各一条 admin 登录 token 记录（Sa-Token 86400s TTL 自然过期），无业务数据变更。

---
（本报告由 implementer 生成；TASK 状态、勾选与任务结果由协调者按报告收尾。原始日志留档：`C:\Users\dowil\AppData\Local\Temp\task0072\`（red-run.log / green-run2.log / dev-run.log / prod-prefix.yml）。）
