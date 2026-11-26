# 实施报告：TASK-0048（iot_status_rule 配置接口与校验 + 状态编码稳定字典）

- 日期：2026-11-22
- 执行者：implementer（subagent）
- BASE：`fb574cf3c3018c196c91f1df0f503c8e72914b4c`（开工时主仓 HEAD，即 TASK-0043 提交）
- 本任务提交：`d74d56f`（main，注明 TASK-0048）
- 结论先行：**五条验收全部通过**。全量回归 2036 = 开工基线实测 1979 + 本批净增 57，0 失败 0 错误 BUILD SUCCESS；升级脚本二次幂等两次 DONE + 两表结构与 brief/p35 逐字一致 + 菜单 4063-4069 七行核对一致；dev 冒烟全序列绿（字典只增 seq 1/2/3 + 重复 4002 + 模板建规则 + priority 冲突 4002 + 表达式校验矩阵 + 上限 10 条拒 + 直绑映射保存与三项校验 + 禁用编码一次性 + 引用已禁用编码拒 + seq 跳号 FAULT=4 + 启停承载 + 删除归档不复用）；prod 档真实启动 9.4s + 真实登录 200 + statusrule/list 200 全程 ERROR=0；本任务提交 22 文件全落声明范围。

---

## 1. 改动文件清单（22 文件，全部在 brief 声明范围内）

### 1.1 新增：升级脚本（1）

| 文件 | 说明 |
| --- | --- |
| `resources/sql/upgrade/20261122_iot_status_rule_domain.sql` | ① iot_status_rule（附录 A p35 逐字 + `binding_map json DEFAULT NULL` 补列，头注「与上游差异」登记）；② iot_status_dict（本任务定义，TD-0013 语义落表，头注登记「与上游差异，建议回写上游」）；③ sys_menu 4063-4069 七行（parent_id=4009，iot:statusrule:list/add/edit/remove + iot:statusdict:list/add/disable）幂等 INSERT IGNORE；④ sys_role_menu 授 role_id=1 七行；头注含用途/差异/接口对应/号段占用/幂等口径/业务纪律/sql-exec 注意 |

### 1.2 新增：主码（17，均在 `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/`）

| 文件 | 说明 |
| --- | --- |
| `entity/IotStatusRule.java` | 状态判定规则实体（p35 逐字 DDL + binding_map 补列注记；priority/template_id 创建后不可改口径） |
| `entity/IotStatusDict.java` | 状态编码稳定字典实体（TD-0013 只增不删、seq 稳定序号锚） |
| `mapper/IotStatusRuleMapper.java` | BaseMapper + `selectOccupiedByPriorities`（含已删行旁路预检，`@InterceptorIgnore(tenantLine)` 显式条件范式）+ `selectActiveCountByTemplate`（上限预检） |
| `mapper/IotStatusRulePriorityRow.java` | 优先级占用旁路行（id/priority/deleted——文案区分「已存在」/「已被删除记录占用，不可复用」） |
| `mapper/IotStatusDictMapper.java` | BaseMapper + `selectMaxSeq`（含禁用行——禁用不复用序号）+ `selectOccupiedByCodes`（含已删行旁路，防御性口径） |
| `mapper/IotStatusDictCodeRow.java` | 编码占用旁路行 |
| `dto/IotStatusRuleCreateForm.java` | 新增表单（templateId/priority/conditionExpr/statusCode/statusName/color/enabled） |
| `dto/IotStatusRuleUpdateForm.java` | 修改表单（白名单 conditionExpr/statusCode/statusName/color/enabled；**结构性无 templateId/priority**——调优先级=删旧建新） |
| `dto/IotStatusRuleBindingMapForm.java` | 直绑映射表单（ruleId+pointId+mappings[{enumValue,statusCode}]——ruleId 为 brief body 规格的必要补全，见偏离 2） |
| `dto/IotStatusDictCreateForm.java` | 字典新增表单（只增——无 status 字段，新增即启用；无编辑/删除表单） |
| `service/IotStatusRuleService.java` | 规则五端点服务接口（list/add/edit/binding-map/remove） |
| `service/IotStatusDictService.java` | 字典三端点服务接口（list/add/disable——无删除无编辑） |
| `service/impl/IotStatusRuleServiceImpl.java` | 规则服务实现：priority 双道闸 4002、上限 ≤10（400「状态规则上限 10 条」）、condition_expr 校验（Expander 同口径 + SELF 引用存在性 4001）、status_code 字典锚定（未登记 4001 提示先建状态编码）、启停由 edit 承载、直绑映射 JSON 幂等覆盖、写路径模板 resolve 防线 |
| `service/impl/IotStatusDictServiceImpl.java` | 字典服务实现：seq=max(seq)+1（含禁用行）、编码双道闸 4002、禁用一次性只置 status=0 不触碰 seq、无删除/编辑路径 |
| `service/support/ConditionExprValidator.java` | 条件表达式校验器（0013 RelativeRefExpander 同口径：RELATIVE_REF/BRACE_REF 同形正则、SELF/PARENT 前缀白名单、编码字符集、长度 ≤1000、括号配平、残留花括号终检；SELF/PARENT/绝对引用收集） |
| `controller/IotStatusRuleController.java` | `/iot/statusrule` 五端点（4063-4066 四码；binding-map 复用 iot:statusrule:edit） |
| `controller/IotStatusDictController.java` | `/iot/statusdict` 三端点（4067-4069 三码）——只增不删纪律的控制器面锁定 |

