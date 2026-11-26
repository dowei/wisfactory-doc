# TASK-0098 验收报告：问卷功能软下线

- 执行者：接续被中断的会话（no-subagents，未再派 subagent）
- 仓库 BASE / 实测 HEAD：`57804b47`（`git -C /d/workspace/WisFactory rev-parse --short HEAD`，一致）
- 执行日期：2026-11-25
- 验收命令工作目录：`/d/workspace/WisFactory/frontend`
- 原始输出说明：以下逐字粘贴命令 stdout/stderr 全文，仅去除终端 ANSI 颜色转义序列（不改变任何文字）；含颜色码的完整原始日志另存于 `C:/Users/dowil/AppData/Local/Temp/task-0098-acceptance/{build-test,lint,typecheck,test}.log`。

## 1 改动文件清单

| 文件 | 状态 | 实际改动 |
| --- | --- | --- |
| `resources/sql/upgrade/20261125_disable_survey.sql` | 新增 | 两条软下线 UPDATE（sys_module.status=0 WHERE module_code='survey' AND deleted=0；sys_menu.visible=0 WHERE module_id=3 AND deleted=0）+ 文件末尾注释附两条反向回滚 UPDATE；无 DROP/DELETE。编码列按实际 schema 取 `module_code`（非 brief 正文所写的 `code`）。 |
| `frontend/src/router/routers.ts` | 修改 | 不再 import 也不展开 `surveyRouters`，`/survey/**` 静态路由入口摘除；survey 代码文件保留。 |
| `frontend/src/router/index.ts` | 修改 | 不再 import `surveyFormWorkbenchRouters` / `surveyRecycleRouter` / `surveyCategoryRouters` / `surveyTemplatePreviewRouters`，不再 push 这四组静态路由。 |
| `frontend/tests/unit/router/build-routes.spec.ts` | 修改 | mock survey 路由导出置空；断言改为 survey 静态路由在任意模块状态下均不挂载、`/survey/**` 落到 404 catch-all（新增 `assertSurveyRoutesRemoved` 断言集）。 |
| `frontend/tests/unit/router/router-guard.spec.ts` | 修改 | 公开页放行用例改用通用 `meta.public` 页面 `/public/demo`；新增「`/s/:key` 无 token 直链跳登录而非放行」用例；survey mock 置空。 |

改动文件共 5 个，与 brief 范围完全一致；未改任何其他文件（`git status` 中 `.agents/`、`.tmp-t0044v/` 为既有未跟踪项，未触碰）。

## 2 每条验收命令原始输出

### 2.1 `pnpm build:test` — EXIT=0

