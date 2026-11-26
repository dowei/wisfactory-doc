| 编号 | 名称 | 所属模块 | 相关功能 | 测试用例 | 状态 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| BUG-0001 | application-prod.yml 顶层 ot 键重复致 prod 档启动失败 | 管理系统（Java） | FEAT-0001 | TC-0022 | 已完成 | P1 |
| BUG-0002 | TDengine 子表名转义映射不单射致不同点位编码写入同子表 | 采集接入（Go） | FEAT-0009 | TC-0023 | 已完成 | P1 |
| BUG-0003 | 未认证失败请求触发 OperateLog 写库撞 user_id 非空约束 | 管理系统（Java） | FEAT-0001 | TC-0024 | 已完成 | P2 |
| BUG-0004 | TDengine st_metric_value 建表与指标查询 period 保留字裸列名被拒 | 管理系统（Java） | FEAT-0001 | TC-0025 | 已完成 | P2 |
| BUG-0005 | 内部快照拉取端点出站体再序列化致下游不可重算 snapshotHash | 管理系统（Java） | FEAT-0008 | TC-0026 | 已完成 | P1 |
| BUG-0006 | 运营方角色缺 iot:tenant:admin 授权导致无法初始化租户管理员 | 管理系统（Java） | FEAT-0001 | TC-0001 | 已完成 | P1 |
| BUG-0007 | AlarmTimeoutScanner 裸 int 构造器致后端 dev 启动失败 | 管理系统（Java） | FEAT-0020 | 不适用：启动类缺陷（Spring 上下文装配，单测直接 new 构造器不覆盖） | 已完成 | P1 |
| BUG-0008 | 实例数据项按钮无法打开抽屉（showDrawer 未 defineExpose） | 前端（Vue 3） | FEAT-0018 | 不适用：组件 ref 调用缺陷（单测测内部函数，未测父组件 ref 调用） | 已完成 | P1 |
| BUG-0009 | 实例数据项无法绑定点位（前端缺 PUT /iot/item/bind 入口） | 前端（Vue 3） | FEAT-0018 | TC-0002（绑定/换绑点位） | 已完成 | P1 |
BUG-0005 占位
