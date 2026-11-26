# TASK-0097 实施报告：点位管理页前端补齐

## 1 改动文件清单

BASE：4e4c1461；提交：449d7eb3（main，未 push）。改动文件 7 个，均在 brief 声明范围内：

| 文件 | 状态 | 说明 |
| --- | --- | --- |
| frontend/src/api/iot/point-api.ts | 改 | 补 getById/detail/references/curve/add/update/changeStatus/remove/export；导入三方法与 queryList 保留 |
| frontend/src/components/iot/iot-consts.ts | 改 | 新增 IOT_POINT_DATA_TYPE_OPTIONS / iotPointDataTypeLabel / IOT_POINT_VALUE_TYPE_OPTIONS（brief 允许复用的字典常量） |
| frontend/src/views/iot/point/index.vue | 改 | 替换 IotPlaceholder：列表分页/8 项过滤/权限按钮/新增编辑启停删除导出/导入向导跳转 |
| frontend/src/views/iot/point/components/point-form-modal.vue | 新 | 表单弹窗：校验对齐 IotPointCreateForm/UpdateForm；pointCode 编辑锁定；status 不入提交；defineExpose(showModal,onSubmit) |
| frontend/src/views/iot/point/components/point-detail-modal.vue | 新 | 详情弹窗：detail/references/curve 三读，聚合 referenceCount，曲线/反查空集诚实提示；defineExpose(showModal) |
| frontend/tests/unit/api/iot/point-api.spec.ts | 新 | 11 例：每方法 URL/HTTP 动词/参数逐项对齐 IotPointController |
| frontend/tests/unit/views/iot/point.spec.ts | 新 | 9 例：占位移除/加载/空态/defineExpose 父 ref/启停/删除/导出/表单 payload 白名单/详情三读空态 |

未触碰 backend / calc / ingest / contract / metric 页 / TASK-0095。

## 2 每条验收命令的原始输出

执行命令：`cd D:/workspace/WisFactory/frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`（四条链式一次跑完，整体 exit 0）。
说明：以下为原始 stdout/stderr 去除 ANSI 颜色转义后的逐字文本（颜色码不影响内容）。完整原始日志：`C:\Users\dowil\AppData\Local\Temp\task0097-final-gates.log`。

### 2.1 `pnpm build:test`（exit 0）

