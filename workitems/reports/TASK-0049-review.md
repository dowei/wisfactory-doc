# 代码评审：TASK-0049 diff 评审

- 评审角色：代码评审（fresh；口径 `D:\workspace\WisFactory\.agents\protocol\L1-execute.md` §13「diff 评审」）
- 评审对象：提交 `fded29b`（BASE=`7628007`，单提交 23 文件纯 `calc/**`）
- 评审输入（仅依据 brief 指定文件 + 工作区源码）：
  - diff 包 `TASK-0049-diff.txt`；实施 brief `TASK-0049-brief.md`；执行者报告 `TASK-0049-report.md`；TASK 详情 `TASK-0049.md`
  - 决策 `FEAT-0016` / `REQ-0007` / `TC-0016` / `ADR-0009` / `ADR-0010`
  - 上游 `TASK-0048-report.md`（状态规则数据源/字典）/ `TASK-0041-report.md`（执行器+ResultSink+Redis）/ `calc/src/**`
  - 权威：详细设计 V1.1 §1.5B（p12）/ §5（p28）+ `CONV-02`（质量码 0-10）+ `tdengine-super-tables.md`（st_item_value 契约）+ `config-snapshot-schema.md`（§11 对象集边界）
- 契约遵守：fresh 判定、未向协调者追问、未再派 subagent、未修改任何文件、产出只进本报告。

## 越界反查（复核）

`git diff --name-only 7628007..fded29b` = 23 文件，全部 `calc/**`；`git log` 该区间仅一提交，无并行交错；backend/frontend/ingest/契约文件零触碰。工作区仅 `?? .agents/`、`?? .tmp-t0044v/`（开工前既有未跟踪项）。**越界：无。**

---

## 发现清单

| # | 严重级 | 出处 | 一句话说明 |
| --- | --- | --- | --- |
| F-01 | **Important** | `TASK-0049-report.md` §3 未做项 + §2.2；`calc/src/test/java/net/otlab/calc/status/TdengineTraceWriterContractTest.java`；`TdengineTraceWriter.java:66-90`（`rest()`） | **真 TDengine 写入未 live 验真，替代证据（3 项契约测试 + 假适配器捕获 SQL）不充分**——只锁了 SQL 形态与代码路径，未在真 taosAdapter 验证写入效果，且该替代证据未能发现 F-02 的契约错位（详见「评审重点 4 · 明确判定」）。 |
| F-02 | **Important** | `TdengineTraceWriter.java:116`；报告 §2.2 步骤 2 `trace_sql.log`；`resources/contract/tdengine-super-tables.md:15,37`、`config-snapshot-schema.md:32,165`（`full_key = instance_code + "." + data_code`）；`OpenApiTimeseriesClient.java:119`（读侧 `WHERE full_key = ?`） | st_item_value 的 `full_key` 标签写成 `'full:T49.EQ001.STATUS'`，带非契约的 `full:` 前缀（全仓仅此一处）；正确值应为 `T49.EQ001.STATUS`。读侧按规范 fullKey 查询将读不到这些状态行，污染 st_item_value 留痕的可用性。 |
| F-03 | Minor | `StatusEvaluatorRuntime.java:1143-1146`（LwtSubscriber 回调）；`RedisLwtEventSource.java:365-372`；报告 §5 遗留 5 | LWT 通道「收到任意 payload 即离线」，未解析 `online` 字段：若发射侧在同通道发 `online:true`，会被误裁为 OFFLINE；且无显式「回到规则求值」通道，恢复仅靠后续数据项更新触发 `evaluate(false)`。报告已登记为遗留，但 brief 重点 5 问的「离线恢复后如何回到规则求值」未闭合。 |
| F-04 | Minor | `StatusEvaluatorRuntime.java:1064-1074`（先更新 `current` 再写、同状态早退）vs `Resp2StateWriter.java:646-650`、`TdengineTraceWriter.java:99-106`（失败故意不更新 own cache 以便重试） | 两层「未变不重写」缓存语义打架：writer 失败时故意不更新自身缓存以图「下次重试」，但 Runtime 的 `current` map 在写入前已更新、同状态直接早退，导致失败写在下一次同状态求值时不会再重试——旧 Redis/DB 值可滞留至下一次变位，减弱短暂故障后的自愈能力。 |
| F-05 | Minor | `calc/README.md`（diff 行 40）、`StatusEvaluator.java` javadoc（diff 行 856-857）vs 实现 `StatusEvaluator.java:890-904`；`StatusEvaluatorContractTest.lwtOfflineOverridesRuleDecision`（无直绑） | 求值链顺序文档与代码不一致：README/javadoc 写「直绑短路 → LWT 离线优先 → 规则」，代码实际为「LWT 优先 → 直绑短路 → 规则」。LWT 与直绑同时有值的交互无测试覆盖；权威 ADR/§1.5B 未定 LWT 与直绑的相对顺序，两种都站得住，但实现与自身文档须一致。 |
| F-06 | Minor | `Resp2StateWriter.java:87,95-101`（`statusName=escape(statusCode)`、`colorOf` 硬编码 switch）；详细设计 §1.5B p12 + `REQ-0007`（「翻译以规则表与字典为准」） | Redis 当前状态 JSON 的 `statusName` 直接用状态码（如 `"RUNNING"` 而非配置的中文名）、`color` 为固定 switch（自定义状态一律 `#FFFFFF`），未取规则表/字典的 `status_name`/`color`；与 §1.5B「名称/颜色翻译以规则表与字典为准」不符。种子 JSON 解析（`StatusRuleSetParser`）也只读 `seq`、不读 name/color，是该缺口的根因。 |
| F-07 | Minor | 报告 §2.2 规则种子 + `StatusEvaluatorContractTest`/`StatusRuntimeContractTest` fixture（FAULT=3/OFFLINE=4）vs `TASK-0048-report.md` §2.3（FAULT=4，ALARM=3 禁用、序号不复用） | 本期 env 种子/测试用的序号（FAULT=3）与 0048 已落库字典（FAULT=4，ALARM 禁用占 3 不复用）不一致。序号「单一来源」机制本身正确，但 0024 接真规则表时必须以真字典 `iot_status_dict.seq` 为准，否则状态时长/变位历史还原会漂移。属演进衔接风险提示。 |

