# 代码评审报告 · TASK-0070 批次一（diff 评审）

- 评审者：code-reviewer（fresh，唯一输入：`TASK-0070-review-batch1-brief.md` 及其「必读」清单 + 工作区源码）
- 评审对象：`f195b58..d02ac44`（8 文件，全落 `frontend/src/**`）；评审时已复核工作区最终态与 `d02ac44` 提交逐字一致（`git diff d02ac44 -- <8文件>` 为空）
- 口径：L1-execute §13「diff 评审」；前端三门（build:test/lint/typecheck 0 error）与联调冒烟由执行者实测（`TASK-0070-report-batch1.md`），评审者按 brief 不复跑构建，采信其原始输出
- 后端契约核对源（评审者实读，d02ac44 与 HEAD 间这些文件无变更）：`IotPublishController.java`、`PublishLogListVO/PublishLogRowVO/ResendForm/IotConfigVersion`、`IotCredentialController.java`、`CredentialItemVO/CredentialViewVO/CredentialWriteForm/CredentialTypes`、`IotPointController.java`（导入三接口）、`PointImportReceipt/PointImportRowResult`、`ResultCode`、`IotCredentialServiceImpl`（rotate/revoke 语义）、`PointImportExcelSaxParser`（格式支持）、sys_menu 种子 SQL

---

## 一、结论

**通过**（无 Critical /无 Important；Minor 7 条见第三节，均不阻塞本批验收）。

批次一三页组（下发状态/批量导入向导/凭据管理）与后端已交付接口的契约逐端点核对一致；凭据安全口径（列表无密文、明文唯一出口 + 权限隐藏 + 二次确认 + 关闭即弃）成立；组件/请求封装/权限交互与 0056 既有范式一致；执行者报告的两处 `defineExpose` 冒烟修复与偏离登记复核属实。

---

## 二、评审重点逐项核对（brief 7 项）

### 1. API 契约一致性 —— 逐端点核对，全部对齐

