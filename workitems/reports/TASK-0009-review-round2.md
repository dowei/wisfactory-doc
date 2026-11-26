# TASK-0009 修订轮 scoped 复评报告（fresh reviewer）

- 评审对象：`TASK-0009-fixdiff.txt`（commit `9045f38`，8 文件 +762 −71，与仓库 HEAD 逐字一致）；依据：原评审 `TASK-0009-review.md`（I-1/I-2/I-3 + 6 Minor）、实施报告 §7 修订轮、首轮实施 brief、协调者修复裁决。
- 本轮范围：只判定原三条 Important 是否修复、修复本身有无新缺陷；不重开全场。重点审区为自实现流式解析器（POI XSSFReader + XMLReader）：资源关闭 / XXE / 异常中断语义 / 替换完整性。
- 评审者独立实证（只读，未改任何文件）：git 仓库状态与提交核验；main 源码 grep 残留扫描；`poi-5.2.5.jar` 常量池提取核验 XMLHelper 安全特性；控制器/服务层/异常处理器/测试导入逐文件核对。

## 一、结论：三条 Important 全部修复

### I-1 物理行号保真 —— 已修复

- rowIndex 改由 `<row r="N">` 属性保真（`DictImportExcelSaxParser.SheetRowHandler`）：XML 缺席的空行不产出条目、XML 占位空行产出可跳过条目（`isSkippable`），两种情况均不产生行号前移；r 缺省回退「上一行 + 1」；表头固定物理第 1 行。行号契约不再依赖「无空行」前提。
- 证据链三面互证：解析器测试（空行后 rowIndex=4，POI sharedStrings 编码通道）+ 控制器测试（真实含空行夹具，断言数据行 [2,4]）+ dev 冒烟 R1 回执行号实测 **[2,4,5]** 逐字通过（报告 §7.5）。

### I-2 首行缺格 null 保真 —— 已修复

- SAX 逐单元格取值，新解析器无任何 alias/列名机制（grep 确认 aliasFirstLine 仅存于 javadoc 记述性文字）；缺格稀疏补 null；空串归一 null（hutool 对 null 写空 inlineStr `<t></t>`，与「缺格」导入语义等价，且复现旧 ignoreEmptyRow 对空行的丢弃口径）；MISSING_REQUIRED 由既有服务层校验对 null 生效，"A"/"B" 替换通道无代码存在。
- 证据链：解析器测试（首数据行 dataName=null，POI sharedStrings 与 Hutool inlineStr 双通道）+ 控制器测试 + dev 冒烟 R2（rowIndex=2 MISSING_REQUIRED data_name；DB 复核 `'REV_MISS_CODE','B','A'` 0 行落库——垃圾码静默落库路径实测封死）。

### I-3 流式解析 + 5000 行 fail-fast —— 已修复（Hutool→POI 偏离按「意图 vs 字面」判定为符合）

- **偏离裁决**：brief 字面提议 Hutool SAX；执行者实证 hutool 5.8.25 的 SAX 通道在本 classpath（POI 5.2.5 namespace-aware 安全 reader 下 default-namespace 元素 qname 为空串，hutool 以 qname 匹配）静默零回调——该根因技术自洽，与 hutool/POI 5.x 已知不兼容类问题一致；升级 hutool 越出「改动限字典导入域」的授权。评审 I-3 意图 = 有界流式解析，POI XSSFReader 自实现达成该意图（逐行回调、超限即止、poi-ooxml 既有依赖零新增）。**判定偏离符合。**
- **fail-fast 中断语义**：第 5001 数据行抛 `DictImportRowLimitException`，RuntimeException 经 JAXP XMLReader.parse 原样透传（MockMvc 全链路测试断言 400 + 精确文案「单批导入上限 5000 行，当前 5001 行」+ 服务层 never() 调用；dev R3 5001 行 0.144s 整批拒绝）。服务层保留 `dataRows.size()`（isSkippable 过滤后口径）第二道闸，与解析层计数同基准，无「5000 数据 + N 说明行被误拒」的口径冲突（已核 ServiceImpl:185 的判定对象是过滤后清单）。
- **资源关闭审计**：OPCPackage / sheet 流 / sharedStrings 部件流均 try-with-resources，成功与中止路径都关闭；POI ZipPackage 构造内自关入参流；流式打开不产生临时文件（无临时文件泄漏面）。未发现泄漏。
- **XXE 审计**：两处 XMLReader（sharedStrings 与 sheet）均取自 POI `XMLHelper.newXMLReader()`——自本机 `poi-5.2.5.jar` 常量池实证：secure-processing、disallow-doctype-decl、external-general/parameter-entities=false、load-external-dtd/grammar=false、accessExternalDTD/Schema/Stylesheet、entityExpansionLimit 全部就位，与 POI 自用口径一致。XXE 防护达标。
- **替换完整性**：main 源码无 `read(1)`/`ExcelReader`/alias 代码残留（grep 核验，仅 javadoc 记述）；`ExcelUtil` 仅存于模板/错误清单写出侧（写出侧无此三问题，注释已声明）；发现一处方法级 javadoc 陈旧残留（见 N-1）。

