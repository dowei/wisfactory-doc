# TASK-0009 diff 评审报告（fresh reviewer）

- 评审对象：`TASK-0009-diff.txt`（单提交 e30de0a，11 文件）；实施 brief / TASK 详情 / 实施报告 / 既有 IotDataDictServiceImpl 前批范式 / L1-execute §13 口径为评审依据。
- 结论速览：**Critical 0 / Important 3 / Minor 6**。与 brief 契约的核心裁决（整批原子性、回执五分类语义、4002/400 口径、批内互撞先行、库内一次 IN 预检含禁用行占码、maxLimit 一行 + 全量回归、菜单 4023 与守卫闭环、Excel 解析对 SysUserController 范式机械对齐）逐项核验通过；发现集中在「Excel 解析的真实行为」与「上传防线」两面。
- 评审者独立实证：本机 jshell（Hutool 5.8.25 + POI 5.2.5，与工程依赖一致）对 `ExcelUtil.getReader(...).read(1)` 做了 4 组探针（A/B/C/D），未修改任何仓库文件，结论已进对应发现。

## 一、契约一致性核验（通过项，简列）

| 核验点 | 结论 |
| --- | --- |
| 整批原子性（任一失败 0 INSERT） | 通过：两级拒绝（批级 paramError / 行级 DictImportRejectException）均发生在 save 循环之前，事务内无任何残留写；DuplicateKeyException 兜底经 rollbackFor=Exception 整批回滚。单测（never insert）与冒烟 S4 双证 |
| 回执五分类 | 通过：MISSING_REQUIRED / TOO_LONG / INVALID_DATA_TYPE / INVALID_CATEGORY / DUPLICATE_CODE 与 brief 裁决一致，批内互撞与库内已有同报 DUPLICATE_CODE |
| 批内互撞先于落库 | 通过：firstSeenRow 内存判定先于任何 INSERT，首现放行、次现指向首现行号 |
| 库内预检一次 IN（含已禁用行占码） | 通过：selectExistingCodes 单条 IN（≤5000 码），@TableLogic 过滤已删行、不过滤 status → 已禁用行占码，与单接口 insertDict 预检同口径 |
| 4002/400 口径区分 | 通过：批量全路径统一 PARAM_ERROR(400)（含并发 DuplicateKey 兜底），4002 仅存于未改动的单接口 duplicateDataCode |
| maxLimit 顺带项 | 通过：MybatisPlusConfig 仅 `setMaxLimit(500L)` 一处；全仓 grep 确认无内部硬编码 >500 的 Page 用法；回归 1229 例绿 + dev/prod 双实证 |
| 菜单 4023 与守卫闭环 | 通过：SQL 与 20261121 脚本范式逐列一致（parent 4008、menu_type=3、sort=6 续号、INSERT IGNORE、(1,4023) 授权）；SaPermissionRegistryConsistencyTest 全绿 |
| SysUserController 范式对齐 | 通过（机械层面）：read(1) / Convert.toStr 按位提取 / writeExcel 与 export 逐行同构，且新增 trim 与说明行裁剪为合理增强 |

## 二、发现清单

### Important

