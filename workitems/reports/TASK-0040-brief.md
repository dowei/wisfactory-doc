# 实施 brief：TASK-0040（依赖图构建与快照消费切换——计算引擎服务引导）

## 批次

- TASK-0040（本批共 1 个，标准任务 5 人天；**计算引擎道开线**——独立顶层 Maven 工程，与 backend 道互不撞车〔独立 mvn〕；前置 0026/0036 均已完成）
- BASE：以开工时主仓 HEAD 为准（`git rev-parse --short HEAD` 记录进报告）
- 改动范围（越界即违约）：
  - `calc/**`（**新顶层目录**，与 backend/frontend/ingest 平级——目录规划裁决：独立 Maven 工程 Spring Boot 应用，go/ingest 同款平级先例）
  - `resources/scripts/`（如需临时脚本）
  - 不碰 backend/**、frontend/**、ingest/**、workitems（报告除外）、契约文件

## 必读（只给路径，不粘正文）

- TASK 详情：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\tasks\TASK-0040.md`
- 决策：workitems 库 `features\FEAT-0013.md`、`requirements\REQ-0020.md`、`testcases\TC-0013.md`、`decisions\ADR-0008.md`（快照按实例有效版本组装）
- 契约权威：`D:\workspace\WisFactory\resources\contract\config-snapshot-schema.md` v1.0.1（五对象：点位/通道/数据项/公式/指标——计算范围消费数据项/公式/引用边/指标）
- 上游契约源（开工先读）：backend/ot-admin/src/main/java/net/otlab/admin/module/openapi/** 与 iot/publish/controller/IotPublishInternalController.java（GET /internal/v1/config/snapshot 响应形态——BUG-0005 修复后原始字节可重算）+ iot/publish/service/impl/IotPublishServiceImpl.java（target 派生：scope_type=2 → 单行 [3/calc]——ACK 回执对象）+ iot/publish/controller/IotPublishInternalController.java（POST /internal/v1/config/ack）
- 详细设计：§3.3 p25（计算引擎消费方式：按租户拉取计算范围快照重建依赖图；新图构建成功才切换，构建失败保持旧图运行**并回执失败**；配置错误不许打停正在跑的计算）+ §1.5B（状态规则表）
- 平台先例（范式基准）：`D:\workspace\WisFactory\ingest\README.md`（独立服务骨架先例/假实现零改动替换/契约测试命名 Contract）+ ingest/internal/snapshot（拉取客户端+原子替换先例——Java 版同语义）
- 框架纪律：TD-0058（Spring Boot 4 = Jackson 3，tools.jackson API）；Java 21

## 接口与决策（逐条结论，不再讨论）

1. **目录与骨架**：`calc/` 新顶层 Maven 工程（artifact calc-engine，包 net.otlab.calc）；Spring Boot 4 + Java 21；README.md 照 ingest README 体例（目录/流水线/边界）；`cmd` 形态=标准 Spring main；配置 env（`CALC_HTTP_ADDR` 默认 `:8095`——端口纪律 8098/8099/8090 已占，启动前探测、用毕即停；`CALC_BACKEND_BASE_URL`、`CALC_INTERNAL_TOKEN`、`CALC_TENANT`——一期单租户形态与 ingest 同）。git 提交只 add calc/**（绝不 git add -A）。
2. **快照消费**：内部令牌 HTTP 客户端拉 `GET /internal/v1/config/snapshot?scopeType=2&scopeId=0`（计算范围=租户粒度）——**哈希可重算校验**（BUG-0005 修复后出站体=原始字节，重算 SHA-256 == snapshotHash，不一致 fail-closed 拒绝应用）；原子替换语义照 ingest snapshot 包（新图成功才换引用）。
3. **依赖图构建**：从快照五对象消费数据项/公式/引用边/指标——节点=数据项/指标全键，边=引用（公式→上游数据项/指标；汇总→子级）；**构建失败主因覆盖**：悬空引用（引用目标不在快照）/实例化失败（公式无法实例化为可执行结构——本期 FormulaFunctionWhitelist 占位语义：DELTA 可引用/INTEGRAL 不开放，真 Aviator 语法归 FEAT-0012——本期「实例化」=表达式结构可解析+白名单校验）/循环依赖（引用边环检测——后端 0036 FormulaCycleDetector 同款语义，快照侧再验一道防御纵深）；失败 → 保持旧图运行 + **ACK 回执失败**。
4. **ACK 回执**：构建成功/失败 → `POST /internal/v1/config/ack`（targetType=3、targetId=calc〔与 0025 派生口径一致——配置化 `CALC_TARGET_ID` 默认 calc〕、ackStatus=1/2、errorMsg）——这是计算引擎与发布链的回执闭环（0025 交付的表与端点首次被真实消费）。
5. **切换语义**：新图构建成功才原子切换（引用替换）；失败保持旧图持续可用；配置错误不打停计算（本期无执行器——语义在图层确立，javadoc 声明 0041 承接）。
6. **config.changed 消费缝**：NotifySender 真联调未通（TD-0056）——本期拉取驱动（启动全量拉取+可配对账周期 `CALC_SNAPSHOT_RECONCILE_SECONDS` 默认 300s 定时对账）+ config.changed 订阅缝接口占位（TD-0057 延续登记）。
7. **不碰/不做**：执行器/算子（0041/0042）、iot_calc_log/state 写（0041+/0063）、st_metric_value（0045）、状态判定（0049）、backend/ingest 任何文件、契约冻结。

### 测试口径（TDD）

契约测试（快照 fixture 样例逐字段取自 config-snapshot-schema v1.0.1——Contract 命名照 ingest 范式）；依赖图构建（正常图/悬空引用失败/循环失败/实例化失败/空快照）；原子切换（成功换/失败保旧）；哈希可重算校验（错哈希 fail-closed）；ACK 回执（成功 1/失败 2 带 errorMsg——httptest 假端点）；对账周期。红转绿留痕。

## 验收命令

环境事实（协调者 2026-11-23 实测）：backend dev 可起（8098，0015/0016 冒烟数据在库——iot_default scope_type=2 有真实快照可拉）；mysql/redis 容器在跑；8095 归 calc；WSL2 空闲关停→长间隙前保活循环。

1. `cd calc && mvn install && mvn test` → BUILD SUCCESS、0 失败 0 错误（新工程首跑 Results 段贴回）
2. **live 真链冒烟**（每步原始输出贴回）：backend dev 起（8098）→ calc 起（8095）→ 快照真拉取（iot_default scope_type=2，哈希重算一致）→ 依赖图构建成功 → **ACK 回执真实落库**（sql-exec.sh 查 iot_config_publish_log ack_status=1 行）→ 后端造坏快照（SQL 改版本行 content 或临时挡端点——任选可行法）→ 拉取/构建失败 → 旧图保持 + ACK 回执失败（ack_status=2+errorMsg）→ 全停归零
3. 范围核对：`git diff --name-only <BASE>..HEAD` 全落 calc/**（+如 README）
4. 清理：无 scratch 库（本批不写 TDengine）；dev 库不动数据

## 报告

报告写到：`D:\Documents\dowei\2.项目资料\智慧工厂\workitems\reports\TASK-0040-report.md`。五项齐全 + TD 登记（config.changed 订阅缝）+ 一句话结论。

## 契约

- no-subagents；只改 calc/** 与资源脚本；提交只 add calc/**（绝不 git add -A）；不把 TASK 置「已完成」；不改 workitems 库（唯一例外：写报告）；契约冻结发现问题报告并停。
- TDD 红转绿，每个修复附「为什么现在对了」。