| 前端调用（文件:行） | 后端端点与形态（实读） | 判定 |
| --- | --- | --- |
| `publish-api.ts:14` GET /iot/publish/list（pageNum/pageSize/scopeType/scopeId） | `IotPublishController.java:67-74` @RequestParam scopeType/scopeId 必传，Result\<Page\<IotConfigVersion\>\> | ✓ 前端未选范围/未选网关不发请求（`publish/index.vue` ajaxQuery 前置卫语句）；scopeType=2 → scopeId 归一 0，与契约 §1.8 一致 |
| `publish-api.ts:22` GET /iot/publish/log/list（{versionId}） | `IotPublishController.java:113-120` versionId 或三元组二选一；Result\<PublishLogListVO\> | ✓ 响应字段 versionId/scopeType/scopeId/version/publishState/rows 与 `PublishLogListVO.java:34-50` 逐字段一致；前端取 `vo.rows`/`vo.publishState` |
| 行列定义 `publish-log-drawer.vue:97-135`（targetType/targetId/ackStatus/ackTime/errorMsg，rowKey=id） | `PublishLogRowVO.java:27-52`（id/versionId/targetType/targetId/ackStatus/ackTime/errorMsg/createTime/timedOut） | ✓ 列全部落在 VO 字段内；rowKey=id 存在 |
| ackStatus 徽标 0 待确认橙 / 1 已生效绿 / 2 失败红（`publish-log-drawer.vue:63-66`） | VO 注释「回执 0 待确认 1 生效 2 失败」 | ✓ 三态映射逐字一致 |
| timedOut 标红：`record.timedOut === true` 加红「已超时」标（:67） | `PublishLogRowVO.java:51` 派生字段仅 ack_status=0 且超窗为 true——红标只可能 pairing 在「待确认」橙标旁 | ✓ 派生口径一致（不改行、查询侧即时标红） |
| publishState 汇总 0 下发中橙 / 1 全部生效绿 / 2 存在失败红（:39-43） | `PublishLogListVO.java:27-32` 三常量（含空行集→0） | ✓ 语义逐字一致 |
| targetType 1 网关 / 2 接入 / 3 计算（:93-100）；scopeType=1 网关 / 2 计算（:134-141） | VO「目标 1 网关 2 接入 3 计算」「范围 1 网关 2 计算（2=0 租户粒度）」 | ✓ |
| `publish-api.ts:29` POST /iot/publish/resend（{versionId, targetId}） | `IotPublishController.java:126-130` + `ResendForm.java`（versionId 可空二选一、targetId @NotBlank） | ✓ 请求体合法；已生效行前端置灰不发请求（:54-57），后端 400 兜底双保险 |
| 版本列表列 id/version/snapshotHash/publishStatus/publishTime | `IotConfigVersion.java:49-75`（publishStatus 1 已发布 2 已回滚） | ✓ 徽标映射一致（2 本期不产生，前端预留橙标无害） |
| `credential-api.ts` 四端点 | `IotCredentialController.java:62-96` | ✓ list channelId 必传 / POST CredentialWriteForm{channelId,credType,credential} / view/{id} / DELETE /{id} 路径、方法、参数全对齐 |
| credType 四值字典（`channel-credential-drawer.vue` CRED_TYPE_OPTIONS） | `CredentialTypes.java:30-31` ALL 四值 | ✓ 逐字一致 |
| 写入表单 credential maxLength=1000 | `CredentialWriteForm.java:39` @Size(max=1000) | ✓（antd maxLength 与 Java String.length 同为 UTF-16 计数） |
| status 徽标 1 启用绿 / 其余已吊销红 | VO「0 禁用（已吊销）1 启用」 | ✓ |
| view 响应取 `data.credential` | `CredentialViewVO.java:34` | ✓ 字段名一致 |
| `point-api.ts` 导入三接口 | `IotPointController.java:174-233` | ✓ multipart 参数名 `file` 与 @RequestParam("file") 一致；模板 GET 下载；error-report POST @RequestBody PointImportReceipt |
| 导入回执形态（`import/index.vue` PointImportReceipt/RowResult interface） | `PointImportReceipt.java:22-34`（totalCount/successCount/failCount/persisted/rows）、`PointImportRowResult.java:63-75`（rowIndex/pointCode/success/reasonCode/message，成功行 reasonCode=null） | ✓ 逐字段一致；行号=Excel 物理行号语义一致（冒烟行 2/3/4 与后端口径吻合） |
| 整批拒绝形态：catch 取 `e.data`（body）、`body.data`（回执） | 控制器返回 `Result.error(PARAM_ERROR=400, msg)` + `setData(receipt)`；axios 拦截器对 code!==200 reject AxiosResponse（body 在 `.data`）、silentError 静默 toast | ✓ 与 `frontend/src/lib/axios.ts:88-108` 实现逐行核对成立；超限 fail-fast 走 `BusinessException(400)` 无回执 → 前端 else 分支 `message.error(body?.message)` 兜底（偏离登记 3），与控制器 fail-fast 抛出路径（`IotPointController.java:268-273`）吻合 |

### 2. 凭据安全口径 —— 成立

- 列表仅渲染 credentialMask（服务端掩码），前端无任何解密/拼接密文逻辑；后端 `CredentialItemVO` 结构上无密文承载字段——页面全形态（列表/确认框/成功提示）无密文无明文。
- 明文唯一出口：view 按钮 `v-privilege="'iot:credential:view'"`（无权整节点移除）→ `Modal.confirm` 二次确认（文案含「高危操作/审计留痕」）→ 警示横幅弹窗 → `closeViewModal` 清空 `viewPlaintext`（前端状态不驻留）。抽屉 `close()` 亦调用 `closeViewModal()`，写表单明文随 close/cancel 清空。
- 无 console 落明文路径；`doView` 失败分支进 otSentry 的错误对象不含明文（失败响应无明文体）。
- 权限位与 sys_menu 种子对齐：4051 iot:credential:edit / 4052 iot:credential:view（`20261122_iot_credential_domain.sql:94-95`，parent_id=4013）；凭据列表复用 iot:channel:list 与后端注解一致；「凭据」行入口挂 `v-privilege="'iot:channel:list'"`（`channel/index.vue`）。

