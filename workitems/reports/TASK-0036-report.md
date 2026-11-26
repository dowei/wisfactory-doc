# TASK-0036 实施报告（绑定/公式/汇总/阈值配置接口与引用边维护）

- 执行者：implementer（subagent，no-subagents 单体会话）
- BASE：`071dd1f`（主仓 HEAD，派发态实测一致）；提交：`9e73d3e`（main，注明 TASK-0036，19 files / +2533 −17）
- 总体结论：**四条验收全部通过**——全量回归 1923 = BASE 实测基线 1866（worktree 在 071dd1f 实跑）+ 新增 57，0 失败 0 错误 BUILD SUCCESS；dev 冒烟 S1-S10 全步绿（0013 生成链建项 → SELF/绝对键混合公式展开落库+ref 去重建边 → impact 正反向+指标空集 → 循环依赖 4001 含环上项 id 列表 → 6 条负路径逐个按码位拒绝 → bind/换绑覆盖旧边 → rollup 出边清空 → 阈值四列落库 → effective-params 合并核对）；prod 档真实启动 9.63s **启动段 ERROR=0** + 真实登录 200 + **impact HTTP 200** + thresholds 写路径 200 + effective-params 200；19 文件改动全落 `dataitem/**` 声明范围。TASK 状态维持「进行中」未回写（协调者收尾）。

---

## 1. 改动文件清单（071dd1f..9e73d3e，19 个，全落在 brief 声明范围 `dataitem/**` 内；instance/** 未触达——继承链扩口经既有 effective-params 完成无需实例域改动）

### 1.1 主码（15）

| 文件 | 性质 | 说明 |
| --- | --- | --- |
| `dataitem/controller/IotDataItemController.java` | 修改 | +5 端点：PUT `/iot/item/bind`、`/iot/item/formula`、`/iot/item/rollup`（§3.1 p22 实际行，iot:item:edit）+ PUT `/iot/item/{itemId}/thresholds`（双路由别名 `/iot/dataitem/{itemId}/thresholds`，iot:item:edit）+ GET `/iot/item/{itemId}/impact`（双路由别名 `/iot/dataitem/{itemId}/impact`，iot:item:query）；写入口挂 @OperateLog UPDATE；类注释登记权限口径 |
| `dataitem/service/IotDataItemService.java` | 修改 | +5 方法契约：bindPoint / saveFormula / saveRollup / overrideThresholds / getItemImpact（javadoc 完整登记校验链与口径） |
| `dataitem/service/impl/IotDataItemServiceImpl.java` | 修改 | 核心实现（+513 行）：①bindPoint——点位存在/同租户（插件注入）/status=1 校验，source_type=1+point_id 落库+退场字段清 NULL，ref 先物理删后插 ref_type=2 点位边；②saveFormula——0013 Expander 同口径语法链（残留花括号/键长 180/展开长 2000/顶级 PARENT 400）+ 引用存在性 4001（SELF/PARENT 码集+指回本/父实例的绝对键；第三方绝对键维持 0013 不核存在）+ 函数白名单占位 4001 + 循环依赖 DFS 4001（环上项 id 列表进文案）+ 落库与 ref 重建同事务；③saveRollup——SUM/AVG 校验，出边清空（动态纳入口径）；④overrideThresholds——0014 overrideParams 另一入口（数据项单键定位→委托）；⑤getItemImpact——正反向反查+双路匹配+对端实例可见性裁边+指标空集占位；⑥getEffectiveParams 扩展取值四要素三态；⑦公共闸 requireInstance/requireItemOfInstance/rebuildRefs/validation4001（4001 常量，ot-core 不扩枚举） |
| `dataitem/mapper/IotDataItemRefMapper.java` | 修改 | +`deleteByItemIdPhysical`：物理删重建专用（uk_item_ref 不含 deleted 列——@TableLogic 的 UPDATE deleted=1 会让旧边继续占用键、重存同引用必撞唯一索引；旁路插件显式 tenant_id 条件，照 selectOccupancyByInstance 范式） |
| `dataitem/dto/IotItemBindForm.java` | 新增 | 绑定表单（instanceId/itemId/pointId 三 @NotNull） |
| `dataitem/dto/IotItemFormulaForm.java` | 新增 | 公式表单（formula @NotBlank @Size(2000)；花括号占位形态为输入契约、裸键展开形态兼容受理） |
| `dataitem/dto/IotItemRollupForm.java` | 新增 | 汇总表单（rollupMethod @Pattern SUM\|AVG） |
| `dataitem/dto/IotItemImpactVO.java` | 新增 | 影响预览响应（本项身份 + outgoing/incoming/metrics 三段——0057 编辑器接线点） |
| `dataitem/dto/IotItemImpactEdgeVO.java` | 新增 | 引用边行（direction OUT/IN + refType + refFullKey + 对端项/实例六字段；裁剪与不可解析语义入注释） |
| `dataitem/dto/IotItemImpactMetricVO.java` | 新增 | 指标关联边占位形态（本期恒空集，0043 接线零结构变更） |
| `dataitem/dto/IotItemEffectiveParamsVO.java` | 修改 | +四要素 8 字段（sourceType/formula/rollupMethod/calcPeriod × 值+来源），注释登记 TASK-0036 扩展口径（formula 模板默认=模板态花括号公式，与实例态裸键形态区分） |
| `dataitem/event/DataItemConfigChangedEvent.java` | 新增 | 领域事件 record（itemId/tenantId/changeType 枚举 BIND/FORMULA/ROLLUP——照 0013 事件范式只发布不建监听，配置快照联动归 0024） |
| `dataitem/service/support/FormulaFunctionWhitelist.java` | 新增 | 一期函数白名单占位校验器（ALLOWED=ABS/ROUND/MAX/MIN/IF/DELTA/RATE 常量 + CLOSED_INTEGRAL；调用点扫描，INTEGRAL 专门文案；真 Aviator 语法归 FEAT-0012） |
| `dataitem/service/support/FormulaCycleDetector.java` | 新增 | 循环依赖 DFS 纯函数件（onPath+done 剪枝；环列表从候选项起按引用方向；下游环不可达候选不误报——brief「本项可达自身」口径） |
| `dataitem/domain/IotRollupRules.java` | 新增 | 汇总规则声明位（brief 决策 4）：ALLOWED_METHODS=SUM/AVG + MISSING_SUBITEM_QUALITY_CODE=7（ADR-0009 TD-0021）+ MISSING_POLICY=PARTIAL_WITH_INCOMPLETE_MARK；计算行为归 FEAT-0012 |

