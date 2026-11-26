# TASK-0095 实施报告：指标管理页与 metric-api 前端补齐

BASE：449d7eb3。验收命令：`cd D:\workspace\WisFactory\frontend && pnpm build:test && pnpm lint && pnpm typecheck && pnpm test`（链式一次跑完，整体 exit 0）。

## 1 改动文件清单

全部改动位于 brief 声明的前端范围（7 个文件）：

| 文件 | 状态 | 说明 |
| --- | --- | --- |
| `frontend/src/api/iot/metric-api.ts` | 新建 | `metricApi` 覆盖 7 端点（list/detail/add/generate/update/changeStatus/remove） |
| `frontend/src/views/iot/metric/index.vue` | 改写 | 替换 IotPlaceholder：列表分页+过滤/详情/新增/编辑/启停/删除/批量生成，`v-privilege` 五权限码 |
| `frontend/src/views/iot/metric/components/metric-form-modal.vue` | 新建 | 指标表单，校验对齐 DTO 注解，`defineExpose({ showModal, validateRatio })` |
| `frontend/src/views/iot/metric/components/metric-detail-modal.vue` | 新建 | 只读详情弹窗（brief 允许的 detail modal），空态诚实 |
| `frontend/tests/unit/api/iot/metric-api.spec.ts` | 新建 | 8 条端点对齐单测 |
| `frontend/tests/unit/views/iot/metric.spec.ts` | 新建 | 7 条页面/表单/启停/删除/批量生成单测，含父 ref `defineExpose` 回归 |
| `frontend/src/components/iot/iot-consts.ts` | 修改 | 追加指标枚举常量（brief 明确允许复用常量） |

未改 backend/calc/ingest/contract、点位页 / TASK-0097；工作副本原有未跟踪 `.agents/`、`.tmp-t0044v/` 未触碰。

## 2 每条验收命令的原始输出

以下为链式验收的完整原始输出（含 ANSI 色码，按命令分段；每段首行为该命令启动行）。基线：`pnpm test` 执行前为 82 文件 / 697 测试全绿；验收后 84 文件 / 712 测试全绿（+2 文件 / +15 测试，零回归）。

### `pnpm build:test`

