# TASK-0022 + TASK-0023 diff 评审报告（微改攒批两提交联合评审）

- 评审者：fresh reviewer（不共享协调者上下文，no-subagents，只评审不修改）
- 评审日期：2026-11-22
- diff 范围：`a961fd1..bfdc11b`（963dc71=TASK-0022 / bfdc11b=TASK-0023，23 文件，+3364/−50），diff 包全文 3915 行逐行读完
- 依据：L1-execute.md §13 + 评审 brief「依据」两样（契约一致性 + 明显质量缺陷）
- 输入：diff 包 / 实施 brief / TASK-0022.md / TASK-0023.md / TASK-0022-0023-report.md（范式基准 0018/0021 为对照参照）

## 0. 范围核对与契约符合性摘要

- **越界反查**：23 文件 = point 域主码 16 + test/iot 7，全落 brief 声明范围；redline 测试增补属 brief 授权例外。`channel/service/**` 零 diff——实施报告偏离 2 的解释经复核属实：ChannelReferenceGuard 点位计数转正已在 BASE 前置提交 `a61fd24`（TASK-0021 修订轮）就位，本批 S8 冒烟实证（删通道拒 400 + 文案提示点位编码），不构成缺交付；网关侧间接计数维持占位归 0024（brief 原文如此）。
- **TASK-0022 契约**：五类逐行校验（编码合规 [A-Za-z0-9_.]≤96 / unit 字典批查 / channelCode→id 解析含歧义拒 / min<=max / dataType+valueType 枚举）、整批不半落库（拒绝路径无先置写 + DuplicateKey 兜底回滚）、幂等（活跃行与已删占用同口径 DUPLICATE_CODE）、配额联动不豁免（校验通过行数 + PointQuotaCounter 当前计数 > quota → 4003 整批拒）、10000 行上限 SAX 流式 fail fast（服务层双保险）、错误清单无状态回传渲染、导出与 list 同源 buildListWrapper 同条件裁剪、导入落库不发 PointEnabledEvent——逐项落地且有对应测试（含 4003/幂等/10000 边界/并发兜底用例）。
- **TASK-0023 契约**：反查 idx_tenant_point 联表裁剪（仅可见实例，instanceId NULL 行 fail-closed 跳过）、详情聚合 referenceCount 与清单同裁剪口径、批量启停整批 fail-closed（防线先于影响计数，不向无权者泄露绑定面）、停用影响计数 / 启用侧恒 0 且不触达计数查询、「变更已保存，重新发布后对接入生效」明示常量、曲线 fail-soft 双层兜底（客户端 + 服务层）、st_point_num 全限定名 + tenant_id/point_code 标签过滤 + Basic 认证 + point_code 白名单防注入 + LIMIT 5000 保护——逐项落地。
- **越权回归面（联合项复核）**：详情/反查/曲线三读共用 requireReadablePoint 同一闸（403 文案不泄露点位标识）；导入行 DTO 结构性无 status/deleted/id/configVersion 注入通道；曲线 SQL 拼接面已收口（库名派生唯一入口 + point_code 白名单）。未发现越权回归。
- **事务边界（联合项复核）**：importPoints / changePointStatus 均 `@Transactional(rollbackFor=Exception)`，整批拒绝发生在任何 INSERT 之前，回滚语义与「不半落库」一致——边界干净。

## 1. TASK-0022 发现（导入导出）

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 1-1 | **Important** | `IotPointServiceImpl` importPoints/validateImportFields/buildImportEntity（对照同文件 createPoint L363-365：validateChannel+validateInstance+validateDept） | 导入对 deptId/boundInstanceId 仅做「整数格式」校验，缺单点建档与编辑都执行的存在性/同租户校验（validateDept/validateInstance）——导入可写入悬挂 id 或跨租户 id 的归属锚点（dept 绑错部门、boundInstanceId 指向不存在/他租户实例），与代码内「照 createPoint 范式」声明不符，两入口对同一业务字段契约不一致；brief 校验清单未点名这两列故不判 Critical，但属明显质量缺陷，建议补批内一次 IN 存在性校验（或按 §12 建 BUG 登记） |
| 1-2 | Minor | `IotPointServiceImpl.failRow` / `rowResultOf` | 按 rowIndex 线性扫描 rowResults，万人批最坏 O(n²)（~10^8 次比较，秒级），改为 rowIndex→result Map 索引即消除；正确性不受影响 |
| 1-3 | Minor | `IotPointServiceImpl.importPoints` 落库段 | 10000 行逐行 `this.save()`（10000 次单条 INSERT 往返同事务），MyBatis-Plus saveBatch/JDBC 批量可显著降导入时延；冒烟 1000 行已实测可行，正确性与原子性不受影响 |
| 1-4 | Minor | `IotPointController.parseImportRows` + `PointImportExcelSaxParser.parse` | 损坏/非 xlsx 上传抛 IORuntimeException 落全局 Exception 兜底（系统级 500 形态），属用户输入质量问题，宜与行数超限同口径映射 PARAM_ERROR(400) |
| 1-5 | Minor | `IotPointServiceImpl.selectExportRows` | 全量 `list()` 无分页无行数上限；有配额租户行数受控，quota=null 租户无界，异常大结果集有内存放大面 |

## 2. TASK-0023 发现（反查/批量启停/曲线）

| # | 严重级 | 出处 | 说明 |
| --- | --- | --- | --- |
| 2-1 | Minor | `PointCurveQueryClient.query24h` 行解析段 | 应答行短于 3 列时 `row.get(1)/row.get(2)` 抛 IndexOutOfBoundsException，客户端自身「一切失败返回空集」契约不完全成立，现靠服务层 queryCurve24h 的 catch(Exception) 双保险兜住（fail-soft 结果不变）；客户端内补行长度防御即闭环 |
| 2-2 | Minor | `IotPointServiceImpl.getPointDetail` | 单次详情调用 dataPermService.resolve 执行两次（requireReadablePoint 内一次 + 聚合取 scope 一次）；且详情读路径每次发起 TDengine REST 尝试（实施报告偏离 5 已登记）——TDengine 跨网段不可达时详情读最多附加 connect-timeout 3s 延迟 |
| 2-3 | Minor | `IotPointServiceImpl.changePointStatus` 更新循环 | 批量启停仍逐行 updateById（未合并单条 UPDATE ... IN），且同态 no-op 行也发 UPDATE、启用侧 no-op 行仍发事件（报告 §5.3 已登记）；逐行形态系 0021 已验收遗留、本批仅改响应体，登记待后续优化不阻塞本批 |

## 3. 联合发现

无新增 Critical/Important。联合审查项（SAX 解析复用质量/批量 UPDATE 效率/事务边界/越权回归面）结论见 §0：SAX 解析器与字典解析器结构同源且三要点（r 属性行号/缺格 null/超限 fail fast）继承到位，独立实现的理由（泛化改造字典域文件越界）成立；事务边界干净；越权面无回归。

## 4. 分级汇总

- **Critical：0**
- **Important：1**（TASK-0022 §1-1 导入缺 deptId/boundInstanceId 存在性/同租户校验——与单点建档契约不一致）
- **Minor：7**（TASK-0022：4；TASK-0023：3；联合：0）

处理建议：Important 1 条走 §12 退回/建 BUG 路径；Minor 7 条记 TASK `任务结果` deferred（收尾进 LOG），不进修订轮。
