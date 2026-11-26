# CONTRACT-35-EVAL 报告：§3.5 推送载荷契约评估与组稿

- 执行：documenter subagent（读类批次，不改代码）
- BASE：04fc3a5（已核对，HEAD=04fc3a5）
- 日期：2026-11-21

## 结论（先行）

**B——并入统一消息模型，unified-message-model.md 已升 v1.1.0**（增补「3.5 数据服务对外 API」章；§3.2 两章逐字未动，已程序化比对确认字节一致）。评估记录：`resources/contract/push-payload-evaluation.md`。

判定要点：数据项订阅推送 4 字段中 ts/value/quality 与 §3.2 同名同义，但 fullKey 字段、单条消息体形态与订阅主题 `iot/open/{tenant_code}/{instance_code}/item` 在 v1.0.0 均缺失 → A 不成立；指标推送 9 扩展字段无同名异义、无类型冲突 → 可安全并入；归属已由 ADR-0007（「统一消息模型扩展指标事件形态并升版本」）、ADR-0010 TD-0030（「保 ADR-0007 载荷冻结契约」）、TASK-0053（「消息体=统一消息模型」）定版 → 不采纳 C。

## 1. 改动文件清单

| 文件 | 动作 |
| --- | --- |
| `resources/contract/unified-message-model.md` | 改：升 v1.0.0 → v1.1.0。头部更新版本/来源/修订记录；正文新增「3.5 数据服务对外 API」章（§3.5 p26 原文逐字）；「与上游差异」按 v1.0.0 声明 + v1.1.0 增补声明两层重组，注明新增来源章节。§3.2 两章与其余正文未动 |
| `resources/contract/push-payload-evaluation.md` | 新增：评估记录（结论、提取核对、两推送面定义、字段级比对表、三选一理由、任务影响、遗留） |
| `D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\CONTRACT-35-EVAL-report.md` | 新增：本报告 |

其余文件一律未碰；未提交 git（契约目录本就未入库，LOG-0004 记载「待与下一任务同批提交」，本批沿用不擅自提交）。

## 2. 提取核对结果

- 来源页码：§3.5 在 PDF p26（共 36 页）。§3.4 与 §3.5 同页；§3.5 起自标题行「3.5 数据服务对外 API（/openapi/v1，appKey + 签名）」，止于页尾注记「……403 类错误全量记录。」；p27 起为 §4，§3.5 无跨页续文。brief 的「约 p26」确认无误。
- 提取方法：pypdf 逐页提取（运行时为 `py -3`，Python 3.14 + pypdf 6.19.0；缺省 `python` 3.8 在 -I 隔离模式下无 pypdf，与 FREEZE brief 所述环境略有差异，方法不变）。
- token 回核：31 个关键 token 全部命中 p26 提取文本——两个推送主题（iot/open/{tenant_code}/{instance_code}/item、iot.metric.result.{tenant}、iot/{tenant}/metric/out）、9 个扩展字段名（metric_code/instance_key/full_key/period/window_start/window_end/value/yoy/mom/quality）、fullKey、compareNote、六条查询路径、示例字段（F01_WATER_AVG_DAY/windowStart/instanceCode/metricCode/unit/m3）、iot_api_call_log/403。
- 字形核对：p26 含 Kangxi 部首字形 20 种（⼀⼚⼝⼦⼩⼯⼿⽅⽆⽐⽔⽤⽰⾄⾏⾯⾼⻅⻚⻬⺟），均为一一对应标准字映射，规范化不改语义；本页无连字。字段名、主题名、JSON 示例逐字保留。
- 冻结完整性核验：以组稿时留存的 v1.0.0 §3.2 两章全文与新文件程序化比对，结果字节一致（True）。

## 3. 未做项与原因

- 未建 TASK/LOG 对象：brief 收尾回写规定「无 TASK 对象（协调动作批次），报告即终稿」。
- 未触发 C 分支（独立契约文件、先停等裁决）：结论为 B，分支不适用。
- 未对两处上游歧义做归一（推送载荷字段名风格 snake_case vs camelCase、window_start/window_end 线上类型）：契约约束「拿不准不猜」，如实留痕于「与上游差异」第 7/8 条与评估记录遗留清单，待 TASK-0053 落地定版。
- 未把 quality 0-10 枚举全表补进新章注释：CONV-02 的全表落点在 TDengine 超级表契约与质量码映射（LOG-0004 遗留 2 裁决「契约独立小节承载」），消息模型新章保持逐字原文，不叠加。

## 4. 偏离计划之处

- 无实质偏离。两点透明说明：① §3.5 按 brief「附 §3.5 原文」整节逐字并入（含六行查询 API），非仅两行推送；契约面（订阅主题行、推送行）在「与上游差异」第 4 条中显式圈定，查询行仅为上下文，鉴权细则不随本契约裁决。② 提取用 `py -3`（见第 2 节），环境与 FREEZE brief 记载的「python + pypdf」有解释器版本差，方法与结果不受影响。
- PDF 内部一致性与库内文档无冲突：§3.5 九字段为 ADR-0007 字段清单（metric_code、period、窗口起止、value、yoy、mom）的超集（多 instance_key 与 quality），与 ADR-0010 TD-0030「指标全键+窗口幂等」互证，无冲突点需裁决。

## 5. 遗留问题

1. 推送载荷线上字段名风格（snake_case / camelCase）未定版——TASK-0053 落地时定版，契约届时升修订号。
2. window_start / window_end 推送载荷线上类型（字符串日期时间 vs 毫秒时间戳）原文未明示——随 TASK-0053 定版。
3. yoy/mom 为 null 时推送载荷是否随附 compareNote：原文 compareNote 语义写在 REST 查询注记中，推送行未提及——随 TASK-0053 定版。
4. 指标推送 Kafka 与 MQTT 双通道是都发还是按对接方可配，原文未明示——联调前与中台对接方确认。
5. 数据项订阅推送频率/背压原文无数值——TASK-0053 联调项。
6. CONV-03 通知动作：v1.1.0 属契约变更（增章），须由协调者向各系统（管理面、数据服务、计算引擎、中台对接方）发出变更通知后相关联调方可依据 v1.1.0「3.5」章进行；TASK-0027/0034 等 §3.2 面任务不受影响、无需返工。
7. resources/contract/ 三契约与本批两文件均未入库，待协调者同批提交（沿用 LOG-0004 的处理）。
