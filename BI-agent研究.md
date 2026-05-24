# BI × Agent 研究：从现代数据栈到 WrenAI

> 整合自 `BI_SEMANTIC_LAYER_GITHUB_PROJECTS.md`、`BI-report.md`、`BI-original-report.md`、`BI-research-report.md`，并据 WrenAI 主分支（2026）更新 Agent 定位。  
> **叙事主线**：数据栈自下而上沉淀「可查的表 → 统一的指标语义 → 给人看的 BI → 给 Agent 的受控问数」；WrenAI 是最后一环的消费者与护栏，不是 dbt/BI 的替代品。

---

## 0. 叙事逻辑（读前先看）

### 0.1 核心问题

当 **AI Agent** 进入分析链路，瓶颈从「能不能画图」变成：

1. **物理层**：库里有没有干净、主题化的表（常由 dbt 或等价 ETL 承担）  
2. **语义层**：指标口径、join 路径是否一处定义（MetricFlow、Cube、MDL 等）  
3. **消费方分裂**：**人**走 BI 看板；**Agent** 走 API/语义查询——不宜裸扫全库  
4. **准确性**：纯 text-to-SQL 在真实 BI 场景有天花板；**收窄查询空间**（语义层）是结构性解法  

### 0.2 统一架构图

```text
原始数据
  → 数据建模（dbt-core：staging / intermediate / marts）
  → 指标 / 语义层（MetricFlow、Cube、LookML、MDL / WrenAI …）
  → 数据目录 / 质量（DataHub、OpenMetadata、Elementary …）
        ├→ BI（Superset、Metabase、Looker …）────────→ 人
        └→ Agent 问数层（WrenAI、Cube API、Vanna …）──→ Agent / 嵌入应用
```

### 0.3 关于 WrenAI 版本

- **主分支（2026）**：**Agent 上下文层**（MDL、Engine、Skills、`wren-langchain` / `wren-pydantic`）；旧版 GenBI 应用在 `legacy/v1`。  
- 下文「WrenAI」均指 **新版 Agent-native 定位**，除非特别说明。

### 0.4 文档结构

| 章节 | 内容 |
|------|------|
| §1 | 历史：BI 1.0 → Agent 时代 |
| §2 | 背景：dbt 与物理建模 |
| §3 | 语义层生态与 2025 仓库战争 |
| §4 | BI：给人看的分析 |
| §5 | 治理、质量与 text-to-SQL 争论 |
| §6 | **重点：WrenAI** |
| §7 | 开源项目速查与研究路径 |
| §8 | 结论 |

---

## 1. 历史演进：从 IT 报表到 Agent 分析

### 1.1 阶段概览

| 阶段 | 时期 | 特征 | 代表 |
|------|------|------|------|
| **BI 1.0** | 1990s | IT 中心化报表、批处理 | Cognos、Oracle BIEE |
| **BI 2.0** | 2000s | 自助 BI、拖拽、内存计算 | Tableau、Power BI、Qlik；Metabase、Superset |
| **BI 3.0 / MDS** | 2015s+ | 云仓、dbt、Headless 语义层、BI as Code | Looker、Cube、Evidence、Rill |
| **Agent 分析** | 2023+ | NL 问数、MCP、语义层成 AI 基础设施 | WrenAI、仓库内置 Semantic/Metric Views |

### 1.2 里程碑（简表）

- **1865 / 1958**：「商业智能」概念与计算机结合（Devens、Luhn）。  
- **1980s**：关系型数据库、数据仓库、OLAP。  
- **2000s**：Tableau（2003）等降低可视化门槛。  
- **2010s**：Hadoop/Spark、Snowflake/BigQuery、**dbt**、开源 BI 爆发。  
- **2020–2022**：MetricFlow（后被 dbt Labs 收购）、Cube、Lightdash；DataHub、OpenMetadata。  
- **2023–2024**：LLM 驱动 NL2SQL；dbt Semantic Layer **2024-10 GA**；各商业 BI 集成 Copilot。  
- **2025+**：Gartner 等判断 **语义层成为 AI 栈常见组件**；Snowflake / Databricks **内置语义对象**；**text-to-SQL vs text-to-Semantic-Layer** 成为架构争论焦点。