### 1.3 新增：测试（4，`backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusrule/`）

| 文件 | 用例数 | 说明 |
| --- | --- | --- |
| `service/impl/IotStatusRuleServiceImplTest.java` | 31 | 规则校验矩阵：新增主链/白名单落库断言、priority 活跃占用/已删占用/DuplicateKey 兜底 4002、上限 10 条拒、模板缺失拒、表达式畸形 token/括号不配平/超长 400、SELF 引用不存在 4001、PARENT/绝对引用不核存在性（0036 口径）、编码未登记/禁用 4001、enabled 值域、修改白名单（template_id/priority 结构性不进更新集）/启停/携带重校验、直绑映射成功 JSON 断言/规则缺失/点位缺失/枚举互撞/编码校验、列表随模板读（Wrapper 形态断言）、批量删除 fail-closed/模板防线 |
| `service/impl/IotStatusDictServiceImplTest.java` | 13 | 字典：seq=max+1（含禁用行跳号断言 FAULT=4）、空表首码 1、status 恒 1、活跃占用/已删占用/并发兜底 4002、字段校验、禁用只置 status 不触碰 seq、禁用幂等、禁用不存在 400、列表 Wrapper（seq 升序锚定 + 过滤） |
| `service/support/ConditionExprValidatorTest.java` | 7 | 语法矩阵：合法表达式收集（SELF/PARENT/绝对）、空白拒、超长拒、括号不配平拒、畸形 token 残留终检拒、关键字大小写不敏感、重复引用去重 |
| `config/IotStatusRuleMenuPermissionGuardTest.java` | 6 | 菜单守卫：SQL 侧 4063-4069 恰好七值映射（删行/换码/挪 parent 即红）、两控制器权限码并集 ⊆ 七值、规则控制器恰好四码、字典控制器恰好三码、字典端点数锁 3（无删除/编辑/恢复端点）、直绑映射锚点 iot:statusrule:edit |

---

## 2. 每条验收命令原始输出

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` —— 通过

```
$ cd backend && mvn install -pl ot-core -DskipTests
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] Total time:  1.663 s
[INFO] Finished at: 2026-11-22T18:03:47+08:00

