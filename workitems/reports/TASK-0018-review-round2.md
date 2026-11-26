# TASK-0018 复评报告（修订轮 scoped · fresh reviewer）

- 复评对象：修复提交 `fa12df9`（3 文件 +92/-27：ChannelConnConfigValidator.java + 其测试 + IotChannelServiceImplTest 连带断言对齐）
- 复核材料：评审 brief（review2）、原评审 TASK-0018-review.md、修复 diff TASK-0018-fixdiff.txt（218 行全读）、实施报告修订轮节（R-1～R-5）、Validator 现文逐行核对、独立复跑测试（见下）

## 结论

- **I-1 已修复**：`requireNoMissing` 改为值感知判定——键存在且值非 null 且字符串非空白，与协调者裁决口径逐字一致（`ChannelConnConfigValidator.java` L239-251），并前移至凭据扫描之后、结构校验之前（L113）；`{"broker":null}` / `{"broker":"   "}` / `{"broker":""}` 均按「缺少必填字段: [broker]」拒，不再被 format/unknown-key 掩盖；空白入参路径改传空 ObjectNode，语义与原 `Set.of()` 等价。正常值不误伤：非空字符串/整数/嵌套对象照常过，独立复跑 45 用例全绿 + 实施报告全套件 1673 绿 + 冒烟 R5a 正常 MQTT 创建成功。
- **I-2 已修复**：`collectCredentialKeys` 重写为全深度递归——标量（isValueNode）终止、数组元素逐个递归（`path[i]`）、对象扫本层键名并递归全部成员值（L270-291）；`creds[0].password`、`signals[0].group[0].secret` 类形状拒且路径可读（新用例 2 条红转绿 + 冒烟 R4a/R4b HTTP 层实证）。正常嵌套业务键不误伤：仅键名子串匹配，值不扫（authType 枚举值 "apiKey" 不受影响）。

## 新发现

- 无（本轮判定范围内）。
- 核验说明（非发现）：① 递归深度限制——conn_config 8192 字符上限 + Jackson（Spring Boot 4.1.0 → ≥2.15）`readTree` StreamReadConstraints 嵌套深度默认上限 1000 双重封顶，全深度递归无栈溢出新面；② 判定顺序——凭据扫描 → required → 结构校验，required 前移不弱化拒绝语义（未知字段与缺必填并存时改报缺必填，仍 PARAM_ERROR 拒）；③ 断言放松核查——两处既有断言调整（testMqttBrokerNotUri 空串段、testUpdate_ProtocolChangeRevalidatesConnConfig）属排序变化的语义对齐而非放松，均仍断言拒绝且报文更直指；④ 原 5 条 Minor 未恶化（M-2 大小写接缝、M-4 无缓存等与本次改动无交集）。

## 复核记录

```
mvn test -pl ot-admin -Dtest='ChannelConnConfigValidatorTest,IotChannelServiceImplTest'
Tests run: 21, Failures: 0 -- ChannelConnConfigValidatorTest
Tests run: 24, Failures: 0 -- IotChannelServiceImplTest
BUILD SUCCESS
```

## 判定理由（一句）

两项 Important 均按裁决口径修复、各自补齐红转绿用例并经全套件与 dev 冒烟双印证，修复实现（含递归深度、判定顺序、断言对齐）经现文逐行核对与独立复跑未发现新缺陷，复评通过。