### 3. 三态/标红/汇总/置灰 —— 一致（见第二节 1 表）；已生效行重发置灰成立（tooltip 显示性瑕疵见 R-1）

### 4. 同框架同组件 —— 一致

- 请求全部走既有 `/@/lib/axios`（getRequest/postRequest/deleteRequest/getDownload/postDownload + silentError 请求级扩展），无自造请求层。
- OtLoading / otSentry / PAGE_SIZE_OPTIONS（`/@/constants/common-const`）/ v-privilege / ot-margin-* 全为既有约定（`.ot-margin-top10` 在 `theme/ot-admin.less:52` 实存）。
- a-pagination 属性组合与 `views/iot/gateway/index.vue:92-101` 逐属性一致；pageSize:500 网关下拉与 channel 页同款（`channel/index.vue:175`）。
- 抽屉 `defineExpose({ showModal })` 与 instance-item-drawer 同范式；写操作 catch 统一 `otSentry.captureError(handleIotConflictError(e))` 与 0056 各 modal 逐字同款（`iot-optimistic-lock.ts` 登记不触发口径未变）。
- 路由/菜单：sys_menu 种子 4015→`iot/publish/index`、4016→`iot/import/index`、4013→`iot/channel/index`（`20261121_iot_console_menus.sql:46-49`）与文件路径一致，菜单动态注册下无需改 router——执行者「未动 router」口径成立。

### 5. 状态管理/内存 —— 无缺陷

- 两处 `defineExpose({ showModal })` 在最终态在位（`publish-log-drawer.vue:222`、`channel-credential-drawer.vue:321`）——执行者冒烟修复复核属实。
- 两抽屉 `close()` 全量复位（rows/tableData/versionRow/publishState/明文/表单）；无定时器、无全局监听器、无未清理副作用。
- 分页参数 pageNum/pageSize 传递正确（v-model + @change 与 0056 同款）；log/list 与凭据列表按契约不分页（每版本目标数≤3 / 每通道≤4 类型）。

### 6. 导入向导交互 —— 成立

- 逐行回执全量渲染不中断（成功+失败同表）；失败不中断时 alert/计数 tag/落库态徽标与回执字段逐一对齐。
- 「下载错误行清单」仅 `failRows.length > 0` 出现，POST 回传完整回执、后端过滤失败行渲染 xlsx（无状态回传）——与控制器实现一致。
- 整批拒绝形态（code=400 + data=回执，silentError 自渲染）与后端对齐；无回执批级拒绝（配额 4003 / 超限 fail fast）补 toast 兜底。
- 导入中 `importing` 禁用上一步/按钮 loading；超时 120000 覆盖全局 30s 兜底，适配万行批量。

### 7. 三态覆盖 / 乐观锁 —— 与 0056 一致

列表 loading/空态/错误 toast（全局拦截器统一 message.error，403/401 全局生效）+ otSentry 上报；写路径 catch 统一接 `handleIotConflictError`（登记不触发，与 0056 全部页面同款）。凭据轮换并发 5001「版本冲突」经全局 toast 可见（`ResultCode.java:32` 实读）。

### 越界反查（复核）

`git diff --name-only f195b58..d02ac44` = 8 文件，全部 `frontend/src/**`，无越界——与协调者比对一致。

---

## 三、发现清单

无 Critical、无 Important。以下 7 条 Minor：