$ mvn test -pl ot-admin（全量，终码）
[WARNING] Tests run: 2036, Failures: 0, Errors: 0, Skipped: 3
[INFO] BUILD SUCCESS
```

- 2036 = 开工基线实测 1979（TASK-0043 报告口径）+ 57（规则服务 31 + 字典服务 13 + 校验器 7 + 守卫 6）。
- 3 skipped 与基线同值（红线二-中台 scope 占位——TASK-0051 接线后启用）。
- 首轮全量前单测迭代 1 处：`IotStatusRuleServiceImplTest.testDelete_Success` 失败——根因：mock 了 `baseMapper.deleteById` 而 MP 3.5.17 `ServiceImpl.removeByIds` 委托 `baseMapper.deleteByIds(Collection)`（照 IotMetricServiceImplTest 既有口径核对确认）；修复为 mock/verify `deleteByIds(anyCollection())`。**为什么现在对了**：mock 与框架实际委托方法对齐，主码零改动。

### 2.2 验收 2：升级脚本二次幂等 + 两表结构 + 菜单七行 —— 通过

第一次执行：

```
$ bash scripts/sql-exec.sh ../resources/sql/upgrade/20261122_iot_status_rule_domain.sql
OK(0)
OK(0)
OK(0)
OK(7)
OK(7)
DONE
```

第二次执行（幂等复核）：

```
$ bash scripts/sql-exec.sh ../resources/sql/upgrade/20261122_iot_status_rule_domain.sql
OK(0)
OK(0)
OK(0)
OK(0)
OK(0)
DONE
```

两次均 DONE；第二次 0 行变更=不改已有行。逐条：SET NAMES + CREATE TABLE iot_status_rule + CREATE TABLE iot_status_dict + sys_menu 七行 + sys_role_menu 七行。

表结构核对（`SHOW CREATE TABLE`）：

```
iot_status_rule	CREATE TABLE `iot_status_rule` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '规则 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `template_id` bigint NOT NULL COMMENT '所属模板 ID',
  `priority` int NOT NULL COMMENT '优先级（小者优先）',
  `condition_expr` varchar(1000) NOT NULL COMMENT '条件表达式（相对引用）',
  `status_code` varchar(32) NOT NULL COMMENT '状态编码',
  `status_name` varchar(50) NOT NULL COMMENT '状态名称',
  `color` varchar(16) DEFAULT NULL COMMENT '展示颜色',
  `enabled` tinyint DEFAULT '1' COMMENT '是否启用 0 停用 1 启用',
  `binding_map` json DEFAULT NULL COMMENT '直绑映射（TD-0014 上游无列本任务新增：{pointId,mappings:[{enumValue,statusCode}]}）',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_template_priority` (`template_id`,`priority`),
  KEY `idx_tenant_template` (`tenant_id`,`template_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='状态判定规则表'

iot_status_dict	CREATE TABLE `iot_status_dict` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '状态字典 ID',
  `tenant_id` bigint NOT NULL COMMENT '租户 ID',
  `seq` int NOT NULL COMMENT '稳定序号（只增不复用——历史 st_item_value 序号不漂移的锚）',
  `status_code` varchar(32) NOT NULL COMMENT '状态编码',
  `status_name` varchar(50) NOT NULL COMMENT '状态名称',
  `color` varchar(16) DEFAULT NULL COMMENT '展示颜色',
  `status` tinyint DEFAULT '1' COMMENT '状态 0 禁用 1 启用（禁用不复用序号）',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `create_by` bigint DEFAULT NULL COMMENT '创建者 ID',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `update_by` bigint DEFAULT NULL COMMENT '修改者 ID',
  `deleted` tinyint DEFAULT '0' COMMENT '删除标识',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_status_code` (`tenant_id`,`status_code`),
  UNIQUE KEY `uk_tenant_seq` (`tenant_id`,`seq`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='状态编码稳定字典表（TD-0013）'
```

逐字核对：iot_status_rule 与附录 A p35 pypdf 提取逐字一致（字形规范化不改语义），binding_map 为 brief 裁决补列（头注登记）；iot_status_dict 与 brief 逐字 DDL 完全一致（`COLLATE=utf8mb4_0900_ai_ci` 为 MySQL 8 默认排序规则随库继承，与点位/指标域脚本同口径）。uk_template_priority/uk_tenant_status_code/uk_tenant_seq 均不含 deleted 列——删除归档不复用纪律的物理基础。

菜单与授权核对（应 7 行 / 7 行）：