```text
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
vite v5.2.12 building for test...
transforming...

./icon.svg referenced in D:/workspace/WisFactory/frontend/node_modules/.pnpm/vue3-json-viewer@2.3.1_vue@3.4.27_typescript@5.6.3_/node_modules/vue3-json-viewer/dist/index.css didn't resolve at build time, it will remain unchanged to be resolved at runtime
[plugin:vite:esbuild] [plugin vite:esbuild] src/views/business/oa/enterprise/components/enterprise-employee-list.vue?vue&type=script&setup=true&lang.ts: This assignment will throw because "columns" is a constant
298|                class: "ot-margin-bottom5",
299|                modelValue: columns,
300|                "onUpdate:modelValue": _cache[1] || (_cache[1] = ($event: any) => ((columns) = $event)),
   |                                                                                    ^
301|                tableId: _unref(TABLE_ID_CONST).BUSINESS.OA.ENTERPRISE_EMPLOYEE,
302|                refresh: queryEmployee

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

✓ 6019 modules transformed.
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
dist/css/metric-form-modal-Da1YSf-b.css                                                                   0.06 kB │ gzip:   0.08 kB
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
dist/css/index-BGXi8PCn.css                                                                               0.12 kB │ gzip:   0.11 kB
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
dist/js/403-CH1eh6kT.js                                                                                   0.06 kB │ gzip:   0.08 kB
dist/js/regular-const-vBMlR054.js                                                                         0.06 kB │ gzip:   0.08 kB
dist/js/_baseClamp-CdDz0ztW.js                                                                            0.09 kB │ gzip:   0.10 kB
dist/js/join-Dox1eePW.js                                                                                  0.09 kB │ gzip:   0.10 kB
dist/js/date-util-g-z0WHNF.js                                                                             0.11 kB │ gzip:   0.12 kB
dist/js/index-RZxSX7jb.js                                                                                 0.11 kB │ gzip:   0.11 kB
dist/js/auth-CSs1QapH.js                                                                                  0.12 kB │ gzip:   0.13 kB
dist/js/index-DVSZTN6g.js                                                                                 0.13 kB │ gzip:   0.13 kB
dist/js/index-BdliusDI.js                                                                                 0.13 kB │ gzip:   0.13 kB
dist/js/iframe-index-BahwcyaB.js                                                                          0.13 kB │ gzip:   0.14 kB
dist/js/message-detail-qm9Ji0p1.js                                                                        0.14 kB │ gzip:   0.14 kB
dist/js/dept-tree-item-DQ9kyQfJ.js                                                                        0.14 kB │ gzip:   0.14 kB
dist/js/role-tree-point-BUqCm1T5.js                                                                       0.14 kB │ gzip:   0.14 kB
dist/js/change-log-modal-DE7Yw8iJ.js                                                                      0.14 kB │ gzip:   0.14 kB
dist/js/to-be-done-modal-BHkvqoHQ.js                                                                      0.14 kB │ gzip:   0.14 kB
dist/js/gateway-detail-modal-CbBsX4JD.js                                                                  0.14 kB │ gzip:   0.14 kB
dist/js/device-component-zone-Dj3TCOn_.js                                                                 0.14 kB │ gzip:   0.14 kB
dist/js/index-D29SYx1_.js                                                                                 0.15 kB │ gzip:   0.13 kB
dist/js/index-CGzdHLVB.js                                                                                 0.16 kB │ gzip:   0.15 kB
dist/js/values-DNeQlIs_.js                                                                                0.16 kB │ gzip:   0.14 kB
dist/js/dict-form-modal-ppIWI2tX.js                                                                       0.16 kB │ gzip:   0.15 kB
dist/js/_baseMap-5-oJxKKN.js                                                                              0.17 kB │ gzip:   0.16 kB
dist/js/job-form-modal-CTBFRR5v.js                                                                        0.17 kB │ gzip:   0.15 kB
dist/js/config-form-modal-DAvuYMYT.js                                                                     0.17 kB │ gzip:   0.15 kB
dist/js/menu-tree-select-CijV_6Qs.js                                                                      0.17 kB │ gzip:   0.15 kB
dist/js/dict-data-form-modal-DfzE_F3c.js                                                                  0.17 kB │ gzip:   0.15 kB
dist/js/fan-search-select-C2WZhMX4.js                                                                     0.17 kB │ gzip:   0.16 kB
dist/js/device-trace-zone-c0lYzTua.js                                                                     0.17 kB │ gzip:   0.16 kB
dist/js/reload-result-list-DCyuNc0P.js                                                                    0.17 kB │ gzip:   0.15 kB
dist/js/keyword-form-modal-UtLSZYsC.js                                                                    0.17 kB │ gzip:   0.16 kB
dist/js/notice-employee-list-DMKQ3tyi.js                                                                  0.17 kB │ gzip:   0.16 kB
dist/js/enterprise-bank-operate-modal-BDgfRQiI.js                                                         0.18 kB │ gzip:   0.16 kB
dist/js/enterprise-invoice-operate-modal-5OTinemx.js                                                      0.18 kB │ gzip:   0.16 kB
dist/js/serial-number-record-list-Lav8zAi7.js                                                             0.19 kB │ gzip:   0.16 kB
dist/js/job-form-modal-90231YTG.js                                                                        0.19 kB │ gzip:   0.16 kB
dist/js/item-BxJo7Ebt.js                                                                                  0.19 kB │ gzip:   0.17 kB
dist/js/dict-form-modal-Ber4wbHu.js                                                                       0.20 kB │ gzip:   0.17 kB
dist/js/code-generator-table-config-form-table-field-DOrfcUmk.js                                          0.20 kB │ gzip:   0.17 kB
dist/js/position-form-CJAH1dVT.js                                                                         0.20 kB │ gzip:   0.16 kB
dist/js/dict-data-form-modal-BayC4PfI.js                                                                  0.20 kB │ gzip:   0.17 kB
dist/js/message-receiver-modal-Cdxyz_W2.js                                                                0.20 kB │ gzip:   0.17 kB
dist/js/do-reload-form-modal-BYQMKeNu.js                                                                  0.20 kB │ gzip:   0.17 kB
dist/js/category-form-modal-CvX4BZ6G.js                                                                   0.20 kB │ gzip:   0.17 kB
dist/js/point-detail-modal-DYzmIrvt.js                                                                    0.20 kB │ gzip:   0.17 kB
dist/js/_isIterateeCall-D_OsaHyu.js                                                                       0.21 kB │ gzip:   0.19 kB
dist/js/metric-detail-modal-DuIEDFq7.js                                                                   0.21 kB │ gzip:   0.17 kB
dist/js/device-status-zone-C5nu1blb.js                                                                    0.21 kB │ gzip:   0.18 kB
dist/js/device-realtime-zone-DXYvvHH4.js                                                                  0.21 kB │ gzip:   0.18 kB
dist/js/instance-detail-modal-GyRS8DdI.js                                                                 0.21 kB │ gzip:   0.17 kB
dist/js/startsWith-C8eeUIpK.js                                                                            0.21 kB │ gzip:   0.19 kB
dist/js/useMessage-T2hGXaXz.js                                                                            0.22 kB │ gzip:   0.16 kB
dist/js/mass-send-detail-drawer-DsVOvwuj.js                                                               0.22 kB │ gzip:   0.17 kB
dist/js/job-log-modal-C96C4cxV.js                                                                         0.22 kB │ gzip:   0.17 kB
dist/js/file-api-Cv0-bzZY.js                                                                              0.22 kB │ gzip:   0.18 kB
dist/js/reload-api-L0-ATMEW.js                                                                            0.22 kB │ gzip:   0.17 kB
dist/js/serial-number-api-Y0r6TyGv.js                                                                     0.23 kB │ gzip:   0.17 kB
dist/js/notice-form-modal-CjIMASBo.js                                                                     0.23 kB │ gzip:   0.18 kB
dist/js/dict-import-modal-CcQC-u7i.js                                                                     0.23 kB │ gzip:   0.18 kB
dist/js/device-curve-zone-XNk3HJGz.js                                                                     0.24 kB │ gzip:   0.19 kB
dist/js/tenant-form-modal-CMAw2Tzr.js                                                                     0.24 kB │ gzip:   0.19 kB
dist/js/tenant-admin-modal-D96J-QUl.js                                                                    0.24 kB │ gzip:   0.19 kB
dist/js/use-wechat-disabled-BmO4guMs.js                                                                   0.24 kB │ gzip:   0.22 kB
dist/js/instance-params-modal-CNy1QgV5.js                                                                 0.24 kB │ gzip:   0.19 kB
dist/js/mass-send-create-modal-mBGZFeiu.js                                                                0.24 kB │ gzip:   0.18 kB
dist/js/gateway-api-CK84Bipb.js                                                                           0.25 kB │ gzip:   0.18 kB
dist/js/item-point-bind-modal-CcNz3tSh.js                                                                 0.25 kB │ gzip:   0.19 kB
dist/js/device-Di7c4Bg6.js                                                                                0.25 kB │ gzip:   0.18 kB
dist/js/source-BUE4nhG0.js                                                                                0.25 kB │ gzip:   0.18 kB
dist/js/serial-number-generate-form-modal-Dct689l_.js                                                     0.25 kB │ gzip:   0.18 kB
dist/js/auth-BiOWeEnk.js                                                                                  0.25 kB │ gzip:   0.19 kB
dist/js/index-DiIoan1v.js                                                                                 0.25 kB │ gzip:   0.19 kB
dist/js/situation-Do5cu75v.js                                                                             0.25 kB │ gzip:   0.18 kB
dist/js/category-api-D8oeopFB.js                                                                          0.25 kB │ gzip:   0.18 kB
dist/js/PositionMap-C6kEvoaJ.js                                                                           0.25 kB │ gzip:   0.19 kB
dist/js/index-BwKR0WXu.js                                                                                 0.26 kB │ gzip:   0.21 kB
dist/js/goods-form-modal-DXA8dHwK.js                                                                      0.26 kB │ gzip:   0.18 kB
dist/js/index-DyDwXELJ.js                                                                                 0.26 kB │ gzip:   0.18 kB
dist/js/index-CTxSvM4y.js                                                                                 0.26 kB │ gzip:   0.18 kB
dist/js/change-log-form-DF9Q-LUS.js                                                                       0.27 kB │ gzip:   0.18 kB
dist/js/dict-form-modal-WMCLZ7rT.js                                                                       0.27 kB │ gzip:   0.20 kB
dist/js/dict-data-modal-BK1uL22x.js                                                                       0.27 kB │ gzip:   0.18 kB
dist/js/point-form-modal-2tH5mPbL.js                                                                      0.27 kB │ gzip:   0.20 kB
dist/js/iframe-index.vue_vue_type_script_setup_true_lang-D95zO_qY.js                                      0.27 kB │ gzip:   0.23 kB
dist/js/channel-form-modal-CEG9wx3T.js                                                                    0.28 kB │ gzip:   0.21 kB
dist/js/menu-api-BfATLOgL.js                                                                              0.28 kB │ gzip:   0.20 kB
dist/js/template-form-modal-DwlWAGA8.js                                                                   0.28 kB │ gzip:   0.20 kB
dist/js/str-util-DvlP_5eO.js                                                                              0.29 kB │ gzip:   0.17 kB
dist/js/help-doc-catalog-tree-select-TQlv0CZe.js                                                          0.29 kB │ gzip:   0.21 kB
dist/js/position-api-C5VUulxo.js                                                                          0.29 kB │ gzip:   0.21 kB
dist/js/item-formula-rollup-modal-BxfPnxoT.js                                                             0.29 kB │ gzip:   0.21 kB
dist/js/code-generator-table-config-form-query-field-DXI6Y8F_.js                                          0.30 kB │ gzip:   0.20 kB
dist/js/channel-api-DBMD2VQ0.js                                                                           0.30 kB │ gzip:   0.20 kB
dist/js/profile-api-DpTBwGs7.js                                                                           0.30 kB │ gzip:   0.20 kB
dist/js/notice-view-record-list-DSyfNczd.js                                                               0.30 kB │ gzip:   0.20 kB
dist/js/file-api-D7XIoEoy.js                                                                              0.31 kB │ gzip:   0.21 kB
dist/js/category-tree-table-_i8W6Twl.js                                                                   0.31 kB │ gzip:   0.20 kB
dist/js/gateway-form-modal-CNm6y4aE.js                                                                    0.31 kB │ gzip:   0.22 kB
dist/js/help-doc-catalog-api-CPf9Bomu.js                                                                  0.31 kB │ gzip:   0.18 kB
dist/js/change-log-api-CLZ1MNWd.js                                                                        0.31 kB │ gzip:   0.19 kB
dist/js/dept-api-ByspwEph.js                                                                              0.32 kB │ gzip:   0.21 kB
dist/js/mitt-CNZ6avp8.js                                                                                  0.32 kB │ gzip:   0.20 kB
dist/js/notice-form-visible-modal-D159uhzm.js                                                             0.33 kB │ gzip:   0.22 kB
dist/js/dict-data-modal-AP3_FZoE.js                                                                       0.33 kB │ gzip:   0.20 kB
dist/js/code-generator-table-config-form-field-9OjdSyO9.js                                                0.34 kB │ gzip:   0.23 kB
dist/js/message-send-form-DbaSu_P6.js                                                                     0.34 kB │ gzip:   0.20 kB
dist/js/instance-form-modal-B2D3-DK0.js                                                                   0.34 kB │ gzip:   0.23 kB
dist/js/instance-api-B1FPvJNT.js                                                                          0.36 kB │ gzip:   0.21 kB
dist/js/module-api-DbEsWj7Q.js                                                                            0.36 kB │ gzip:   0.22 kB
dist/js/tag-filter-IviZpaC6.js                                                                            0.37 kB │ gzip:   0.22 kB
dist/js/useFormSetting-DBMjyr_-.js                                                                        0.37 kB │ gzip:   0.28 kB
dist/js/index-B82grEYj.js                                                                                 0.37 kB │ gzip:   0.28 kB
dist/js/publish-api-Bq5ETF9k.js                                                                           0.37 kB │ gzip:   0.21 kB
dist/js/template-item-modal-BXP0avLW.js                                                                   0.38 kB │ gzip:   0.25 kB
dist/js/wechat-disabled-alert-CP2_rifs.js                                                                 0.38 kB │ gzip:   0.41 kB
dist/js/dept-form-modal-DOd2rO36.js                                                                       0.38 kB │ gzip:   0.21 kB
dist/js/404-DUtEXEZU.js                                                                                   0.39 kB │ gzip:   0.35 kB
dist/js/tenant-api-BYJ-w4cB.js                                                                            0.41 kB │ gzip:   0.25 kB
dist/js/table-id-const-BsJohLR_.js                                                                        0.43 kB │ gzip:   0.27 kB
dist/js/job-api-D54gT9Tx.js                                                                               0.43 kB │ gzip:   0.22 kB
dist/js/help-doc-catalog-form-modal-vY89Kfua.js                                                           0.43 kB │ gzip:   0.25 kB
dist/js/use-display-tree-B55P0gi2.js                                                                      0.44 kB │ gzip:   0.35 kB
dist/js/metric-api-BmzwJ85H.js                                                                            0.45 kB │ gzip:   0.27 kB
dist/js/_baseUniq-4BVe-k-a.js                                                                             0.45 kB │ gzip:   0.32 kB
dist/js/job-api-DkudkF6D.js                                                                               0.46 kB │ gzip:   0.27 kB
dist/js/config-list-kzxHclSo.js                                                                           0.47 kB │ gzip:   0.27 kB
dist/js/theme-api-DIiF8jOI.js                                                                             0.48 kB │ gzip:   0.24 kB
dist/js/code-generator-api-Ci7aotTX.js                                                                    0.48 kB │ gzip:   0.24 kB
dist/js/iot-placeholder-BcL3vDEV.js                                                                       0.49 kB │ gzip:   0.43 kB
dist/js/goods-catalog-BXJnaB1H.js                                                                         0.50 kB │ gzip:   0.31 kB
dist/js/job-log-list-modal-D2fDPRWv.js                                                                    0.50 kB │ gzip:   0.29 kB
dist/js/user-api-BrJFpWwv.js                                                                              0.51 kB │ gzip:   0.29 kB
dist/js/table-auto-height-CD3xsVEr.js                                                                     0.51 kB │ gzip:   0.34 kB
dist/js/common-BzOvJVeF.js                                                                                0.51 kB │ gzip:   0.32 kB
dist/js/custom-catalog-DCxMzjkK.js                                                                        0.52 kB │ gzip:   0.33 kB
dist/js/dict-api-CYoiY7Le.js                                                                              0.52 kB │ gzip:   0.31 kB
dist/js/auth.vue_vue_type_script_setup_true_lang-DRKsXEnL.js                                              0.53 kB │ gzip:   0.46 kB
dist/js/index-DXeI6reS.js                                                                                 0.55 kB │ gzip:   0.33 kB
dist/js/dict-api-CEzjzMYK.js                                                                              0.55 kB │ gzip:   0.25 kB
dist/js/enterprise-bank-list-CWI6BrE5.js                                                                  0.56 kB │ gzip:   0.30 kB
dist/js/enterprise-invoice-list-BiIsz9YL.js                                                               0.57 kB │ gzip:   0.30 kB
dist/js/index-DYsTR_u2.js                                                                                 0.57 kB │ gzip:   0.32 kB
dist/js/index.vue_vue_type_script_setup_true_lang-_1MqF8wi.js                                             0.57 kB │ gzip:   0.42 kB
dist/js/deleted-job-list-DnHVDOSd.js                                                                      0.58 kB │ gzip:   0.30 kB
dist/js/help-doc-api-ieL3N1ji.js                                                                          0.58 kB │ gzip:   0.26 kB
dist/js/ProjectForm-B30mfdel.js                                                                           0.58 kB │ gzip:   0.44 kB
dist/js/role-tree-point.vue_vue_type_script_setup_true_lang-O-FRmS-d.js                                   0.59 kB │ gzip:   0.39 kB
dist/js/item-api-CAQJoJZL.js                                                                              0.59 kB │ gzip:   0.30 kB
dist/js/send-common-DdiR-N0d.js                                                                           0.60 kB │ gzip:   0.37 kB
dist/js/default-time-ranges-CkiqSGaH.js                                                                   0.60 kB │ gzip:   0.26 kB
dist/js/FormQrcode-s0SYPq3Q.js                                                                            0.61 kB │ gzip:   0.44 kB
dist/js/template-api-BcW1cPjR.js                                                                          0.62 kB │ gzip:   0.29 kB
dist/js/data-api-MQ4mBox7.js                                                                              0.62 kB │ gzip:   0.37 kB
dist/js/report-api-B-TURIY6.js                                                                            0.65 kB │ gzip:   0.27 kB
dist/js/enterprise-api-BjKICr9O.js                                                                        0.67 kB │ gzip:   0.30 kB
dist/js/template-api-BW2er91T.js                                                                          0.68 kB │ gzip:   0.30 kB
dist/js/FileList-Bp7Uf-B-.js                                                                              0.70 kB │ gzip:   0.44 kB
dist/js/notice-api-D-uErmhv.js                                                                            0.71 kB │ gzip:   0.29 kB
dist/js/index-C7Q0yfzo.js                                                                                 0.73 kB │ gzip:   0.43 kB
dist/js/change-log-modal.vue_vue_type_script_setup_true_lang-1cd-2trd.js                                  0.77 kB │ gzip:   0.52 kB
dist/js/report-format-vflwHJHL.js                                                                         0.77 kB │ gzip:   0.48 kB
dist/js/ImgList-BVVd7I3R.js                                                                               0.77 kB │ gzip:   0.53 kB
dist/js/index-BN8K7jsH.js                                                                                 0.77 kB │ gzip:   0.34 kB
dist/js/point-api-CxTUFnyz.js                                                                             0.77 kB │ gzip:   0.39 kB
dist/js/file-url-QBG_Fsuu.js                                                                              0.77 kB │ gzip:   0.47 kB
dist/js/index.vue_vue_type_script_setup_true_lang-Cf2eXv0l.js                                             0.78 kB │ gzip:   0.53 kB
dist/js/help-doc-list-BqdjVXr0.js                                                                         0.83 kB │ gzip:   0.40 kB
dist/js/_charsStartIndex-vYaCg6DN.js                                                                      0.84 kB │ gzip:   0.46 kB
dist/js/index.vue_vue_type_script_setup_true_lang-BfGzOi-J.js                                             0.84 kB │ gzip:   0.49 kB
dist/js/index-CKqNekIh.js                                                                                 0.84 kB │ gzip:   0.55 kB
dist/js/role-tree-checkbox-jK_Kpdyn.js                                                                    0.85 kB │ gzip:   0.53 kB
dist/js/project-list-columns-BqIdc9YB.js                                                                  0.87 kB │ gzip:   0.38 kB
dist/js/MinusOutlined-YTMl_jKJ.js                                                                         0.88 kB │ gzip:   0.55 kB
dist/js/CaretDownOutlined-cIU8k-s1.js                                                                     0.91 kB │ gzip:   0.58 kB
dist/js/role-api-BY4XQGjO.js                                                                              0.91 kB │ gzip:   0.35 kB
dist/js/CaretRightOutlined-BvdDd_E3.js                                                                    0.91 kB │ gzip:   0.58 kB
dist/js/setting-api-DDyNp-j4.js                                                                           0.92 kB │ gzip:   0.47 kB
dist/js/index-CDW_svbs.js                                                                                 0.92 kB │ gzip:   0.43 kB
dist/js/form-api-Cin15isS.js                                                                              0.94 kB │ gzip:   0.38 kB
dist/js/menu-tree-select.vue_vue_type_script_setup_true_lang-DVPyeuC9.js                                  0.95 kB │ gzip:   0.61 kB
dist/js/dept-tree-item.vue_vue_type_script_setup_true_lang-x_qWi196.js                                    0.96 kB │ gzip:   0.57 kB
dist/js/MobileOutlined-B_Y7YVWl.js                                                                        0.96 kB │ gzip:   0.60 kB
dist/js/LeftOutlined-DJIx0Y3Y.js                                                                          0.97 kB │ gzip:   0.61 kB
dist/js/RightOutlined-Ca1nur2l.js                                                                         0.97 kB │ gzip:   0.61 kB
dist/js/DownOutlined-B2x8c06n.js                                                                          0.97 kB │ gzip:   0.61 kB
dist/js/DownloadFile-CPSFw5-p.js                                                                          0.97 kB │ gzip:   0.71 kB
dist/js/pie-q_tnOwqY.js                                                                                   0.98 kB │ gzip:   0.66 kB
dist/js/ArrowLeftOutlined-Bv_O6qZp.js                                                                     1.01 kB │ gzip:   0.65 kB
dist/js/TextCopy-CGF9InkK.js                                                                              1.04 kB │ gzip:   0.70 kB
dist/js/device.vue_vue_type_script_setup_true_lang-C3UqlzrC.js                                            1.07 kB │ gzip:   0.69 kB
dist/js/UploadOutlined-BieBgKsC.js                                                                        1.09 kB │ gzip:   0.67 kB
dist/js/index.vue_vue_type_script_setup_true_lang-rPPR9CTi.js                                             1.10 kB │ gzip:   0.65 kB
dist/js/sortBy-CgTngIg1.js                                                                                1.10 kB │ gzip:   0.60 kB
dist/js/index.vue_vue_type_script_setup_true_lang-BKVuOSCY.js                                             1.10 kB │ gzip:   0.66 kB
dist/js/redpack-helper-BxpknW9c.js                                                                        1.11 kB │ gzip:   0.53 kB
dist/js/default-home-card-BUjUMiqA.js                                                                     1.12 kB │ gzip:   0.63 kB
dist/js/menu-data-handler-B0dvsIEp.js                                                                     1.13 kB │ gzip:   0.53 kB
dist/js/display-slot-card-DZPlAQDs.js                                                                     1.15 kB │ gzip:   0.66 kB
dist/js/CheckSquareOutlined-B3ZvVZoS.js                                                                   1.15 kB │ gzip:   0.71 kB
dist/js/CopyOutlined-DlK-Woi9.js                                                                          1.16 kB │ gzip:   0.72 kB
dist/js/to-be-done-modal.vue_vue_type_script_setup_true_lang-CBOpT1w_.js                                  1.17 kB │ gzip:   0.79 kB
dist/js/gauge-BKxbytUN.js                                                                                 1.18 kB │ gzip:   0.73 kB
dist/js/preview-CKQkTQUJ.js                                                                               1.18 kB │ gzip:   0.74 kB
dist/js/EditOutlined-ukuvGaUl.js                                                                          1.19 kB │ gzip:   0.74 kB
dist/js/PictureOutlined-C1rYeBwb.js                                                                       1.19 kB │ gzip:   0.73 kB
dist/js/BellOutlined-6-boE_1Q.js                                                                          1.22 kB │ gzip:   0.75 kB
dist/js/dept-tree-checkbox-CMR6GArT.js                                                                    1.23 kB │ gzip:   0.66 kB
dist/js/category-UXHv--2q.js                                                                              1.23 kB │ gzip:   0.70 kB
dist/js/ApartmentOutlined-1-uQYz7I.js                                                                     1.24 kB │ gzip:   0.68 kB
dist/js/message-detail.vue_vue_type_script_setup_true_lang-RfNxKSvB.js                                    1.24 kB │ gzip:   0.71 kB
dist/js/fan-search-select.vue_vue_type_script_setup_true_lang-CrmHSXO5.js                                 1.25 kB │ gzip:   0.81 kB
dist/js/BarChartOutlined-DIPGYhUO.js                                                                      1.26 kB │ gzip:   0.62 kB
dist/js/index-D03gmE_1.js                                                                                 1.27 kB │ gzip:   0.70 kB
dist/js/help-doc-catalog-tree-select.vue_vue_type_script_setup_true_lang-Blo_dNRG.js                      1.27 kB │ gzip:   0.75 kB
dist/js/index.vue_vue_type_script_setup_true_lang-Cp9jvzGV.js                                             1.28 kB │ gzip:   0.72 kB
dist/js/situation.vue_vue_type_script_setup_true_lang-Drh0g4PR.js                                         1.28 kB │ gzip:   0.82 kB
dist/js/notice-detail-modal-D6mNfvU8.js                                                                   1.29 kB │ gzip:   0.72 kB
dist/js/index-DSJ9P_ym.js                                                                                 1.30 kB │ gzip:   0.70 kB
dist/js/AppstoreOutlined-Crk_va4d.js                                                                      1.30 kB │ gzip:   0.61 kB
dist/js/index-grdv7kId.js                                                                                 1.30 kB │ gzip:   0.86 kB
dist/js/home-notice-DJ5Iq0Lx.js                                                                           1.31 kB │ gzip:   0.79 kB
dist/js/mass-send-helper-Bp6pvDDI.js                                                                      1.31 kB │ gzip:   0.83 kB
dist/js/notice-form-visible-modal.vue_vue_type_script_setup_true_lang-D6celML4.js                         1.32 kB │ gzip:   0.70 kB
dist/js/InboxOutlined-B_A9q5V9.js                                                                         1.32 kB │ gzip:   0.82 kB
dist/js/useProject-CGpQ-291.js                                                                            1.33 kB │ gzip:   0.75 kB
dist/js/index.vue_vue_type_script_setup_true_lang-CwVSYP9V.js                                             1.34 kB │ gzip:   0.78 kB
dist/js/changelog-card-CZH1lCUW.js                                                                        1.36 kB │ gzip:   0.91 kB
dist/js/source.vue_vue_type_script_setup_true_lang-B4hiWCSa.js                                            1.38 kB │ gzip:   0.84 kB
dist/js/wechat-api-BndBWujp.js                                                                            1.40 kB │ gzip:   0.57 kB
dist/js/item.vue_vue_type_script_setup_true_lang-BhxCL0QE.js                                              1.40 kB │ gzip:   0.75 kB
dist/js/index.vue_vue_type_script_setup_true_lang-Bl_O3ffj.js                                             1.43 kB │ gzip:   0.75 kB
dist/js/ReloadOutlined-L5nKG-bG.js                                                                        1.48 kB │ gzip:   0.87 kB
dist/js/reload-result-list.vue_vue_type_script_setup_true_lang-BaApQ7_W.js                                1.50 kB │ gzip:   0.95 kB
dist/js/SyncOutlined-L018hOUs.js                                                                          1.51 kB │ gzip:   0.91 kB
dist/js/home-quick-entry-modal-CijsQPeS.js                                                                1.51 kB │ gzip:   0.64 kB
dist/js/apiclient-secret-modal-CE1G737u.js                                                                1.53 kB │ gzip:   1.03 kB
dist/js/iot-consts-BURryymr.js                                                                            1.54 kB │ gzip:   1.03 kB
dist/js/MatrixView-CHBVnmq2.js                                                                            1.54 kB │ gzip:   0.73 kB
dist/js/index-CrgP46ma.js                                                                                 1.55 kB │ gzip:   0.73 kB
dist/js/FullscreenOutlined-CSDUvZiD.js                                                                    1.59 kB │ gzip:   0.86 kB
dist/js/do-reload-form-modal.vue_vue_type_script_setup_true_lang-BToSlGQb.js                              1.60 kB │ gzip:   0.94 kB
dist/js/index-B8mZ9r5f.js                                                                                 1.62 kB │ gzip:   0.86 kB
dist/js/UrlOperate-kWsxJSji.js                                                                            1.63 kB │ gzip:   0.90 kB
dist/js/CloudUploadOutlined-CF2XWltL.js                                                                   1.64 kB │ gzip:   0.98 kB
dist/js/UserAddOutlined-BGnDzYtt.js                                                                       1.66 kB │ gzip:   0.95 kB
dist/js/index-pe1d2Vxw.js                                                                                 1.69 kB │ gzip:   0.92 kB
dist/js/metric-utils-CS9quiXT.js                                                                          1.69 kB │ gzip:   1.14 kB
dist/js/help-doc-manage-list-DocvyCnL.js                                                                  1.71 kB │ gzip:   0.79 kB
dist/js/member-editor-BRqLQskX.js                                                                         1.71 kB │ gzip:   0.79 kB
dist/js/index-RgAfokXY.js                                                                                 1.72 kB │ gzip:   0.81 kB
dist/js/category-form-modal.vue_vue_type_script_setup_true_lang-CoIMje80.js                               1.73 kB │ gzip:   1.07 kB
dist/js/QrcodeOutlined-DvxNGQs4.js                                                                        1.77 kB │ gzip:   0.75 kB
dist/js/ExamResultCard-Z4MRoyJ5.js                                                                        1.78 kB │ gzip:   1.04 kB
dist/js/index-C_Wo1jQq.js                                                                                 1.78 kB │ gzip:   0.96 kB
dist/js/home-quick-entry-modal.vue_vue_type_script_setup_true_lang-BDLWxbQL.js                            1.78 kB │ gzip:   1.03 kB
dist/js/index-3JSs_cnh.js                                                                                 1.79 kB │ gzip:   1.21 kB
dist/js/dict-form-modal.vue_vue_type_script_setup_true_lang-Du2Hrgw9.js                                   1.79 kB │ gzip:   1.05 kB
dist/js/channel-detail-modal-gdgwRu6L.js                                                                  1.81 kB │ gzip:   0.95 kB
dist/js/index-CMvC1FzY.js                                                                                 1.82 kB │ gzip:   1.01 kB
dist/js/module-list-DhSDET_P.js                                                                           1.82 kB │ gzip:   0.75 kB
dist/js/notice-form-visible-transfer-department-DfYoBtL6.js                                               1.89 kB │ gzip:   1.02 kB
dist/js/helper-DO7GiL3O.js                                                                                1.89 kB │ gzip:   1.06 kB
dist/js/data-masking-list-BzU8v6CH.js                                                                     1.90 kB │ gzip:   1.32 kB
dist/js/template-send-helper-DHVwuTcS.js                                                                  1.90 kB │ gzip:   1.06 kB
dist/js/gateway-detail-modal.vue_vue_type_script_setup_true_lang-FolXMy__.js                              1.92 kB │ gzip:   0.82 kB
dist/js/serial-number-record-list.vue_vue_type_script_setup_true_lang-T9UlGjoe.js                         1.92 kB │ gzip:   1.12 kB
dist/js/display-factory-card-B-uC4hjT.js                                                                  1.92 kB │ gzip:   0.98 kB
dist/js/index-BNZ9gcdY.js                                                                                 1.99 kB │ gzip:   1.11 kB
dist/js/BaseTable-BV1pIEUm.js                                                                             2.04 kB │ gzip:   1.07 kB
dist/js/metric-card-LSg5RBkF.js                                                                           2.04 kB │ gzip:   1.02 kB
dist/js/cache-list-aINselYA.js                                                                            2.05 kB │ gzip:   1.42 kB
dist/js/PositionMap.vue_vue_type_script_setup_true_lang-BW51PSXw.js                                       2.05 kB │ gzip:   1.27 kB
dist/js/menu-list-CZsX0xgG.js                                                                             2.06 kB │ gzip:   0.82 kB
dist/js/item-point-bind-modal.vue_vue_type_script_setup_true_lang-cw-iSsaM.js                             2.07 kB │ gzip:   1.35 kB
dist/js/dict-form-modal.vue_vue_type_script_setup_true_lang-5I2Il4xL.js                                   2.07 kB │ gzip:   1.14 kB
dist/js/index-DXwtBmm6.js                                                                                 2.09 kB │ gzip:   1.22 kB
dist/js/index-C-E8n6Bi.js                                                                                 2.12 kB │ gzip:   1.61 kB
dist/js/account-menu-DEFyEoYw.js                                                                          2.14 kB │ gzip:   0.99 kB
dist/js/instance-detail-modal.vue_vue_type_script_setup_true_lang-BxncSKME.js                             2.15 kB │ gzip:   0.99 kB
dist/js/FilterOutlined-DuxCu79w.js                                                                        2.17 kB │ gzip:   0.95 kB
dist/js/serial-number-generate-form-modal.vue_vue_type_script_setup_true_lang-C_etbvoc.js                 2.19 kB │ gzip:   1.10 kB
dist/js/tenant-admin-modal.vue_vue_type_script_setup_true_lang-CGtTy-6S.js                                2.19 kB │ gzip:   1.29 kB
dist/js/notice-employee-detail-NZZ0kx_D.js                                                                2.20 kB │ gzip:   1.24 kB
dist/js/index-C18V3cfi.js                                                                                 2.20 kB │ gzip:   1.26 kB
dist/js/metric-detail-modal.vue_vue_type_script_setup_true_lang-DFIYqjmv.js                               2.21 kB │ gzip:   1.00 kB
dist/js/help-doc-user-view-DQXjurjA.js                                                                    2.21 kB │ gzip:   1.19 kB
dist/js/follow-RTo_Qmxn.js                                                                                2.26 kB │ gzip:   1.35 kB
dist/js/index-BpNSuknc.js                                                                                 2.28 kB │ gzip:   1.28 kB
dist/js/help-doc-catalog-form-modal.vue_vue_type_script_setup_true_lang-DuWt6MRV.js                       2.28 kB │ gzip:   1.26 kB
dist/js/TextListModal-B-MAcMgo.js                                                                         2.28 kB │ gzip:   1.26 kB
dist/js/config-form-modal.vue_vue_type_script_setup_true_lang-CS334zHk.js                                 2.29 kB │ gzip:   1.18 kB
dist/js/import-Dt1skfA_.js                                                                                2.29 kB │ gzip:   1.56 kB
dist/js/gradient-BFkTcjf6.js                                                                              2.32 kB │ gzip:   1.06 kB
dist/js/chart-BlexBQqc.js                                                                                 2.37 kB │ gzip:   1.12 kB
dist/js/device-component-zone.vue_vue_type_script_setup_true_lang-60HXkWBV.js                             2.42 kB │ gzip:   1.33 kB
dist/js/index-DBr9Pvrl.js                                                                                 2.45 kB │ gzip:   1.35 kB
dist/js/position-form.vue_vue_type_script_setup_true_lang-DZxTTodC.js                                     2.48 kB │ gzip:   1.28 kB
dist/js/device-realtime-zone.vue_vue_type_script_setup_true_lang-DZMN9TFa.js                              2.50 kB │ gzip:   1.35 kB
dist/js/index-89-DnpCl.js                                                                                 2.51 kB │ gzip:   1.21 kB
dist/js/notice-form-visible-transfer-employee-Dzpili8o.js                                                 2.51 kB │ gzip:   1.31 kB
dist/js/display-api-DGEbu1mV.js                                                                           2.54 kB │ gzip:   1.01 kB
dist/js/keyword-match-modal-Cqa_HMeE.js                                                                   2.54 kB │ gzip:   1.40 kB
dist/js/index-BbrP4l39.js                                                                                 2.58 kB │ gzip:   1.12 kB
dist/js/SettingOutlined-CfWoPUPA.js                                                                       2.60 kB │ gzip:   1.31 kB
dist/js/index-D9iThMi0.js                                                                                 2.63 kB │ gzip:   1.35 kB
dist/js/index-DCuWks1H.js                                                                                 2.63 kB │ gzip:   1.57 kB
dist/js/code-generator-table-config-form-table-field.vue_vue_type_script_setup_true_lang-CIXxvq1t.js      2.63 kB │ gzip:   1.22 kB
dist/js/index-DYSsxMoK.js                                                                                 2.65 kB │ gzip:   1.27 kB
dist/js/item-formula-rollup-modal.vue_vue_type_script_setup_true_lang-B5Iw_xI5.js                         2.66 kB │ gzip:   1.70 kB
dist/js/device-status-zone.vue_vue_type_script_setup_true_lang-D9IqYcDL.js                                2.67 kB │ gzip:   1.30 kB
dist/js/dict-data-form-modal.vue_vue_type_script_setup_true_lang-DIerMxm9.js                              2.69 kB │ gzip:   1.42 kB
dist/js/message-receiver-modal.vue_vue_type_script_setup_true_lang-DKHpHafi.js                            2.70 kB │ gzip:   1.35 kB
dist/js/reload-list-DQOzo954.js                                                                           2.73 kB │ gzip:   1.78 kB
dist/js/device-utils-6k4pcSn7.js                                                                          2.77 kB │ gzip:   1.48 kB
dist/js/dept-form-modal.vue_vue_type_script_setup_true_lang-CwJY1Hu1.js                                   2.77 kB │ gzip:   1.46 kB
dist/js/category-tree-table.vue_vue_type_script_setup_true_lang-zajHR4X3.js                               2.79 kB │ gzip:   1.43 kB
dist/js/job-form-modal.vue_vue_type_script_setup_true_lang-rHx8z1Qx.js                                    2.79 kB │ gzip:   1.45 kB
dist/js/metric-trend-DWe8gF9O.js                                                                          2.80 kB │ gzip:   1.70 kB
dist/js/enterprise-bank-operate-modal.vue_vue_type_script_setup_true_lang-DExGCiDH.js                     2.85 kB │ gzip:   1.38 kB
dist/js/MatrixCard-vt9LsaiG.js                                                                            2.85 kB │ gzip:   1.30 kB
dist/js/notice-form-modal.vue_vue_type_script_setup_true_lang-CebG-b7S.js                                 2.85 kB │ gzip:   1.42 kB
dist/js/help-doc-view-record-list-_3hrUkyl.js                                                             2.88 kB │ gzip:   1.49 kB
dist/js/message-send-form.vue_vue_type_script_setup_true_lang-sQjvbVG5.js                                 2.88 kB │ gzip:   1.43 kB
dist/js/code-generator-table-config-form-delete-xN_DfBLp.js                                               2.91 kB │ gzip:   1.47 kB
dist/js/index-DvubI4GH.js                                                                                 2.92 kB │ gzip:   1.46 kB
dist/js/index-CljZEBG0.js                                                                                 2.94 kB │ gzip:   1.65 kB
dist/js/enterprise-invoice-operate-modal.vue_vue_type_script_setup_true_lang-3ATTIoha.js                  2.99 kB │ gzip:   1.40 kB
dist/js/dict-import-modal.vue_vue_type_script_setup_true_lang-BMOkvp1b.js                                 3.00 kB │ gzip:   1.70 kB
dist/js/change-log-form.vue_vue_type_script_setup_true_lang-1P-w5RjH.js                                   3.03 kB │ gzip:   1.45 kB
dist/js/serial-number-list-Lzy3jdFd.js                                                                    3.05 kB │ gzip:   1.90 kB
dist/js/dict-data-form-modal.vue_vue_type_script_setup_true_lang-BB4t0YJa.js                              3.06 kB │ gzip:   1.42 kB
dist/js/home-to-be-done-DL5sdnRg.js                                                                       3.06 kB │ gzip:   1.32 kB
dist/js/index-BXOULHqS.js                                                                                 3.07 kB │ gzip:   1.57 kB
dist/js/index-d9739DtR.js                                                                                 3.09 kB │ gzip:   1.70 kB
dist/js/snapshot-drawer-BoBk-IpU.js                                                                       3.11 kB │ gzip:   1.66 kB
dist/js/TeamOutlined-BrOf-7Py.js                                                                          3.14 kB │ gzip:   1.34 kB
dist/js/index-CyawQLeT.js                                                                                 3.16 kB │ gzip:   1.60 kB
dist/js/submit-DUtxc5S5.js                                                                                3.18 kB │ gzip:   1.69 kB
dist/js/notice-view-record-list.vue_vue_type_script_setup_true_lang-Dqu1_oLg.js                           3.20 kB │ gzip:   1.55 kB
dist/js/public-BQdEfOCE.js                                                                                3.20 kB │ gzip:   1.59 kB
dist/js/role-tree-menu-DFkDdQTT.js                                                                        3.21 kB │ gzip:   1.50 kB
dist/js/login-DaC9NS6D.js                                                                                 3.23 kB │ gzip:   1.66 kB
dist/js/feedback-list-joWEAl3u.js                                                                         3.24 kB │ gzip:   1.61 kB
dist/js/dict-form-modal.vue_vue_type_script_setup_true_lang-do4l5n5B.js                                   3.30 kB │ gzip:   1.75 kB
dist/js/formatTableColumn-CHsnCt-I.js                                                                     3.31 kB │ gzip:   1.16 kB
dist/js/template-form-modal.vue_vue_type_script_setup_true_lang-D5py46WR.js                               3.32 kB │ gzip:   1.75 kB
dist/js/index-BQOQ8FHj.js                                                                                 3.37 kB │ gzip:   1.80 kB
dist/js/code-generator-util-X5KUoHCl.js                                                                   3.40 kB │ gzip:   0.90 kB
dist/js/index-CUkMHrBU.js                                                                                 3.44 kB │ gzip:   1.76 kB
dist/js/help-doc-layout-Bfb2LsXM.js                                                                       3.47 kB │ gzip:   1.68 kB
dist/js/category-CfQJvjqa.js                                                                              3.49 kB │ gzip:   1.61 kB
dist/js/instance-params-modal.vue_vue_type_script_setup_true_lang-CXpyrurv.js                             3.50 kB │ gzip:   1.74 kB
dist/js/index-Bo7CVtsu.js                                                                                 3.51 kB │ gzip:   1.57 kB
dist/js/category-CkhZDyIa.js                                                                              3.51 kB │ gzip:   1.62 kB
dist/js/heart-beat-list-CmBokJyz.js                                                                       3.58 kB │ gzip:   2.08 kB
dist/js/point-detail-modal.vue_vue_type_script_setup_true_lang-BKZt0Fn8.js                                3.64 kB │ gzip:   1.66 kB
dist/js/operate-log-detail-modal-DdSNSXPB.js                                                              3.65 kB │ gzip:   1.58 kB
dist/js/notice-employee-list.vue_vue_type_script_setup_true_lang-HKvu3u9z.js                              3.65 kB │ gzip:   1.75 kB
dist/js/gateway-form-modal.vue_vue_type_script_setup_true_lang-D-WruqMM.js                                3.65 kB │ gzip:   1.90 kB
dist/js/mass-send-detail-drawer.vue_vue_type_script_setup_true_lang-BJIXmKNu.js                           3.69 kB │ gzip:   1.72 kB
dist/js/config-list-CHmm8H8N.js                                                                           3.70 kB │ gzip:   1.74 kB
dist/js/home-quick-entry-D45o2Wx0.js                                                                      3.70 kB │ gzip:   1.77 kB
dist/js/tenant-form-modal.vue_vue_type_script_setup_true_lang-CYhw0pFT.js                                 3.72 kB │ gzip:   1.76 kB
dist/js/card-Pxb_aL4Z.js                                                                                  3.76 kB │ gzip:   1.68 kB
dist/js/notice-detail-5-jHB6AC.js                                                                         3.76 kB │ gzip:   1.70 kB
dist/js/index-D4WRdip3.js                                                                                 3.80 kB │ gzip:   1.80 kB
dist/js/redpack-BhO4vori.js                                                                               3.83 kB │ gzip:   2.16 kB
dist/js/config-operate-drawer-B509gSpf.js                                                                 3.83 kB │ gzip:   1.86 kB
dist/js/index-CLpHRbqY.js                                                                                 3.91 kB │ gzip:   1.81 kB
dist/js/device-trace-zone.vue_vue_type_script_setup_true_lang-fzg3pwPZ.js                                 3.95 kB │ gzip:   1.93 kB
dist/js/table-mwCZts7n.js                                                                                 3.97 kB │ gzip:   1.69 kB
dist/js/mass-send-create-modal.vue_vue_type_script_setup_true_lang-CrTGHSM0.js                            4.01 kB │ gzip:   2.05 kB
dist/js/DataPush-D3HewU5f.js                                                                              4.02 kB │ gzip:   2.08 kB
dist/js/device-curve-zone.vue_vue_type_script_setup_true_lang-CZsYfWE3.js                                 4.04 kB │ gzip:   2.22 kB
dist/js/ThunderboltOutlined-B-fH3Z-h.js                                                                   4.07 kB │ gzip:   1.49 kB
dist/js/analysis-C_mQUYk7.js                                                                              4.07 kB │ gzip:   2.06 kB
dist/js/publish-log-drawer-Ct-Cjbqg.js                                                                    4.08 kB │ gzip:   1.96 kB
dist/js/panel-D6hvg7QJ.js                                                                                 4.11 kB │ gzip:   2.08 kB
dist/js/code-generator-table-config-form-ROxMGl0p.js                                                      4.13 kB │ gzip:   1.70 kB
dist/js/cross-BCrapybl.js                                                                                 4.13 kB │ gzip:   2.03 kB
dist/js/index-BCjUrBZD.js                                                                                 4.13 kB │ gzip:   2.22 kB
dist/js/index-DNoTcG1s.js                                                                                 4.13 kB │ gzip:   2.05 kB
dist/js/index-LsnIhcJ3.js                                                                                 4.17 kB │ gzip:   2.17 kB
dist/js/index-CATiHU6v.js                                                                                 4.18 kB │ gzip:   1.81 kB
dist/js/index-C6jfo01H.js                                                                                 4.19 kB │ gzip:   1.96 kB
dist/js/code-generator-table-config-form-query-field.vue_vue_type_script_setup_true_lang-6ViZ1uk6.js      4.20 kB │ gzip:   1.84 kB
dist/js/index-Ze_Pz9UP.js                                                                                 4.24 kB │ gzip:   2.19 kB
dist/js/dict-data-modal.vue_vue_type_script_setup_true_lang-6CKSQU60.js                                   4.28 kB │ gzip:   2.12 kB
dist/js/notify-BME2f4b3.js                                                                                4.28 kB │ gzip:   2.18 kB
dist/js/FileOutlined-MGR7PYc5.js                                                                          4.29 kB │ gzip:   1.17 kB
dist/js/index-Bdk29Lwj.js                                                                                 4.29 kB │ gzip:   2.16 kB
dist/js/index-Ck6kVJPX.js                                                                                 4.34 kB │ gzip:   2.20 kB
dist/js/share-CknchH5e.js                                                                                 4.37 kB │ gzip:   1.98 kB
dist/js/position-list-BNgcnB4k.js                                                                         4.39 kB │ gzip:   2.03 kB
dist/js/module-list.vue_vue_type_script_setup_true_lang-Z5PdUNC7.js                                       4.41 kB │ gzip:   2.02 kB
dist/js/index-DA-pKswG.js                                                                                 4.45 kB │ gzip:   2.56 kB
dist/js/template-send-detail-drawer-D5USzfu-.js                                                           4.45 kB │ gzip:   2.06 kB
dist/js/code-generator-list-CAYPjRrK.js                                                                   4.51 kB │ gzip:   2.00 kB
dist/js/module-form-BC_uOFIo.js                                                                           4.56 kB │ gzip:   2.29 kB
dist/js/index-B3JjrriL.js                                                                                 4.57 kB │ gzip:   2.14 kB
dist/js/job-log-list-modal.vue_vue_type_script_setup_true_lang-C0yfJh9T.js                                4.62 kB │ gzip:   2.06 kB
dist/js/job-log-modal.vue_vue_type_script_setup_true_lang-RcdYZNxP.js                                     4.69 kB │ gzip:   2.15 kB
dist/js/DataFilter-C-n4ybhP.js                                                                            4.72 kB │ gzip:   1.88 kB
dist/js/dict-data-modal.vue_vue_type_script_setup_true_lang-DukNSv4n.js                                   4.79 kB │ gzip:   2.20 kB
dist/js/help-doc-list.vue_vue_type_script_setup_true_lang-BSnm8YnE.js                                     4.79 kB │ gzip:   2.19 kB
dist/js/login-fail-list-DxZzC9WN.js                                                                       4.81 kB │ gzip:   2.19 kB
dist/js/enterprise-invoice-list.vue_vue_type_script_setup_true_lang-e-5WBCAx.js                           4.81 kB │ gzip:   2.30 kB
dist/js/apiclient-form-modal-NJgAueK9.js                                                                  4.82 kB │ gzip:   2.48 kB
dist/js/enterprise-bank-list.vue_vue_type_script_setup_true_lang-Dl9VqMwZ.js                              4.82 kB │ gzip:   2.27 kB
dist/js/create-D3X_rqlT.js                                                                                4.85 kB │ gzip:   2.40 kB
dist/js/index-K1FBxK_G.js                                                                                 4.86 kB │ gzip:   2.06 kB
dist/js/index-CTXuNJ-O.js                                                                                 4.94 kB │ gzip:   2.29 kB
dist/js/ZoomOutOutlined-CrUluwAz.js                                                                       5.02 kB │ gzip:   1.43 kB
dist/js/message-list-B7mOJ_g9.js                                                                          5.12 kB │ gzip:   2.27 kB
dist/js/keyword-form-modal.vue_vue_type_script_setup_true_lang-BlBhHIpm.js                                5.15 kB │ gzip:   2.35 kB
dist/js/instance-item-drawer-CtL1NmDN.js                                                                  5.18 kB │ gzip:   2.49 kB
dist/js/index-CTogYwSI.js                                                                                 5.18 kB │ gzip:   2.22 kB
dist/js/layout-const-CD0qf-xH.js                                                                          5.21 kB │ gzip:   2.63 kB
dist/js/job-form-modal.vue_vue_type_script_setup_true_lang-BsGNESZ_.js                                    5.33 kB │ gzip:   2.17 kB
dist/js/index-CyJBUNnB.js                                                                                 5.44 kB │ gzip:   2.35 kB
dist/js/index-Bmvb4KQF.js                                                                                 5.45 kB │ gzip:   2.52 kB
dist/js/help-doc-form-drawer-BQsqQzmI.js                                                                  5.46 kB │ gzip:   2.54 kB
dist/js/help-doc-catalog-tree-CiZLdbTk.js                                                                 5.46 kB │ gzip:   2.48 kB
dist/js/template-send-create-modal-BtSGWJfK.js                                                            5.48 kB │ gzip:   2.92 kB
dist/js/code-generator-table-config-form-field.vue_vue_type_script_setup_true_lang-CwaklRqU.js            5.58 kB │ gzip:   2.27 kB
dist/js/index-BsCC_9Av.js                                                                                 5.58 kB │ gzip:   2.46 kB
dist/js/index-B9VF74CM.js                                                                                 5.59 kB │ gzip:   2.38 kB
dist/js/config-list.vue_vue_type_script_setup_true_lang-CWO0DavN.js                                       5.63 kB │ gzip:   2.32 kB
dist/js/index-Rmdgo-So.js                                                                                 5.69 kB │ gzip:   2.32 kB
dist/js/index-CoaXyCHJ.js                                                                                 5.70 kB │ gzip:   2.58 kB
dist/js/iot-formula-editor-DpwIhFt9.js                                                                    5.72 kB │ gzip:   2.89 kB
dist/js/index-BI94v6er.js                                                                                 5.73 kB │ gzip:   2.48 kB
dist/js/index.vue_vue_type_style_index_0_lang-TKrjMzQa.js                                                 5.74 kB │ gzip:   1.88 kB
dist/js/enterprise-list-B5RrOQt1.js                                                                       5.77 kB │ gzip:   2.65 kB
dist/js/index-CTSzIpxT.js                                                                                 5.81 kB │ gzip:   2.66 kB
dist/js/instance-form-modal.vue_vue_type_script_setup_true_lang-DLsyw4OX.js                               5.89 kB │ gzip:   2.79 kB
dist/js/point-form-modal.vue_vue_type_script_setup_true_lang-DKnsL7DU.js                                  5.94 kB │ gzip:   2.59 kB
dist/js/api-encrypt-index-O-2F3Wpx.js                                                                     5.99 kB │ gzip:   1.74 kB
dist/js/goods-form-modal.vue_vue_type_script_setup_true_lang-1nvBehmA.js                                  6.11 kB │ gzip:   2.58 kB
dist/js/file-list-BNQ2ro_1.js                                                                             6.14 kB │ gzip:   2.44 kB
dist/js/index-DEASGYZK.js                                                                                 6.15 kB │ gzip:   2.77 kB
dist/js/level3-protect-config-index-DPY4YO9m.js                                                           6.20 kB │ gzip:   2.81 kB
dist/js/index-BAFB0O9Z.js                                                                                 6.20 kB │ gzip:   2.88 kB
dist/js/channel-credential-drawer-Fkyr947z.js                                                             6.20 kB │ gzip:   3.11 kB
dist/js/notice-form-drawer-C7awOnqH.js                                                                    6.22 kB │ gzip:   2.70 kB
dist/js/deleted-job-list.vue_vue_type_script_setup_true_lang-Cud-BHhM.js                                  6.23 kB │ gzip:   2.61 kB
dist/js/login-log-list-CdqAygcQ.js                                                                        6.24 kB │ gzip:   2.75 kB
dist/js/change-log-list-CPD7Kv7X.js                                                                       6.25 kB │ gzip:   2.55 kB
dist/js/index-28uhnp3P.js                                                                                 6.28 kB │ gzip:   2.56 kB
dist/js/index-By8sSTwC.js                                                                                 6.36 kB │ gzip:   2.62 kB
dist/js/index-C2ivei5R.js                                                                                 6.47 kB │ gzip:   2.70 kB
dist/js/channel-form-modal.vue_vue_type_script_setup_true_lang-B1TySj5J.js                                6.52 kB │ gzip:   3.21 kB
dist/js/ViewData-gzPn44iO.js                                                                              6.56 kB │ gzip:   2.46 kB
dist/js/index-mzO4p4at.js                                                                                 6.58 kB │ gzip:   2.72 kB
dist/js/index-DJi9twKl.js                                                                                 6.68 kB │ gzip:   3.13 kB
dist/js/index-D9R8TtBg.js                                                                                 6.71 kB │ gzip:   2.85 kB
dist/js/ViewOrUpdate-Ctk2c6Wf.js                                                                          6.74 kB │ gzip:   3.25 kB
dist/js/index-DlwiZI0p.js                                                                                 6.80 kB │ gzip:   2.99 kB
dist/js/index-DdAbrUlO.js                                                                                 6.89 kB │ gzip:   2.61 kB
dist/js/code-generator-table-config-form-insert-and-update-BapSp_gg.js                                    7.05 kB │ gzip:   2.56 kB
dist/js/metric-form-modal-M9Hqp3Zs.js                                                                     7.11 kB │ gzip:   3.01 kB
dist/js/publish-confirm-modal-BjtCBPcB.js                                                                 7.22 kB │ gzip:   3.50 kB
dist/js/code-generator-table-config-form-basic-BGT6bMta.js                                                7.39 kB │ gzip:   2.78 kB
dist/js/index-CvnenzLy.js                                                                                 7.41 kB │ gzip:   3.26 kB
dist/js/notice-list-BIzS_zdN.js                                                                           7.45 kB │ gzip:   2.90 kB
dist/js/menu-list.vue_vue_type_script_setup_true_lang-oM_S-bqx.js                                         7.50 kB │ gzip:   2.93 kB
dist/js/index-C4T4sheo.js                                                                                 7.55 kB │ gzip:   3.53 kB
dist/js/operate-log-list-DvS2x3KP.js                                                                      7.65 kB │ gzip:   3.21 kB
dist/js/index-DZSOTSIw.js                                                                                 7.72 kB │ gzip:   3.26 kB
dist/js/job-list-aJSSTyJo.js                                                                              8.38 kB │ gzip:   3.34 kB
dist/js/index.vue_vue_type_script_setup_true_lang-BxjImxH5.js                                             8.59 kB │ gzip:   3.68 kB
dist/js/enterprise-employee-list-CeE9Yw4Z.js                                                              8.63 kB │ gzip:   3.32 kB
dist/js/index-C0jcw5AH.js                                                                                 8.75 kB │ gzip:   3.22 kB
dist/js/index-DUW_6t6h.js                                                                                 8.99 kB │ gzip:   4.14 kB
dist/js/index-DFUV_rAz.js                                                                                 9.12 kB │ gzip:   3.90 kB
dist/js/index-tUR9rMEJ.js                                                                                 9.55 kB │ gzip:   4.01 kB
dist/js/index-BTgrXi76.js                                                                                 9.67 kB │ gzip:   3.82 kB
dist/js/index-B0Xuon0F.js                                                                                 9.74 kB │ gzip:   3.94 kB
dist/js/index-Cw_zddCL.js                                                                                 9.86 kB │ gzip:   4.05 kB
dist/js/index-BGS8eLMk.js                                                                                10.17 kB │ gzip:   3.82 kB
dist/js/template-item-modal.vue_vue_type_script_setup_true_lang-BGF_l_4O.js                              10.39 kB │ gzip:   4.03 kB
dist/js/SendOutlined-BAl_ghTq.js                                                                         10.86 kB │ gzip:   2.66 kB
dist/js/index-DmqFOPUe.js                                                                                10.94 kB │ gzip:   4.74 kB
dist/js/index-7sPJMVio.js                                                                                11.15 kB │ gzip:   4.35 kB
dist/js/index-fDkzEEAL.js                                                                                11.17 kB │ gzip:   3.31 kB
dist/js/menu-operate-modal-C3q0kbbJ.js                                                                   11.36 kB │ gzip:   4.13 kB
dist/js/write-C_qNgCKW.js                                                                                12.75 kB │ gzip:   3.98 kB
dist/js/goods-list-DF55i1Kh.js                                                                           15.49 kB │ gzip:   5.61 kB
dist/js/index-Nq5dfe0A.js                                                                                16.01 kB │ gzip:   6.63 kB
dist/js/ua-parser-Bbu8swv-.js                                                                            16.51 kB │ gzip:   7.33 kB
dist/js/EnterOutlined-B5EuV85C.js                                                                        18.83 kB │ gzip:   3.43 kB
dist/js/VideoCameraOutlined-CL7VXU-B.js                                                                  20.27 kB │ gzip:   4.81 kB
dist/js/qrcode.vue.esm-CyaV9a5q.js                                                                       20.75 kB │ gzip:   7.31 kB
dist/js/bundle.esm-D_bxrnq8.js                                                                           20.76 kB │ gzip:   5.97 kB
dist/js/index-C0-QzLeg.js                                                                                25.00 kB │ gzip:   8.49 kB
dist/js/avatar-cropper-modal-9eGbvLYS.js                                                                 35.31 kB │ gzip:  10.36 kB
dist/js/sortable.esm-CRp0PNdb.js                                                                         35.52 kB │ gzip:  12.26 kB
dist/js/index.esm-B4d0L6QS.js                                                                            45.58 kB │ gzip:  17.23 kB
dist/js/enterprise-detail-4eOnKI1M.js                                                                    57.70 kB │ gzip:  17.09 kB
dist/js/index-yhoIbBJp.js                                                                                57.90 kB │ gzip:  14.30 kB
dist/js/enterprise-operate-modal-DFvEu6Ma.js                                                            151.85 kB │ gzip:  34.98 kB
dist/js/index-CtrUK5tZ.js                                                                               170.10 kB │ gzip:  49.29 kB
dist/js/home-header-C03icUw1.js                                                                         294.39 kB │ gzip: 102.76 kB
dist/js/index.esm-DW1Lcunr.js                                                                           458.39 kB │ gzip: 137.65 kB
dist/js/index.esm-DnZ9rX2D.js                                                                           547.28 kB │ gzip: 157.18 kB
dist/js/index-CuOlWmH4.js                                                                               652.71 kB │ gzip: 205.35 kB
dist/js/code-generator-preview-modal-DBxC0uY1.js                                                        915.72 kB │ gzip: 303.29 kB
dist/js/index-aT9mlqw0.js                                                                               928.90 kB │ gzip: 274.04 kB
dist/js/index-BEX-QJne.js                                                                               992.43 kB │ gzip: 159.99 kB
dist/js/index-YnO6xPSo.js                                                                             1,008.31 kB │ gzip: 324.79 kB
dist/js/index-BAsMtFso.js                                                                             1,015.77 kB │ gzip: 329.40 kB
dist/js/index-CtKOeYUs.js                                                                             1,316.21 kB │ gzip: 432.27 kB

(!) Some chunks are larger than 500 kB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking: https://rollupjs.org/configuration-options/#output-manualchunks
- Adjust chunk size limit for this warning via build.chunkSizeWarningLimit.
✓ built in 2m 4s```

