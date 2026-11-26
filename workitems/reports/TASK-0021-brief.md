# 实施 brief：TASK-0021（iot_point 点位 CRUD 与配额校验）

## 批次

- TASK-0021（本批共 1 个，标准任务 3 人天；backend 串行道——**0003 预留的 PointQuotaCounter 接线点 + F5 数据权限裁剪接线批**）
- BASE：以派发时主仓 HEAD 为准（协调者派发前回填，开工时记录进报告）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/point/**`（点位域新包）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/quota/**`（PointQuotaCounter 真实实现——0003 预留接口的接线，实现类新增）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/gateway/service/impl/IotGatewayServiceImpl.java`（F5 接线：网关列表数据权限裁剪——0017 评审 F5 裁决随本批落地）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_point_domain.sql`（新增）
  - 不碰 frontend/、ingest/、模板/实例/字典域（只读）、dataperm 组件本体

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0021.md`
- 关联对象：同目录 `features\FEAT-0007.md`、`requirements\REQ-0008.md`（点位 9 接口与验收）、`testcases\TC-0007.md`
- 决策：`decisions\ADR-0009.md`（TD-0015/0016：未绑点位锚点、补传豁免）；`decisions\ADR-0011.md`（TD-0045 单位归 sys_dict 强制校验）；`decisions\ADR-0016.md`（裁决 6 点位双条件 NULL 不特判）
- 配额接线对象：`iot/quota/PointQuotaCounter.java`（0003 接口）+ `SysTenantServiceImpl.checkPointQuota`（校验器已就位——**本任务实现计数器 + CRUD 调用方接线，接线即生效**；TOCTOU 残余按 0003 评审 M-7 裁决接受并登记遗留）
- 数据权限裁剪接线（F5 裁决）：网关列表 + 点位列表**同批接裁剪**——条件形态照 ADR-0016 裁决 6：`dept_id IN 部门集 OR bound_instance_id ∈ 可见集`（网关版：dept_id IN 部门集 OR instance_id ∈ 可见集）；null 双条件不特判（fail-closed）
- 表结构权威：附录 A p31 iot_point DDL（pypdf 自行提取核对）；接口行 §3.1 p21-22 点位段（自行提取——9 接口清单）
- 既有范式：iot/channel（0018 双 guard 模式）、iot/dataperm（0005 组件消费）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_point_domain.sql，逐字 + 幂等 + 头注）

`iot_point`：附录 A p31 逐字（uk_tenant_point_code + 三索引；OPC UA 发现字段预留=protocol_addr JSON 内约定，不拆列）；菜单种子 **4053 起**连续（4052 已占）：iot:point:* 五行（list/query/add/edit/remove）挂 4011 点位管理页下，幂等 + 超管授权。**status 列即启停**（建子表触发按「启用」语义）。

### CRUD 九接口（/iot/point 段，按 §3.1 点位段实际行——自行提取对齐）

list（分页+过滤 pointCode/pointName/dataType/valueType/status/channelId/deptId/boundInstanceId + **数据权限裁剪接线**）、detail、add（point_code 租户内唯一双道闸 4002 + 不可改后续；data_type 1-4/value_type 枚举校验；**unit 强制 sys_dict 单位字典校验**（TD-0045——字典类型编码 `iot_unit`，条目不存在 → 拒绝并提示归口运营方维护——ADR-0011 附条件口径）、min<=max、channel_id 归属校验、bound_instance_id/dept_id 校验存在同租户、**配额强控接线**：insert 前调 checkPointQuota（tenantId 从会话）——超限 4003 拒绝）、edit（point_code 不可改；白名单）、**批量启停**（如 §3.1 有独立行则独立接口，无则 edit 承载）、remove（软删；编码删除归档不复用——逻辑删占用 uk 同款口径）、点位启停事件（见裁决 4）。

### 配额计数器真实实现（0003 接线）

`QuotaPointCounterImpl implements PointQuotaCounter`——count(tenantId)=iot_point 活跃行 COUNT（@TableLogic 过滤）；checkPointQuota 在 add 前调用（**超限 4003**——0003 已交付校验器，本任务只补计数器 bean + 调用方；单测红转绿：99/100 放行、100/100 拒、未装配视为 0 的兜底语义随真实 bean 移除）。

### 建子表触发事件（裁决）

点位建档**启用**（add with status=1 或启停操作置 1）→ 发 Spring 应用事件 `PointEnabledEvent`（pointId/tenantId/pointCode/valueType/采样配置）——**写入方（TDengine 子表创建）归 TASK-0029**，本期只发布事件 + javadoc 契约声明（判据按业务事件，端到端入库验证归 TC-0009/联调）——照 0013 InstanceItemsGeneratedEvent 范式（只发布不建监听）。

### 数据权限裁剪接线（F5 裁决落地）

点位 list + 网关 list 同批接组件：resolve(当前用户) → allTenant 直过；否则条件拼装（点位：dept_id IN OR bound_instance_id IN，NULL 不特判；网关：dept_id IN OR instance_id IN）；详情 403（SCOPE_FORBIDDEN）；纯 scope=4 → 列表 OR create_by？点位无 create_by 查询语义（配置类）——**点位/网关列表 create_by 分支不适用**（资产类非个人创建物——登记口径留痕，与 REQ-0002 裁决 3 「配置类对象」的字段适用性对齐：create_by 分支仅对有创建者语义的查询生效，资产归属类以 dept/instance 锚定）。补红线套件回归断言（*RedlineTest 增点位/网关越权用例——**允许改 redline 测试包**）。

### 测试口径（TDD）

CRUD 九接口逐个（唯一 4002 双道闸/改码拒/unit 字典校验/min<=max/配额 4003 接线红转绿/启停事件发布断言/删除归档不复用）；裁剪接线（点位/网关列表越权 403/空集/NULL 行 fail-closed——红线套件同步补）；控制器注解与 4053-4057 守卫。
冒烟（dev）：admin 建点位（unit 用字典外值 → 拒）→ 字典补登记 → 成功 → 配额 100 场景（临时把租户 point_quota 调小模拟）→ 100/100 拒 4003 → 裁剪场景（受限用户 list）→ 启停事件 Redis/日志留痕核对（事件无消费方，发布即可）。

## 验收命令（整体验收）

环境事实同前（WSL docker；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线以开工实测为准 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对表结构 + 4053-4057 五行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + point/list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内（redline 测试包增补为授权例外）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0021-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0021）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