```
$ SELECT id, menu_name, menu_type, perms FROM sys_menu WHERE id BETWEEN 4063 AND 4069;
id	menu_name	menu_type	perms
4063	状态规则查询	3	iot:statusrule:list
4064	状态规则新增	3	iot:statusrule:add
4065	状态规则修改	3	iot:statusrule:edit
4066	状态规则删除	3	iot:statusrule:remove
4067	状态字典查询	3	iot:statusdict:list
4068	状态字典新增	3	iot:statusdict:add
4069	状态字典禁用	3	iot:statusdict:disable
$ SELECT role_id, menu_id FROM sys_role_menu WHERE menu_id BETWEEN 4063 AND 4069;
role_id	menu_id
1	4063
1	4064
1	4065
1	4066
1	4067
1	4068
1	4069
```

### 2.3 验收 3：dev 冒烟（8098，dev 档）—— 通过

启动：`dev.ps1 start`（8098）→ 冒烟脚本 `backend/target/t0048_smoke.py` 原始输出（完整贴回；终轮 EXIT=0）：

```
=== S1) dev login（admin） === S1a -> HTTP 200 code 200 | 操作成功  [期望200] OK
=== S2) admin 建状态编码 RUNNING/STOPPED/ALARM（只增——seq 服务层 max+1） ===
S2-RUNNING -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2-STOPPED -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2-ALARM -> HTTP 200 code 200 | 操作成功  [期望200] OK
S2 断言: RUNNING=1 STOPPED=2 ALARM=3（seq 升序锚定） OK
=== S2b) status_code 重复拒（4002——uk_tenant_status_code 双道闸） ===
S2b -> HTTP 200 code 4002 | 状态编码已存在: RUNNING  [期望4002] OK
S2b 断言: 重复编码 4002 拒 OK
=== S3) 模板建规则（condition_expr 引用模板数据项 ELEC_METER/LOAD_RATE，priority=1） ===
S3a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S3 断言: 规则 id=21 落库（随模板读，priority 升序） OK
=== S4) priority 冲突拒（4002——uk_template_priority 物理 + 预检同款） ===
S4a -> HTTP 200 code 4002 | 该模板下优先级已存在: 1  [期望4002] OK
S4 断言: priority 冲突 4002 拒且未落库 OK
=== S5) condition_expr 校验矩阵（Expander 同口径 + 引用存在性） ===
S5a 畸形 token -> HTTP 200 code 400 | 条件表达式包含无法识别的引用 token（残留花括号）: 示例 { SELF.ELEC_METER } > 0  [期望400] OK
S5b 括号不配平 -> HTTP 200 code 400 | 条件表达式基础格式非法：括号/占位符不配平（表达式真语义校验归计算引擎，本任务仅做基础格式校验）  [期望400] OK
S5c 引用不存在 -> HTTP 200 code 4001 | 条件表达式引用的数据项不存在: SELF.NOT_THERE（模板 6）  [期望4001] OK
S5d 编码未登记 -> HTTP 200 code 4001 | 状态编码未登记，请先在状态字典中创建: NO_SUCH_CODE  [期望4001] OK
S5 断言: 畸形 token/括号不配平 400、引用不存在/编码未登记 4001，全部未落库 OK
=== S6) 规则条数上限：补至 10 条后第 11 条拒（400「状态规则上限 10 条」） ===
S6-p2..p10 -> HTTP 200 code 200 ×9 | 操作成功  [期望200] OK
S6-第11条 -> HTTP 200 code 400 | 状态规则上限 10 条（当前 10 条），新增拒绝  [期望400] OK
S6 断言: 10 条落库 + priority 升序；第 11 条 400「状态规则上限 10 条」拒且未落库 OK
=== S7) 直绑映射保存（TD-0014——binding_map 随规则行，幂等覆盖） ===
S7a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S7 断言: binding_map 落列（pointId=9001 + 枚举值→状态编码） OK
S7b 编码未登记 -> HTTP 200 code 4001 | 状态编码未登记，请先在状态字典中创建: NO_SUCH_CODE  [期望4001] OK
S7c 枚举值互撞 -> HTTP 200 code 400 | 直绑映射枚举值重复: 1  [期望400] OK
S7d 点位不存在 -> HTTP 200 code 4001 | 直绑映射的点位不存在或已删除: 999999  [期望4001] OK
S7 断言: 编码未登记 4001 / 枚举值互撞 400 / 点位不存在 4001，均未覆盖既有映射 OK
=== S8) 禁用编码 ALARM（一次性）→ 引用该编码的新规则拒 ===
S8a disable -> HTTP 200 code 200 | 操作成功  [期望200] OK
S8b disable 幂等重放 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S8c-腾额度 删规则 30 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S8d 引用已禁用编码 -> HTTP 200 code 4001 | 状态编码已禁用，不可被规则引用: ALARM  [期望4001] OK
S8 断言: 禁用落库（一次性幂等）；引用已禁用编码的新规则 4001 拒且未落库 OK
=== S9) 禁用后再增新码：seq 跳号（ALARM 持 3 不复用——TD-0013） ===
S9a -> HTTP 200 code 200 | 操作成功  [期望200] OK
S9 断言: FAULT seq=4（禁用行 ALARM 序号 3 不复用——序号不漂移锚） OK
=== S10) 规则编辑（白名单 + 启停承载 enabled 0/1） ===
S10a 停用 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S10b 启用 -> HTTP 200 code 200 | 操作成功  [期望200] OK
S10c enabled 值域外 -> HTTP 200 code 400 | 启用状态仅允许 0 停用 / 1 启用  [期望400] OK
S10 断言: 启停落库（edit 承载）+ 白名单改名 + 值域外 400 OK
=== S11) 收尾：删除全部冒烟规则（逻辑删；优先级删除归档不复用——登记） ===
S11a 批量删除 [21, 22, 23, 24, 25, 26, 27, 28, 29] -> HTTP 200 code 200 | 操作成功  [期望200] OK
S11b 已删优先级重存 -> HTTP 200 code 4002 | 优先级已被删除记录占用，不可复用: 1  [期望4002] OK
S11 断言: 软删后列表空；已删优先级重存 4002（删除归档不复用——uk 不含 deleted） OK
SMOKE DONE
```