```text
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
transforming...

./icon.svg referenced in D:/workspace/WisFactory/frontend/node_modules/.pnpm/vue3-json-viewer@2.3.1_vue@3.4.27_typescript@5.6.3_/node_modules/vue3-json-viewer/dist/index.css didn't resolve at build time, it will remain unchanged to be resolved at runtime
[plugin:vite:esbuild] [plugin vite:esbuild] src/views/survey/components/FormAuthModal/panel.vue?vue&type=script&setup=true&lang.ts: This assignment will throw because "editors" is a constant
244|                _createVNode(AuthMemberEditor, {
245|                  "member-ids": editors,
246|                  "onUpdate:memberIds": _cache[1] || (_cache[1] = ($event: any) => ((editors) = $event)),
   |                                                                                     ^
247|                  title: "编辑名单",
248|                  tip: "名单内成员可编辑该问卷（含查看）；支持按用户、角色、部门授权",

[plugin:vite:esbuild] [plugin vite:esbuild] src/views/survey/components/FormAuthModal/panel.vue?vue&type=script&setup=true&lang.ts: This assignment will throw because "viewers" is a constant
254|                _createVNode(AuthMemberEditor, {
255|                  "member-ids": viewers,
256|                  "onUpdate:memberIds": _cache[2] || (_cache[2] = ($event: any) => ((viewers) = $event)),
   |                                                                                     ^
257|                  title: "查看名单",
258|                  tip: "名单内成员仅可查看该问卷（含数据与统计）；编辑名单成员已含查看权",

[plugin:vite:esbuild] [plugin vite:esbuild] src/views/business/oa/enterprise/components/enterprise-employee-list.vue?vue&type=script&setup=true&lang.ts: This assignment will throw because "columns" is a constant
298|                class: "ot-margin-bottom5",
299|                modelValue: columns,
300|                "onUpdate:modelValue": _cache[1] || (_cache[1] = ($event: any) => ((columns) = $event)),
   |                                                                                    ^
301|                tableId: _unref(TABLE_ID_CONST).BUSINESS.OA.ENTERPRISE_EMPLOYEE,
302|                refresh: queryEmployee

✓ 6013 modules transformed.
rendering chunks...
warnings when minifying css:
▲ [WARNING] Unexpected "2of" [css-syntax-error]

    <stdin>:1:99250:
      1 │ ...ntents}.tox .tox-ai :nth-child(2of.tox-ai__response-status-icon)...
        ╵                                   ~~~


[plugin:vite:reporter] [plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/src/api/system/login-api.ts is dynamically imported by D:/workspace/WisFactory/frontend/src/router/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/layout/components/change-password/regular-change-password-modal.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/layout/components/header-user-space/header-avatar.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/main.ts, D:/workspace/WisFactory/frontend/src/views/system/account/components/center/index.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/views/system/login3/login.vue?vue&type=script&setup=true&lang.ts, dynamic import will not move module into another chunk.

[plugin:vite:reporter] [plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/src/plugin/admin-ui/index.ts is dynamically imported by D:/workspace/WisFactory/frontend/src/router/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/main.ts, dynamic import will not move module into another chunk.

[plugin:vite:reporter] [plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/node_modules/.pnpm/ant-design-vue@4.2.5_vue@3.4.27_typescript@5.6.3_/node_modules/ant-design-vue/dist/reset.css is dynamically imported by D:/workspace/WisFactory/frontend/src/plugin/admin-ui/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/main.ts, dynamic import will not move module into another chunk.

computing gzip size...
dist/index.html                                                                                           0.77 kB │ gzip:   0.48 kB
dist/png/ot-admin-logo-white-Dn8J4Vek.png                                                                 8.11 kB
dist/png/appearset_bgc_big-CBsi9EZh.png                                                                   8.61 kB
dist/png/ot-admin-logo-cDl7Ur4Y.png                                                                      14.33 kB
dist/png/99d0-BLQr3eGW.png                                                                               37.33 kB
dist/png/admin-logo-icon-BJtvcrOo.png                                                                    49.41 kB
dist/png/login-illustration-9HORgnQJ.png                                                                235.70 kB
dist/png/login-background-Dk6HcIGp.png                                                                  760.38 kB
dist/css/index-i0u-c3z8.css                                                                               0.04 kB │ gzip:   0.05 kB
dist/css/DownloadFile-DAvaQD3e.css                                                                        0.05 kB │ gzip:   0.07 kB
dist/css/index-D6rOUVPI.css                                                                               0.05 kB │ gzip:   0.07 kB
dist/css/index-CdH8jf3y.css                                                                               0.05 kB │ gzip:   0.07 kB
dist/css/publish-log-drawer-s6nV0jMT.css                                                                  0.05 kB │ gzip:   0.07 kB
dist/css/index-CnywH5a8.css                                                                               0.05 kB │ gzip:   0.07 kB
dist/css/index-CTnqawDD.css                                                                               0.05 kB │ gzip:   0.07 kB
dist/css/ProjectForm-KVnI2Kwr.css                                                                         0.05 kB │ gzip:   0.07 kB
dist/css/iot-placeholder-Bmzk0EMC.css                                                                     0.06 kB │ gzip:   0.08 kB
dist/css/instance-item-drawer-DDYU_lFS.css                                                                0.06 kB │ gzip:   0.08 kB
dist/css/index-SkaSDKcc.css                                                                               0.06 kB │ gzip:   0.08 kB
dist/css/follow-DtSFEDAs.css                                                                              0.06 kB │ gzip:   0.08 kB
dist/css/index-rU0VK28O.css                                                                               0.07 kB │ gzip:   0.09 kB
dist/css/index-Bv8IxLq4.css                                                                               0.07 kB │ gzip:   0.09 kB
dist/css/ImgList-B6haYOiW.css                                                                             0.07 kB │ gzip:   0.09 kB
dist/css/apiclient-secret-modal-Dd9zq1y6.css                                                              0.08 kB │ gzip:   0.10 kB
dist/css/index-DHSD_xKM.css                                                                               0.08 kB │ gzip:   0.10 kB
dist/css/index-B-eu0trW.css                                                                               0.09 kB │ gzip:   0.10 kB
dist/css/notice-detail-modal-CfRr7vB7.css                                                                 0.09 kB │ gzip:   0.10 kB
dist/css/apiclient-form-modal-BLXq3tyN.css                                                                0.10 kB │ gzip:   0.11 kB
dist/css/table-SRAIJii3.css                                                                               0.11 kB │ gzip:   0.11 kB
dist/css/index-BYAU4CBC.css                                                                               0.11 kB │ gzip:   0.11 kB
dist/css/index-C2VZgmO1.css                                                                               0.11 kB │ gzip:   0.11 kB
dist/css/index-YhouMItC.css                                                                               0.12 kB │ gzip:   0.12 kB
dist/css/index-kgmyyiGO.css                                                                               0.12 kB │ gzip:   0.13 kB
dist/css/notice-form-drawer-CXtGDo1N.css                                                                  0.12 kB │ gzip:   0.10 kB
dist/css/publish-confirm-modal-CX1YV7gg.css                                                               0.12 kB │ gzip:   0.12 kB
dist/css/code-generator-table-config-form-eeOT1m_L.css                                                    0.12 kB │ gzip:   0.10 kB
dist/css/help-doc-form-drawer-BxBxRTJR.css                                                                0.12 kB │ gzip:   0.10 kB
dist/css/code-generator-table-config-form-delete-DbyNiuDr.css                                             0.13 kB │ gzip:   0.12 kB
dist/css/index-DH5T8DG5.css                                                                               0.13 kB │ gzip:   0.12 kB
dist/css/index-CrZqPORT.css                                                                               0.13 kB │ gzip:   0.13 kB
dist/css/channel-detail-modal-CwLvL1Y3.css                                                                0.13 kB │ gzip:   0.13 kB
dist/css/import-CfDmKo2w.css                                                                              0.14 kB │ gzip:   0.13 kB
dist/css/BaseTable-B-tnrLUs.css                                                                           0.14 kB │ gzip:   0.13 kB
dist/css/index-Cpqpe2cV.css                                                                               0.14 kB │ gzip:   0.13 kB
dist/css/index-CM2MpC6G.css                                                                               0.14 kB │ gzip:   0.13 kB
dist/css/index-C-uHsV3_.css                                                                               0.15 kB │ gzip:   0.14 kB
dist/css/index-DQWCfq80.css                                                                               0.15 kB │ gzip:   0.12 kB
dist/css/index-Czq7odZB.css                                                                               0.15 kB │ gzip:   0.14 kB
dist/css/pie-GiX0BmiW.css                                                                                 0.15 kB │ gzip:   0.13 kB
dist/css/gauge-BlkLKyTy.css                                                                               0.15 kB │ gzip:   0.14 kB
dist/css/keyword-match-modal-B7yCxDP5.css                                                                 0.15 kB │ gzip:   0.13 kB
dist/css/category-BKKxPDYF.css                                                                            0.16 kB │ gzip:   0.14 kB
dist/css/gradient-ublseZuy.css                                                                            0.16 kB │ gzip:   0.14 kB
dist/css/code-generator-table-config-form-basic-CmKoTgJy.css                                              0.16 kB │ gzip:   0.14 kB
dist/css/menu-operate-modal-CROAn4FB.css                                                                  0.16 kB │ gzip:   0.15 kB
dist/css/module-form-BppZTgmd.css                                                                         0.16 kB │ gzip:   0.15 kB
dist/css/index-BCQNDPW4.css                                                                               0.16 kB │ gzip:   0.15 kB
dist/css/enterprise-employee-list-y_g8Pw_X.css                                                            0.17 kB │ gzip:   0.14 kB
dist/css/enterprise-operate-modal-BvDlyYS9.css                                                            0.17 kB │ gzip:   0.14 kB
dist/css/code-generator-table-config-form-insert-and-update-C-D2A4nJ.css                                  0.17 kB │ gzip:   0.14 kB
dist/css/index-BRnrwGM4.css                                                                               0.18 kB │ gzip:   0.15 kB
dist/css/help-doc-manage-list-Db1ISVYa.css                                                                0.18 kB │ gzip:   0.14 kB
dist/css/analysis-CQAzIk2d.css                                                                            0.18 kB │ gzip:   0.13 kB
dist/css/index-CnamIkRN.css                                                                               0.18 kB │ gzip:   0.15 kB
dist/css/index-V20uJDbO.css                                                                               0.21 kB │ gzip:   0.18 kB
dist/css/index-BQdngwWb.css                                                                               0.22 kB │ gzip:   0.15 kB
dist/css/index-Dp2Qu0a6.css                                                                               0.23 kB │ gzip:   0.14 kB
dist/css/TextListModal-lOMN9N1-.css                                                                       0.23 kB │ gzip:   0.14 kB
dist/css/goods-list-DOfkcpfl.css                                                                          0.25 kB │ gzip:   0.17 kB
dist/css/index-BKN7Vqi-.css                                                                               0.25 kB │ gzip:   0.15 kB
dist/css/index-BM5H80w0.css                                                                               0.25 kB │ gzip:   0.15 kB
dist/css/channel-form-modal-BtyQPGhT.css                                                                  0.25 kB │ gzip:   0.16 kB
dist/css/config-operate-drawer-CMQS0JWH.css                                                               0.25 kB │ gzip:   0.21 kB
dist/css/changelog-card-BIZYkilz.css                                                                      0.25 kB │ gzip:   0.19 kB
dist/css/template-send-detail-drawer-Iw0qJ7aB.css                                                         0.26 kB │ gzip:   0.16 kB
dist/css/index-C0GpnHaV.css                                                                               0.26 kB │ gzip:   0.17 kB
dist/css/default-home-card-BOGSXst4.css                                                                   0.26 kB │ gzip:   0.18 kB
dist/css/index-BnB06Whl.css                                                                               0.27 kB │ gzip:   0.15 kB
dist/css/preview-DVqL3VoK.css                                                                             0.27 kB │ gzip:   0.18 kB
dist/css/chart-CXuTlDAW.css                                                                               0.28 kB │ gzip:   0.17 kB
dist/css/role-tree-menu-Dnqv_GrS.css                                                                      0.28 kB │ gzip:   0.21 kB
dist/css/index-yMfMkHJV.css                                                                               0.29 kB │ gzip:   0.18 kB
dist/css/index-DQHmgcs_.css                                                                               0.29 kB │ gzip:   0.17 kB
dist/css/channel-credential-drawer-wEyPghzB.css                                                           0.29 kB │ gzip:   0.19 kB
dist/css/index-WxN6EUWJ.css                                                                               0.29 kB │ gzip:   0.19 kB
dist/css/index-Iei4Wp0G.css                                                                               0.33 kB │ gzip:   0.18 kB
dist/css/index-CCeTqU-9.css                                                                               0.33 kB │ gzip:   0.16 kB
dist/css/snapshot-drawer-coJc1-CA.css                                                                     0.33 kB │ gzip:   0.20 kB
dist/css/FormQrcode-zDibsgKU.css                                                                          0.34 kB │ gzip:   0.25 kB
dist/css/index-CEUAXvQU.css                                                                               0.34 kB │ gzip:   0.22 kB
dist/css/help-doc-view-record-list-DXe94vIw.css                                                           0.34 kB │ gzip:   0.14 kB
dist/css/panel-CzgXg4MK.css                                                                               0.35 kB │ gzip:   0.18 kB
dist/css/DataPush-BWg8P04L.css                                                                            0.37 kB │ gzip:   0.21 kB
dist/css/cross-DYHCe1z7.css                                                                               0.37 kB │ gzip:   0.19 kB
dist/css/help-doc-catalog-tree-5cONHUj1.css                                                               0.37 kB │ gzip:   0.19 kB
dist/css/notice-detail-CPlxUPVT.css                                                                       0.39 kB │ gzip:   0.21 kB
dist/css/MatrixView-BQ8L2aDx.css                                                                          0.41 kB │ gzip:   0.24 kB
dist/css/home-notice-B12SMd0n.css                                                                         0.41 kB │ gzip:   0.24 kB
dist/css/index-BqHga3su.css                                                                               0.42 kB │ gzip:   0.25 kB
dist/css/index-DCfFfRkA.css                                                                               0.42 kB │ gzip:   0.21 kB
dist/css/dept-tree-checkbox-HxZiRqXx.css                                                                  0.42 kB │ gzip:   0.21 kB
dist/css/layout-const-CZJ-siCP.css                                                                        0.43 kB │ gzip:   0.25 kB
dist/css/index-kaqxmmba.css                                                                               0.43 kB │ gzip:   0.20 kB
dist/css/TextCopy-DuZngPZh.css                                                                            0.43 kB │ gzip:   0.26 kB
dist/css/DataFilter-CGvhUU-b.css                                                                          0.43 kB │ gzip:   0.19 kB
dist/css/template-send-create-modal-BpQLZrJw.css                                                          0.44 kB │ gzip:   0.24 kB
dist/css/operate-log-detail-modal-D1pNfz_S.css                                                            0.46 kB │ gzip:   0.24 kB
dist/css/member-editor-CINJ5hXC.css                                                                       0.52 kB │ gzip:   0.20 kB
dist/css/index-C7jUeuFy.css                                                                               0.57 kB │ gzip:   0.29 kB
dist/css/index-CTahEBHb.css                                                                               0.60 kB │ gzip:   0.30 kB
dist/css/home-to-be-done-DHoODtdh.css                                                                     0.60 kB │ gzip:   0.31 kB
dist/css/index-Cfspf8pS.css                                                                               0.62 kB │ gzip:   0.29 kB
dist/css/iot-formula-editor-Bujj7HnT.css                                                                  0.63 kB │ gzip:   0.31 kB
dist/css/ViewData-BpDSevMt.css                                                                            0.65 kB │ gzip:   0.30 kB
dist/css/FileList-CWWNXZQD.css                                                                            0.65 kB │ gzip:   0.32 kB
dist/css/index-BENQLd1l.css                                                                               0.70 kB │ gzip:   0.31 kB
dist/css/notice-employee-detail-CkXuZHTQ.css                                                              0.71 kB │ gzip:   0.28 kB
dist/css/public-B-i-NxjN.css                                                                              0.74 kB │ gzip:   0.35 kB
dist/css/home-header-BMGP0g5m.css                                                                         0.75 kB │ gzip:   0.32 kB
dist/css/display-factory-card-CnkDoY9p.css                                                                0.76 kB │ gzip:   0.31 kB
dist/css/display-slot-card-CEjwz2CP.css                                                                   0.76 kB │ gzip:   0.33 kB
dist/css/create-DvTvq8Xh.css                                                                              0.77 kB │ gzip:   0.36 kB
dist/css/index-CVHl2D3B.css                                                                               0.79 kB │ gzip:   0.32 kB
dist/css/MatrixCard-BVqQUzzx.css                                                                          0.86 kB │ gzip:   0.25 kB
dist/css/index-By8oNveA.css                                                                               0.89 kB │ gzip:   0.42 kB
dist/css/index-D5AwHx5d.css                                                                               0.89 kB │ gzip:   0.37 kB
dist/css/index-BhWXYNfe.css                                                                               0.94 kB │ gzip:   0.43 kB
dist/css/index-WcB63oQd.css                                                                               1.01 kB │ gzip:   0.39 kB
dist/css/index-E3TlBhR5.css                                                                               1.05 kB │ gzip:   0.47 kB
dist/css/home-quick-entry-hZEhId5q.css                                                                    1.05 kB │ gzip:   0.37 kB
dist/css/ExamResultCard-DJSfDte_.css                                                                      1.12 kB │ gzip:   0.43 kB
dist/css/index-BphPVy-R.css                                                                               1.21 kB │ gzip:   0.44 kB
dist/css/index-FmDNBQz6.css                                                                               1.22 kB │ gzip:   0.50 kB
dist/css/redpack-VZ0AqG97.css                                                                             1.25 kB │ gzip:   0.46 kB
dist/css/role-tree-checkbox-mC0FzCf8.css                                                                  1.30 kB │ gzip:   0.42 kB
dist/css/write-HdA4QHHZ.css                                                                               1.32 kB │ gzip:   0.48 kB
dist/css/ViewOrUpdate-7fM-g4hB.css                                                                        1.32 kB │ gzip:   0.50 kB
dist/css/index-DMJnr6jw.css                                                                               1.34 kB │ gzip:   0.43 kB
dist/css/submit-CVT9IhCQ.css                                                                              1.41 kB │ gzip:   0.49 kB
dist/css/help-doc-layout-Cq-8T7YQ.css                                                                     1.43 kB │ gzip:   0.53 kB
dist/css/metric-card-BQXE-Rnr.css                                                                         1.44 kB │ gzip:   0.46 kB
dist/css/index-ul1VSKkd.css                                                                               1.65 kB │ gzip:   0.54 kB
dist/css/code-generator-preview-modal-Datvi8o-.css                                                        1.65 kB │ gzip:   0.75 kB
dist/css/index-BZNtYzVB.css                                                                               1.72 kB │ gzip:   0.53 kB
dist/css/login-Bvf6hYTl.css                                                                               1.78 kB │ gzip:   0.61 kB
dist/css/notify-B_8E5ukD.css                                                                              1.81 kB │ gzip:   0.61 kB
dist/css/card-CKfzlZHl.css                                                                                1.96 kB │ gzip:   0.60 kB
dist/css/notice-form-visible-transfer-department-Bmq10wNm.css                                             2.07 kB │ gzip:   0.53 kB
dist/css/notice-form-visible-transfer-employee-BI-wxgA3.css                                               2.09 kB │ gzip:   0.53 kB
dist/css/index-BzWu19bk.css                                                                               2.94 kB │ gzip:   0.88 kB
dist/css/share-C55ToGJo.css                                                                               3.00 kB │ gzip:   0.84 kB
dist/css/help-doc-user-view-BJvWW1qh.css                                                                  3.79 kB │ gzip:   0.90 kB
dist/css/index-CNnfYsqh.css                                                                               3.87 kB │ gzip:   0.83 kB
dist/css/avatar-cropper-modal-9VFWDLF-.css                                                                4.05 kB │ gzip:   1.19 kB
dist/css/index-a9zfQtlE.css                                                                               4.07 kB │ gzip:   0.86 kB
dist/css/metric-trend-CJ1AF7NT.css                                                                        4.14 kB │ gzip:   0.83 kB
dist/css/index-I1cZW9eG.css                                                                               4.38 kB │ gzip:   0.92 kB
dist/css/index-9sHncdbb.css                                                                               4.40 kB │ gzip:   0.94 kB
dist/css/index-CYd4SvCj.css                                                                               4.49 kB │ gzip:   1.26 kB
dist/css/index-A-o_SUqN.css                                                                               4.53 kB │ gzip:   1.15 kB
dist/css/enterprise-detail-CDC-O28W.css                                                                  17.22 kB │ gzip:   2.61 kB
dist/css/index-BA3svnqk.css                                                                              17.47 kB │ gzip:   3.26 kB
dist/css/index-CL4e_1PC.css                                                                              22.01 kB │ gzip:   3.78 kB
dist/css/index-BwsqDDIv.css                                                                              50.33 kB │ gzip:  10.36 kB
dist/css/index-DNTQ0I4-.css                                                                             128.82 kB │ gzip:  19.08 kB
dist/css/index-D6ofH3mU.css                                                                             172.81 kB │ gzip:  24.33 kB
dist/css/style-BGjGDtq1.css                                                                             392.36 kB │ gzip:  69.45 kB
dist/js/iot-optimistic-lock-CHOgIYgR.js                                                                   0.04 kB │ gzip:   0.06 kB
dist/js/403-CUV_w2s3.js                                                                                   0.06 kB │ gzip:   0.08 kB
dist/js/regular-const-vBMlR054.js                                                                         0.06 kB │ gzip:   0.08 kB
dist/js/_baseClamp-CdDz0ztW.js                                                                            0.09 kB │ gzip:   0.10 kB
dist/js/join-Dox1eePW.js                                                                                  0.09 kB │ gzip:   0.10 kB
dist/js/date-util-BpS9H3Jk.js                                                                             0.11 kB │ gzip:   0.12 kB
dist/js/index-0BF_lBHB.js                                                                                 0.11 kB │ gzip:   0.11 kB
dist/js/auth-Cv4R9J2V.js                                                                                  0.12 kB │ gzip:   0.13 kB
dist/js/index-DbhGEk-F.js                                                                                 0.13 kB │ gzip:   0.13 kB
dist/js/index-BWojQapM.js                                                                                 0.13 kB │ gzip:   0.13 kB
dist/js/iframe-index-gaN4qLPf.js                                                                          0.13 kB │ gzip:   0.14 kB
dist/js/message-detail-Qu5xdoTc.js                                                                        0.14 kB │ gzip:   0.14 kB
dist/js/dept-tree-item-BWLFxUeT.js                                                                        0.14 kB │ gzip:   0.14 kB
dist/js/role-tree-point-BmLT_2YH.js                                                                       0.14 kB │ gzip:   0.14 kB
dist/js/change-log-modal-Bj6CKXOC.js                                                                      0.14 kB │ gzip:   0.14 kB
dist/js/to-be-done-modal-DkuToduN.js                                                                      0.14 kB │ gzip:   0.14 kB
dist/js/gateway-detail-modal-i-YhNu8a.js                                                                  0.14 kB │ gzip:   0.14 kB
dist/js/device-component-zone-DOIj1WNM.js                                                                 0.14 kB │ gzip:   0.14 kB
dist/js/index-C2UnLm3k.js                                                                                 0.15 kB │ gzip:   0.13 kB
dist/js/index-Ct_vsiVl.js                                                                                 0.16 kB │ gzip:   0.15 kB
dist/js/values-C3dqcJvZ.js                                                                                0.16 kB │ gzip:   0.14 kB
dist/js/dict-form-modal-ConA5ZI5.js                                                                       0.16 kB │ gzip:   0.15 kB
dist/js/_baseMap-1pz_5IJP.js                                                                              0.17 kB │ gzip:   0.16 kB
dist/js/job-form-modal-CAoEY62B.js                                                                        0.17 kB │ gzip:   0.15 kB
dist/js/config-form-modal-Czql4QyS.js                                                                     0.17 kB │ gzip:   0.15 kB
dist/js/menu-tree-select-D8irlyu1.js                                                                      0.17 kB │ gzip:   0.15 kB
dist/js/dict-data-form-modal-Dj22Qkib.js                                                                  0.17 kB │ gzip:   0.15 kB
dist/js/fan-search-select-bw-TSHV8.js                                                                     0.17 kB │ gzip:   0.16 kB
dist/js/device-trace-zone-DU-6AjI6.js                                                                     0.17 kB │ gzip:   0.16 kB
dist/js/reload-result-list-3oXpRBQW.js                                                                    0.17 kB │ gzip:   0.16 kB
dist/js/keyword-form-modal-whiSLdGW.js                                                                    0.17 kB │ gzip:   0.16 kB
dist/js/notice-employee-list-BOLF-7bw.js                                                                  0.17 kB │ gzip:   0.16 kB
dist/js/enterprise-bank-operate-modal-B7njRu31.js                                                         0.18 kB │ gzip:   0.16 kB
dist/js/enterprise-invoice-operate-modal-CFybSfLZ.js                                                      0.18 kB │ gzip:   0.16 kB
dist/js/serial-number-record-list-YFXaZ4kl.js                                                             0.19 kB │ gzip:   0.16 kB
dist/js/job-form-modal-DNVRxWs3.js                                                                        0.19 kB │ gzip:   0.16 kB
dist/js/item-Dh-PzqzG.js                                                                                  0.19 kB │ gzip:   0.17 kB
dist/js/dict-form-modal-DcSEZ0w2.js                                                                       0.20 kB │ gzip:   0.17 kB
dist/js/code-generator-table-config-form-table-field-DwyURA52.js                                          0.20 kB │ gzip:   0.17 kB
dist/js/position-form-C-2oscZW.js                                                                         0.20 kB │ gzip:   0.16 kB
dist/js/dict-data-form-modal-C8k_60_l.js                                                                  0.20 kB │ gzip:   0.17 kB
dist/js/message-receiver-modal-DMztCa-D.js                                                                0.20 kB │ gzip:   0.17 kB
dist/js/do-reload-form-modal-BLFdKMm5.js                                                                  0.20 kB │ gzip:   0.17 kB
dist/js/category-form-modal-CMWY6WdP.js                                                                   0.20 kB │ gzip:   0.17 kB
dist/js/point-detail-modal-AToO1iUh.js                                                                    0.20 kB │ gzip:   0.17 kB
dist/js/_isIterateeCall-4dKUEGHZ.js                                                                       0.21 kB │ gzip:   0.19 kB
dist/js/device-status-zone-DzBGUmpJ.js                                                                    0.21 kB │ gzip:   0.17 kB
dist/js/device-realtime-zone-Db03bArn.js                                                                  0.21 kB │ gzip:   0.17 kB
dist/js/instance-detail-modal-BpgZfS57.js                                                                 0.21 kB │ gzip:   0.17 kB
dist/js/startsWith-DLEvf0Mo.js                                                                            0.21 kB │ gzip:   0.19 kB
dist/js/index-B-g-cHOc.js                                                                                 0.22 kB │ gzip:   0.21 kB
dist/js/useMessage-CdbasrFd.js                                                                            0.22 kB │ gzip:   0.16 kB
dist/js/mass-send-detail-drawer-CQjzFgXt.js                                                               0.22 kB │ gzip:   0.17 kB
dist/js/job-log-modal-CMObOlEh.js                                                                         0.22 kB │ gzip:   0.17 kB
dist/js/file-api-Bq9jYQsZ.js                                                                              0.22 kB │ gzip:   0.18 kB
dist/js/reload-api-p6FIb0k2.js                                                                            0.22 kB │ gzip:   0.18 kB
dist/js/serial-number-api-PiQZyzlM.js                                                                     0.23 kB │ gzip:   0.17 kB
dist/js/notice-form-modal-ICIEWAFh.js                                                                     0.23 kB │ gzip:   0.18 kB
dist/js/dict-import-modal-Dcrqxs7R.js                                                                     0.23 kB │ gzip:   0.18 kB
dist/js/device-curve-zone-CsJfemSa.js                                                                     0.24 kB │ gzip:   0.19 kB
dist/js/tenant-form-modal-Ds1Ey5XH.js                                                                     0.24 kB │ gzip:   0.19 kB
dist/js/tenant-admin-modal-Oz6fygl4.js                                                                    0.24 kB │ gzip:   0.19 kB
dist/js/use-wechat-disabled-BTrVqgjK.js                                                                   0.24 kB │ gzip:   0.22 kB
dist/js/instance-params-modal-CGIjHYvz.js                                                                 0.24 kB │ gzip:   0.19 kB
dist/js/mass-send-create-modal-D0ZNP8s6.js                                                                0.24 kB │ gzip:   0.18 kB
dist/js/gateway-api-BZ1SXUrD.js                                                                           0.25 kB │ gzip:   0.18 kB
dist/js/item-point-bind-modal-CD1VgX4y.js                                                                 0.25 kB │ gzip:   0.19 kB
dist/js/device-CZXHt_5p.js                                                                                0.25 kB │ gzip:   0.18 kB
dist/js/source-DYFkCEKn.js                                                                                0.25 kB │ gzip:   0.18 kB
dist/js/serial-number-generate-form-modal-BR1g2nWC.js                                                     0.25 kB │ gzip:   0.19 kB
dist/js/auth-DI4AZOVl.js                                                                                  0.25 kB │ gzip:   0.19 kB
dist/js/index-DLtNAuJl.js                                                                                 0.25 kB │ gzip:   0.19 kB
dist/js/situation-DLZ0HZ5J.js                                                                             0.25 kB │ gzip:   0.18 kB
dist/js/category-api-DuppPev1.js                                                                          0.25 kB │ gzip:   0.18 kB
dist/js/PositionMap-CqjNhZ_T.js                                                                           0.25 kB │ gzip:   0.19 kB
dist/js/index-BecrdXtS.js                                                                                 0.26 kB │ gzip:   0.21 kB
dist/js/goods-form-modal-DEUazgUI.js                                                                      0.26 kB │ gzip:   0.18 kB
dist/js/index-Cn8C_WMP.js                                                                                 0.26 kB │ gzip:   0.18 kB
dist/js/index-DFiZOUsR.js                                                                                 0.26 kB │ gzip:   0.18 kB
dist/js/change-log-form-BIyRttrv.js                                                                       0.27 kB │ gzip:   0.18 kB
dist/js/dict-form-modal-CtHFNwFg.js                                                                       0.27 kB │ gzip:   0.21 kB
dist/js/dict-data-modal-Bf8nsTCg.js                                                                       0.27 kB │ gzip:   0.18 kB
dist/js/point-form-modal-DCdSF1Y0.js                                                                      0.27 kB │ gzip:   0.21 kB
dist/js/iframe-index.vue_vue_type_script_setup_true_lang-D_NGW2RW.js                                      0.27 kB │ gzip:   0.23 kB
dist/js/channel-form-modal-BuTuKa9k.js                                                                    0.28 kB │ gzip:   0.21 kB
dist/js/menu-api-C96PXiPH.js                                                                              0.28 kB │ gzip:   0.20 kB
dist/js/template-form-modal-DDew8YI0.js                                                                   0.28 kB │ gzip:   0.21 kB
dist/js/str-util-DvlP_5eO.js                                                                              0.29 kB │ gzip:   0.17 kB
dist/js/help-doc-catalog-tree-select-rUa0lU9V.js                                                          0.29 kB │ gzip:   0.21 kB
dist/js/position-api-BeT2RLlL.js                                                                          0.29 kB │ gzip:   0.21 kB
dist/js/item-formula-rollup-modal-Bldv7Mgt.js                                                             0.29 kB │ gzip:   0.21 kB
dist/js/code-generator-table-config-form-query-field-CyyX3rjF.js                                          0.30 kB │ gzip:   0.21 kB
dist/js/channel-api-aO1Pu8a8.js                                                                           0.30 kB │ gzip:   0.20 kB
dist/js/profile-api-D4dMAXR5.js                                                                           0.30 kB │ gzip:   0.20 kB
dist/js/notice-view-record-list-CEn2N2Ld.js                                                               0.30 kB │ gzip:   0.20 kB
dist/js/file-api-kq0YytcD.js                                                                              0.31 kB │ gzip:   0.21 kB
dist/js/category-tree-table-DdMpwO1y.js                                                                   0.31 kB │ gzip:   0.20 kB
dist/js/gateway-form-modal-CmZkDpl4.js                                                                    0.31 kB │ gzip:   0.22 kB
dist/js/help-doc-catalog-api-DzldvOOU.js                                                                  0.31 kB │ gzip:   0.19 kB
dist/js/change-log-api-BH18O5P9.js                                                                        0.31 kB │ gzip:   0.19 kB
dist/js/dept-api-DJTZ2Ohf.js                                                                              0.32 kB │ gzip:   0.21 kB
dist/js/mitt-CNZ6avp8.js                                                                                  0.32 kB │ gzip:   0.20 kB
dist/js/notice-form-visible-modal-BhTekCUT.js                                                             0.33 kB │ gzip:   0.22 kB
dist/js/dict-data-modal-niuCdypo.js                                                                       0.33 kB │ gzip:   0.20 kB
dist/js/code-generator-table-config-form-field-CWq5PM15.js                                                0.34 kB │ gzip:   0.23 kB
dist/js/message-send-form-DKp5HZG3.js                                                                     0.34 kB │ gzip:   0.20 kB
dist/js/instance-form-modal-CUBL5jlc.js                                                                   0.34 kB │ gzip:   0.23 kB
dist/js/instance-api-DhbTr3cs.js                                                                          0.36 kB │ gzip:   0.22 kB
dist/js/module-api-CSOoD5wJ.js                                                                            0.36 kB │ gzip:   0.22 kB
dist/js/tag-filter-IviZpaC6.js                                                                            0.37 kB │ gzip:   0.22 kB
dist/js/useFormSetting-DoPTLvu6.js                                                                        0.37 kB │ gzip:   0.28 kB
dist/js/index-CUv6gi09.js                                                                                 0.37 kB │ gzip:   0.29 kB
dist/js/publish-api-BGrVYUut.js                                                                           0.37 kB │ gzip:   0.22 kB
dist/js/template-item-modal-CqwykKcQ.js                                                                   0.38 kB │ gzip:   0.25 kB
dist/js/wechat-disabled-alert-BmlyhWAN.js                                                                 0.38 kB │ gzip:   0.41 kB
dist/js/dept-form-modal-wqiKWoy_.js                                                                       0.38 kB │ gzip:   0.21 kB
dist/js/404-GybJFliZ.js                                                                                   0.39 kB │ gzip:   0.35 kB
dist/js/tenant-api-CTh_W3dO.js                                                                            0.41 kB │ gzip:   0.25 kB
dist/js/table-id-const-BsJohLR_.js                                                                        0.43 kB │ gzip:   0.27 kB
dist/js/job-api-cQOFkcwA.js                                                                               0.43 kB │ gzip:   0.22 kB
dist/js/help-doc-catalog-form-modal-BcRIf4ZT.js                                                           0.43 kB │ gzip:   0.25 kB
dist/js/use-display-tree-ci3nu6VW.js                                                                      0.44 kB │ gzip:   0.35 kB
dist/js/_baseUniq-DhG47J4-.js                                                                             0.45 kB │ gzip:   0.32 kB
dist/js/job-api-tzmuWkcx.js                                                                               0.46 kB │ gzip:   0.27 kB
dist/js/config-list-BL5bfVCp.js                                                                           0.47 kB │ gzip:   0.28 kB
dist/js/theme-api-BLdBjtzl.js                                                                             0.48 kB │ gzip:   0.24 kB
dist/js/code-generator-api-hhdCPyLP.js                                                                    0.48 kB │ gzip:   0.24 kB
dist/js/iot-placeholder-Jx_9CwmY.js                                                                       0.49 kB │ gzip:   0.43 kB
dist/js/goods-catalog-DGwIrSAL.js                                                                         0.50 kB │ gzip:   0.31 kB
dist/js/job-log-list-modal-B1PwrztI.js                                                                    0.50 kB │ gzip:   0.29 kB
dist/js/user-api-BQzbPnqB.js                                                                              0.51 kB │ gzip:   0.29 kB
dist/js/table-auto-height-BnHtpSCn.js                                                                     0.51 kB │ gzip:   0.34 kB
dist/js/common-BzOvJVeF.js                                                                                0.51 kB │ gzip:   0.32 kB
dist/js/custom-catalog-BqpbYlsw.js                                                                        0.52 kB │ gzip:   0.33 kB
dist/js/dict-api-9toGdld9.js                                                                              0.52 kB │ gzip:   0.31 kB
dist/js/auth.vue_vue_type_script_setup_true_lang-LESr4MtW.js                                              0.53 kB │ gzip:   0.46 kB
dist/js/index-CCj2E-QG.js                                                                                 0.55 kB │ gzip:   0.33 kB
dist/js/dict-api-WhRPySeP.js                                                                              0.55 kB │ gzip:   0.25 kB
dist/js/enterprise-bank-list-BFPejWsD.js                                                                  0.56 kB │ gzip:   0.30 kB
dist/js/enterprise-invoice-list-BQtZzPo_.js                                                               0.57 kB │ gzip:   0.30 kB
dist/js/index-DYsTR_u2.js                                                                                 0.57 kB │ gzip:   0.32 kB
dist/js/index.vue_vue_type_script_setup_true_lang-DNgRyR1r.js                                             0.57 kB │ gzip:   0.42 kB
dist/js/deleted-job-list-BtMkpt17.js                                                                      0.58 kB │ gzip:   0.31 kB
dist/js/help-doc-api-Buq0NxpY.js                                                                          0.58 kB │ gzip:   0.26 kB
dist/js/ProjectForm-VM5Pym66.js                                                                           0.58 kB │ gzip:   0.44 kB
dist/js/role-tree-point.vue_vue_type_script_setup_true_lang-DWvP4icP.js                                   0.59 kB │ gzip:   0.39 kB
dist/js/item-api-CNub-9ar.js                                                                              0.59 kB │ gzip:   0.31 kB
dist/js/send-common-DdiR-N0d.js                                                                           0.60 kB │ gzip:   0.37 kB
dist/js/default-time-ranges-CbMj07jC.js                                                                   0.60 kB │ gzip:   0.26 kB
dist/js/FormQrcode-CG1eiPjQ.js                                                                            0.61 kB │ gzip:   0.44 kB
dist/js/template-api-DL6KprAV.js                                                                          0.62 kB │ gzip:   0.29 kB
dist/js/data-api-Bfoo4d6-.js                                                                              0.62 kB │ gzip:   0.37 kB
dist/js/report-api-CUBx9fLG.js                                                                            0.65 kB │ gzip:   0.27 kB
dist/js/enterprise-api-DnZ4arWz.js                                                                        0.67 kB │ gzip:   0.30 kB
dist/js/template-api-CNiMD_LS.js                                                                          0.68 kB │ gzip:   0.30 kB
dist/js/FileList-Ca0skF-A.js                                                                              0.70 kB │ gzip:   0.44 kB
dist/js/notice-api-DuScymgs.js                                                                            0.71 kB │ gzip:   0.29 kB
dist/js/index-CWUxgF9S.js                                                                                 0.73 kB │ gzip:   0.43 kB
dist/js/change-log-modal.vue_vue_type_script_setup_true_lang-Csb3qSVx.js                                  0.77 kB │ gzip:   0.52 kB
dist/js/report-format-vflwHJHL.js                                                                         0.77 kB │ gzip:   0.48 kB
dist/js/ImgList-DhSmhIic.js                                                                               0.77 kB │ gzip:   0.53 kB
dist/js/index-BFK7O6qf.js                                                                                 0.77 kB │ gzip:   0.34 kB
dist/js/point-api-D6nHkams.js                                                                             0.77 kB │ gzip:   0.40 kB
dist/js/file-url-D-GAk1wM.js                                                                              0.77 kB │ gzip:   0.47 kB
dist/js/index.vue_vue_type_script_setup_true_lang-B4-IIeN_.js                                             0.78 kB │ gzip:   0.54 kB
dist/js/help-doc-list-DWJktQjp.js                                                                         0.83 kB │ gzip:   0.40 kB
dist/js/_charsStartIndex-CDR-w1di.js                                                                      0.84 kB │ gzip:   0.46 kB
dist/js/index.vue_vue_type_script_setup_true_lang-DiMgE0NI.js                                             0.84 kB │ gzip:   0.49 kB
dist/js/index-DSGLM3-a.js                                                                                 0.84 kB │ gzip:   0.55 kB
dist/js/role-tree-checkbox-DlNtnEkw.js                                                                    0.85 kB │ gzip:   0.53 kB
dist/js/project-list-columns-BqIdc9YB.js                                                                  0.87 kB │ gzip:   0.38 kB
dist/js/MinusOutlined-C9wqBO8f.js                                                                         0.88 kB │ gzip:   0.55 kB
dist/js/CaretDownOutlined-XnHpV8Sc.js                                                                     0.91 kB │ gzip:   0.58 kB
dist/js/role-api-yyXdnwAz.js                                                                              0.91 kB │ gzip:   0.35 kB
dist/js/CaretRightOutlined-BsxzsCQK.js                                                                    0.91 kB │ gzip:   0.58 kB
dist/js/setting-api-DoMMPOU3.js                                                                           0.92 kB │ gzip:   0.47 kB
dist/js/index-DO8XKm21.js                                                                                 0.92 kB │ gzip:   0.43 kB
dist/js/form-api-DgldMZWV.js                                                                              0.94 kB │ gzip:   0.39 kB
dist/js/menu-tree-select.vue_vue_type_script_setup_true_lang-BM2KlNC-.js                                  0.95 kB │ gzip:   0.61 kB
dist/js/dept-tree-item.vue_vue_type_script_setup_true_lang-Bz4aAUHG.js                                    0.96 kB │ gzip:   0.57 kB
dist/js/MobileOutlined-C1HUXVQH.js                                                                        0.96 kB │ gzip:   0.60 kB
dist/js/LeftOutlined-DdyfeuAB.js                                                                          0.97 kB │ gzip:   0.61 kB
dist/js/RightOutlined-lE9L0CQ5.js                                                                         0.97 kB │ gzip:   0.61 kB
dist/js/DownOutlined-CdWovfX9.js                                                                          0.97 kB │ gzip:   0.61 kB
dist/js/DownloadFile-DdoS06cC.js                                                                          0.97 kB │ gzip:   0.71 kB
dist/js/pie-CDabppHl.js                                                                                   0.99 kB │ gzip:   0.66 kB
dist/js/ArrowLeftOutlined-Baq8JEWR.js                                                                     1.01 kB │ gzip:   0.65 kB
dist/js/TextCopy-Dg3KjcDb.js                                                                              1.04 kB │ gzip:   0.70 kB
dist/js/device.vue_vue_type_script_setup_true_lang-jmd7Bab-.js                                            1.07 kB │ gzip:   0.69 kB
dist/js/UploadOutlined-CxKZtprA.js                                                                        1.09 kB │ gzip:   0.67 kB
dist/js/index.vue_vue_type_script_setup_true_lang-CiupVVjY.js                                             1.10 kB │ gzip:   0.65 kB
dist/js/sortBy-BdoQ0_uF.js                                                                                1.10 kB │ gzip:   0.61 kB
dist/js/index.vue_vue_type_script_setup_true_lang-C_xhdFlI.js                                             1.10 kB │ gzip:   0.66 kB
dist/js/redpack-helper-BxpknW9c.js                                                                        1.11 kB │ gzip:   0.53 kB
dist/js/default-home-card-BGM2W2sq.js                                                                     1.12 kB │ gzip:   0.63 kB
dist/js/menu-data-handler-D2eWFJZJ.js                                                                     1.13 kB │ gzip:   0.53 kB
dist/js/display-slot-card-Cse0_LRz.js                                                                     1.15 kB │ gzip:   0.66 kB
dist/js/CheckSquareOutlined-Df5FAGtW.js                                                                   1.15 kB │ gzip:   0.71 kB
dist/js/CopyOutlined-E8pK8EgY.js                                                                          1.16 kB │ gzip:   0.72 kB
dist/js/to-be-done-modal.vue_vue_type_script_setup_true_lang-ChUAFb40.js                                  1.17 kB │ gzip:   0.79 kB
dist/js/gauge-BIomrCAo.js                                                                                 1.18 kB │ gzip:   0.73 kB
dist/js/preview-sTo1O7Yl.js                                                                               1.18 kB │ gzip:   0.74 kB
dist/js/EditOutlined-C00RIqMp.js                                                                          1.19 kB │ gzip:   0.74 kB
dist/js/PictureOutlined-CgYXTjnp.js                                                                       1.19 kB │ gzip:   0.73 kB
dist/js/BellOutlined-zwrf5iB_.js                                                                          1.22 kB │ gzip:   0.75 kB
dist/js/dept-tree-checkbox-CnY5REHZ.js                                                                    1.23 kB │ gzip:   0.66 kB
dist/js/category-RXMuNqge.js                                                                              1.23 kB │ gzip:   0.71 kB
dist/js/ApartmentOutlined-C3e6dYN5.js                                                                     1.24 kB │ gzip:   0.68 kB
dist/js/message-detail.vue_vue_type_script_setup_true_lang-CY2meR4g.js                                    1.24 kB │ gzip:   0.70 kB
dist/js/fan-search-select.vue_vue_type_script_setup_true_lang-BVcLOVOb.js                                 1.25 kB │ gzip:   0.81 kB
dist/js/BarChartOutlined-SvBy7s1f.js                                                                      1.26 kB │ gzip:   0.62 kB
dist/js/index-CNgkkffC.js                                                                                 1.27 kB │ gzip:   0.70 kB
dist/js/help-doc-catalog-tree-select.vue_vue_type_script_setup_true_lang-DcXnlKIF.js                      1.27 kB │ gzip:   0.75 kB
dist/js/index.vue_vue_type_script_setup_true_lang-BUgyv9mX.js                                             1.28 kB │ gzip:   0.72 kB
dist/js/situation.vue_vue_type_script_setup_true_lang-C8ZPpteU.js                                         1.28 kB │ gzip:   0.82 kB
dist/js/notice-detail-modal-BycGv_RV.js                                                                   1.29 kB │ gzip:   0.72 kB
dist/js/index-I-8s5S12.js                                                                                 1.30 kB │ gzip:   0.70 kB
dist/js/AppstoreOutlined-HOstAWNd.js                                                                      1.30 kB │ gzip:   0.61 kB
dist/js/index-CzkjqlYb.js                                                                                 1.30 kB │ gzip:   0.86 kB
dist/js/home-notice-CD-lNm4S.js                                                                           1.31 kB │ gzip:   0.79 kB
dist/js/mass-send-helper-CjEBSKOh.js                                                                      1.31 kB │ gzip:   0.84 kB
dist/js/notice-form-visible-modal.vue_vue_type_script_setup_true_lang-BERyLDvI.js                         1.32 kB │ gzip:   0.70 kB
dist/js/InboxOutlined-Nc-aVeGJ.js                                                                         1.32 kB │ gzip:   0.82 kB
dist/js/iot-consts-AFE3xLz0.js                                                                            1.33 kB │ gzip:   0.94 kB
dist/js/useProject-BSn_DaPt.js                                                                            1.33 kB │ gzip:   0.75 kB
dist/js/index.vue_vue_type_script_setup_true_lang-BcdCeG6g.js                                             1.34 kB │ gzip:   0.78 kB
dist/js/changelog-card-DkaEjojD.js                                                                        1.36 kB │ gzip:   0.92 kB
dist/js/source.vue_vue_type_script_setup_true_lang-Csn7s93h.js                                            1.38 kB │ gzip:   0.85 kB
dist/js/wechat-api-Bgy0Z_ln.js                                                                            1.40 kB │ gzip:   0.57 kB
dist/js/item.vue_vue_type_script_setup_true_lang-D4J82K5D.js                                              1.40 kB │ gzip:   0.75 kB
dist/js/index.vue_vue_type_script_setup_true_lang-YislGCVt.js                                             1.43 kB │ gzip:   0.75 kB
dist/js/ReloadOutlined-D211sggp.js                                                                        1.48 kB │ gzip:   0.87 kB
dist/js/reload-result-list.vue_vue_type_script_setup_true_lang-DZMtyBFl.js                                1.50 kB │ gzip:   0.94 kB
dist/js/SyncOutlined-2NcZfGHk.js                                                                          1.51 kB │ gzip:   0.91 kB
dist/js/home-quick-entry-modal-9CjekMeh.js                                                                1.51 kB │ gzip:   0.65 kB
dist/js/apiclient-secret-modal-0HiyKKbF.js                                                                1.53 kB │ gzip:   1.03 kB
dist/js/MatrixView-DoyMb-ik.js                                                                            1.54 kB │ gzip:   0.73 kB
dist/js/index-xj--k3MH.js                                                                                 1.55 kB │ gzip:   0.73 kB
dist/js/FullscreenOutlined-CECoXmnN.js                                                                    1.59 kB │ gzip:   0.86 kB
dist/js/do-reload-form-modal.vue_vue_type_script_setup_true_lang-SlYolQYR.js                              1.60 kB │ gzip:   0.94 kB
dist/js/index-Lct3UOrb.js                                                                                 1.62 kB │ gzip:   0.86 kB
dist/js/UrlOperate-DueVat-d.js                                                                            1.63 kB │ gzip:   0.89 kB
dist/js/CloudUploadOutlined-BujzC3S5.js                                                                   1.64 kB │ gzip:   0.98 kB
dist/js/UserAddOutlined-DZ5bEqxN.js                                                                       1.66 kB │ gzip:   0.95 kB
dist/js/index-C8Cn4UDB.js                                                                                 1.69 kB │ gzip:   0.92 kB
dist/js/metric-utils-CS9quiXT.js                                                                          1.69 kB │ gzip:   1.14 kB
dist/js/help-doc-manage-list-CT6WFsVG.js                                                                  1.71 kB │ gzip:   0.80 kB
dist/js/member-editor-B1Er66xX.js                                                                         1.71 kB │ gzip:   0.80 kB
dist/js/index-Dr9miDsN.js                                                                                 1.73 kB │ gzip:   0.81 kB
dist/js/category-form-modal.vue_vue_type_script_setup_true_lang-C9Im55a9.js                               1.73 kB │ gzip:   1.07 kB
dist/js/QrcodeOutlined-CTkrXgNQ.js                                                                        1.77 kB │ gzip:   0.75 kB
dist/js/ExamResultCard-CULIIVj5.js                                                                        1.78 kB │ gzip:   1.04 kB
dist/js/index-BXayZ0AE.js                                                                                 1.78 kB │ gzip:   0.96 kB
dist/js/home-quick-entry-modal.vue_vue_type_script_setup_true_lang-Bk0WzlaF.js                            1.78 kB │ gzip:   1.03 kB
dist/js/index-CSpHTk2D.js                                                                                 1.79 kB │ gzip:   1.21 kB
dist/js/dict-form-modal.vue_vue_type_script_setup_true_lang-DMpbc0Df.js                                   1.79 kB │ gzip:   1.05 kB
dist/js/channel-detail-modal-0ST1V8KW.js                                                                  1.81 kB │ gzip:   0.95 kB
dist/js/index-McoC0VOk.js                                                                                 1.82 kB │ gzip:   1.01 kB
dist/js/module-list-D8Vs_HPR.js                                                                           1.82 kB │ gzip:   0.76 kB
dist/js/notice-form-visible-transfer-department-IZtc4bVs.js                                               1.89 kB │ gzip:   1.02 kB
dist/js/helper-C-j-GxZr.js                                                                                1.89 kB │ gzip:   1.06 kB
dist/js/data-masking-list-oDmZi_HN.js                                                                     1.90 kB │ gzip:   1.32 kB
dist/js/template-send-helper-BkxKut14.js                                                                  1.90 kB │ gzip:   1.06 kB
dist/js/gateway-detail-modal.vue_vue_type_script_setup_true_lang-DNbTVu8i.js                              1.92 kB │ gzip:   0.82 kB
dist/js/serial-number-record-list.vue_vue_type_script_setup_true_lang--Kz5g9gK.js                         1.92 kB │ gzip:   1.12 kB
dist/js/display-factory-card-6pSQzFfP.js                                                                  1.92 kB │ gzip:   0.98 kB
dist/js/index-DNLdrGKc.js                                                                                 1.99 kB │ gzip:   1.10 kB
dist/js/BaseTable-BSz9k5O4.js                                                                             2.04 kB │ gzip:   1.07 kB
dist/js/metric-card-CJbY_eAG.js                                                                           2.04 kB │ gzip:   1.02 kB
dist/js/cache-list-RShRaK1h.js                                                                            2.05 kB │ gzip:   1.42 kB
dist/js/PositionMap.vue_vue_type_script_setup_true_lang-CEmmeZ4h.js                                       2.05 kB │ gzip:   1.28 kB
dist/js/menu-list-BJcre9g8.js                                                                             2.06 kB │ gzip:   0.83 kB
dist/js/item-point-bind-modal.vue_vue_type_script_setup_true_lang-QGG-z_gj.js                             2.07 kB │ gzip:   1.35 kB
dist/js/dict-form-modal.vue_vue_type_script_setup_true_lang-DAs3zUPe.js                                   2.07 kB │ gzip:   1.14 kB
dist/js/index-DzAOgd8y.js                                                                                 2.09 kB │ gzip:   1.22 kB
dist/js/index-BuwPPdhU.js                                                                                 2.12 kB │ gzip:   1.60 kB
dist/js/account-menu-CQs1mfCT.js                                                                          2.14 kB │ gzip:   0.99 kB
dist/js/instance-detail-modal.vue_vue_type_script_setup_true_lang-B_Abddbu.js                             2.15 kB │ gzip:   0.99 kB
dist/js/FilterOutlined-mFA32d3s.js                                                                        2.17 kB │ gzip:   0.95 kB
dist/js/serial-number-generate-form-modal.vue_vue_type_script_setup_true_lang-XZcqY7JV.js                 2.19 kB │ gzip:   1.11 kB
dist/js/tenant-admin-modal.vue_vue_type_script_setup_true_lang-D1tbuzZ2.js                                2.19 kB │ gzip:   1.29 kB
dist/js/notice-employee-detail-BBNBrG8p.js                                                                2.20 kB │ gzip:   1.24 kB
dist/js/index-Bwns6-2y.js                                                                                 2.20 kB │ gzip:   1.25 kB
dist/js/help-doc-user-view-D80qI3yI.js                                                                    2.21 kB │ gzip:   1.19 kB
dist/js/follow-ac8QJrcD.js                                                                                2.26 kB │ gzip:   1.35 kB
dist/js/index-5fPbvHES.js                                                                                 2.28 kB │ gzip:   1.28 kB
dist/js/help-doc-catalog-form-modal.vue_vue_type_script_setup_true_lang-DEYVhWRe.js                       2.28 kB │ gzip:   1.26 kB
dist/js/TextListModal-Bi9FfO15.js                                                                         2.28 kB │ gzip:   1.25 kB
dist/js/config-form-modal.vue_vue_type_script_setup_true_lang-D7jmvdOV.js                                 2.29 kB │ gzip:   1.18 kB
dist/js/import-C_ZN9ju-.js                                                                                2.30 kB │ gzip:   1.56 kB
dist/js/gradient-8Yccf6Vy.js                                                                              2.32 kB │ gzip:   1.07 kB
dist/js/chart-0puzmkje.js                                                                                 2.37 kB │ gzip:   1.11 kB
dist/js/device-component-zone.vue_vue_type_script_setup_true_lang-BuywC-lS.js                             2.42 kB │ gzip:   1.33 kB
dist/js/index-QjXybxUJ.js                                                                                 2.45 kB │ gzip:   1.36 kB
dist/js/position-form.vue_vue_type_script_setup_true_lang-D4KuUo-U.js                                     2.48 kB │ gzip:   1.28 kB
dist/js/device-realtime-zone.vue_vue_type_script_setup_true_lang-B3uj23Do.js                              2.50 kB │ gzip:   1.35 kB
dist/js/index-BWLaCAiH.js                                                                                 2.51 kB │ gzip:   1.21 kB
dist/js/notice-form-visible-transfer-employee-FRR8W8lq.js                                                 2.51 kB │ gzip:   1.31 kB
dist/js/display-api-BVA-RnWs.js                                                                           2.54 kB │ gzip:   1.01 kB
dist/js/keyword-match-modal-DNmRtG-_.js                                                                   2.54 kB │ gzip:   1.39 kB
dist/js/index-dYfrT_Cs.js                                                                                 2.58 kB │ gzip:   1.12 kB
dist/js/SettingOutlined-sw6pyv-a.js                                                                       2.60 kB │ gzip:   1.31 kB
dist/js/index-DjUoJS9k.js                                                                                 2.63 kB │ gzip:   1.36 kB
dist/js/index-DesiOmRb.js                                                                                 2.63 kB │ gzip:   1.57 kB
dist/js/code-generator-table-config-form-table-field.vue_vue_type_script_setup_true_lang-Ba2pewlT.js      2.63 kB │ gzip:   1.22 kB
dist/js/index-BbVmbhXU.js                                                                                 2.65 kB │ gzip:   1.27 kB
dist/js/item-formula-rollup-modal.vue_vue_type_script_setup_true_lang-BgzzJCTk.js                         2.66 kB │ gzip:   1.70 kB
dist/js/device-status-zone.vue_vue_type_script_setup_true_lang-DVOxnAAJ.js                                2.67 kB │ gzip:   1.30 kB
dist/js/dict-data-form-modal.vue_vue_type_script_setup_true_lang-OpJdtGsi.js                              2.69 kB │ gzip:   1.42 kB
dist/js/message-receiver-modal.vue_vue_type_script_setup_true_lang-CzuUcmUn.js                            2.70 kB │ gzip:   1.35 kB
dist/js/reload-list-Bv8h8rz7.js                                                                           2.73 kB │ gzip:   1.79 kB
dist/js/device-utils-6k4pcSn7.js                                                                          2.77 kB │ gzip:   1.48 kB
dist/js/dept-form-modal.vue_vue_type_script_setup_true_lang-Ktx49AIb.js                                   2.77 kB │ gzip:   1.46 kB
dist/js/category-tree-table.vue_vue_type_script_setup_true_lang-CPcrvkMJ.js                               2.79 kB │ gzip:   1.43 kB
dist/js/job-form-modal.vue_vue_type_script_setup_true_lang-B9IoDNyx.js                                    2.79 kB │ gzip:   1.45 kB
dist/js/metric-trend-XhTiYEqa.js                                                                          2.80 kB │ gzip:   1.70 kB
dist/js/enterprise-bank-operate-modal.vue_vue_type_script_setup_true_lang-CEDedx6o.js                     2.85 kB │ gzip:   1.38 kB
dist/js/MatrixCard-DO3DVjsB.js                                                                            2.85 kB │ gzip:   1.30 kB
dist/js/notice-form-modal.vue_vue_type_script_setup_true_lang-DXxlAMsl.js                                 2.85 kB │ gzip:   1.42 kB
dist/js/help-doc-view-record-list-YKXVze5l.js                                                             2.88 kB │ gzip:   1.49 kB
dist/js/message-send-form.vue_vue_type_script_setup_true_lang-B2KWDf3o.js                                 2.88 kB │ gzip:   1.42 kB
dist/js/code-generator-table-config-form-delete-Cl6jrOMA.js                                               2.91 kB │ gzip:   1.47 kB
dist/js/index-C96yaA4e.js                                                                                 2.92 kB │ gzip:   1.46 kB
dist/js/index-QdDFYA_l.js                                                                                 2.94 kB │ gzip:   1.65 kB
dist/js/enterprise-invoice-operate-modal.vue_vue_type_script_setup_true_lang-CWLuK1Je.js                  2.99 kB │ gzip:   1.40 kB
dist/js/dict-import-modal.vue_vue_type_script_setup_true_lang-C3pOoIr7.js                                 3.00 kB │ gzip:   1.70 kB
dist/js/change-log-form.vue_vue_type_script_setup_true_lang-D-bWCVS8.js                                   3.03 kB │ gzip:   1.45 kB
dist/js/serial-number-list-C2-mXqaB.js                                                                    3.05 kB │ gzip:   1.91 kB
dist/js/dict-data-form-modal.vue_vue_type_script_setup_true_lang-XbljfWrp.js                              3.06 kB │ gzip:   1.42 kB
dist/js/home-to-be-done-CPSISn4N.js                                                                       3.06 kB │ gzip:   1.32 kB
dist/js/index-RioNv5br.js                                                                                 3.07 kB │ gzip:   1.57 kB
dist/js/index-B32Hxifn.js                                                                                 3.09 kB │ gzip:   1.70 kB
dist/js/snapshot-drawer-CGVxq6yy.js                                                                       3.11 kB │ gzip:   1.66 kB
dist/js/TeamOutlined-D_BoCUXg.js                                                                          3.14 kB │ gzip:   1.35 kB
dist/js/index-QhQj4CpM.js                                                                                 3.16 kB │ gzip:   1.59 kB
dist/js/submit-CNPzS9MV.js                                                                                3.18 kB │ gzip:   1.69 kB
dist/js/notice-view-record-list.vue_vue_type_script_setup_true_lang-Bv0GBVjI.js                           3.20 kB │ gzip:   1.55 kB
dist/js/public-CEmK92zf.js                                                                                3.20 kB │ gzip:   1.59 kB
dist/js/role-tree-menu-Da0LSc93.js                                                                        3.21 kB │ gzip:   1.50 kB
dist/js/login-Dc0nNduB.js                                                                                 3.23 kB │ gzip:   1.66 kB
dist/js/feedback-list-Bj-mkhzg.js                                                                         3.24 kB │ gzip:   1.61 kB
dist/js/dict-form-modal.vue_vue_type_script_setup_true_lang-BHvgtVw5.js                                   3.30 kB │ gzip:   1.75 kB
dist/js/formatTableColumn-CHsnCt-I.js                                                                     3.31 kB │ gzip:   1.16 kB
dist/js/template-form-modal.vue_vue_type_script_setup_true_lang-BWO9zFU_.js                               3.32 kB │ gzip:   1.75 kB
dist/js/index-CnwtQmDx.js                                                                                 3.37 kB │ gzip:   1.80 kB
dist/js/code-generator-util-X5KUoHCl.js                                                                   3.40 kB │ gzip:   0.90 kB
dist/js/index-B3IBmNYc.js                                                                                 3.44 kB │ gzip:   1.76 kB
dist/js/help-doc-layout-CzAYOxGn.js                                                                       3.47 kB │ gzip:   1.68 kB
dist/js/category-rZrvZCEE.js                                                                              3.49 kB │ gzip:   1.61 kB
dist/js/instance-params-modal.vue_vue_type_script_setup_true_lang-Dgxe7G-6.js                             3.50 kB │ gzip:   1.74 kB
dist/js/index-CP2FAtC0.js                                                                                 3.51 kB │ gzip:   1.56 kB
dist/js/category-DnEmw2T1.js                                                                              3.51 kB │ gzip:   1.62 kB
dist/js/heart-beat-list-Dbp4cEr4.js                                                                       3.58 kB │ gzip:   2.09 kB
dist/js/point-detail-modal.vue_vue_type_script_setup_true_lang-B2nqyEGo.js                                3.64 kB │ gzip:   1.66 kB
dist/js/operate-log-detail-modal-DRXzB6EJ.js                                                              3.65 kB │ gzip:   1.58 kB
dist/js/notice-employee-list.vue_vue_type_script_setup_true_lang-Cu6KoTdw.js                              3.65 kB │ gzip:   1.75 kB
dist/js/gateway-form-modal.vue_vue_type_script_setup_true_lang-F00aH4PA.js                                3.65 kB │ gzip:   1.90 kB
dist/js/mass-send-detail-drawer.vue_vue_type_script_setup_true_lang-BjexPlRh.js                           3.69 kB │ gzip:   1.72 kB
dist/js/config-list-B3CCv8Gp.js                                                                           3.70 kB │ gzip:   1.74 kB
dist/js/home-quick-entry-B9J91AOT.js                                                                      3.70 kB │ gzip:   1.78 kB
dist/js/tenant-form-modal.vue_vue_type_script_setup_true_lang-CwjP2trM.js                                 3.72 kB │ gzip:   1.76 kB
dist/js/card-70Q8NVDC.js                                                                                  3.76 kB │ gzip:   1.68 kB
dist/js/notice-detail-Cci6E_58.js                                                                         3.76 kB │ gzip:   1.70 kB
dist/js/index-ku31llN5.js                                                                                 3.80 kB │ gzip:   1.80 kB
dist/js/redpack-BuyMAaas.js                                                                               3.83 kB │ gzip:   2.16 kB
dist/js/config-operate-drawer-DM9p2_Tg.js                                                                 3.83 kB │ gzip:   1.86 kB
dist/js/index-DSyVLFOh.js                                                                                 3.91 kB │ gzip:   1.80 kB
dist/js/device-trace-zone.vue_vue_type_script_setup_true_lang-DWWo0Asv.js                                 3.95 kB │ gzip:   1.93 kB
dist/js/table-CGS1aOeX.js                                                                                 3.97 kB │ gzip:   1.69 kB
dist/js/mass-send-create-modal.vue_vue_type_script_setup_true_lang-MAuusUFS.js                            4.01 kB │ gzip:   2.05 kB
dist/js/DataPush-BWOZzoRl.js                                                                              4.02 kB │ gzip:   2.08 kB
dist/js/device-curve-zone.vue_vue_type_script_setup_true_lang-BzfATOOO.js                                 4.04 kB │ gzip:   2.22 kB
dist/js/ThunderboltOutlined-B3mAEvyt.js                                                                   4.07 kB │ gzip:   1.49 kB
dist/js/analysis-B1aYUWSn.js                                                                              4.07 kB │ gzip:   2.07 kB
dist/js/publish-log-drawer-guvjH-JW.js                                                                    4.08 kB │ gzip:   1.96 kB
dist/js/panel-Csv2EF8u.js                                                                                 4.11 kB │ gzip:   2.07 kB
dist/js/code-generator-table-config-form-YlWtCUMp.js                                                      4.13 kB │ gzip:   1.70 kB
dist/js/cross-1uCGyIY-.js                                                                                 4.13 kB │ gzip:   2.03 kB
dist/js/index-BuIYcd6u.js                                                                                 4.13 kB │ gzip:   2.21 kB
dist/js/index-DaUzHPgZ.js                                                                                 4.14 kB │ gzip:   2.05 kB
dist/js/index-xiO_Fag9.js                                                                                 4.17 kB │ gzip:   2.18 kB
dist/js/index-BmmoSoTx.js                                                                                 4.18 kB │ gzip:   1.81 kB
dist/js/index-CS8n01Nr.js                                                                                 4.19 kB │ gzip:   1.96 kB
dist/js/code-generator-table-config-form-query-field.vue_vue_type_script_setup_true_lang-tHtO4dqM.js      4.20 kB │ gzip:   1.84 kB
dist/js/index-DAElapng.js                                                                                 4.24 kB │ gzip:   2.19 kB
dist/js/dict-data-modal.vue_vue_type_script_setup_true_lang-Cqr4gdum.js                                   4.28 kB │ gzip:   2.12 kB
dist/js/notify-B-tbiw3-.js                                                                                4.28 kB │ gzip:   2.18 kB
dist/js/FileOutlined-DQ7-_5bE.js                                                                          4.29 kB │ gzip:   1.17 kB
dist/js/index-nxG36qtp.js                                                                                 4.29 kB │ gzip:   2.16 kB
dist/js/index-BCpy6ucX.js                                                                                 4.34 kB │ gzip:   2.20 kB
dist/js/share-CqkMroFt.js                                                                                 4.37 kB │ gzip:   1.98 kB
dist/js/position-list-DG-A7AuC.js                                                                         4.39 kB │ gzip:   2.03 kB
dist/js/module-list.vue_vue_type_script_setup_true_lang-DjqFVREz.js                                       4.41 kB │ gzip:   2.02 kB
dist/js/index-DAMld9N9.js                                                                                 4.45 kB │ gzip:   2.55 kB
dist/js/template-send-detail-drawer-K5z8H6iu.js                                                           4.45 kB │ gzip:   2.06 kB
dist/js/code-generator-list-BeoXAdUZ.js                                                                   4.51 kB │ gzip:   2.00 kB
dist/js/module-form-q5iUwHK3.js                                                                           4.56 kB │ gzip:   2.29 kB
dist/js/index-BesLxVX8.js                                                                                 4.57 kB │ gzip:   2.14 kB
dist/js/job-log-list-modal.vue_vue_type_script_setup_true_lang-crcKSlRv.js                                4.62 kB │ gzip:   2.06 kB
dist/js/job-log-modal.vue_vue_type_script_setup_true_lang-Dghc03_0.js                                     4.69 kB │ gzip:   2.15 kB
dist/js/DataFilter-D0fBfVAM.js                                                                            4.72 kB │ gzip:   1.88 kB
dist/js/dict-data-modal.vue_vue_type_script_setup_true_lang-COZygbVH.js                                   4.79 kB │ gzip:   2.19 kB
dist/js/help-doc-list.vue_vue_type_script_setup_true_lang-ryLPqcaA.js                                     4.79 kB │ gzip:   2.19 kB
dist/js/login-fail-list-CwR8v6nH.js                                                                       4.81 kB │ gzip:   2.19 kB
dist/js/enterprise-invoice-list.vue_vue_type_script_setup_true_lang-D6Dz-_34.js                           4.81 kB │ gzip:   2.30 kB
dist/js/apiclient-form-modal-OSFujn3D.js                                                                  4.82 kB │ gzip:   2.48 kB
dist/js/enterprise-bank-list.vue_vue_type_script_setup_true_lang-Bqt94v6B.js                              4.82 kB │ gzip:   2.27 kB
dist/js/create-Dr4iORXz.js                                                                                4.85 kB │ gzip:   2.40 kB
dist/js/index-D6Wpj_G4.js                                                                                 4.86 kB │ gzip:   2.07 kB
dist/js/index-C_U6buDR.js                                                                                 4.94 kB │ gzip:   2.29 kB
dist/js/ZoomOutOutlined-BqJuOKyN.js                                                                       5.02 kB │ gzip:   1.43 kB
dist/js/message-list-CI2tVL-p.js                                                                          5.12 kB │ gzip:   2.27 kB
dist/js/keyword-form-modal.vue_vue_type_script_setup_true_lang-sVCCzfNT.js                                5.15 kB │ gzip:   2.35 kB
dist/js/instance-item-drawer-DkYniQ65.js                                                                  5.18 kB │ gzip:   2.49 kB
dist/js/index-Iuh9sOh8.js                                                                                 5.18 kB │ gzip:   2.22 kB
dist/js/layout-const-Hku3nqdr.js                                                                          5.21 kB │ gzip:   2.63 kB
dist/js/job-form-modal.vue_vue_type_script_setup_true_lang-CpM9y5w5.js                                    5.33 kB │ gzip:   2.17 kB
dist/js/index-DH8cn2kE.js                                                                                 5.44 kB │ gzip:   2.35 kB
dist/js/index-D_eml76O.js                                                                                 5.45 kB │ gzip:   2.52 kB
dist/js/help-doc-form-drawer-Bey-e_1Q.js                                                                  5.46 kB │ gzip:   2.54 kB
dist/js/help-doc-catalog-tree-C7aI43XK.js                                                                 5.46 kB │ gzip:   2.49 kB
dist/js/template-send-create-modal-DpLjjKKD.js                                                            5.48 kB │ gzip:   2.92 kB
dist/js/code-generator-table-config-form-field.vue_vue_type_script_setup_true_lang-DrQZ2rm0.js            5.58 kB │ gzip:   2.27 kB
dist/js/index-BF5exjsA.js                                                                                 5.58 kB │ gzip:   2.46 kB
dist/js/index-2HjqzyuJ.js                                                                                 5.59 kB │ gzip:   2.38 kB
dist/js/config-list.vue_vue_type_script_setup_true_lang-DfpZRpOy.js                                       5.63 kB │ gzip:   2.32 kB
dist/js/index-DJud9LRL.js                                                                                 5.69 kB │ gzip:   2.32 kB
dist/js/index-Brhxb6fL.js                                                                                 5.70 kB │ gzip:   2.57 kB
dist/js/iot-formula-editor-CIt-4sC4.js                                                                    5.72 kB │ gzip:   2.89 kB
dist/js/index-BcIl7EBF.js                                                                                 5.73 kB │ gzip:   2.48 kB
dist/js/index.vue_vue_type_style_index_0_lang-DuNzQ3L-.js                                                 5.74 kB │ gzip:   1.88 kB
dist/js/enterprise-list-DclPRu0U.js                                                                       5.77 kB │ gzip:   2.65 kB
dist/js/index-Cz80xuxW.js                                                                                 5.81 kB │ gzip:   2.66 kB
dist/js/instance-form-modal.vue_vue_type_script_setup_true_lang-BUfmrBBo.js                               5.89 kB │ gzip:   2.79 kB
dist/js/point-form-modal.vue_vue_type_script_setup_true_lang-CXMl3g0B.js                                  5.94 kB │ gzip:   2.59 kB
dist/js/api-encrypt-index-Cdkl4n0f.js                                                                     5.99 kB │ gzip:   1.74 kB
dist/js/goods-form-modal.vue_vue_type_script_setup_true_lang-jvqq5yJ4.js                                  6.11 kB │ gzip:   2.58 kB
dist/js/file-list-OYk3TALR.js                                                                             6.14 kB │ gzip:   2.44 kB
dist/js/index-DbXt5V6a.js                                                                                 6.14 kB │ gzip:   2.77 kB
dist/js/level3-protect-config-index-DJzJ7PUW.js                                                           6.20 kB │ gzip:   2.81 kB
dist/js/index-aw0IRFHS.js                                                                                 6.20 kB │ gzip:   2.88 kB
dist/js/channel-credential-drawer-DIQ5mmTJ.js                                                             6.20 kB │ gzip:   3.11 kB
dist/js/notice-form-drawer-DWaetQhi.js                                                                    6.22 kB │ gzip:   2.71 kB
dist/js/deleted-job-list.vue_vue_type_script_setup_true_lang-O3V6Dycq.js                                  6.23 kB │ gzip:   2.61 kB
dist/js/login-log-list-C-EVEusJ.js                                                                        6.24 kB │ gzip:   2.75 kB
dist/js/change-log-list-C-xy_YeY.js                                                                       6.25 kB │ gzip:   2.56 kB
dist/js/index-BDUuIwN2.js                                                                                 6.28 kB │ gzip:   2.55 kB
dist/js/index-BNtp2eRX.js                                                                                 6.36 kB │ gzip:   2.62 kB
dist/js/index-CgyK1Ne4.js                                                                                 6.47 kB │ gzip:   2.70 kB
dist/js/channel-form-modal.vue_vue_type_script_setup_true_lang-DIJgqnMt.js                                6.52 kB │ gzip:   3.21 kB
dist/js/ViewData-hfcVZI-Q.js                                                                              6.56 kB │ gzip:   2.46 kB
dist/js/index-CfzSSaeL.js                                                                                 6.58 kB │ gzip:   2.72 kB
dist/js/index-CzQI3dKe.js                                                                                 6.68 kB │ gzip:   3.13 kB
dist/js/index-C9WYaK7E.js                                                                                 6.71 kB │ gzip:   2.85 kB
dist/js/ViewOrUpdate-a8HBl9ZC.js                                                                          6.74 kB │ gzip:   3.25 kB
dist/js/index-BZ291QFu.js                                                                                 6.80 kB │ gzip:   3.00 kB
dist/js/index-Bxn899LZ.js                                                                                 6.89 kB │ gzip:   2.61 kB
dist/js/code-generator-table-config-form-insert-and-update-Bj-8mcHj.js                                    7.05 kB │ gzip:   2.56 kB
dist/js/publish-confirm-modal-Nqe8xU0i.js                                                                 7.22 kB │ gzip:   3.50 kB
dist/js/code-generator-table-config-form-basic-CkTe258r.js                                                7.39 kB │ gzip:   2.78 kB
dist/js/index-CvWzWWD_.js                                                                                 7.41 kB │ gzip:   3.26 kB
dist/js/notice-list-DmVKX1BI.js                                                                           7.45 kB │ gzip:   2.90 kB
dist/js/menu-list.vue_vue_type_script_setup_true_lang-BcqJR9Jc.js                                         7.50 kB │ gzip:   2.93 kB
dist/js/index-Dr2Rasm4.js                                                                                 7.55 kB │ gzip:   3.53 kB
dist/js/operate-log-list-BXFyZTF6.js                                                                      7.65 kB │ gzip:   3.22 kB
dist/js/index-BYZQhVQ7.js                                                                                 7.72 kB │ gzip:   3.27 kB
dist/js/job-list-DgKbzmfG.js                                                                              8.38 kB │ gzip:   3.34 kB
dist/js/index.vue_vue_type_script_setup_true_lang-DYLOKXTL.js                                             8.59 kB │ gzip:   3.68 kB
dist/js/enterprise-employee-list-q817v8_s.js                                                              8.63 kB │ gzip:   3.33 kB
dist/js/index-wIkaknZh.js                                                                                 8.75 kB │ gzip:   3.22 kB
dist/js/index-C2ctvQWb.js                                                                                 8.99 kB │ gzip:   4.14 kB
dist/js/index-Cx-C8rLv.js                                                                                 9.12 kB │ gzip:   3.90 kB
dist/js/index-CRseteu7.js                                                                                 9.67 kB │ gzip:   3.82 kB
dist/js/index-DIOF7m34.js                                                                                 9.74 kB │ gzip:   3.94 kB
dist/js/index-DcGFq8kX.js                                                                                 9.86 kB │ gzip:   4.05 kB
dist/js/index-B76_PT5z.js                                                                                10.17 kB │ gzip:   3.82 kB
dist/js/template-item-modal.vue_vue_type_script_setup_true_lang-CIVFf39j.js                              10.39 kB │ gzip:   4.03 kB
dist/js/SendOutlined-DTTtJr-h.js                                                                         10.86 kB │ gzip:   2.66 kB
dist/js/index-IqSk_dsZ.js                                                                                10.94 kB │ gzip:   4.74 kB
dist/js/index-BLJhGwtu.js                                                                                11.15 kB │ gzip:   4.35 kB
dist/js/index-CFtxqlSH.js                                                                                11.17 kB │ gzip:   3.31 kB
dist/js/menu-operate-modal-DmvQqlRf.js                                                                   11.36 kB │ gzip:   4.13 kB
dist/js/write-xe6BUgm0.js                                                                                12.75 kB │ gzip:   3.98 kB
dist/js/goods-list-CvNzGFTu.js                                                                           15.49 kB │ gzip:   5.61 kB
dist/js/index-C6Hz9yiv.js                                                                                16.01 kB │ gzip:   6.63 kB
dist/js/ua-parser-BtRD7UHp.js                                                                            16.51 kB │ gzip:   7.33 kB
dist/js/EnterOutlined-CTKvn3Fl.js                                                                        18.83 kB │ gzip:   3.43 kB
dist/js/VideoCameraOutlined-BBrJu3tl.js                                                                  20.27 kB │ gzip:   4.81 kB
dist/js/qrcode.vue.esm-DTXgeYp3.js                                                                       20.75 kB │ gzip:   7.31 kB
dist/js/bundle.esm-CaGrYWBJ.js                                                                           20.76 kB │ gzip:   5.97 kB
dist/js/index-CL5yiEuV.js                                                                                25.00 kB │ gzip:   8.49 kB
dist/js/avatar-cropper-modal-CWheqtIX.js                                                                 35.31 kB │ gzip:  10.36 kB
dist/js/sortable.esm-CRp0PNdb.js                                                                         35.52 kB │ gzip:  12.26 kB
dist/js/index.esm-CucVdF0F.js                                                                            45.58 kB │ gzip:  17.23 kB
dist/js/enterprise-detail-Cv85HFYU.js                                                                    57.70 kB │ gzip:  17.09 kB
dist/js/index-Ec7sKtvG.js                                                                                57.90 kB │ gzip:  14.30 kB
dist/js/enterprise-operate-modal-d7GhDTce.js                                                            151.85 kB │ gzip:  34.98 kB
dist/js/index-bN6REZbu.js                                                                               170.10 kB │ gzip:  49.29 kB
dist/js/home-header-CFV7EzRd.js                                                                         294.39 kB │ gzip: 102.76 kB
dist/js/index.esm-JmGZWCIh.js                                                                           458.39 kB │ gzip: 137.65 kB
dist/js/index.esm-qSPBCXoL.js                                                                           547.28 kB │ gzip: 157.18 kB
dist/js/index-BLN_bknD.js                                                                               652.71 kB │ gzip: 205.35 kB
dist/js/code-generator-preview-modal-CBH_jyCZ.js                                                        915.72 kB │ gzip: 303.29 kB
dist/js/index-DUUKvo6j.js                                                                               928.90 kB │ gzip: 274.04 kB
dist/js/index-ByXtPwwH.js                                                                               992.43 kB │ gzip: 160.00 kB
dist/js/index-CUXSwkl6.js                                                                             1,010.68 kB │ gzip: 325.55 kB
dist/js/index-BAsMtFso.js                                                                             1,015.77 kB │ gzip: 329.40 kB
dist/js/index-D5YFAv_y.js                                                                             1,316.21 kB │ gzip: 432.27 kB

(!) Some chunks are larger than 500 kB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking: https://rollupjs.org/configuration-options/#output-manualchunks
- Adjust chunk size limit for this warning via build.chunkSizeWarningLimit.
✓ built in 1m 12s
```

