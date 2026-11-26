# 复评报告：TASK-0007 修订轮 scoped 复评（review round 2）

- 复评者：fresh reviewer（scoped：只判断口径不对称修复是否正确、有无新缺陷；不重开全场）
- 复评日期：2026-11-22
- 输入：评审 brief、原实施报告（TASK-0007-report.md）、修复 diff（TASK-0007-fixdiff.txt，commit `eb01ba2`，5 文件 +94/-9）
- **结论：口径修齐完成；新发现 0（无 Critical / Important / Minor）**

## 1. 判定明细（对照 brief 判定标准）

### 1.1 三处读路径口径一致（通过）

| 读路径 | 位置 | 核对结果 |
| --- | --- | --- |
| instance 详情 | `IotModelInstanceServiceImpl.getInstanceDetail`（ot-admin, :639） | 判定改走 `isInstanceReadableByUser(scope, instance)`；403/strategy 分支（SCOPE_FORBIDDEN、原文案）逐字未动 |
| dataitem 清单 | `IotDataItemServiceImpl.selectItemsByInstanceId`（:380） | 同一谓词；集合类语义保留（不可见 → `List.of()`），实例实体改为单次取出复用，无额外查询 |
| dataitem params | `IotDataItemServiceImpl.getEffectiveParams`（:455） | 同一谓词；内联抛出与原 `requireDataPerm` 调用**同码同文案**（FORBIDDEN + "无权访问该实例（数据权限不命中）"），拒绝路径行为不变 |

谓词与列表 OR 分支（`selectInstanceList` :596-608）**逐分支核对等价**：createByOnly+可见集非空 = `id IN (...) OR create_by=me`；createByOnly+空集 = 仅 `create_by=me`；非 createByOnly = 仅 `id IN`。谓词中 `createBy != null` 为 Java 侧空安全守卫，与 SQL `create_by = me` 对 NULL 行不求真的语义等价；`SecurityUtils.getUserId()` 返回 `Long`，`Long.equals(Long)` 值比较正确。

### 1.2 他建实例仍拒绝（通过）

- 非 createByOnly 用户（厂长类）：OR 分支不生效，判定退化为原 `instanceIds.contains`，行为零变化。
- createByOnly 用户对他建实例：双向断言在位——scenario5 对 INST_A_FACTORY（create_by=ADMIN）详情 403；DataItem 新用例对 INST_A_DEVICE 清单空 + params 403（校验 FORBIDDEN 码）。存在性泄露面未扩大（清单空集、403 文案均不变）。

### 1.3 写防线未动（通过，逐调用点核实）

- instance：`requireDataPerm`（:680，instanceIds 口径）仍护 create-under（:159）与 update（:309）；`deleteInstances` 内联校验（:402-405）instanceIds 口径，diff 未触及。
- dataitem：`overrideParams` → `requireDataPerm`（:412）；`regenerateForInstance` 内联 instanceIds 校验（:337），均未触及。
- 新谓词仅被上述三处读路径引用（grep 全量核对），无任何写路径挂接。

### 1.4 create_by 语义正确（通过）

谓词比较 `instance.getCreateBy()`（`MybatisPlusConfig.insertFill` ← `StpUtil.getLoginIdAsLong()` 审计填充）与 `SecurityUtils.getUserId()`——即创建者 userId。测试断言行值（`assertEquals(USER_ME, own.getCreateBy())`），dev 冒烟 S2 实测新建行 createBy=8 与登录 userId 一致，语义闭环。

### 1.5 范围与断言质量（通过）

- diff 范围 = 授权范围（instance 详情一处 + dataitem 清单/params——后者为裁决口径明示项，非蔓延）；无额外文件；HEAD 主代码与 `eb01ba2` 逐字一致（`git diff eb01ba2 -- .../main/java` 为空）。
- 无断言放松：scenario5 保留原树/列表断言并只增（详情 200 + instanceCode + createBy 行值 + 他建 403）；DataItem 新用例正负双向且校验具体值（清单条目 id、VO instanceId/fullKey、403 码），非"不抛异常即过"。
- 转正用例取 dept_id=NULL 的纯 scope=4 自建实例——仅 create_by 分支可达，是最锐利的对齐证明；观察项转正断言质量充分。

## 2. 新发现

无。（非缺陷备注：谓词在两个 impl 各持一份 identical 副本，与既有 `requireDataPerm` 双份并存的本地惯例一致，未破坏任何既有单一事实源约定，不计缺陷。）

## 3. 一句判定理由

三处读路径统一走同一 `create_by=me` 谓词且与列表 OR 分支逐分支等价、他建拒绝面与写防线逐调用点核实未动、断言只增不放松——观察项确认的口径不对称缺陷按裁决口径修复完成，无新缺陷。