冒烟场景种子（dev 库内既有数据，不入库不进提交）：模板 6 = TPL13EQ「冒烟设备模板T13」（数据项 ELEC_METER/ELEC_CONS/LOAD_RATE/TOTAL_ELEC）；点位 9001 = T21_SMOKE_DEPT4。S8c「腾额度」步骤：10 条满额时上限预检先于字典校验（主码顺序正确），先删一条规则使 S8d 越过上限闸触达编码校验——补测场景而非主码缺陷。

用毕即停：`dev.ps1 stop` → PID=39136 已停 → `netstat :8098 LISTENING` 复核无输出。dev 日志中 31 条 ERROR 全部为 GlobalExceptionHandler「业务异常」留痕（冒烟负向断言的预期拒绝），无未预期错误。

冒烟脚本两处迭代（均为脚本侧断言问题，主码零改动）：
1. S7 断言首版按紧凑 JSON 子串匹配（`"pointId":9001`）失败——根因：MySQL 8 对 json 列的文本渲染在冒号/逗号后带空格（5.7 为紧凑形态），服务端落库为 Jackson 紧凑序列化（jshell 以应用同版本 jar 复现确认）。修复：断言改 `json.loads` 结构化解析。**为什么现在对了**：断言核对 JSON 语义而非 MySQL 渲染形态。
2. S8d 首版直接建第 11 条规则命中上限 400 而非 4001——根因：上限预检先于字典校验（顺序正确），满额场景触达不到编码校验。修复：先删一条规则腾额度。**为什么现在对了**：场景越过上限闸后编码校验按预期触发。

### 2.4 验收 4：prod 档真实启动回归 —— 通过

打包 `mvn package -DskipTests -pl ot-admin -am`（终码）→ ot-admin-1.0.0.jar。启动命令（沿 0017/0021/0043 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

干净单轮（日志 /tmp/task0048/prod_clean.log，Redis 预置 captcha:code:t0048p="AB43" JSON 字符串形态 EX 600）：