### 1.2 测试（4 文件，净增 57 用例）

| 文件 | 性质 | 说明 |
| --- | --- | --- |
| `dataitem/service/impl/IotDataItemServiceConfigTest.java` | 新增 37 例 | 绑定成功/换绑覆盖（InOrder 先删后插+点位边内容）/点位不存在与停用拒/pointId 空拒/三防线；公式成功 5 形态（SELF+绝对键混合展开落库+ref_item_id 回填/先删后插次序/裸键形态兼容/第三方绝对键不核存在 ref_item_id=NULL/PARENT 展开回填）+ 拒存 8 形态（SELF/PARENT 不存在 4001/顶级 PARENT 400/畸形 token 400/展开超长 400/键超长 400/INTEGRAL 与 FOO 4001/存在性先于 DFS）+ 循环 4 例（自环/两节点/跨实例 PARENT 闭环含两 id/旧边替换不参与 DFS）+ rollup 3 例 + 阈值入口 2 例 + impact 5 例（正反向+指标空集/双路匹配/裁剪/不可解析保留原文/防线 400·403）+ 继承链扩展 2 例 + 空公式拒 + 事件 payload |
| `dataitem/service/support/FormulaFunctionWhitelistTest.java` | 新增 4 例 | 白名单放行/INTEGRAL 专门文案/集合外拒/常量恰七函数 |
| `dataitem/service/support/FormulaCycleDetectorTest.java` | 新增 8 例 | 自环/两节点/三节点/无环链分叉/下游环不误报/带分支出边仍检出/空图/环序从候选起 |
| `dataitem/controller/IotDataItemControllerTest.java` | 修改 +8 例 | 五端点权限注解（bind/formula/rollup/thresholds=iot:item:edit、impact=iot:item:query）/四写入口 @OperateLog UPDATE/bind 三元透传+pointId 缺失 @Valid 拒/formula 透传+4001 透传/rollup 透传+MAX 400 透传/thresholds 双路由/impact 双路由+403 透传 |