```text
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
[36mvite v5.2.12 [32mbuilding for test...[36m[39m
transforming...

./icon.svg referenced in D:/workspace/WisFactory/frontend/node_modules/.pnpm/vue3-json-viewer@2.3.1_vue@3.4.27_typescript@5.6.3_/node_modules/vue3-json-viewer/dist/index.css didn't resolve at build time, it will remain unchanged to be resolved at runtime
[1m[33m[plugin:vite:esbuild][39m[22m [33m[plugin vite:esbuild] src/views/survey/components/FormAuthModal/panel.vue?vue&type=script&setup=true&lang.ts: [33mThis assignment will throw because "editors" is a constant[33m
244|                _createVNode(AuthMemberEditor, {
245|                  "member-ids": editors,
246|                  "onUpdate:memberIds": _cache[1] || (_cache[1] = ($event: any) => ((editors) = $event)),
   |                                                                                     ^
247|                  title: "编辑名单",
248|                  tip: "名单内成员可编辑该问卷（含查看）；支持按用户、角色、部门授权",
[39m
[1m[33m[plugin:vite:esbuild][39m[22m [33m[plugin vite:esbuild] src/views/survey/components/FormAuthModal/panel.vue?vue&type=script&setup=true&lang.ts: [33mThis assignment will throw because "viewers" is a constant[33m
254|                _createVNode(AuthMemberEditor, {
255|                  "member-ids": viewers,
256|                  "onUpdate:memberIds": _cache[2] || (_cache[2] = ($event: any) => ((viewers) = $event)),
   |                                                                                     ^
257|                  title: "查看名单",
258|                  tip: "名单内成员仅可查看该问卷（含数据与统计）；编辑名单成员已含查看权",
[39m
[1m[33m[plugin:vite:esbuild][39m[22m [33m[plugin vite:esbuild] src/views/business/oa/enterprise/components/enterprise-employee-list.vue?vue&type=script&setup=true&lang.ts: [33mThis assignment will throw because "columns" is a constant[33m
298|                class: "ot-margin-bottom5",
299|                modelValue: columns,
300|                "onUpdate:modelValue": _cache[1] || (_cache[1] = ($event: any) => ((columns) = $event)),
   |                                                                                    ^
301|                tableId: _unref(TABLE_ID_CONST).BUSINESS.OA.ENTERPRISE_EMPLOYEE,
302|                refresh: queryEmployee
[39m
[32m✓[39m 6020 modules transformed.
rendering chunks...
[33mwarnings when minifying css:
▲ [WARNING] Unexpected "2of" [css-syntax-error]

    <stdin>:1:99250:
      1 │ ...ntents}.tox .tox-ai :nth-child(2of.tox-ai__response-status-icon)...
        ╵                                   ~~~

[39m
[1m[33m[plugin:vite:reporter][39m[22m [33m[plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/src/api/system/login-api.ts is dynamically imported by D:/workspace/WisFactory/frontend/src/router/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/layout/components/change-password/regular-change-password-modal.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/layout/components/header-user-space/header-avatar.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/main.ts, D:/workspace/WisFactory/frontend/src/views/system/account/components/center/index.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/views/system/login3/login.vue?vue&type=script&setup=true&lang.ts, dynamic import will not move module into another chunk.
[39m
[1m[33m[plugin:vite:reporter][39m[22m [33m[plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/src/plugin/admin-ui/index.ts is dynamically imported by D:/workspace/WisFactory/frontend/src/router/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/main.ts, dynamic import will not move module into another chunk.
[39m
[1m[33m[plugin:vite:reporter][39m[22m [33m[plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/node_modules/.pnpm/ant-design-vue@4.2.5_vue@3.4.27_typescript@5.6.3_/node_modules/ant-design-vue/dist/reset.css is dynamically imported by D:/workspace/WisFactory/frontend/src/plugin/admin-ui/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/main.ts, dynamic import will not move module into another chunk.
[39m
computing gzip size...
[2mdist/[22m[32mindex.html                                                                                       [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.48 kB[22m
[2mdist/[22m[32mpng/ot-admin-logo-white-Dn8J4Vek.png                                                             [39m[1m[2m    8.11 kB[22m[1m[22m
[2mdist/[22m[32mpng/appearset_bgc_big-CBsi9EZh.png                                                               [39m[1m[2m    8.61 kB[22m[1m[22m
[2mdist/[22m[32mpng/ot-admin-logo-cDl7Ur4Y.png                                                                   [39m[1m[2m   14.33 kB[22m[1m[22m
[2mdist/[22m[32mpng/99d0-BLQr3eGW.png                                                                            [39m[1m[2m   37.33 kB[22m[1m[22m
[2mdist/[22m[32mpng/admin-logo-icon-BJtvcrOo.png                                                                 [39m[1m[2m   49.41 kB[22m[1m[22m
[2mdist/[22m[32mpng/login-illustration-9HORgnQJ.png                                                              [39m[1m[2m  235.70 kB[22m[1m[22m
[2mdist/[22m[32mpng/login-background-Dk6HcIGp.png                                                                [39m[1m[2m  760.38 kB[22m[1m[22m
[2mdist/[22m[35mcss/index-i0u-c3z8.css                                                                           [39m[1m[2m    0.04 kB[22m[1m[22m[2m │ gzip:   0.05 kB[22m
[2mdist/[22m[35mcss/DownloadFile-DAvaQD3e.css                                                                    [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-D6rOUVPI.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-CdH8jf3y.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/publish-log-drawer-s6nV0jMT.css                                                              [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-CnywH5a8.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-CTnqawDD.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/ProjectForm-KVnI2Kwr.css                                                                     [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/iot-placeholder-Bmzk0EMC.css                                                                 [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/metric-form-modal-D0QUa8s2.css                                                               [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/instance-item-drawer-DDYU_lFS.css                                                            [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/index-SkaSDKcc.css                                                                           [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/follow-DtSFEDAs.css                                                                          [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/index-rU0VK28O.css                                                                           [39m[1m[2m    0.07 kB[22m[1m[22m[2m │ gzip:   0.09 kB[22m
[2mdist/[22m[35mcss/index-Bv8IxLq4.css                                                                           [39m[1m[2m    0.07 kB[22m[1m[22m[2m │ gzip:   0.09 kB[22m
[2mdist/[22m[35mcss/ImgList-B6haYOiW.css                                                                         [39m[1m[2m    0.07 kB[22m[1m[22m[2m │ gzip:   0.09 kB[22m
[2mdist/[22m[35mcss/apiclient-secret-modal-Dd9zq1y6.css                                                          [39m[1m[2m    0.08 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/index-DHSD_xKM.css                                                                           [39m[1m[2m    0.08 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/index-B-eu0trW.css                                                                           [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/notice-detail-modal-CfRr7vB7.css                                                             [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/apiclient-form-modal-BLXq3tyN.css                                                            [39m[1m[2m    0.10 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/table-SRAIJii3.css                                                                           [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-BYAU4CBC.css                                                                           [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-C2VZgmO1.css                                                                           [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-BGXi8PCn.css                                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-YhouMItC.css                                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-kgmyyiGO.css                                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/notice-form-drawer-CXtGDo1N.css                                                              [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/publish-confirm-modal-CX1YV7gg.css                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-eeOT1m_L.css                                                [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/help-doc-form-drawer-BxBxRTJR.css                                                            [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-delete-DbyNiuDr.css                                         [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-DH5T8DG5.css                                                                           [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-CrZqPORT.css                                                                           [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/channel-detail-modal-CwLvL1Y3.css                                                            [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/import-CfDmKo2w.css                                                                          [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/BaseTable-B-tnrLUs.css                                                                       [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-Cpqpe2cV.css                                                                           [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-CM2MpC6G.css                                                                           [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-C-uHsV3_.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/index-DQWCfq80.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-Czq7odZB.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/pie-GiX0BmiW.css                                                                             [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/gauge-BlkLKyTy.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/keyword-match-modal-B7yCxDP5.css                                                             [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/category-BKKxPDYF.css                                                                        [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/gradient-ublseZuy.css                                                                        [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-basic-CmKoTgJy.css                                          [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/menu-operate-modal-CROAn4FB.css                                                              [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/module-form-BppZTgmd.css                                                                     [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-BCQNDPW4.css                                                                           [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/enterprise-employee-list-y_g8Pw_X.css                                                        [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/enterprise-operate-modal-BvDlyYS9.css                                                        [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-insert-and-update-C-D2A4nJ.css                              [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/index-BRnrwGM4.css                                                                           [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/help-doc-manage-list-Db1ISVYa.css                                                            [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/analysis-CQAzIk2d.css                                                                        [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-CnamIkRN.css                                                                           [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-V20uJDbO.css                                                                           [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-BQdngwWb.css                                                                           [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-Dp2Qu0a6.css                                                                           [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/TextListModal-lOMN9N1-.css                                                                   [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/goods-list-DOfkcpfl.css                                                                      [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/index-BKN7Vqi-.css                                                                           [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-BM5H80w0.css                                                                           [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/channel-form-modal-BtyQPGhT.css                                                              [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[35mcss/config-operate-drawer-CMQS0JWH.css                                                           [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/changelog-card-BIZYkilz.css                                                                  [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/template-send-detail-drawer-Iw0qJ7aB.css                                                     [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[35mcss/index-C0GpnHaV.css                                                                           [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/default-home-card-BOGSXst4.css                                                               [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-BnB06Whl.css                                                                           [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/preview-DVqL3VoK.css                                                                         [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/chart-CXuTlDAW.css                                                                           [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/role-tree-menu-Dnqv_GrS.css                                                                  [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/index-yMfMkHJV.css                                                                           [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-DQHmgcs_.css                                                                           [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/channel-credential-drawer-wEyPghzB.css                                                       [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/index-WxN6EUWJ.css                                                                           [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/index-Iei4Wp0G.css                                                                           [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-CCeTqU-9.css                                                                           [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[35mcss/snapshot-drawer-coJc1-CA.css                                                                 [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[35mcss/FormQrcode-zDibsgKU.css                                                                      [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-CEUAXvQU.css                                                                           [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[35mcss/help-doc-view-record-list-DXe94vIw.css                                                       [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/panel-CzgXg4MK.css                                                                           [39m[1m[2m    0.35 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/DataPush-BWg8P04L.css                                                                        [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/cross-DYHCe1z7.css                                                                           [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/help-doc-catalog-tree-5cONHUj1.css                                                           [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/notice-detail-CPlxUPVT.css                                                                   [39m[1m[2m    0.39 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/MatrixView-BQ8L2aDx.css                                                                      [39m[1m[2m    0.41 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/home-notice-B12SMd0n.css                                                                     [39m[1m[2m    0.41 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/index-BqHga3su.css                                                                           [39m[1m[2m    0.42 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-DCfFfRkA.css                                                                           [39m[1m[2m    0.42 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/dept-tree-checkbox-HxZiRqXx.css                                                              [39m[1m[2m    0.42 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/layout-const-CZJ-siCP.css                                                                    [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-kaqxmmba.css                                                                           [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[35mcss/TextCopy-DuZngPZh.css                                                                        [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.26 kB[22m
[2mdist/[22m[35mcss/DataFilter-CGvhUU-b.css                                                                      [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/template-send-create-modal-BpQLZrJw.css                                                      [39m[1m[2m    0.44 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/operate-log-detail-modal-D1pNfz_S.css                                                        [39m[1m[2m    0.46 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/member-editor-CINJ5hXC.css                                                                   [39m[1m[2m    0.52 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[35mcss/index-C7jUeuFy.css                                                                           [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[35mcss/index-CTahEBHb.css                                                                           [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[35mcss/home-to-be-done-DHoODtdh.css                                                                 [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/index-Cfspf8pS.css                                                                           [39m[1m[2m    0.62 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[35mcss/iot-formula-editor-Bujj7HnT.css                                                              [39m[1m[2m    0.63 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/ViewData-BpDSevMt.css                                                                        [39m[1m[2m    0.65 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[35mcss/FileList-CWWNXZQD.css                                                                        [39m[1m[2m    0.65 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[35mcss/index-BENQLd1l.css                                                                           [39m[1m[2m    0.70 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/notice-employee-detail-CkXuZHTQ.css                                                          [39m[1m[2m    0.71 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[35mcss/public-B-i-NxjN.css                                                                          [39m[1m[2m    0.74 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[35mcss/home-header-BMGP0g5m.css                                                                     [39m[1m[2m    0.75 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[35mcss/display-factory-card-CnkDoY9p.css                                                            [39m[1m[2m    0.76 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/display-slot-card-CEjwz2CP.css                                                               [39m[1m[2m    0.76 kB[22m[1m[22m[2m │ gzip:   0.33 kB[22m
[2mdist/[22m[35mcss/create-DvTvq8Xh.css                                                                          [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.36 kB[22m
[2mdist/[22m[35mcss/index-CVHl2D3B.css                                                                           [39m[1m[2m    0.79 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[35mcss/MatrixCard-BVqQUzzx.css                                                                      [39m[1m[2m    0.86 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-By8oNveA.css                                                                           [39m[1m[2m    0.89 kB[22m[1m[22m[2m │ gzip:   0.42 kB[22m
[2mdist/[22m[35mcss/index-D5AwHx5d.css                                                                           [39m[1m[2m    0.89 kB[22m[1m[22m[2m │ gzip:   0.37 kB[22m
[2mdist/[22m[35mcss/index-BhWXYNfe.css                                                                           [39m[1m[2m    0.94 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[35mcss/index-WcB63oQd.css                                                                           [39m[1m[2m    1.01 kB[22m[1m[22m[2m │ gzip:   0.39 kB[22m
[2mdist/[22m[35mcss/index-E3TlBhR5.css                                                                           [39m[1m[2m    1.05 kB[22m[1m[22m[2m │ gzip:   0.47 kB[22m
[2mdist/[22m[35mcss/home-quick-entry-hZEhId5q.css                                                                [39m[1m[2m    1.05 kB[22m[1m[22m[2m │ gzip:   0.37 kB[22m
[2mdist/[22m[35mcss/ExamResultCard-DJSfDte_.css                                                                  [39m[1m[2m    1.12 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[35mcss/index-BphPVy-R.css                                                                           [39m[1m[2m    1.21 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[35mcss/index-FmDNBQz6.css                                                                           [39m[1m[2m    1.22 kB[22m[1m[22m[2m │ gzip:   0.50 kB[22m
[2mdist/[22m[35mcss/redpack-VZ0AqG97.css                                                                         [39m[1m[2m    1.25 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[35mcss/role-tree-checkbox-mC0FzCf8.css                                                              [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.42 kB[22m
[2mdist/[22m[35mcss/write-HdA4QHHZ.css                                                                           [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.48 kB[22m
[2mdist/[22m[35mcss/ViewOrUpdate-7fM-g4hB.css                                                                    [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.50 kB[22m
[2mdist/[22m[35mcss/index-DMJnr6jw.css                                                                           [39m[1m[2m    1.34 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[35mcss/submit-CVT9IhCQ.css                                                                          [39m[1m[2m    1.41 kB[22m[1m[22m[2m │ gzip:   0.49 kB[22m
[2mdist/[22m[35mcss/help-doc-layout-Cq-8T7YQ.css                                                                 [39m[1m[2m    1.43 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/metric-card-BQXE-Rnr.css                                                                     [39m[1m[2m    1.44 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[35mcss/index-ul1VSKkd.css                                                                           [39m[1m[2m    1.65 kB[22m[1m[22m[2m │ gzip:   0.54 kB[22m
[2mdist/[22m[35mcss/code-generator-preview-modal-Datvi8o-.css                                                    [39m[1m[2m    1.65 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[35mcss/index-BZNtYzVB.css                                                                           [39m[1m[2m    1.72 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/login-Bvf6hYTl.css                                                                           [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[35mcss/notify-B_8E5ukD.css                                                                          [39m[1m[2m    1.81 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[35mcss/card-CKfzlZHl.css                                                                            [39m[1m[2m    1.96 kB[22m[1m[22m[2m │ gzip:   0.60 kB[22m
[2mdist/[22m[35mcss/notice-form-visible-transfer-department-Bmq10wNm.css                                         [39m[1m[2m    2.07 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/notice-form-visible-transfer-employee-BI-wxgA3.css                                           [39m[1m[2m    2.09 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/index-BzWu19bk.css                                                                           [39m[1m[2m    2.94 kB[22m[1m[22m[2m │ gzip:   0.88 kB[22m
[2mdist/[22m[35mcss/share-C55ToGJo.css                                                                           [39m[1m[2m    3.00 kB[22m[1m[22m[2m │ gzip:   0.84 kB[22m
[2mdist/[22m[35mcss/help-doc-user-view-BJvWW1qh.css                                                              [39m[1m[2m    3.79 kB[22m[1m[22m[2m │ gzip:   0.90 kB[22m
[2mdist/[22m[35mcss/index-CNnfYsqh.css                                                                           [39m[1m[2m    3.87 kB[22m[1m[22m[2m │ gzip:   0.83 kB[22m
[2mdist/[22m[35mcss/avatar-cropper-modal-9VFWDLF-.css                                                            [39m[1m[2m    4.05 kB[22m[1m[22m[2m │ gzip:   1.19 kB[22m
[2mdist/[22m[35mcss/index-a9zfQtlE.css                                                                           [39m[1m[2m    4.07 kB[22m[1m[22m[2m │ gzip:   0.86 kB[22m
[2mdist/[22m[35mcss/metric-trend-CJ1AF7NT.css                                                                    [39m[1m[2m    4.14 kB[22m[1m[22m[2m │ gzip:   0.83 kB[22m
[2mdist/[22m[35mcss/index-I1cZW9eG.css                                                                           [39m[1m[2m    4.38 kB[22m[1m[22m[2m │ gzip:   0.92 kB[22m
[2mdist/[22m[35mcss/index-9sHncdbb.css                                                                           [39m[1m[2m    4.40 kB[22m[1m[22m[2m │ gzip:   0.94 kB[22m
[2mdist/[22m[35mcss/index-CYd4SvCj.css                                                                           [39m[1m[2m    4.49 kB[22m[1m[22m[2m │ gzip:   1.26 kB[22m
[2mdist/[22m[35mcss/index-A-o_SUqN.css                                                                           [39m[1m[2m    4.53 kB[22m[1m[22m[2m │ gzip:   1.15 kB[22m
[2mdist/[22m[35mcss/enterprise-detail-CDC-O28W.css                                                               [39m[1m[2m   17.22 kB[22m[1m[22m[2m │ gzip:   2.61 kB[22m
[2mdist/[22m[35mcss/index-BA3svnqk.css                                                                           [39m[1m[2m   17.47 kB[22m[1m[22m[2m │ gzip:   3.26 kB[22m
[2mdist/[22m[35mcss/index-CL4e_1PC.css                                                                           [39m[1m[2m   22.01 kB[22m[1m[22m[2m │ gzip:   3.78 kB[22m
[2mdist/[22m[35mcss/index-BwsqDDIv.css                                                                           [39m[1m[2m   50.33 kB[22m[1m[22m[2m │ gzip:  10.36 kB[22m
[2mdist/[22m[35mcss/index-DNTQ0I4-.css                                                                           [39m[1m[2m  128.82 kB[22m[1m[22m[2m │ gzip:  19.08 kB[22m
[2mdist/[22m[35mcss/index-D6ofH3mU.css                                                                           [39m[1m[2m  172.81 kB[22m[1m[22m[2m │ gzip:  24.33 kB[22m
[2mdist/[22m[35mcss/style-BGjGDtq1.css                                                                           [39m[1m[2m  392.36 kB[22m[1m[22m[2m │ gzip:  69.45 kB[22m
[2mdist/[22m[36mjs/iot-optimistic-lock-CHOgIYgR.js                                                               [39m[1m[2m    0.04 kB[22m[1m[22m[2m │ gzip:   0.06 kB[22m
[2mdist/[22m[36mjs/403-CIquAXY1.js                                                                               [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[36mjs/regular-const-vBMlR054.js                                                                     [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[36mjs/_baseClamp-CdDz0ztW.js                                                                        [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[36mjs/join-Dox1eePW.js                                                                              [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[36mjs/date-util-D5CDYxFp.js                                                                         [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[36mjs/index-DZMsfMYC.js                                                                             [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[36mjs/auth-DoR4tlqU.js                                                                              [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/index-62Af5fm2.js                                                                             [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/index-BgT_G07D.js                                                                             [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/iframe-index-spxEnTif.js                                                                      [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/message-detail-C8IsKOIm.js                                                                    [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/dept-tree-item-CBel-1EL.js                                                                    [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/role-tree-point-DzgE9VKl.js                                                                   [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/change-log-modal-CdRvLF41.js                                                                  [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/to-be-done-modal-DG6KTsIT.js                                                                  [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/gateway-detail-modal-B5pqX7wM.js                                                              [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/device-component-zone-B2W5KH8e.js                                                             [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/index-DGDLpjbB.js                                                                             [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/index-DIga43Zy.js                                                                             [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/values-Df_hrfT-.js                                                                            [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/dict-form-modal-Bndr1nvx.js                                                                   [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/_baseMap-Xolt3JFf.js                                                                          [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/job-form-modal-B3D-q3Ot.js                                                                    [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/config-form-modal-CsbRv0BT.js                                                                 [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/menu-tree-select--kgq_2ma.js                                                                  [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal-K9VYe9rF.js                                                              [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/fan-search-select-jusle7A0.js                                                                 [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/device-trace-zone-BIPE7PFo.js                                                                 [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/reload-result-list-DIUTvZrb.js                                                                [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/keyword-form-modal-C9zNkPTe.js                                                                [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/notice-employee-list-DAbLpCWV.js                                                              [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-operate-modal-DoIn0aLr.js                                                     [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-operate-modal-D_rHLqUP.js                                                  [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/serial-number-record-list-B7bmQa1E.js                                                         [39m[1m[2m    0.19 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/job-form-modal-lU8fUV2F.js                                                                    [39m[1m[2m    0.19 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/item-BwPepPow.js                                                                              [39m[1m[2m    0.19 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/dict-form-modal-awHOoFba.js                                                                   [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-table-field-CLek2k5w.js                                      [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/position-form-Db71VQ6O.js                                                                     [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal-cD74p63C.js                                                              [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/message-receiver-modal-C8WtzHy8.js                                                            [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/do-reload-form-modal-C_RbjYog.js                                                              [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/category-form-modal-CcFV4GCc.js                                                               [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/point-detail-modal-BLpAcqpF.js                                                                [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/_isIterateeCall-B-E7XaXT.js                                                                   [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/metric-detail-modal-CvQu5r0B.js                                                               [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/device-status-zone-DGUoP_s6.js                                                                [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/device-realtime-zone-Bsy6IaQo.js                                                              [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/instance-detail-modal-DCMiWDNs.js                                                             [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/startsWith-B-gxTA6a.js                                                                        [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/useMessage-CsF3WWka.js                                                                        [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/mass-send-detail-drawer-D9DEnqo3.js                                                           [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/job-log-modal-C0Zjqg88.js                                                                     [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/file-api-Ckcapov8.js                                                                          [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/reload-api-C7zFoy66.js                                                                        [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/serial-number-api-DJddEHXF.js                                                                 [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/notice-form-modal-Cy9w0Yqz.js                                                                 [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/dict-import-modal-Deqao4IE.js                                                                 [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/device-curve-zone-BYdztJuj.js                                                                 [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/tenant-form-modal-CFkUvyBQ.js                                                                 [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/tenant-admin-modal-BEzvzWkd.js                                                                [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/use-wechat-disabled-BvA3I8lW.js                                                               [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/instance-params-modal-BGehZryI.js                                                             [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/mass-send-create-modal-DHPyX0Kq.js                                                            [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/gateway-api-BK7_lr_h.js                                                                       [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/item-point-bind-modal-DHF6p12Q.js                                                             [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/device-BV1wQC-Y.js                                                                            [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/source-DyYmhK0x.js                                                                            [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/serial-number-generate-form-modal-C6QYWZIo.js                                                 [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/auth-Dnqa2NR8.js                                                                              [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/index-BYThQIkI.js                                                                             [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/situation-CONvFQhQ.js                                                                         [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/category-api-CPWYZ3Lq.js                                                                      [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/PositionMap-DgupY72j.js                                                                       [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/index-C0SvTSn8.js                                                                             [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/goods-form-modal-BY9V3zQY.js                                                                  [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/index-nabHare6.js                                                                             [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/index-DnIckDJT.js                                                                             [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/change-log-form-oMT2bMzg.js                                                                   [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/dict-form-modal-B6mG8_aY.js                                                                   [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/dict-data-modal-DCVPEi1r.js                                                                   [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/point-form-modal-CCqc7JZn.js                                                                  [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/iframe-index.vue_vue_type_script_setup_true_lang-DHslptRA.js                                  [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/channel-form-modal-D53Qi3GN.js                                                                [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/menu-api-Digiom3u.js                                                                          [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/template-form-modal-BuCffyeO.js                                                               [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/str-util-DvlP_5eO.js                                                                          [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-tree-select-DnhzLMBQ.js                                                      [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/position-api-DagedT6S.js                                                                      [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/item-formula-rollup-modal-DN5QzNx7.js                                                         [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-query-field-CxogHTZ5.js                                      [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/channel-api-Wh5Yyb4R.js                                                                       [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/profile-api-D-XWoWnC.js                                                                       [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/notice-view-record-list-DQmS8Mov.js                                                           [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/file-api-DK-X36--.js                                                                          [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/category-tree-table-Z4vEfzeX.js                                                               [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/gateway-form-modal-B1mbAVtd.js                                                                [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-api-CCQJHMfr.js                                                              [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/change-log-api-D3Kw4JNH.js                                                                    [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/dept-api-Bbcrst2E.js                                                                          [39m[1m[2m    0.32 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/mitt-CNZ6avp8.js                                                                              [39m[1m[2m    0.32 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-modal-P0LO8lbb.js                                                         [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/dict-data-modal-BhiZM_fr.js                                                                   [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-field-C3TM_e8G.js                                            [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/message-send-form-zZhjAJMP.js                                                                 [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/instance-form-modal-XPSEFCW3.js                                                               [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/instance-api-OOg_n9Rt.js                                                                      [39m[1m[2m    0.36 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/module-api-BYG4dAD6.js                                                                        [39m[1m[2m    0.36 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/tag-filter-IviZpaC6.js                                                                        [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/useFormSetting-Y5tR6Rc2.js                                                                    [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/index-BxBa9Osh.js                                                                             [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/publish-api-ZtzHrNEN.js                                                                       [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/template-item-modal-1tV14rR5.js                                                               [39m[1m[2m    0.38 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/wechat-disabled-alert-BoY4Zo2r.js                                                             [39m[1m[2m    0.38 kB[22m[1m[22m[2m │ gzip:   0.41 kB[22m
[2mdist/[22m[36mjs/dept-form-modal-74wo_lpe.js                                                                   [39m[1m[2m    0.38 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/404-B78zalUT.js                                                                               [39m[1m[2m    0.39 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[36mjs/tenant-api-BM3FHKZs.js                                                                        [39m[1m[2m    0.41 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/table-id-const-BsJohLR_.js                                                                    [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/job-api-CGdMl3cT.js                                                                           [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-form-modal-hUOzegiJ.js                                                       [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/use-display-tree-DW04OPWY.js                                                                  [39m[1m[2m    0.44 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[36mjs/metric-api-4_ia9F46.js                                                                        [39m[1m[2m    0.45 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/_baseUniq-D1us2jLS.js                                                                         [39m[1m[2m    0.45 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[36mjs/job-api-CpyA05gy.js                                                                           [39m[1m[2m    0.46 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/config-list-z7wyiSvt.js                                                                       [39m[1m[2m    0.47 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/theme-api-Ca3TcEwc.js                                                                         [39m[1m[2m    0.48 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/code-generator-api-B-Vw34wS.js                                                                [39m[1m[2m    0.48 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[36mjs/iot-placeholder-DWmF75K9.js                                                                   [39m[1m[2m    0.49 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[36mjs/goods-catalog-1HDlUL9n.js                                                                     [39m[1m[2m    0.50 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[36mjs/job-log-list-modal-B9v0U1yZ.js                                                                [39m[1m[2m    0.50 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/user-api-QvtvM4iK.js                                                                          [39m[1m[2m    0.51 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/table-auto-height-DQG3vtLF.js                                                                 [39m[1m[2m    0.51 kB[22m[1m[22m[2m │ gzip:   0.34 kB[22m
[2mdist/[22m[36mjs/common-BzOvJVeF.js                                                                            [39m[1m[2m    0.51 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[36mjs/custom-catalog-3cfJjDd7.js                                                                    [39m[1m[2m    0.52 kB[22m[1m[22m[2m │ gzip:   0.33 kB[22m
[2mdist/[22m[36mjs/dict-api-DgjiWV2J.js                                                                          [39m[1m[2m    0.52 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[36mjs/auth.vue_vue_type_script_setup_true_lang-DhAf-JTg.js                                          [39m[1m[2m    0.53 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[36mjs/index-B1b4aZQj.js                                                                             [39m[1m[2m    0.55 kB[22m[1m[22m[2m │ gzip:   0.33 kB[22m
[2mdist/[22m[36mjs/dict-api-D5CwUs8J.js                                                                          [39m[1m[2m    0.55 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-list-DccV0Ubi.js                                                              [39m[1m[2m    0.56 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-list-B2rHTk4r.js                                                           [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/index-DYsTR_u2.js                                                                             [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-keW4TFto.js                                         [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.42 kB[22m
[2mdist/[22m[36mjs/deleted-job-list-BqtBvmg6.js                                                                  [39m[1m[2m    0.58 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[36mjs/help-doc-api-DHDGPb9S.js                                                                      [39m[1m[2m    0.58 kB[22m[1m[22m[2m │ gzip:   0.26 kB[22m
[2mdist/[22m[36mjs/ProjectForm-DEnL1GTE.js                                                                       [39m[1m[2m    0.58 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[36mjs/role-tree-point.vue_vue_type_script_setup_true_lang-DKrxTSAZ.js                               [39m[1m[2m    0.59 kB[22m[1m[22m[2m │ gzip:   0.39 kB[22m
[2mdist/[22m[36mjs/item-api-_MXLuxik.js                                                                          [39m[1m[2m    0.59 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[36mjs/send-common-DdiR-N0d.js                                                                       [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.37 kB[22m
[2mdist/[22m[36mjs/default-time-ranges-BFoVJ5IX.js                                                               [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.26 kB[22m
[2mdist/[22m[36mjs/FormQrcode-DHkT1QKS.js                                                                        [39m[1m[2m    0.61 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[36mjs/template-api-a26rljYV.js                                                                      [39m[1m[2m    0.62 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/data-api-7KGp7N00.js                                                                          [39m[1m[2m    0.62 kB[22m[1m[22m[2m │ gzip:   0.36 kB[22m
[2mdist/[22m[36mjs/report-api-DarwTKVJ.js                                                                        [39m[1m[2m    0.65 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/enterprise-api-DlzIa0EY.js                                                                    [39m[1m[2m    0.67 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/template-api-D2Zm6sNC.js                                                                      [39m[1m[2m    0.68 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/FileList-CbtxCHZN.js                                                                          [39m[1m[2m    0.70 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[36mjs/notice-api-DXpRNoxn.js                                                                        [39m[1m[2m    0.71 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/index-gHbFOn2G.js                                                                             [39m[1m[2m    0.73 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[36mjs/change-log-modal.vue_vue_type_script_setup_true_lang-CT5ZjsiE.js                              [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.51 kB[22m
[2mdist/[22m[36mjs/report-format-vflwHJHL.js                                                                     [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.48 kB[22m
[2mdist/[22m[36mjs/ImgList-bo5S1xHU.js                                                                           [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/index-D33bfjU2.js                                                                             [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.34 kB[22m
[2mdist/[22m[36mjs/point-api-DB2RXDvJ.js                                                                         [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.39 kB[22m
[2mdist/[22m[36mjs/file-url-D4vjUgCV.js                                                                          [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.47 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-CNR-TvGm.js                                         [39m[1m[2m    0.78 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/help-doc-list-CNmoOBkh.js                                                                     [39m[1m[2m    0.83 kB[22m[1m[22m[2m │ gzip:   0.40 kB[22m
[2mdist/[22m[36mjs/_charsStartIndex-h3CzRq-W.js                                                                  [39m[1m[2m    0.84 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-D5MwPAe6.js                                         [39m[1m[2m    0.84 kB[22m[1m[22m[2m │ gzip:   0.49 kB[22m
[2mdist/[22m[36mjs/index-TOIz0FMv.js                                                                             [39m[1m[2m    0.84 kB[22m[1m[22m[2m │ gzip:   0.55 kB[22m
[2mdist/[22m[36mjs/role-tree-checkbox-BIvBKzRN.js                                                                [39m[1m[2m    0.85 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/project-list-columns-BqIdc9YB.js                                                              [39m[1m[2m    0.87 kB[22m[1m[22m[2m │ gzip:   0.38 kB[22m
[2mdist/[22m[36mjs/MinusOutlined-D4csvgKm.js                                                                     [39m[1m[2m    0.88 kB[22m[1m[22m[2m │ gzip:   0.55 kB[22m
[2mdist/[22m[36mjs/CaretDownOutlined-BcFoHlNA.js                                                                 [39m[1m[2m    0.91 kB[22m[1m[22m[2m │ gzip:   0.57 kB[22m
[2mdist/[22m[36mjs/role-api-DTCRo1vi.js                                                                          [39m[1m[2m    0.91 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[36mjs/CaretRightOutlined-DNj9NE0W.js                                                                [39m[1m[2m    0.91 kB[22m[1m[22m[2m │ gzip:   0.58 kB[22m
[2mdist/[22m[36mjs/setting-api-DWFiML6k.js                                                                       [39m[1m[2m    0.92 kB[22m[1m[22m[2m │ gzip:   0.47 kB[22m
[2mdist/[22m[36mjs/index-DRGT06U1.js                                                                             [39m[1m[2m    0.92 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[36mjs/form-api-B1QndBlF.js                                                                          [39m[1m[2m    0.94 kB[22m[1m[22m[2m │ gzip:   0.38 kB[22m
[2mdist/[22m[36mjs/menu-tree-select.vue_vue_type_script_setup_true_lang-CWVe6PvN.js                              [39m[1m[2m    0.95 kB[22m[1m[22m[2m │ gzip:   0.60 kB[22m
[2mdist/[22m[36mjs/dept-tree-item.vue_vue_type_script_setup_true_lang-lh4oYt5y.js                                [39m[1m[2m    0.96 kB[22m[1m[22m[2m │ gzip:   0.57 kB[22m
[2mdist/[22m[36mjs/MobileOutlined-B3ECD1aD.js                                                                    [39m[1m[2m    0.96 kB[22m[1m[22m[2m │ gzip:   0.60 kB[22m
[2mdist/[22m[36mjs/LeftOutlined-cg_Q-7J5.js                                                                      [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/RightOutlined-BG3ekzp_.js                                                                     [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/DownOutlined-Cd9Z80Td.js                                                                      [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/DownloadFile-ChBMxAQN.js                                                                      [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.71 kB[22m
[2mdist/[22m[36mjs/pie-DndFHOM_.js                                                                               [39m[1m[2m    0.98 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/ArrowLeftOutlined-DQm-pSBK.js                                                                 [39m[1m[2m    1.01 kB[22m[1m[22m[2m │ gzip:   0.65 kB[22m
[2mdist/[22m[36mjs/TextCopy-COFESU_P.js                                                                          [39m[1m[2m    1.04 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/device.vue_vue_type_script_setup_true_lang-CtB2Pvwd.js                                        [39m[1m[2m    1.07 kB[22m[1m[22m[2m │ gzip:   0.69 kB[22m
[2mdist/[22m[36mjs/UploadOutlined-BjVPvZ65.js                                                                    [39m[1m[2m    1.09 kB[22m[1m[22m[2m │ gzip:   0.67 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-yA461421.js                                         [39m[1m[2m    1.10 kB[22m[1m[22m[2m │ gzip:   0.65 kB[22m
[2mdist/[22m[36mjs/sortBy-DZuiYnuM.js                                                                            [39m[1m[2m    1.10 kB[22m[1m[22m[2m │ gzip:   0.60 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-Cd8lALw7.js                                         [39m[1m[2m    1.10 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/redpack-helper-BxpknW9c.js                                                                    [39m[1m[2m    1.11 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/default-home-card-ByRRo3ft.js                                                                 [39m[1m[2m    1.12 kB[22m[1m[22m[2m │ gzip:   0.63 kB[22m
[2mdist/[22m[36mjs/menu-data-handler-DNsjJbOZ.js                                                                 [39m[1m[2m    1.13 kB[22m[1m[22m[2m │ gzip:   0.52 kB[22m
[2mdist/[22m[36mjs/display-slot-card-DEzQ-v0C.js                                                                 [39m[1m[2m    1.15 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/CheckSquareOutlined-FcBrtFBK.js                                                               [39m[1m[2m    1.15 kB[22m[1m[22m[2m │ gzip:   0.71 kB[22m
[2mdist/[22m[36mjs/CopyOutlined-BNe_xAAr.js                                                                      [39m[1m[2m    1.16 kB[22m[1m[22m[2m │ gzip:   0.72 kB[22m
[2mdist/[22m[36mjs/to-be-done-modal.vue_vue_type_script_setup_true_lang-Tj-4RUHo.js                              [39m[1m[2m    1.17 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/gauge-CiMKXKXY.js                                                                             [39m[1m[2m    1.18 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/preview-3GGuSATX.js                                                                           [39m[1m[2m    1.18 kB[22m[1m[22m[2m │ gzip:   0.74 kB[22m
[2mdist/[22m[36mjs/EditOutlined-Ck80vrdP.js                                                                      [39m[1m[2m    1.19 kB[22m[1m[22m[2m │ gzip:   0.74 kB[22m
[2mdist/[22m[36mjs/PictureOutlined-CrH9uiIa.js                                                                   [39m[1m[2m    1.19 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/BellOutlined-2WQTpGd5.js                                                                      [39m[1m[2m    1.22 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/dept-tree-checkbox-Ca-s0fwi.js                                                                [39m[1m[2m    1.23 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/category-DqLxOZTA.js                                                                          [39m[1m[2m    1.23 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/ApartmentOutlined-BN6iLpzT.js                                                                 [39m[1m[2m    1.24 kB[22m[1m[22m[2m │ gzip:   0.68 kB[22m
[2mdist/[22m[36mjs/message-detail.vue_vue_type_script_setup_true_lang-DzdioG1j.js                                [39m[1m[2m    1.24 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/fan-search-select.vue_vue_type_script_setup_true_lang-BR5A8vse.js                             [39m[1m[2m    1.25 kB[22m[1m[22m[2m │ gzip:   0.81 kB[22m
[2mdist/[22m[36mjs/BarChartOutlined-DITzSr2X.js                                                                  [39m[1m[2m    1.26 kB[22m[1m[22m[2m │ gzip:   0.62 kB[22m
[2mdist/[22m[36mjs/index-Sd1ZsqSc.js                                                                             [39m[1m[2m    1.27 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-tree-select.vue_vue_type_script_setup_true_lang-I0BsGOGR.js                  [39m[1m[2m    1.27 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-B6HfvSNz.js                                         [39m[1m[2m    1.28 kB[22m[1m[22m[2m │ gzip:   0.72 kB[22m
[2mdist/[22m[36mjs/situation.vue_vue_type_script_setup_true_lang-Cai5iJTu.js                                     [39m[1m[2m    1.28 kB[22m[1m[22m[2m │ gzip:   0.82 kB[22m
[2mdist/[22m[36mjs/notice-detail-modal-9izbQaQH.js                                                               [39m[1m[2m    1.29 kB[22m[1m[22m[2m │ gzip:   0.72 kB[22m
[2mdist/[22m[36mjs/index-kcREc2JQ.js                                                                             [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/AppstoreOutlined-BhPTXR_N.js                                                                  [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/index-COZPXTA0.js                                                                             [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.85 kB[22m
[2mdist/[22m[36mjs/home-notice-Bda4QlgS.js                                                                       [39m[1m[2m    1.31 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/mass-send-helper-B3eNN9ft.js                                                                  [39m[1m[2m    1.31 kB[22m[1m[22m[2m │ gzip:   0.83 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-modal.vue_vue_type_script_setup_true_lang-UT-2qbQo.js                     [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/InboxOutlined-Df1vJj04.js                                                                     [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.82 kB[22m
[2mdist/[22m[36mjs/useProject-BxJJ2eAk.js                                                                        [39m[1m[2m    1.33 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-D--PmZ0p.js                                         [39m[1m[2m    1.34 kB[22m[1m[22m[2m │ gzip:   0.78 kB[22m
[2mdist/[22m[36mjs/changelog-card-dMcJBxjr.js                                                                    [39m[1m[2m    1.36 kB[22m[1m[22m[2m │ gzip:   0.92 kB[22m
[2mdist/[22m[36mjs/source.vue_vue_type_script_setup_true_lang-HX4i70uR.js                                        [39m[1m[2m    1.38 kB[22m[1m[22m[2m │ gzip:   0.84 kB[22m
[2mdist/[22m[36mjs/wechat-api-CszkSTy8.js                                                                        [39m[1m[2m    1.40 kB[22m[1m[22m[2m │ gzip:   0.57 kB[22m
[2mdist/[22m[36mjs/item.vue_vue_type_script_setup_true_lang-Bs3nVlMw.js                                          [39m[1m[2m    1.40 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-CbxjQa-s.js                                         [39m[1m[2m    1.43 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/ReloadOutlined-8irah91A.js                                                                    [39m[1m[2m    1.48 kB[22m[1m[22m[2m │ gzip:   0.87 kB[22m
[2mdist/[22m[36mjs/reload-result-list.vue_vue_type_script_setup_true_lang-Dx8fUzCm.js                            [39m[1m[2m    1.50 kB[22m[1m[22m[2m │ gzip:   0.94 kB[22m
[2mdist/[22m[36mjs/SyncOutlined-CoMeIAMr.js                                                                      [39m[1m[2m    1.51 kB[22m[1m[22m[2m │ gzip:   0.91 kB[22m
[2mdist/[22m[36mjs/home-quick-entry-modal-DGmtV4HY.js                                                            [39m[1m[2m    1.51 kB[22m[1m[22m[2m │ gzip:   0.65 kB[22m
[2mdist/[22m[36mjs/apiclient-secret-modal-3Wn3KUcq.js                                                            [39m[1m[2m    1.53 kB[22m[1m[22m[2m │ gzip:   1.03 kB[22m
[2mdist/[22m[36mjs/iot-consts-BURryymr.js                                                                        [39m[1m[2m    1.54 kB[22m[1m[22m[2m │ gzip:   1.03 kB[22m
[2mdist/[22m[36mjs/MatrixView-CPpOD3RU.js                                                                        [39m[1m[2m    1.54 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/index-Asp80zge.js                                                                             [39m[1m[2m    1.55 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/FullscreenOutlined-BIWbta5j.js                                                                [39m[1m[2m    1.59 kB[22m[1m[22m[2m │ gzip:   0.86 kB[22m
[2mdist/[22m[36mjs/do-reload-form-modal.vue_vue_type_script_setup_true_lang-Bccs0gr0.js                          [39m[1m[2m    1.60 kB[22m[1m[22m[2m │ gzip:   0.94 kB[22m
[2mdist/[22m[36mjs/index-Cvw-AAW4.js                                                                             [39m[1m[2m    1.62 kB[22m[1m[22m[2m │ gzip:   0.86 kB[22m
[2mdist/[22m[36mjs/UrlOperate-LWmDTRIL.js                                                                        [39m[1m[2m    1.63 kB[22m[1m[22m[2m │ gzip:   0.89 kB[22m
[2mdist/[22m[36mjs/CloudUploadOutlined-_I3PSSc-.js                                                               [39m[1m[2m    1.64 kB[22m[1m[22m[2m │ gzip:   0.98 kB[22m
[2mdist/[22m[36mjs/UserAddOutlined-BpNDKV_8.js                                                                   [39m[1m[2m    1.66 kB[22m[1m[22m[2m │ gzip:   0.95 kB[22m
[2mdist/[22m[36mjs/index-DEStoSCJ.js                                                                             [39m[1m[2m    1.69 kB[22m[1m[22m[2m │ gzip:   0.92 kB[22m
[2mdist/[22m[36mjs/metric-utils-CS9quiXT.js                                                                      [39m[1m[2m    1.69 kB[22m[1m[22m[2m │ gzip:   1.14 kB[22m
[2mdist/[22m[36mjs/help-doc-manage-list-DhFWsfs4.js                                                              [39m[1m[2m    1.71 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/member-editor-DDfV8ksy.js                                                                     [39m[1m[2m    1.71 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/index-Dv9IrfMp.js                                                                             [39m[1m[2m    1.72 kB[22m[1m[22m[2m │ gzip:   0.81 kB[22m
[2mdist/[22m[36mjs/category-form-modal.vue_vue_type_script_setup_true_lang-BOtkRdPE.js                           [39m[1m[2m    1.73 kB[22m[1m[22m[2m │ gzip:   1.07 kB[22m
[2mdist/[22m[36mjs/QrcodeOutlined-CnPiof_j.js                                                                    [39m[1m[2m    1.77 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/ExamResultCard-D7WfAJed.js                                                                    [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   1.04 kB[22m
[2mdist/[22m[36mjs/index-CbkOV7Im.js                                                                             [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   0.97 kB[22m
[2mdist/[22m[36mjs/home-quick-entry-modal.vue_vue_type_script_setup_true_lang-CWf91QAf.js                        [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   1.03 kB[22m
[2mdist/[22m[36mjs/index-Dv62k8tM.js                                                                             [39m[1m[2m    1.79 kB[22m[1m[22m[2m │ gzip:   1.21 kB[22m
[2mdist/[22m[36mjs/dict-form-modal.vue_vue_type_script_setup_true_lang-49yCXVCT.js                               [39m[1m[2m    1.79 kB[22m[1m[22m[2m │ gzip:   1.05 kB[22m
[2mdist/[22m[36mjs/channel-detail-modal-DR4TCs_F.js                                                              [39m[1m[2m    1.81 kB[22m[1m[22m[2m │ gzip:   0.95 kB[22m
[2mdist/[22m[36mjs/index-Br9yp3mU.js                                                                             [39m[1m[2m    1.82 kB[22m[1m[22m[2m │ gzip:   1.01 kB[22m
[2mdist/[22m[36mjs/module-list-x3ytu4-s.js                                                                       [39m[1m[2m    1.82 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-transfer-department-B5RF0U5A.js                                           [39m[1m[2m    1.89 kB[22m[1m[22m[2m │ gzip:   1.02 kB[22m
[2mdist/[22m[36mjs/helper-P-Oa7HAp.js                                                                            [39m[1m[2m    1.89 kB[22m[1m[22m[2m │ gzip:   1.06 kB[22m
[2mdist/[22m[36mjs/data-masking-list-C25KKvfr.js                                                                 [39m[1m[2m    1.90 kB[22m[1m[22m[2m │ gzip:   1.32 kB[22m
[2mdist/[22m[36mjs/template-send-helper-DaDutCdo.js                                                              [39m[1m[2m    1.90 kB[22m[1m[22m[2m │ gzip:   1.06 kB[22m
[2mdist/[22m[36mjs/gateway-detail-modal.vue_vue_type_script_setup_true_lang-KtKU42Jx.js                          [39m[1m[2m    1.92 kB[22m[1m[22m[2m │ gzip:   0.82 kB[22m
[2mdist/[22m[36mjs/serial-number-record-list.vue_vue_type_script_setup_true_lang-CRx3vh-A.js                     [39m[1m[2m    1.92 kB[22m[1m[22m[2m │ gzip:   1.12 kB[22m
[2mdist/[22m[36mjs/display-factory-card-BLR008_5.js                                                              [39m[1m[2m    1.92 kB[22m[1m[22m[2m │ gzip:   0.98 kB[22m
[2mdist/[22m[36mjs/index-BBTwlcbL.js                                                                             [39m[1m[2m    1.99 kB[22m[1m[22m[2m │ gzip:   1.10 kB[22m
[2mdist/[22m[36mjs/BaseTable-DbjHap-K.js                                                                         [39m[1m[2m    2.04 kB[22m[1m[22m[2m │ gzip:   1.07 kB[22m
[2mdist/[22m[36mjs/metric-card-fT5IC_Mx.js                                                                       [39m[1m[2m    2.04 kB[22m[1m[22m[2m │ gzip:   1.02 kB[22m
[2mdist/[22m[36mjs/cache-list-UnFZAPyc.js                                                                        [39m[1m[2m    2.05 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/PositionMap.vue_vue_type_script_setup_true_lang-DKatiEN1.js                                   [39m[1m[2m    2.05 kB[22m[1m[22m[2m │ gzip:   1.27 kB[22m
[2mdist/[22m[36mjs/menu-list-DBJOVx5G.js                                                                         [39m[1m[2m    2.06 kB[22m[1m[22m[2m │ gzip:   0.82 kB[22m
[2mdist/[22m[36mjs/item-point-bind-modal.vue_vue_type_script_setup_true_lang-RetQ7ZWJ.js                         [39m[1m[2m    2.07 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/dict-form-modal.vue_vue_type_script_setup_true_lang-B2M5O_S3.js                               [39m[1m[2m    2.07 kB[22m[1m[22m[2m │ gzip:   1.14 kB[22m
[2mdist/[22m[36mjs/index-XqGRoj2z.js                                                                             [39m[1m[2m    2.09 kB[22m[1m[22m[2m │ gzip:   1.22 kB[22m
[2mdist/[22m[36mjs/index-R2zfsfw0.js                                                                             [39m[1m[2m    2.12 kB[22m[1m[22m[2m │ gzip:   1.60 kB[22m
[2mdist/[22m[36mjs/account-menu-BGel4B9I.js                                                                      [39m[1m[2m    2.14 kB[22m[1m[22m[2m │ gzip:   0.99 kB[22m
[2mdist/[22m[36mjs/instance-detail-modal.vue_vue_type_script_setup_true_lang-CFWG5QhF.js                         [39m[1m[2m    2.15 kB[22m[1m[22m[2m │ gzip:   0.99 kB[22m
[2mdist/[22m[36mjs/FilterOutlined-gPPR_OVx.js                                                                    [39m[1m[2m    2.17 kB[22m[1m[22m[2m │ gzip:   0.95 kB[22m
[2mdist/[22m[36mjs/serial-number-generate-form-modal.vue_vue_type_script_setup_true_lang-CtKcySp4.js             [39m[1m[2m    2.19 kB[22m[1m[22m[2m │ gzip:   1.10 kB[22m
[2mdist/[22m[36mjs/tenant-admin-modal.vue_vue_type_script_setup_true_lang-mFpnzD10.js                            [39m[1m[2m    2.19 kB[22m[1m[22m[2m │ gzip:   1.29 kB[22m
[2mdist/[22m[36mjs/notice-employee-detail-CMtx2FCm.js                                                            [39m[1m[2m    2.20 kB[22m[1m[22m[2m │ gzip:   1.24 kB[22m
[2mdist/[22m[36mjs/index-DyH4yhIf.js                                                                             [39m[1m[2m    2.20 kB[22m[1m[22m[2m │ gzip:   1.25 kB[22m
[2mdist/[22m[36mjs/metric-detail-modal.vue_vue_type_script_setup_true_lang-BZDPSX91.js                           [39m[1m[2m    2.21 kB[22m[1m[22m[2m │ gzip:   0.99 kB[22m
[2mdist/[22m[36mjs/help-doc-user-view-4PJD6O2K.js                                                                [39m[1m[2m    2.21 kB[22m[1m[22m[2m │ gzip:   1.19 kB[22m
[2mdist/[22m[36mjs/follow-22WdDz6R.js                                                                            [39m[1m[2m    2.26 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/index-m96ueJVm.js                                                                             [39m[1m[2m    2.28 kB[22m[1m[22m[2m │ gzip:   1.28 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-form-modal.vue_vue_type_script_setup_true_lang-DMbB6-3y.js                   [39m[1m[2m    2.28 kB[22m[1m[22m[2m │ gzip:   1.26 kB[22m
[2mdist/[22m[36mjs/TextListModal-C4xkFxKl.js                                                                     [39m[1m[2m    2.28 kB[22m[1m[22m[2m │ gzip:   1.25 kB[22m
[2mdist/[22m[36mjs/config-form-modal.vue_vue_type_script_setup_true_lang-B0n3srZ2.js                             [39m[1m[2m    2.29 kB[22m[1m[22m[2m │ gzip:   1.18 kB[22m
[2mdist/[22m[36mjs/import-DKeC0w-P.js                                                                            [39m[1m[2m    2.29 kB[22m[1m[22m[2m │ gzip:   1.56 kB[22m
[2mdist/[22m[36mjs/gradient-XCAz1VAs.js                                                                          [39m[1m[2m    2.32 kB[22m[1m[22m[2m │ gzip:   1.06 kB[22m
[2mdist/[22m[36mjs/chart-B47Ms7rn.js                                                                             [39m[1m[2m    2.37 kB[22m[1m[22m[2m │ gzip:   1.11 kB[22m
[2mdist/[22m[36mjs/device-component-zone.vue_vue_type_script_setup_true_lang-IRhlHsqx.js                         [39m[1m[2m    2.42 kB[22m[1m[22m[2m │ gzip:   1.33 kB[22m
[2mdist/[22m[36mjs/index-DexiA1EZ.js                                                                             [39m[1m[2m    2.45 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/position-form.vue_vue_type_script_setup_true_lang-B7TNFNQ4.js                                 [39m[1m[2m    2.48 kB[22m[1m[22m[2m │ gzip:   1.28 kB[22m
[2mdist/[22m[36mjs/device-realtime-zone.vue_vue_type_script_setup_true_lang-D6fXgFnV.js                          [39m[1m[2m    2.50 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/index-G6zQAfAI.js                                                                             [39m[1m[2m    2.51 kB[22m[1m[22m[2m │ gzip:   1.21 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-transfer-employee-DtvIU_EI.js                                             [39m[1m[2m    2.51 kB[22m[1m[22m[2m │ gzip:   1.31 kB[22m
[2mdist/[22m[36mjs/display-api-CJe5-2wI.js                                                                       [39m[1m[2m    2.54 kB[22m[1m[22m[2m │ gzip:   1.01 kB[22m
[2mdist/[22m[36mjs/keyword-match-modal-DaYTczv_.js                                                               [39m[1m[2m    2.54 kB[22m[1m[22m[2m │ gzip:   1.39 kB[22m
[2mdist/[22m[36mjs/index-DUuUzjtL.js                                                                             [39m[1m[2m    2.58 kB[22m[1m[22m[2m │ gzip:   1.11 kB[22m
[2mdist/[22m[36mjs/SettingOutlined-Dgu12ja6.js                                                                   [39m[1m[2m    2.60 kB[22m[1m[22m[2m │ gzip:   1.31 kB[22m
[2mdist/[22m[36mjs/index-Md8drGLI.js                                                                             [39m[1m[2m    2.63 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/index-CnRHbilN.js                                                                             [39m[1m[2m    2.63 kB[22m[1m[22m[2m │ gzip:   1.56 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-table-field.vue_vue_type_script_setup_true_lang-B8yKoaFo.js  [39m[1m[2m    2.63 kB[22m[1m[22m[2m │ gzip:   1.22 kB[22m
[2mdist/[22m[36mjs/index-D1gVZe8k.js                                                                             [39m[1m[2m    2.65 kB[22m[1m[22m[2m │ gzip:   1.27 kB[22m
[2mdist/[22m[36mjs/item-formula-rollup-modal.vue_vue_type_script_setup_true_lang-Dd7H-N9O.js                     [39m[1m[2m    2.66 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/device-status-zone.vue_vue_type_script_setup_true_lang-BkhFP19A.js                            [39m[1m[2m    2.67 kB[22m[1m[22m[2m │ gzip:   1.30 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal.vue_vue_type_script_setup_true_lang-ChTDNh0H.js                          [39m[1m[2m    2.69 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/message-receiver-modal.vue_vue_type_script_setup_true_lang-Bc-5_3gq.js                        [39m[1m[2m    2.70 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/reload-list-DqvSwvLJ.js                                                                       [39m[1m[2m    2.73 kB[22m[1m[22m[2m │ gzip:   1.78 kB[22m
[2mdist/[22m[36mjs/device-utils-6k4pcSn7.js                                                                      [39m[1m[2m    2.77 kB[22m[1m[22m[2m │ gzip:   1.48 kB[22m
[2mdist/[22m[36mjs/dept-form-modal.vue_vue_type_script_setup_true_lang-CmehDCNl.js                               [39m[1m[2m    2.77 kB[22m[1m[22m[2m │ gzip:   1.46 kB[22m
[2mdist/[22m[36mjs/category-tree-table.vue_vue_type_script_setup_true_lang-qXdU5Cz-.js                           [39m[1m[2m    2.79 kB[22m[1m[22m[2m │ gzip:   1.43 kB[22m
[2mdist/[22m[36mjs/job-form-modal.vue_vue_type_script_setup_true_lang-BjMU5Ke5.js                                [39m[1m[2m    2.79 kB[22m[1m[22m[2m │ gzip:   1.44 kB[22m
[2mdist/[22m[36mjs/metric-trend-CBL7iTfX.js                                                                      [39m[1m[2m    2.80 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-operate-modal.vue_vue_type_script_setup_true_lang-q66TQgCK.js                 [39m[1m[2m    2.85 kB[22m[1m[22m[2m │ gzip:   1.38 kB[22m
[2mdist/[22m[36mjs/MatrixCard-BfaIPVoa.js                                                                        [39m[1m[2m    2.85 kB[22m[1m[22m[2m │ gzip:   1.30 kB[22m
[2mdist/[22m[36mjs/notice-form-modal.vue_vue_type_script_setup_true_lang-D0u8s9bV.js                             [39m[1m[2m    2.85 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/help-doc-view-record-list-DHge8hIT.js                                                         [39m[1m[2m    2.88 kB[22m[1m[22m[2m │ gzip:   1.49 kB[22m
[2mdist/[22m[36mjs/message-send-form.vue_vue_type_script_setup_true_lang-DAgfg5ox.js                             [39m[1m[2m    2.88 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-delete-DDfdjGOZ.js                                           [39m[1m[2m    2.91 kB[22m[1m[22m[2m │ gzip:   1.47 kB[22m
[2mdist/[22m[36mjs/index-DCPmn5_i.js                                                                             [39m[1m[2m    2.92 kB[22m[1m[22m[2m │ gzip:   1.46 kB[22m
[2mdist/[22m[36mjs/index-BZFb9uRv.js                                                                             [39m[1m[2m    2.94 kB[22m[1m[22m[2m │ gzip:   1.65 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-operate-modal.vue_vue_type_script_setup_true_lang-3GtMive7.js              [39m[1m[2m    2.99 kB[22m[1m[22m[2m │ gzip:   1.40 kB[22m
[2mdist/[22m[36mjs/dict-import-modal.vue_vue_type_script_setup_true_lang-CPodNFq7.js                             [39m[1m[2m    3.00 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/change-log-form.vue_vue_type_script_setup_true_lang-0eHCDoJA.js                               [39m[1m[2m    3.03 kB[22m[1m[22m[2m │ gzip:   1.45 kB[22m
[2mdist/[22m[36mjs/serial-number-list-KZVkfhsW.js                                                                [39m[1m[2m    3.05 kB[22m[1m[22m[2m │ gzip:   1.90 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal.vue_vue_type_script_setup_true_lang-B3F13Udk.js                          [39m[1m[2m    3.06 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/home-to-be-done-FuMwFOBD.js                                                                   [39m[1m[2m    3.06 kB[22m[1m[22m[2m │ gzip:   1.32 kB[22m
[2mdist/[22m[36mjs/index-BklswOKz.js                                                                             [39m[1m[2m    3.07 kB[22m[1m[22m[2m │ gzip:   1.57 kB[22m
[2mdist/[22m[36mjs/index-C1wg5xFf.js                                                                             [39m[1m[2m    3.09 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/snapshot-drawer-DzwQ-FzI.js                                                                   [39m[1m[2m    3.11 kB[22m[1m[22m[2m │ gzip:   1.66 kB[22m
[2mdist/[22m[36mjs/TeamOutlined-DGWMX3Aa.js                                                                      [39m[1m[2m    3.14 kB[22m[1m[22m[2m │ gzip:   1.34 kB[22m
[2mdist/[22m[36mjs/index-BvGBJAoW.js                                                                             [39m[1m[2m    3.16 kB[22m[1m[22m[2m │ gzip:   1.60 kB[22m
[2mdist/[22m[36mjs/submit-lHVSbUBM.js                                                                            [39m[1m[2m    3.18 kB[22m[1m[22m[2m │ gzip:   1.69 kB[22m
[2mdist/[22m[36mjs/notice-view-record-list.vue_vue_type_script_setup_true_lang-B4EHJ-I_.js                       [39m[1m[2m    3.20 kB[22m[1m[22m[2m │ gzip:   1.55 kB[22m
[2mdist/[22m[36mjs/public-Ei70B28b.js                                                                            [39m[1m[2m    3.20 kB[22m[1m[22m[2m │ gzip:   1.58 kB[22m
[2mdist/[22m[36mjs/role-tree-menu-DSp_XXNN.js                                                                    [39m[1m[2m    3.21 kB[22m[1m[22m[2m │ gzip:   1.50 kB[22m
[2mdist/[22m[36mjs/login-CZgoDv4Y.js                                                                             [39m[1m[2m    3.23 kB[22m[1m[22m[2m │ gzip:   1.66 kB[22m
[2mdist/[22m[36mjs/feedback-list-Do9nJtmV.js                                                                     [39m[1m[2m    3.24 kB[22m[1m[22m[2m │ gzip:   1.61 kB[22m
[2mdist/[22m[36mjs/dict-form-modal.vue_vue_type_script_setup_true_lang-jXXK78zK.js                               [39m[1m[2m    3.30 kB[22m[1m[22m[2m │ gzip:   1.75 kB[22m
[2mdist/[22m[36mjs/formatTableColumn-CHsnCt-I.js                                                                 [39m[1m[2m    3.31 kB[22m[1m[22m[2m │ gzip:   1.16 kB[22m
[2mdist/[22m[36mjs/template-form-modal.vue_vue_type_script_setup_true_lang-CIGPMe70.js                           [39m[1m[2m    3.32 kB[22m[1m[22m[2m │ gzip:   1.75 kB[22m
[2mdist/[22m[36mjs/index-DtiCAMEw.js                                                                             [39m[1m[2m    3.37 kB[22m[1m[22m[2m │ gzip:   1.80 kB[22m
[2mdist/[22m[36mjs/code-generator-util-X5KUoHCl.js                                                               [39m[1m[2m    3.40 kB[22m[1m[22m[2m │ gzip:   0.90 kB[22m
[2mdist/[22m[36mjs/index-a-2kWJeW.js                                                                             [39m[1m[2m    3.44 kB[22m[1m[22m[2m │ gzip:   1.76 kB[22m
[2mdist/[22m[36mjs/help-doc-layout-BojgJ0zb.js                                                                   [39m[1m[2m    3.47 kB[22m[1m[22m[2m │ gzip:   1.68 kB[22m
[2mdist/[22m[36mjs/category-CJ5HeWPr.js                                                                          [39m[1m[2m    3.49 kB[22m[1m[22m[2m │ gzip:   1.61 kB[22m
[2mdist/[22m[36mjs/instance-params-modal.vue_vue_type_script_setup_true_lang-DApQ0iv0.js                         [39m[1m[2m    3.50 kB[22m[1m[22m[2m │ gzip:   1.74 kB[22m
[2mdist/[22m[36mjs/index-CUgYWIli.js                                                                             [39m[1m[2m    3.51 kB[22m[1m[22m[2m │ gzip:   1.57 kB[22m
[2mdist/[22m[36mjs/category-CDTe9QDZ.js                                                                          [39m[1m[2m    3.51 kB[22m[1m[22m[2m │ gzip:   1.62 kB[22m
[2mdist/[22m[36mjs/heart-beat-list-CsOddlWP.js                                                                   [39m[1m[2m    3.58 kB[22m[1m[22m[2m │ gzip:   2.08 kB[22m
[2mdist/[22m[36mjs/point-detail-modal.vue_vue_type_script_setup_true_lang-B9ggD5Qb.js                            [39m[1m[2m    3.64 kB[22m[1m[22m[2m │ gzip:   1.66 kB[22m
[2mdist/[22m[36mjs/operate-log-detail-modal-Dcc3ptB3.js                                                          [39m[1m[2m    3.65 kB[22m[1m[22m[2m │ gzip:   1.58 kB[22m
[2mdist/[22m[36mjs/notice-employee-list.vue_vue_type_script_setup_true_lang-BkpYAGXQ.js                          [39m[1m[2m    3.65 kB[22m[1m[22m[2m │ gzip:   1.75 kB[22m
[2mdist/[22m[36mjs/gateway-form-modal.vue_vue_type_script_setup_true_lang-DZ_-ddlw.js                            [39m[1m[2m    3.65 kB[22m[1m[22m[2m │ gzip:   1.89 kB[22m
[2mdist/[22m[36mjs/mass-send-detail-drawer.vue_vue_type_script_setup_true_lang-BEn8eCuK.js                       [39m[1m[2m    3.69 kB[22m[1m[22m[2m │ gzip:   1.72 kB[22m
[2mdist/[22m[36mjs/config-list-Cpwy3nL0.js                                                                       [39m[1m[2m    3.70 kB[22m[1m[22m[2m │ gzip:   1.74 kB[22m
[2mdist/[22m[36mjs/home-quick-entry-CH8X_p5_.js                                                                  [39m[1m[2m    3.70 kB[22m[1m[22m[2m │ gzip:   1.77 kB[22m
[2mdist/[22m[36mjs/tenant-form-modal.vue_vue_type_script_setup_true_lang-D7lYxBNS.js                             [39m[1m[2m    3.72 kB[22m[1m[22m[2m │ gzip:   1.76 kB[22m
[2mdist/[22m[36mjs/card-DH2C5cua.js                                                                              [39m[1m[2m    3.76 kB[22m[1m[22m[2m │ gzip:   1.68 kB[22m
[2mdist/[22m[36mjs/notice-detail-Cw11rcx8.js                                                                     [39m[1m[2m    3.76 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/index-Cinza2Zg.js                                                                             [39m[1m[2m    3.80 kB[22m[1m[22m[2m │ gzip:   1.80 kB[22m
[2mdist/[22m[36mjs/redpack-BYneCPuB.js                                                                           [39m[1m[2m    3.83 kB[22m[1m[22m[2m │ gzip:   2.16 kB[22m
[2mdist/[22m[36mjs/config-operate-drawer-DLSj7ncI.js                                                             [39m[1m[2m    3.83 kB[22m[1m[22m[2m │ gzip:   1.86 kB[22m
[2mdist/[22m[36mjs/index-DMEGRfNB.js                                                                             [39m[1m[2m    3.91 kB[22m[1m[22m[2m │ gzip:   1.80 kB[22m
[2mdist/[22m[36mjs/device-trace-zone.vue_vue_type_script_setup_true_lang-B33SmiJq.js                             [39m[1m[2m    3.95 kB[22m[1m[22m[2m │ gzip:   1.93 kB[22m
[2mdist/[22m[36mjs/table-BNdv2-jZ.js                                                                             [39m[1m[2m    3.97 kB[22m[1m[22m[2m │ gzip:   1.69 kB[22m
[2mdist/[22m[36mjs/mass-send-create-modal.vue_vue_type_script_setup_true_lang-KebhUME2.js                        [39m[1m[2m    4.01 kB[22m[1m[22m[2m │ gzip:   2.05 kB[22m
[2mdist/[22m[36mjs/DataPush-gNMKBhN7.js                                                                          [39m[1m[2m    4.02 kB[22m[1m[22m[2m │ gzip:   2.08 kB[22m
[2mdist/[22m[36mjs/device-curve-zone.vue_vue_type_script_setup_true_lang-BHhFMkC-.js                             [39m[1m[2m    4.04 kB[22m[1m[22m[2m │ gzip:   2.22 kB[22m
[2mdist/[22m[36mjs/ThunderboltOutlined-CiyBymox.js                                                               [39m[1m[2m    4.07 kB[22m[1m[22m[2m │ gzip:   1.49 kB[22m
[2mdist/[22m[36mjs/analysis-B7SDp5ln.js                                                                          [39m[1m[2m    4.07 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/publish-log-drawer-PBP3od9U.js                                                                [39m[1m[2m    4.08 kB[22m[1m[22m[2m │ gzip:   1.95 kB[22m
[2mdist/[22m[36mjs/panel-C5CeHp2J.js                                                                             [39m[1m[2m    4.11 kB[22m[1m[22m[2m │ gzip:   2.07 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-SEluygqX.js                                                  [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/cross-DX7JatCx.js                                                                             [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   2.03 kB[22m
[2mdist/[22m[36mjs/index-C9t5wAEX.js                                                                             [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   2.22 kB[22m
[2mdist/[22m[36mjs/index-BTtg3ID7.js                                                                             [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   2.05 kB[22m
[2mdist/[22m[36mjs/index-BCUX8Ifg.js                                                                             [39m[1m[2m    4.17 kB[22m[1m[22m[2m │ gzip:   2.17 kB[22m
[2mdist/[22m[36mjs/index-BmsJnTiG.js                                                                             [39m[1m[2m    4.18 kB[22m[1m[22m[2m │ gzip:   1.81 kB[22m
[2mdist/[22m[36mjs/index-CLCqc0dB.js                                                                             [39m[1m[2m    4.19 kB[22m[1m[22m[2m │ gzip:   1.96 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-query-field.vue_vue_type_script_setup_true_lang-Culpw5k3.js  [39m[1m[2m    4.20 kB[22m[1m[22m[2m │ gzip:   1.84 kB[22m
[2mdist/[22m[36mjs/index-CaKaydNl.js                                                                             [39m[1m[2m    4.24 kB[22m[1m[22m[2m │ gzip:   2.19 kB[22m
[2mdist/[22m[36mjs/dict-data-modal.vue_vue_type_script_setup_true_lang-DQjAaAMZ.js                               [39m[1m[2m    4.28 kB[22m[1m[22m[2m │ gzip:   2.12 kB[22m
[2mdist/[22m[36mjs/notify-B0BE61Sy.js                                                                            [39m[1m[2m    4.28 kB[22m[1m[22m[2m │ gzip:   2.18 kB[22m
[2mdist/[22m[36mjs/FileOutlined-DcnGUgAz.js                                                                      [39m[1m[2m    4.29 kB[22m[1m[22m[2m │ gzip:   1.17 kB[22m
[2mdist/[22m[36mjs/index-D3Kme7qj.js                                                                             [39m[1m[2m    4.29 kB[22m[1m[22m[2m │ gzip:   2.15 kB[22m
[2mdist/[22m[36mjs/index-Dfzu0Cjo.js                                                                             [39m[1m[2m    4.34 kB[22m[1m[22m[2m │ gzip:   2.20 kB[22m
[2mdist/[22m[36mjs/share-DOVEnO7t.js                                                                             [39m[1m[2m    4.37 kB[22m[1m[22m[2m │ gzip:   1.98 kB[22m
[2mdist/[22m[36mjs/position-list-B6HIAPTd.js                                                                     [39m[1m[2m    4.39 kB[22m[1m[22m[2m │ gzip:   2.03 kB[22m
[2mdist/[22m[36mjs/module-list.vue_vue_type_script_setup_true_lang-B_QrlwOj.js                                   [39m[1m[2m    4.41 kB[22m[1m[22m[2m │ gzip:   2.01 kB[22m
[2mdist/[22m[36mjs/index-BPFCoRk4.js                                                                             [39m[1m[2m    4.45 kB[22m[1m[22m[2m │ gzip:   2.55 kB[22m
[2mdist/[22m[36mjs/template-send-detail-drawer-C59n3hI6.js                                                       [39m[1m[2m    4.45 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/code-generator-list-DlmQ45DY.js                                                               [39m[1m[2m    4.51 kB[22m[1m[22m[2m │ gzip:   2.00 kB[22m
[2mdist/[22m[36mjs/module-form-CSsprbLh.js                                                                       [39m[1m[2m    4.56 kB[22m[1m[22m[2m │ gzip:   2.28 kB[22m
[2mdist/[22m[36mjs/index-Be8ezv8x.js                                                                             [39m[1m[2m    4.57 kB[22m[1m[22m[2m │ gzip:   2.14 kB[22m
[2mdist/[22m[36mjs/job-log-list-modal.vue_vue_type_script_setup_true_lang-BvGkt_1r.js                            [39m[1m[2m    4.62 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/job-log-modal.vue_vue_type_script_setup_true_lang-XU4WRbLH.js                                 [39m[1m[2m    4.69 kB[22m[1m[22m[2m │ gzip:   2.15 kB[22m
[2mdist/[22m[36mjs/DataFilter-CkFFFpvy.js                                                                        [39m[1m[2m    4.72 kB[22m[1m[22m[2m │ gzip:   1.87 kB[22m
[2mdist/[22m[36mjs/dict-data-modal.vue_vue_type_script_setup_true_lang-9KwSVQ7o.js                               [39m[1m[2m    4.79 kB[22m[1m[22m[2m │ gzip:   2.19 kB[22m
[2mdist/[22m[36mjs/help-doc-list.vue_vue_type_script_setup_true_lang-C9WSJYZ0.js                                 [39m[1m[2m    4.79 kB[22m[1m[22m[2m │ gzip:   2.19 kB[22m
[2mdist/[22m[36mjs/login-fail-list-GVaC1i3C.js                                                                   [39m[1m[2m    4.81 kB[22m[1m[22m[2m │ gzip:   2.18 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-list.vue_vue_type_script_setup_true_lang-CvL6dG_p.js                       [39m[1m[2m    4.81 kB[22m[1m[22m[2m │ gzip:   2.30 kB[22m
[2mdist/[22m[36mjs/apiclient-form-modal-Dwgy31Ef.js                                                              [39m[1m[2m    4.82 kB[22m[1m[22m[2m │ gzip:   2.48 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-list.vue_vue_type_script_setup_true_lang-BU7jkTOW.js                          [39m[1m[2m    4.82 kB[22m[1m[22m[2m │ gzip:   2.27 kB[22m
[2mdist/[22m[36mjs/create-D20JzPdd.js                                                                            [39m[1m[2m    4.85 kB[22m[1m[22m[2m │ gzip:   2.40 kB[22m
[2mdist/[22m[36mjs/index-Bp9N5p-5.js                                                                             [39m[1m[2m    4.86 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/index-Vm70QDjl.js                                                                             [39m[1m[2m    4.94 kB[22m[1m[22m[2m │ gzip:   2.29 kB[22m
[2mdist/[22m[36mjs/ZoomOutOutlined-DiZf_zUf.js                                                                   [39m[1m[2m    5.02 kB[22m[1m[22m[2m │ gzip:   1.43 kB[22m
[2mdist/[22m[36mjs/message-list-BaGKroVA.js                                                                      [39m[1m[2m    5.12 kB[22m[1m[22m[2m │ gzip:   2.27 kB[22m
[2mdist/[22m[36mjs/keyword-form-modal.vue_vue_type_script_setup_true_lang-CT_NPxop.js                            [39m[1m[2m    5.15 kB[22m[1m[22m[2m │ gzip:   2.35 kB[22m
[2mdist/[22m[36mjs/instance-item-drawer-Cnax4k0A.js                                                              [39m[1m[2m    5.18 kB[22m[1m[22m[2m │ gzip:   2.49 kB[22m
[2mdist/[22m[36mjs/index-Wt7fxvxF.js                                                                             [39m[1m[2m    5.18 kB[22m[1m[22m[2m │ gzip:   2.22 kB[22m
[2mdist/[22m[36mjs/layout-const-BBvDbc76.js                                                                      [39m[1m[2m    5.21 kB[22m[1m[22m[2m │ gzip:   2.63 kB[22m
[2mdist/[22m[36mjs/job-form-modal.vue_vue_type_script_setup_true_lang-CUOXNl1z.js                                [39m[1m[2m    5.33 kB[22m[1m[22m[2m │ gzip:   2.17 kB[22m
[2mdist/[22m[36mjs/index-CozwsGVz.js                                                                             [39m[1m[2m    5.44 kB[22m[1m[22m[2m │ gzip:   2.35 kB[22m
[2mdist/[22m[36mjs/index-B-R48x9d.js                                                                             [39m[1m[2m    5.45 kB[22m[1m[22m[2m │ gzip:   2.52 kB[22m
[2mdist/[22m[36mjs/help-doc-form-drawer-B20lPpxw.js                                                              [39m[1m[2m    5.46 kB[22m[1m[22m[2m │ gzip:   2.54 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-tree-BMEI6va7.js                                                             [39m[1m[2m    5.46 kB[22m[1m[22m[2m │ gzip:   2.49 kB[22m
[2mdist/[22m[36mjs/template-send-create-modal-BpvjxQdL.js                                                        [39m[1m[2m    5.48 kB[22m[1m[22m[2m │ gzip:   2.91 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-field.vue_vue_type_script_setup_true_lang-DQs4V4Ef.js        [39m[1m[2m    5.58 kB[22m[1m[22m[2m │ gzip:   2.27 kB[22m
[2mdist/[22m[36mjs/index-rwuEYU9O.js                                                                             [39m[1m[2m    5.58 kB[22m[1m[22m[2m │ gzip:   2.46 kB[22m
[2mdist/[22m[36mjs/index-DPWE7eIN.js                                                                             [39m[1m[2m    5.59 kB[22m[1m[22m[2m │ gzip:   2.38 kB[22m
[2mdist/[22m[36mjs/config-list.vue_vue_type_script_setup_true_lang-DiLPWnRq.js                                   [39m[1m[2m    5.63 kB[22m[1m[22m[2m │ gzip:   2.31 kB[22m
[2mdist/[22m[36mjs/index-CRIBxBqg.js                                                                             [39m[1m[2m    5.69 kB[22m[1m[22m[2m │ gzip:   2.32 kB[22m
[2mdist/[22m[36mjs/index-qjbzY-pG.js                                                                             [39m[1m[2m    5.70 kB[22m[1m[22m[2m │ gzip:   2.58 kB[22m
[2mdist/[22m[36mjs/iot-formula-editor-D7Z08JqL.js                                                                [39m[1m[2m    5.72 kB[22m[1m[22m[2m │ gzip:   2.89 kB[22m
[2mdist/[22m[36mjs/index-CNE6F9Pn.js                                                                             [39m[1m[2m    5.73 kB[22m[1m[22m[2m │ gzip:   2.48 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_style_index_0_lang-EHr1HQrk.js                                             [39m[1m[2m    5.74 kB[22m[1m[22m[2m │ gzip:   1.88 kB[22m
[2mdist/[22m[36mjs/enterprise-list-CHr-XaJP.js                                                                   [39m[1m[2m    5.77 kB[22m[1m[22m[2m │ gzip:   2.64 kB[22m
[2mdist/[22m[36mjs/index-AXbtQWm3.js                                                                             [39m[1m[2m    5.81 kB[22m[1m[22m[2m │ gzip:   2.66 kB[22m
[2mdist/[22m[36mjs/instance-form-modal.vue_vue_type_script_setup_true_lang-CIbIU7gR.js                           [39m[1m[2m    5.89 kB[22m[1m[22m[2m │ gzip:   2.79 kB[22m
[2mdist/[22m[36mjs/point-form-modal.vue_vue_type_script_setup_true_lang-7J_gm41l.js                              [39m[1m[2m    5.94 kB[22m[1m[22m[2m │ gzip:   2.59 kB[22m
[2mdist/[22m[36mjs/api-encrypt-index-w2_qoBq3.js                                                                 [39m[1m[2m    5.99 kB[22m[1m[22m[2m │ gzip:   1.74 kB[22m
[2mdist/[22m[36mjs/goods-form-modal.vue_vue_type_script_setup_true_lang-CkkYiH5K.js                              [39m[1m[2m    6.11 kB[22m[1m[22m[2m │ gzip:   2.58 kB[22m
[2mdist/[22m[36mjs/file-list-B9yMwB1Y.js                                                                         [39m[1m[2m    6.14 kB[22m[1m[22m[2m │ gzip:   2.44 kB[22m
[2mdist/[22m[36mjs/index-DHdPSwMi.js                                                                             [39m[1m[2m    6.15 kB[22m[1m[22m[2m │ gzip:   2.77 kB[22m
[2mdist/[22m[36mjs/level3-protect-config-index-DCH8guCD.js                                                       [39m[1m[2m    6.20 kB[22m[1m[22m[2m │ gzip:   2.80 kB[22m
[2mdist/[22m[36mjs/index-xRFTRGcR.js                                                                             [39m[1m[2m    6.20 kB[22m[1m[22m[2m │ gzip:   2.88 kB[22m
[2mdist/[22m[36mjs/channel-credential-drawer-D2lRYPiH.js                                                         [39m[1m[2m    6.20 kB[22m[1m[22m[2m │ gzip:   3.11 kB[22m
[2mdist/[22m[36mjs/notice-form-drawer-D2RhKs5Y.js                                                                [39m[1m[2m    6.22 kB[22m[1m[22m[2m │ gzip:   2.71 kB[22m
[2mdist/[22m[36mjs/deleted-job-list.vue_vue_type_script_setup_true_lang-BOS6Hkca.js                              [39m[1m[2m    6.23 kB[22m[1m[22m[2m │ gzip:   2.61 kB[22m
[2mdist/[22m[36mjs/login-log-list-BHK0SLGl.js                                                                    [39m[1m[2m    6.24 kB[22m[1m[22m[2m │ gzip:   2.75 kB[22m
[2mdist/[22m[36mjs/change-log-list-C2Q_XrYA.js                                                                   [39m[1m[2m    6.25 kB[22m[1m[22m[2m │ gzip:   2.55 kB[22m
[2mdist/[22m[36mjs/index-W9XwhcsW.js                                                                             [39m[1m[2m    6.28 kB[22m[1m[22m[2m │ gzip:   2.55 kB[22m
[2mdist/[22m[36mjs/index-0NTN2EzY.js                                                                             [39m[1m[2m    6.36 kB[22m[1m[22m[2m │ gzip:   2.62 kB[22m
[2mdist/[22m[36mjs/index-DgoN7Cj-.js                                                                             [39m[1m[2m    6.47 kB[22m[1m[22m[2m │ gzip:   2.69 kB[22m
[2mdist/[22m[36mjs/channel-form-modal.vue_vue_type_script_setup_true_lang-DvXhp4Fb.js                            [39m[1m[2m    6.52 kB[22m[1m[22m[2m │ gzip:   3.21 kB[22m
[2mdist/[22m[36mjs/ViewData-CRkS8K97.js                                                                          [39m[1m[2m    6.56 kB[22m[1m[22m[2m │ gzip:   2.46 kB[22m
[2mdist/[22m[36mjs/index-DYiiq2k1.js                                                                             [39m[1m[2m    6.58 kB[22m[1m[22m[2m │ gzip:   2.72 kB[22m
[2mdist/[22m[36mjs/index-DFFf95Tk.js                                                                             [39m[1m[2m    6.68 kB[22m[1m[22m[2m │ gzip:   3.13 kB[22m
[2mdist/[22m[36mjs/index-EZlDy_os.js                                                                             [39m[1m[2m    6.71 kB[22m[1m[22m[2m │ gzip:   2.85 kB[22m
[2mdist/[22m[36mjs/ViewOrUpdate-D8TCPGLo.js                                                                      [39m[1m[2m    6.74 kB[22m[1m[22m[2m │ gzip:   3.25 kB[22m
[2mdist/[22m[36mjs/index-1RccozLA.js                                                                             [39m[1m[2m    6.80 kB[22m[1m[22m[2m │ gzip:   3.00 kB[22m
[2mdist/[22m[36mjs/index-RZpwXbJj.js                                                                             [39m[1m[2m    6.89 kB[22m[1m[22m[2m │ gzip:   2.61 kB[22m
[2mdist/[22m[36mjs/metric-form-modal-BAwqCTsl.js                                                                 [39m[1m[2m    6.96 kB[22m[1m[22m[2m │ gzip:   2.96 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-insert-and-update-CWabf2tV.js                                [39m[1m[2m    7.05 kB[22m[1m[22m[2m │ gzip:   2.56 kB[22m
[2mdist/[22m[36mjs/publish-confirm-modal-B_QfJp2b.js                                                             [39m[1m[2m    7.22 kB[22m[1m[22m[2m │ gzip:   3.50 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-basic-CBMS_YoS.js                                            [39m[1m[2m    7.39 kB[22m[1m[22m[2m │ gzip:   2.78 kB[22m
[2mdist/[22m[36mjs/index-DGlYjzSV.js                                                                             [39m[1m[2m    7.41 kB[22m[1m[22m[2m │ gzip:   3.26 kB[22m
[2mdist/[22m[36mjs/notice-list-CS0fadvA.js                                                                       [39m[1m[2m    7.45 kB[22m[1m[22m[2m │ gzip:   2.89 kB[22m
[2mdist/[22m[36mjs/menu-list.vue_vue_type_script_setup_true_lang-yRy9lPrX.js                                     [39m[1m[2m    7.50 kB[22m[1m[22m[2m │ gzip:   2.93 kB[22m
[2mdist/[22m[36mjs/index-CWEK0quZ.js                                                                             [39m[1m[2m    7.55 kB[22m[1m[22m[2m │ gzip:   3.53 kB[22m
[2mdist/[22m[36mjs/operate-log-list-BuI_pUAD.js                                                                  [39m[1m[2m    7.65 kB[22m[1m[22m[2m │ gzip:   3.21 kB[22m
[2mdist/[22m[36mjs/index-BxBUxnM-.js                                                                             [39m[1m[2m    7.72 kB[22m[1m[22m[2m │ gzip:   3.26 kB[22m
[2mdist/[22m[36mjs/job-list-DRKYHClr.js                                                                          [39m[1m[2m    8.38 kB[22m[1m[22m[2m │ gzip:   3.34 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-Di0Bhnxf.js                                         [39m[1m[2m    8.59 kB[22m[1m[22m[2m │ gzip:   3.68 kB[22m
[2mdist/[22m[36mjs/enterprise-employee-list-DnnG5WCf.js                                                          [39m[1m[2m    8.63 kB[22m[1m[22m[2m │ gzip:   3.32 kB[22m
[2mdist/[22m[36mjs/index-DC9mK8Ih.js                                                                             [39m[1m[2m    8.75 kB[22m[1m[22m[2m │ gzip:   3.22 kB[22m
[2mdist/[22m[36mjs/index-CPY75N6h.js                                                                             [39m[1m[2m    8.99 kB[22m[1m[22m[2m │ gzip:   4.14 kB[22m
[2mdist/[22m[36mjs/index-CJs2CZcR.js                                                                             [39m[1m[2m    9.12 kB[22m[1m[22m[2m │ gzip:   3.90 kB[22m
[2mdist/[22m[36mjs/index-DcLjq755.js                                                                             [39m[1m[2m    9.55 kB[22m[1m[22m[2m │ gzip:   4.01 kB[22m
[2mdist/[22m[36mjs/index-Dq0p7cpa.js                                                                             [39m[1m[2m    9.67 kB[22m[1m[22m[2m │ gzip:   3.82 kB[22m
[2mdist/[22m[36mjs/index-BE2UHgSh.js                                                                             [39m[1m[2m    9.74 kB[22m[1m[22m[2m │ gzip:   3.94 kB[22m
[2mdist/[22m[36mjs/index-DiSF0CXt.js                                                                             [39m[1m[2m    9.86 kB[22m[1m[22m[2m │ gzip:   4.05 kB[22m
[2mdist/[22m[36mjs/index-CXShLSBR.js                                                                             [39m[1m[2m   10.17 kB[22m[1m[22m[2m │ gzip:   3.82 kB[22m
[2mdist/[22m[36mjs/template-item-modal.vue_vue_type_script_setup_true_lang-CpHqJ7m8.js                           [39m[1m[2m   10.39 kB[22m[1m[22m[2m │ gzip:   4.03 kB[22m
[2mdist/[22m[36mjs/SendOutlined-H1E0esji.js                                                                      [39m[1m[2m   10.86 kB[22m[1m[22m[2m │ gzip:   2.66 kB[22m
[2mdist/[22m[36mjs/index-CKD0uLv4.js                                                                             [39m[1m[2m   10.94 kB[22m[1m[22m[2m │ gzip:   4.74 kB[22m
[2mdist/[22m[36mjs/index-EGxhiOnA.js                                                                             [39m[1m[2m   11.15 kB[22m[1m[22m[2m │ gzip:   4.35 kB[22m
[2mdist/[22m[36mjs/index-BE6jecPF.js                                                                             [39m[1m[2m   11.17 kB[22m[1m[22m[2m │ gzip:   3.31 kB[22m
[2mdist/[22m[36mjs/menu-operate-modal-DRNKj4Hc.js                                                                [39m[1m[2m   11.36 kB[22m[1m[22m[2m │ gzip:   4.12 kB[22m
[2mdist/[22m[36mjs/write-CvIhQrrk.js                                                                             [39m[1m[2m   12.75 kB[22m[1m[22m[2m │ gzip:   3.98 kB[22m
[2mdist/[22m[36mjs/goods-list-CDWcYs1h.js                                                                        [39m[1m[2m   15.49 kB[22m[1m[22m[2m │ gzip:   5.61 kB[22m
[2mdist/[22m[36mjs/index-BY6KIZDF.js                                                                             [39m[1m[2m   16.01 kB[22m[1m[22m[2m │ gzip:   6.63 kB[22m
[2mdist/[22m[36mjs/ua-parser-BVvIs2SD.js                                                                         [39m[1m[2m   16.51 kB[22m[1m[22m[2m │ gzip:   7.33 kB[22m
[2mdist/[22m[36mjs/EnterOutlined-p_X2kYL4.js                                                                     [39m[1m[2m   18.83 kB[22m[1m[22m[2m │ gzip:   3.43 kB[22m
[2mdist/[22m[36mjs/VideoCameraOutlined-CrBA_UUX.js                                                               [39m[1m[2m   20.27 kB[22m[1m[22m[2m │ gzip:   4.81 kB[22m
[2mdist/[22m[36mjs/qrcode.vue.esm-BlhLhp_9.js                                                                    [39m[1m[2m   20.75 kB[22m[1m[22m[2m │ gzip:   7.31 kB[22m
[2mdist/[22m[36mjs/bundle.esm-BjRzdrCS.js                                                                        [39m[1m[2m   20.76 kB[22m[1m[22m[2m │ gzip:   5.97 kB[22m
[2mdist/[22m[36mjs/index-BnB1119F.js                                                                             [39m[1m[2m   25.00 kB[22m[1m[22m[2m │ gzip:   8.49 kB[22m
[2mdist/[22m[36mjs/avatar-cropper-modal-BrOX9AZY.js                                                              [39m[1m[2m   35.31 kB[22m[1m[22m[2m │ gzip:  10.36 kB[22m
[2mdist/[22m[36mjs/sortable.esm-CRp0PNdb.js                                                                      [39m[1m[2m   35.52 kB[22m[1m[22m[2m │ gzip:  12.26 kB[22m
[2mdist/[22m[36mjs/index.esm-Blk7uwCv.js                                                                         [39m[1m[2m   45.58 kB[22m[1m[22m[2m │ gzip:  17.23 kB[22m
[2mdist/[22m[36mjs/enterprise-detail-vSgpm--t.js                                                                 [39m[1m[2m   57.70 kB[22m[1m[22m[2m │ gzip:  17.09 kB[22m
[2mdist/[22m[36mjs/index-3QHXOE11.js                                                                             [39m[1m[2m   57.90 kB[22m[1m[22m[2m │ gzip:  14.30 kB[22m
[2mdist/[22m[36mjs/enterprise-operate-modal-BkdG2erP.js                                                          [39m[1m[2m  151.85 kB[22m[1m[22m[2m │ gzip:  34.98 kB[22m
[2mdist/[22m[36mjs/index-D7dAzSsZ.js                                                                             [39m[1m[2m  170.10 kB[22m[1m[22m[2m │ gzip:  49.29 kB[22m
[2mdist/[22m[36mjs/home-header-CaHL31kq.js                                                                       [39m[1m[2m  294.39 kB[22m[1m[22m[2m │ gzip: 102.76 kB[22m
[2mdist/[22m[36mjs/index.esm-e1HGhDsu.js                                                                         [39m[1m[2m  458.39 kB[22m[1m[22m[2m │ gzip: 137.65 kB[22m
[2mdist/[22m[36mjs/index.esm-tTKwO1D1.js                                                                         [39m[1m[33m  547.28 kB[39m[22m[2m │ gzip: 157.18 kB[22m
[2mdist/[22m[36mjs/index-tA1Ph8hP.js                                                                             [39m[1m[33m  652.71 kB[39m[22m[2m │ gzip: 205.35 kB[22m
[2mdist/[22m[36mjs/code-generator-preview-modal-Dqj4UpPc.js                                                      [39m[1m[33m  915.72 kB[39m[22m[2m │ gzip: 303.29 kB[22m
[2mdist/[22m[36mjs/index-CrOPYkJv.js                                                                             [39m[1m[33m  928.90 kB[39m[22m[2m │ gzip: 274.04 kB[22m
[2mdist/[22m[36mjs/index-CehIs2yE.js                                                                             [39m[1m[33m  992.43 kB[39m[22m[2m │ gzip: 159.99 kB[22m
[2mdist/[22m[36mjs/index-BhzyNGex.js                                                                             [39m[1m[33m1,011.26 kB[39m[22m[2m │ gzip: 325.72 kB[22m
[2mdist/[22m[36mjs/index-BAsMtFso.js                                                                             [39m[1m[33m1,015.77 kB[39m[22m[2m │ gzip: 329.40 kB[22m
[2mdist/[22m[36mjs/index-COjayrzZ.js                                                                             [39m[1m[33m1,316.21 kB[39m[22m[2m │ gzip: 432.27 kB[22m
[33m
(!) Some chunks are larger than 500 kB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking: https://rollupjs.org/configuration-options/#output-manualchunks
- Adjust chunk size limit for this warning via build.chunkSizeWarningLimit.[39m
[32m✓ built in 1m 14s[39m
```