无 Critical。**2 项 Important（F-01、F-02）+ 5 项 Minor（F-03~F-07）。**

---

## 评审重点逐条结论

**1. 求值链顺序**（直绑短路 / LWT 离线优先 / priority 升序首命中 / UNKNOWN+码 3 / 失败隔离）
主体符合：`StatusEvaluator.evaluate` 按 `Comparator.comparingInt(priority)` 升序首命中（列表乱序无关），高优先级故障先于运行；全不命中落 `StatusDecision.unknown(ts, 3)`（CONV-02 码 3）；单规则求值 `try/catch` 跳过、停用行过滤，`ArithmeticEvaluator` 无值输入抛异常被隔离——与 §1.5B / ADR-0009 TD-0014 / brief 决策 1 一致，契约测试逐条锁定。唯一偏差是 LWT 与直绑的相对顺序文档/代码相反且无交互测试（F-05）。**结论：基本通过，挂 F-05（Minor）。**

**2. 稳定序号（ADR-0009 防漂移）**
`StatusRuleSet.seqOf()` 为唯一读口，`StatusEvaluator.decide()` 统一经它取序号；主码无散落硬编码序号（硬编码 `colorOf` 是展示色，非序号，另见 F-06）；未登记编码 → `seq=null` 不写 0。`stableSequenceIsDictionarySingleSource` 测试锁定不随 priority/ts 漂移。机制层面单一来源成立。种子/测试值与 0048 真字典不一致（F-07）属衔接风险，非机制缺陷。**结论：通过，挂 F-07（Minor）。**

**3. Redis 当前状态**
key 形态 `iot:st:{tenant}:{instance}` 正确（`RedisStateWriter.Recording.keyOf`）；`Resp2StateWriter` RESP2 纯 JDK Socket、零依赖（0041 同款）；未变不重写（按 `statusCode` 缓存）、失败不写缓存/不覆盖旧值均成立；env 门控真 Redis 单跑日志（真 SET + GET 前后一致 + 变化重写）为证。失败重试的两层缓存缺口挂 F-04（Minor）。**结论：通过，挂 F-04（Minor）。**

**4. st_item_value 留痕——见下方「明确判定」。** SQL 形态（逐语句、全限定库名、USING…TAGS、无分号/无 USE、value=稳定序号/quality=0、UNKNOWN 写 -1+码 3）与 0029 纪律和 st_item_value 契约整体吻合；但真写入未验真（F-01，Important）+ full_key 标签契约错位（F-02，Important）。

**5. LWT 离线输入（ADR-0010）**
订阅 `iot:gwstatus:{tenant}`（0029 拟定形态）、离线优先于规则推导、走同一判定链、只订阅不写状态——符合 ADR-0010 TD-0049「接入侧不直写、离线是判定输入」。离线恢复无显式路径 / online 事件误判挂 F-03（Minor）。**结论：通过，挂 F-03（Minor）。**

**6. ResultSink 桥接**
独立组件以 `ResultSink` 装饰器接在 0041 落地边界后，未侵入 `DualTriggerExecutor` 主链；`status-enabled=false` 时 `statusRuntime=null`、Bean 返回 null、`@Nullable` 注入，`CalcEngineApplicationTest.contextLoads` PASS，140 基线逐类零回归；3 Skip = 0041 env 门控 1 + 本任务 env 门控 2（真 Redis 另跑 PASS），门控合理。**结论：通过，无发现。**

**7. 状态规则源 env 种子 JSON 缝**
`StatusRuleProvider` 接口 + `InMemory` 实现构成清晰接缝；冻结快照 §11 明示 `iot_status_rule` 不在五对象集内、承载对象上游未定，实现未改任何冻结契约、未装载该对象，登记为 0024 演进位——缝清晰、无契约突破、后续替换 provider 即可接真规则表，`StatusRuleSetParser` fail-closed。根因性缺口（种子不携带 name/color）挂 F-06。**结论：通过，挂 F-06（Minor）。**