### 2.2 `pnpm lint` — EXIT=0

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
  0 errors and 14 warnings potentially fixable with the `--fix` option.```

### 2.3 `pnpm typecheck` — EXIT=0

```text
$ vue-tsc --noEmit```

### 2.4 `pnpm test` — EXIT=0

```text
$ vitest run

 RUN  v2.1.9 D:/workspace/WisFactory/frontend

 ✓ tests/unit/components/iot/formula-rules.spec.ts (32 tests) 59ms
 ✓ tests/unit/lib/axios.spec.ts (27 tests) 356ms
 ✓ tests/unit/composables/survey/use-edit-lock.spec.ts (12 tests) 107ms
 ✓ tests/unit/wechat/template-send-helper.spec.ts (17 tests) 21ms
 ✓ tests/unit/router/router-guard.spec.ts (8 tests) 125ms
 ✓ tests/unit/router/build-routes.spec.ts (15 tests) 93ms
 ✓ tests/unit/components/iot/iot-formula-editor.spec.ts (15 tests) 1959ms
   ✓ 公式编辑器：渲染与实时校验 > 初始公式渲染到 textarea，合法公式无错误提示 437ms
 ✓ tests/unit/views/iot/item-formula-rollup-modal.spec.ts (5 tests) 2197ms
   ✓ 实例数据项取值编辑弹窗（TASK-0094） > formula 模式：父 ref 可调 showModal，回填公式并按清单构造 SELF 引用分组 729ms
   ✓ 实例数据项取值编辑弹窗（TASK-0094） > formula 模式：空公式/语法错误拦截，不调 saveFormula 382ms
   ✓ 实例数据项取值编辑弹窗（TASK-0094） > formula 模式：合法公式 → saveFormula(1001,2002,formula) → 成功提示 + reloadList + 关窗 409ms
   ✓ 实例数据项取值编辑弹窗（TASK-0094） > rollup 模式：SUM/AVG 单选，合法方法 → saveRollup(1001,2002,"AVG") → 成功提示 + 关窗 335ms
   ✓ 实例数据项取值编辑弹窗（TASK-0094） > 保存失败：captureError + message.error 提示，弹窗不关闭 330ms
 ✓ tests/unit/display/metrics-board-read-ports.spec.ts (6 tests) 2949ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > metricValues→卡片同源展示：value/yoy/mom/unit 原样，不现算 1046ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > 同 metricCode 多行 ts 升序（去年对比行在前）→ 本期值取最大 ts 行，不取首行旧值（I-1） 400ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > null 伴随值保持空态，不冒充 0 462ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > completeness available=true 填 map，低于阈值标记可见 429ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > completeness available=false（当前诚实空集）→ 保持空态 + readGap 告警，不静默判完整 307ms
   ✓ 指标看板接真值读口（TASK-0089 D3） > 读口失败：metrics 定义仍在、值/完整度空态，不炸页面（局部 catch 降级） 301ms
 ✓ tests/unit/display/display-read-ports.spec.ts (10 tests) 13ms
 ✓ tests/unit/store/user-module.spec.ts (9 tests) 258ms
 ✓ tests/unit/wechat/mass-send-helper.spec.ts (15 tests) 13ms
 ✓ tests/unit/views/iot/point-bind-modal.spec.ts (4 tests) 3270ms
   ✓ 实例数据项绑定点位弹窗（TASK-0092） > 父 ref 可调 showModal；拉 status=1 启用点位列表并展示点位编码/名称/值类型 1320ms
   ✓ 实例数据项绑定点位弹窗（TASK-0092） > 选一行点确定 → bindPoint(1001,2002,301) → 成功提示 + 事件通知 + 关窗 896ms
   ✓ 实例数据项绑定点位弹窗（TASK-0092） > 未选择点位点确定 → 不调 bindPoint，提示先选择 417ms
   ✓ 实例数据项绑定点位弹窗（TASK-0092） > 绑定失败 → captureError + message.error 提示，弹窗不关闭 635ms