### `pnpm lint`

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

### `pnpm typecheck`

```text
$ vue-tsc --noEmit
```

### `pnpm test`

```text
$ vitest run

[1m[7m[36m RUN [39m[27m[22m [36mv2.1.9 [39m[90mD:/workspace/WisFactory/frontend[39m

 [32m✓[39m tests/unit/components/iot/formula-rules.spec.ts [2m([22m[2m32 tests[22m[2m)[22m[90m 22[2mms[22m[39m
 [32m✓[39m tests/unit/lib/axios.spec.ts [2m([22m[2m27 tests[22m[2m)[22m[33m 307[2mms[22m[39m
 [32m✓[39m tests/unit/composables/survey/use-edit-lock.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[90m 26[2mms[22m[39m
 [32m✓[39m tests/unit/wechat/template-send-helper.spec.ts [2m([22m[2m17 tests[22m[2m)[22m[90m 18[2mms[22m[39m
 [32m✓[39m tests/unit/router/build-routes.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[90m 56[2mms[22m[39m
 [32m✓[39m tests/unit/router/router-guard.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[90m 61[2mms[22m[39m
 [32m✓[39m tests/unit/components/iot/iot-formula-editor.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[33m 1394[2mms[22m[39m
   [33m[2m✓[22m[39m 公式编辑器：渲染与实时校验[2m > [22m初始公式渲染到 textarea，合法公式无错误提示 [33m436[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/item-formula-rollup-modal.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[33m 1706[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项取值编辑弹窗（TASK-0094）[2m > [22mformula 模式：父 ref 可调 showModal，回填公式并按清单构造 SELF 引用分组 [33m632[2mms[22m[39m
 [32m✓[39m tests/unit/display/metrics-board-read-ports.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[33m 2239[2mms[22m[39m
   [33m[2m✓[22m[39m 指标看板接真值读口（TASK-0089 D3）[2m > [22mmetricValues→卡片同源展示：value/yoy/mom/unit 原样，不现算 [33m711[2mms[22m[39m
   [33m[2m✓[22m[39m 指标看板接真值读口（TASK-0089 D3）[2m > [22m同 metricCode 多行 ts 升序（去年对比行在前）→ 本期值取最大 ts 行，不取首行旧值（I-1） [33m463[2mms[22m[39m
   [33m[2m✓[22m[39m 指标看板接真值读口（TASK-0089 D3）[2m > [22m读口失败：metrics 定义仍在、值/完整度空态，不炸页面（局部 catch 降级） [33m333[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m[2m顶行内同时渲染一级菜单与用户区（用户区不再挂在下方 header）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m[2m用户区排在菜单之后（DOM 顺序即视觉左右顺序：菜单在左、用户区在右）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m[2m顶行不再渲染二级菜单（二级菜单回到侧栏，由 layout 直接挂载）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m用户区组件 menu-user-space.vue[2m > [22m[2m渲染模块切换 / 消息 / 设置 / 头像四组元素
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m用户区组件 menu-user-space.vue[2m > [22m[2m组件自持配色解析，明暗切换不报错（学术蓝白底深字 / 经典深底白字）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m「顶部」布局复用同一用户区组件（top-menu/index.vue）[2m > [22m[2m渲染后用户区存在且与分组布局同源
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace> 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/top-expand-user-space.spec.ts [2m([22m[2m17 tests[22m[2m)[22m[33m 2668[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m顶行内同时渲染一级菜单与用户区（用户区不再挂在下方 header） [33m1007[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m用户区排在菜单之后（DOM 顺序即视觉左右顺序：菜单在左、用户区在右） [33m620[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m顶行不再渲染二级菜单（二级菜单回到侧栏，由 layout 直接挂载） [33m336[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/metric.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[33m 7173[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m列表加载真实后端数据、无 IotPlaceholder；查询参数透传 [33m1610[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m详情按钮 → detail(id) → 详情弹窗展示（父 ref defineExpose） [33m1055[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m表单弹窗父 ref 可调用 showModal / validateRatio；比值型缺分子分母被拦 [33m1310[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m新增提交：calendarType 锁 1，非比值型 ratio_* 置空，metricCode 原样透传 [33m840[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m编辑提交：metricCode 携带原值（不可改语义），status 不进更新集，走 update [33m717[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m启停走 changeStatus，删除走 remove，不经过 update [33m1093[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m批量生成：instanceId 作 query 入参、清单作 body；空清单/非法口径被前置拦截 [33m545[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-read-ports.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 20[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-cache-frame.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[33m 7551[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m回显：后端 0/0 显示为未开启，1/1 显示为开启（null/0 不得误判为真） [33m2784[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m提交：布尔开关映射为 0/1（后端只接受 0/1，不接受布尔） [33m1652[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m提交：开启外链后 frameFlag=1 且带上 frameUrl [33m1338[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m提交：关闭外链时清空外链地址，避免库中残留旧地址与开关状态不一致 [33m1070[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m外链菜单仍需填组件路径（buildRoutes 会跳过无 component 的菜单项） [33m697[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m「顶部」布局仍不显示该设置项（无左侧栏）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m分组布局显示该设置项，且取值回显 store 宽度
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m传统布局仍显示该设置项（原行为不回归）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m「展开」(side-expand) 不显示该设置项（侧栏宽度由内部固定值决定，调了不生效）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/top-expand-side-menu.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[33m 4172[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局左侧菜单固定显示（BUG-0097）[2m > [22m无二级菜单（首页场景）时容器与 logo 区仍渲染且不被隐藏 [33m342[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m「顶部」布局仍不显示该设置项（无左侧栏） [33m1441[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m分组布局显示该设置项，且取值回显 store 宽度 [33m926[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m传统布局仍显示该设置项（原行为不回归） [33m836[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m「展开」(side-expand) 不显示该设置项（侧栏宽度由内部固定值决定，调了不生效） [33m440[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/point.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[33m 8716[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m挂载拉列表（默认分页）且无占位骨架文案 [33m1604[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m空数据时展示诚实空态，不造假数据 [33m803[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m新建/详情通过父 ref 调 defineExpose.showModal（BUG-0008 回归） [33m1176[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m启停调 changeStatus({ids,status})，停用含发布生效提示 [33m968[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m删除调 remove([id])（API 层拼路径参数），导出调 export 带过滤参数 [33m894[2mms[22m[39m
   [33m[2m✓[22m[39m 点位表单弹窗（TASK-0097）[2m > [22m新增提交调 add，payload 不含 status/id [33m854[2mms[22m[39m
   [33m[2m✓[22m[39m 点位表单弹窗（TASK-0097）[2m > [22m编辑提交调 update，携带 id 与 pointCode 同值，且不含 status [33m924[2mms[22m[39m
   [33m[2m✓[22m[39m 点位表单弹窗（TASK-0097）[2m > [22m编辑 showModal 后 pointCode 禁用（创建后不可改） [33m551[2mms[22m[39m
   [33m[2m✓[22m[39m 点位详情弹窗（TASK-0097）[2m > [22m父 ref 调 showModal(id) → detail/references/curve 三读，曲线空集展示诚实提示 [33m939[2mms[22m[39m
 [32m✓[39m tests/unit/store/user-module.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 210[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/instance-tree-edit.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[33m 7808[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22mtitle 插槽在工厂/车间/设备各级节点旁渲染 EditOutlined 编辑按钮 [33m2003[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m点击工厂节点编辑按钮 → 复用 addInstance 打开编辑弹窗（完整 record 传 showModal） [33m1048[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m骨架节点（缺 deptId/sort）点击 → 拉详情补齐字段后打开编辑弹窗 [33m736[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m详情加载失败时不开空弹窗，并上报 otSentry [33m614[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22mlocked 设备节点编辑按钮透传 locked=1（弹窗自身显示仅改状态提示） [33m847[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m无 iot:instance:edit 权限时编辑按钮被 v-privilege 指令移除（树节点仍在） [33m667[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m树过滤仍保留命中节点及其祖先路径，且编辑按钮随树保留 [33m834[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m树节点选择仍触发 onSelectTree（点击标签文字检索子树，与编辑按钮独立） [33m1056[2mms[22m[39m
[90mstderr[2m | unknown test
[22m[39mOPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
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

[90mstderr[2m | unknown test
[22m[39mDOMException [AbortError]: The operation was aborted.
    at Fetch.onAsyncTaskManagerAbort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/fetch/Fetch.ts:711:17[90m)[39m
    at Object.<anonymous> [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/fetch/Fetch.ts:545:10[90m)[39m
    at AsyncTaskManager.abortAll [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/async-task-manager/AsyncTaskManager.ts:331:21[90m)[39m
    at AsyncTaskManager.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/async-task-manager/AsyncTaskManager.ts:61:15[90m)[39m
    at DetachedBrowserFrame.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/browser/detached-browser/DetachedBrowserFrame.ts:139:49[90m)[39m
    at DetachedWindowAPI.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/window/DetachedWindowAPI.ts:62:29[90m)[39m
    at teardownWindow [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/index.K90BXFOx.js:363:24[90m)[39m
    at Object.teardown [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/index.K90BXFOx.js:415:15[90m)[39m
    at withEnv [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/runBaseTests.3qpJUEJM.js:94:15[90m)[39m
    at run [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/runBaseTests.3qpJUEJM.js:112:3[90m)[39m
DOMException [AbortError]: The operation was aborted.
    at Fetch.onAsyncTaskManagerAbort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/fetch/Fetch.ts:711:17[90m)[39m
    at Object.<anonymous> [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/fetch/Fetch.ts:545:10[90m)[39m
    at AsyncTaskManager.abortAll [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/async-task-manager/AsyncTaskManager.ts:331:21[90m)[39m
    at AsyncTaskManager.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/async-task-manager/AsyncTaskManager.ts:61:15[90m)[39m
    at DetachedBrowserFrame.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/browser/detached-browser/DetachedBrowserFrame.ts:139:49[90m)[39m
    at DetachedWindowAPI.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/window/DetachedWindowAPI.ts:62:29[90m)[39m
    at teardownWindow [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/index.K90BXFOx.js:363:24[90m)[39m
    at Object.teardown [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/index.K90BXFOx.js:415:15[90m)[39m
    at withEnv [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/runBaseTests.3qpJUEJM.js:94:15[90m)[39m
    at run [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/runBaseTests.3qpJUEJM.js:112:3[90m)[39m
DOMException [AbortError]: The operation was aborted.
    at Fetch.onAsyncTaskManagerAbort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/fetch/Fetch.ts:711:17[90m)[39m
    at Object.<anonymous> [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/fetch/Fetch.ts:545:10[90m)[39m
    at AsyncTaskManager.abortAll [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/async-task-manager/AsyncTaskManager.ts:331:21[90m)[39m
    at AsyncTaskManager.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/async-task-manager/AsyncTaskManager.ts:61:15[90m)[39m
    at DetachedBrowserFrame.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/browser/detached-browser/DetachedBrowserFrame.ts:139:49[90m)[39m
    at DetachedWindowAPI.abort [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/happy-dom@20.12.0/node_modules/[4mhappy-dom[24m/src/window/DetachedWindowAPI.ts:62:29[90m)[39m
    at teardownWindow [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/index.K90BXFOx.js:363:24[90m)[39m
    at Object.teardown [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/index.K90BXFOx.js:415:15[90m)[39m
    at withEnv [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/runBaseTests.3qpJUEJM.js:94:15[90m)[39m
    at run [90m(file:///D:/workspace/WisFactory/frontend/[39mnode_modules/[4m.pnpm[24m/vitest@2.1.9_@types+node@25_a451c38dbe93c0a23f5cd7047bf64320/node_modules/[4mvitest[24m/dist/chunks/runBaseTests.3qpJUEJM.js:112:3[90m)[39m

[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m[2mlayout/index.vue 渲染设置抽屉一份
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m[2m发 settingMitt open 事件后抽屉打开
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
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

 [32m✓[39m tests/unit/views/iot/point-bind-modal.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 3179[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项绑定点位弹窗（TASK-0092）[2m > [22m父 ref 可调 showModal；拉 status=1 启用点位列表并展示点位编码/名称/值类型 [33m1669[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项绑定点位弹窗（TASK-0092）[2m > [22m选一行点确定 → bindPoint(1001,2002,301) → 成功提示 + 事件通知 + 关窗 [33m886[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项绑定点位弹窗（TASK-0092）[2m > [22m绑定失败 → captureError + message.error 提示，弹窗不关闭 [33m440[2mms[22m[39m
 [32m✓[39m tests/unit/utils/public-utils-layering.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 20[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m[2m切换「菜单布局」后抽屉仍打开（修复前会随布局组件卸载而消失）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/wechat/mass-send-helper.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-perms-mapping.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[33m 11046[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m编辑回显：输入框显示库中 perms（修复前恒空） [33m1646[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m库中 perms 为 null 时回显空串，不显示字面量 null [33m1340[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m改值提交：updateMenu 收到 params.perms 为目标值（修复前后端收不到该字段） [33m1815[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m清空提交：提交空串而非 null（null 会被后端 UPDATE 跳过、清不掉原权限串） [33m1360[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m新增功能点带权限串提交：addMenu 收到 params.perms [33m681[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095）[2m > [22m编辑回显：输入框显示库中 query（修复前绑 queryParam，后端无此字段故恒空） [33m757[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095）[2m > [22m改值提交：updateMenu 收到 params.query 为目标值，且不再有 queryParam [33m1266[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095）[2m > [22m清空提交：提交空串而非 null（同 perms 口径，null 会被后端 UPDATE 跳过） [33m932[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090）[2m > [22m菜单类型抽屉无「路由名称」表单项 [33m466[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090）[2m > [22m提交报文不含 routeName [33m779[2mms[22m[39m
 [32m✓[39m tests/unit/survey/report-format.spec.ts [2m([22m[2m22 tests[22m[2m)[22m[90m 15[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉触发链路（BUG-0098 源码层）[2m > [22m[2m设置按钮仍渲染（用户区功能不回归）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/setting-drawer-persist.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[33m 3856[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22mlayout/index.vue 渲染设置抽屉一份 [33m364[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m发 settingMitt open 事件后抽屉打开 [33m2335[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m切换「菜单布局」后抽屉仍打开（修复前会随布局组件卸载而消失） [33m941[2mms[22m[39m
 [32m✓[39m tests/unit/theme/disabled-button-text.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 12[2mms[22m[39m
 [32m✓[39m tests/unit/utils/sanitize-html.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 143[2mms[22m[39m
 [32m✓[39m tests/unit/store/app-config-remote.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 50[2mms[22m[39m
 [32m✓[39m tests/unit/store/menu-module-helper.spec.ts [2m([22m[2m21 tests[22m[2m)[22m[90m 14[2mms[22m[39m
 [32m✓[39m tests/unit/composables/survey/project-list-columns.spec.ts [2m([22m[2m19 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/instance-item-value-editor.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 2791[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22m公式项/汇总项各显示对应编辑按钮，权限不足时按钮被移除 [33m944[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22m点击「编辑公式」→ 弹窗打开 → 保存 → 调 saveFormula 并刷新清单 [33m878[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22m点击「编辑汇总」→ 弹窗打开 → 切 AVG 保存 → 调 saveRollup 并刷新清单 [33m498[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22mlocked=1：编辑按钮禁用，点击提示锁定，不打开弹窗、不提交 [33m451[2mms[22m[39m
 [32m✓[39m tests/unit/layout/menu-path-match.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 8[2mms[22m[39m
[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m默认全部展开：每个顶级分类都渲染折叠图标（收起态图标不出现）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m点折叠图标收起该分类：图标切为收起态且其子级隐藏
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m再点一次恢复展开（收起可逆）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m多开：收起一个不影响其他分类的展开态
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m非顶级节点不渲染折叠图标（子目录/菜单不参与折叠）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m无子级的顶级节点不渲染折叠图标（防御性）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m点击折叠图标不会触发复选框勾选（click.stop 防回归）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

 [32m✓[39m tests/unit/views/system/role-tree-collapse.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[33m 1159[2mms[22m[39m
   [33m[2m✓[22m[39m 角色授权树顶级分类折叠[2m > [22m默认全部展开：每个顶级分类都渲染折叠图标（收起态图标不出现） [33m350[2mms[22m[39m
 [32m✓[39m tests/unit/survey/write-storage.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[90m 16[2mms[22m[39m
 [32m✓[39m tests/unit/views/survey/province-match.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 8[2mms[22m[39m
 [32m✓[39m tests/unit/display/metric-utils.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/actuator-api.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 22[2mms[22m[39m
 [32m✓[39m tests/unit/api/iot/point-api.spec.ts [2m([22m[2m11 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/survey/wx-auth.spec.ts [2m([22m[2m18 tests[22m[2m)[22m[90m 18[2mms[22m[39m
 [32m✓[39m tests/unit/layout/top-expand-theme.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[33m 688[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局一级菜单主题联动（top-menu.vue）[2m > [22m学术蓝（sideMenuTheme=light）→ 菜单渲染为 light，不含 dark [33m445[2mms[22m[39m
 [32m✓[39m tests/unit/views/survey/write-time-window.spec.ts [2m([22m[2m11 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/display/device-utils.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/survey/option-label-display.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[90m 247[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-card-status-values.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[33m 1213[2mms[22m[39m
   [33m[2m✓[22m[39m 工厂卡片接状态数真值（TASK-0058 批次二 D2）[2m > [22m挂载即用 node.id 取 factory-summary，运行/异常/离线三槽位填真值 [33m570[2mms[22m[39m
 [32m✓[39m tests/unit/store/ui-config-merge.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/views/survey/redpack-columns.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[33m 2684[2mms[22m[39m
   [33m[2m✓[22m[39m 问卷红包列表「发放人/发放结果」两列[2m > [22m列定义含「发放人」与「发放结果」两列，既有列不丢 [33m1401[2mms[22m[39m
   [33m[2m✓[22m[39m 问卷红包列表「发放人/发放结果」两列[2m > [22m发放人列取 sendByName，缺失显示「—」（未发放行/老后端行） [33m634[2mms[22m[39m
   [33m[2m✓[22m[39m 问卷红包列表「发放人/发放结果」两列[2m > [22m发放结果列五态文案与配色（成功绿/失败红/发放中蓝/待确认橙/未发放默认） [33m646[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/point-bind-drawer.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[33m 2072[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉绑定点位入口（TASK-0092）[2m > [22m「绑定点位」按钮挂 iot:item:edit 权限；无该权限时按钮被移除 [33m959[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉绑定点位入口（TASK-0092）[2m > [22m行内「绑定点位」按钮点击 → 拉启用点位列表 → 选点 → bindPoint → 刷新并显示点位编码 [33m1111[2mms[22m[39m
 [32m✓[39m tests/unit/display/device-realtime-zone.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[33m 1350[2mms[22m[39m
   [33m[2m✓[22m[39m 设备实时区接 realtime 真值（TASK-0089 D2）[2m > [22m绑定类 + 公式类统一取 realtime 真值（v/q/ts 同源），来源标注「实时值读口」 [33m807[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m改动配置后按作用域上报服务端（模块作用域）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/survey/option-label-map.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/actuator-helper.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[90m 114[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m无当前模块时上报到账户全局（moduleId=0）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m内部态变化（remoteItems/remoteLoaded）不上报，防模块间配置串写
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m恢复默认配置：删除当前作用域远端配置
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/header-setting-persist.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 622[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉界面偏好持久化[2m > [22m改动配置后按作用域上报服务端（模块作用域） [33m492[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/role-list-selection.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[33m 1992[2mms[22m[39m
   [33m[2m✓[22m[39m 角色列表刷新后选中态（组件级）[2m > [22m首次加载自动选中第一个角色 [33m729[2mms[22m[39m
   [33m[2m✓[22m[39m 角色列表刷新后选中态（组件级）[2m > [22m编辑提交触发刷新后，仍选中原来那个角色（不跳回超级管理员） [33m596[2mms[22m[39m
   [33m[2m✓[22m[39m 角色列表刷新后选中态（组件级）[2m > [22m删除的正是当前选中角色时，回落到第一个 [33m660[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/cache-monitor-api.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 28[2mms[22m[39m
 [32m✓[39m tests/unit/api/iot/metric-api.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[90m 14[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-operate-modal.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 6038[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m编辑回显：库中「不显示」的菜单开关为不显示，未动开关提交不翻转原值 [33m2485[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m关「显示状态」提交 visible=0（用户实测场景：修复前后端只收到旧值 1） [33m1078[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m关「菜单状态」提交 status=0，且可见性保持显示 [33m1781[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m新增菜单未动开关时提交 visible=1、status=1 [33m692[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-summary-read-port.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/survey/redpack-helper.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[90m 6[2mms[22m[39m
 [32m✓[39m tests/unit/views/wechat/tag-filter.spec.ts [2m([22m[2m13 tests[22m[2m)[22m[90m 8[2mms[22m[39m
 [32m✓[39m tests/unit/survey/redpack-list-helper.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/cache-helper.spec.ts [2m([22m[2m11 tests[22m[2m)[22m[90m 12[2mms[22m[39m
 [32m✓[39m tests/unit/store/dict.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 31[2mms[22m[39m
 [32m✓[39m tests/unit/survey/expand-original-data.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-tree.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-list-perms-column.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/utils/file-url.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/store/app-config.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[90m 297[2mms[22m[39m
 [32m✓[39m tests/unit/theme/theme-preset.spec.ts [2m([22m[2m13 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/survey/normalize-field-schemes.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 7[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-field-mapping.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/api/iot/point-bind-api.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 12[2mms[22m[39m
 [32m✓[39m tests/unit/api/survey/edit-lock-api.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-routes.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/iot-inspect-helper.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/login-entry-render.spec.ts [2m([22m[2m1 test[22m[2m)[22m[33m 909[2mms[22m[39m
   [33m[2m✓[22m[39m 登录页首屏组件渲染（入口精简集）[2m > [22m只注册 entryComponents 时，用户名/密码/验证码三件套都能渲染 [33m907[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/alarm-helper.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/plugin/entry-ui.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[33m 449[2mms[22m[39m
   [33m[2m✓[22m[39m plugin/entry-ui 入口精简集[2m > [22m登录表单（a-form / a-form-item / a-input-password）能渲染出密码输入框 [33m441[2mms[22m[39m
 [32m✓[39m tests/unit/wechat/send-common.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-page-metrics.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[33m 483[2mms[22m[39m
   [33m[2m✓[22m[39m 工厂页接工厂级指标真值（TASK-0058 批次二 D2）[2m > [22m工厂级指标看板：metricValues 同源展示 value/yoy/mom（不现算、null 不冒充 0） [33m321[2mms[22m[39m
 [32m✓[39m tests/unit/survey/serial-number.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 7[2mms[22m[39m
 [32m✓[39m tests/unit/lib/encrypt.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/layout/module-switcher.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[33m 396[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/instance-item-drawer-ref.spec.ts [2m([22m[2m1 test[22m[2m)[22m[33m 543[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉父组件 ref 暴露（BUG-0008）[2m > [22m父组件 ref 可拿到 showDrawer/onClose；调用后抽屉打开且标题可见 [33m541[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-card-summary.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 189[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/login-captcha-polling.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[33m 396[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-slot-card.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[90m 114[2mms[22m[39m
 [32m✓[39m tests/unit/composables/survey/use-project-creator.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 3[2mms[22m[39m

[2m Test Files [22m [1m[32m84 passed[39m[22m[90m (84)[39m
[2m      Tests [22m [1m[32m712 passed[39m[22m[90m (712)[39m
[2m   Start at [22m 21:03:03
[2m   Duration [22m 70.17s[2m (transform 36.92s, setup 0ms, collect 507.86s, tests 87.66s, environment 144.49s, prepare 31.01s)[22m

```