```
2026-11-22T18:20:42.438+08:00  INFO 45144 --- [ot-framework] [main] net.otlab.admin.OtAdminApplication :
    Started OtAdminApplication in 9.416 seconds (process running for 9.937)
=== ERROR count（启动段+探针后复核） ===
0
LOGIN HTTP 200 code 200 message 操作成功
LOGIN token 85a81a81...
LIST /iot/statusrule/list HTTP 200 code 200 total 0
RULE-LIST rows: []
LIST /iot/statusdict/list HTTP 200 code 200 total 4
DICT-LIST rows: [{"statusCode": "RUNNING", "seq": 1, "status": 1}, {"statusCode": "STOPPED", "seq": 2, "status": 1},
  {"statusCode": "ALARM", "seq": 3, "status": 0}, {"statusCode": "FAULT", "seq": 4, "status": 1}]
PROD PROBE DONE
```

真实登录 200 + statusrule/list 200（total 0——冒烟规则已软删、列表过滤逻辑删行，正确行为）+ statusdict/list 200（4 行冒烟字典残留对照面，ALARM 禁用态/FAULT 跳号形态与冒烟终态一致）；启动段与全程 **ERROR=0**（探针后再核一次仍为 0）。单轮即干净（Redis 验证码值按 0021 教训预置为 JSON 字符串形态）。用毕即停：进程已 kill → `netstat :8098 LISTENING` 复核无输出，Redis 验证码键已被登录消费（GET 空 → DEL 0）。

### 2.5 验收 5：改动范围核对 —— 通过

`git diff --name-only fb574cf3c3018c196c91f1df0f503c8e72914b4c..HEAD`（本任务提交 d74d56f 相对 BASE；22 文件）：

```
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/controller/IotStatusDictController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/controller/IotStatusRuleController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/dto/IotStatusDictCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/dto/IotStatusRuleBindingMapForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/dto/IotStatusRuleCreateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/dto/IotStatusRuleUpdateForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/entity/IotStatusDict.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/entity/IotStatusRule.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/mapper/IotStatusDictCodeRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/mapper/IotStatusDictMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/mapper/IotStatusRuleMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/mapper/IotStatusRulePriorityRow.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/service/IotStatusDictService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/service/IotStatusRuleService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/service/impl/IotStatusDictServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/service/impl/IotStatusRuleServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/statusrule/service/support/ConditionExprValidator.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusrule/config/IotStatusRuleMenuPermissionGuardTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusrule/service/impl/IotStatusDictServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusrule/service/impl/IotStatusRuleServiceImplTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/statusrule/service/support/ConditionExprValidatorTest.java
resources/sql/upgrade/20261122_iot_status_rule_domain.sql
```

全部落在 brief 声明范围（statusrule 主码包 / iot 测试包 / 指定升级脚本）内；不碰 frontend/、ingest/、模板域与数据项域（只读校验）、workitems 库其他文件。TASK 状态维持「进行中」，未回写任务详情（协调者收尾）。

---

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| 规则随配置发布下发的「生效」动作 | brief 口径「保存≠生效，登记 0024 发布链」——规则表无 config_version 列（上游 DDL 无此列，未自行加列），生效承接归 TASK-0024 配置快照发布链 |
| condition_expr 表达式真语义校验 | brief 明文「真语义校验归计算引擎」（0036 同款口径登记）——本任务交付语法面（Expander 同口径）+ SELF 引用存在性 |
| 字典名称修正接口 | brief 明文不开放（名称修正属语义变更须评估历史 st_item_value 序号漂移）——无编辑接口为纪律而非缺漏，守卫测试锁定端点数=3 |
| 字典恢复启用接口 | 禁用一次性（brief 只给 disable；恢复=语义漂移面，同上登记） |
| PARENT / 绝对全键引用的存在性核对 | 0036 一期口径「被引实例可能后建」；模板层无法解析父模板（iot_model_template 无父子链），收集不核存在性（报告登记） |
| condition_expr 函数白名单（0036 FormulaFunctionWhitelist 同款接入） | brief 校验面枚举为「花括号/字符集/前缀白名单/长度/括号配平 + 引用存在性」，无函数白名单项——未自行扩面；如需与公式域同口径可在修订轮补 |
| 规则 detail（GET /{id}）端点 | brief 规则接口清单无 detail 行——规则集按模板整读（list 承载回显），未自行扩面 |
| 冒烟受限用户裁剪场景 | 规则挂模板、模板无部门锚点——brief 明文「数据权限裁剪收敛为模板存在同租户校验」，无 dept/instance 裁剪面可测（TC-0002 裁剪红线不适用本域，红线套件未增补） |