1. **Important · `IotDataDictController.parseImportRows`（回执行号物理对齐契约破坏）**：Hutool `read(1)` 默认 `ignoreEmptyRow=true` 逐字节码实证（ListSheetReader：空行整行跳过、不产占位记录），而控制器按「返回序」顺序分配 rowIndex 并自述「行号保持 Excel 物理行号」——含中间空行/未触达行的文件中，空行之后的行回执行号整体前移（实测：header+数据+空行+数据 → 仅返回 2 行），回执与错误清单会把用户指到错误的 Excel 行；报告遗留 4 声称该对齐已被「含中间空行的控制器测试实证」不成立（testImport_ParsesExcelRows 夹具无空行、testImport_TrimsCellsAndKeepsBlankRows 名实不符——夹具亦无空行），任务验收标准「100 行含错文件逐行回执」在含空行文件上失真。
2. **Important · `IotDataDictController.parseImportRows`（首行缺格单元格被填成 Excel 列名字母，必填缺失对首行失效并可静默落库垃圾码）**：`read(1)` 携带 `aliasFirstLine=true`，首个返回行（Excel 第 2 行）经 aliasHeader 处理时 null 单元格被替换为列名（实测：缺中间格返回 `[CODE_X, B, ELEC]`）——用户在模板首行填编码漏名称时 data_name 得 "B"、漏编码时 data_code 得 "A"，必填缺失（MISSING_REQUIRED）对该行永不触发，且 "A"/"B" 能通过全部校验静默落库成真实字典条目；字典编码「只增不删」语义下垃圾码被永久占用，收尾需按回执修正的用户预期被双重破坏。
3. **Important · `IotDataDictController.importDict`/`parseImportRows`（上传防线与内存放大面）**：单批 5000 行上限在**全量 XSSF DOM 解析完成之后**才生效，解析前无任何行数/内容防线，唯一硬限制是既有 multipart `max-file-size: 50MB`（application.yml，先于本任务存在）+ MaxUploadSizeExceededException 兜底文案；50MB 合法压缩比内的 xlsx（POI ZipSecureFile 阈值内）经 DOM 放大可达 GB 级堆占用（JVM 级 OOM 影响全进程），无流式（SAX/分块）解析或预解析防御——与 brief 重点②「Excel 解析的内存放大面（超大文件/恶意构造）」直接对应；缓解因素如实记录：鉴权 + iot:dict:import 权限门 + @RepeatSubmit + 50MB 上限 + 与 SysUserController 同范式（既有面）。

### Minor

1. **Minor · `IotDataDictController.importDict` + `RepeatSubmitAspect`（@RepeatSubmit 语义）**：切面参数指纹排除 MultipartFile → 上传场景键退化为「用户+端点」，3 秒内换任意文件也互斥（沿 SysUserController 范式）；且「回执拒绝」路径控制器 catch 后正常返回、@AfterThrowing 不触发、锁不释放（异常路径才释放）——用户按回执修正后 3 秒内重导会被「不允许重复提交」误拦。
2. **Minor · `importDict`（无文件类型防线）**：不校验扩展名/Content-Type/魔数，非 xlsx/xls 或损坏文件经 POI/Hutool 异常落入 GlobalExceptionHandler 兜底 → HTTP 200 + code 500「系统繁忙」，无专项文案，用户无从区分「格式不对」还是「系统故障」。
3. **Minor · `IotDataDictServiceImpl.isSkippableRow`（魔法前缀静默吞行）**：真实业务编码若以「说明：」开头（导入侧无禁止，单点新增接口可造出该码），导入时被静默跳过——不进回执、不计 totalCount，与实施报告偏离 6 反对示例行魔法前缀的理由自相矛盾。
4. **Minor · `IotDataDictServiceImpl.validateFields`（纯空白字符行当数据行）**：空行裁剪只按 null 判定，含空白字符的行经 cell() trim 后 dataCode="" 非 null → 不被跳过 → 必填缺失失败行 → 单条空格行即可整批拒绝，与「空行不计入」的口径冲突。
5. **Minor · `IotDataDictServiceImpl.importDicts` 落库循环（写路径逐行 save）**：≤5000 次单条 INSERT 往返（读侧已优化为一次 IN，写侧未用 saveBatch 批量化），事务时长与远程库场景下延迟放大。
6. **Minor · `IotDataDictController.importErrorReport`（回执 JSON 无上限）**：请求体行数/大小不设防，超大回执可放大内存/CPU（BigExcelWriter 流式写出与鉴权+权限门缓解了主面，残留面为模版下载共用的 writeExcel 在写出中途异常时响应头已提交、错误体与 xlsx 字节混写）。

## 三、无法判定 / 边界说明

- Excel 数值单元格转码：实测 `1001`（Integer）与 `1002.0`（Double）均转 "1001"/"1002"，无 ".0" 垃圾后缀问题，不构成发现。
- 实施报告的环境事故恢复（R7）与 prod 启动机械问题（R8）属执行过程记录，不在 diff 评审口径内，未核；报告其余验收输出与 diff 交叉印证一致。