## 3 未做项与原因

- 指标模板载体自动生成：后端 `POST /iot/metric/generate` 的 `IotMetricGenerateForm` 仅接受“指标清单”，模板载体在后端 DTO 与 FEAT-0014 均标注为未定义占位。brief 裁决允许留“按清单批量生成”输入方式并在报告登记；前端不伪造模板，故未做模板选择器。
- `src_type` 来源类型（数据项/上游指标级联，ADR-0009）：后端 `IotMetric` 实体与 CreateForm 均未落该列/字段（实体头注明确“本期不落列不校验”），无对应接口字段，前端不自造控件。
- 上看板标记：后端实体头注标明随展示端任务落，本期无列无字段，未做。
- 数据权限前端裁剪：由后端 list/detail 负责，前端不造假数据、不做越权兜底过滤。
## 4 偏离计划之处

- 增加 `metric-detail-modal.vue`（brief 允许“可按需拆 detail modal”），属声明范围内的可选组件；详情通过 `GET /iot/metric/{id}` 读取并展示空态。
- 为复用枚举与避免字典漂移，在 brief 允许的 `iot-consts.ts` 追加 5 个指标常量（aggType/period/calendarNatural/ratioAggType/generateBatchMax）；未改其它枚举语义。
- 修复过程中的一次测试失败根因与修正：列表单测最初断言 document.body.textContent，但 VTU 默认挂载到脱离文档的节点，document.body 为空（与被测页面无关）。改为断言挂载实例自身的 wrapper.text() 后通过——先红后绿的实测证据见下方“验收过程留痕”。
- typecheck 发现两处 vi.fn 零参 mock 导致 .mock.calls[0][0] 元组越界，给 mock 增加明确入参签名后通过；未改实现或绕过检查。
- 表单弹窗/detail 弹窗均 `defineExpose`，父组件 ref 调用由单测覆盖，规避 BUG-0008。
## 5 遗留问题

- 批量生成的 `metricsText` 以 JSON 数组输入（附带字段口径前置提示），体验弱于结构化模板选择器；待指标模板载体定义后应替换为模板/清单控件。逐项 DTO 校验与整批事务仍以后端为准。
- 表单中“来源数据项/比值分子分母”为自由下拉加实例数据项清单（`itemApi.queryItemList`）；若后端对“同实例启用且存在”另有裁剪，以后端 4001 拒存为准。
- lint 全仓仍有 14 条 prettier warning，均为基线既有文件（point/business/survey 等），本任务新增/改动文件已清零 warning；0 error，不阻塞验收。
- TASK 状态保持「进行中」，收尾由协调者执行；修改内容/任务结果与计划子任务勾选写入 TASK 详情。

## 验收过程留痕（先红后绿）

1. 新增两测试文件首跑：`metric-api.spec.ts` 8/8 通过；`metric.spec.ts` 6 通过 / 1 失败——`expected '' to contain 'INS1_DAY_AVG'`。根因：VTU 默认 detached mount，document.body 未挂载组件文本；改断言为 `wrapper.text()` 后 7/7 通过。未修改实现或降低期望。
2. `pnpm typecheck` 首跑报 4 处 TS2493/TS18048：hoisted `vi.fn(() => ...)` 被推断为 0 元组，`.mock.calls[0][0]` 越界且 payload 可能 undefined。给 `addMetric`/`updateMetric` mock 增加必填入参类型后通过；未改实现或绕过检查。
3. 最终链式验收 exit 0；四段原始输出见 §2。

## 评审退回修复（Important：新增态比值型不可达）

### 根因

`metric-form-modal.vue` 的 `loadItemOptions(instanceId)` 原仅在 `showModal` 的 `if (rowData)` 编辑分支被调用；新增态 `showModal()` 无 `rowData`，`itemOptions` 被置空且无任何实例变更触发重载。`a-select` 无 `allowCreate`，用户选不到「来源数据项 / 比值分子 / 比值分母」，而后端 `validateAndBuild` / `validateRatio` 要求 aggType=5 时分子分母必填——直接新建比值型指标不可达（只能先建非比值型再编辑）。

### 修复

- 归属实例 `a-select` 增加 `@change="onInstanceChange"`；`onInstanceChange(instanceId)` 调用项目既有 `itemApi.queryItemList(instanceId)` 按实例加载数据项选项（真实后端数据，无实例/无数据项时诚实空态）。
- 换实例时清空 `sourceItemId` / `ratioNumItemId` / `ratioDenItemId`，避免提交跨实例引用（后端 4001 拒存）。
- `onInstanceChange` 加入 `defineExpose`，父 ref 可调用（与 `showModal` / `validateRatio` 同风格，BUG-0008 教训）。
- 改动文件仅 `frontend/src/views/iot/metric/components/metric-form-modal.vue` 与 `frontend/tests/unit/views/iot/metric.spec.ts`，均在 brief 声明范围内。

### 回归测试（先红后绿，原始输出）

新增用例「新增态选实例后加载数据项下拉，比值型可选分子/分母并成功提交（评审 Important 回归）」。

先红（修复前，仅运行该用例，`-t 评审 Important 回归`）：

```text
 RUN  v2.1.9 D:/workspace/WisFactory/frontend

 ❯ tests/unit/views/iot/metric.spec.ts (8 tests | 1 failed | 7 skipped) 367ms
   × 指标管理页与表单校验（TASK-0095） > 新增态选实例后加载数据项下拉，比值型可选分子/分母并成功提交（评审 Important 回归） 366ms
     → expected "spy" to be called with arguments: [ 1001 ]

 Received: 

 Number of calls: 0

 FAIL  tests/unit/views/iot/metric.spec.ts > 指标管理页与表单校验（TASK-0095） > 新增态选实例后加载数据项下拉，比值型可选分子/分母并成功提交（评审 Important 回归）
 AssertionError: expected "spy" to be called with arguments: [ 1001 ]
  Number of calls: 0
   ❯ tests/unit/views/iot/metric.spec.ts:201:27
       199|     instanceSelect.vm.$emit('change', 1001);
       200|     await flushPromises();
       201|     expect(queryItemList).toHaveBeenCalledWith(1001);
          |                          ^
       202|     expect(formVm.itemOptions).toEqual([{ label: 'I-001（用电量）', value: 11 }]);

 Test Files  1 failed (1)
      Tests  1 failed | 7 skipped (8)
   Duration  5.01s
```