## 二、新发现（Important 0 / Minor 4）

1. **Minor N-1 · `IotDataDictController.importDict` javadoc 陈旧残留**：方法注释仍写「控制器只做 Excel 单元格→字符串的机械提取（照 SysUserController 导入范式，Hutool ExcelReader）」——修订后该方法不再提取单元格（委托 SAX 解析器）。类级与 parseImportRows 注释已更新，此方法级注释漏改。纯文档残留，无行为影响。
2. **Minor N-2 · 解析器内存声明与实现不符（fail-fast 只覆盖数据行，skippable 行物化无上限）**：行数中止仅按非可跳过行计数；空占位行/说明行条目被 `finishRow` 无条件物化进 rows 清单且无上限（解析器测试亦印证空占位行产出条目）。空占位行密集的合法文件（普通 deflate 比率即可通过 ZipSecureFile 0.01 闸）可放大至数百 MB XML + 数百万 DictImportRow 条目；叠加 `OPCPackage.open(InputStream)` 固有的全量包缓冲（multipart 临时文件实际在磁盘上，未走 open(File) 随机访问路径；POI javadoc 自述 open(InputStream) 显著更耗内存）——报告/代码所称「内存 O(共享字符串文本 + 单行)」不成立，实际界 = 上传上限 × 压缩比 × 常数。定性：残留放大面 + 声明失实，**非回归**——旧 DOM 路径在同等输入上还要再加全量 DOM，新路径严格更优；数据行维度的 5000 契约已正确有界并实测。改进方向（超出本轮范围）：中止计数覆盖全部 row 元素，或改用磁盘临时文件 open(File)。
3. **Minor N-3 · inlineStr 富文本 run 合并丢失（两编码通道行为不一致）**：`<is><r><t>A</t></r><r><t>B</t></r></is>` 场景每个 `<t>` 开始即重置缓冲 → 仅保留最后 run（"B"）；sharedStrings 通道则正确拼接为 "AB"。inline 富文本为罕见编码（Excel 走 sharedStrings，Hutool 写纯 inlineStr），实际暴露面极小，但属自实现解析器与旧 DOM 取值的行为分歧。
4. **Minor N-4 · sharedStrings phonetic run（`<rPh>`）被并入取值**：假名注音文本会追加到单元格值（旧 POI DOM 路径剥离 phonetic——行为分歧）；实现注释已自认该取舍（POI/Hutool 产物不含 phonetic、Excel 缺省关闭），中文环境暴露面极小，属已知接受风险，记录在案。

## 三、边界说明（不计发现）

- r 属性非法（如 "x"）→ NumberFormatException 裸透传 → 兜底 500「系统繁忙」——与损坏文件同桶（deferred Minor 2 语义），不劣于旧路径。
- 单元格无 r 属性（ECMA-376 可选）→ 该格被丢弃（fail-closed：MISSING_REQUIRED 或整行被判空跳过）；主流写入方（Excel/POI/Hutool/openpyxl/xlsxwriter）均写 r，暴露面近零。
- @RepeatSubmit 与新中止路径交互：BusinessException 走 AfterThrowing 释放锁——over-limit 路径的锁释放较旧路径（服务层正常返回不释放）反而改善，未恶化 deferred Minor 1。
- deferred Minor 4（空白行）逐例核验未被恶化：空格行 trim 后 "" 非空 → 不跳过 → 必填失败（与旧一致）；纯空串行归一 null → 跳过（与旧 ignoreEmptyRow 丢弃口径一致）。
- 工作区存在未提交的并行任务改动（TASK-0010/0011 模板引用校验器：IotDataDictService/Impl 增量方法 + DictCodeStatus 新文件）——不在 9045f38 评审范围，与导入解析路径无交集。
- 报告数字闭环核验通过：控制器测试 9→12、解析器新测 7 例、并行道 +4（OperateLogAspectTest，TASK-0075）、1229+3+7+4=1243 与 §7.4 一致。

## 四、判定理由（一句）

三条 Important 的修复在代码、测试、dev 实测三面互相印证，自实现解析器的资源关闭、XXE 防护（jar 常量池实证）、异常透传（全链路实测）与替换完整性四项审计全部通过，Hutool→POI 偏离按「意图=有界流式已达成」判定符合；新发现仅 4 条 Minor（1 文档残留 + 3 解析器边缘/声明失实），无 Important 级新缺陷。