### 1.3 三条演进逻辑

1. **架构可组合化**：从单体 BI 到「建模 + 语义 + 展示 + Agent」分层。  
2. **决策近实时**：从月报到近实时看板（Grafana、Rill 等偏运营/时序）。  
3. **计算下推**：聚合与建模 increasingly 在数仓完成，BI/Agent 消费成品。

### 1.4 市场背景（为何开源 BI 仍重要）

- 2024 年全球分析与 BI 软件市场约 **203 亿美元**，预计 2029 年约 285 亿美元（CAGR ~7%）。  
- 2019 年巨头并购（Tableau、Looker 等）说明：**可视化能力商品化**，竞争转向生态与数据平台。  
- 2025 年 Salesforce 收购 Informatica 等动向表明：**治理 + AI Agent 平台** 成为新战场。  
- 开源机会在于 **无厂商锁定、可组合**；与商业 BI 是不同价值主张，非单纯功能对标。

---

## 2. 背景层：dbt 与物理建模

> **本章回答**：表从哪来？没有 dbt（或等价 ETL）时，语义层和 Agent 面对什么风险？

### 2.1 dbt 是什么（名词）

| 名词 | 含义 |
|------|------|
| **dbt Labs** | 公司 |
| **dbt-labs** | GitHub **组织**（无单一 `dbt-lab` 总仓库） |
| **dbt / dbt-core** | 开源核心：[dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core) |
| **dbt-native** | 工具原生围绕 dbt 项目（如 Lightdash） |

