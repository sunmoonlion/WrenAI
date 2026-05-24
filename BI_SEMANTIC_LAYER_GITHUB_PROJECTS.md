# BI、语义层与 AI 问数相关 GitHub 项目梳理

这份文档整理一组适合研究 BI、语义层、指标层、数据治理和 AI 问数的权威开源项目。目标不是列链接，而是帮助判断每类项目解决什么问题、和 WrenAI 的关系是什么、应该按什么顺序研究。

## 1. 总体图景

如果目标是让 BI 或 AI Agent 更准确地回答业务问题，通常需要几层能力：

```text
原始数据
  -> 数据建模 / 数据转换
  -> 业务 marts
  -> 指标层 / 语义层
  -> BI 展示 / API / Agent 查询
  -> 数据目录 / 血缘 / 治理 / 质量信号
```

对应到工具：

- **dbt Core**：把 raw tables 转成稳定的业务表、事实表、维表、marts。
- **MetricFlow / Cube / Lightdash / Malloy**：定义指标、维度、关系、查询语义。
- **Superset / Metabase / Redash / Grafana / Rill / Evidence**：做 BI 展示、dashboard、报表、实时看板或 BI as Code。
- **DataHub / OpenMetadata**：提供数据目录、血缘、owner、glossary、质量信号和治理上下文。
- **WrenAI / Vanna**：面向 AI Agent 或自然语言问数，把语义层暴露给 LLM 使用。

WrenAI 更适合放在最后一层理解：它不是替代所有 BI/指标/治理工具，而是让 AI Agent 能消费这些业务语义和查询约束。

### 1.1 语义层、dbt / Cube / Looker 与 BI 的关系

**Semantic Layer（语义层）** 不是某个叫 `semantic-layer` 的 GitHub 仓库，而是架构上的一层能力：统一「指标叫什么、怎么算、维度怎么切、表怎么 join」，查询时由引擎生成 SQL，避免每个 BI 或 Agent 各写一套、口径不一致。

在 dbt 生态里，名字容易混，建议分开记：