根因证据：选中实例后 `queryItemList` 调用次数为 0——新增态没有任何加载数据项的触发路径。

后绿（修复后，仅运行该用例）：

```text
 RUN  v2.1.9 D:/workspace/WisFactory/frontend

 ✓ tests/unit/views/iot/metric.spec.ts (8 tests | 1 passed | 7 skipped) 679ms
   ✓ 指标管理页与表单校验（TASK-0095） > 新增态选实例后加载数据项下拉，比值型可选分子/分母并成功提交（评审 Important 回归） 677ms

 Test Files  1 passed (1)
      Tests  1 passed | 7 skipped (8)
   Duration  5.17s
```

修复后整组 metric 单测：2 文件 / 16 用例全绿（api 8 + view 8）。未修改测试期望或绕过检查，测试通过实例选择触发真实 `@change` 路径并断言 `queryItemList(1001)` + 选项值 + 比值型提交 payload。

### 修复后完整四门验收原始输出（链式一次跑完，exit 0）

基线 82 文件 / 697 用例 → 首轮 84/712 → 本轮 84/713（+1 回归用例，零回归）。

#### `pnpm build:test`

```text
$ vite build  --base=/admin/ --mode test
NODE_ENV=production is not supported in the .env file. Only NODE_ENV=development is supported to create a development build of your project. If you need to set process.env.NODE_ENV, you can set it in the Vite config instead.
[36mvite v5.2.12 [32mbuilding for test...[36m[39m
transforming...

./icon.svg referenced in D:/workspace/WisFactory/frontend/node_modules/.pnpm/vue3-json-viewer@2.3.1_vue@3.4.27_typescript@5.6.3_/node_modules/vue3-json-viewer/dist/index.css didn't resolve at build time, it will remain unchanged to be resolved at runtime
[1m[33m[plugin:vite:esbuild][39m[22m [33m[plugin vite:esbuild] src/views/business/oa/enterprise/components/enterprise-employee-list.vue?vue&type=script&setup=true&lang.ts: [33mThis assignment will throw because "columns" is a constant[33m
298|                class: "ot-margin-bottom5",
299|                modelValue: columns,
300|                "onUpdate:modelValue": _cache[1] || (_cache[1] = ($event: any) => ((columns) = $event)),
   |                                                                                    ^
301|                tableId: _unref(TABLE_ID_CONST).BUSINESS.OA.ENTERPRISE_EMPLOYEE,
302|                refresh: queryEmployee
[39m
[1m[33m[plugin:vite:esbuild][39m[22m [33m[plugin vite:esbuild] src/views/survey/components/FormAuthModal/panel.vue?vue&type=script&setup=true&lang.ts: [33mThis assignment will throw because "editors" is a constant[33m
244|                _createVNode(AuthMemberEditor, {
245|                  "member-ids": editors,
246|                  "onUpdate:memberIds": _cache[1] || (_cache[1] = ($event: any) => ((editors) = $event)),
   |                                                                                     ^
247|                  title: "编辑名单",
248|                  tip: "名单内成员可编辑该问卷（含查看）；支持按用户、角色、部门授权",
[39m
[1m[33m[plugin:vite:esbuild][39m[22m [33m[plugin vite:esbuild] src/views/survey/components/FormAuthModal/panel.vue?vue&type=script&setup=true&lang.ts: [33mThis assignment will throw because "viewers" is a constant[33m
254|                _createVNode(AuthMemberEditor, {
255|                  "member-ids": viewers,
256|                  "onUpdate:memberIds": _cache[2] || (_cache[2] = ($event: any) => ((viewers) = $event)),
   |                                                                                     ^
257|                  title: "查看名单",
258|                  tip: "名单内成员仅可查看该问卷（含数据与统计）；编辑名单成员已含查看权",
[39m
[32m✓[39m 6020 modules transformed.
rendering chunks...
[33mwarnings when minifying css:
▲ [WARNING] Unexpected "2of" [css-syntax-error]

    <stdin>:1:99250:
      1 │ ...ntents}.tox .tox-ai :nth-child(2of.tox-ai__response-status-icon)...
        ╵                                   ~~~

[39m
[1m[33m[plugin:vite:reporter][39m[22m [33m[plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/src/api/system/login-api.ts is dynamically imported by D:/workspace/WisFactory/frontend/src/router/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/layout/components/change-password/regular-change-password-modal.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/layout/components/header-user-space/header-avatar.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/main.ts, D:/workspace/WisFactory/frontend/src/views/system/account/components/center/index.vue?vue&type=script&setup=true&lang.ts, D:/workspace/WisFactory/frontend/src/views/system/login3/login.vue?vue&type=script&setup=true&lang.ts, dynamic import will not move module into another chunk.
[39m
[1m[33m[plugin:vite:reporter][39m[22m [33m[plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/src/plugin/admin-ui/index.ts is dynamically imported by D:/workspace/WisFactory/frontend/src/router/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/main.ts, dynamic import will not move module into another chunk.
[39m
[1m[33m[plugin:vite:reporter][39m[22m [33m[plugin vite:reporter] 
(!) D:/workspace/WisFactory/frontend/node_modules/.pnpm/ant-design-vue@4.2.5_vue@3.4.27_typescript@5.6.3_/node_modules/ant-design-vue/dist/reset.css is dynamically imported by D:/workspace/WisFactory/frontend/src/plugin/admin-ui/index.ts but also statically imported by D:/workspace/WisFactory/frontend/src/main.ts, dynamic import will not move module into another chunk.
[39m
computing gzip size...
[2mdist/[22m[32mindex.html                                                                                       [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.48 kB[22m
[2mdist/[22m[32mpng/ot-admin-logo-white-Dn8J4Vek.png                                                             [39m[1m[2m    8.11 kB[22m[1m[22m
[2mdist/[22m[32mpng/appearset_bgc_big-CBsi9EZh.png                                                               [39m[1m[2m    8.61 kB[22m[1m[22m
[2mdist/[22m[32mpng/ot-admin-logo-cDl7Ur4Y.png                                                                   [39m[1m[2m   14.33 kB[22m[1m[22m
[2mdist/[22m[32mpng/99d0-BLQr3eGW.png                                                                            [39m[1m[2m   37.33 kB[22m[1m[22m
[2mdist/[22m[32mpng/admin-logo-icon-BJtvcrOo.png                                                                 [39m[1m[2m   49.41 kB[22m[1m[22m
[2mdist/[22m[32mpng/login-illustration-9HORgnQJ.png                                                              [39m[1m[2m  235.70 kB[22m[1m[22m
[2mdist/[22m[32mpng/login-background-Dk6HcIGp.png                                                                [39m[1m[2m  760.38 kB[22m[1m[22m
[2mdist/[22m[35mcss/index-i0u-c3z8.css                                                                           [39m[1m[2m    0.04 kB[22m[1m[22m[2m │ gzip:   0.05 kB[22m
[2mdist/[22m[35mcss/DownloadFile-DAvaQD3e.css                                                                    [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-D6rOUVPI.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-CdH8jf3y.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/publish-log-drawer-s6nV0jMT.css                                                              [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-CnywH5a8.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/index-CTnqawDD.css                                                                           [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/ProjectForm-KVnI2Kwr.css                                                                     [39m[1m[2m    0.05 kB[22m[1m[22m[2m │ gzip:   0.07 kB[22m
[2mdist/[22m[35mcss/iot-placeholder-Bmzk0EMC.css                                                                 [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/metric-form-modal-Da1YSf-b.css                                                               [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/instance-item-drawer-DDYU_lFS.css                                                            [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/index-SkaSDKcc.css                                                                           [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/follow-DtSFEDAs.css                                                                          [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[35mcss/index-rU0VK28O.css                                                                           [39m[1m[2m    0.07 kB[22m[1m[22m[2m │ gzip:   0.09 kB[22m
[2mdist/[22m[35mcss/index-Bv8IxLq4.css                                                                           [39m[1m[2m    0.07 kB[22m[1m[22m[2m │ gzip:   0.09 kB[22m
[2mdist/[22m[35mcss/ImgList-B6haYOiW.css                                                                         [39m[1m[2m    0.07 kB[22m[1m[22m[2m │ gzip:   0.09 kB[22m
[2mdist/[22m[35mcss/apiclient-secret-modal-Dd9zq1y6.css                                                          [39m[1m[2m    0.08 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/index-DHSD_xKM.css                                                                           [39m[1m[2m    0.08 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/index-B-eu0trW.css                                                                           [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/notice-detail-modal-CfRr7vB7.css                                                             [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/apiclient-form-modal-BLXq3tyN.css                                                            [39m[1m[2m    0.10 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/table-SRAIJii3.css                                                                           [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-BYAU4CBC.css                                                                           [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-C2VZgmO1.css                                                                           [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-BGXi8PCn.css                                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[35mcss/index-YhouMItC.css                                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-kgmyyiGO.css                                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/notice-form-drawer-CXtGDo1N.css                                                              [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/publish-confirm-modal-CX1YV7gg.css                                                           [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-eeOT1m_L.css                                                [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/help-doc-form-drawer-BxBxRTJR.css                                                            [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-delete-DbyNiuDr.css                                         [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-DH5T8DG5.css                                                                           [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-CrZqPORT.css                                                                           [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/channel-detail-modal-CwLvL1Y3.css                                                            [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/import-CfDmKo2w.css                                                                          [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/BaseTable-B-tnrLUs.css                                                                       [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-Cpqpe2cV.css                                                                           [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-CM2MpC6G.css                                                                           [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-C-uHsV3_.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/index-DQWCfq80.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[35mcss/index-Czq7odZB.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/pie-GiX0BmiW.css                                                                             [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/gauge-BlkLKyTy.css                                                                           [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/keyword-match-modal-B7yCxDP5.css                                                             [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/category-BKKxPDYF.css                                                                        [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/gradient-ublseZuy.css                                                                        [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-basic-CmKoTgJy.css                                          [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/menu-operate-modal-CROAn4FB.css                                                              [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/module-form-BppZTgmd.css                                                                     [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-BCQNDPW4.css                                                                           [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/enterprise-employee-list-y_g8Pw_X.css                                                        [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/enterprise-operate-modal-BvDlyYS9.css                                                        [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/code-generator-table-config-form-insert-and-update-C-D2A4nJ.css                              [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/index-BRnrwGM4.css                                                                           [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/help-doc-manage-list-Db1ISVYa.css                                                            [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/analysis-CQAzIk2d.css                                                                        [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[35mcss/index-CnamIkRN.css                                                                           [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-V20uJDbO.css                                                                           [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-BQdngwWb.css                                                                           [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-Dp2Qu0a6.css                                                                           [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/TextListModal-lOMN9N1-.css                                                                   [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/goods-list-DOfkcpfl.css                                                                      [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/index-BKN7Vqi-.css                                                                           [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/index-BM5H80w0.css                                                                           [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/channel-form-modal-BtyQPGhT.css                                                              [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[35mcss/config-operate-drawer-CMQS0JWH.css                                                           [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/changelog-card-BIZYkilz.css                                                                  [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/template-send-detail-drawer-Iw0qJ7aB.css                                                     [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[35mcss/index-C0GpnHaV.css                                                                           [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/default-home-card-BOGSXst4.css                                                               [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-BnB06Whl.css                                                                           [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[35mcss/preview-DVqL3VoK.css                                                                         [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/chart-CXuTlDAW.css                                                                           [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/role-tree-menu-Dnqv_GrS.css                                                                  [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/index-yMfMkHJV.css                                                                           [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-DQHmgcs_.css                                                                           [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[35mcss/channel-credential-drawer-wEyPghzB.css                                                       [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/index-WxN6EUWJ.css                                                                           [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/index-Iei4Wp0G.css                                                                           [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/index-CCeTqU-9.css                                                                           [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[35mcss/snapshot-drawer-coJc1-CA.css                                                                 [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[35mcss/FormQrcode-zDibsgKU.css                                                                      [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-CEUAXvQU.css                                                                           [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[35mcss/help-doc-view-record-list-DXe94vIw.css                                                       [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[35mcss/panel-CzgXg4MK.css                                                                           [39m[1m[2m    0.35 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[35mcss/DataPush-BWg8P04L.css                                                                        [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/cross-DYHCe1z7.css                                                                           [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/help-doc-catalog-tree-5cONHUj1.css                                                           [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/notice-detail-CPlxUPVT.css                                                                   [39m[1m[2m    0.39 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/MatrixView-BQ8L2aDx.css                                                                      [39m[1m[2m    0.41 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/home-notice-B12SMd0n.css                                                                     [39m[1m[2m    0.41 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/index-BqHga3su.css                                                                           [39m[1m[2m    0.42 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-DCfFfRkA.css                                                                           [39m[1m[2m    0.42 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/dept-tree-checkbox-HxZiRqXx.css                                                              [39m[1m[2m    0.42 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[35mcss/layout-const-CZJ-siCP.css                                                                    [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-kaqxmmba.css                                                                           [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[35mcss/TextCopy-DuZngPZh.css                                                                        [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.26 kB[22m
[2mdist/[22m[35mcss/DataFilter-CGvhUU-b.css                                                                      [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[35mcss/template-send-create-modal-BpQLZrJw.css                                                      [39m[1m[2m    0.44 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/operate-log-detail-modal-D1pNfz_S.css                                                        [39m[1m[2m    0.46 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[35mcss/member-editor-CINJ5hXC.css                                                                   [39m[1m[2m    0.52 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[35mcss/index-C7jUeuFy.css                                                                           [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[35mcss/index-CTahEBHb.css                                                                           [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[35mcss/home-to-be-done-DHoODtdh.css                                                                 [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/index-Cfspf8pS.css                                                                           [39m[1m[2m    0.62 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[35mcss/iot-formula-editor-Bujj7HnT.css                                                              [39m[1m[2m    0.63 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/ViewData-BpDSevMt.css                                                                        [39m[1m[2m    0.65 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[35mcss/FileList-CWWNXZQD.css                                                                        [39m[1m[2m    0.65 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[35mcss/index-BENQLd1l.css                                                                           [39m[1m[2m    0.70 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/notice-employee-detail-CkXuZHTQ.css                                                          [39m[1m[2m    0.71 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[35mcss/public-B-i-NxjN.css                                                                          [39m[1m[2m    0.74 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[35mcss/home-header-BMGP0g5m.css                                                                     [39m[1m[2m    0.75 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[35mcss/display-factory-card-CnkDoY9p.css                                                            [39m[1m[2m    0.76 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[35mcss/display-slot-card-CEjwz2CP.css                                                               [39m[1m[2m    0.76 kB[22m[1m[22m[2m │ gzip:   0.33 kB[22m
[2mdist/[22m[35mcss/create-DvTvq8Xh.css                                                                          [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.36 kB[22m
[2mdist/[22m[35mcss/index-CVHl2D3B.css                                                                           [39m[1m[2m    0.79 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[35mcss/MatrixCard-BVqQUzzx.css                                                                      [39m[1m[2m    0.86 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[35mcss/index-By8oNveA.css                                                                           [39m[1m[2m    0.89 kB[22m[1m[22m[2m │ gzip:   0.42 kB[22m
[2mdist/[22m[35mcss/index-D5AwHx5d.css                                                                           [39m[1m[2m    0.89 kB[22m[1m[22m[2m │ gzip:   0.37 kB[22m
[2mdist/[22m[35mcss/index-BhWXYNfe.css                                                                           [39m[1m[2m    0.94 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[35mcss/index-WcB63oQd.css                                                                           [39m[1m[2m    1.01 kB[22m[1m[22m[2m │ gzip:   0.39 kB[22m
[2mdist/[22m[35mcss/index-E3TlBhR5.css                                                                           [39m[1m[2m    1.05 kB[22m[1m[22m[2m │ gzip:   0.47 kB[22m
[2mdist/[22m[35mcss/home-quick-entry-hZEhId5q.css                                                                [39m[1m[2m    1.05 kB[22m[1m[22m[2m │ gzip:   0.37 kB[22m
[2mdist/[22m[35mcss/ExamResultCard-DJSfDte_.css                                                                  [39m[1m[2m    1.12 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[35mcss/index-BphPVy-R.css                                                                           [39m[1m[2m    1.21 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[35mcss/index-FmDNBQz6.css                                                                           [39m[1m[2m    1.22 kB[22m[1m[22m[2m │ gzip:   0.50 kB[22m
[2mdist/[22m[35mcss/redpack-VZ0AqG97.css                                                                         [39m[1m[2m    1.25 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[35mcss/role-tree-checkbox-mC0FzCf8.css                                                              [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.42 kB[22m
[2mdist/[22m[35mcss/write-HdA4QHHZ.css                                                                           [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.48 kB[22m
[2mdist/[22m[35mcss/ViewOrUpdate-7fM-g4hB.css                                                                    [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.50 kB[22m
[2mdist/[22m[35mcss/index-DMJnr6jw.css                                                                           [39m[1m[2m    1.34 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[35mcss/submit-CVT9IhCQ.css                                                                          [39m[1m[2m    1.41 kB[22m[1m[22m[2m │ gzip:   0.49 kB[22m
[2mdist/[22m[35mcss/help-doc-layout-Cq-8T7YQ.css                                                                 [39m[1m[2m    1.43 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/metric-card-BQXE-Rnr.css                                                                     [39m[1m[2m    1.44 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[35mcss/index-ul1VSKkd.css                                                                           [39m[1m[2m    1.65 kB[22m[1m[22m[2m │ gzip:   0.54 kB[22m
[2mdist/[22m[35mcss/code-generator-preview-modal-Datvi8o-.css                                                    [39m[1m[2m    1.65 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[35mcss/index-BZNtYzVB.css                                                                           [39m[1m[2m    1.72 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/login-Bvf6hYTl.css                                                                           [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[35mcss/notify-B_8E5ukD.css                                                                          [39m[1m[2m    1.81 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[35mcss/card-CKfzlZHl.css                                                                            [39m[1m[2m    1.96 kB[22m[1m[22m[2m │ gzip:   0.60 kB[22m
[2mdist/[22m[35mcss/notice-form-visible-transfer-department-Bmq10wNm.css                                         [39m[1m[2m    2.07 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/notice-form-visible-transfer-employee-BI-wxgA3.css                                           [39m[1m[2m    2.09 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[35mcss/index-BzWu19bk.css                                                                           [39m[1m[2m    2.94 kB[22m[1m[22m[2m │ gzip:   0.88 kB[22m
[2mdist/[22m[35mcss/share-C55ToGJo.css                                                                           [39m[1m[2m    3.00 kB[22m[1m[22m[2m │ gzip:   0.84 kB[22m
[2mdist/[22m[35mcss/help-doc-user-view-BJvWW1qh.css                                                              [39m[1m[2m    3.79 kB[22m[1m[22m[2m │ gzip:   0.90 kB[22m
[2mdist/[22m[35mcss/index-CNnfYsqh.css                                                                           [39m[1m[2m    3.87 kB[22m[1m[22m[2m │ gzip:   0.83 kB[22m
[2mdist/[22m[35mcss/avatar-cropper-modal-9VFWDLF-.css                                                            [39m[1m[2m    4.05 kB[22m[1m[22m[2m │ gzip:   1.19 kB[22m
[2mdist/[22m[35mcss/index-a9zfQtlE.css                                                                           [39m[1m[2m    4.07 kB[22m[1m[22m[2m │ gzip:   0.86 kB[22m
[2mdist/[22m[35mcss/metric-trend-CJ1AF7NT.css                                                                    [39m[1m[2m    4.14 kB[22m[1m[22m[2m │ gzip:   0.83 kB[22m
[2mdist/[22m[35mcss/index-I1cZW9eG.css                                                                           [39m[1m[2m    4.38 kB[22m[1m[22m[2m │ gzip:   0.92 kB[22m
[2mdist/[22m[35mcss/index-9sHncdbb.css                                                                           [39m[1m[2m    4.40 kB[22m[1m[22m[2m │ gzip:   0.94 kB[22m
[2mdist/[22m[35mcss/index-CYd4SvCj.css                                                                           [39m[1m[2m    4.49 kB[22m[1m[22m[2m │ gzip:   1.26 kB[22m
[2mdist/[22m[35mcss/index-A-o_SUqN.css                                                                           [39m[1m[2m    4.53 kB[22m[1m[22m[2m │ gzip:   1.15 kB[22m
[2mdist/[22m[35mcss/enterprise-detail-CDC-O28W.css                                                               [39m[1m[2m   17.22 kB[22m[1m[22m[2m │ gzip:   2.61 kB[22m
[2mdist/[22m[35mcss/index-BA3svnqk.css                                                                           [39m[1m[2m   17.47 kB[22m[1m[22m[2m │ gzip:   3.26 kB[22m
[2mdist/[22m[35mcss/index-CL4e_1PC.css                                                                           [39m[1m[2m   22.01 kB[22m[1m[22m[2m │ gzip:   3.78 kB[22m
[2mdist/[22m[35mcss/index-BwsqDDIv.css                                                                           [39m[1m[2m   50.33 kB[22m[1m[22m[2m │ gzip:  10.36 kB[22m
[2mdist/[22m[35mcss/index-DNTQ0I4-.css                                                                           [39m[1m[2m  128.82 kB[22m[1m[22m[2m │ gzip:  19.08 kB[22m
[2mdist/[22m[35mcss/index-D6ofH3mU.css                                                                           [39m[1m[2m  172.81 kB[22m[1m[22m[2m │ gzip:  24.33 kB[22m
[2mdist/[22m[35mcss/style-BGjGDtq1.css                                                                           [39m[1m[2m  392.36 kB[22m[1m[22m[2m │ gzip:  69.45 kB[22m
[2mdist/[22m[36mjs/iot-optimistic-lock-CHOgIYgR.js                                                               [39m[1m[2m    0.04 kB[22m[1m[22m[2m │ gzip:   0.06 kB[22m
[2mdist/[22m[36mjs/403-CM8HbDZQ.js                                                                               [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[36mjs/regular-const-vBMlR054.js                                                                     [39m[1m[2m    0.06 kB[22m[1m[22m[2m │ gzip:   0.08 kB[22m
[2mdist/[22m[36mjs/_baseClamp-CdDz0ztW.js                                                                        [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[36mjs/join-Dox1eePW.js                                                                              [39m[1m[2m    0.09 kB[22m[1m[22m[2m │ gzip:   0.10 kB[22m
[2mdist/[22m[36mjs/date-util-B8PYxbSX.js                                                                         [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.12 kB[22m
[2mdist/[22m[36mjs/index-CJWUe81_.js                                                                             [39m[1m[2m    0.11 kB[22m[1m[22m[2m │ gzip:   0.11 kB[22m
[2mdist/[22m[36mjs/auth-iPP-ZAr7.js                                                                              [39m[1m[2m    0.12 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/index-CuLENaVm.js                                                                             [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/index-CbNndvFi.js                                                                             [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/iframe-index-DzQOtofJ.js                                                                      [39m[1m[2m    0.13 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/message-detail-DNakqQ33.js                                                                    [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/dept-tree-item-Cj33jJSu.js                                                                    [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/role-tree-point-BrCxq9L6.js                                                                   [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/change-log-modal-0Q3jQ6XE.js                                                                  [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/to-be-done-modal-bft497cN.js                                                                  [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/gateway-detail-modal-DWO9dUE0.js                                                              [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/device-component-zone-BF8w2EBm.js                                                             [39m[1m[2m    0.14 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/index-C7ctcRZX.js                                                                             [39m[1m[2m    0.15 kB[22m[1m[22m[2m │ gzip:   0.13 kB[22m
[2mdist/[22m[36mjs/index-BI0acfYY.js                                                                             [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/values-B8TsfwBN.js                                                                            [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.14 kB[22m
[2mdist/[22m[36mjs/dict-form-modal-PbWdHUpY.js                                                                   [39m[1m[2m    0.16 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/_baseMap-CSBdnQR5.js                                                                          [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/job-form-modal-CviBn1ru.js                                                                    [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/config-form-modal-CYiH8aTd.js                                                                 [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/menu-tree-select-DhLf-ULV.js                                                                  [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal-Di0Irv8J.js                                                              [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/fan-search-select-C0O_JuOD.js                                                                 [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/device-trace-zone-Bo9Mgmfj.js                                                                 [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/reload-result-list-DNMyaWEY.js                                                                [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/keyword-form-modal-D0Sygu1u.js                                                                [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/notice-employee-list-lpliS1XW.js                                                              [39m[1m[2m    0.17 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-operate-modal-Dp8EnyTI.js                                                     [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-operate-modal-bGWe2dcf.js                                                  [39m[1m[2m    0.18 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/serial-number-record-list-xBjCwIdZ.js                                                         [39m[1m[2m    0.19 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/job-form-modal-CUznZJpy.js                                                                    [39m[1m[2m    0.19 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/item-DynHGMGd.js                                                                              [39m[1m[2m    0.19 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/dict-form-modal-Ddl_6_PJ.js                                                                   [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-table-field-Co8PlKKR.js                                      [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/position-form-0r5c6JRB.js                                                                     [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.16 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal-C2SKLm9d.js                                                              [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/message-receiver-modal-C29c-I4J.js                                                            [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/do-reload-form-modal-SK3Vdi83.js                                                              [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/category-form-modal-Ds_Fy283.js                                                               [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/point-detail-modal-B7DZUxzT.js                                                                [39m[1m[2m    0.20 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/_isIterateeCall-D00er07Q.js                                                                   [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/metric-detail-modal-77Katips.js                                                               [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/device-status-zone-BZBqAoZP.js                                                                [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/device-realtime-zone-B5O0ecR_.js                                                              [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/instance-detail-modal-BUWuCtYP.js                                                             [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/startsWith-DtKVmyQR.js                                                                        [39m[1m[2m    0.21 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/useMessage-DHxeqVXi.js                                                                        [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.15 kB[22m
[2mdist/[22m[36mjs/mass-send-detail-drawer-CSWAAZXB.js                                                           [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/job-log-modal-COa08E7x.js                                                                     [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/file-api-DNj-82bR.js                                                                          [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/reload-api-Dlg48htj.js                                                                        [39m[1m[2m    0.22 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/serial-number-api-Cn4GwXjl.js                                                                 [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/notice-form-modal-CKU8W9-Y.js                                                                 [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/dict-import-modal-C61ijEPI.js                                                                 [39m[1m[2m    0.23 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/device-curve-zone-Bqg88s6v.js                                                                 [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/tenant-form-modal-D1g5ma_U.js                                                                 [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/tenant-admin-modal-WWrDCLTY.js                                                                [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/use-wechat-disabled-BPvWw22F.js                                                               [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/instance-params-modal-ThLnCWRU.js                                                             [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/mass-send-create-modal-B-mkgiEd.js                                                            [39m[1m[2m    0.24 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/gateway-api-B_SnvvtL.js                                                                       [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/item-point-bind-modal-Cx5miWHP.js                                                             [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/device-D6oEgpPt.js                                                                            [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/source-iILqLSRX.js                                                                            [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/serial-number-generate-form-modal-CBSZmIZ5.js                                                 [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/auth-DOxKROep.js                                                                              [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/index-Ctom-v-F.js                                                                             [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/situation-CX4JGsBC.js                                                                         [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/category-api-D2Oskxbl.js                                                                      [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/PositionMap-PDo_1UqW.js                                                                       [39m[1m[2m    0.25 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/index-YTD63IXe.js                                                                             [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/goods-form-modal-DP8-O_IJ.js                                                                  [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/index-CNuejBXu.js                                                                             [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/index-CGVa8WM9.js                                                                             [39m[1m[2m    0.26 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/change-log-form-BnzUzTJc.js                                                                   [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/dict-form-modal-Dyn8ymdV.js                                                                   [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/dict-data-modal-BFwQ12b6.js                                                                   [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/point-form-modal-DLIPwpfn.js                                                                  [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/iframe-index.vue_vue_type_script_setup_true_lang-CrZeU0za.js                                  [39m[1m[2m    0.27 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/channel-form-modal-B_KCse5j.js                                                                [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/menu-api-Dh3bUwIJ.js                                                                          [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/template-form-modal-kjheezY-.js                                                               [39m[1m[2m    0.28 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/str-util-DvlP_5eO.js                                                                          [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.17 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-tree-select-CMora3-_.js                                                      [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/position-api-DHjFA1ld.js                                                                      [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/item-formula-rollup-modal-B090HVyO.js                                                         [39m[1m[2m    0.29 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-query-field-k_YK-LrL.js                                      [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/channel-api-C1u5e5GP.js                                                                       [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/profile-api-LO01jzS5.js                                                                       [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/notice-view-record-list-CEMDyq9n.js                                                           [39m[1m[2m    0.30 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/file-api-B-1z_fMn.js                                                                          [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/category-tree-table-D20s3y2s.js                                                               [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/gateway-form-modal-Bu2DBWM2.js                                                                [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-api-DKVk_SOT.js                                                              [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.18 kB[22m
[2mdist/[22m[36mjs/change-log-api-BN1Gl062.js                                                                    [39m[1m[2m    0.31 kB[22m[1m[22m[2m │ gzip:   0.19 kB[22m
[2mdist/[22m[36mjs/dept-api-C1Ouz6DL.js                                                                          [39m[1m[2m    0.32 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/mitt-CNZ6avp8.js                                                                              [39m[1m[2m    0.32 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-modal-CoU1b3Rg.js                                                         [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/dict-data-modal-RIqNIsbL.js                                                                   [39m[1m[2m    0.33 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-field-2D9jKzVq.js                                            [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/message-send-form-gZEdYPbl.js                                                                 [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.20 kB[22m
[2mdist/[22m[36mjs/instance-form-modal-SQTAGHQt.js                                                               [39m[1m[2m    0.34 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/instance-api-AzPBxriN.js                                                                      [39m[1m[2m    0.36 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/module-api-Oe_V0NWJ.js                                                                        [39m[1m[2m    0.36 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/tag-filter-IviZpaC6.js                                                                        [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/useFormSetting-CP__zG4H.js                                                                    [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/index-b3r0-dnC.js                                                                             [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/publish-api-CyGVZoJL.js                                                                       [39m[1m[2m    0.37 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/template-item-modal-_-9GYDhS.js                                                               [39m[1m[2m    0.38 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/wechat-disabled-alert-Bkn74i_5.js                                                             [39m[1m[2m    0.38 kB[22m[1m[22m[2m │ gzip:   0.41 kB[22m
[2mdist/[22m[36mjs/dept-form-modal-BZeVg9k1.js                                                                   [39m[1m[2m    0.38 kB[22m[1m[22m[2m │ gzip:   0.21 kB[22m
[2mdist/[22m[36mjs/404-CD-rVit-.js                                                                               [39m[1m[2m    0.39 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[36mjs/tenant-api-D6R5DA7Z.js                                                                        [39m[1m[2m    0.41 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/table-id-const-BsJohLR_.js                                                                    [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/job-api-CUDBPCUl.js                                                                           [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.22 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-form-modal-CogxyZ2T.js                                                       [39m[1m[2m    0.43 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/use-display-tree-CiiHing9.js                                                                  [39m[1m[2m    0.44 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[36mjs/metric-api-u38VBsWn.js                                                                        [39m[1m[2m    0.45 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/_baseUniq-VJ-RQ_7O.js                                                                         [39m[1m[2m    0.45 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[36mjs/job-api-DEZIiy1K.js                                                                           [39m[1m[2m    0.46 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/config-list-Dz3w9s3g.js                                                                       [39m[1m[2m    0.47 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/theme-api-pd8vnxG1.js                                                                         [39m[1m[2m    0.48 kB[22m[1m[22m[2m │ gzip:   0.23 kB[22m
[2mdist/[22m[36mjs/code-generator-api-D7tGV4X9.js                                                                [39m[1m[2m    0.48 kB[22m[1m[22m[2m │ gzip:   0.24 kB[22m
[2mdist/[22m[36mjs/iot-placeholder-ju_Ti2du.js                                                                   [39m[1m[2m    0.49 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[36mjs/goods-catalog-DMOLpuGt.js                                                                     [39m[1m[2m    0.50 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[36mjs/job-log-list-modal-DKwTn87C.js                                                                [39m[1m[2m    0.50 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/user-api-CDYub9yQ.js                                                                          [39m[1m[2m    0.51 kB[22m[1m[22m[2m │ gzip:   0.28 kB[22m
[2mdist/[22m[36mjs/table-auto-height-Cwt7a1Fg.js                                                                 [39m[1m[2m    0.51 kB[22m[1m[22m[2m │ gzip:   0.34 kB[22m
[2mdist/[22m[36mjs/common-BzOvJVeF.js                                                                            [39m[1m[2m    0.51 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[36mjs/custom-catalog-DEqDUCHh.js                                                                    [39m[1m[2m    0.52 kB[22m[1m[22m[2m │ gzip:   0.33 kB[22m
[2mdist/[22m[36mjs/dict-api-Rw0RFa54.js                                                                          [39m[1m[2m    0.52 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[36mjs/auth.vue_vue_type_script_setup_true_lang-hXJkGp0b.js                                          [39m[1m[2m    0.53 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[36mjs/index-DQoJ95g_.js                                                                             [39m[1m[2m    0.55 kB[22m[1m[22m[2m │ gzip:   0.33 kB[22m
[2mdist/[22m[36mjs/dict-api-CR2LKk9m.js                                                                          [39m[1m[2m    0.55 kB[22m[1m[22m[2m │ gzip:   0.25 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-list-CvMPGhk9.js                                                              [39m[1m[2m    0.56 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-list-Bc3eKM_j.js                                                           [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[36mjs/index-DYsTR_u2.js                                                                             [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.32 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-BUK_zKvl.js                                         [39m[1m[2m    0.57 kB[22m[1m[22m[2m │ gzip:   0.42 kB[22m
[2mdist/[22m[36mjs/deleted-job-list-BXIvCSkJ.js                                                                  [39m[1m[2m    0.58 kB[22m[1m[22m[2m │ gzip:   0.31 kB[22m
[2mdist/[22m[36mjs/help-doc-api-BXT-pb3w.js                                                                      [39m[1m[2m    0.58 kB[22m[1m[22m[2m │ gzip:   0.26 kB[22m
[2mdist/[22m[36mjs/ProjectForm-TNYuusdx.js                                                                       [39m[1m[2m    0.58 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[36mjs/role-tree-point.vue_vue_type_script_setup_true_lang-BjD09PSp.js                               [39m[1m[2m    0.59 kB[22m[1m[22m[2m │ gzip:   0.39 kB[22m
[2mdist/[22m[36mjs/item-api-dU-CB0Uo.js                                                                          [39m[1m[2m    0.59 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[36mjs/send-common-DdiR-N0d.js                                                                       [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.37 kB[22m
[2mdist/[22m[36mjs/default-time-ranges-CnKX8bss.js                                                               [39m[1m[2m    0.60 kB[22m[1m[22m[2m │ gzip:   0.26 kB[22m
[2mdist/[22m[36mjs/FormQrcode-D0DzYmRO.js                                                                        [39m[1m[2m    0.61 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[36mjs/template-api-BRz4FyRL.js                                                                      [39m[1m[2m    0.62 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/data-api-Bk_vpfyO.js                                                                          [39m[1m[2m    0.62 kB[22m[1m[22m[2m │ gzip:   0.36 kB[22m
[2mdist/[22m[36mjs/report-api-CWnk2al6.js                                                                        [39m[1m[2m    0.65 kB[22m[1m[22m[2m │ gzip:   0.27 kB[22m
[2mdist/[22m[36mjs/enterprise-api-BVl3Zjif.js                                                                    [39m[1m[2m    0.67 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/template-api-CAH2mKee.js                                                                      [39m[1m[2m    0.68 kB[22m[1m[22m[2m │ gzip:   0.30 kB[22m
[2mdist/[22m[36mjs/FileList-Wd6SVVJw.js                                                                          [39m[1m[2m    0.70 kB[22m[1m[22m[2m │ gzip:   0.44 kB[22m
[2mdist/[22m[36mjs/notice-api-D2ld0YKO.js                                                                        [39m[1m[2m    0.71 kB[22m[1m[22m[2m │ gzip:   0.29 kB[22m
[2mdist/[22m[36mjs/index-DehaA6kI.js                                                                             [39m[1m[2m    0.73 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[36mjs/change-log-modal.vue_vue_type_script_setup_true_lang-nKDeRC8V.js                              [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.51 kB[22m
[2mdist/[22m[36mjs/report-format-vflwHJHL.js                                                                     [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.48 kB[22m
[2mdist/[22m[36mjs/ImgList-DEfcerZL.js                                                                           [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/index-DGFm7LsM.js                                                                             [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.34 kB[22m
[2mdist/[22m[36mjs/point-api-BNbMLzbA.js                                                                         [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.39 kB[22m
[2mdist/[22m[36mjs/file-url-kWpq9kM8.js                                                                          [39m[1m[2m    0.77 kB[22m[1m[22m[2m │ gzip:   0.47 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-DK7NAA8c.js                                         [39m[1m[2m    0.78 kB[22m[1m[22m[2m │ gzip:   0.54 kB[22m
[2mdist/[22m[36mjs/help-doc-list-CKa_dzi2.js                                                                     [39m[1m[2m    0.83 kB[22m[1m[22m[2m │ gzip:   0.40 kB[22m
[2mdist/[22m[36mjs/_charsStartIndex-BoDFCEzm.js                                                                  [39m[1m[2m    0.84 kB[22m[1m[22m[2m │ gzip:   0.46 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-_qy0uNlq.js                                         [39m[1m[2m    0.84 kB[22m[1m[22m[2m │ gzip:   0.48 kB[22m
[2mdist/[22m[36mjs/index-8ETk_9jw.js                                                                             [39m[1m[2m    0.84 kB[22m[1m[22m[2m │ gzip:   0.54 kB[22m
[2mdist/[22m[36mjs/role-tree-checkbox-CjcMcxRe.js                                                                [39m[1m[2m    0.85 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/project-list-columns-BqIdc9YB.js                                                              [39m[1m[2m    0.87 kB[22m[1m[22m[2m │ gzip:   0.38 kB[22m
[2mdist/[22m[36mjs/MinusOutlined-CLN5uC84.js                                                                     [39m[1m[2m    0.88 kB[22m[1m[22m[2m │ gzip:   0.55 kB[22m
[2mdist/[22m[36mjs/CaretDownOutlined-B1oXPNy0.js                                                                 [39m[1m[2m    0.91 kB[22m[1m[22m[2m │ gzip:   0.58 kB[22m
[2mdist/[22m[36mjs/role-api-5adxo-QX.js                                                                          [39m[1m[2m    0.91 kB[22m[1m[22m[2m │ gzip:   0.35 kB[22m
[2mdist/[22m[36mjs/CaretRightOutlined-CFAv6NrQ.js                                                                [39m[1m[2m    0.91 kB[22m[1m[22m[2m │ gzip:   0.58 kB[22m
[2mdist/[22m[36mjs/setting-api-9qL44j2S.js                                                                       [39m[1m[2m    0.92 kB[22m[1m[22m[2m │ gzip:   0.47 kB[22m
[2mdist/[22m[36mjs/index-Ia9UdTVg.js                                                                             [39m[1m[2m    0.92 kB[22m[1m[22m[2m │ gzip:   0.43 kB[22m
[2mdist/[22m[36mjs/form-api-Bx6ZLt00.js                                                                          [39m[1m[2m    0.94 kB[22m[1m[22m[2m │ gzip:   0.39 kB[22m
[2mdist/[22m[36mjs/menu-tree-select.vue_vue_type_script_setup_true_lang-CRVVeK9w.js                              [39m[1m[2m    0.95 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/dept-tree-item.vue_vue_type_script_setup_true_lang-BSBrDF3P.js                                [39m[1m[2m    0.96 kB[22m[1m[22m[2m │ gzip:   0.57 kB[22m
[2mdist/[22m[36mjs/MobileOutlined-x_HBJb86.js                                                                    [39m[1m[2m    0.96 kB[22m[1m[22m[2m │ gzip:   0.60 kB[22m
[2mdist/[22m[36mjs/LeftOutlined-BMs0CgRM.js                                                                      [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/RightOutlined-B0PmcOcG.js                                                                     [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/DownOutlined-lm4KfxXF.js                                                                      [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/DownloadFile-BUQstNj2.js                                                                      [39m[1m[2m    0.97 kB[22m[1m[22m[2m │ gzip:   0.71 kB[22m
[2mdist/[22m[36mjs/pie-FSeSKa6q.js                                                                               [39m[1m[2m    0.98 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/ArrowLeftOutlined-gkMLKAi1.js                                                                 [39m[1m[2m    1.01 kB[22m[1m[22m[2m │ gzip:   0.65 kB[22m
[2mdist/[22m[36mjs/TextCopy-DqODbpcr.js                                                                          [39m[1m[2m    1.04 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/device.vue_vue_type_script_setup_true_lang-C0ZnSvuJ.js                                        [39m[1m[2m    1.07 kB[22m[1m[22m[2m │ gzip:   0.69 kB[22m
[2mdist/[22m[36mjs/UploadOutlined-LyrQV8nH.js                                                                    [39m[1m[2m    1.09 kB[22m[1m[22m[2m │ gzip:   0.67 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-Dj1uFG5Q.js                                         [39m[1m[2m    1.10 kB[22m[1m[22m[2m │ gzip:   0.65 kB[22m
[2mdist/[22m[36mjs/sortBy-BR8y3uiY.js                                                                            [39m[1m[2m    1.10 kB[22m[1m[22m[2m │ gzip:   0.60 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-Bz0O4cM0.js                                         [39m[1m[2m    1.10 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/redpack-helper-BxpknW9c.js                                                                    [39m[1m[2m    1.11 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/default-home-card-D2Gnxy7M.js                                                                 [39m[1m[2m    1.12 kB[22m[1m[22m[2m │ gzip:   0.63 kB[22m
[2mdist/[22m[36mjs/menu-data-handler-Bi92265j.js                                                                 [39m[1m[2m    1.13 kB[22m[1m[22m[2m │ gzip:   0.53 kB[22m
[2mdist/[22m[36mjs/display-slot-card-Dqr7BERC.js                                                                 [39m[1m[2m    1.15 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/CheckSquareOutlined-BvNJt3FP.js                                                               [39m[1m[2m    1.15 kB[22m[1m[22m[2m │ gzip:   0.71 kB[22m
[2mdist/[22m[36mjs/CopyOutlined-DIPqJ-VV.js                                                                      [39m[1m[2m    1.16 kB[22m[1m[22m[2m │ gzip:   0.72 kB[22m
[2mdist/[22m[36mjs/to-be-done-modal.vue_vue_type_script_setup_true_lang-CXKx7clG.js                              [39m[1m[2m    1.17 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/gauge-CO-5qq8N.js                                                                             [39m[1m[2m    1.18 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/preview-eClJBtry.js                                                                           [39m[1m[2m    1.18 kB[22m[1m[22m[2m │ gzip:   0.74 kB[22m
[2mdist/[22m[36mjs/EditOutlined-CGK0T3h_.js                                                                      [39m[1m[2m    1.19 kB[22m[1m[22m[2m │ gzip:   0.74 kB[22m
[2mdist/[22m[36mjs/PictureOutlined-CSgLbA5P.js                                                                   [39m[1m[2m    1.19 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/BellOutlined-CMEaLFQO.js                                                                      [39m[1m[2m    1.22 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/dept-tree-checkbox-BIOT8CDI.js                                                                [39m[1m[2m    1.23 kB[22m[1m[22m[2m │ gzip:   0.66 kB[22m
[2mdist/[22m[36mjs/category-To1k4_b2.js                                                                          [39m[1m[2m    1.23 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/ApartmentOutlined-RC7x9Ute.js                                                                 [39m[1m[2m    1.24 kB[22m[1m[22m[2m │ gzip:   0.68 kB[22m
[2mdist/[22m[36mjs/message-detail.vue_vue_type_script_setup_true_lang-rK590LSQ.js                                [39m[1m[2m    1.24 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/fan-search-select.vue_vue_type_script_setup_true_lang-CBbYaMK4.js                             [39m[1m[2m    1.25 kB[22m[1m[22m[2m │ gzip:   0.81 kB[22m
[2mdist/[22m[36mjs/BarChartOutlined-CXRKEQlH.js                                                                  [39m[1m[2m    1.26 kB[22m[1m[22m[2m │ gzip:   0.62 kB[22m
[2mdist/[22m[36mjs/index-DekLGu2l.js                                                                             [39m[1m[2m    1.27 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-tree-select.vue_vue_type_script_setup_true_lang-DJ_bOkbx.js                  [39m[1m[2m    1.27 kB[22m[1m[22m[2m │ gzip:   0.76 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-D-8bwAH8.js                                         [39m[1m[2m    1.28 kB[22m[1m[22m[2m │ gzip:   0.72 kB[22m
[2mdist/[22m[36mjs/situation.vue_vue_type_script_setup_true_lang-BncF_egx.js                                     [39m[1m[2m    1.28 kB[22m[1m[22m[2m │ gzip:   0.82 kB[22m
[2mdist/[22m[36mjs/notice-detail-modal-B5mgGoW0.js                                                               [39m[1m[2m    1.29 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/index-DBTtW5LN.js                                                                             [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/AppstoreOutlined-CqmcY-BV.js                                                                  [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.61 kB[22m
[2mdist/[22m[36mjs/index-DSzFaqMp.js                                                                             [39m[1m[2m    1.30 kB[22m[1m[22m[2m │ gzip:   0.85 kB[22m
[2mdist/[22m[36mjs/home-notice-fvVXnfZM.js                                                                       [39m[1m[2m    1.31 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/mass-send-helper-DyuVIUAK.js                                                                  [39m[1m[2m    1.31 kB[22m[1m[22m[2m │ gzip:   0.83 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-modal.vue_vue_type_script_setup_true_lang-DfRDVxS8.js                     [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.70 kB[22m
[2mdist/[22m[36mjs/InboxOutlined-N1bJDrNa.js                                                                     [39m[1m[2m    1.32 kB[22m[1m[22m[2m │ gzip:   0.82 kB[22m
[2mdist/[22m[36mjs/useProject-5bJFSaTo.js                                                                        [39m[1m[2m    1.33 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-BBGUepNt.js                                         [39m[1m[2m    1.34 kB[22m[1m[22m[2m │ gzip:   0.78 kB[22m
[2mdist/[22m[36mjs/changelog-card-BzuVN2bq.js                                                                    [39m[1m[2m    1.36 kB[22m[1m[22m[2m │ gzip:   0.92 kB[22m
[2mdist/[22m[36mjs/source.vue_vue_type_script_setup_true_lang-3cMYQ3Ss.js                                        [39m[1m[2m    1.38 kB[22m[1m[22m[2m │ gzip:   0.84 kB[22m
[2mdist/[22m[36mjs/wechat-api-D2-s3RkH.js                                                                        [39m[1m[2m    1.40 kB[22m[1m[22m[2m │ gzip:   0.57 kB[22m
[2mdist/[22m[36mjs/item.vue_vue_type_script_setup_true_lang-DeWEECCS.js                                          [39m[1m[2m    1.40 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-D005DJO1.js                                         [39m[1m[2m    1.43 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/ReloadOutlined-DHDbDDSl.js                                                                    [39m[1m[2m    1.48 kB[22m[1m[22m[2m │ gzip:   0.87 kB[22m
[2mdist/[22m[36mjs/reload-result-list.vue_vue_type_script_setup_true_lang-IqlWQ1yH.js                            [39m[1m[2m    1.50 kB[22m[1m[22m[2m │ gzip:   0.94 kB[22m
[2mdist/[22m[36mjs/SyncOutlined-D-mGM12v.js                                                                      [39m[1m[2m    1.51 kB[22m[1m[22m[2m │ gzip:   0.91 kB[22m
[2mdist/[22m[36mjs/home-quick-entry-modal-CsWhLB4h.js                                                            [39m[1m[2m    1.51 kB[22m[1m[22m[2m │ gzip:   0.65 kB[22m
[2mdist/[22m[36mjs/apiclient-secret-modal-Mn8iDW0d.js                                                            [39m[1m[2m    1.53 kB[22m[1m[22m[2m │ gzip:   1.03 kB[22m
[2mdist/[22m[36mjs/iot-consts-BURryymr.js                                                                        [39m[1m[2m    1.54 kB[22m[1m[22m[2m │ gzip:   1.03 kB[22m
[2mdist/[22m[36mjs/MatrixView-DcVZqrph.js                                                                        [39m[1m[2m    1.54 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/index-DBbqWBBh.js                                                                             [39m[1m[2m    1.55 kB[22m[1m[22m[2m │ gzip:   0.73 kB[22m
[2mdist/[22m[36mjs/FullscreenOutlined-CwTuECgF.js                                                                [39m[1m[2m    1.59 kB[22m[1m[22m[2m │ gzip:   0.86 kB[22m
[2mdist/[22m[36mjs/do-reload-form-modal.vue_vue_type_script_setup_true_lang-Dbr9eH1i.js                          [39m[1m[2m    1.60 kB[22m[1m[22m[2m │ gzip:   0.93 kB[22m
[2mdist/[22m[36mjs/index-P2r7Ss2p.js                                                                             [39m[1m[2m    1.62 kB[22m[1m[22m[2m │ gzip:   0.86 kB[22m
[2mdist/[22m[36mjs/UrlOperate-DHgvZ7fy.js                                                                        [39m[1m[2m    1.63 kB[22m[1m[22m[2m │ gzip:   0.89 kB[22m
[2mdist/[22m[36mjs/CloudUploadOutlined-DuI77sRw.js                                                               [39m[1m[2m    1.64 kB[22m[1m[22m[2m │ gzip:   0.98 kB[22m
[2mdist/[22m[36mjs/UserAddOutlined-4PsWVIC2.js                                                                   [39m[1m[2m    1.66 kB[22m[1m[22m[2m │ gzip:   0.95 kB[22m
[2mdist/[22m[36mjs/index-CgJIVX_n.js                                                                             [39m[1m[2m    1.69 kB[22m[1m[22m[2m │ gzip:   0.92 kB[22m
[2mdist/[22m[36mjs/metric-utils-CS9quiXT.js                                                                      [39m[1m[2m    1.69 kB[22m[1m[22m[2m │ gzip:   1.14 kB[22m
[2mdist/[22m[36mjs/help-doc-manage-list-6DjuEgpt.js                                                              [39m[1m[2m    1.71 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/member-editor-CUWEkMhT.js                                                                     [39m[1m[2m    1.71 kB[22m[1m[22m[2m │ gzip:   0.79 kB[22m
[2mdist/[22m[36mjs/index-CA_Jk55W.js                                                                             [39m[1m[2m    1.72 kB[22m[1m[22m[2m │ gzip:   0.81 kB[22m
[2mdist/[22m[36mjs/category-form-modal.vue_vue_type_script_setup_true_lang-C1yQrWFG.js                           [39m[1m[2m    1.73 kB[22m[1m[22m[2m │ gzip:   1.07 kB[22m
[2mdist/[22m[36mjs/QrcodeOutlined-u44_7bit.js                                                                    [39m[1m[2m    1.77 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/ExamResultCard-0aHaabcY.js                                                                    [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   1.04 kB[22m
[2mdist/[22m[36mjs/index-u2pIi-we.js                                                                             [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   0.97 kB[22m
[2mdist/[22m[36mjs/home-quick-entry-modal.vue_vue_type_script_setup_true_lang-XLNHXNHH.js                        [39m[1m[2m    1.78 kB[22m[1m[22m[2m │ gzip:   1.03 kB[22m
[2mdist/[22m[36mjs/index-DDJs1hKm.js                                                                             [39m[1m[2m    1.79 kB[22m[1m[22m[2m │ gzip:   1.21 kB[22m
[2mdist/[22m[36mjs/dict-form-modal.vue_vue_type_script_setup_true_lang-Bw7m5MMB.js                               [39m[1m[2m    1.79 kB[22m[1m[22m[2m │ gzip:   1.05 kB[22m
[2mdist/[22m[36mjs/channel-detail-modal-BAMy2Xp-.js                                                              [39m[1m[2m    1.81 kB[22m[1m[22m[2m │ gzip:   0.95 kB[22m
[2mdist/[22m[36mjs/index-DKzdoUGt.js                                                                             [39m[1m[2m    1.82 kB[22m[1m[22m[2m │ gzip:   1.01 kB[22m
[2mdist/[22m[36mjs/module-list-Bkv1ituF.js                                                                       [39m[1m[2m    1.82 kB[22m[1m[22m[2m │ gzip:   0.75 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-transfer-department-nsWxMfDl.js                                           [39m[1m[2m    1.89 kB[22m[1m[22m[2m │ gzip:   1.02 kB[22m
[2mdist/[22m[36mjs/helper-Dq1QDmcg.js                                                                            [39m[1m[2m    1.89 kB[22m[1m[22m[2m │ gzip:   1.06 kB[22m
[2mdist/[22m[36mjs/data-masking-list-B1hW0H4Y.js                                                                 [39m[1m[2m    1.90 kB[22m[1m[22m[2m │ gzip:   1.32 kB[22m
[2mdist/[22m[36mjs/template-send-helper-Bb8l-3Ym.js                                                              [39m[1m[2m    1.90 kB[22m[1m[22m[2m │ gzip:   1.06 kB[22m
[2mdist/[22m[36mjs/gateway-detail-modal.vue_vue_type_script_setup_true_lang-D4tAeUXs.js                          [39m[1m[2m    1.92 kB[22m[1m[22m[2m │ gzip:   0.82 kB[22m
[2mdist/[22m[36mjs/serial-number-record-list.vue_vue_type_script_setup_true_lang-CoB-gTiM.js                     [39m[1m[2m    1.92 kB[22m[1m[22m[2m │ gzip:   1.12 kB[22m
[2mdist/[22m[36mjs/display-factory-card-DkEINQOO.js                                                              [39m[1m[2m    1.92 kB[22m[1m[22m[2m │ gzip:   0.98 kB[22m
[2mdist/[22m[36mjs/index-bYsqgyUx.js                                                                             [39m[1m[2m    1.99 kB[22m[1m[22m[2m │ gzip:   1.10 kB[22m
[2mdist/[22m[36mjs/BaseTable-DukWN8Jj.js                                                                         [39m[1m[2m    2.04 kB[22m[1m[22m[2m │ gzip:   1.07 kB[22m
[2mdist/[22m[36mjs/metric-card-aBZrj2Wz.js                                                                       [39m[1m[2m    2.04 kB[22m[1m[22m[2m │ gzip:   1.02 kB[22m
[2mdist/[22m[36mjs/cache-list-BAECANLU.js                                                                        [39m[1m[2m    2.05 kB[22m[1m[22m[2m │ gzip:   1.41 kB[22m
[2mdist/[22m[36mjs/PositionMap.vue_vue_type_script_setup_true_lang-B6xM89rG.js                                   [39m[1m[2m    2.05 kB[22m[1m[22m[2m │ gzip:   1.27 kB[22m
[2mdist/[22m[36mjs/menu-list-DLNOXS1x.js                                                                         [39m[1m[2m    2.06 kB[22m[1m[22m[2m │ gzip:   0.83 kB[22m
[2mdist/[22m[36mjs/item-point-bind-modal.vue_vue_type_script_setup_true_lang-B3auXi0V.js                         [39m[1m[2m    2.07 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/dict-form-modal.vue_vue_type_script_setup_true_lang-CNbqMrux.js                               [39m[1m[2m    2.07 kB[22m[1m[22m[2m │ gzip:   1.14 kB[22m
[2mdist/[22m[36mjs/index-Byo6frjB.js                                                                             [39m[1m[2m    2.09 kB[22m[1m[22m[2m │ gzip:   1.22 kB[22m
[2mdist/[22m[36mjs/index-DqVAMzue.js                                                                             [39m[1m[2m    2.12 kB[22m[1m[22m[2m │ gzip:   1.60 kB[22m
[2mdist/[22m[36mjs/account-menu-DQldE4_C.js                                                                      [39m[1m[2m    2.14 kB[22m[1m[22m[2m │ gzip:   0.98 kB[22m
[2mdist/[22m[36mjs/instance-detail-modal.vue_vue_type_script_setup_true_lang-CKbLLhRb.js                         [39m[1m[2m    2.15 kB[22m[1m[22m[2m │ gzip:   0.99 kB[22m
[2mdist/[22m[36mjs/FilterOutlined-CGZuhKhw.js                                                                    [39m[1m[2m    2.17 kB[22m[1m[22m[2m │ gzip:   0.95 kB[22m
[2mdist/[22m[36mjs/serial-number-generate-form-modal.vue_vue_type_script_setup_true_lang-COazO6Fb.js             [39m[1m[2m    2.19 kB[22m[1m[22m[2m │ gzip:   1.10 kB[22m
[2mdist/[22m[36mjs/tenant-admin-modal.vue_vue_type_script_setup_true_lang-DBC2SNdl.js                            [39m[1m[2m    2.19 kB[22m[1m[22m[2m │ gzip:   1.29 kB[22m
[2mdist/[22m[36mjs/notice-employee-detail-528iM0VF.js                                                            [39m[1m[2m    2.20 kB[22m[1m[22m[2m │ gzip:   1.25 kB[22m
[2mdist/[22m[36mjs/index-dRWUZI-c.js                                                                             [39m[1m[2m    2.20 kB[22m[1m[22m[2m │ gzip:   1.25 kB[22m
[2mdist/[22m[36mjs/metric-detail-modal.vue_vue_type_script_setup_true_lang-BcEwuYQa.js                           [39m[1m[2m    2.21 kB[22m[1m[22m[2m │ gzip:   0.99 kB[22m
[2mdist/[22m[36mjs/help-doc-user-view-miglJ8E1.js                                                                [39m[1m[2m    2.21 kB[22m[1m[22m[2m │ gzip:   1.19 kB[22m
[2mdist/[22m[36mjs/follow-C7kLrOJo.js                                                                            [39m[1m[2m    2.26 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/index-CbPYLXOQ.js                                                                             [39m[1m[2m    2.28 kB[22m[1m[22m[2m │ gzip:   1.28 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-form-modal.vue_vue_type_script_setup_true_lang-Dl5-iD4P.js                   [39m[1m[2m    2.28 kB[22m[1m[22m[2m │ gzip:   1.25 kB[22m
[2mdist/[22m[36mjs/TextListModal-DmPugZRX.js                                                                     [39m[1m[2m    2.28 kB[22m[1m[22m[2m │ gzip:   1.25 kB[22m
[2mdist/[22m[36mjs/config-form-modal.vue_vue_type_script_setup_true_lang-Cdz-SJbT.js                             [39m[1m[2m    2.29 kB[22m[1m[22m[2m │ gzip:   1.18 kB[22m
[2mdist/[22m[36mjs/import-CNwqGlSJ.js                                                                            [39m[1m[2m    2.29 kB[22m[1m[22m[2m │ gzip:   1.56 kB[22m
[2mdist/[22m[36mjs/gradient-30fJeKVk.js                                                                          [39m[1m[2m    2.32 kB[22m[1m[22m[2m │ gzip:   1.06 kB[22m
[2mdist/[22m[36mjs/chart-SuQQgcTC.js                                                                             [39m[1m[2m    2.37 kB[22m[1m[22m[2m │ gzip:   1.11 kB[22m
[2mdist/[22m[36mjs/device-component-zone.vue_vue_type_script_setup_true_lang-C3TBmYw2.js                         [39m[1m[2m    2.42 kB[22m[1m[22m[2m │ gzip:   1.33 kB[22m
[2mdist/[22m[36mjs/index-BR9PJ8xA.js                                                                             [39m[1m[2m    2.45 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/position-form.vue_vue_type_script_setup_true_lang-DEJK2gKm.js                                 [39m[1m[2m    2.48 kB[22m[1m[22m[2m │ gzip:   1.28 kB[22m
[2mdist/[22m[36mjs/device-realtime-zone.vue_vue_type_script_setup_true_lang-C6VHeyAz.js                          [39m[1m[2m    2.50 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/index-DdziUg7-.js                                                                             [39m[1m[2m    2.51 kB[22m[1m[22m[2m │ gzip:   1.21 kB[22m
[2mdist/[22m[36mjs/notice-form-visible-transfer-employee-Bckx3Vs2.js                                             [39m[1m[2m    2.51 kB[22m[1m[22m[2m │ gzip:   1.31 kB[22m
[2mdist/[22m[36mjs/display-api-D9oCki-Y.js                                                                       [39m[1m[2m    2.54 kB[22m[1m[22m[2m │ gzip:   1.01 kB[22m
[2mdist/[22m[36mjs/keyword-match-modal-DMQJOHzB.js                                                               [39m[1m[2m    2.54 kB[22m[1m[22m[2m │ gzip:   1.39 kB[22m
[2mdist/[22m[36mjs/index-N_C_If_j.js                                                                             [39m[1m[2m    2.58 kB[22m[1m[22m[2m │ gzip:   1.12 kB[22m
[2mdist/[22m[36mjs/SettingOutlined-CgWp0iJh.js                                                                   [39m[1m[2m    2.60 kB[22m[1m[22m[2m │ gzip:   1.31 kB[22m
[2mdist/[22m[36mjs/index-BX20uOG-.js                                                                             [39m[1m[2m    2.63 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/index-BQxFF000.js                                                                             [39m[1m[2m    2.63 kB[22m[1m[22m[2m │ gzip:   1.56 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-table-field.vue_vue_type_script_setup_true_lang-3VrAUtBj.js  [39m[1m[2m    2.63 kB[22m[1m[22m[2m │ gzip:   1.22 kB[22m
[2mdist/[22m[36mjs/index-CgkiOuPS.js                                                                             [39m[1m[2m    2.65 kB[22m[1m[22m[2m │ gzip:   1.27 kB[22m
[2mdist/[22m[36mjs/item-formula-rollup-modal.vue_vue_type_script_setup_true_lang-CODUnhfQ.js                     [39m[1m[2m    2.66 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/device-status-zone.vue_vue_type_script_setup_true_lang-zCpAF9oj.js                            [39m[1m[2m    2.67 kB[22m[1m[22m[2m │ gzip:   1.30 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal.vue_vue_type_script_setup_true_lang-BD887cT3.js                          [39m[1m[2m    2.69 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/message-receiver-modal.vue_vue_type_script_setup_true_lang-D26D7JLD.js                        [39m[1m[2m    2.70 kB[22m[1m[22m[2m │ gzip:   1.35 kB[22m
[2mdist/[22m[36mjs/reload-list-Bc_vvMTY.js                                                                       [39m[1m[2m    2.73 kB[22m[1m[22m[2m │ gzip:   1.78 kB[22m
[2mdist/[22m[36mjs/device-utils-6k4pcSn7.js                                                                      [39m[1m[2m    2.77 kB[22m[1m[22m[2m │ gzip:   1.48 kB[22m
[2mdist/[22m[36mjs/dept-form-modal.vue_vue_type_script_setup_true_lang-C16AKMVo.js                               [39m[1m[2m    2.77 kB[22m[1m[22m[2m │ gzip:   1.46 kB[22m
[2mdist/[22m[36mjs/category-tree-table.vue_vue_type_script_setup_true_lang-IOO-WUgg.js                           [39m[1m[2m    2.79 kB[22m[1m[22m[2m │ gzip:   1.43 kB[22m
[2mdist/[22m[36mjs/job-form-modal.vue_vue_type_script_setup_true_lang-BEkVKg0C.js                                [39m[1m[2m    2.79 kB[22m[1m[22m[2m │ gzip:   1.45 kB[22m
[2mdist/[22m[36mjs/metric-trend-CFjVVxXF.js                                                                      [39m[1m[2m    2.80 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-operate-modal.vue_vue_type_script_setup_true_lang-hfIBfE6p.js                 [39m[1m[2m    2.85 kB[22m[1m[22m[2m │ gzip:   1.38 kB[22m
[2mdist/[22m[36mjs/MatrixCard-BgpgxnWS.js                                                                        [39m[1m[2m    2.85 kB[22m[1m[22m[2m │ gzip:   1.30 kB[22m
[2mdist/[22m[36mjs/notice-form-modal.vue_vue_type_script_setup_true_lang-ACOxkuoL.js                             [39m[1m[2m    2.85 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/help-doc-view-record-list-X5OnipRG.js                                                         [39m[1m[2m    2.88 kB[22m[1m[22m[2m │ gzip:   1.49 kB[22m
[2mdist/[22m[36mjs/message-send-form.vue_vue_type_script_setup_true_lang-AOptB0L9.js                             [39m[1m[2m    2.88 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-delete-Dg9NE6lN.js                                           [39m[1m[2m    2.91 kB[22m[1m[22m[2m │ gzip:   1.47 kB[22m
[2mdist/[22m[36mjs/index-CgCzCMyQ.js                                                                             [39m[1m[2m    2.92 kB[22m[1m[22m[2m │ gzip:   1.46 kB[22m
[2mdist/[22m[36mjs/index-DtsIfzhA.js                                                                             [39m[1m[2m    2.94 kB[22m[1m[22m[2m │ gzip:   1.65 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-operate-modal.vue_vue_type_script_setup_true_lang-DA1_Hoq4.js              [39m[1m[2m    2.99 kB[22m[1m[22m[2m │ gzip:   1.40 kB[22m
[2mdist/[22m[36mjs/dict-import-modal.vue_vue_type_script_setup_true_lang-MaIv9iH1.js                             [39m[1m[2m    3.00 kB[22m[1m[22m[2m │ gzip:   1.69 kB[22m
[2mdist/[22m[36mjs/change-log-form.vue_vue_type_script_setup_true_lang-Dkxhz7yV.js                               [39m[1m[2m    3.03 kB[22m[1m[22m[2m │ gzip:   1.44 kB[22m
[2mdist/[22m[36mjs/serial-number-list-7hlJrY76.js                                                                [39m[1m[2m    3.05 kB[22m[1m[22m[2m │ gzip:   1.90 kB[22m
[2mdist/[22m[36mjs/dict-data-form-modal.vue_vue_type_script_setup_true_lang-CafD2K6y.js                          [39m[1m[2m    3.06 kB[22m[1m[22m[2m │ gzip:   1.42 kB[22m
[2mdist/[22m[36mjs/home-to-be-done-DDSI6-id.js                                                                   [39m[1m[2m    3.06 kB[22m[1m[22m[2m │ gzip:   1.32 kB[22m
[2mdist/[22m[36mjs/index-Dgl1j1sO.js                                                                             [39m[1m[2m    3.07 kB[22m[1m[22m[2m │ gzip:   1.57 kB[22m
[2mdist/[22m[36mjs/index-D0L39gC5.js                                                                             [39m[1m[2m    3.09 kB[22m[1m[22m[2m │ gzip:   1.69 kB[22m
[2mdist/[22m[36mjs/snapshot-drawer-U5DDiy1p.js                                                                   [39m[1m[2m    3.11 kB[22m[1m[22m[2m │ gzip:   1.66 kB[22m
[2mdist/[22m[36mjs/TeamOutlined-BGKO8QW7.js                                                                      [39m[1m[2m    3.14 kB[22m[1m[22m[2m │ gzip:   1.34 kB[22m
[2mdist/[22m[36mjs/index-CyFEPhS0.js                                                                             [39m[1m[2m    3.16 kB[22m[1m[22m[2m │ gzip:   1.60 kB[22m
[2mdist/[22m[36mjs/submit-DQh0IkiA.js                                                                            [39m[1m[2m    3.18 kB[22m[1m[22m[2m │ gzip:   1.69 kB[22m
[2mdist/[22m[36mjs/notice-view-record-list.vue_vue_type_script_setup_true_lang-ChIYk8S2.js                       [39m[1m[2m    3.20 kB[22m[1m[22m[2m │ gzip:   1.55 kB[22m
[2mdist/[22m[36mjs/public-CEqyw-B3.js                                                                            [39m[1m[2m    3.20 kB[22m[1m[22m[2m │ gzip:   1.59 kB[22m
[2mdist/[22m[36mjs/role-tree-menu-DLRJeuLK.js                                                                    [39m[1m[2m    3.21 kB[22m[1m[22m[2m │ gzip:   1.50 kB[22m
[2mdist/[22m[36mjs/login-e-W7-Ync.js                                                                             [39m[1m[2m    3.23 kB[22m[1m[22m[2m │ gzip:   1.66 kB[22m
[2mdist/[22m[36mjs/feedback-list-ByJZA4fq.js                                                                     [39m[1m[2m    3.24 kB[22m[1m[22m[2m │ gzip:   1.61 kB[22m
[2mdist/[22m[36mjs/dict-form-modal.vue_vue_type_script_setup_true_lang-CU-etkCZ.js                               [39m[1m[2m    3.30 kB[22m[1m[22m[2m │ gzip:   1.75 kB[22m
[2mdist/[22m[36mjs/formatTableColumn-CHsnCt-I.js                                                                 [39m[1m[2m    3.31 kB[22m[1m[22m[2m │ gzip:   1.16 kB[22m
[2mdist/[22m[36mjs/template-form-modal.vue_vue_type_script_setup_true_lang-Bqnkuh7X.js                           [39m[1m[2m    3.32 kB[22m[1m[22m[2m │ gzip:   1.75 kB[22m
[2mdist/[22m[36mjs/index-CZw_Mq3n.js                                                                             [39m[1m[2m    3.37 kB[22m[1m[22m[2m │ gzip:   1.80 kB[22m
[2mdist/[22m[36mjs/code-generator-util-X5KUoHCl.js                                                               [39m[1m[2m    3.40 kB[22m[1m[22m[2m │ gzip:   0.90 kB[22m
[2mdist/[22m[36mjs/index-BMZLqwFu.js                                                                             [39m[1m[2m    3.44 kB[22m[1m[22m[2m │ gzip:   1.76 kB[22m
[2mdist/[22m[36mjs/help-doc-layout--6QJcpN4.js                                                                   [39m[1m[2m    3.47 kB[22m[1m[22m[2m │ gzip:   1.68 kB[22m
[2mdist/[22m[36mjs/category-CKx2VeqQ.js                                                                          [39m[1m[2m    3.49 kB[22m[1m[22m[2m │ gzip:   1.61 kB[22m
[2mdist/[22m[36mjs/instance-params-modal.vue_vue_type_script_setup_true_lang-DY3X2rXN.js                         [39m[1m[2m    3.50 kB[22m[1m[22m[2m │ gzip:   1.74 kB[22m
[2mdist/[22m[36mjs/index-O6vaXGpi.js                                                                             [39m[1m[2m    3.51 kB[22m[1m[22m[2m │ gzip:   1.56 kB[22m
[2mdist/[22m[36mjs/category-oV18JkK1.js                                                                          [39m[1m[2m    3.51 kB[22m[1m[22m[2m │ gzip:   1.62 kB[22m
[2mdist/[22m[36mjs/heart-beat-list-cSxEGA-x.js                                                                   [39m[1m[2m    3.58 kB[22m[1m[22m[2m │ gzip:   2.08 kB[22m
[2mdist/[22m[36mjs/point-detail-modal.vue_vue_type_script_setup_true_lang-Db_zocY4.js                            [39m[1m[2m    3.64 kB[22m[1m[22m[2m │ gzip:   1.66 kB[22m
[2mdist/[22m[36mjs/operate-log-detail-modal-DeSq_2cb.js                                                          [39m[1m[2m    3.65 kB[22m[1m[22m[2m │ gzip:   1.58 kB[22m
[2mdist/[22m[36mjs/notice-employee-list.vue_vue_type_script_setup_true_lang-BrVuM4eb.js                          [39m[1m[2m    3.65 kB[22m[1m[22m[2m │ gzip:   1.75 kB[22m
[2mdist/[22m[36mjs/gateway-form-modal.vue_vue_type_script_setup_true_lang-BPTP0ALC.js                            [39m[1m[2m    3.65 kB[22m[1m[22m[2m │ gzip:   1.90 kB[22m
[2mdist/[22m[36mjs/mass-send-detail-drawer.vue_vue_type_script_setup_true_lang-DHOpQ1hj.js                       [39m[1m[2m    3.69 kB[22m[1m[22m[2m │ gzip:   1.72 kB[22m
[2mdist/[22m[36mjs/config-list-4M8zKK8B.js                                                                       [39m[1m[2m    3.70 kB[22m[1m[22m[2m │ gzip:   1.74 kB[22m
[2mdist/[22m[36mjs/home-quick-entry-DHHdpQX2.js                                                                  [39m[1m[2m    3.70 kB[22m[1m[22m[2m │ gzip:   1.78 kB[22m
[2mdist/[22m[36mjs/tenant-form-modal.vue_vue_type_script_setup_true_lang-uByIrfAf.js                             [39m[1m[2m    3.72 kB[22m[1m[22m[2m │ gzip:   1.76 kB[22m
[2mdist/[22m[36mjs/card-CCM5VrtV.js                                                                              [39m[1m[2m    3.76 kB[22m[1m[22m[2m │ gzip:   1.68 kB[22m
[2mdist/[22m[36mjs/notice-detail-eFagpApR.js                                                                     [39m[1m[2m    3.76 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/index-Dnsv0GtR.js                                                                             [39m[1m[2m    3.80 kB[22m[1m[22m[2m │ gzip:   1.80 kB[22m
[2mdist/[22m[36mjs/redpack-BmB8hJv3.js                                                                           [39m[1m[2m    3.83 kB[22m[1m[22m[2m │ gzip:   2.16 kB[22m
[2mdist/[22m[36mjs/config-operate-drawer-rnv_g2l-.js                                                             [39m[1m[2m    3.83 kB[22m[1m[22m[2m │ gzip:   1.86 kB[22m
[2mdist/[22m[36mjs/index-CuLwUZdK.js                                                                             [39m[1m[2m    3.91 kB[22m[1m[22m[2m │ gzip:   1.80 kB[22m
[2mdist/[22m[36mjs/device-trace-zone.vue_vue_type_script_setup_true_lang-BmYtaVrL.js                             [39m[1m[2m    3.95 kB[22m[1m[22m[2m │ gzip:   1.93 kB[22m
[2mdist/[22m[36mjs/table-NVCBNGE6.js                                                                             [39m[1m[2m    3.97 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/mass-send-create-modal.vue_vue_type_script_setup_true_lang-qgXECDiZ.js                        [39m[1m[2m    4.01 kB[22m[1m[22m[2m │ gzip:   2.05 kB[22m
[2mdist/[22m[36mjs/DataPush-vNNgflEH.js                                                                          [39m[1m[2m    4.02 kB[22m[1m[22m[2m │ gzip:   2.08 kB[22m
[2mdist/[22m[36mjs/device-curve-zone.vue_vue_type_script_setup_true_lang-DSF8M9Uz.js                             [39m[1m[2m    4.04 kB[22m[1m[22m[2m │ gzip:   2.22 kB[22m
[2mdist/[22m[36mjs/ThunderboltOutlined-mJrJmJa8.js                                                               [39m[1m[2m    4.07 kB[22m[1m[22m[2m │ gzip:   1.49 kB[22m
[2mdist/[22m[36mjs/analysis-D35hr2F0.js                                                                          [39m[1m[2m    4.07 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/publish-log-drawer-Db9sDea_.js                                                                [39m[1m[2m    4.08 kB[22m[1m[22m[2m │ gzip:   1.96 kB[22m
[2mdist/[22m[36mjs/panel-B6H6XuGn.js                                                                             [39m[1m[2m    4.11 kB[22m[1m[22m[2m │ gzip:   2.08 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-Dx1cGAE-.js                                                  [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   1.70 kB[22m
[2mdist/[22m[36mjs/cross-B8MvvFTt.js                                                                             [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   2.03 kB[22m
[2mdist/[22m[36mjs/index-B66VDXvd.js                                                                             [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   2.22 kB[22m
[2mdist/[22m[36mjs/index-CHloA8Im.js                                                                             [39m[1m[2m    4.13 kB[22m[1m[22m[2m │ gzip:   2.05 kB[22m
[2mdist/[22m[36mjs/index-BuVClEbJ.js                                                                             [39m[1m[2m    4.17 kB[22m[1m[22m[2m │ gzip:   2.18 kB[22m
[2mdist/[22m[36mjs/index-Cd_h8VZC.js                                                                             [39m[1m[2m    4.18 kB[22m[1m[22m[2m │ gzip:   1.81 kB[22m
[2mdist/[22m[36mjs/index-BLM-cdH4.js                                                                             [39m[1m[2m    4.19 kB[22m[1m[22m[2m │ gzip:   1.96 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-query-field.vue_vue_type_script_setup_true_lang-DAgsRFZr.js  [39m[1m[2m    4.20 kB[22m[1m[22m[2m │ gzip:   1.84 kB[22m
[2mdist/[22m[36mjs/index-ChIS2gZ6.js                                                                             [39m[1m[2m    4.24 kB[22m[1m[22m[2m │ gzip:   2.19 kB[22m
[2mdist/[22m[36mjs/dict-data-modal.vue_vue_type_script_setup_true_lang-S6cYX003.js                               [39m[1m[2m    4.28 kB[22m[1m[22m[2m │ gzip:   2.12 kB[22m
[2mdist/[22m[36mjs/notify-DcSv-1zG.js                                                                            [39m[1m[2m    4.28 kB[22m[1m[22m[2m │ gzip:   2.18 kB[22m
[2mdist/[22m[36mjs/FileOutlined-C97mkViZ.js                                                                      [39m[1m[2m    4.29 kB[22m[1m[22m[2m │ gzip:   1.17 kB[22m
[2mdist/[22m[36mjs/index-2Ue9mHoG.js                                                                             [39m[1m[2m    4.29 kB[22m[1m[22m[2m │ gzip:   2.16 kB[22m
[2mdist/[22m[36mjs/index-Dqb9s-xk.js                                                                             [39m[1m[2m    4.34 kB[22m[1m[22m[2m │ gzip:   2.20 kB[22m
[2mdist/[22m[36mjs/share-Jtyu-Kdi.js                                                                             [39m[1m[2m    4.37 kB[22m[1m[22m[2m │ gzip:   1.98 kB[22m
[2mdist/[22m[36mjs/position-list-PIY2U0s2.js                                                                     [39m[1m[2m    4.39 kB[22m[1m[22m[2m │ gzip:   2.04 kB[22m
[2mdist/[22m[36mjs/module-list.vue_vue_type_script_setup_true_lang-U-vB33IZ.js                                   [39m[1m[2m    4.41 kB[22m[1m[22m[2m │ gzip:   2.01 kB[22m
[2mdist/[22m[36mjs/index-DJllTeWh.js                                                                             [39m[1m[2m    4.45 kB[22m[1m[22m[2m │ gzip:   2.55 kB[22m
[2mdist/[22m[36mjs/template-send-detail-drawer-CGYZd836.js                                                       [39m[1m[2m    4.45 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/code-generator-list-CjmVHETH.js                                                               [39m[1m[2m    4.51 kB[22m[1m[22m[2m │ gzip:   2.00 kB[22m
[2mdist/[22m[36mjs/module-form-B_n5Bn8t.js                                                                       [39m[1m[2m    4.56 kB[22m[1m[22m[2m │ gzip:   2.29 kB[22m
[2mdist/[22m[36mjs/index-BeE38fEr.js                                                                             [39m[1m[2m    4.57 kB[22m[1m[22m[2m │ gzip:   2.14 kB[22m
[2mdist/[22m[36mjs/job-log-list-modal.vue_vue_type_script_setup_true_lang-DTeMHVSq.js                            [39m[1m[2m    4.62 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/job-log-modal.vue_vue_type_script_setup_true_lang-ChQqEA3b.js                                 [39m[1m[2m    4.69 kB[22m[1m[22m[2m │ gzip:   2.15 kB[22m
[2mdist/[22m[36mjs/DataFilter-LoxqMYXF.js                                                                        [39m[1m[2m    4.72 kB[22m[1m[22m[2m │ gzip:   1.87 kB[22m
[2mdist/[22m[36mjs/dict-data-modal.vue_vue_type_script_setup_true_lang-Cec_CvzY.js                               [39m[1m[2m    4.79 kB[22m[1m[22m[2m │ gzip:   2.19 kB[22m
[2mdist/[22m[36mjs/help-doc-list.vue_vue_type_script_setup_true_lang-B1qjBDwB.js                                 [39m[1m[2m    4.79 kB[22m[1m[22m[2m │ gzip:   2.19 kB[22m
[2mdist/[22m[36mjs/login-fail-list-ChGCWeta.js                                                                   [39m[1m[2m    4.81 kB[22m[1m[22m[2m │ gzip:   2.19 kB[22m
[2mdist/[22m[36mjs/enterprise-invoice-list.vue_vue_type_script_setup_true_lang-BGvkVzdo.js                       [39m[1m[2m    4.81 kB[22m[1m[22m[2m │ gzip:   2.29 kB[22m
[2mdist/[22m[36mjs/apiclient-form-modal-ChwFEtX3.js                                                              [39m[1m[2m    4.82 kB[22m[1m[22m[2m │ gzip:   2.48 kB[22m
[2mdist/[22m[36mjs/enterprise-bank-list.vue_vue_type_script_setup_true_lang-C1PEjkIC.js                          [39m[1m[2m    4.82 kB[22m[1m[22m[2m │ gzip:   2.27 kB[22m
[2mdist/[22m[36mjs/create-DNOyQAwp.js                                                                            [39m[1m[2m    4.85 kB[22m[1m[22m[2m │ gzip:   2.40 kB[22m
[2mdist/[22m[36mjs/index-VGjRJWIE.js                                                                             [39m[1m[2m    4.86 kB[22m[1m[22m[2m │ gzip:   2.06 kB[22m
[2mdist/[22m[36mjs/index-VB3kNYra.js                                                                             [39m[1m[2m    4.94 kB[22m[1m[22m[2m │ gzip:   2.29 kB[22m
[2mdist/[22m[36mjs/ZoomOutOutlined-D9wymeX2.js                                                                   [39m[1m[2m    5.02 kB[22m[1m[22m[2m │ gzip:   1.43 kB[22m
[2mdist/[22m[36mjs/message-list-lcgQrqOQ.js                                                                      [39m[1m[2m    5.12 kB[22m[1m[22m[2m │ gzip:   2.27 kB[22m
[2mdist/[22m[36mjs/keyword-form-modal.vue_vue_type_script_setup_true_lang-hsE7WpLV.js                            [39m[1m[2m    5.15 kB[22m[1m[22m[2m │ gzip:   2.35 kB[22m
[2mdist/[22m[36mjs/instance-item-drawer-D2hJdNP_.js                                                              [39m[1m[2m    5.18 kB[22m[1m[22m[2m │ gzip:   2.49 kB[22m
[2mdist/[22m[36mjs/index-Vbz9CEXn.js                                                                             [39m[1m[2m    5.18 kB[22m[1m[22m[2m │ gzip:   2.22 kB[22m
[2mdist/[22m[36mjs/layout-const-Cm9Qp59K.js                                                                      [39m[1m[2m    5.21 kB[22m[1m[22m[2m │ gzip:   2.63 kB[22m
[2mdist/[22m[36mjs/job-form-modal.vue_vue_type_script_setup_true_lang-Bl8Te3-t.js                                [39m[1m[2m    5.33 kB[22m[1m[22m[2m │ gzip:   2.17 kB[22m
[2mdist/[22m[36mjs/index-BN-sUZ7a.js                                                                             [39m[1m[2m    5.44 kB[22m[1m[22m[2m │ gzip:   2.35 kB[22m
[2mdist/[22m[36mjs/index-d8ZJVfun.js                                                                             [39m[1m[2m    5.45 kB[22m[1m[22m[2m │ gzip:   2.52 kB[22m
[2mdist/[22m[36mjs/help-doc-form-drawer-QLfyjG4z.js                                                              [39m[1m[2m    5.46 kB[22m[1m[22m[2m │ gzip:   2.54 kB[22m
[2mdist/[22m[36mjs/help-doc-catalog-tree-CqsPzmwP.js                                                             [39m[1m[2m    5.46 kB[22m[1m[22m[2m │ gzip:   2.49 kB[22m
[2mdist/[22m[36mjs/template-send-create-modal-Ba-7bDhC.js                                                        [39m[1m[2m    5.48 kB[22m[1m[22m[2m │ gzip:   2.92 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-field.vue_vue_type_script_setup_true_lang-C1rP5fXb.js        [39m[1m[2m    5.58 kB[22m[1m[22m[2m │ gzip:   2.27 kB[22m
[2mdist/[22m[36mjs/index-DtaBVDis.js                                                                             [39m[1m[2m    5.58 kB[22m[1m[22m[2m │ gzip:   2.46 kB[22m
[2mdist/[22m[36mjs/index-ChZBj8TH.js                                                                             [39m[1m[2m    5.59 kB[22m[1m[22m[2m │ gzip:   2.38 kB[22m
[2mdist/[22m[36mjs/config-list.vue_vue_type_script_setup_true_lang-DgHkHldv.js                                   [39m[1m[2m    5.63 kB[22m[1m[22m[2m │ gzip:   2.31 kB[22m
[2mdist/[22m[36mjs/index-BgUIW1oR.js                                                                             [39m[1m[2m    5.69 kB[22m[1m[22m[2m │ gzip:   2.32 kB[22m
[2mdist/[22m[36mjs/index-Brk4W5XO.js                                                                             [39m[1m[2m    5.70 kB[22m[1m[22m[2m │ gzip:   2.57 kB[22m
[2mdist/[22m[36mjs/iot-formula-editor-CJlM6qzZ.js                                                                [39m[1m[2m    5.72 kB[22m[1m[22m[2m │ gzip:   2.89 kB[22m
[2mdist/[22m[36mjs/index-Bt4WLs_Q.js                                                                             [39m[1m[2m    5.73 kB[22m[1m[22m[2m │ gzip:   2.48 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_style_index_0_lang-DOqf9raF.js                                             [39m[1m[2m    5.74 kB[22m[1m[22m[2m │ gzip:   1.87 kB[22m
[2mdist/[22m[36mjs/enterprise-list-B4QMscIy.js                                                                   [39m[1m[2m    5.77 kB[22m[1m[22m[2m │ gzip:   2.64 kB[22m
[2mdist/[22m[36mjs/index-D2fwdFns.js                                                                             [39m[1m[2m    5.81 kB[22m[1m[22m[2m │ gzip:   2.66 kB[22m
[2mdist/[22m[36mjs/instance-form-modal.vue_vue_type_script_setup_true_lang-BA_Fg5Qp.js                           [39m[1m[2m    5.89 kB[22m[1m[22m[2m │ gzip:   2.79 kB[22m
[2mdist/[22m[36mjs/point-form-modal.vue_vue_type_script_setup_true_lang-BlMOx03S.js                              [39m[1m[2m    5.94 kB[22m[1m[22m[2m │ gzip:   2.59 kB[22m
[2mdist/[22m[36mjs/api-encrypt-index-B_KdaSjH.js                                                                 [39m[1m[2m    5.99 kB[22m[1m[22m[2m │ gzip:   1.74 kB[22m
[2mdist/[22m[36mjs/goods-form-modal.vue_vue_type_script_setup_true_lang-BRDSqTK5.js                              [39m[1m[2m    6.11 kB[22m[1m[22m[2m │ gzip:   2.58 kB[22m
[2mdist/[22m[36mjs/file-list-DGtd-udq.js                                                                         [39m[1m[2m    6.14 kB[22m[1m[22m[2m │ gzip:   2.44 kB[22m
[2mdist/[22m[36mjs/index-XpCB7XpM.js                                                                             [39m[1m[2m    6.15 kB[22m[1m[22m[2m │ gzip:   2.77 kB[22m
[2mdist/[22m[36mjs/level3-protect-config-index-kCW8i17R.js                                                       [39m[1m[2m    6.20 kB[22m[1m[22m[2m │ gzip:   2.80 kB[22m
[2mdist/[22m[36mjs/index-BoCF6Ij8.js                                                                             [39m[1m[2m    6.20 kB[22m[1m[22m[2m │ gzip:   2.88 kB[22m
[2mdist/[22m[36mjs/channel-credential-drawer-FaivP8V1.js                                                         [39m[1m[2m    6.20 kB[22m[1m[22m[2m │ gzip:   3.11 kB[22m
[2mdist/[22m[36mjs/notice-form-drawer-kWX1_YS9.js                                                                [39m[1m[2m    6.22 kB[22m[1m[22m[2m │ gzip:   2.71 kB[22m
[2mdist/[22m[36mjs/deleted-job-list.vue_vue_type_script_setup_true_lang-CTKmdiZm.js                              [39m[1m[2m    6.23 kB[22m[1m[22m[2m │ gzip:   2.61 kB[22m
[2mdist/[22m[36mjs/login-log-list-C6qkvbzo.js                                                                    [39m[1m[2m    6.24 kB[22m[1m[22m[2m │ gzip:   2.75 kB[22m
[2mdist/[22m[36mjs/change-log-list-T8vyXSZc.js                                                                   [39m[1m[2m    6.25 kB[22m[1m[22m[2m │ gzip:   2.55 kB[22m
[2mdist/[22m[36mjs/index-BP06FyqV.js                                                                             [39m[1m[2m    6.28 kB[22m[1m[22m[2m │ gzip:   2.56 kB[22m
[2mdist/[22m[36mjs/index-DM_iw7XJ.js                                                                             [39m[1m[2m    6.36 kB[22m[1m[22m[2m │ gzip:   2.62 kB[22m
[2mdist/[22m[36mjs/index-D5qSoUZQ.js                                                                             [39m[1m[2m    6.47 kB[22m[1m[22m[2m │ gzip:   2.69 kB[22m
[2mdist/[22m[36mjs/channel-form-modal.vue_vue_type_script_setup_true_lang-D6BuIKWT.js                            [39m[1m[2m    6.52 kB[22m[1m[22m[2m │ gzip:   3.21 kB[22m
[2mdist/[22m[36mjs/ViewData-B4GA3p0B.js                                                                          [39m[1m[2m    6.56 kB[22m[1m[22m[2m │ gzip:   2.46 kB[22m
[2mdist/[22m[36mjs/index-k8aqTZ_M.js                                                                             [39m[1m[2m    6.58 kB[22m[1m[22m[2m │ gzip:   2.72 kB[22m
[2mdist/[22m[36mjs/index-Cw3wwwQ2.js                                                                             [39m[1m[2m    6.68 kB[22m[1m[22m[2m │ gzip:   3.13 kB[22m
[2mdist/[22m[36mjs/index-DHzgxnky.js                                                                             [39m[1m[2m    6.71 kB[22m[1m[22m[2m │ gzip:   2.85 kB[22m
[2mdist/[22m[36mjs/ViewOrUpdate-BGwqnnwv.js                                                                      [39m[1m[2m    6.74 kB[22m[1m[22m[2m │ gzip:   3.26 kB[22m
[2mdist/[22m[36mjs/index-CSs1lPMi.js                                                                             [39m[1m[2m    6.80 kB[22m[1m[22m[2m │ gzip:   2.99 kB[22m
[2mdist/[22m[36mjs/index-Bb89iU0I.js                                                                             [39m[1m[2m    6.89 kB[22m[1m[22m[2m │ gzip:   2.61 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-insert-and-update-Qr7DE_Sy.js                                [39m[1m[2m    7.05 kB[22m[1m[22m[2m │ gzip:   2.56 kB[22m
[2mdist/[22m[36mjs/metric-form-modal-DA017gt9.js                                                                 [39m[1m[2m    7.11 kB[22m[1m[22m[2m │ gzip:   3.01 kB[22m
[2mdist/[22m[36mjs/publish-confirm-modal-DsMoEd1h.js                                                             [39m[1m[2m    7.22 kB[22m[1m[22m[2m │ gzip:   3.50 kB[22m
[2mdist/[22m[36mjs/code-generator-table-config-form-basic-DhLgKaKm.js                                            [39m[1m[2m    7.39 kB[22m[1m[22m[2m │ gzip:   2.78 kB[22m
[2mdist/[22m[36mjs/index-DK4yTOQG.js                                                                             [39m[1m[2m    7.41 kB[22m[1m[22m[2m │ gzip:   3.26 kB[22m
[2mdist/[22m[36mjs/notice-list-AqI8hfY4.js                                                                       [39m[1m[2m    7.45 kB[22m[1m[22m[2m │ gzip:   2.90 kB[22m
[2mdist/[22m[36mjs/menu-list.vue_vue_type_script_setup_true_lang-CTEZwX7F.js                                     [39m[1m[2m    7.50 kB[22m[1m[22m[2m │ gzip:   2.93 kB[22m
[2mdist/[22m[36mjs/index-BMmpdKKS.js                                                                             [39m[1m[2m    7.55 kB[22m[1m[22m[2m │ gzip:   3.53 kB[22m
[2mdist/[22m[36mjs/operate-log-list-Dy0qeChC.js                                                                  [39m[1m[2m    7.65 kB[22m[1m[22m[2m │ gzip:   3.21 kB[22m
[2mdist/[22m[36mjs/index-D_MqNlnr.js                                                                             [39m[1m[2m    7.72 kB[22m[1m[22m[2m │ gzip:   3.26 kB[22m
[2mdist/[22m[36mjs/job-list-Caz8ykl2.js                                                                          [39m[1m[2m    8.38 kB[22m[1m[22m[2m │ gzip:   3.34 kB[22m
[2mdist/[22m[36mjs/index.vue_vue_type_script_setup_true_lang-JPX3ZWUW.js                                         [39m[1m[2m    8.59 kB[22m[1m[22m[2m │ gzip:   3.68 kB[22m
[2mdist/[22m[36mjs/enterprise-employee-list-lgSBl6ej.js                                                          [39m[1m[2m    8.63 kB[22m[1m[22m[2m │ gzip:   3.33 kB[22m
[2mdist/[22m[36mjs/index-CorMmZd6.js                                                                             [39m[1m[2m    8.75 kB[22m[1m[22m[2m │ gzip:   3.22 kB[22m
[2mdist/[22m[36mjs/index-B_6sSlII.js                                                                             [39m[1m[2m    8.99 kB[22m[1m[22m[2m │ gzip:   4.14 kB[22m
[2mdist/[22m[36mjs/index-CcY-0D8M.js                                                                             [39m[1m[2m    9.12 kB[22m[1m[22m[2m │ gzip:   3.90 kB[22m
[2mdist/[22m[36mjs/index-V082PAlH.js                                                                             [39m[1m[2m    9.55 kB[22m[1m[22m[2m │ gzip:   4.00 kB[22m
[2mdist/[22m[36mjs/index-DLF29sKA.js                                                                             [39m[1m[2m    9.67 kB[22m[1m[22m[2m │ gzip:   3.82 kB[22m
[2mdist/[22m[36mjs/index-BcR3CC77.js                                                                             [39m[1m[2m    9.74 kB[22m[1m[22m[2m │ gzip:   3.94 kB[22m
[2mdist/[22m[36mjs/index-CpAYYvTC.js                                                                             [39m[1m[2m    9.86 kB[22m[1m[22m[2m │ gzip:   4.05 kB[22m
[2mdist/[22m[36mjs/index-BE-DGU9_.js                                                                             [39m[1m[2m   10.17 kB[22m[1m[22m[2m │ gzip:   3.82 kB[22m
[2mdist/[22m[36mjs/template-item-modal.vue_vue_type_script_setup_true_lang-CwWzhWHb.js                           [39m[1m[2m   10.39 kB[22m[1m[22m[2m │ gzip:   4.03 kB[22m
[2mdist/[22m[36mjs/SendOutlined-Cb7qJACV.js                                                                      [39m[1m[2m   10.86 kB[22m[1m[22m[2m │ gzip:   2.66 kB[22m
[2mdist/[22m[36mjs/index-CdhhRE6R.js                                                                             [39m[1m[2m   10.94 kB[22m[1m[22m[2m │ gzip:   4.74 kB[22m
[2mdist/[22m[36mjs/index-C3ks1BxV.js                                                                             [39m[1m[2m   11.15 kB[22m[1m[22m[2m │ gzip:   4.35 kB[22m
[2mdist/[22m[36mjs/index-kIVq2DXg.js                                                                             [39m[1m[2m   11.17 kB[22m[1m[22m[2m │ gzip:   3.31 kB[22m
[2mdist/[22m[36mjs/menu-operate-modal-BTCk_CCQ.js                                                                [39m[1m[2m   11.36 kB[22m[1m[22m[2m │ gzip:   4.13 kB[22m
[2mdist/[22m[36mjs/write-C4zoJ8BD.js                                                                             [39m[1m[2m   12.75 kB[22m[1m[22m[2m │ gzip:   3.98 kB[22m
[2mdist/[22m[36mjs/goods-list-DhUr2og2.js                                                                        [39m[1m[2m   15.49 kB[22m[1m[22m[2m │ gzip:   5.61 kB[22m
[2mdist/[22m[36mjs/index-qDd37Kqo.js                                                                             [39m[1m[2m   16.01 kB[22m[1m[22m[2m │ gzip:   6.63 kB[22m
[2mdist/[22m[36mjs/ua-parser-CMsvxOnP.js                                                                         [39m[1m[2m   16.51 kB[22m[1m[22m[2m │ gzip:   7.34 kB[22m
[2mdist/[22m[36mjs/EnterOutlined-BsSkBvRQ.js                                                                     [39m[1m[2m   18.83 kB[22m[1m[22m[2m │ gzip:   3.43 kB[22m
[2mdist/[22m[36mjs/VideoCameraOutlined-7Czc_uqZ.js                                                               [39m[1m[2m   20.27 kB[22m[1m[22m[2m │ gzip:   4.81 kB[22m
[2mdist/[22m[36mjs/qrcode.vue.esm-CBvx-gBj.js                                                                    [39m[1m[2m   20.75 kB[22m[1m[22m[2m │ gzip:   7.31 kB[22m
[2mdist/[22m[36mjs/bundle.esm-Dr9gGUWg.js                                                                        [39m[1m[2m   20.76 kB[22m[1m[22m[2m │ gzip:   5.97 kB[22m
[2mdist/[22m[36mjs/index-COfN53vn.js                                                                             [39m[1m[2m   25.00 kB[22m[1m[22m[2m │ gzip:   8.49 kB[22m
[2mdist/[22m[36mjs/avatar-cropper-modal-CH_GilIM.js                                                              [39m[1m[2m   35.31 kB[22m[1m[22m[2m │ gzip:  10.36 kB[22m
[2mdist/[22m[36mjs/sortable.esm-CRp0PNdb.js                                                                      [39m[1m[2m   35.52 kB[22m[1m[22m[2m │ gzip:  12.26 kB[22m
[2mdist/[22m[36mjs/index.esm-BtDI5tGi.js                                                                         [39m[1m[2m   45.58 kB[22m[1m[22m[2m │ gzip:  17.23 kB[22m
[2mdist/[22m[36mjs/enterprise-detail-BcKeVsux.js                                                                 [39m[1m[2m   57.70 kB[22m[1m[22m[2m │ gzip:  17.09 kB[22m
[2mdist/[22m[36mjs/index-Cn2swI3W.js                                                                             [39m[1m[2m   57.90 kB[22m[1m[22m[2m │ gzip:  14.30 kB[22m
[2mdist/[22m[36mjs/enterprise-operate-modal-BCRY-6gL.js                                                          [39m[1m[2m  151.85 kB[22m[1m[22m[2m │ gzip:  34.97 kB[22m
[2mdist/[22m[36mjs/index-BFlp8De4.js                                                                             [39m[1m[2m  170.10 kB[22m[1m[22m[2m │ gzip:  49.29 kB[22m
[2mdist/[22m[36mjs/home-header-DrR_Z2lu.js                                                                       [39m[1m[2m  294.39 kB[22m[1m[22m[2m │ gzip: 102.76 kB[22m
[2mdist/[22m[36mjs/index.esm-BnH0H9nq.js                                                                         [39m[1m[2m  458.39 kB[22m[1m[22m[2m │ gzip: 137.65 kB[22m
[2mdist/[22m[36mjs/index.esm-lztK2ICv.js                                                                         [39m[1m[33m  547.28 kB[39m[22m[2m │ gzip: 157.18 kB[22m
[2mdist/[22m[36mjs/index-CN3O-Art.js                                                                             [39m[1m[33m  652.71 kB[39m[22m[2m │ gzip: 205.35 kB[22m
[2mdist/[22m[36mjs/code-generator-preview-modal-BB4ongiy.js                                                      [39m[1m[33m  915.72 kB[39m[22m[2m │ gzip: 303.29 kB[22m
[2mdist/[22m[36mjs/index-BFKZD-pc.js                                                                             [39m[1m[33m  928.90 kB[39m[22m[2m │ gzip: 274.04 kB[22m
[2mdist/[22m[36mjs/index-2gfuB-dF.js                                                                             [39m[1m[33m  992.43 kB[39m[22m[2m │ gzip: 159.99 kB[22m
[2mdist/[22m[36mjs/index-BdCjgtU4.js                                                                             [39m[1m[33m1,011.26 kB[39m[22m[2m │ gzip: 325.74 kB[22m
[2mdist/[22m[36mjs/index-BAsMtFso.js                                                                             [39m[1m[33m1,015.77 kB[39m[22m[2m │ gzip: 329.40 kB[22m
[2mdist/[22m[36mjs/index-BMhHitu8.js                                                                             [39m[1m[33m1,316.21 kB[39m[22m[2m │ gzip: 432.27 kB[22m
[33m
(!) Some chunks are larger than 500 kB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking: https://rollupjs.org/configuration-options/#output-manualchunks
- Adjust chunk size limit for this warning via build.chunkSizeWarningLimit.[39m
[32m✓ built in 1m 15s[39m
```