**8. 测试有效性**
`mvn test` 159 = 140 基线（逐类零回归）+ 19 新增（8+6+3+2），0F0E3Skip BUILD SUCCESS；live 冒烟 `RUNNING → FAULT（故障优先）→ LWT OFFLINE → UNKNOWN（quality=3）` 四态逐条与断言对应，同状态 3 次零重写双写口实证；3 处红转绿留痕（求值/运行时双写口/留痕 SQL）附「为什么现在对了」。有效性足够locking逻辑；盲区在真 TDengine（F-01）、LWT+直绑交互（F-05）、statusName/color（F-06）、序号真值（F-07）。**结论：通过（带上述 Minor 盲区）。**

---

## 评审重点 4 · 明确判定：TDengine 真写入未验真的证据充分性

**判定：不充分（Insufficient）。按 brief 要求判 Important（F-01）。**

理由：

1. **验证层级只到「形」未到「效」。** 3 项契约测试（`TdengineTraceWriterContractTest`）用注入的 `SqlRunner` 断言 SQL 字符串形态、未变不重写、失败上抛；live 假适配器（Python HTTPServer@18049）回显并捕获了生产代码生成的真实 INSERT SQL。两者都只证明「代码生成了预期形态的单条 SQL 并调用了 REST 出口」，**没有任何一步让真 taosAdapter 接受并执行这条 SQL**——子表 `INSERT…USING…TAGS` 能否按 6 个 tag 的顺序/类型自动建表、库 `iot_<tenant>` 是否存在、value=1.0（DOUBLE）/quality=0 是否按预期落列、`rest()` 工厂的 Basic 认证 + `/rest/sql` + `"code":0` 解析能否对上真应答，全部未验。
2. **这是首个真实 ResultSink 写 st_item_value（报告自述），且 0029 已实证多语句信封会「code=0 静默空操作」。** 恰恰是「SQL 看似正确但真库不落行」这一历史教训要求真机复核；仅凭形态测试无法排除同类静默失败。
3. **实证：替代证据确实漏掉了一个真实契约错位（F-02）。** 假适配器回显的 `full_key='full:T49.EQ001.STATUS'` 带非契约的 `full:` 前缀，而冻结契约与全仓其他代码都定义 `full_key = instance_code + "." + data_code`；契约测试只断言了 `startsWith(...TAGS ())` / `contains("'status'")` / `contains("VALUES (...)")`，**没有断言 full_key 的具体值**，因此测试全绿也没发现它。这直接证明「形态测试 + 假适配器」的证据强度不足以替代真写入 + 读回验证。
4. 环境阻断（REST 21s reset、HTTP=000）是真实且已如实登记（非隐瞒），但属于「遗留问题」而非「已验真」；报告结论把「不影响三条验收判定」说得成立，却不足以支撑「留痕真实可用」这一更宽的断言。

**需补什么（按优先级）：**

1. **先修 F-02**：`insertSql()` 的 full_key 标签改为 `instanceCode + "." + STATUS_DATA_CODE`（去掉 `full:`），并新增契约测试**精确断言完整 tag 六元组**（`tenant_id, full_key, instance_code, data_code, category, source_type`）的字段名与值，尤其 full_key 的规范形态。
2. **TDengine 恢复后真机联调（报告 §5 遗留 1 已列，须执行而非只登记）**：建 scratch 库 → 经生产 `TdengineTraceWriter.rest()` 真写一条 STATUS 行 → `SELECT ... WHERE full_key = '<规范 fullKey>' ORDER BY ts DESC LIMIT 1` 读回，断言 `value=seq`、`quality=0`、`UNKNOWN 行 value=-1/quality=3`、未变不重写（行数不变）→ DROP scratch 库。**必须含读回**，否则仍是「写入成功」而非「落点正确」。
3. 真机验证 `rest()` 对真 taosAdapter 应答的 `"code":0` 解析（当前只见过假 runner 返回的 `{"code":0}`）。
4. 明确 `iot_<tenant>` 库的创建/存在责任（本实现假定已存在，未建库）。

在 1+2 完成前，st_item_value 留痕的「真实写入可用」应标为**未验真**，不能按已闭合处理。

---

## 总体结论

**不通过（退回一轮）**：存在 2 项 Important（F-01 真写入证据不足、F-02 full_key 标签契约错位），按 §13 应走退回路径。F-01 的修复含「补真机写入 + 读回」与「修正并锁定 full_key」两项；F-02 须在修订轮一并修正。其余 5 项 Minor（F-03~F-07）记 TASK `任务结果` deferred，随收尾进 LOG，不单独触发修订轮。

验收命令本身（mvn 159/0F、live 四态序列、范围核对 23 文件全 calc/**）证据齐全、可复现；缺陷集中在留痕写口的真实验证与一处标签值口径，不涉求值主链与越界。复评为 scoped：只看 F-01/F-02 的修复 diff 与本报告两条 Important。
