# 实施 brief：TASK-0013（实例创建批量生成数据项与相对引用展开）

## 批次

- TASK-0013（本批共 1 个，标准任务 3 人天；backend 串行道——模型域核心拼图）
- BASE：`46e923d`（主仓 HEAD）
- 改动范围（越界即违约）：
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/dataitem/**`（实例数据项域新包）
  - `backend/ot-admin/src/main/java/net/otlab/admin/module/iot/instance/service/impl/IotModelInstanceServiceImpl.java`（**仅一处**：删除保护接入数据项引用计数——0012 预留的接线点）
  - `backend/ot-admin/src/test/java/net/otlab/admin/module/iot/**`（新增/调整测试）
  - `resources/sql/upgrade/20261122_iot_data_item_domain.sql`（新增）
  - 不碰 frontend/、ingest/、模板域（只读其数据项清单）、dataperm 组件

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0013.md`
- 关联对象：同目录 `features\FEAT-0004.md`、`requirements\REQ-0004.md`（验收 2——相对引用展开为唯一键）、`testcases\TC-0004.md`
- 决策：`decisions\ADR-0008.md`（模板变更三档——**新增项默认不参与上级汇总=一档开关默认关**）；`decisions\ADR-0009.md`（TD-0013/0014/0015/0016 相关口径按需）
- 表结构权威（pypdf 自行提取核对）：详细设计 V1.1 PDF §1.6（p13-14 数据项字段表语义）、附录 A p32（iot_data_item / iot_data_item_ref DDL——iot_data_item 起于 p32「CREATE TABLE `iot_data_item`」）；相对引用语法（SELF./PARENT. 前缀）见 §1.6 正文
- 接线对象：`iot/instance/service/impl/IotModelInstanceServiceImpl.java`（删除保护引用计数——0012 预留）；`iot/template/**`（模板数据项清单为生成源，只读）
- 既有范式：iot/template 批量保存（0011）、iot/instance 创建与守卫（0012）

## 接口与决策（逐条结论，不再讨论）

### DDL（脚本 20261122_iot_data_item_domain.sql，CREATE TABLE IF NOT EXISTS 幂等）

1. `iot_data_item`：附录 A p32 逐字（uk_tenant_instance_data (tenant_id,instance_id,data_code) + uk_tenant_full_key (tenant_id,full_key) 双唯一 + idx_tenant_point/idx_tenant_instance）。
2. `iot_data_item_ref`：附录 A p32 逐字（引用关系：item_id/ref_full_key/ref_type 1 数据项 2 点位）。
3. 菜单种子 **4036 起**连续（4030-4035 已占）：iot:item:list/query/add/edit/remove，挂 4010 实例管理页下（数据项管理在实例详情内），幂等 + 超管授权。**不设独立生成权限点**——生成随实例创建（iot:instance:add）与重生成入口（iot:item:add 覆盖）。

### 核心语义（REQ-0004 验收 2 的实现）

1. **批量生成**（实例创建后触发 + 重生成入口）：
   - 触发一：`POST /iot/instance` 创建选模板实例时——0012 的创建流程内**同事务**调用生成服务（生成事务内一致是任务正文硬要求）：按模板数据项清单逐项生成实例数据项。
   - 触发二：`POST /iot/instance/{id}/items/regenerate`（iot:item:add）重生成入口——按模板清单**对齐**：新增缺失项；模板已删项的实例项**保留**（历史不删——实例数据项可能已被绑定/计算）；模板字段变更不在此接口更新（归 0015 模板变更三档流程）。幂等：已存在的 (instance_id, data_code) 跳过不重建。
2. **字段映射**：data_code/data_name/category/unit/value_kind→（无此列，value_kind 留模板不落实例）/default_source_type→source_type/default_formula→formula/agg_method→rollup_method/default_valid_min→valid_min（四阈值同映射）/default_calc_period→calc_period（ADR-0011 补字段的下游兑现）；calc_period 单位换算按模板字段语义（若模板存周期码而实例存秒——以 PDF §1.6 字段表实际口径为准，报告里说明映射）。
3. **相对引用展开**：模板公式中的相对引用（SELF.<code> / PARENT.<code> 语法，以 PDF §1.6 实际语法为准——提取后照做，不猜）展开为该实例唯一键：SELF.X→<instance_code>.X；PARENT.X→<parent_instance_code>.X（顶级实例的 PARENT 引用=生成失败并报明确文案）。展开后公式写 instance 行 formula；同时**逐引用写 iot_data_item_ref**（item_id/ref_full_key/ref_type=1）——依赖图落库。展开失败的项：整批失败（与 0011 批量保存同纪律）。
4. **full_key 冗余存储**：full_key = instance_code + "." + data_code（任务正文），随生成写入，uk_tenant_full_key 兜底。
5. **新增项默认不参与上级汇总**（ADR-0008 一档）：汇总参与开关的载体在上级汇总项的依赖图（ref）——生成的新增项**不主动**改写任何上级汇总项的引用集（上级汇总项引用了哪些下级是在模板里定义的，生成时按模板展开即是其引用集；「默认不参与」=新生成项不新增任何上级对自己的引用）——报告里按此口径说明实现落点。
6. **领域事件**：生成完成后发 Spring 应用事件 `InstanceItemsGeneratedEvent`（payload：instanceId/tenantId/生成项 id 列表/依赖图变更标记）——计算引擎（FEAT-0012）为未来消费方；本期事件发布即可，无消费方注册（监听器不建，报告登记）。**配置版本联动（config_version 字段）归 FEAT-0008 发布链路（TASK-0024），本期置 NULL。**
7. **实例删除保护接线**：IotModelInstanceServiceImpl.deleteInstances 的引用计数加 iot_data_item（instance_id 引用）——0012 预留点，数据项存在时拒绝删除实例（提示项数）；单行改动。
8. **点位绑定项**（source_type=1，point_id）：生成时来自模板的默认绑定若为空则 point_id=NULL（绑定动作归 FEAT-0007 点位绑定 TASK-0023）；ref_type=2 点位引用随绑定动作写入——本期生成只处理公式/汇总类引用。
9. 白名单构建 + 双道闸 4002（uk 双唯一并发兜底）+ `@Transactional(rollbackFor = Exception.class)`（生成主链）照既有范式。

### 测试口径（TDD）

- 生成：选模板创建实例 → 数据项按清单全量生成（逐项字段映射断言/full_key 正确/ref 图行数与内容）；PARENT 引用展开为父实例键；顶级实例 PARENT 引用失败整批；重生成幂等（已存在跳过/模板新增项补生成/模板已删项保留）；新增项不新增上级引用。
- 实例删除保护：有数据项 → 拒（提示项数）；清项后 → 删成功。
- 控制器：重生成路由/权限/参数。
- 冒烟（dev）：0012 冒烟链续做——建模板（含 SELF./PARENT. 公式项）→ 建父实例 → 建子实例（选模板）→ 查数据项清单（full_key/公式展开后/ref 图）→ 重生成幂等 → 删子实例拒（有项）→ 清项后删成功。

## 验收命令（整体验收）

环境事实同前（WSL docker 容器承载 MySQL/Redis；8098 端口纪律；sql-exec.sh 行尾分号）。

1. `cd backend && mvn install -pl ot-core -DskipTests && mvn test -pl ot-admin` → BUILD SUCCESS、0 失败 0 错误（基线 1400 + 新增，Results 段贴回）
2. 升级脚本二次幂等 → 两次 DONE；核对两表 + uk 双唯一 + 4036-4040 五行
3. dev 冒烟（每步原始输出贴回，序列见测试口径）
4. prod 档真实启动回归 → 0 ERROR + item/list 200
5. 改动范围核对：`git diff --name-only <BASE>..HEAD` 全落在声明内（instance 服务一处接线为授权例外）

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0013-report.md`。五项：改动文件清单 / 每条验收命令原始输出 / 未做项与原因 / 偏离计划之处 / 遗留问题。

## 契约

- no-subagents；不改范围外文件；不把 TASK 置「已完成」；不改 workitems 库其他文件。
- 测试失败先定位根因再修；每个修复附「为什么现在对了」。允许 git commit（main，注明 TASK-0013）。

## 收尾回写

- 不要回写 TASK 详情（协调者收尾）。