**未触碰**：frontend/、ingest/、模板域、metric 域、instance/**、point/**、ot-core（4001 用域内常量不扩 ResultCode 枚举）、resources/sql（详见偏离 3 的权限码口径）、workitems 库（本报告除外）。

## 2. 验收命令原始输出

### 2.0 pypdf 提取 §3.1 数据项配置段（接口行核对，brief 裁决依据）

《工厂数据采集系统详细设计（数据库与接口）》V1.1 p22（factory-data-platform-detail-design-v1.1.pdf，pypdf 逐字提取 §3.1「管理系统 API（下）：实例、数据项、指标、发布」数据项段）：

```
GET /api/v1/iot/item/list    iot:item:list  按 instanceId 查实例数据项列表
PUT /api/v1/iot/item/bind    iot:item:bind  绑定点位（source_type=1，写 point_id）
PUT /api/v1/iot/item/formula iot:item:edit  保存公式（校验通过才落库并重建 iot_data_item_ref）
PUT /api/v1/iot/item/rollup  iot:item:edit  设置子级汇总（source_type=3 + rollup_method）
POST /api/v1/iot/item/validate iot:item:edit 公式校验（语法、引用存在性、循环检测），不落库
POST /api/v1/iot/item/preview  iot:item:edit 公式试算（…归 FEAT-0012/0037·0038）
```

判定：brief 的「数据项级 PUT 或按 §3.1 实际行」取 §3.1 实际行——三类配置接口按 `PUT /iot/item/bind|formula|rollup` 落地（本工程无 `/api/v1` 前缀，与既有控制器命名空间一致）；阈值/影响预览无 §3.1 行，按 brief 字面路径以既有命名空间为主路由、brief 字面 `/iot/dataitem/**` 作双路由别名（照 0023 `/iot/point/status` 别名范式）。

### 2.1 验收 1：`cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` —— 通过

```
mvn install -pl ot-core -DskipTests:（-q 静默构建，无错误输出）
===CORE-INSTALL-DONE===

mvn test -pl ot-admin（Results 段）:
[INFO] Results:
[INFO]
[WARNING] Tests run: 1923, Failures: 0, Errors: 0, Skipped: 3
[INFO]
[INFO] BUILD SUCCESS
[INFO] Total time:  47.268 s
[INFO] Finished at: 2026-11-22T16:07:15+08:00
```

**基线闭环（开工实测）**：在 BASE=`071dd1f` 的独立 worktree 实跑同命令——`Tests run: 1866, Failures: 0, Errors: 0, Skipped: 3`，BUILD SUCCESS。**1923 = 1866 + 57 新增**（FormulaFunctionWhitelistTest 4 / FormulaCycleDetectorTest 8 / IotDataItemServiceConfigTest 37 / IotDataItemControllerTest +8），既有测试零修改零跳过（IotDataItemControllerTest 仅追加方法，既有 15 例原样通过）。新增测试类定向：

```
[INFO] Tests run: 37, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataitem.service.impl.IotDataItemServiceConfigTest
[INFO] Tests run: 8,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataitem.service.support.FormulaCycleDetectorTest
[INFO] Tests run: 4,  Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataitem.service.support.FormulaFunctionWhitelistTest
[INFO] Tests run: 23, Failures: 0, Errors: 0, Skipped: 0 -- in net.otlab.admin.module.iot.dataitem.controller.IotDataItemControllerTest（15 既有 + 8 新增）
```

### 2.2 验收 2：dev 冒烟（mvn spring-boot:run，日志 backend/target/task0036_dev.log）—— 通过

启动前 8098 探测：无 LISTENING。启动：`Started OtAdminApplication in 9.601 seconds (process running for 10.019)`；**启动段 ERROR=0**；全程 7 条 ERROR 均为负路径故意触发的业务异常留痕（见 S5/S6/S10a，无意外错误）。用毕即停：taskkill PID 48432 → 8098 复核无 LISTENING。

夹具前置：复用既有模板 TPL13EQ（id=6，设备级 4 项：ELEC_METER 直绑 / ELEC_CONS 公式 `DELTA({SELF.ELEC_METER})` MINUTE / LOAD_RATE 公式 `{SELF.ELEC_CONS} / {PARENT.ELEC_CONS} * 100` HOUR / TOTAL_ELEC 汇总 SUM DAY）与既有点位 9001/9002（status=1）；经 API 新建实例（0013 生成链）：

```
=== S1 admin 登录（dev /auth/login/test）===
{"code":200,...,"data":{"userId":1,"username":"admin","token":"2f1e7c03-..."}}

=== S2 建实例 POST /iot/instance {shortCode:EQ36, parentId:8(F01.WS13), nodeType:4, templateId:6} ===
{"code":200,"message":"操作成功"}
DB: 实例 id=19 F01.WS13.EQ36（父=8）；生成 4 项——
  18 ELEC_METER(source_type=1)
  19 ELEC_CONS(source_type=2, formula=DELTA(F01.WS13.EQ36.ELEC_METER), calc_period=60)  ← SELF 展开落库
  20 LOAD_RATE(source_type=2, formula=F01.WS13.EQ36.ELEC_CONS / F01.WS13.ELEC_CONS * 100, calc_period=3600)  ← SELF+PARENT 双展开
  21 TOTAL_ELEC(source_type=3, rollup=SUM, calc_period=86400)
ref 边：19→18、20→19、20→F01.WS13.ELEC_CONS(id=1)（ref_item_id 均回填）

=== S3 formula 配置（SELF/绝对键混合）PUT /iot/item/formula
    {instanceId:19, itemId:19, formula:"{SELF.ELEC_METER} * 0.85 + {F01.WS13.EQ36.ELEC_METER}"} ===
{"code":200,"message":"操作成功"}
DB: item 19 → source_type=2, formula=F01.WS13.EQ36.ELEC_METER * 0.85 + F01.WS13.EQ36.ELEC_METER
    （展开后裸键形态、point_id/rollup_method 清 NULL）；ref 重建后恰 1 条边
    id=11: item_id=19 → F01.WS13.EQ36.ELEC_METER (type=1, ref_item_id=18)  ← 同引用去重

=== S4 impact 反查 GET /iot/item/19/impact ===
{"code":200,"data":{"itemId":19,"instanceCode":"F01.WS13.EQ36","dataCode":"ELEC_CONS",
 "outgoing":[{"direction":"OUT","refType":1,"refFullKey":"F01.WS13.EQ36.ELEC_METER",
   "peerItemId":18,"peerFullKey":"F01.WS13.EQ36.ELEC_METER","peerDataCode":"ELEC_METER",
   "peerInstanceId":19,"peerInstanceCode":"F01.WS13.EQ36"}],
 "incoming":[{"direction":"IN","refType":1,"refFullKey":"F01.WS13.EQ36.ELEC_CONS",
   "peerItemId":20,"peerDataCode":"LOAD_RATE","peerInstanceCode":"F01.WS13.EQ36"}],   ← 受影响上级项
 "metrics":[]}}   ← 指标空集占位（0057 接线点口径）

=== S5 制造循环（A 引 B 引 A）===
S5a: PUT formula itemId=20 LOAD_RATE {SELF.ELEC_CONS} → {"code":200}
S5b: PUT formula itemId=19 ELEC_CONS {SELF.LOAD_RATE} →
{"code":4001,"message":"循环依赖: 公式引用形成闭环，保存拒绝（环上数据项 id: [19, 20]）——请调整引用关系后重试（项 ELEC_CONS）"}
DB 复核: item 19 formula 未变（拒存零写入）；item 20 formula=F01.WS13.EQ36.ELEC_CONS（S5a 合法落库，旧边 9/10 物理删重建为单边 id=12）

=== S6 负路径六连 ===
S6a INTEGRAL({SELF.ELEC_METER},3600) → {"code":4001,"message":"公式函数 INTEGRAL 一期不开放（仅随 REQ-0019 能源计量预置公式提供），保存拒绝（项 ELEC_CONS）"}
S6b {SELF.ELEC-X} → {"code":400,"message":"公式包含无法识别的引用 token（展开后残留花括号）: 示例 {SELF.ELEC-X}"}
S6c FOO({SELF.ELEC_METER}) → {"code":4001,"message":"公式函数不在一期白名单: FOO（一期仅允许 ABS/ROUND/MAX/MIN/IF/DELTA/RATE，真实语法校验归 FEAT-0012）（项 ELEC_CONS）"}
S6d {SELF.NOTHING} → {"code":4001,"message":"公式引用的数据项不存在: SELF.NOTHING（实例 F01.WS13.EQ36，项 ELEC_CONS）"}
S6e bind pointId=99999 → {"code":400,"message":"绑定点位不存在或已删除: 99999"}
S6f rollup MAX → {"code":400,"message":"汇总方式仅允许 SUM/AVG"}

=== S7 绑定与换绑（ref_type=2 点位边）===
S7a: PUT /iot/item/bind {itemId:18, pointId:9001} → {"code":200}
DB: item 18 → source_type=1, point_id=9001, formula/rollup_method=NULL；边 id=13: item 18 → T21_SMOKE_DEPT4 (type=2)
S7b 换绑: PUT /iot/item/bind {itemId:18, pointId:9002} → {"code":200}
DB: item 18 → point_id=9002；边重建后 id=14: item 18 → T21_SMOKE_DEPT2 (type=2)（旧边 13 物理删——换绑覆盖旧边）

=== S8 rollup TOTAL_ELEC→AVG（汇总标识+范围规则存储，出边清空）===
PUT /iot/item/rollup {itemId:21, rollupMethod:AVG} → {"code":200}
DB: item 21 → source_type=3, rollup_method=AVG, formula/point_id=NULL；edges_of_21 = 0（ref 不展开子级全量边，动态纳入口径）

=== S9 阈值覆盖（brief 字面别名路由）===
PUT /iot/dataitem/21/thresholds {validMin:0, validMax:5000, alarmLow:1, alarmHigh:4500} → {"code":200}
DB: item 21 → valid_min=0.000000 / valid_max=5000.000000 / alarm_low=1.000000 / alarm_high=4500.000000

=== S10 阈值主路径负例 + effective-params 合并核对 + impact 别名 ===
S10a: PUT /iot/item/21/thresholds {validMin:6000, validMax:5000} →
{"code":400,"message":"阈值配对非法：结果行 valid_min > valid_max（本次提交缺省列取行内现值参与配对）"}  ← 0014「结果行四列」口径
S10b: GET /iot/instance/19/items/21/effective-params →
{"validMin":0.000000,"validMinSource":"INSTANCE","validMax":5000.000000,"validMaxSource":"INSTANCE",
 "alarmLow":1.000000,"alarmLowSource":"INSTANCE","alarmHigh":4500.000000,"alarmHighSource":"INSTANCE",
 "sourceType":3,"sourceTypeSource":"INSTANCE","formula":null,"formulaSource":"NONE",
 "rollupMethod":"AVG","rollupMethodSource":"INSTANCE","calcPeriod":86400,"calcPeriodSource":"INSTANCE"}
  ← 阈值四列 INSTANCE（S9 覆盖）+ 取值四要素纳入同一继承链口径（brief 决策 3：formula 双空 NONE）
S10c: GET /iot/dataitem/19/impact（brief 字面别名路由）→ {"code":200,...} 同 S4 数据
```

判定：**通过**——brief 冒烟序列全步覆盖（0013 生成链建实例数据项 → formula 配置 SELF/绝对键混合 → impact 反查 → 制造循环 → 4001 环列表 → 阈值覆盖 → effective-params 合并核对），另加 bind/换绑/rollup/负路径六连补齐三类接口面。

### 2.3 验收 3：prod 档真实启动回归 —— 通过

打包 `mvn package -pl ot-admin -am -DskipTests` → ot-admin-1.0.0.jar（133,015,350 字节）。启动（沿 0022-0023 确认命令 + 三密钥）：

```
DB_HOST=localhost DB_PORT=3307 DB_NAME=ot_wisfactory DB_USERNAME=root DB_PASSWORD=root \
REDIS_HOST=localhost SM4_KEY=otlab_260829____ CRYPTO_DATA_KEY=otlab-data260930 CRYPTO_INDEX_KEY=otlab-idx-260930 \
java -jar ot-admin/target/ot-admin-1.0.0.jar --spring.profiles.active=prod
```

启动前 8098 探测：无 LISTENING。启动（日志 backend/target/task0036_prod.log）：

```
2026-11-22T16:41:35.548+08:00  INFO 44456 --- [main] net.otlab.admin.OtAdminApplication : Started OtAdminApplication in 9.63 seconds (process running for 10.113)
启动段 ERROR 计数 = 0
```

真实登录（Redis 预置 captcha:code:task0036p="AB36" JSON 字符串 EX 600 + SM4 hex 密文 base64，jshell ROUNDTRIP=admin123 验证）与探针：

```
=== P1 POST /auth/login（uuid=captcha 键名）===
{"code":200,...,"token":"8a74a058-...","userId":1,"username":"admin"}
（此前一次探针因字段名笔误 captchaKey → {"code":500,"message":"验证码标识不能为空"}——本人失误留痕，非应用缺陷，见偏离 7）

=== P2 GET /iot/item/19/impact ===
HTTP_STATUS:200
{"code":200,"data":{"dataCode":"ELEC_CONS",...,"outgoing":[...],"incoming":[...],"metrics":[...空]}...}

=== P3 GET /iot/item/list?instanceId=19 ===
HTTP_STATUS:200（dev 冒烟数据在位）

=== P4 PUT /iot/item/21/thresholds（别名路由写路径 prod 档连通）===
{"code":200,"message":"操作成功"}

=== P5 GET /iot/instance/19/items/21/effective-params ===
{"code":200,"data":{"validMin":0.000000,"validMinSource":"INSTANCE",...,"sourceType":3,"sourceTypeSource":"INSTANCE",...}}
```

全程 `grep -c " ERROR "` = **1**（唯一一条即 P1 前的字段名笔误触发「验证码标识不能为空」业务异常留痕；修正后全绿）。用毕即停：taskkill PID 44456 → 8098 复核无 LISTENING → Redis 验证码键 DEL → 0。

### 2.4 验收 4：改动范围核对 —— 通过

```
$ git diff --name-only 071dd1f..HEAD
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemController.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/domain/IotRollupRules.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemBindForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemEffectiveParamsVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemFormulaForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemImpactEdgeVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemImpactMetricVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemImpactVO.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/dto/IotItemRollupForm.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/event/DataItemConfigChangedEvent.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/mapper/IotDataItemRefMapper.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/IotDataItemService.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceImpl.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/FormulaCycleDetector.java
backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/service/support/FormulaFunctionWhitelist.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/controller/IotDataItemControllerTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/impl/IotDataItemServiceConfigTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/support/FormulaCycleDetectorTest.java
backend/ot-admin/src/test/java/net/otlab/admin/module/iot/dataitem/service/support/FormulaFunctionWhitelistTest.java
```

19 文件全部落在 brief 声明范围 1/3 两行（`dataitem/**` 主码 + `test/iot/**` 测试）；frontend/、ingest/、模板域、metric 域、resources/sql、workitems 库零接触。IotDataItemMenuPermissionGuardTest 与 SaPermissionRegistryConsistencyTest 全量回归中通过（注解 ⊆ sys_menu 登记五行）。

## 3. 未做项与原因

| 项 | 原因 |
| --- | --- |
| POST /iot/item/validate、/iot/item/preview 两接口（§3.1 另两行） | brief 决策 5 明文：Aviator 真语法校验与试算归 FEAT-0012（0037/0038 试算转发计算引擎）——本任务交付引用结构与循环依赖层；validate 的校验链本体已在 saveFormula 内交付（后续 validate 接口即同链复用不落库形态） |
| `iot:item:bind` 独立权限码 | §3.1 行写 iot:item:bind，但 sys_menu 未登记该码（0013 种子仅五行 4036-4040）且本批 resources/sql 不在改动范围、SaPermissionRegistryConsistencyTest 强制注解⊆登记集——按 0013 升级脚本头注预登记口径复用 iot:item:edit（REQ-0015「改绑单独权限」留待菜单种子扩行，见遗留 3） |
| iot_point.bound_instance_id 回填 | brief 契约只含 point_id 必填/点位校验/ref_type=2 边——点位冗余归属回填归点位域任务（本批未获授权触碰 point/**） |
| rollup_method 收 MAX/MIN（REQ-0017 提及四选一） | brief 决策 1 契约=SUM/AVG（与 iot_data_item.rollup_method DDL 注释一致）；MAX/MIN 扩口随 FEAT-0012 汇总计算语义一并放开（IotRollupRules.ALLOWED_METHODS 单点扩） |
| 汇总子级全量引用边 | brief 决策 1/4：汇总范围动态纳入（引用边在计算期展开），ref 不展开子级全量边——出边清空即该口径的存储面表达；「参与上级汇总」开关落边归 0043/计算引擎链路 |
| 影响预览指标边 | brief 决策 6：ref 图不含指标边——空集占位，指标引用边归 0043/计算引擎登记（响应形态已冻结 IotItemImpactMetricVO 三要素） |
| DataItemConfigChangedEvent 监听器 | brief 决策 7：照 0013 事件范式只发布不建监听；配置快照联动归 0024 |
| 阈值接口独立的配对校验实现 | 0014 overrideParams 已交付「结果行四列」配对全量语义与时间线钩子——brief 明示「另一入口，语义一致」，实现为定位后完全委托（零重复实现，单测覆盖委托链） |

## 4. 偏离计划之处

1. **三类接口路径与请求形态**（brief 字面 vs §3.1 实际行）：brief 决策 1 自带裁决「数据项级 PUT，或按 §3.1 实际行对齐」——取 §3.1 实际行 `PUT /iot/item/bind|formula|rollup`（body 携 instanceId+itemId 双键定位，防线锚实例）；brief 字面的 `/iot/dataitem/{itemId}/binding` 等未单独立口。
2. **thresholds/impact 主路由落既有命名空间 `/iot/item/**`，brief 字面 `/iot/dataitem/{itemId}/...` 作同 handler 双路由别名**（0023 `/iot/point/status` 别名范式）：dataitem 是 Java 包名而非 URL 命名空间，既有数据项域 URL 均为 /iot/item/**（0013 起四接口在位），双路由同时满足 brief 字面与命名空间一致性；两条路由 dev/prod 均实测 200。
3. **bind/rollup 清退场字段**（brief 未明说）：取值方式三选一互斥——bind 落库时 formula/rollup_method 写 NULL、formula 落库时 point_id/rollup_method 写 NULL、rollup 落库时 formula/point_id 写 NULL；否则实例行残留跨取值方式死配置、effective 继承链读取口径被污染。brief 决策 2「formula 换绑/清除时旧边清理」的清除语义即经此通道达成。
4. **循环依赖检测的图构建口径**：brief 决策 8 只给「DFS 跨实例（PARENT 边向上+绝对键）」——实现按「候选项新引用边替换其 DB 旧边后，全租户数据项依赖图（ref_type=1，ref_item_id 尽力回填缺失时按 ref_full_key 兜底解析）DFS 本项可达自身」；PARENT 边闭环在 ref 图上与单实例闭环同构，无需 ancestors 单独遍历（ancestors 语义由 PARENT 展开时的父实例解析承担）；环在下游但不可达候选不拒存（brief 字面「本项可达自身」）。
5. **引用存在性的绝对键核对范围**：brief「绝对键逐个核对本实例与父实例数据项」——指回本实例/父实例的绝对键强制核对（4001）；指向其他实例的绝对键维持 0013 生成链一期口径不核存在（被引实例可能后建，ref_item_id 尽力回填），dev 冒烟 S6 系与单测双证。另加「已展开裸键形态兼容受理」（§3.1 preview 示例形态无花括号可剥——点分全键按绝对键提取入图与核对），否则该形态会静默零边入库。
6. **WSL 环境事件（无代码影响，登记备查）**：dev 冒烟准备期 WSL VM 反复在空闲 ~30-60s 后被终止（journal 「uncleanly shut down」、docker 容器随 VM 死、mirrored loopback 随之拒绝连接），根因判定为 **WSL2 VM 空闲终止**（本次会话无持久 WSL client 保活，与既往任务「IDE/终端长会话在场」的隐含前提不同）。期间做过两次**临时** `.wslconfig` 实验（autoMemoryReclaim=disabled / firewall=false）均未改变现象，且均已在验收 3 开始前**恢复原文件逐字同款**（cat 核对 gradual 原值在位）；最终稳定方案=启动持久 `wsl -d Ubuntu -- sleep 3600` client 会话保活（零配置改动），冒烟与 prod 全程网络稳定（100s 逐 10s 探针全 OK）。user 目录 `.wslconfig` 现状=原样未动。
7. **prod 全程 1 条 ERROR**：为执行者首次登录探针字段名笔误（captchaKey vs LoginDTO.uuid）触发的「验证码标识不能为空」业务异常留痕，非应用缺陷；启动段 ERROR=0，修正后登录/impact/thresholds/effective-params 全 200。
8. **mvn install -pl ot-core 采用 -q 静默**：验收 1 首段无逐行输出（BUILD SUCCESS 以无错误输出 + `===CORE-INSTALL-DONE===` 标记佐证）；ot-admin test 段 Results 原始输出完整贴回（2.1）。

## 5. 遗留问题

1. **WSL VM 空闲终止行为**（本机环境，建议协调者归口）：autoMemoryReclaim=gradual + mirrored networking 配置下，无持久 client 时 VM 约 60s 空闲即终止并带走 docker 容器（恢复靠下次 wsl 调用自启）。本次以 sleep 3600 保活会话完成验收（约 16:36 起 1h 内自然到期）；若后续任务仍遇「30s 后 DB/Redis 拒连」，先检查有无 wsl.exe 活跃 client。根治选项（用户裁决）：`.wslconfig` 加 `vmIdleTimeout`/关闭实验特性，或给 WSL 配常驻服务任务。
2. **dev 库冒烟留痕**（是否清理由协调者裁决）：实例 F01.WS13.EQ36（id=19，template 6）+ 数据项 4 行（id 18-21：ELEC_METER 绑 9002 / ELEC_CONS 混合公式 / LOAD_RATE {SELF.ELEC_CONS} / TOTAL_ELEC AVG+阈值 0/5000/1/4500）+ ref 边 3 行（id 11/12/14）。重跑冒烟需换实例短码（同码撞占用属预期）。
3. **iot:item:bind 独立权限码扩行**：需 sys_menu 新按钮行（号段 4041+，parent_id=4010）+ 对应升级脚本——resources/sql 不在本批范围，留独立 SQL 任务；届时控制器注解从 iot:item:edit 换 iot:item:bind 即可（守卫测试同步锚点）。
4. **引用边物理删无历史**：uk_item_ref 不含 deleted，重建=物理删+插（依赖图为派生数据、配置变更历史归 iot_data_item_history 时间线，0039 交付）；若未来需要边级变更审计，须另立机制（建议随 0039 时间线一并评估）。
5. **0057 前端接线**：impact API 已就绪（主路由 GET /iot/item/{itemId}/impact + 别名 /iot/dataitem/{itemId}/impact，权限 iot:item:query，响应含 full_key/实例编码/名称+正反向+指标空集）——按 0057 报告预估「一处事件监听 + 一处 API 调用 + showPendingActions 收回」接线即可；metrics 恒空集，0043 登记指标边后前端无需改结构。
6. **Redline 域未扩**：数据项域红线测试（DataItemRedlineTest）为 0013 生成链口径，本批新写路径的防线由 IotDataItemServiceConfigTest 37 例覆盖（403 文案不泄露、防线顺序、零写入断言）；如评审要求红线矩阵扩列，归协调者裁决。

---

## 附 1：契约自查

- 未派 subagent；未把 TASK-0036 状态改为「已完成」；workitems 库仅新增本报告文件；TASK 详情未回写（协调者收尾）。
- 提交恰 19 文件（`git show 9e73d3e --stat` 可核），全部落在 brief 改动范围声明内；frontend/、ingest/、模板域、metric 域、ot-core、resources/sql 零触碰；worktree 基线核对目录已删除（`git worktree list` 仅余派发前既有的 WisFactory-base-check）。
- 8098 纪律：dev/prod 两实例均「启动前探测无监听 → 用毕 taskkill → 复核 0 监听」；Redis captcha 键已 DEL。
- 四条验收命令原样执行、原始输出贴回（2.1-2.4）；测试失败修复两处均为测试侧自身笔误（raw 长度断言算错 1040→1427、缺 import），主码无「先错后改」记录——TDD 红转绿发生在新增测试对未实现接口的编译失败/断言失败阶段（IotDataItemServiceConfigTest 首轮 37 例 1 失败即上述断言笔误）。
- brief 决策 1-8 逐条落点：①三接口+阈值（§3.1 行，见偏离 1/2）②先删后插+换绑清旧边（物理删 mapper，InOrder 断言+冒烟 S7）③继承链扩展（effective 四要素，冒烟 S10b）④汇总标识+范围规则+缺失打码 7 声明位（IotRollupRules，出边清空口径）⑤DELTA 可用/INTEGRAL 封闭+函数白名单占位（FormulaFunctionWhitelist）⑥impact 反查（正反向+指标空集+裁剪，0057 接线点）⑦事件三类只发布不建监听（DataItemConfigChangedEvent）⑧循环依赖 DFS 跨实例（同图覆盖，环 id 列表进 4001 文案）。