stderr | tests/unit/store/user-module.spec.ts
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}
Error: connect ECONNREFUSED 127.0.0.1:8098
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:2017:16) {
  errno: -4078,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 8098
}

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

 ✓ tests/unit/layout/top-expand-user-space.spec.ts (17 tests) 2934ms
   ✓ 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 顶行内同时渲染一级菜单与用户区（用户区不再挂在下方 header） 980ms
   ✓ 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 用户区排在菜单之后（DOM 顺序即视觉左右顺序：菜单在左、用户区在右） 511ms
   ✓ 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue） > 顶行不再渲染二级菜单（二级菜单回到侧栏，由 layout 直接挂载） 375ms
   ✓ 「顶部」布局复用同一用户区组件（top-menu/index.vue） > 渲染后用户区存在且与分组布局同源 536ms
 ✓ tests/unit/survey/report-format.spec.ts (22 tests) 14ms
 ✓ tests/unit/utils/public-utils-layering.spec.ts (5 tests) 17ms
 ✓ tests/unit/views/system/menu-cache-frame.spec.ts (7 tests) 10124ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 回显：后端 0/0 显示为未开启，1/1 显示为开启（null/0 不得误判为真） 3049ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 提交：布尔开关映射为 0/1（后端只接受 0/1，不接受布尔） 1731ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 提交：开启外链后 frameFlag=1 且带上 frameUrl 1615ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 提交：关闭外链时清空外链地址，避免库中残留旧地址与开关状态不一致 2083ms
   ✓ 菜单「是否缓存 / 是否外链」开关回显与提交 > 外链菜单仍需填组件路径（buildRoutes 会跳过无 component 的菜单项） 1641ms
 ✓ tests/unit/theme/disabled-button-text.spec.ts (9 tests) 17ms
 ✓ tests/unit/views/iot/metric.spec.ts (8 tests) 11875ms
   ✓ 指标管理页与表单校验（TASK-0095） > 列表加载真实后端数据、无 IotPlaceholder；查询参数透传 2000ms
   ✓ 指标管理页与表单校验（TASK-0095） > 详情按钮 → detail(id) → 详情弹窗展示（父 ref defineExpose） 1386ms
   ✓ 指标管理页与表单校验（TASK-0095） > 表单弹窗父 ref 可调用 showModal / validateRatio；比值型缺分子分母被拦 1053ms
   ✓ 指标管理页与表单校验（TASK-0095） > 新增态选实例后加载数据项下拉，比值型可选分子/分母并成功提交（评审 Important 回归） 1797ms
   ✓ 指标管理页与表单校验（TASK-0095） > 新增提交：calendarType 锁 1，非比值型 ratio_* 置空，metricCode 原样透传 1400ms
   ✓ 指标管理页与表单校验（TASK-0095） > 编辑提交：metricCode 携带原值（不可改语义），status 不进更新集，走 update 1731ms
   ✓ 指标管理页与表单校验（TASK-0095） > 启停走 changeStatus，删除走 remove，不经过 update 1311ms
   ✓ 指标管理页与表单校验（TASK-0095） > 批量生成：instanceId 作 query 入参、清单作 body；空清单/非法口径被前置拦截 1193ms
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

 ✓ tests/unit/layout/top-expand-side-menu.spec.ts (12 tests) 6008ms
   ✓ 分组布局左侧菜单固定显示（BUG-0097） > 无二级菜单（首页场景）时容器与 logo 区仍渲染且不被隐藏 325ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 「顶部」布局仍不显示该设置项（无左侧栏） 2335ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 分组布局显示该设置项，且取值回显 store 宽度 1298ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 传统布局仍显示该设置项（原行为不回归） 1242ms
   ✓ 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092） > 「展开」(side-expand) 不显示该设置项（侧栏宽度由内部固定值决定，调了不生效） 641ms
 ✓ tests/unit/views/iot/point.spec.ts (9 tests) 12505ms
   ✓ 点位管理页（TASK-0097） > 挂载拉列表（默认分页）且无占位骨架文案 2655ms
   ✓ 点位管理页（TASK-0097） > 空数据时展示诚实空态，不造假数据 1198ms
   ✓ 点位管理页（TASK-0097） > 新建/详情通过父 ref 调 defineExpose.showModal（BUG-0008 回归） 1354ms
   ✓ 点位管理页（TASK-0097） > 启停调 changeStatus({ids,status})，停用含发布生效提示 1507ms
   ✓ 点位管理页（TASK-0097） > 删除调 remove([id])（API 层拼路径参数），导出调 export 带过滤参数 1835ms
   ✓ 点位表单弹窗（TASK-0097） > 新增提交调 add，payload 不含 status/id 1149ms
   ✓ 点位表单弹窗（TASK-0097） > 编辑提交调 update，携带 id 与 pointCode 同值，且不含 status 1105ms
   ✓ 点位表单弹窗（TASK-0097） > 编辑 showModal 后 pointCode 禁用（创建后不可改） 742ms
   ✓ 点位详情弹窗（TASK-0097） > 父 ref 调 showModal(id) → detail/references/curve 三读，曲线空集展示诚实提示 953ms
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

 ✓ tests/unit/views/iot/instance-tree-edit.spec.ts (8 tests) 10776ms
   ✓ 实例树节点编辑入口（TASK-0093） > title 插槽在工厂/车间/设备各级节点旁渲染 EditOutlined 编辑按钮 2200ms
   ✓ 实例树节点编辑入口（TASK-0093） > 点击工厂节点编辑按钮 → 复用 addInstance 打开编辑弹窗（完整 record 传 showModal） 1224ms
   ✓ 实例树节点编辑入口（TASK-0093） > 骨架节点（缺 deptId/sort）点击 → 拉详情补齐字段后打开编辑弹窗 1123ms
   ✓ 实例树节点编辑入口（TASK-0093） > 详情加载失败时不开空弹窗，并上报 otSentry 1107ms
   ✓ 实例树节点编辑入口（TASK-0093） > locked 设备节点编辑按钮透传 locked=1（弹窗自身显示仅改状态提示） 1345ms
   ✓ 实例树节点编辑入口（TASK-0093） > 无 iot:instance:edit 权限时编辑按钮被 v-privilege 指令移除（树节点仍在） 914ms
   ✓ 实例树节点编辑入口（TASK-0093） > 树过滤仍保留命中节点及其祖先路径，且编辑按钮随树保留 1461ms
   ✓ 实例树节点编辑入口（TASK-0093） > 树节点选择仍触发 onSelectTree（点击标签文字检索子树，与编辑按钮独立） 1397ms
 ✓ tests/unit/utils/sanitize-html.spec.ts (10 tests) 216ms
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

 ✓ tests/unit/store/app-config-remote.spec.ts (9 tests) 40ms
