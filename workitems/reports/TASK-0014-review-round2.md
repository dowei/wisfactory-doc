# 评审报告：TASK-0014 修订轮 scoped 复评（round 2）

- 评审者：fresh reviewer（brief：reports/TASK-0014-review2-brief.md）
- 评审对象：修复 diff 包 `TASK-0014-fixdiff.txt`（159 行，完整读入）＋ 仓库 HEAD（ef7a84d）实况核对——实现 `IotDataItemServiceImpl.validateThresholds/overrideParams`、接口 javadoc、测试文件逐行与 diff 一致；`git show --stat ef7a84d` = 3 文件（接口 javadoc / 实现 / 测试），全部落 dataitem 域声明范围，无越界。
- 独立复跑：`IotDataItemServiceImplParamOverrideTest` **24/24 全绿**（surefire 实测，BUILD SUCCESS），与本类 21→24 增量、报告全量 1483 = 1480 + 3 吻合。

## 一、原 I-1 是否消除（逐条对照 brief 判定标准）

1. **两步绕过序列第二步拒 + 零写入：已消除**。`validateThresholds(form, item)` 配对输入=提交值优先、缺省列取行内现值、NULL 不参与比较（`IotDataItemServiceImpl.java:509-521`），valid 对与 alarm 对同构，与裁决口径逐字一致；行内 (5000,null) 时 PUT 仅 {validMax:100} → 400「阈值配对非法」且 never update、never 钩子（新增 valid/alarm 两条用例断言三连：状态码+文案+零写入零钩子；dev 冒烟 R1/R2/R2b 另有 DB 零写入取证）。接口签名、写路径四列 SET、钩子载荷均未动。
2. **合法 partial 覆盖不误伤：满足**。行内合法对 (0,100000) 上 PUT {validMax:100} → 200、update 恰 1 次、钩子恰 1 次（新增不误伤用例 + 冒烟 R4）；校验模型下所有模型合法的提交均放行。
3. **清除覆盖（null）回归继承路径不受影响：满足**。第一轮锁定清除语义的用例原样保留且绿（partial 未提交列写 NULL 清除 :299-311、全 null 清除合法 :317、清除钩子 :372），冒烟 R6 全 null 清除 → 200 且 DB 回 (null×4)；行内配对合法/为空时全 null 提交的配对模型=行内现值（合法）→ 放行，清除机制本身未被本次修复触动。
4. **红转绿与回归可信**。红输出恰为两条绕过用例、行号与现测试文件吻合，且与修复前逻辑核读一致（旧实现仅同提交双方非空才比较，第二步必放行）；全量回归 1483 全绿有报告原始输出。

## 二、新发现

**无**（Critical 0 / Important 0 / Minor 0）。原 6 条 Minor 均未被修复恶化（per-field 负值/decimal 校验逻辑与 M-1 示例错值原样未动，属既登记 deferred 范围）。

备注（非发现，登记备查）：「缺省列取行内现值参与配对」的校验模型与写语义（缺省列写 NULL 清除，第一轮用例 :311 锁定）存在口径性错位——行内现值与提交值冲突时，「清除+单侧设置」组合提交（如行内 (null,5000) 时 PUT {validMin:10000}，实际写入 (10000,NULL) 本合法）会被 400 拒。此为裁决口径的固有推论（判定标准要求的绕过第二步拒即其镜像形态），方向 fail-closed（实际写入行恒不浓于校验模型行，无非法态可漏过）、逃逸路径明确（先全 null 清除或整对合法化提交），实施报告「推论边界登记」已记载且「清除动作恒放行」备选已 tee up 待协调者裁决——不构成本轮新缺陷，不阻塞。

## 三、判定理由（一句）

修复与裁决口径逐字一致、两步绕过第二步 400+零写入有单测与 dev 冒烟双取证、合法 partial 与清除路径实测不误伤、断言无放松、3 文件无越界、独立复跑 24/24 绿——原 I-1 已修复且修复未引入新缺陷。