| # | 级别 | 出处 | 说明 |
| --- | --- | --- | --- |
| R-1 | Minor | `frontend/src/views/iot/publish/components/publish-log-drawer.vue:54-56` | Tooltip 直接包裹原生 `disabled` 按钮存在 antd 已知限制（disabled 按钮不触发鼠标事件，官方要求 span 包裹），「已生效无需重发」tooltip 悬浮显示未证实——置灰本身生效、语义可达（冒烟仅断言 disabled 属性），建议批次二顺手包 span |
| R-2 | Minor | `frontend/src/views/iot/import/index.vue:40` | `accept=".xlsx,.xls"` 放行 .xls，而后端解析器仅支持 .xlsx（`PointImportExcelSaxParser` 走 OPCPackage/XSSFReader）——.xls 上传将得到解析失败通用错误；0018 dict-import-modal 同款存量口径照抄，非本批引入，可与 0018 一并登记后续修正 |
| R-3 | Minor | `frontend/src/views/iot/channel/components/channel-credential-drawer.vue:225` | 对「已吊销」行的同类型重复写入，确认文案只讲「轮换（key_version 递增、旧密文被替换）」，未提示后端 rotate 同时「复活 status 重新启用」（`IotCredentialServiceImpl.rotate` SET status 复活）——方向正确、语义不完整，轮换后行会从「已吊销」变「启用」 |
| R-4 | Minor | `frontend/src/views/iot/channel/components/channel-credential-drawer.vue:211-217` | 写入表单校验为 `message.warning` 手写，未用 0056 表单范式 `:rules` + `formRef.validate()`（channel-form-modal/tenant-form-modal 均为后者）——功能等价，但无内联红字校验提示，属范式微偏差 |
| R-5 | Minor | `frontend/src/views/iot/import/index.vue:75` | 回执表 `:pagination="false"` 全量渲染——单批上限 10000 行整批拒绝时最坏 DOM 量大（仅 y:320 滚动）；0018 dict-import-modal 同款存量口径（y:220），如后续反馈卡顿再补分页/虚拟滚动 |
| R-6 | Minor | `frontend/src/views/iot/import/index.vue:53` | 「下一步：开始导入」按钮未挂 `iot:point:add`（dragger/模板下载/错误行下载均已挂）——无该权限角色进入 4016 得到不可用空壳页（按钮因无文件恒 disabled，后端有权限拦截），纯 UX 死路 |
| R-7 | Minor | `frontend/src/views/iot/publish/components/publish-log-drawer.vue:134-141` | scopeType=1 时「发布范围」显示「网关（scopeId=N）」原始数字 id 而非网关编码/名称（版本行 VO 信封不含网关主数据，属后端形态限制下的取舍）——可读性弱，登记可选增强 |

---

## 四、对执行者报告的复核

1. **defineExpose 冒烟修复（2 处）**：最终态两文件均在位（第三节第二节 5 已核）——修复完整，无第三处遗漏（本批仅这两个抽屉组件需要对外暴露 showModal；channel/index.vue 与 publish/index.vue 的 ref 调用签名匹配）。
2. **偏离登记 1-6**：逐条与代码事实相符——凭据抽屉挂 4013 / 下发状态挂 4015 的形态落地与菜单种子（凭据无独立菜单行、4051/4052 parent_id=4013）吻合；无 emit reloadList 与 0056 tenant-admin-modal 口径一致（凭据操作不影响通道列表数据）；导入兜底 toast 仅一行文案未自造请求层。
3. **遗留清单**：快照入口未做（4071 预留）、重发 ack=0/2 与凭据写路径实际落库未走查、timedOut 未真实复现——均为执行者如实登记的验证边界，与本评审静态核对结论不冲突（对应分支的模板条件渲染经静态审查正确）。
4. **三门构建/冒烟**：按 brief 评审者不复跑、不重放，采信报告原始输出（build:test/lint/typecheck EXIT=0 + 三页走查记录）。

## 五、无法判定项

无。所有评审重点均可依据 brief「必读」+ 工作区源码 + 后端契约源判定；唯一未独立复核的是执行者实测的构建输出与冒烟截图（按 brief 口径采信，非存疑）。
