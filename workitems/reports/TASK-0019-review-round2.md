# TASK-0019 修订轮 scoped 复评报告：I-1 并发轮换丢失更新（提交 46d9391）

- 评审者：fresh reviewer（不向协调者追问；no-subagents；只评审不修改任何源码文件）
- 输入：TASK-0019-review2-brief.md 及其必读清单四项逐项读入（原评审报告全文、修复 diff 3 文件约 225 行逐行读完、协调者裁决口径、实施报告修订轮节 R-1~R-6 含红转绿与冒烟原始输出）；另回源核验终码树（IotCredentialServiceImpl / ResultCode / BaseEntity / 实体 / 全仓 grep）与 MyBatis-Plus 3.5.17 jar 内 AbstractMethod 字节码

## 一、结论

**I-1 已修复。** 新发现 1 条 Minor（信息项，无需返工）。

## 二、判定理由（对照判定标准逐条）

### 1. 原 I-1 是否消除——丢失更新通道封死：是

- 机制：`rotate()` 由裸 `updateById`（客户端算版本）改为原子条件 UPDATE——`SET ciphertext=<新密文>, status=1, key_version = IFNULL(key_version,0) + 1 WHERE id=? AND key_version=<前置读值> AND deleted=0`。key_version 在 DB 行内自增（非客户端赋值），两并发方同前置读时：InnoDB 行锁串行 + UPDATE 现读语义下先到者条件命中（kv 恰 +1）并 200，后到者条件不命中影响行数 0 → `BusinessException(5001 VERSION_CONFLICT, "凭据已被并发轮换，请刷新后重试")`，败者密文不落库、不覆盖。恰一方成功、(密文, key_version) 配对与真实代次严格对齐，双 200 单存活的丢失更新通道封死。
- legacy NULL 行同闸：前置读为 null 时走 `.isNull(key_version)`（eq/isNull 两条件布尔互补，不产生 `key_version = null` 非法 SQL），并发下首个 IFNULL(NULL,0)+1=1 后 IS NULL 即不命中——null 路径同样不丢更新。
- 证据链：单测 `testRotate_ConcurrentConflict5001`（0 行 → 5001 精确匹配）+ `testRotate_TwoCallsSamePreRead_OneWins`（同前置读双调用先 200 后 5001、两个 wrapper 均带服务端自增与同一前置读条件）；真实 DB 冒烟 R3（独立会话 FOR UPDATE 持行锁 12s 期间改写 kv=42 后提交，复刻「前置读之后、UPDATE 之前版本被改写」窗口 → 并发轮换 5001、败者未覆盖、终态 42=并发方代次胜出）+ R4（刷新重试恢复 200，42→43 恰 +1）。

### 2. 原子 UPDATE 的正确性：成立

- WHERE 谓词完整性：id + 前置读 key_version + deleted=0 三条件齐备。deleted=0 由 MP 对 wrapper UPDATE 自动追加的声明已独立核验：BaseEntity `deleted` 带 `@TableLogic`，MP 3.5.17 `AbstractMethod` 字节码常量池含 `isWithLogicDelete`/`getLogicDeleteSql` 引用（即 wrapper UPDATE 的 WHERE 构建走逻辑删除分支）。即便该谓词缺失亦无后果——凭据域无任何软删路径（吊销=置 status=0，实体 javadoc 明示不软删）。
- 事务边界：`writeCredential` 为 `@Transactional(rollbackFor=Exception.class)`；旋转路径 UPDATE 前仅 SELECT，5001 抛出时无半写可回滚；密文、status、key_version 同语句落库，无中间态。单语句 CAS 的原子性不依赖事务配置，autocommit 下同样成立。
- 密文与掩码一致性：掩码非库列，列表读时「解密→CredentialMask 掩码化」派生（原评审已实证），不存在掩码落后于密文的回填窗口——裁决二选一取舍在 javadoc 与报告 R-1 说明充分，判定标准所指隐患在本设计下不存在。
- IFNULL 口径：null/空白/非数值经 MySQL varchar 数值运算归一 +1 落 "1"，与被删除的 `nextKeyVersion` 回落语义一致；M-1 所指客户端边界随函数删除整体消失。理论精度边界（varchar(16) 数值超 ~2^53 才失真）属 deferred M-1 同级不可达 territory，不构成发现。

### 3. 修复 diff 新引入缺陷核查：无功能性缺陷

- 5001 语义全局性：`ResultCode.VERSION_CONFLICT(5001, "版本冲突")` 为全仓唯一 5001 常量（grep 全部命中仅本修复 + 字典导入测试中与错误码无关的「5001 行数据」计数）；枚举 message 通用兜底 + 抛出方携带具体提示，照 QUOTA_EXCEEDED 先例；裁决明文授权。GlobalExceptionHandler 透传 BusinessException.code（冒烟 R3a 实测 HTTP 响应体 code=5001）。
- 断言放松：见下「新发现 N-1」，属覆盖面信息项而非缺陷——机制本体断言反而更硬（服务端自增 SQL 片段 + 前置读条件双 wrapper 断言），DB 侧数值正确性由真实冒烟 R2（43→44→45）与 R4b（42→43）实证。
- 审计不受影响：审计套件 7/7 绿（R-3 全量 1755=1723+1+并行任务 31，套件计数与「-1/+3」自洽）；冒烟终态 sys_operate_log 行 270 实测并发冲突拒绝亦落审计（status=0、failReason=并发提示、param 列 NULL），无明文禁令在失败路径同样成立。
- 范围收敛：3 文件全落「credential 域 + ResultCode（裁决授权）」；`nextKeyVersion` 全仓无残留引用（仅 javadoc 历史指称一处）；DuplicateKey 转轮换路径行为未变（仅测试桩 updateById→update 同口径），未重开一期已评定项。
- 原 7 条 Minor 均 defer 未恶化（M-1 随 nextKeyVersion 删除顺带闭环；M-4 实体 toString 未被本轮触碰，维持原状）。

## 三、新发现

- **N-1（Minor，信息项）/ `IotCredentialServiceImplTest.testWrite_RotateUpdatesExistingRow` / 轮换路径 SET 密文值断言较旧版放松**：旧轮换测试断言 `upd.getCiphertext().startsWith("{v1}")` + decrypt 还原；新 wrapper 测试只断言 `sqlSet.contains("ciphertext=")` 片段存在，不再校验 SET 参数的密文值本身。加密动作在 writeCredential 内 rotate 之前且为两条路径共用，新建路径测试仍断言 {v1}+decrypt 往返（测试 166-181 行）、fail-closed 拒存有独立用例，故加密链路覆盖未失守；仅「轮换这一条支路的落库值」失去直接断言（补一行 `containsValue`/startsWith("{v1}") 即可闭合）。不阻塞收尾。

## 四、复评范围声明

仅判定 I-1 修复有效性及其新缺陷，未复跑测试/冒烟（采信 R-2/R-3/R-4 原始输出并与套件计数交叉自洽），未重开一期 Critical/Important/Minor 全场评审。