| 名词 | 是什么 | 仓库 |
|------|--------|------|
| **dbt Labs** | 公司名 | — |
| **dbt-labs** | GitHub **组织**（不是单一总项目） | 组织页：[github.com/dbt-labs](https://github.com/dbt-labs) |
| **dbt-core** | 数仓里建表、跑模型、测试 | [dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core) |
| **MetricFlow** | 读指标定义、编译成 SQL 的开源引擎 | [dbt-labs/metricflow](https://github.com/dbt-labs/metricflow)（与 dbt-core **同组织、不同仓库**） |
| **dbt Semantic Layer** | 产品/能力名：把指标稳定暴露给 BI、API、AI | 无独立 `semantic-layer` repo；引擎侧主要是 MetricFlow |

**dbt、Cube、Looker 是否做类似的事？** 在「语义层」这一块有重叠，但不是三个可互换产品：

| 产品 | 主要干什么 | 更像什么 |
|------|------------|----------|
| **dbt**（+ MetricFlow） | 数仓里**建表、清洗、建模**；指标在 YAML 里定义并**编译 SQL** | 数据工程 + 偏工程侧的语义层 |
| **Cube** | **Headless 语义层**：指标模型 + 查询 + **REST / GraphQL / SQL API**（组织 [cube-js](https://github.com/cube-js)，**不属于 dbt Labs**） | 给多个应用、BI、Agent 共用的指标中间层 |
| **Looker** | **完整 BI**：LookML 语义层 + 探索 + 看板 + 权限（商业产品；开源多为 LookML 工具） | 语义层与 BI **一体** |
| **Lightdash** | dbt-native 路线：在 dbt 旁定义 metrics，自带可视化 | 介于「dbt + 语义层」与「BI」之间 |

和 **后面的 BI**（Superset、Metabase、Redash、Tableau、Power BI 等）的关系，可以记成「上游定口径，下游做展示」：

```text
源系统 → 数仓
           ↓
        dbt-core（物理表：staging / intermediate / marts）
           ↓
    语义层（定义指标与维度，生成或约束 SQL）
      · dbt Semantic Layer + MetricFlow
      · Cube
      · LookML（Looker）/ Lightdash 内置语义
           ↓
    BI（图表、看板、拖拽、订阅、权限展示）
      · Superset / Metabase / Redash / Grafana / Rill …
           ↓
    业务用户 / 分析师
           ↘
        WrenAI / Vanna（自然语言问数，宜消费 marts 或语义层，不宜裸扫全库）
```

三种常见组合：

| 模式 | 链路 | 特点 |
|------|------|------|
| **简配** | dbt marts → BI **直连数仓表** | 最常见；BI 里各自拖字段、写度量，口径易散 |
| **标准配** | dbt marts → MetricFlow / Cube / SL → BI **查指标+维度** | 指标一处定义；BI 是语义层的客户端 |
| **一体化** | dbt marts → **Looker**（或 Lightdash） | LookML + 看板一家；通常不再叠第二层纯 BI |

依赖关系（选型时够用）：

| 问题 | 答案 |
|------|------|
| 没有 dbt 能有 BI 吗？ | 能，BI 可直接连原始表（口径难控）。 |
| 没有 BI 能有语义层吗？ | 能，API、Notebook、Agent 可直接查 Cube / SL。 |
| dbt 能替代 Superset 吗？ | **不能**，dbt 不提供拖拽看板。 |
| BI 能替代 dbt 吗？ | **不能**，BI 一般不负责批量建表、测试、血缘。 |
| 语义层能替代 BI 吗？ | **大部分不能**；Looker、Lightdash 例外（自带 BI）。 |
| Cube 是 dbt Labs 的项目吗？ | **不是**，与 dbt 可集成，但是独立厂商与仓库。 |

实际团队常见叠法：**dbt +（Cube 或 Looker / Lightdash）**，或 **dbt 表 + 开源 BI**；WrenAI 类工具放在最上层，理想情况是 Agent 查询已治理的 marts 或语义层接口，而不是替代 dbt 或 BI 整层。

### 1.2 人 vs Agent：谁消费哪一层

一个常见误解是：**BI 主要给「不懂财务 / 不懂 SQL 的人」看；把 dbt marts 给 Agent 看主要是 WrenAI。** 大方向对，但需要收窄。

**BI 主要是给人用的 —— 对**

BI 的核心是图表、看板、拖拽筛选、订阅与权限，心智模型是 **「人看数、人探索」**。典型用户包括：

- 业务、运营、销售、管理层：少写 SQL，靠点选和固定看板
- 分析师：会 SQL，但仍常用 BI 做交付和协作
- 数据团队：有时也用 BI 做内部报表

因此 BI 不等于「只服务不懂财务的人」；更准确是 **服务「要以可视化方式消费已定义数据」的人**，其中很多人不负责定义指标口径。

**把 marts / 语义给 Agent —— WrenAI 是重要路线，不是唯一**

dbt 的 **marts** 是 Agent 的理想**物理数据源**（表已主题化、口径在建模阶段沉淀过），但生产上通常 **不会** 把「整库 marts + 全部列」直接交给 LLM 自由生成 SQL，而是再加约束：

| 消费方 | 常见载体 | 要什么 |
|--------|----------|--------|
| **人** | Superset、Metabase、Looker、Lightdash | 看图、探索、固定报表、权限内的自助分析 |
| **程序 / Agent** | Cube API、dbt Semantic Layer、Malloy Publisher、[WrenAI](https://github.com/Canner/WrenAI)、[Vanna](https://github.com/vanna-ai/vanna) | 可执行的查询边界：允许哪些表/指标、join 路径、策略与校验 |

**WrenAI** 的定位是 **Agent-native 问数**（MDL、dry-run、策略、memory 等），不是「唯一能把 dbt 暴露给 Agent 的工具」。同类或互补路线还包括：Vanna（NL2SQL 框架）、Cube + 自建 Agent（调 SQL/REST API）、dbt SL + 自研 Agent（只查已定义 metrics）。

推荐记成 **两条并行链路**，而不是 BI 与 Agent 二选一：

```text
dbt marts  ──→  BI（Superset / Metabase / Looker …）  ──→  人
     │
     └──→  语义 / 治理（MetricFlow、Cube、MDL / WrenAI …）  ──→  Agent / API / 嵌入应用
```

要点：

- **人走路径**：marts 可直接进 BI（简配），或经语义层再进 BI（口径更统一）。
- **Agent 走路径**：优先 **marts + 语义/治理层**；WrenAI 解决的是「在已定语义内安全问数」，不替代 dbt 建表，也不替代 BI 给人看图。
- **已有成熟 dbt marts + LookML/Cube/MetricFlow，且 Agent 只能查权威接口** 时，WrenAI 的增量在于 Agent 工作流本身，而非「替 BI 读表」。

下文第 3 节展开各语义层项目，第 4 节展开纯 BI，第 6 节展开 AI 问数；研究顺序见第 8 节；与 WrenAI 的边界见第 10 节。

## 2. 数据建模与转换

### dbt Core

GitHub: [dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core)

dbt Core 是现代数据栈里最重要的数据建模工具之一。它用 SQL 和 YAML 管理数据转换，把原始数据清洗、标准化、聚合成业务可用的模型。

它解决的问题：

- 把 raw tables 转成 staging、intermediate、marts。
- 用代码管理数据转换逻辑。
- 支持测试、文档、血缘和 CI。
- 让数据团队用软件工程方式维护数据模型。

典型产物：

```text
models/
  staging/
  intermediate/
  marts/
```

这三层是 dbt 项目里常见的加工顺序，表示数据从“原始”到“给业务用”的路径：

| 目录 | 作用 | 典型例子 |
|------|------|----------|
| `staging/` | 贴源清洗：改列名、统一类型、去重、过滤脏数据；一行仍大致对应源表一行 | `stg_orders`：规范后的订单表 |
| `intermediate/` | 中间计算：多表 join、聚合、复杂业务规则；供下游复用，一般不直接给报表 | `int_orders_enriched`：订单 + 客户 + 商品 |
| `marts/` | 主题成品：按分析主题建模，给 BI、指标层、问数直接用 | `mart_revenue_daily`：按天收入（口径已定） |

```text
raw 表 → staging/ → intermediate/ → marts/
```

名词补充（文档后文会用到）：

- **dbt**：整套方法和生态（在仓库里用 SQL 做建模与转换）。
- **dbt-core**：开源核心程序，对应 [dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core)；本地 `dbt run` 主要由它执行。
- **dbt-native**：形容词，指工具原生围绕 dbt 项目设计（如 Lightdash 直接读 dbt 的 models 和 metrics），不必在 BI 里再建一套平行模型。

在 BI 准确性里，dbt 的价值是：先把“数据长什么样”整理成“业务可查的表”。如果没有这层，语义层和 AI 问数会直接面对混乱的 raw schema。

适合优先研究。

## 3. 指标层与语义层

### MetricFlow

GitHub: [dbt-labs/metricflow](https://github.com/dbt-labs/metricflow)

MetricFlow 是 dbt Labs 的语义层 / 指标层核心项目（仓库 [dbt-labs/metricflow](https://github.com/dbt-labs/metricflow)，与 [dbt-core](https://github.com/dbt-labs/dbt-core) **同组织、独立仓库、独立发版**）。它允许把指标定义成代码，然后把指标请求编译成数据库 SQL；**dbt Semantic Layer** 是产品侧对这一能力的称呼，引擎侧主要是 MetricFlow，并没有名为 `semantic-layer` 的单一开源总仓库。

它解决的问题：

- 定义 revenue、active users、conversion rate 等指标。
- 支持维度切分和时间粒度聚合。
- 支持复杂指标类型，例如 ratio、expression、cumulative。
- 基于 dbt 项目和 dbt adapter 工作。

适合研究的问题：

- 指标如何从 YAML 定义变成 SQL。
- 多跳 join 如何在指标查询中处理。
- 指标层如何保证不同 BI/应用查询同一口径。

它和 WrenAI 的关系：

MetricFlow 更偏“指标定义和 SQL 编译”。WrenAI 更偏“Agent 如何发现、理解并调用语义查询能力”。

### Cube

GitHub: [cube-js/cube](https://github.com/cube-js/cube)

Cube 是开源语义层（[cube-js/cube](https://github.com/cube-js/cube)，**不属于 dbt Labs**），面向 BI、嵌入式分析、API 和 AI Agent。它定义 metrics、dimensions、joins，并通过 REST、GraphQL、SQL 等接口暴露；与 MetricFlow 在「统一指标语义」上类似，但 Cube 偏 **headless API**，不负责数仓里 dbt 式的物理建模。和 BI、dbt 的分工见上文 [§1.1](#11-语义层dbt--cube--looker-与-bi-的关系)。

它解决的问题：

- 把指标和维度集中定义。
- 给多个前端、BI、应用和 Agent 复用同一套语义层。
- 支持多种数据库和数据仓库。
- 内置缓存和预聚合，适合高并发分析 API。

适合研究的问题：

- 一个 headless semantic layer 应该怎么设计。
- 语义层如何服务前端应用、BI 和 AI。
- 预聚合、缓存、API 化在实际生产中的意义。

如果要研究“语义层如何产品化”，Cube 很值得看。

### Lightdash

GitHub: [lightdash/lightdash](https://github.com/lightdash/lightdash)

Lightdash 是 dbt-native 的开源 BI 工具，也可以理解为开源 Looker 替代路线。它让你在 dbt 模型旁边定义 metrics 和 dimensions，然后提供可视化、dashboard 和自助分析。

它解决的问题：

- 直接复用 dbt 项目。
- 在 YAML 中定义指标和维度。
- 给业务用户 self-service BI。
- 支持开发预览环境、CI/CD、内容校验。

适合研究的问题：

- dbt 模型如何自然延伸成 BI 语义层。
- 指标定义如何和 dashboard 结合。
- 业务用户如何消费工程化定义。

它比 WrenAI 更偏传统 BI；WrenAI 更偏 AI Agent 问数。

### Malloy

GitHub: [malloydata/malloy](https://github.com/malloydata/malloy)

Malloy 是一种开源语义建模语言和查询语言。它试图用比 SQL 更贴近分析语义的方式描述数据关系、转换和查询，然后编译到底层 SQL 引擎执行。

相关项目：

- [malloydata/publisher](https://github.com/malloydata/publisher)：Malloy 语义模型服务器，提供 REST / MCP API。
- [malloydata/malloy-vscode-extension](https://github.com/malloydata/malloy-vscode-extension)：VS Code 扩展。

它解决的问题：

- 用专门语言表达分析模型。
- 把查询和建模放在同一套语言中。
- 支持 BigQuery、Snowflake、Postgres、MySQL、Trino、Presto、DuckDB 等。

适合研究的问题：

- SQL 之外是否需要新的分析语言。
- 语义建模语言如何设计。
- REST / MCP 如何把语义模型暴露给工具和 Agent。

## 4. BI 与报表工具

这一节是真正更接近“BI 产品”的项目。它们的核心目标是让用户探索数据、写查询、做图表、搭 dashboard、分享报表。和 `dbt`、`MetricFlow`、`Cube` 这类底层建模/语义层工具不同，BI 工具通常直接面向分析师、业务用户或数据消费者。

**与语义层的关系（详见 [§1.1](#11-语义层dbt--cube--looker-与-bi-的关系)）**：BI 多数情况下消费 **dbt 产出的 marts 表**（简配），或通过 **Cube / dbt Semantic Layer** 查询已定义的指标（标准配）。BI **不替代** dbt 建表，也 **不自动承担** 全公司指标口径治理——除非选用自带语义层的 Looker、Lightdash 路线。

### Apache Superset

GitHub: [apache/superset](https://github.com/apache/superset)

Apache Superset 是成熟的开源 BI 和数据可视化平台。它提供 dashboard、SQL Editor、图表构建和轻量 semantic layer。

它解决的问题：

- 快速构建 dashboard。
- 提供 SQL Lab 给高级用户探索。
- 支持大量 SQL 数据源。
- 提供基础指标和维度配置。

适合研究的问题：

- 开源 BI 平台如何组织数据集、图表、dashboard。
- 权限、缓存、SQL 编辑器、可视化如何集成。

Superset 的语义层较轻，不是最适合研究复杂指标建模的项目，但适合理解 BI 平台本身。

### Metabase

GitHub: [metabase/metabase](https://github.com/metabase/metabase)

Metabase 是易用型开源 BI 工具，主打让非技术用户更容易问数、看报表、构建 dashboard。

它解决的问题：

- 低门槛数据探索。
- 可视化查询构建。
- dashboard 和 embedded analytics。
- Canonical metrics / Data Studio 等业务定义能力。

适合研究的问题：

- 面向业务用户的 BI 体验如何设计。
- 如何降低 SQL 门槛。
- 一个产品化 BI 工具如何组织问题、报表、模型和权限。

Metabase 很适合理解“业务用户怎么用 BI”，但不是最强语义层样本。

### Redash

GitHub: [getredash/redash](https://github.com/getredash/redash)

Redash 是 SQL-first 的开源 BI / dashboard 工具。它面向会写 SQL 的数据分析师和工程师，强调连接多种数据源、写查询、做可视化、组合 dashboard、定时刷新和分享。

它解决的问题：

- 浏览器里写 SQL / NoSQL 查询。
- 连接大量 SQL 和 NoSQL 数据源。
- 把查询结果做成图表和 dashboard。
- 定时刷新报表。
- 分享查询和 dashboard，支持协作 review。
- 通过 REST API 操作查询和可视化资源。

适合研究的问题：

- SQL-first BI 工具如何设计。
- 查询、可视化、dashboard、refresh schedule 如何组织。
- 面向数据团队的轻量 BI 和 Superset / Metabase 的差异。

客观评价：

Redash 曾经非常流行，GitHub star 多、使用面广。它适合研究 SQL-first BI 的产品形态。不过如果是新项目选型，需要额外关注它的维护活跃度、部署方案和社区状态。

### Grafana

GitHub: [grafana/grafana](https://github.com/grafana/grafana)

Grafana 是开源 observability 和数据可视化平台。它最强的场景不是传统经营 BI，而是监控、时序指标、日志、trace、实时 dashboard 和告警。

它解决的问题：

- 从 Prometheus、Loki、Elasticsearch、InfluxDB、Postgres、MySQL 等数据源查询数据。
- 构建动态 dashboard。
- 支持变量、时间范围、panel、drill-down。
- 做指标、日志、trace 的统一可视化。
- 配置告警和通知。

适合研究的问题：

- 实时 dashboard 如何设计。
- 多数据源可视化平台如何抽象 datasource 和 panel。
- observability dashboard 和传统 BI dashboard 的差异。
- 告警、变量、时间范围、面板系统如何组合。

客观评价：

Grafana 是 dashboard 领域最权威的开源项目之一，但它更偏运维监控和实时指标，不是最典型的财务/销售/经营分析 BI。它适合研究“实时指标看板”和“可视化平台”，不适合作为指标语义层的主要样本。

### Rill

GitHub: [rilldata/rill](https://github.com/rilldata/rill)

Rill 是现代 BI-as-code / operational BI 项目。它强调从数据湖或文件快速到 dashboard，使用 SQL model 和 YAML 定义 metrics、dashboard，并通过 DuckDB / ClickHouse 等引擎提供快速交互式分析。

它解决的问题：

- 用代码定义数据源、SQL 模型、metrics view 和 dashboard。
- 快速构建可 drill-down 的 operational dashboard。
- 本地和云端都能运行。
- 用 DuckDB / ClickHouse 做快速分析。
- 把 metrics 层服务给 dashboard、API 和 AI 系统。

适合研究的问题：

- BI-as-code 在现代数据项目中如何落地。
- SQL model + YAML metrics + dashboard 的目录结构如何设计。
- 快速交互式 dashboard 如何依赖本地/嵌入式分析引擎。
- 面向 AI Agent 的实时 metrics layer 如何设计。

客观评价：

Rill 比 Evidence 更偏交互式 dashboard，比 Superset/Metabase 更偏工程化和速度。它值得和 Evidence、Lightdash 一起研究。

### Evidence

GitHub: [evidence-dev/evidence](https://github.com/evidence-dev/evidence)

Evidence 是 BI as Code 工具，用 SQL 和 Markdown 构建数据产品、报告和可交互页面。

它解决的问题：

- 用代码写报表。
- SQL 查询嵌入 Markdown。
- 生成静态或可部署的数据网站。
- 支持图表、表格、模板、循环和条件渲染。

适合研究的问题：

- 报表如何工程化。
- 分析文档和查询结果如何绑定。
- 数据产品如何像软件项目一样版本管理。

Evidence 不强调复杂语义层，更像“报表和数据应用 as code”。

### Appsmith / Openblocks 这类低代码 dashboard 工具

相关项目：

- [appsmithorg/appsmith](https://github.com/appsmithorg/appsmith)
- [openblocks-dev/openblocks](https://github.com/openblocks-dev/openblocks)

这类项目经常被拿来做内部 dashboard、管理后台、业务工具和简单数据应用，但严格说它们不是传统 BI。它们更像开源 Retool 替代品。

它们解决的问题：

- 拖拽式构建内部工具。
- 连接数据库和 API。
- 做管理后台、审批流、运营工具、客户 360 页面。
- 快速做带交互的内部数据应用。

适合研究的问题：

- BI dashboard 和内部业务应用的边界在哪里。
- 数据查询结果如何嵌入业务流程。
- 低代码工具如何连接数据库、API 和 UI 组件。

客观评价：

如果目标是研究 BI 核心能力，优先级低于 Superset、Metabase、Lightdash、Rill、Evidence、Redash、Grafana。但如果目标是“把数据分析嵌入业务操作页面”，这类项目值得看。

## 5. 数据目录、治理与上下文

### DataHub

GitHub: [datahub-project/datahub](https://github.com/datahub-project/datahub)

DataHub 是开源元数据平台，起源于 LinkedIn。它提供数据目录、血缘、owner、文档、标签、glossary、质量信号、使用统计和 MCP。

它解决的问题：

- 数据资产发现。
- 表、字段、dashboard、pipeline 的元数据管理。
- 上下游血缘和影响分析。
- owner、domain、tag、glossary 管理。
- 给 AI Agent 提供数据上下文。

对 AI 问数的价值：

DataHub 不一定直接生成 SQL，但它能告诉 Agent：

- 哪张表更权威。
- 谁拥有这个表。
- 这个字段是什么意思。
- 上游来自哪里。
- 下游哪些报表依赖它。
- 历史查询是怎么用它的。

这类上下文对减少 Agent 乱猜很重要。

### OpenMetadata

GitHub: [open-metadata/OpenMetadata](https://github.com/open-metadata/OpenMetadata)

OpenMetadata 是另一个重要的开源数据目录和治理平台。它把技术元数据、质量、血缘、owner、glossary、metrics、domain、policy 等组织成统一 metadata graph。

相关项目：

- [open-metadata/OpenMetadataStandards](https://github.com/open-metadata/OpenMetadataStandards)：元数据 schema、ontology、标准定义。

它解决的问题：

- 数据发现。
- 数据质量。
- 列级血缘。
- glossary 和业务术语。
- metrics / KPI 元数据。
- MCP / API / SDK 给 AI 系统提供上下文。

DataHub 和 OpenMetadata 都不是传统语义层，但它们对“让 Agent 更懂数据资产上下文”很关键。

## 6. AI 问数与 NL2SQL

### WrenAI

GitHub: [Canner/WrenAI](https://github.com/Canner/WrenAI)

WrenAI 是面向 AI Agent 的开放上下文层 / 语义 SQL 层。它通过 MDL 定义模型、字段、关系、权限、memory 和查询约束，让 Agent 不直接面对整个数据库。

它解决的问题：

- Agent 直接查 raw warehouse 容易猜错。
- 业务口径散落在表、文档、SQL 和人脑中。
- 需要 dry-plan、dry-run、访问控制、memory 和 Agent SDK。

它更像是：

```text
Agent -> WrenAI -> 受控语义 SQL -> warehouse
```

而不是：

```text
Agent -> raw warehouse
```

适合研究的问题：

- AI Agent 如何使用语义层。
- MDL 如何参与 SQL planning。
- memory 如何沉淀已确认的 NL-SQL。
- Agent 工作流中如何加入 dry-run、policy 和权限。

### Vanna

GitHub: [vanna-ai/vanna](https://github.com/vanna-ai/vanna)

Vanna 是 Text-to-SQL / Chat with database 项目，主打通过训练材料、schema、SQL 示例和 RAG 生成 SQL。

它解决的问题：

- 自然语言生成 SQL。
- 用历史 SQL、DDL、文档提升 SQL 生成质量。
- 提供聊天式数据库查询体验。

注意：搜索结果显示该仓库已 archived。可以研究它的产品思路和训练/RAG 方法，但如果做新项目，不建议完全押注。

它和 WrenAI 的区别：

- Vanna 更偏 NL2SQL 生成框架。
- WrenAI 更偏受控语义层 + Agent 工作流。

## 7. LookML / Looker 相关项目

### Looker Open Source

GitHub: [looker-open-source](https://github.com/looker-open-source)

LookML 本身属于 Looker 生态。Looker 是商业 BI 产品，LookML 是其建模语言，用来定义 explores、views、joins、dimensions、measures。

开源相关项目：

- [looker-open-source/sdk-codegen](https://github.com/looker-open-source/sdk-codegen)：Looker SDK 代码生成和 API Explorer。
- [looker-open-source/look-at-me-sideways](https://github.com/looker-open-source/look-at-me-sideways)：LookML linter / style guide。
- [looker-open-source/app-lookml-diagram](https://github.com/looker-open-source/app-lookml-diagram)：LookML 图谱可视化。
- [lkrdev/lookml-language-server](https://github.com/lkrdev/lookml-language-server)：LookML language server。

LookML 值得研究的原因：

- 它是语义建模在商业 BI 中非常成熟的实践。
- 它强调统一 join、dimension、measure 和 explore。
- 它证明了“语义层 + BI”的长期价值：**Looker 把语义层与 BI 前端合在一个产品里**，典型栈是 `dbt 建表 → LookML 定义指标与探索 → 看板交付`，而不是再叠一层 Superset。

但 Looker 本体不是完全开源项目，因此学习时更多是研究思想和生态工具。与 dbt、Cube、开源 BI 的分工对照见 [§1.1](#11-语义层dbt--cube--looker-与-bi-的关系)。

## 8. 推荐研究顺序

如果你的目标是理解“如何让 BI / AI 问数更准”，建议按这个顺序：

### 第一阶段：数据建模基础

先研究：

- [dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core)

要搞清楚：

- raw、staging、intermediate、marts 的分层。
- 数据转换如何代码化。
- 测试和文档如何维护。
- BI 为什么不应该直接查 raw tables。

### 第二阶段：指标与语义层

再研究：

- [dbt-labs/metricflow](https://github.com/dbt-labs/metricflow)
- [cube-js/cube](https://github.com/cube-js/cube)
- [lightdash/lightdash](https://github.com/lightdash/lightdash)
- [malloydata/malloy](https://github.com/malloydata/malloy)

要搞清楚：

- revenue、active users、conversion rate 如何定义成代码。
- 指标请求如何编译成 SQL。
- join path、time grain、dimension、measure 如何治理。
- 语义层如何被 BI、API、Agent 复用。

### 第三阶段：BI 产品形态

再研究：

- [apache/superset](https://github.com/apache/superset)
- [metabase/metabase](https://github.com/metabase/metabase)
- [getredash/redash](https://github.com/getredash/redash)
- [grafana/grafana](https://github.com/grafana/grafana)
- [rilldata/rill](https://github.com/rilldata/rill)
- [evidence-dev/evidence](https://github.com/evidence-dev/evidence)

要搞清楚：

- 业务用户如何实际消费数据。
- dashboard、SQL Editor、可视化、权限如何组织。
- SQL-first BI、self-service BI、observability dashboard、operational BI、BI as Code 和传统 BI 的差异。

### 第四阶段：治理与上下文

再研究：

- [datahub-project/datahub](https://github.com/datahub-project/datahub)
- [open-metadata/OpenMetadata](https://github.com/open-metadata/OpenMetadata)

要搞清楚：

- 数据目录如何帮助判断权威表。
- glossary 如何表达业务术语。
- lineage 如何做影响分析。
- owner、tag、quality signal 如何帮助 Agent 判断可信度。

### 第五阶段：AI 问数

最后研究：

- [Canner/WrenAI](https://github.com/Canner/WrenAI)
- [vanna-ai/vanna](https://github.com/vanna-ai/vanna)

要搞清楚：

- Agent 如何调用语义层。
- NL2SQL 如何利用 schema、指标、历史 SQL、memory。
- 如何防止 Agent 直接查错表。
- dry-plan、dry-run、policy、权限如何进入工作流。

## 9. 最小推荐组合

如果只选几个重点项目，建议看：

```text
dbt-core
MetricFlow
Cube
Lightdash
DataHub 或 OpenMetadata
WrenAI
```

原因：

- `dbt-core` 解决数据建模。
- `MetricFlow` 解决指标代码化。
- `Cube` 解决语义层 API 化。
- `Lightdash` 解决 dbt-native BI。
- `DataHub / OpenMetadata` 解决治理上下文。
- `WrenAI` 解决 Agent 如何消费语义层。

这条线基本覆盖从 raw data 到 AI 问数的完整链路。

## 10. 和 WrenAI 的关系总结

WrenAI 的位置不应该被理解成“替代所有 BI 工具”，而应该理解成：

```text
dbt / marts / metrics / semantic layer / catalog
  -> WrenAI
  -> Agent / natural language query
```

与 BI 的分工（详见 [§1.2](#12-人-vs-agent谁消费哪一层)）：

```text
dbt marts  →  BI  →  人（看板、探索、报表）
dbt marts  →  语义层 / MDL（WrenAI、Cube、MetricFlow …）  →  Agent（自然语言问数、API）
```

BI 和 WrenAI **并行**，不是「BI 给人、WrenAI 独占 marts」。WrenAI 也不等于「给不懂财务的人用的 BI」——它面向的是 **Agent 在受控语义内生成查询**；懂口径的数据工程师仍主要用 dbt，看图的分析师仍主要用 BI。

如果前面没有权威数据模型和指标定义，WrenAI 很难凭空变准。它的优势在于把已有或正在沉淀的业务语义变成 Agent 可调用、可 dry-run、可治理、可记忆的查询接口。

所以最合理的判断是：

**dbt、MetricFlow、Cube、Lightdash、DataHub/OpenMetadata 是语义和治理的基础设施；BI 主要是给人消费数；WrenAI（及 Vanna、Cube API 等）是让 Agent 在 marts/语义层之上问数的一种 Agent-native 查询层，而不是 marts 的唯一出口。**