## 4. 偏离计划之处

1. **菜单种子七行的构成**（偏离登记，请协调者知悉）：brief 种子段枚举「iot:statusrule:list/query/add/edit/remove + iot:statusdict:add/disable 共七行」，而字典段裁决「新设 iot:statusdict:list 权限行」，且 brief 规则接口清单无 detail 端点（无 query 码消费方）。三处并读下唯一自洽解 = 规则四码（list/add/edit/remove）+ 字典三码（list/add/disable）恰七行 4063-4069——满足验收命令「4063-4069 七行」与「权限标识与接口清单一一对应」纪律（未登记码任何角色都无法获得，admin 的权限串亦来自 sys_menu 行）。取舍：弃 statusrule:query（无端点）；0013「预留行」先例不采纳因会破七行计数。已写进升级脚本头注与守卫测试 javadoc。如认应保留 query 预留行可修订轮改为八行 4063-4070。
2. **直绑映射 body 补 ruleId**：brief body 规格为「pointId + 枚举值映射数组」，但存储裁决为「随规则行」（iot_status_rule.binding_map 列）——无规则定位则映射无处落行，故 body 增加 ruleId（必填）。一条规则行承载一个点位的枚举映射，PUT 整份幂等覆盖。
3. **status_code 校验错误码取 4001**：brief 只说「未登记拒并提示先建状态编码」未定错误码——取引用不存在口径同款 4001（与 SELF 引用不存在/点位不存在一致）。
4. **「活跃规则数」口径**：上限预检的「活跃」= 未逻辑删（deleted=0，含停用行）——停用规则仍是已配置规则集成员不腾条数；brief 未细定义，取保守口径并在脚本头注登记。
5. **规则 list 返回 Page 分页**：brief 未定列表形态，照 metric/point 域 `/list` 分页范式（规则 ≤10 条，pageSize=100 即全量）。
6. **修改表单结构性无 templateId/priority 字段**（brief「不可改」的实现方式）——而非「携带不一致才拒」：调优先级=删旧建新的原语在 DELETE+POST，表单不提供改的通道。

## 5. 遗留问题

1. **dev 库冒烟残留**（不影响仓库与提交）：iot_status_dict 4 行（RUNNING seq1 / STOPPED seq2 / ALARM seq3 已禁用 / FAULT seq4——只增不删纪律无删除接口，属正常业务数据，后续任务可复用为字典锚点）；iot_status_rule 10 条逻辑删行（模板 6 下优先级 1-10 归档占用——后续如需在模板 6 建规则须换模板或手工清理逻辑删行）。
2. **「与上游差异」两处待回写上游**（升级脚本头注登记，建议随 default_calc_period / tenant_id 一并攒上游修订）：① iot_status_dict 为本任务定义新表（上游附录 A 无状态字典 DDL，TD-0013 只给语义）；② iot_status_rule 较 p35 逐字 DDL 增补 binding_map json 列（TD-0014）。
3. **0024 发布链承接**：规则行与直绑映射随配置快照下发计算引擎的接线（规则表无 config_version 列，保存≠生效口径已登记）。
4. **判定/展示链依赖**：状态判定求值（FEAT-0012/TASK-0049）、Redis 当前状态与 st_item_value 留痕（TASK-0050）、展示端翻译为后续任务——本任务只交付配置段（规则/字典/直绑映射）。
5. **环境备注（非本任务引入）**：WSL docker 空闲自停导致 3307 短暂拒连（重试即恢复）；`dev.ps1 start` 在 jar 已存在时不重建——冒烟前需显式 `mvn package` 或 `dev.ps1 build`，否则拉起旧 jar（本次实证并绕过）。