### 2.2 `pnpm lint`（exit 0，0 error / 14 warning，均为 prettier/prettier 存量告警）

```text
$ eslint .

D:\workspace\WisFactory\frontend\src\views\iot\point\components\point-detail-modal.vue
  88:74  warning  Replace `⏎········pointApi.detail(id),⏎········pointApi.references(id),⏎········pointApi.curve(id),⏎······` with `pointApi.detail(id),·pointApi.references(id),·pointApi.curve(id)`  prettier/prettier

D:\workspace\WisFactory\frontend\src\views\iot\point\components\point-form-modal.vue
  14:13  warning  Replace `·v-if="form.id"·type="info"·show-icon·class="ot-margin-bottom10"·message="点位编码创建后不可修改；启停请通过列表行操作「启用/停用」执行"` with `⏎······v-if="form.id"⏎······type="info"⏎······show-icon⏎······class="ot-margin-bottom10"⏎······message="点位编码创建后不可修改；启停请通过列表行操作「启用/停用」执行"⏎···`  prettier/prettier

D:\workspace\WisFactory\frontend\src\views\iot\point\index.vue
  218:21  warning  Replace `⏎··········?·`停用已保存，受影响数据项·${affected·??·0}·个${note·?·`；${note}`·:·''}`⏎·········` with `·?·`停用已保存，受影响数据项·${affected·??·0}·个${note·?·`；${note}`·:·''}``  prettier/prettier
  232:98  warning  Replace `status·===·0·?·'停用后将影响绑定数据项，需重新发布后对接入生效。'·:·''` with `⏎········status·===·0·?·'停用后将影响绑定数据项，需重新发布后对接入生效。'·:·''⏎······`                                    prettier/prettier
  244:94  warning  Replace `status·===·0·?·'停用后需重新发布才对接入生效。'·:·''` with `⏎········status·===·0·?·'停用后需重新发布才对接入生效。'·:·''⏎······`                                                      prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\api\survey\edit-lock-api.spec.ts
  13:9  warning  Replace `␍⏎··acquireEditLockRequest,␍⏎··heartbeatEditLockRequest,␍⏎··releaseEditLockRequest,␍⏎··takeoverEditLockRequest,␍⏎` with `·acquireEditLockRequest,·heartbeatEditLockRequest,·releaseEditLockRequest,·takeoverEditLockRequest·`  prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\layout\setting-drawer-persist.spec.ts
  136:36  warning  Replace `"import·HeaderSetting"` with `'import·HeaderSetting'`  prettier/prettier
  137:35  warning  Replace `"import·HeaderSetting"` with `'import·HeaderSetting'`  prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\lib\axios.spec.ts
   44:137  warning  Replace `␍⏎··await·import('/@/lib/axios'` with `·await·import(␍⏎··'/@/lib/axios'␍⏎`                                                                                                                                                                                                                             prettier/prettier
  366:23   warning  Replace `blobResponse({·'content-type':·'application/octet-stream',·'content-disposition':·'attachment;·filename="%E6%8A%A5%E8%A1%A8.xlsx"'·})` with `␍⏎········blobResponse({·'content-type':·'application/octet-stream',·'content-disposition':·'attachment;·filename="%E6%8A%A5%E8%A1%A8.xlsx"'·})␍⏎······`  prettier/prettier
  376:21   warning  Replace `·Promise.resolve(blobResponse({·'content-type':·'application/octet-stream',·'content-disposition':·'attachment;·fileName="a.txt"'·}))` with `␍⏎······Promise.resolve(blobResponse({·'content-type':·'application/octet-stream',·'content-disposition':·'attachment;·fileName="a.txt"'·}))␍⏎····`       prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\survey\redpack-list-helper.spec.ts
  12:76  warning  Delete `␍⏎·`  prettier/prettier

D:\workspace\WisFactory\frontend\tests\unit\views\survey\redpack-columns.spec.ts
  35:4  warning  Replace `·dataId:·1,·nickname:·'张三',·amount:·888,·redpackStatus:·1,·simulated:·false,·sendByName:·'操作员A',·sendTime:·'2026-10-03·10:00:00',·finishTime:·'2026-10-03·10:00:05'` with `␍⏎····dataId:·1,␍⏎····nickname:·'张三',␍⏎····amount:·888,␍⏎····redpackStatus:·1,␍⏎····simulated:·false,␍⏎····sendByName:·'操作员A',␍⏎····sendTime:·'2026-10-03·10:00:00',␍⏎····finishTime:·'2026-10-03·10:00:05',␍⏎·`  prettier/prettier
  37:4  warning  Replace `·dataId:·2,·nickname:·'李四',·amount:·888,·redpackStatus:·2,·errCode:·'NOAUTH',·errMsg:·'未关注公众号',·sendByName:·'操作员B',·finishTime:·'2026-10-03·10:01:00'` with `␍⏎····dataId:·2,␍⏎····nickname:·'李四',␍⏎····amount:·888,␍⏎····redpackStatus:·2,␍⏎····errCode:·'NOAUTH',␍⏎····errMsg:·'未关注公众号',␍⏎····sendByName:·'操作员B',␍⏎····finishTime:·'2026-10-03·10:01:00',␍⏎·`                              prettier/prettier

✖ 14 problems (0 errors, 14 warnings)
  0 errors and 14 warnings potentially fixable with the `--fix` option.

```

### 2.3 `pnpm typecheck`（exit 0）

```text
$ vue-tsc --noEmit
$ vitest run

```

### 2.4 `pnpm test`（exit 0，82 文件 / 697 例通过，≥677 基线）

```text
 RUN  v2.1.9 D:/workspace/WisFactory/frontend

 ✓ tests/unit/components/iot/formula-rules.spec.ts (32 tests) 24ms
 ✓ tests/unit/composables/survey/use-edit-lock.spec.ts (12 tests) 33ms
 ✓ tests/unit/lib/axios.spec.ts (27 tests) 239ms
 ✓ tests/unit/wechat/template-send-helper.spec.ts (17 tests) 15ms
 ✓ tests/unit/router/build-routes.spec.ts (15 tests) 59ms
 ✓ tests/unit/router/router-guard.spec.ts (7 tests) 63ms
 ✓ tests/unit/views/iot/item-formula-rollup-modal.spec.ts (5 tests) 1263ms
   ✓ 实例数据项取值编辑弹窗（TASK-0094） > formula 模式：父 ref 可调 showModal，回填公式并按清单构造 SELF 引用分组 424ms
 ✓ tests/unit/components/iot/iot-formula-editor.spec.ts (15 tests) 1355ms
   ✓ 公式编辑器：渲染与实时校验 > 初始公式渲染到 textarea，合法公式无错误提示 407ms
 ✓ tests/unit/display/metrics-board-read-ports.spec.ts (6 tests) 1764ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > metricValues→卡片同源展示：value/yoy/mom/unit 原样，不现算 618ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > completeness available=true 填 map，低于阈值标记可见 333ms
 ✓ tests/unit/display/display-read-ports.spec.ts (10 tests) 19ms
 ✓ tests/unit/views/iot/point-bind-modal.spec.ts (4 tests) 2379ms
   ✓ 实例数据项绑定点位弹窗（TASK-0092） > 父 ref 可调 showModal；拉 status=1 启用点位列表并展示点位编码/名称/值类型 950ms
   ✓ 实例数据项绑定点位弹窗（TASK-0092） > 选一行点确定 → bindPoint(1001,2002,301) → 成功提示 + 事件通知 + 关窗 820ms
   ✓ 实例数据项绑定点位弹窗（TASK-0092） > 绑定失败 → captureError + message.error 提示，弹窗不关闭 364ms
 ✓ tests/unit/wechat/mass-send-helper.spec.ts (15 tests) 15ms
stderr | tests/unit/layout/top-expand-user-space.spec.ts > 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 顶行内同时渲染一级菜单与用户区（用户区不再挂在下方 header）
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-user-space.spec.ts > 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 用户区排在菜单之后（DOM 顺序即视觉左右顺序：菜单在左、用户区在右）
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-user-space.spec.ts > 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 顶行不再渲染二级菜单（二级菜单回到侧栏，由 layout 直接挂载）
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-user-space.spec.ts > 用户区组件 menu-user-space.vue > 渲染模块切换 / 消息 / 设置 / 头像四组元素
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-user-space.spec.ts > 用户区组件 menu-user-space.vue > 组件自持配色解析，明暗切换不报错（学术蓝白底深字 / 经典深底白字）
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-user-space.spec.ts > 「顶部」布局复用同一用户区组件（top-menu/index.vue） > 渲染后用户区存在且与分组布局同源
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace> 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

 ✓ tests/unit/layout/top-expand-user-space.spec.ts (17 tests) 2079ms
   ✓ 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 顶行内同时渲染一级菜单与用户区（用户区不再挂在下方 header） 902ms
   ✓ 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 用户区排在菜单之后（DOM 顺序即视觉左右顺序：菜单在左、用户区在右） 389ms
 ✓ tests/unit/views/system/menu-cache-frame.spec.ts (7 tests) 7277ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 回显：后端 0/0 显示为未开启，1/1 显示为开启（null/0 不得误判为真） 2288ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 提交：布尔开关映射为 0/1（后端只接受 0/1，不接受布尔） 1609ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 提交：开启外链后 frameFlag=1 且带上 frameUrl 1048ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 提交：关闭外链时清空外链地址，避免库中残留旧地址与开关状态不一致 1499ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 外链菜单仍需填组件路径（buildRoutes 会跳过无 component 的菜单项） 822ms
 ✓ tests/unit/views/iot/instance-tree-edit.spec.ts (8 tests) 6217ms
   ✓ 实例树节点编辑入口（TASK-0093） > title 插槽在工厂/车间/设备各级节点旁渲染 EditOutlined 编辑按钮 1457ms
   ✓ 实例树节点编辑入口（TASK-0093） > 点击工厂节点编辑按钮 → 复用 addInstance 打开编辑弹窗（完整 record 传 showModal） 860ms
   ✓ 实例树节点编辑入口（TASK-0093） > 骨架节点（缺 deptId/sort）点击 → 拉详情补齐字段后打开编辑弹窗 536ms
   ✓ 实例树节点编辑入口（TASK-0093） > 详情加载失败时不开空弹窗，并上报 otSentry 651ms
   ✓ 实例树节点编辑入口（TASK-0093） > locked 设备节点编辑按钮透传 locked=1（弹窗自身显示仅改状态提示） 492ms
   ✓ 实例树节点编辑入口（TASK-0093） > 无 iot:instance:edit 权限时编辑按钮被 v-privilege 指令移除（树节点仍在） 735ms
   ✓ 实例树节点编辑入口（TASK-0093） > 树过滤仍保留命中节点及其祖先路径，且编辑按钮随树保留 801ms
   ✓ 实例树节点编辑入口（TASK-0093） > 树节点选择仍触发 onSelectTree（点击标签文字检索子树，与编辑按钮独立） 681ms
 ✓ tests/unit/survey/report-format.spec.ts (22 tests) 11ms
 ✓ tests/unit/views/iot/point.spec.ts (9 tests) 8109ms
   ✓ 点位管理页（TASK-0097） > 挂载拉列表（默认分页）且无占位骨架文案 1586ms
   ✓ 点位管理页（TASK-0097） > 空数据时展示诚实空态，不造假数据 595ms
   ✓ 点位管理页（TASK-0097） > 新建/详情通过父 ref 调 defineExpose.showModal（BUG-0008 回归） 1020ms
   ✓ 点位管理页（TASK-0097） > 启停调 changeStatus({ids,status})，停用含发布生效提示 923ms
   ✓ 点位管理页（TASK-0097） > 删除调 remove([id])（API 层拼路径参数），导出调 export 带过滤参数 931ms
   ✓ 点位表单弹窗（TASK-0097） > 新增提交调 add，payload 不含 status/id 867ms
   ✓ 点位表单弹窗（TASK-0097） > 编辑提交调 update，携带 id 与 pointCode 同值，且不含 status 758ms
   ✓ 点位表单弹窗（TASK-0097） > 编辑 showModal 后 pointCode 禁用（创建后不可改） 617ms
   ✓ 点位详情弹窗（TASK-0097） > 父 ref 调 showModal(id) → detail/references/curve 三读，曲线空集展示诚实提示 807ms
 ✓ tests/unit/store/user-module.spec.ts (9 tests) 169ms
 ✓ tests/unit/utils/public-utils-layering.spec.ts (5 tests) 24ms
stderr | unknown test
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".

stderr | unknown test
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".
OPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".

stderr | tests/unit/layout/top-expand-side-menu.spec.ts > 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 「顶部」布局仍不显示该设置项（无左侧栏）
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-side-menu.spec.ts > 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 分组布局显示该设置项，且取值回显 store 宽度
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-side-menu.spec.ts > 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 传统布局仍显示该设置项（原行为不回归）
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/top-expand-side-menu.spec.ts > 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 「展开」(side-expand) 不显示该设置项（侧栏宽度由内部固定值决定，调了不生效）
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 ✓ tests/unit/layout/top-expand-side-menu.spec.ts (12 tests) 4411ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 「顶部」布局仍不显示该设置项（无左侧栏） 1666ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 分组布局显示该设置项，且取值回显 store 宽度 978ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 传统布局仍显示该设置项（原行为不回归） 801ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 「展开」(side-expand) 不显示该设置项（侧栏宽度由内部固定值决定，调了不生效） 689ms
stderr | tests/unit/views/system/role-tree-collapse.spec.ts > 角色授权树顶级分类折叠 > 默认全部展开：每个顶级分类都渲染折叠图标（收起态图标不出现）
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

stderr | tests/unit/views/system/role-tree-collapse.spec.ts > 角色授权树顶级分类折叠 > 点折叠图标收起该分类：图标切为收起态且其子级隐藏
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

stderr | tests/unit/views/system/role-tree-collapse.spec.ts > 角色授权树顶级分类折叠 > 再点一次恢复展开（收起可逆）
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

stderr | tests/unit/views/system/role-tree-collapse.spec.ts > 角色授权树顶级分类折叠 > 多开：收起一个不影响其他分类的展开态
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

stderr | tests/unit/views/system/role-tree-collapse.spec.ts > 角色授权树顶级分类折叠 > 非顶级节点不渲染折叠图标（子目录/菜单不参与折叠）
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

stderr | tests/unit/views/system/role-tree-collapse.spec.ts > 角色授权树顶级分类折叠 > 无子级的顶级节点不渲染折叠图标（防御性）
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

stderr | tests/unit/views/system/role-tree-collapse.spec.ts > 角色授权树顶级分类折叠 > 点击折叠图标不会触发复选框勾选（click.stop 防回归）
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

 ✓ tests/unit/views/system/role-tree-collapse.spec.ts (9 tests) 1661ms
   ✓ 角色授权树顶级分类折叠 > 默认全部展开：每个顶级分类都渲染折叠图标（收起态图标不出现） 431ms
 ✓ tests/unit/views/system/menu-perms-mapping.spec.ts (10 tests) 9903ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 编辑回显：输入框显示库中 perms（修复前恒空） 1586ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 库中 perms 为 null 时回显空串，不显示字面量 null 814ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 改值提交：updateMenu 收到 params.perms 为目标值（修复前后端收不到该字段） 1522ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 清空提交：提交空串而非 null（null 会被后端 UPDATE 跳过、清不掉原权限串） 919ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 新增功能点带权限串提交：addMenu 收到 params.perms 896ms
   ✓ 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095） > 编辑回显：输入框显示库中 query（修复前绑 queryParam，后端无此字段故恒空） 791ms
   ✓ 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095） > 改值提交：updateMenu 收到 params.query 为目标值，且不再有 queryParam 1070ms
   ✓ 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095） > 清空提交：提交空串而非 null（同 perms 口径，null 会被后端 UPDATE 跳过） 754ms
   ✓ 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090） > 菜单类型抽屉无「路由名称」表单项 838ms
   ✓ 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090） > 提交报文不含 routeName 710ms
 ✓ tests/unit/theme/disabled-button-text.spec.ts (9 tests) 15ms
 ✓ tests/unit/utils/sanitize-html.spec.ts (10 tests) 55ms
 ✓ tests/unit/store/app-config-remote.spec.ts (9 tests) 86ms
stderr | tests/unit/layout/setting-drawer-persist.spec.ts > 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > layout/index.vue 渲染设置抽屉一份
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/setting-drawer-persist.spec.ts > 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > 发 settingMitt open 事件后抽屉打开
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

 ✓ tests/unit/composables/survey/project-list-columns.spec.ts (19 tests) 10ms
 ✓ tests/unit/store/menu-module-helper.spec.ts (21 tests) 15ms
stderr | tests/unit/layout/setting-drawer-persist.spec.ts > 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > 切换「菜单布局」后抽屉仍打开（修复前会随布局组件卸载而消失）
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

 ✓ tests/unit/layout/setting-drawer-persist.spec.ts (7 tests) 4026ms
   ✓ 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > 发 settingMitt open 事件后抽屉打开 2560ms
   ✓ 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > 切换「菜单布局」后抽屉仍打开（修复前会随布局组件卸载而消失） 1046ms
stderr | tests/unit/layout/setting-drawer-persist.spec.ts > 设置抽屉触发链路（BUG-0098 源码层） > 设置按钮仍渲染（用户区功能不回归）
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

 ✓ tests/unit/views/iot/instance-item-value-editor.spec.ts (4 tests) 3853ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > 公式项/汇总项各显示对应编辑按钮，权限不足时按钮被移除 1093ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > 点击「编辑公式」→ 弹窗打开 → 保存 → 调 saveFormula 并刷新清单 1254ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > 点击「编辑汇总」→ 弹窗打开 → 切 AVG 保存 → 调 saveRollup 并刷新清单 1051ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > locked=1：编辑按钮禁用，点击提示锁定，不打开弹窗、不提交 451ms
 ✓ tests/unit/survey/write-storage.spec.ts (15 tests) 16ms
 ✓ tests/unit/layout/menu-path-match.spec.ts (9 tests) 11ms
 ✓ tests/unit/views/survey/province-match.spec.ts (6 tests) 8ms
 ✓ tests/unit/display/metric-utils.spec.ts (10 tests) 11ms
 ✓ tests/unit/api/iot/point-api.spec.ts (11 tests) 13ms
 ✓ tests/unit/monitor/actuator-api.spec.ts (10 tests) 13ms
 ✓ tests/unit/views/survey/write-time-window.spec.ts (11 tests) 13ms
 ✓ tests/unit/survey/wx-auth.spec.ts (18 tests) 10ms
 ✓ tests/unit/display/device-utils.spec.ts (8 tests) 12ms
 ✓ tests/unit/layout/top-expand-theme.spec.ts (9 tests) 574ms
   ✓ 分组布局一级菜单主题联动（top-menu.vue） > 学术蓝（sideMenuTheme=light）→ 菜单渲染为 light，不含 dark 364ms
 ✓ tests/unit/views/survey/redpack-columns.spec.ts (3 tests) 1735ms
   ✓ 问卷红包列表「发放人/发放结果」两列 > 列定义含「发放人」与「发放结果」两列，既有列不丢 805ms
   ✓ 问卷红包列表「发放人/发放结果」两列 > 发放人列取 sendByName，缺失显示「—」（未发放行/老后端行） 535ms
   ✓ 问卷红包列表「发放人/发放结果」两列 > 发放结果列五态文案与配色（成功绿/失败红/发放中蓝/待确认橙/未发放默认） 393ms
 ✓ tests/unit/display/factory-card-status-values.spec.ts (5 tests) 871ms
   ✓ 工厂卡片接状态数真值（TASK-0058 批次二 D2） > 挂载即用 node.id 取 factory-summary，运行/异常/离线三槽位填真值 374ms
 ✓ tests/unit/store/ui-config-merge.spec.ts (9 tests) 11ms
 ✓ tests/unit/survey/option-label-display.spec.ts (12 tests) 150ms
stderr | tests/unit/layout/header-setting-persist.spec.ts > 设置抽屉界面偏好持久化 > 改动配置后按作用域上报服务端（模块作用域）
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/header-setting-persist.spec.ts > 设置抽屉界面偏好持久化 > 无当前模块时上报到账户全局（moduleId=0）
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/header-setting-persist.spec.ts > 设置抽屉界面偏好持久化 > 内部态变化（remoteItems/remoteLoaded）不上报，防模块间配置串写
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

stderr | tests/unit/layout/header-setting-persist.spec.ts > 设置抽屉界面偏好持久化 > 恢复默认配置：删除当前作用域远端配置
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 ✓ tests/unit/layout/header-setting-persist.spec.ts (4 tests) 291ms
 ✓ tests/unit/monitor/cache-monitor-api.spec.ts (5 tests) 9ms
 ✓ tests/unit/views/iot/point-bind-drawer.spec.ts (2 tests) 1679ms
   ✓ 实例数据项抽屉绑定点位入口（TASK-0092） > 「绑定点位」按钮挂 iot:item:edit 权限；无该权限时按钮被移除 855ms
   ✓ 实例数据项抽屉绑定点位入口（TASK-0092） > 行内「绑定点位」按钮点击 → 拉启用点位列表 → 选点 → bindPoint → 刷新并显示点位编码 821ms
 ✓ tests/unit/survey/option-label-map.spec.ts (10 tests) 9ms
 ✓ tests/unit/monitor/actuator-helper.spec.ts (12 tests) 33ms
 ✓ tests/unit/display/device-realtime-zone.spec.ts (5 tests) 853ms
   ✓ 设备实时区接 realtime 真值（TASK-0089 D2） > 绑定类 + 公式类统一取 realtime 真值（v/q/ts 同源），来源标注「实时值读口」 509ms
 ✓ tests/unit/display/factory-summary-read-port.spec.ts (5 tests) 14ms
 ✓ tests/unit/survey/redpack-helper.spec.ts (8 tests) 9ms
 ✓ tests/unit/views/wechat/tag-filter.spec.ts (13 tests) 8ms
 ✓ tests/unit/views/system/menu-operate-modal.spec.ts (4 tests) 3465ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 编辑回显：库中「不显示」的菜单开关为不显示，未动开关提交不翻转原值 1588ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 关「显示状态」提交 visible=0（用户实测场景：修复前后端只收到旧值 1） 754ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 关「菜单状态」提交 status=0，且可见性保持显示 511ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 新增菜单未动开关时提交 visible=1、status=1 611ms
 ✓ tests/unit/views/system/role-list-selection.spec.ts (8 tests) 973ms
   ✓ 角色列表刷新后选中态（组件级） > 首次加载自动选中第一个角色 515ms
 ✓ tests/unit/survey/redpack-list-helper.spec.ts (7 tests) 13ms
 ✓ tests/unit/monitor/cache-helper.spec.ts (11 tests) 8ms
 ✓ tests/unit/store/dict.spec.ts (4 tests) 17ms
 ✓ tests/unit/survey/expand-original-data.spec.ts (6 tests) 8ms
 ✓ tests/unit/display/display-tree.spec.ts (5 tests) 23ms
 ✓ tests/unit/utils/file-url.spec.ts (7 tests) 9ms
 ✓ tests/unit/views/system/menu-list-perms-column.spec.ts (5 tests) 23ms
 ✓ tests/unit/theme/theme-preset.spec.ts (13 tests) 10ms
 ✓ tests/unit/store/app-config.spec.ts (3 tests) 262ms
 ✓ tests/unit/views/system/menu-field-mapping.spec.ts (4 tests) 12ms
 ✓ tests/unit/survey/normalize-field-schemes.spec.ts (5 tests) 7ms
 ✓ tests/unit/api/survey/edit-lock-api.spec.ts (6 tests) 9ms
 ✓ tests/unit/api/iot/point-bind-api.spec.ts (4 tests) 9ms
 ✓ tests/unit/display/display-routes.spec.ts (3 tests) 7ms
 ✓ tests/unit/views/system/login-entry-render.spec.ts (1 test) 482ms
   ✓ 登录页首屏组件渲染（入口精简集） > 只注册 entryComponents 时，用户名/密码/验证码三件套都能渲染 480ms
 ✓ tests/unit/monitor/iot-inspect-helper.spec.ts (5 tests) 8ms
 ✓ tests/unit/display/factory-page-metrics.spec.ts (3 tests) 615ms
   ✓ 工厂页接工厂级指标真值（TASK-0058 批次二 D2） > 工厂级指标看板：metricValues 同源展示 value/yoy/mom（不现算、null 不冒充 0） 388ms
 ✓ tests/unit/plugin/entry-ui.spec.ts (2 tests) 353ms
   ✓ plugin/entry-ui 入口精简集 > 登录表单（a-form / a-form-item / a-input-password）能渲染出密码输入框 347ms
 ✓ tests/unit/monitor/alarm-helper.spec.ts (4 tests) 9ms
 ✓ tests/unit/wechat/send-common.spec.ts (4 tests) 6ms
 ✓ tests/unit/lib/encrypt.spec.ts (4 tests) 9ms
 ✓ tests/unit/views/iot/instance-item-drawer-ref.spec.ts (1 test) 507ms
   ✓ 实例数据项抽屉父组件 ref 暴露（BUG-0008） > 父组件 ref 可拿到 showDrawer/onClose；调用后抽屉打开且标题可见 506ms
 ✓ tests/unit/survey/serial-number.spec.ts (6 tests) 6ms
 ✓ tests/unit/layout/module-switcher.spec.ts (6 tests) 303ms
 ✓ tests/unit/display/factory-card-summary.spec.ts (5 tests) 165ms
 ✓ tests/unit/views/system/login-captcha-polling.spec.ts (2 tests) 363ms
 ✓ tests/unit/display/display-slot-card.spec.ts (2 tests) 99ms
 ✓ tests/unit/composables/survey/use-project-creator.spec.ts (5 tests) 4ms

 Test Files  82 passed (82)
      Tests  697 passed (697)
   Start at  20:40:03
   Duration  59.66s (transform 34.32s, setup 0ms, collect 431.76s, tests 69.29s, environment 126.53s, prepare 26.91s)

```

## 3 未做项与原因

- 未做：点位导入向导本页内实现（模板下载/逐行回执/错误清单下载）。原因：brief 裁决——导入向导已有独立菜单 4016（`views/iot/import/index.vue`），本页只保留跳转链接 `to="/iot/import"`，不重复实现；导入三方法在 point-api 中保留供向导复用。
- 未做：真实 24h 曲线数据与真实反查数据造数。原因：后端本期 fail-soft 空集占位（TASK-0023 明确真实数据归 TASK-0029 写库后联调），前端按契约展示诚实空态，不伪造数据。
- 未做：单位字典/通道/实例/部门下拉数据源接线。原因：brief 允许无数据源时用原生输入或既有字典封装；本页对 channelId/deptId/boundInstanceId 用原生数字输入、unit 用文本输入并在提示中声明后端字典校验为准，避免造假选项。
- 未做：发布后生效链路。原因：属 FEAT-0008 发布页职责（TASK-0024 未交付），点位页仅在启停响应中透出后端 effectNote「变更已保存，重新发布后对接入生效」，不改发布纪律。

## 4 偏离计划之处

- 实施方案阶段发现并修复以下真实缺陷（先红后绿，均未改测试期望迁就）：
  1. 详情弹窗模板调用 `dataTypeLabel(...)` 而脚本导入名为 `iotPointDataTypeLabel` → 首次运行报 `_ctx.dataTypeLabel is not a function` 且详情渲染失败；改为模板使用正确导入名，并加 `as number` 满足 typecheck。
  2. 表单弹窗使用无效组件 `<a-input.Textarea>`（Vue warn: Failed to resolve component）→ 改为 `<a-textarea>`。
  3. API 封装初版 `getRequest(url)` 只传一个参数与 axios 封装签名（2-3 参数）不符 → 补齐 `undefined` 第二参数（与 dictApi.queryDictById 既有范式一致），并同步更新单测期望值。
  4. `point.spec.ts` 中 `vi.fn` 无参签名导致 `mock.calls[0][0]` 索引越界类型错误 → 为 mock 声明真实入参类型（非改断言）。
- 为满足 BUG-0008（defineExpose）回归，表单弹窗除 `showModal` 外额外暴露 `onSubmit` 供父级/测试直接调用；不改变产品行为。
- 除上述外无偏离：未改验收命令、未改后端、未改指标页。

## 5 遗留问题

- 详情曲线/反查目前仅能展示后端 fail-soft 空集；待 TASK-0029 真实曲线数据落库后联调（属后端计划内，非本批缺陷）。
- 单位输入为文本而非下拉选码，依赖后端 iot_unit 字典校验拒绝非法值；若后续要前置提示，需接 sys_dict 字典数据源（brief 允许本次不做）。
- 构建仍有存量告警：survey 模块 const 赋值告警、大 chunk >500KB 提示、14 条 prettier warning（存量），与本批无关，未擅自修改。
- TASK 状态保持「进行中」，未置「已完成」；LOG、索引、状态收尾由协调者执行 close-task.sh。
