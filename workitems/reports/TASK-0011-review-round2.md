# TASK-0011 修订轮 scoped 复评报告（round 2）

- 评审者：fresh reviewer（仅依据复评 brief「必读」清单：原评审报告、修复 diff 全文 252 行已完整读入、实施报告 §8；未向协调者追问，no-subagents，未修改任何评审外文件）
- 评审对象：修订提交 `c94c9b7`（5 文件 +121 −29），对照原发现 I-1 与 `6126aa1` 首版
- 交叉核验（本复评自行核实，非采信报告）：`IotDataItemTemplateMapper.selectByTemplateAndCodes` 实际源码（@InterceptorIgnore + 原生 @Select，`#{templateId}` / foreach `#{code}` 全参数化）、`IotDataItemTemplateServiceImpl.saveItems` 实际源码（requireTemplateExists 在第 86 行先于旁路预检第 124 行；两文案分支逐字核对）、`resources/sql/upgrade/20261122_iot_template_domain.sql` 第 144 行（uk_template_data_code 确为 (template_id, data_code) 二列、不含 deleted）、`BaseEntity.deleted`（@TableLogic Integer，null 判定与 deleted==1 分支类型匹配）、TASK-0002 旁路先例 `SysUserMapper`（显式条件 + @InterceptorIgnore + 理由 javadoc 同范式）、`selectByTemplateAndCodes` 全库唯一调用点即 saveItems 预检（无越权扩散）

## 一、原 I-1 是否消除：**已修复**

1. **预检拦截**：saveItems 第二遍预检改走含已删行旁路 Mapper（原生 @Select 天然不过 @TableLogic 注入，SQL 无 deleted 过滤）——删项后重存同码在落库前被拦截，不再撞 uk_template_data_code 物理索引。
2. **文案区分（与裁决逐字一致）**：已删行 → 4002「模板数据项编码已被删除记录占用，不可复用: \<code\>」；活跃行 → 4002「模板数据项编码已存在: \<code\>」。物理 uk 保证同 (template_id, data_code) 至多一行，deleted 分支无歧义；按请求顺序取首命中项。
3. **「请重试」瞬态措辞收敛**：该措辞仅存于 DuplicateKeyException 兜底，经含已删行预检后只剩真实并发插入窗口可触达（注释与 javadoc 已写明该语义前提），永久死路路径不再误报瞬态冲突。
4. **决策声明补齐**：控制器与 service 接口 javadoc 均补「同模板数据项逻辑删后编码不可复用」声明（与模板编码域同款对齐），并登记「未来要复用须立 ADR」的演进口径。
5. **实测证据链完整**：红轮实证修复前误报文案（expected 已删占用文案 / was「并发冲突请重试」，双缺陷同现）；绿轮 23/23；全量 1331 通过（=首版 1330 + 新增 1）；dev 冒烟 R4 DB 核对 deleted=1 → R5 重存同码 4002「已被删除记录占用」、R7-retry 活跃行 4002「已存在」，均与裁决逐字一致，且 R7 首发被 @RepeatSubmit 拦截属既有防重闸非缺陷。

## 二、旁路 Mapper 安全范式：满足

- 显式条件 `template_id = #{templateId} AND data_code IN (...)` 与物理唯一索引作用域精确一致；全参数化（#{}/${} 零混用，foreach 亦 #{），无注入通道。
- 租户隔离经 template_id 传递成立：requireTemplateExists（插件隔离的 selectById，iot_ 前缀 fail-closed 纳入）先于旁路预检执行，templateId 必属会话租户；旁路不构成跨租户放宽。
- `@InterceptorIgnore(tenantLine="true")` + 理由 javadoc 照 TASK-0002 SysUserMapper 既有范式；含已删行是本修复的目的行为而非越权放宽；调用方非空保证结构性成立（items 1-500 逐项非空码校验 + 批内去重先判，IN () 不可达）。

## 三、修复 diff 新引入缺陷：无

- 测试断言**收紧**非放松：两文案由 contains 改逐字 assertEquals，双用例均补 verify(never()).insert；红转绿用例 mock insert 抛 DuplicateKeyException 以实证旧路径，断言「未走到 insert」。
- `testSaveItems_DuplicateKeyFallback` 残留一处已失效的 `selectList` stub（生产代码不再走该路径，Mockito 默认空表使测试语义仍成立、兜底闸仍被有效覆盖）——纯卫生级残留，不构成断言放松/旁路滥用/事务边界缺陷，不计新发现。
- 事务边界未动：@Transactional(rollbackFor=Exception.class) 整批回滚语义保持，预检为同事务内普通读。
- 原 7 条 Minor 未被本次修复触碰或恶化（M-2 400 映射口径、M-1 头注笔误等均原样 deferred）。

## 四、结论

- **I-1 已修复**；新发现：**0**（Important 0 / Minor 0）。
- 一句判定理由：删项→重存同码在预检即被含已删行查询拦截并给出与裁决逐字一致的「已被删除记录占用，不可复用」文案，活跃行报「已存在」，「请重试」仅剩真实并发窗口可触达；旁路 Mapper 满足显式条件/参数化/理由声明范式且租户隔离经 template_id 传递成立，测试断言收紧、实测红转绿与冒烟 R5/R7 证据链完整，无新缺陷。