#### `pnpm lint`

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

#### `pnpm typecheck`

```text
$ vue-tsc --noEmit
```

#### `pnpm test`

```text
$ vitest run

[1m[7m[36m RUN [39m[27m[22m [36mv2.1.9 [39m[90mD:/workspace/WisFactory/frontend[39m

 [32m✓[39m tests/unit/components/iot/formula-rules.spec.ts [2m([22m[2m32 tests[22m[2m)[22m[90m 23[2mms[22m[39m
 [32m✓[39m tests/unit/lib/axios.spec.ts [2m([22m[2m27 tests[22m[2m)[22m[33m 475[2mms[22m[39m
 [32m✓[39m tests/unit/composables/survey/use-edit-lock.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[90m 78[2mms[22m[39m
 [32m✓[39m tests/unit/wechat/template-send-helper.spec.ts [2m([22m[2m17 tests[22m[2m)[22m[90m 18[2mms[22m[39m
 [32m✓[39m tests/unit/router/build-routes.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[90m 133[2mms[22m[39m
 [32m✓[39m tests/unit/router/router-guard.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[90m 50[2mms[22m[39m
 [32m✓[39m tests/unit/components/iot/iot-formula-editor.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[33m 1583[2mms[22m[39m
   [33m[2m✓[22m[39m 公式编辑器：渲染与实时校验[2m > [22m初始公式渲染到 textarea，合法公式无错误提示 [33m360[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/item-formula-rollup-modal.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[33m 1692[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项取值编辑弹窗（TASK-0094）[2m > [22mformula 模式：父 ref 可调 showModal，回填公式并按清单构造 SELF 引用分组 [33m584[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项取值编辑弹窗（TASK-0094）[2m > [22mformula 模式：合法公式 → saveFormula(1001,2002,formula) → 成功提示 + reloadList + 关窗 [33m309[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项取值编辑弹窗（TASK-0094）[2m > [22m保存失败：captureError + message.error 提示，弹窗不关闭 [33m319[2mms[22m[39m
 [32m✓[39m tests/unit/display/metrics-board-read-ports.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[33m 2404[2mms[22m[39m
   [33m[2m✓[22m[39m 指标看板接真值读口（TASK-0089 D3）[2m > [22mmetricValues→卡片同源展示：value/yoy/mom/unit 原样，不现算 [33m681[2mms[22m[39m
   [33m[2m✓[22m[39m 指标看板接真值读口（TASK-0089 D3）[2m > [22mnull 伴随值保持空态，不冒充 0 [33m503[2mms[22m[39m
   [33m[2m✓[22m[39m 指标看板接真值读口（TASK-0089 D3）[2m > [22mcompleteness available=false（当前诚实空集）→ 保持空态 + readGap 告警，不静默判完整 [33m453[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-read-ports.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 13[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m[2m顶行内同时渲染一级菜单与用户区（用户区不再挂在下方 header）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m[2m用户区排在菜单之后（DOM 顺序即视觉左右顺序：菜单在左、用户区在右）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m[2m顶行不再渲染二级菜单（二级菜单回到侧栏，由 layout 直接挂载）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace class="user-space" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m用户区组件 menu-user-space.vue[2m > [22m[2m渲染模块切换 / 消息 / 设置 / 头像四组元素
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m用户区组件 menu-user-space.vue[2m > [22m[2m组件自持配色解析，明暗切换不报错（学术蓝白底深字 / 经典深底白字）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-user-space.spec.ts[2m > [22m[2m「顶部」布局复用同一用户区组件（top-menu/index.vue）[2m > [22m[2m渲染后用户区存在且与分组布局同源
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace> 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/top-expand-user-space.spec.ts [2m([22m[2m17 tests[22m[2m)[22m[33m 2203[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m顶行内同时渲染一级菜单与用户区（用户区不再挂在下方 header） [33m1008[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局顶行：一级菜单与用户区同一行（top-expand-menu/index.vue）[2m > [22m用户区排在菜单之后（DOM 顺序即视觉左右顺序：菜单在左、用户区在右） [33m385[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/instance-tree-edit.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[33m 6518[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22mtitle 插槽在工厂/车间/设备各级节点旁渲染 EditOutlined 编辑按钮 [33m1624[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m点击工厂节点编辑按钮 → 复用 addInstance 打开编辑弹窗（完整 record 传 showModal） [33m566[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m骨架节点（缺 deptId/sort）点击 → 拉详情补齐字段后打开编辑弹窗 [33m729[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m详情加载失败时不开空弹窗，并上报 otSentry [33m796[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22mlocked 设备节点编辑按钮透传 locked=1（弹窗自身显示仅改状态提示） [33m719[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m无 iot:instance:edit 权限时编辑按钮被 v-privilege 指令移除（树节点仍在） [33m544[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m树过滤仍保留命中节点及其祖先路径，且编辑按钮随树保留 [33m747[2mms[22m[39m
   [33m[2m✓[22m[39m 实例树节点编辑入口（TASK-0093）[2m > [22m树节点选择仍触发 onSelectTree（点击标签文字检索子树，与编辑按钮独立） [33m789[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-cache-frame.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[33m 7417[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m回显：后端 0/0 显示为未开启，1/1 显示为开启（null/0 不得误判为真） [33m2673[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m提交：布尔开关映射为 0/1（后端只接受 0/1，不接受布尔） [33m1026[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m提交：开启外链后 frameFlag=1 且带上 frameUrl [33m1553[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m提交：关闭外链时清空外链地址，避免库中残留旧地址与开关状态不一致 [33m1450[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单「是否缓存 / 是否外链」开关回显与提交[2m > [22m外链菜单仍需填组件路径（buildRoutes 会跳过无 component 的菜单项） [33m711[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m「顶部」布局仍不显示该设置项（无左侧栏）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m分组布局显示该设置项，且取值回显 store 宽度
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m传统布局仍显示该设置项（原行为不回归）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/top-expand-side-menu.spec.ts[2m > [22m[2m设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m[2m「展开」(side-expand) 不显示该设置项（侧栏宽度由内部固定值决定，调了不生效）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/top-expand-side-menu.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[33m 4398[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局左侧菜单固定显示（BUG-0097）[2m > [22m无二级菜单（首页场景）时容器与 logo 区仍渲染且不被隐藏 [33m339[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m「顶部」布局仍不显示该设置项（无左侧栏） [33m1677[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m分组布局显示该设置项，且取值回显 store 宽度 [33m866[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m传统布局仍显示该设置项（原行为不回归） [33m716[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉「菜单宽度」在分组模式可见（FEAT-0092）[2m > [22m「展开」(side-expand) 不显示该设置项（侧栏宽度由内部固定值决定，调了不生效） [33m632[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/metric.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[33m 8914[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m列表加载真实后端数据、无 IotPlaceholder；查询参数透传 [33m1499[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m详情按钮 → detail(id) → 详情弹窗展示（父 ref defineExpose） [33m1156[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m表单弹窗父 ref 可调用 showModal / validateRatio；比值型缺分子分母被拦 [33m1056[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m新增态选实例后加载数据项下拉，比值型可选分子/分母并成功提交（评审 Important 回归） [33m1385[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m新增提交：calendarType 锁 1，非比值型 ratio_* 置空，metricCode 原样透传 [33m698[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m编辑提交：metricCode 携带原值（不可改语义），status 不进更新集，走 update [33m937[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m启停走 changeStatus，删除走 remove，不经过 update [33m995[2mms[22m[39m
   [33m[2m✓[22m[39m 指标管理页与表单校验（TASK-0095）[2m > [22m批量生成：instanceId 作 query 入参、清单作 body；空清单/非法口径被前置拦截 [33m1183[2mms[22m[39m
 [32m✓[39m tests/unit/store/user-module.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 233[2mms[22m[39m
 [32m✓[39m tests/unit/wechat/mass-send-helper.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[90m 13[2mms[22m[39m
[90mstderr[2m | unknown test
[22m[39mOPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at "http://127.0.0.1:8098/system/userUiConfig/list".

 [32m✓[39m tests/unit/views/iot/point.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[33m 10267[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m挂载拉列表（默认分页）且无占位骨架文案 [33m2514[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m空数据时展示诚实空态，不造假数据 [33m1142[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m新建/详情通过父 ref 调 defineExpose.showModal（BUG-0008 回归） [33m1036[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m启停调 changeStatus({ids,status})，停用含发布生效提示 [33m1133[2mms[22m[39m
   [33m[2m✓[22m[39m 点位管理页（TASK-0097）[2m > [22m删除调 remove([id])（API 层拼路径参数），导出调 export 带过滤参数 [33m971[2mms[22m[39m
   [33m[2m✓[22m[39m 点位表单弹窗（TASK-0097）[2m > [22m新增提交调 add，payload 不含 status/id [33m705[2mms[22m[39m
   [33m[2m✓[22m[39m 点位表单弹窗（TASK-0097）[2m > [22m编辑提交调 update，携带 id 与 pointCode 同值，且不含 status [33m1345[2mms[22m[39m
   [33m[2m✓[22m[39m 点位表单弹窗（TASK-0097）[2m > [22m编辑 showModal 后 pointCode 禁用（创建后不可改） [33m635[2mms[22m[39m
   [33m[2m✓[22m[39m 点位详情弹窗（TASK-0097）[2m > [22m父 ref 调 showModal(id) → detail/references/curve 三读，曲线空集展示诚实提示 [33m783[2mms[22m[39m
[90mstderr[2m | unknown test
[22m[39mOPTIONS http://127.0.0.1:8098/system/userUiConfig/list 500 ()
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

[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m[2mlayout/index.vue 渲染设置抽屉一份
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m[2m发 settingMitt open 事件后抽屉打开
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
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

 [32m✓[39m tests/unit/survey/report-format.spec.ts [2m([22m[2m22 tests[22m[2m)[22m[90m 15[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/point-bind-modal.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 2768[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项绑定点位弹窗（TASK-0092）[2m > [22m父 ref 可调 showModal；拉 status=1 启用点位列表并展示点位编码/名称/值类型 [33m1449[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项绑定点位弹窗（TASK-0092）[2m > [22m选一行点确定 → bindPoint(1001,2002,301) → 成功提示 + 事件通知 + 关窗 [33m669[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项绑定点位弹窗（TASK-0092）[2m > [22m绑定失败 → captureError + message.error 提示，弹窗不关闭 [33m382[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-perms-mapping.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[33m 11221[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m编辑回显：输入框显示库中 perms（修复前恒空） [33m1613[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m库中 perms 为 null 时回显空串，不显示字面量 null [33m916[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m改值提交：updateMenu 收到 params.perms 为目标值（修复前后端收不到该字段） [33m1698[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m清空提交：提交空串而非 null（null 会被后端 UPDATE 跳过、清不掉原权限串） [33m1268[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「权限字符」→ perms 映射[2m > [22m新增功能点带权限串提交：addMenu 收到 params.perms [33m669[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095）[2m > [22m编辑回显：输入框显示库中 query（修复前绑 queryParam，后端无此字段故恒空） [33m883[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095）[2m > [22m改值提交：updateMenu 收到 params.query 为目标值，且不再有 queryParam [33m1030[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗「路由参数」→ query 映射（BUG-0095）[2m > [22m清空提交：提交空串而非 null（同 perms 口径，null 会被后端 UPDATE 跳过） [33m1231[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090）[2m > [22m菜单类型抽屉无「路由名称」表单项 [33m1030[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单表单不再有「路由名称」（routeName 死字段，FEAT-0090）[2m > [22m提交报文不含 routeName [33m880[2mms[22m[39m
 [32m✓[39m tests/unit/utils/public-utils-layering.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 14[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m[2m切换「菜单布局」后抽屉仍打开（修复前会随布局组件卸载而消失）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>
[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="headerSettingRef" > 
  at <Index ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/setting-drawer-persist.spec.ts[2m > [22m[2m设置抽屉触发链路（BUG-0098 源码层）[2m > [22m[2m设置按钮仍渲染（用户区功能不回归）
[22m[39m[Vue warn]: Failed to resolve component: setting-outlined
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <MenuUserSpace ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/setting-drawer-persist.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[33m 4089[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m发 settingMitt open 事件后抽屉打开 [33m2667[2mms[22m[39m
   [33m[2m✓[22m[39m 设置抽屉挂在布局公共父级（BUG-0098 渲染层）[2m > [22m切换「菜单布局」后抽屉仍打开（修复前会随布局组件卸载而消失） [33m816[2mms[22m[39m
 [32m✓[39m tests/unit/theme/disabled-button-text.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 23[2mms[22m[39m
 [32m✓[39m tests/unit/utils/sanitize-html.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 97[2mms[22m[39m
 [32m✓[39m tests/unit/store/app-config-remote.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 45[2mms[22m[39m
 [32m✓[39m tests/unit/composables/survey/project-list-columns.spec.ts [2m([22m[2m19 tests[22m[2m)[22m[90m 8[2mms[22m[39m
 [32m✓[39m tests/unit/store/menu-module-helper.spec.ts [2m([22m[2m21 tests[22m[2m)[22m[90m 15[2mms[22m[39m
 [32m✓[39m tests/unit/layout/menu-path-match.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 7[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/instance-item-value-editor.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 2948[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22m公式项/汇总项各显示对应编辑按钮，权限不足时按钮被移除 [33m1149[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22m点击「编辑公式」→ 弹窗打开 → 保存 → 调 saveFormula 并刷新清单 [33m796[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22m点击「编辑汇总」→ 弹窗打开 → 切 AVG 保存 → 调 saveRollup 并刷新清单 [33m646[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉取值编辑入口（TASK-0094）[2m > [22mlocked=1：编辑按钮禁用，点击提示锁定，不打开弹窗、不提交 [33m356[2mms[22m[39m
[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m默认全部展开：每个顶级分类都渲染折叠图标（收起态图标不出现）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m点折叠图标收起该分类：图标切为收起态且其子级隐藏
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m再点一次恢复展开（收起可逆）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m多开：收起一个不影响其他分类的展开态
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m非顶级节点不渲染折叠图标（子目录/菜单不参与折叠）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m无子级的顶级节点不渲染折叠图标（防御性）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

[90mstderr[2m | tests/unit/views/system/role-tree-collapse.spec.ts[2m > [22m[2m角色授权树顶级分类折叠[2m > [22m[2m点击折叠图标不会触发复选框勾选（click.stop 防回归）
[22m[39mWarning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?
Warning: [ant-design-vue: Checkbox] `value` is not validate prop, do you mean `checked`?

 [32m✓[39m tests/unit/views/system/role-tree-collapse.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[33m 1230[2mms[22m[39m
   [33m[2m✓[22m[39m 角色授权树顶级分类折叠[2m > [22m默认全部展开：每个顶级分类都渲染折叠图标（收起态图标不出现） [33m496[2mms[22m[39m
 [32m✓[39m tests/unit/views/survey/province-match.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 8[2mms[22m[39m
 [32m✓[39m tests/unit/survey/write-storage.spec.ts [2m([22m[2m15 tests[22m[2m)[22m[90m 16[2mms[22m[39m
 [32m✓[39m tests/unit/display/metric-utils.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/api/iot/point-api.spec.ts [2m([22m[2m11 tests[22m[2m)[22m[90m 14[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/actuator-api.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 17[2mms[22m[39m
 [32m✓[39m tests/unit/survey/wx-auth.spec.ts [2m([22m[2m18 tests[22m[2m)[22m[90m 65[2mms[22m[39m
 [32m✓[39m tests/unit/layout/top-expand-theme.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[33m 788[2mms[22m[39m
   [33m[2m✓[22m[39m 分组布局一级菜单主题联动（top-menu.vue）[2m > [22m学术蓝（sideMenuTheme=light）→ 菜单渲染为 light，不含 dark [33m428[2mms[22m[39m
 [32m✓[39m tests/unit/views/survey/write-time-window.spec.ts [2m([22m[2m11 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/survey/option-label-display.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[90m 206[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-card-status-values.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[33m 1101[2mms[22m[39m
   [33m[2m✓[22m[39m 工厂卡片接状态数真值（TASK-0058 批次二 D2）[2m > [22m挂载即用 node.id 取 factory-summary，运行/异常/离线三槽位填真值 [33m563[2mms[22m[39m
 [32m✓[39m tests/unit/display/device-utils.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/store/ui-config-merge.spec.ts [2m([22m[2m9 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/views/survey/redpack-columns.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[33m 3031[2mms[22m[39m
   [33m[2m✓[22m[39m 问卷红包列表「发放人/发放结果」两列[2m > [22m列定义含「发放人」与「发放结果」两列，既有列不丢 [33m1779[2mms[22m[39m
   [33m[2m✓[22m[39m 问卷红包列表「发放人/发放结果」两列[2m > [22m发放人列取 sendByName，缺失显示「—」（未发放行/老后端行） [33m778[2mms[22m[39m
   [33m[2m✓[22m[39m 问卷红包列表「发放人/发放结果」两列[2m > [22m发放结果列五态文案与配色（成功绿/失败红/发放中蓝/待确认橙/未发放默认） [33m471[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/point-bind-drawer.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[33m 1979[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉绑定点位入口（TASK-0092）[2m > [22m「绑定点位」按钮挂 iot:item:edit 权限；无该权限时按钮被移除 [33m991[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉绑定点位入口（TASK-0092）[2m > [22m行内「绑定点位」按钮点击 → 拉启用点位列表 → 选点 → bindPoint → 刷新并显示点位编码 [33m986[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m改动配置后按作用域上报服务端（模块作用域）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m无当前模块时上报到账户全局（moduleId=0）
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m内部态变化（remoteItems/remoteLoaded）不上报，防模块间配置串写
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/monitor/actuator-helper.spec.ts [2m([22m[2m12 tests[22m[2m)[22m[90m 50[2mms[22m[39m
 [32m✓[39m tests/unit/survey/option-label-map.spec.ts [2m([22m[2m10 tests[22m[2m)[22m[90m 9[2mms[22m[39m
[90mstderr[2m | tests/unit/layout/header-setting-persist.spec.ts[2m > [22m[2m设置抽屉界面偏好持久化[2m > [22m[2m恢复默认配置：删除当前作用域远端配置
[22m[39m[Vue warn]: Failed to resolve component: CheckSquareFilled
If this is a native custom element, make sure to exclude it from component resolution via compilerOptions.isCustomElement. 
  at <HeaderSetting ref="VTU_COMPONENT" > 
  at <VTUROOT>

 [32m✓[39m tests/unit/layout/header-setting-persist.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 734[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/cache-monitor-api.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 18[2mms[22m[39m
 [32m✓[39m tests/unit/display/device-realtime-zone.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[33m 1238[2mms[22m[39m
   [33m[2m✓[22m[39m 设备实时区接 realtime 真值（TASK-0089 D2）[2m > [22m绑定类 + 公式类统一取 realtime 真值（v/q/ts 同源），来源标注「实时值读口」 [33m661[2mms[22m[39m
 [32m✓[39m tests/unit/api/iot/metric-api.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/role-list-selection.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[33m 1403[2mms[22m[39m
   [33m[2m✓[22m[39m 角色列表刷新后选中态（组件级）[2m > [22m首次加载自动选中第一个角色 [33m579[2mms[22m[39m
   [33m[2m✓[22m[39m 角色列表刷新后选中态（组件级）[2m > [22m编辑提交触发刷新后，仍选中原来那个角色（不跳回超级管理员） [33m385[2mms[22m[39m
   [33m[2m✓[22m[39m 角色列表刷新后选中态（组件级）[2m > [22m删除的正是当前选中角色时，回落到第一个 [33m429[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-summary-read-port.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-operate-modal.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[33m 4425[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m编辑回显：库中「不显示」的菜单开关为不显示，未动开关提交不翻转原值 [33m1493[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m关「显示状态」提交 visible=0（用户实测场景：修复前后端只收到旧值 1） [33m1031[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m关「菜单状态」提交 status=0，且可见性保持显示 [33m746[2mms[22m[39m
   [33m[2m✓[22m[39m 菜单编辑弹窗 visible/status 映射[2m > [22m新增菜单未动开关时提交 visible=1、status=1 [33m1153[2mms[22m[39m
 [32m✓[39m tests/unit/survey/redpack-helper.spec.ts [2m([22m[2m8 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/views/wechat/tag-filter.spec.ts [2m([22m[2m13 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/survey/redpack-list-helper.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/cache-helper.spec.ts [2m([22m[2m11 tests[22m[2m)[22m[90m 8[2mms[22m[39m
 [32m✓[39m tests/unit/store/dict.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 21[2mms[22m[39m
 [32m✓[39m tests/unit/survey/expand-original-data.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-tree.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 13[2mms[22m[39m
 [32m✓[39m tests/unit/utils/file-url.spec.ts [2m([22m[2m7 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-list-perms-column.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/theme/theme-preset.spec.ts [2m([22m[2m13 tests[22m[2m)[22m[90m 12[2mms[22m[39m
 [32m✓[39m tests/unit/store/app-config.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[90m 286[2mms[22m[39m
 [32m✓[39m tests/unit/survey/normalize-field-schemes.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 8[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/menu-field-mapping.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/api/survey/edit-lock-api.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/api/iot/point-bind-api.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-routes.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[90m 9[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/iot-inspect-helper.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 8[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/login-entry-render.spec.ts [2m([22m[2m1 test[22m[2m)[22m[33m 509[2mms[22m[39m
   [33m[2m✓[22m[39m 登录页首屏组件渲染（入口精简集）[2m > [22m只注册 entryComponents 时，用户名/密码/验证码三件套都能渲染 [33m507[2mms[22m[39m
 [32m✓[39m tests/unit/plugin/entry-ui.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[33m 439[2mms[22m[39m
   [33m[2m✓[22m[39m plugin/entry-ui 入口精简集[2m > [22m登录表单（a-form / a-form-item / a-input-password）能渲染出密码输入框 [33m432[2mms[22m[39m
 [32m✓[39m tests/unit/monitor/alarm-helper.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-page-metrics.spec.ts [2m([22m[2m3 tests[22m[2m)[22m[33m 650[2mms[22m[39m
   [33m[2m✓[22m[39m 工厂页接工厂级指标真值（TASK-0058 批次二 D2）[2m > [22m工厂级指标看板：metricValues 同源展示 value/yoy/mom（不现算、null 不冒充 0） [33m343[2mms[22m[39m
 [32m✓[39m tests/unit/wechat/send-common.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 11[2mms[22m[39m
 [32m✓[39m tests/unit/survey/serial-number.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[90m 6[2mms[22m[39m
 [32m✓[39m tests/unit/lib/encrypt.spec.ts [2m([22m[2m4 tests[22m[2m)[22m[90m 10[2mms[22m[39m
 [32m✓[39m tests/unit/display/factory-card-summary.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 230[2mms[22m[39m
 [32m✓[39m tests/unit/views/iot/instance-item-drawer-ref.spec.ts [2m([22m[2m1 test[22m[2m)[22m[33m 594[2mms[22m[39m
   [33m[2m✓[22m[39m 实例数据项抽屉父组件 ref 暴露（BUG-0008）[2m > [22m父组件 ref 可拿到 showDrawer/onClose；调用后抽屉打开且标题可见 [33m592[2mms[22m[39m
 [32m✓[39m tests/unit/layout/module-switcher.spec.ts [2m([22m[2m6 tests[22m[2m)[22m[33m 316[2mms[22m[39m
 [32m✓[39m tests/unit/views/system/login-captcha-polling.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[33m 400[2mms[22m[39m
 [32m✓[39m tests/unit/display/display-slot-card.spec.ts [2m([22m[2m2 tests[22m[2m)[22m[90m 120[2mms[22m[39m
 [32m✓[39m tests/unit/composables/survey/use-project-creator.spec.ts [2m([22m[2m5 tests[22m[2m)[22m[90m 3[2mms[22m[39m

[2m Test Files [22m [1m[32m84 passed[39m[22m[90m (84)[39m
[2m      Tests [22m [1m[32m713 passed[39m[22m[90m (713)[39m
[2m   Start at [22m 21:14:10
[2m   Duration [22m 67.28s[2m (transform 24.63s, setup 0ms, collect 477.81s, tests 87.84s, environment 141.32s, prepare 30.02s)[22m

```