stderr | tests/unit/layout/setting-drawer-persist.spec.ts > 设置抽屉触发链路（BUG-0098 源码层） > 设置按钮仍渲染（用户区功能不回归）
[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

 ✓ tests/unit/layout/setting-drawer-persist.spec.ts (7 tests) 7458ms
   ✓ 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > layout/index.vue 渲染设置抽屉一份 532ms
   ✓ 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > 发 settingMitt open 事件后抽屉打开 4264ms
   ✓ 设置抽屉挂在布局公共父级（BUG-0098 渲染层） > 切换「菜单布局」后抽屉仍打开（修复前会随布局组件卸载而消失） 2281ms
   ✓ 设置抽屉触发链路（BUG-0098 源码层） > 设置按钮仍渲染（用户区功能不回归） 334ms
 ✓ tests/unit/composables/survey/project-list-columns.spec.ts (19 tests) 34ms
 ✓ tests/unit/views/iot/instance-item-value-editor.spec.ts (4 tests) 7053ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > 公式项/汇总项各显示对应编辑按钮，权限不足时按钮被移除 2131ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > 点击「编辑公式」→ 弹窗打开 → 保存 → 调 saveFormula 并刷新清单 2342ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > 点击「编辑汇总」→ 弹窗打开 → 切 AVG 保存 → 调 saveRollup 并刷新清单 1582ms
   ✓ 实例数据项抽屉取值编辑入口（TASK-0094） > locked=1：编辑按钮禁用，点击提示锁定，不打开弹窗、不提交 994ms
 ✓ tests/unit/store/menu-module-helper.spec.ts (21 tests) 12ms
 ✓ tests/unit/layout/menu-path-match.spec.ts (9 tests) 7ms
 ✓ tests/unit/views/system/menu-perms-mapping.spec.ts (10 tests) 20361ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 编辑回显：输入框显示库中 perms（修复前恒空） 1885ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 库中 perms 为 null 时回显空串，不显示字面量 null 1194ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 改值提交：updateMenu 收到 params.perms 为目标值（修复前后端收不到该字段） 1681ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 清空提交：提交空串而非 null（null 会被后端 UPDATE 跳过、清不掉原权限串） 1660ms
   ✓ 菜单编辑弹窗「权限字符」→ perms 映射 > 新增功能点带权限串提交：addMenu 收到 params.perms 1323ms
   ✓ 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095） > 编辑回显：输入框显示库中 query（修复前绑 queryParam，后端无此字段故恒空） 2061ms
   ✓ 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095） > 改值提交：updateMenu 收到 params.query 为目标值，且不再有 queryParam 2545ms
   ✓ 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095） > 清空提交：提交空串而非 null（同 perms 口径，null 会被后端 UPDATE 跳过） 4088ms
   ✓ 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090） > 菜单类型抽屉无「路由名称」表单项 1163ms
   ✓ 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090） > 提交报文不含 routeName 2758ms
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

 ✓ tests/unit/views/system/role-tree-collapse.spec.ts (9 tests) 1885ms
   ✓ 角色授权树顶级分类折叠 > 默认全部展开：每个顶级分类都渲染折叠图标（收起态图标不出现） 559ms
 ✓ tests/unit/survey/write-storage.spec.ts (15 tests) 15ms
 ✓ tests/unit/views/survey/province-match.spec.ts (6 tests) 8ms
 ✓ tests/unit/monitor/actuator-api.spec.ts (10 tests) 13ms
 ✓ tests/unit/display/metric-utils.spec.ts (10 tests) 15ms
 ✓ tests/unit/api/iot/point-api.spec.ts (11 tests) 13ms
 ✓ tests/unit/views/survey/write-time-window.spec.ts (11 tests) 8ms
 ✓ tests/unit/display/device-utils.spec.ts (8 tests) 21ms
 ✓ tests/unit/store/ui-config-merge.spec.ts (9 tests) 12ms
 ✓ tests/unit/survey/wx-auth.spec.ts (18 tests) 37ms
 ✓ tests/unit/layout/top-expand-theme.spec.ts (9 tests) 1539ms
   ✓ 分组布局一级菜单主题联动（top-menu.vue） > 学术蓝（sideMenuTheme=light）→ 菜单渲染为 light，不含 dark 845ms
   ✓ 分组布局一级菜单主题联动（top-menu.vue） > 经典（sideMenuTheme=dark）→ 菜单渲染为 dark，保留原观感 310ms
   ✓ 分组布局一级菜单主题联动（top-menu.vue） > 主题切换后菜单明暗随之更新（响应式联动，非一次性求值） 320ms
 ✓ tests/unit/display/factory-card-status-values.spec.ts (5 tests) 1019ms
   ✓ 工厂卡片接状态数真值（TASK-0058 批次二 D2） > 挂载即用 node.id 取 factory-summary，运行/异常/离线三槽位填真值 552ms
 ✓ tests/unit/monitor/cache-monitor-api.spec.ts (5 tests) 11ms
 ✓ tests/unit/monitor/actuator-helper.spec.ts (12 tests) 39ms
 ✓ tests/unit/survey/option-label-map.spec.ts (10 tests) 9ms
 ✓ tests/unit/display/factory-summary-read-port.spec.ts (5 tests) 11ms
 ✓ tests/unit/api/iot/metric-api.spec.ts (8 tests) 11ms
 ✓ tests/unit/survey/redpack-helper.spec.ts (8 tests) 8ms
 ✓ tests/unit/views/iot/point-bind-drawer.spec.ts (2 tests) 1418ms
   ✓ 实例数据项抽屉绑定点位入口（TASK-0092） > 「绑定点位」按钮挂 iot:item:edit 权限；无该权限时按钮被移除 760ms
   ✓ 实例数据项抽屉绑定点位入口（TASK-0092） > 行内「绑定点位」按钮点击 → 拉启用点位列表 → 选点 → bindPoint → 刷新并显示点位编码 654ms
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

 ✓ tests/unit/layout/header-setting-persist.spec.ts (4 tests) 271ms
 ✓ tests/unit/views/wechat/tag-filter.spec.ts (13 tests) 9ms
 ✓ tests/unit/survey/redpack-list-helper.spec.ts (7 tests) 10ms
 ✓ tests/unit/monitor/cache-helper.spec.ts (11 tests) 8ms
 ✓ tests/unit/display/factory-page-metrics.spec.ts (3 tests) 722ms
   ✓ 工厂页接工厂级指标真值（TASK-0058 批次二 D2） > 工厂级指标看板：metricValues 同源展示 value/yoy/mom（不现算、null 不冒充 0） 412ms
 ✓ tests/unit/views/system/menu-list-perms-column.spec.ts (5 tests) 13ms
 ✓ tests/unit/store/dict.spec.ts (4 tests) 98ms
 ✓ tests/unit/survey/expand-original-data.spec.ts (6 tests) 9ms
 ✓ tests/unit/display/display-tree.spec.ts (5 tests) 21ms
 ✓ tests/unit/survey/option-label-display.spec.ts (12 tests) 793ms
   ✓ ViewData 答卷详情弹窗 > RADIO/CHECKBOX 显示选项文案 507ms
 ✓ tests/unit/display/device-realtime-zone.spec.ts (5 tests) 1668ms
   ✓ 设备实时区接 realtime 真值（TASK-0089 D2） > 绑定类 + 公式类统一取 realtime 真值（v/q/ts 同源），来源标注「实时值读口」 916ms
   ✓ 设备实时区接 realtime 真值（TASK-0089 D2） > v=null（无实时值）→ 该行显示「数据接入中」，不冒充 0 546ms
 ✓ tests/unit/utils/file-url.spec.ts (7 tests) 10ms
 ✓ tests/unit/store/app-config.spec.ts (3 tests) 608ms
   ✓ appConfig store > 老版本 localStorage 数据无 themeName 键时，store 初始化兜底为出厂默认（现为 academic） 587ms
 ✓ tests/unit/views/system/menu-field-mapping.spec.ts (4 tests) 14ms
 ✓ tests/unit/survey/normalize-field-schemes.spec.ts (5 tests) 9ms
 ✓ tests/unit/views/survey/redpack-columns.spec.ts (3 tests) 4889ms
   ✓ 问卷红包列表「发放人/发放结果」两列 > 列定义含「发放人」与「发放结果」两列，既有列不丢 2324ms
   ✓ 问卷红包列表「发放人/发放结果」两列 > 发放人列取 sendByName，缺失显示「—」（未发放行/老后端行） 1221ms
   ✓ 问卷红包列表「发放人/发放结果」两列 > 发放结果列五态文案与配色（成功绿/失败红/发放中蓝/待确认橙/未发放默认） 1340ms
 ✓ tests/unit/views/system/role-list-selection.spec.ts (8 tests) 1497ms
   ✓ 角色列表刷新后选中态（组件级） > 首次加载自动选中第一个角色 777ms
   ✓ 角色列表刷新后选中态（组件级） > 删除的正是当前选中角色时，回落到第一个 371ms
 ✓ tests/unit/theme/theme-preset.spec.ts (13 tests) 50ms
 ✓ tests/unit/api/survey/edit-lock-api.spec.ts (6 tests) 12ms
 ✓ tests/unit/api/iot/point-bind-api.spec.ts (4 tests) 18ms
 ✓ tests/unit/monitor/iot-inspect-helper.spec.ts (5 tests) 9ms
 ✓ tests/unit/display/display-routes.spec.ts (3 tests) 13ms
 ✓ tests/unit/monitor/alarm-helper.spec.ts (4 tests) 32ms
 ✓ tests/unit/views/system/login-entry-render.spec.ts (1 test) 579ms
   ✓ 登录页首屏组件渲染（入口精简集） > 只注册 entryComponents 时，用户名/密码/验证码三件套都能渲染 576ms
 ✓ tests/unit/views/system/menu-operate-modal.spec.ts (4 tests) 6686ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 编辑回显：库中「不显示」的菜单开关为不显示，未动开关提交不翻转原值 2758ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 关「显示状态」提交 visible=0（用户实测场景：修复前后端只收到旧值 1） 1479ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 关「菜单状态」提交 status=0，且可见性保持显示 1129ms
   ✓ 菜单编辑弹窗 visible/status 映射 > 新增菜单未动开关时提交 visible=1、status=1 1316ms
 ✓ tests/unit/wechat/send-common.spec.ts (4 tests) 10ms
 ✓ tests/unit/lib/encrypt.spec.ts (4 tests) 31ms
 ✓ tests/unit/survey/serial-number.spec.ts (6 tests) 6ms
 ✓ tests/unit/plugin/entry-ui.spec.ts (2 tests) 329ms
   ✓ plugin/entry-ui 入口精简集 > 登录表单（a-form / a-form-item / a-input-password）能渲染出密码输入框 317ms
 ✓ tests/unit/views/system/login-captcha-polling.spec.ts (2 tests) 672ms
   ✓ 登录页验证码轮询生命周期（BUG-0104） > 挂载时拉取一次验证码，此后每 55 秒轮询刷新 494ms
 ✓ tests/unit/views/iot/instance-item-drawer-ref.spec.ts (1 test) 759ms
   ✓ 实例数据项抽屉父组件 ref 暴露（BUG-0008） > 父组件 ref 可拿到 showDrawer/onClose；调用后抽屉打开且标题可见 757ms
 ✓ tests/unit/layout/module-switcher.spec.ts (6 tests) 389ms
 ✓ tests/unit/display/factory-card-summary.spec.ts (5 tests) 225ms
 ✓ tests/unit/display/display-slot-card.spec.ts (2 tests) 153ms
 ✓ tests/unit/composables/survey/use-project-creator.spec.ts (5 tests) 9ms

 Test Files  84 passed (84)
      Tests  714 passed (714)
   Start at  23:10:34
   Duration  95.13s (transform 35.15s, setup 0ms, collect 688.43s, tests 127.59s, environment 241.86s, prepare 42.36s)```

### 2.5 SQL 脚本内容自检与幂等性论证（brief 附加验收项）

脚本内容自检（在 `/d/workspace/WisFactory` 下以 grep 核对）：

```text
=== active (non-comment) SQL lines ===
UPDATE `sys_module` SET `status` = 0 WHERE `module_code` = 'survey' AND `deleted` = 0;
UPDATE `sys_menu` SET `visible` = 0 WHERE `module_id` = 3 AND `deleted` = 0;

=== DROP/DELETE check (whole file, case-insensitive) ===
7:--        无 DROP / DELETE，不触碰 sys_role_menu 与任何 survey_* 表，可完整回滚。

=== UPDATE count (active) ===
2

=== sys_module schema column evidence ===
resources/sql\01_base.sql:137:CREATE TABLE `sys_module`  (
resources/sql\01_base.sql:139:  `module_code` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '模块编码',
resources/sql\01_base.sql:151:  INDEX `idx_module_code`(`module_code`) USING BTREE
resources/sql\upgrade\20261006_sys_module_init.sql:15:CREATE TABLE IF NOT EXISTS `sys_module` (
resources/sql\upgrade\20261006_sys_module_init.sql:17:  `module_code` varchar(50) NOT NULL COMMENT '模块编码',
resources/sql\upgrade\20261006_sys_module_init.sql:29:  KEY `idx_module_code` (`module_code`)
resources/sql\upgrade\20261125_disable_survey.sql:13:UPDATE `sys_module` SET `status` = 0 WHERE `module_code` = 'survey' AND `deleted` = 0;
resources/sql\upgrade\20261125_disable_survey.sql:21:-- UPDATE `sys_module` SET `status` = 1 WHERE `module_code` = 'survey' AND `deleted` = 0;
```

`sys_module` DDL 列确认（`01_base.sql:137-152`、`20261006_sys_module_init.sql:15-30`）：存在 `module_code`、`status`（`0禁用 1启用`）、`deleted`（`0`），无 `code` 列；`sys_menu` 存在 `module_id`（`01_base.sql:163`）与 `visible`（`0隐藏 1显示`，`01_base.sql:172`）。执行者已据此在脚本中使用 `module_code`，否则会报 `Unknown column`。

幂等性静态论证：两条语句均为「把目标列设为固定值」的幂等 UPDATE，而非自增/翻转。第 1 次执行把 survey 模块 `status` 置 0、module_id=3 的未删除菜单 `visible` 置 0；第 2 次及以后再执行，受影响行已是 0，SET 0 仍写成 0，结果状态完全相同（受影响行数可能为 0，但数据终态一致）。语句带 `AND deleted = 0`，已软删除行不被触及，重复执行同样不改变结果。因此「两次执行结果一致」成立。脚本不含 DROP/DELETE，不触碰 `sys_role_menu` 与任何 `survey_*` 表，反向回滚注释把两列分别改回 1，数据与授权完整保留，可完全复原。按 brief 歧义裁决 5，以本静态论证替代 prod 档真实启动/连库验证。

四条前端命令退出码汇总：`build:test=0`、`lint=0`（0 error / 14 条既有 prettier warning）、`typecheck=0`、`test=0`（84 个测试文件、714 条用例全部通过）。四门一次通过，未发生失败，因此没有根因修复；未修改任何测试期望或产物以凑输出。

## 3 未做项与原因

- 未真实连库执行 SQL：按 brief 歧义裁决 2 与执行口径，幂等性给出静态论证即可，不要求真实连库执行。
- 未做 prod 档真实启动验证：按 brief 歧义裁决 5，本任务 SQL 仅两条 UPDATE、不改表结构、不会导致应用启动失败，以 SQL 自检 + 幂等性论证替代。
- 未物理删除 survey 代码文件（views/api/components/store/assets/composables/plugin/types/utils/survey）：按 brief 歧义裁决 1，避免与只读上游分叉，后端服务仍存活。
- 未改后端（module/survey、wechat handler、SysJobServiceImpl、i18n）、未改历史 SQL 脚本（02_survey.sql 与 5 个 upgrade survey 脚本）、未动 sys_role_menu 与 16 张 survey_* 表及数据：按 brief 歧义裁决 2/3/4，保留完整回滚路径。
- 未改索引 `tasks.md`、未跑 `close-task.sh`、未把 TASK 状态改为「已完成」：按本次执行约束，这些由协调者收尾；状态保持「进行中」。
- 未修 lint 的 14 条 prettier warning：均位于本批改动范围外的既有文件（iot 视图、survey 相关单测等），0 error 已满足验收口径；修改会越界，故保留。

## 4 偏离计划之处

- brief 写「移除 /s/:key public 白名单项」：代码中不存在独立的 public 白名单常量，匿名放行由路由的 `meta.public` 通用守卫机制驱动。survey 路由摘除（routers.ts 不再展开 surveyRouters、index.ts 不再 push 四组静态路由）后 `/s/:key` 已自动失去 `meta.public` 放行；`router-guard.spec.ts` 已新增「`/s/:key` 无 token 直链跳登录」断言实测验证。故未去改一个不存在的「白名单常量」，以等价且有效的方式达成目标。
- brief/TASK 正文 SQL 条件写 `WHERE code='survey'`：实际 schema 中 `sys_module` 无 `code` 列，编码列名为 `module_code`（`01_base.sql:139`、`20261006_sys_module_init.sql:17` 实证）。脚本改用 `module_code`，否则执行会报 `Unknown column`。这是对原文笔误的必要修正，非越界。
- 报告中命令输出为去除 ANSI 颜色码后的逐字文本（文字内容未删改），完整含颜色码原始日志保留在本地临时目录，便于需要时逐字节复核。
- 除上述两点外，实施与 brief 计划一致：仅改范围内的 5 个文件，四门验收一次全绿，无额外修复。

## 5 遗留问题

- 数据层脚本未在真实库执行、未查询验证 `status=0` / `visible=0` 的实际行数；上线时需由执行方按脚本执行并查询确认，本任务按裁决以静态论证替代。
- survey 前端源码文件与后端代码、16 张表及存量数据、sys_role_menu 授权均保留；仅摘除前端路由入口，仓库体积与 survey 代码的物理存在未减少，未来 subtree pull 仍需关注该部分冲突面。
- 前端仍存在 survey 相关单测（如 use-edit-lock、serial-number 等）与 iot 视图的既有 prettier warning，均不在本批范围，未处理。
- `build:test` 输出中的既有告警（`.env` 中 NODE_ENV=production 提示、vue3-json-viewer 的 `icon.svg` 运行时解析、部分 chunk >500 kB、esbuild 常量赋值警告）为基线固有，非本批引入，未修。
- 回滚操作需人工执行反向 SQL 并恢复 `routers.ts` / `index.ts` 中 survey 路由挂载；本次未编写自动化回滚脚本（brief 未要求）。
