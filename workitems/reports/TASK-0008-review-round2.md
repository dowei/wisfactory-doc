# TASK-0008 修订轮 scoped 复评报告（fresh reviewer）

- 复评对象：`reports/TASK-0008-fixdiff.txt`（提交 `d971c10`，2 文件：IotDataDictServiceImpl.java + IotDataDictServiceImplTest.java，与报告申报一致）
- 依据：复评 brief（判定标准 + 协调者裁决口径）、原评审报告（I-1 / Minor 3）、实施报告修订轮一节（TDD 红→绿、全量 1135、dev 冒烟 R1-R3b 与 DB ghost_rows=0）
- 范围纪律：只判原 I-1 是否修复、修复本身有无新缺陷；Minor 3 不在判定范围，未重开全场

## 结论：I-1 已修复，新发现 0

## 判定理由（一句）

insertDict / updateDict / disableDict 三条写路径现全部由服务层白名单构建落库实体（insert 强制 status=1 / deleted=0、tenant_id 会话覆盖、id 与审计字段留给框架；update 更新集仅 id+dataName/category/defaultUnit；disable 仅 id+status=0），控制器五端点之外无任何其他写入通道（mapper 空 BaseMapper、无 XML、全仓无 deleteById/deleteByIds 触达 iot_data_dict），故任何 POST/PUT 载荷组合下落库行不可能出现 deleted=1 或非预期 status，幽灵行永久占用 uk_tenant_data_code 的路径已被封死。

## 核对证据（scoped，简列）

- **白名单完整性（insert）**：实体字段全集 = id/tenantId/dataCode/dataName/category/dataType/defaultUnit/status + BaseEntity(createBy/createTime/updateBy/updateTime/deleted @TableLogic)。修复后 insertDict 对每个字段均有处置：五业务字段照请求复制、status=1、deleted=0、tenantId=会话、id/审计字段置空由框架填充（MybatisPlusConfig strictInsertFill 仅填空值，已核源码 63-66 行）——无遗漏字段，请求体不再有直达持久层的通道。
- **白名单完整性（update）**：更新集仅 id（取自库内 existing，非请求）+ 三个可变字段；dataCode/dataType 经不可改校验后不进 SET（比原「库内值回写 no-op」更收紧）、status/deleted/tenantId/审计字段维持排除；校验顺序未变（存在性 → 编码锁 → 类型锁），dataCode/dataType 未携带或不一致仍 PARAM_ERROR 拒绝。与裁决口径逐字吻合。
- **无旁路**：控制器仅 list/getById/add/edit/remove 五端点；全仓 grep IotDataDict 引用仅实体/mapper/service/controller 五个主代码文件；mapper 为空 BaseMapper 无 XML；iot 模块唯一 removeByIds 命中在 SysTenantServiceImpl（sys_tenant，无关）。
- **测试未放松**：`testUpdateDict_WhitelistFieldsSaved` 的 dataCode/dataType 断言由「库内值回写」改「不进更新集」是加强而非放松（不可变字段连 no-op 都不写）；其余既有断言全数保留；新增 3 例攻击向量测试均显式携带 status=0/deleted=1/id=888/审计字段，断言落库白名单值且幽灵重建走预检闸 4002。TDD 红→绿证据链完整（红时 status 落库 0、绿后强制 1）。
- **运行证据自洽**：全量 1135 = 基线 1105 + 首轮 27 + 修订轮 3（数学闭环，0 失败）；dev 冒烟 R1/R1b（POST 携带 status=0/deleted=1 落库 status=1/deleted=0）、R2（同码重建 4002 走预检=行可见可排查）、R3/R3b（PUT 携带 status=1/deleted=1 状态不变未复活）+ DB 层 ghost_rows=0 四角互证。
- **改动范围**：`git show d971c10 --name-only` 实核恰 2 文件，严格限裁决范围；Minor 3 项（分页上限 / data_code 字符集 / 租户隔离单测）未被本轮触碰，无恶化。