**MetricFlow**：[dbt-labs/metricflow](https://github.com/dbt-labs/metricflow) 与 dbt-core **同组织、不同仓库**。**dbt Semantic Layer** 是产品名，引擎侧主要是 MetricFlow，**没有**名为 `semantic-layer` 的独立总仓库。

### 2.2 分层与目录

```text
raw → staging/ → intermediate/ → marts/
```

| 目录 | 作用 | 例子 |
|------|------|------|
| `staging/` | 贴源清洗、统一类型 | `stg_orders` |
| `intermediate/` | 中间 join/规则，供复用 | `int_orders_enriched` |
| `marts/` | 主题成品，给 BI/指标/问数 | `mart_revenue_daily` |

**价值**：把「物理表长什么样」整理成「业务可查的表」。无此层，语义层与 Agent 直接面对混乱 raw schema。

### 2.3 与 WrenAI 的关系

- dbt **不负责** Agent 自然语言接口，也 **不提供** 拖拽看板。  
- **可以没有 dbt**：用脚本/厂商库建仓 + WrenAI MDL 仍可行（如已结构化的财报库）；工程压力在 **导入质量 + MDL**，不在 dbt 本身。  
- **推荐生产栈**：dbt marts → 语义层（MetricFlow/Cube/**WrenAI MDL**）→ Agent。

---

## 3. 语义层：定义、竞品与 2025 博弈

### 3.1 Semantic Layer 是什么

**语义层**不是某个 GitHub 仓库名，而是**一层能力**：统一指标名、维度、join；查询时由引擎生成 SQL，避免 BI 与 Agent 各写一套。

**dbt、Cube、Looker 在语义层上重叠，但不可互换：**

| 产品 | 主要能力 | 组织 |
|------|----------|------|
| **dbt + MetricFlow** | 数仓建模 + 指标 YAML → SQL | dbt Labs |
| **Cube** | Headless 语义层 + REST/GraphQL/SQL API | [cube-js](https://github.com/cube-js)（**非** dbt Labs） |
| **Looker** | LookML 语义层 + BI 一体 | 商业 |
| **Lightdash** | dbt-native BI + metrics | 开源 |
| **Malloy** | 语义建模语言 + Publisher（REST/MCP） | malloydata |

**WrenAI 与 Cube/MetricFlow 的重叠**：在「指标/关系/受控 SQL」上承担 **部分语义层**工作；**不替代** dbt 的批量物理建模（除非你用别的方式建仓）。

### 3.2 与 BI 的衔接（三种组合）

```text
源系统 → dbt-core（marts）
           ↓
    语义层（MetricFlow / Cube / LookML / MDL）
           ↓
    BI（Superset / Metabase …）→ 人
           ↘
    WrenAI / Vanna … → Agent
```

| 模式 | 链路 | 特点 |
|------|------|------|
| **简配** | marts → BI 直连表 | 常见；口径易散 |
| **标准配** | marts → 语义层 → BI 查指标 | 指标一处定义 |
| **一体化** | marts → Looker / Lightdash | 语义+看板一家 |

**依赖关系（选型）：**

| 问题 | 答案 |
|------|------|
| 没有 dbt 能有 BI 吗？ | 能 |
| 没有 BI 能有语义层吗？ | 能（API/Agent） |
| dbt 能替代 Superset 吗？ | **不能** |
| BI 能替代 dbt 吗？ | **不能** |
| 语义层能替代 BI 吗？ | 大部分不能（Looker/Lightdash 例外） |

### 3.3 人 vs Agent：谁消费哪一层

- **BI**：图表、看板、订阅——服务「要以可视化消费数据」的人（含分析师，不限于「不懂财务」）。  
- **Agent**：要 **可执行边界**（允许哪些模型/指标、join、策略、校验）。  

```text
dbt marts ──→ BI ──→ 人
     └──→ 语义/MDL（WrenAI、Cube、MetricFlow …）──→ Agent
```

WrenAI **不是** marts 的唯一出口；与 BI **并行**。

### 3.4 2025 仓库原生语义层（行业变量）

Snowflake **Semantic Views**（2025 GA）、Databricks **Metric Views** 等，把实体/指标放进**仓库内元数据**，诉求是零外部依赖、SQL 客户端即用。

**与独立语义层矛盾**：仓库厂商赌「语义住在仓里」；弱点是 **多云/多仓** 与跨平台复用——故 Snowflake 推动 **OSI（Open Semantic Interchange）** 等互操作倡议（仍早期）。

| 方案 | 多仓库 | 零外部依赖 | 开源 | AI 集成 |
|------|--------|------------|------|---------|
| dbt MetricFlow | ✅ | ❌（常需 Cloud） | ✅ | 中 |
| Snowflake Semantic Views | ❌ | ✅（仓内） | ❌ | 强（Cortex） |
| Databricks Metric Views | 有限 | ✅（平台内） | 有限 | 强（Genie） |

### 3.5 语义层碎片化

Snowflake Semantic Views、Databricks Metric Views、dbt metrics、Cube schema、Lightdash YAML、**WrenAI MDL**、LookML **互不兼容**。指标在 dbt 定义后不能直接在 Snowflake Semantic Views 复用——**语义模型可能成为新孤岛**；OSI 等标准值得 2026+ 跟踪。

### 3.6 主要开源语义层项目（简表）

| 项目 | GitHub | 要点 |
|------|--------|------|
| MetricFlow | [dbt-labs/metricflow](https://github.com/dbt-labs/metricflow) | 指标→SQL；偏 dbt 生态 |
| Cube | [cube-js/cube](https://github.com/cube-js/cube) | API、缓存、预聚合；Cube 0.35+ WASM SQL、Authz SDK |
| Lightdash | [lightdash/lightdash](https://github.com/lightdash/lightdash) | dbt-native BI；2025 AI Agent/Slack |
| Malloy | [malloydata/malloy](https://github.com/malloydata/malloy) | 语言级语义 + MCP |

---

## 4. BI 层：给人看的分析

> **本章回答**：BI 做什么？与 dbt、语义层、WrenAI 如何分工？

BI 的核心是 **探索、图表、Dashboard、权限与协作**，面向分析师与业务用户。与 dbt/MetricFlow/Cube **不同层**；多数消费 **marts 表**（简配）或 **语义层指标**（标准配）。

### 4.1 工具速览

| 工具 | Stars（约） | 定位 | 语义层 | 备注 |
|------|-------------|------|--------|------|
| **Superset** | ~72k | SQL-first 企业 BI | 轻量 | 强可视化；不适合作复杂指标样本 |
| **Metabase** | ~44k | 低门槛自助 BI | 弱 | 业务用户友好 |
| **Grafana** | ~71k | 可观测/时序 | ✗ | 运维监控，非典型经营 BI |
| **Redash** | ~28k | SQL-first 轻量 | ✗ | 原版停维；社区分支仍维护 |
| **Rill** | ~5k | BI-as-Code、运营看板 | YAML metrics | DuckDB/ClickHouse 极速 |
| **Evidence** | ~6k | SQL+Markdown 报表 | ✗ | 静态/站点化数据产品 |
| **Lightdash** | ~5k | dbt-native BI | dbt metrics | 介于语义层与 BI |

**低代码内部工具**（Appsmith、Openblocks）：偏 Retool 类内部应用，研究 BI 核心时优先级低于上表。

**补充（原调研提及）**：Observable Framework（JS 响应式）、GoodData.CN（Headless、多租户）——偏特定场景。

### 4.2 Excel 仍未被替代

财务等领域 **Excel** 仍因可确定性、What-if、零部署而顽固；Copilot 等在强化 Excel 而非取代它。BI 若在「即时操控」上接不上，很难仅靠「功能更多」替换 Excel。

### 4.3 BI 与 WrenAI

- BI **主要给人**；WrenAI **主要给 Agent**（受控问数）。  
- 已有成熟 Cube/LookML/MetricFlow 且 Agent 只查权威接口时，WrenAI 增量在 **Agent 工作流**（dry-plan、memory、SDK），而非替 BI 读表。

---

## 5. 治理、数据质量与 NL2SQL 争论

### 5.1 数据目录（Agent 的「地图」）

| 项目 | GitHub | 作用 |
|------|--------|------|
| **DataHub** | [datahub-project/datahub](https://github.com/datahub-project/datahub) | 血缘、owner、glossary；图库+事件；MCP |
| **OpenMetadata** | [open-metadata/OpenMetadata](https://github.com/open-metadata/OpenMetadata) | 发现、质量、契约；一站式 |

**对 Agent**：不一定生成 SQL，但可回答「哪张表权威、字段含义、上下游」——减少乱猜。DataHub 偏平台组件；OpenMetadata 偏一体化上手。

**建议栈**：治理层 → 语义层 → WrenAI（原调研「治理 → Cube/WrenAI MDL → 交互」）。

### 5.2 数据质量（隐形杀手）

**Elementary**（dbt 原生）、**Soda Core** 等：在数据进语义层前做质量观测。若底层缺值、口径不一，Agent 会 **自信地答错**。AI 问数链路中常被低估，应与 dbt/语义层串联。

### 5.3 text-to-SQL vs text-to-Semantic-Layer

| 路线 | 代表 | 逻辑 | 弱点 |
|------|------|------|------|
| **text-to-SQL** | Vanna（已 archive）、早期仓内 Copilot | 给 LLM schema/示例，生成 SQL | 歧义、schema 复杂时天花板明显 |
| **text-to-Semantic-Layer** | WrenAI、Cube、仓库 Semantic Views | LLM 对**语义对象**提问，引擎出 SQL | 依赖语义层建设成熟度 |

**学术侧（2024–2025）**：

- 自然语言 **一问多 SQL**；评测函数本身有偏差 → benchmark 准确率常高于生产。  
- 通用 NL2SQL benchmark（Spider 等）与真实 BI 问题结构差异大。  

**行业侧**：Snowflake/Databricks 从「喂 YAML 的 text-to-SQL」转向 **内置语义层**，被部分从业者视为对 **语义层路线** 的方向性承认。

**对 WrenAI**：技术路线与「收窄查询空间」一致；瓶颈往往在 **企业尚未建好语义/MDL**，而非引擎 alone。

### 5.4 2025 趋势摘要

1. **语义层从可选变必选**（生成式 AI + 分析）。  
2. **Agentic BI**：Agent 编排准备→分析→报告（与 WrenAI 定位一致）。  
3. **BI-as-Code**：Rill、Evidence、Lightdash——Git 管指标与看板。  
4. **MCP**：DataHub、OpenMetadata、Malloy Publisher 等作为 Agent 工具接口。  
5. **开源 BI ROI**：部分调研显示开源 BI/AI 工具正向 ROI 比例上升；近半数企业计划增加开源 BI 使用（具体比例以原始报告为准）。

---

## 6. 重点：WrenAI（Agent 上下文层）

> **本章回答**：WrenAI 是什么、解决什么、与 dbt/BI/其他语义层如何共存、如何研究？

### 6.1 定位（新版主分支）

GitHub：[Canner/WrenAI](https://github.com/Canner/WrenAI)

**一句话**：面向 **AI Agent** 的 **开放上下文层（open context layer）**——给 Agent 提供 schema 之外的 **业务语义、示例、memory、治理**，并通过 **受控 SQL** 访问数仓。

```text
Agent → WrenAI（MDL + Engine + policy）→ 受控 SQL → Warehouse
```

**不是**：

```text
Agent → raw warehouse（易猜错表、口径）
```

**与旧版 GenBI**：`legacy/v1` 为旧应用；当前主线是 **Skills + CLI + SDK + wren-core**。

### 6.2 组件

| 组件 | 说明 |
|------|------|
| **MDL** | models、columns、relationships、views、cubes、metrics、RLAC/CLAC |
| **Engine** | Apache DataFusion（`wren-core`）；22+ 数据源 |
| **Memory** | LanceDB；可版本化、可检索历史问法 |
| **Agent SDK** | `wren-langchain`（LangChain/**LangGraph**）、`wren-pydantic` |
| **Skills** | `/wren-onboarding`、`/wren-enrich-context` 等，教编码 Agent 驱动 CLI |
| **正确性原语** | dry-plan、dry-run、structured errors、eval（持续加强） |

**项目结构（摘要）**：

```text
core/wren-core/      # Rust 语义引擎
core/wren/           # Python SDK / CLI（wrenai）
sdk/wren-langchain/  # LangGraph 等集成
skills/              # Agent skills
```

### 6.3 解决的核心问题

1. Agent 直连数仓易 **猜错表/字段**。  
2. 口径散落在表、文档、SQL、人脑。  
3. 需要 **可审查、可复现** 的上下文（Git-friendly MDL/instructions）。  
4. 多 Agent（Claude Code、Cursor、LangGraph 应用等）**共享**同一上下文层，而非锁死在某一 BI UI。

### 6.4 与 Vanna 对比

| | **WrenAI** | **Vanna** |
|---|------------|-----------|
| 思路 | 限制查询空间 + 语义层 | RAG + 训练让模型猜更准 |
| 状态 | 活跃 | 仓库 **已 archive** |
| 生产 | 可作 Agent 语义层 | 新项目不建议押注 |

### 6.5 与 MetricFlow / Cube 对比

| 维度 | MetricFlow / Cube | WrenAI |
|------|-------------------|--------|
| 物理建模 | 不建 staging（dbt 做） | 不建 staging |
| 指标/API | MetricFlow 编译；Cube 强 API/缓存 | MDL + Engine；偏 **Agent 工作流** |
| 重叠 | 语义定义 + SQL 生成 | 同左 |
| 差异 | 工程/平台语义层 | **Agent-native**：memory、skills、dry-plan、多框架 SDK |

**部分替代关系**：在「语义 + 问数」上 WrenAI 可 **分担 Cube 类职责**；**不能**替代 dbt 的 ELT。无 dbt 时可用 **脚本建仓 + WrenAI MDL**（财报等已结构化场景）。

### 6.6 能否「只用 WrenAI」？

| 任务 | 仅 WrenAI |
|------|-----------|
| 导入数据 | ❌ 需自建 ETL/厂商库 |
| 描述性分析（问数） | ✅ 需 MDL + 表质量 |
| 严肃投研（回归、面板、可发表） | ⚠️ 常需 **Notebook** 等 |
| 预测建模 | ❌ 非主业；需 Python/量化栈 |

**WrenAI 不提供** 交易多 Agent、多空辩论；那是 TradingAgents 类产品域。二者可集成：WrenAI 作 **fundamental data vendor**（精细数字）→ TA 作 **叙事与决策**（需自写 `route_to_vendor` 适配）。

### 6.7 准确性：不会「天然更准」

优势在于 **收窄** Agent 查询到 MDL + 可执行规划 + 治理 + 验证 + memory。  
若底层无权威 marts/指标，**无法凭空变准**。与 BI 关系：**前置条件决定上限**。

### 6.8 快速上手（官方路径）

```bash
npx skills add Canner/WrenAI --skill '*'
# Agent 侧：/wren-onboarding → 连库、脚手架、首查
# 可选：/wren-enrich-context 沉淀 MDL、instructions、memory
wren ask "..."   # 或通过 Agent 自然语言
```

文档：[docs.getwren.ai](https://docs.getwren.ai)

---

## 7. 开源生态速查与研究路径

### 7.1 横向对比矩阵

| 工具 | 层次 | 语义层 | 非技术用户 | AI 集成 | 维护 |
|------|------|--------|------------|---------|------|
| dbt Core | 建模 | 间接 | ✗ | 中 | 活跃 |
| MetricFlow | 语义 | 核心 | ✗ | 中 | 活跃 |
| Cube | 语义 | 核心 | 有限 | 强 | 活跃 |
| Lightdash | 语义+BI | dbt 原生 | 部分 | AI Agent | 活跃 |
| Malloy | 语义 | 语言级 | ✗ | MCP | 较活跃 |
| Superset | BI | 轻量 | 有限 | 有 | 活跃 |
| Metabase | BI | ✗ | 强 | 基础 | 活跃 |
| Grafana | BI/监控 | ✗ | 部分 | 有 | 活跃 |
| Redash | BI | ✗ | 有限 | 有限 | 社区 |
| Rill | BI | YAML | 部分 | 有 | 活跃 |
| Evidence | BI | ✗ | ✗ | 有限 | 活跃 |
| DataHub | 治理 | 元数据 | 有限 | MCP | 活跃 |
| OpenMetadata | 治理 | Glossary | 部分 | MCP | 活跃 |
| **WrenAI** | **Agent 问数** | **MDL 核心** | Agent 用户 | **核心** | 活跃 |
| Vanna | AI 问数 | ✗ | 聊天 | LLM | Archive |

> Stars 为各源文档 2025 参考值，请以 GitHub 实时为准。

### 7.2 按团队选型

| 团队 | 组合 |
|------|------|
| dbt 数据团队 | dbt + MetricFlow + Lightdash |
| 嵌入式分析 | dbt + Cube + 自研前端 |
| 业务自助 | dbt + Metabase |
| DevOps | Grafana + Prometheus/Loki |
| **AI Agent 问数** | dbt（或等价建仓）+ Cube/MetricFlow **或** MDL-only + **WrenAI** |
| 治理 + AI | 上表 + DataHub 或 OpenMetadata |

### 7.3 按 BI 场景

| 场景 | 推荐 |
|------|------|
| 业务自助 Dashboard | Metabase |
| 数据团队高级分析 | Superset |
| 实时运维 | Grafana |
| SQL 分析师轻量 | Redash |
| dbt 指标看板 | Lightdash |
| 代码化报表 | Evidence |
| 运营极速看板 | Rill |

### 7.4 推荐研究顺序（以理解 WrenAI 为目标）

1. **建模**：[dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core) — staging/marts、测试、为何不查 raw。  
2. **语义**：[metricflow](https://github.com/dbt-labs/metricflow)、[cube](https://github.com/cube-js/cube)、[lightdash](https://github.com/lightdash/lightdash)、[malloy](https://github.com/malloydata/malloy) — 指标代码化、API、join 治理。  
3. **BI**：[superset](https://github.com/apache/superset)、[metabase](https://github.com/metabase/metabase)、[grafana](https://github.com/grafana/grafana)、[rill](https://github.com/rilldata/rill)、[evidence](https://github.com/evidence-dev/evidence) — 人如何消费数据。  
4. **治理**：[datahub](https://github.com/datahub-project/datahub)、[OpenMetadata](https://github.com/open-metadata/OpenMetadata) — 权威表与 glossary。  
5. **Agent**：[WrenAI](https://github.com/Canner/WrenAI) — MDL、dry-plan、SDK、skills；Vanna 仅作 RAG 思路参考。

### 7.5 最小项目组合

```text
dbt-core
MetricFlow 或 Cube
Lightdash（可选，理解 dbt-native BI）
DataHub 或 OpenMetadata
WrenAI
```

覆盖 **raw → marts → 语义 → 治理 → Agent** 全链路。

---

## 8. 结论：三个判断 + WrenAI 站位

### 8.1 三个行业判断（整合原独立调研）

1. **仓库原生语义层**是独立 MetricFlow/Cube 的最大竞争压力，但 **多云/多仓** 企业仍需要独立语义层与互操作标准（OSI 等）。  
2. **text-to-Semantic-Layer** 在 BI 场景优于纯 text-to-SQL；**语义层建设成熟度** 是全行业瓶颈，也是 WrenAI 的前置条件。  
3. **数据质量**（Elementary、Soda 等）应进入 Agent 链路，否则语义层再完整也会「自信地错」。

### 8.2 WrenAI 在栈中的最终站位

```text
dbt marts  →  BI  →  人
     │
     └──→  MetricFlow / Cube / MDL（WrenAI）
              └──→  Agent（LangGraph、Copilot、TradingAgents+适配 …）
```

- **dbt、MetricFlow、Cube、Lightdash、DataHub/OpenMetadata**：建模、指标、给人看的 BI、治理 **基础设施**。  
- **WrenAI**：让 Agent **在已定语义内** 问数、校验、记忆；**Agent-native 查询/上下文层**，不是 marts 唯一出口，也不替代 BI。  
- **新版 WrenAI** 的产品方向就是 **接入各类 Agent**（Skills、LangGraph SDK）；与 TradingAgents 等结合属于 **合理延伸**，需自写数据 vendor / 模板层，无官方一键插件。

### 8.3 给研究者的实操建议

1. 先跑通 **dbt 或 jaffle_shop + `/wren-onboarding`**，理解 MDL 与 `wren ask`。  
2. 对比同一问题在 **Metabase（人）** 与 **WrenAI（Agent）** 的路径差异。  
3. 读 `core/wren` 的 `dry_plan` / `query`，建立「语义层如何限制 SQL」直觉。  
4. 若做 A 股/财报：**建仓 + MDL** 优先于纠结是否上 dbt；若做交易 Agent：**WrenAI 供数 + TradingAgents 供叙事**。

---

## 附录：整合说明

| 源文件 | 主要并入章节 |
|--------|----------------|
| `BI_SEMANTIC_LAYER_GITHUB_PROJECTS.md` | §0–§4、§6–§7（架构、dbt/BI 分工、人 vs Agent、项目清单） |
| `BI-report.md` | §1 历史、§4 BI 详表、§7 矩阵与选型 |
| `BI-original-report.md` | §1.4 市场、§3.4–3.5 仓内语义层、§5 NL2SQL/质量/Excel、§8.1 三判断 |
| `BI-research-report.md` | §1 三阶段表、§3 Headless/BI-as-Code、§6.4–6.5 互补性 |

**舍弃或压缩**：四份文档中重复的「推荐研究顺序」合并为 §7.4；重复的工具段落改为矩阵+简表；与 2026 WrenAI 主分支冲突的「纯 GenBI 应用」表述已按 README 更新。

---

*文档版本：整合稿 v1 | 路径：`WrenAI/BI-agent研究.md`*
