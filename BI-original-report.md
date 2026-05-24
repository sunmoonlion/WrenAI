# BI 现状独立研究报告
## 你的参考文档没有告诉你的事

> 本报告基于独立搜索研究，重点挖掘新信息、新趋势和新矛盾，不复述你已有的背景材料。

---

## 一、市场真实规模：这是一个 200 亿美元的战场

商业 BI 不是一个小圈子里的工程话题。2024 年全球分析与 BI 软件市场规模达到 **203 亿美元**，同比增长 10%，预计 2029 年达到 285 亿美元，CAGR 约 7%。市场前十大厂商占据了 64.1% 的份额，由 Salesforce（Tableau）以 14.8% 的份额领跑，其次是 SAP、SAS Institute、Microsoft、Oracle。

这个数字的意义在于：开源工具所争夺的，是这 203 亿美元里那些对价格敏感、对定制化有需求、对供应商锁定有顾虑的用户群。这是一个结构性的机会，不是边缘市场。

---

## 二、商业 BI 的并购潮：巨头在买什么

2019 年是 BI 行业最戏剧化的一年，短短几周内发生了三件定义行业格局的事：

- **Salesforce 以 157 亿美元收购 Tableau**（史上第三大软件收购案）
- **Google 以 26 亿美元收购 Looker**
- **Sisense 收购 Periscope Data**

三笔交易合计花掉 **184 亿美元**。这不是偶然。Forrester 当时的判断是：BI 核心能力（可视化、切片分析）已经商品化，未来的竞争不在功能，而在生态集成——谁能把 BI 嵌进 CRM、云平台、企业软件，谁就赢。

2025 年这条逻辑还在延续：Salesforce 宣布以 **80 亿美元收购 Informatica**（预计 2027 财年初完成），目的是把数据治理、元数据管理和 AI Agent 平台打通——这已经不是买 BI 工具，而是在买"AI 时代的数据操作系统"。

**对开源的含义**：商业巨头在用并购锁定企业客户的生态依赖；开源工具的竞争优势不在功能对齐，而在于不锁定——这是不同的价值主张，不是同赛道的竞争。

---

## 三、语义层战争：2025 年最重要的架构博弈

你的参考文档讲了 MetricFlow、Cube 等独立语义层工具。但 2025 年发生了一件更大的事：**数据仓库厂商直接把语义层内置进来了**，这是一次根本性的架构入侵。

### Snowflake Semantic Views（GA：2025 年 8 月）

Snowflake 在 2025 年 Summit 发布了 `SEMANTIC VIEW`——一种新的数据库对象类型，和表、视图、物化视图并列。它把业务语义（实体、关系、维度、指标）直接存在 Snowflake 的元数据目录里，无需任何外部工具。

```sql
CREATE OR REPLACE SEMANTIC VIEW sales_analytics
  TABLES (
    cust AS sales.customers PRIMARY KEY (cust_id),
    ord  AS sales.orders    PRIMARY KEY (order_id)
  )
  RELATIONSHIPS ( ord (cust_id) REFERENCES cust (cust_id) )
  METRICS (
    total_revenue AS SUM(unit_price * qty) COMMENT 'Gross revenue'
  );
```

**核心诉求**：零依赖、零网络跳转、任何 SQL 客户端立即可用。

### Databricks Metric Views（2025 年 GA）

Databricks 在 Unity Catalog 里引入了 Metric Views，把指标定义为 Delta Lake 之上的一等公民对象，在 Databricks SQL 和 Notebook 里直接可见。核心目标：统一数据科学、ML 和 BI 的语义定义。

### 这场博弈的本质矛盾

两家仓库巨头做的是同一个赌注：**语义层应该住在数据仓库里，而不是外部中间件**。这直接威胁了 MetricFlow、Cube 等独立语义层工具的生存空间。

但这个赌注有一个致命弱点——**边界问题**。Snowflake 自己都意识到了：2025 年它发起了 **OSI（Open Semantic Interchange）** 倡议，试图制定语义模型跨平台流转的标准。一个公司不会推开放标准，如果它的内置方案已经够用的话。

现实是：企业数据往往不只在一个仓库里。仓库原生语义层只能覆盖自己里面的数据，仓库外的数据在语义上是"暗区"。MetricFlow 的多仓库支持（Snowflake、BigQuery、Databricks、Redshift、Postgres、Trino）在这里反而是优势。

### 2025 年三条路的对比

| 方案 | 多仓库 | 零依赖 | 开源 | AI 集成 | 适合谁 |
|------|--------|--------|------|---------|--------|
| dbt MetricFlow | ✅ 6 种仓库 | ❌ 需 dbt Cloud | ✅ | 中 | 多云/多仓库团队 |
| Snowflake Semantic Views | ❌ 仅 Snowflake | ✅ | ❌ | 强（Cortex AI） | 全押 Snowflake 的团队 |
| Databricks Metric Views | 有限 | ✅（Databricks 内） | 有限 | 强（Genie） | Lakehouse 团队 |

---

## 四、text-to-SQL 的真实准确性问题：学术研究说了什么

开源 AI 问数工具的宣传材料都在说"高准确率"，但学术研究的结论更冷静。

Apple 和滑铁卢大学 2025 年发表的研究《评估 Text2SQL 解决方案的根本挑战》指出了两个被严重低估的问题：

1. **自然语言的歧义性**：同一个业务问题可以合法对应多种不同的 SQL 查询。现有 benchmark 没有捕捉这种"一问多答"的概率性本质，导致准确率被高估。

2. **SQL 等价性的判定偏差**：用于评估 SQL 是否正确的匹配函数本身就是近似方法，会系统性地偏高或偏低估准确率。

更直白的结论是：**在学术 benchmark 上准确率很高的模型，在真实 BI 生产场景里表现往往远不及预期**。这是因为真实 BI 问题的 schema 更复杂、业务定义更模糊、问题更开放。

另一篇 2024 年 ICSOC 接收的论文专门构建了一个"BI 场景 NL2SQL benchmark"，结论是：现有通用 NL2SQL benchmark（如 WikiSQL、Spider）根本不适合评估 BI 场景——它们的数据结构和问题类型与真实 BI 查询差异太大。

**对 WrenAI 等工具的含义**：这恰好解释了为什么"语义层路线"比"纯 RAG 生成路线"更适合 BI 场景——通过约束查询空间而不是依赖模型猜对，规避了这个根本性的歧义问题。

---

## 五、被低估的工具：参考文档没有提到的

### Observable Framework（observablehq/framework）

D3.js 创始人 Mike Bostock 在 2024 年 3 月发布的新框架。采用响应式编程模型，在浏览器端执行计算，性能极高。定位和 Evidence 类似（代码化数据应用），但底层是 JavaScript 生态而非 SQL+Markdown。适合需要极强可视化自由度的数据新闻和数据应用团队。

### GoodData.CN

经常被忽略的 Headless BI 选项。开源核心，重点在：多租户（同一套 semantic model 服务多个客户）、API-first 架构、内存加速引擎。2025 年新发布的"Logical Data Modeler"用 YAML 简化了语义建模流程。适合要把分析能力嵌进 SaaS 产品、需要强多租户隔离的产品团队。

### Cube 0.35（2025 年）

Cube 在 2025 年发布的 0.35 版本带来了两个重要新特性：
- **WASM-based SQL transpilation**：亚秒级查询，无需预聚合
- **Authz SDK**：行级安全的细粒度权限控制

这让 Cube 在嵌入式分析场景的竞争力显著提升。

### Elementary Data / Soda Core

数据质量观测工具，通常不被归为 BI，但 2025 年它们越来越重要：**AI Agent 如果消费了质量差的数据，会自信地给出错误答案**。Elementary（dbt 原生）和 Soda Core 是在数据进入语义层之前的质量守门人，是 AI 问数链路里被严重低估的一环。

---

## 六、Excel 没有死：一个令人不舒服的事实

BI 社区有一种集体倾向——把 Excel 当成"需要被取代的遗产"。但现实更复杂。

一项对 20 个财务团队的访谈研究发现：**手工 Excel 流程仍然是拖累财务团队的最大隐性挑战**。一家 1.1 亿美元规模的设备租赁公司已经找了三年能替代 Excel 的工具，还没找到。

Excel 不会死的真正原因不是用户保守，而是几个结构性优势：

- **可确定性**：公式结果是确定的、可追溯的。BI Dashboard 的数字有时来源不透明。
- **即时可操作性**：可以手工修改数据，做假设推演（What-if）。
- **无需 IT**：不需要部署、不需要数据管道，打开就用。
- **Microsoft Copilot 加持**：2025 年 9 月，Excel Copilot 开放了 Anthropic Claude 4 和 OpenAI 推理模型——Excel 正在被 AI 强化，而不是被 AI 替代。

**对 BI 工具设计的含义**：真正威胁 Excel 的 BI 工具，必须在"即时性"和"可操控性"上接近 Excel，而不是在"功能更多"上赢。Metabase 的无 SQL 查询构建器是对的方向，但还不够。

---

## 七、开源 BI 的真实增长数据

- 2024 年 IBM 研究：使用开源 AI 和 BI 工具的企业中，**51% 报告了正向 ROI**，而使用专有平台的只有 41%。
- **近半数**受调企业计划在 2025 年增加开源 BI 工具的使用。
- Gartner 预测：到 **2027 年，超过 70% 的企业将把语义层作为 AI 栈的组成部分**部署。

---

## 八、一个根本性的架构争论：text-to-SQL vs text-to-Semantic-Layer

这是 2025 年 BI 圈里最重要、但讨论得最不公开的技术争论。

**text-to-SQL 阵营**（Vanna、早期 Snowflake/Databricks 方案）的逻辑：给 LLM 足够的上下文（DDL、schema 描述、示例 SQL），它就能生成正确的 SQL。

**text-to-Semantic-Layer 阵营**（WrenAI、Cube、Delphi）的逻辑：把业务问题翻译成对语义层的查询请求，由语义层负责生成 SQL。LLM 只需要理解业务意图，不需要理解底层表结构。

一位深度参与这个领域的从业者（David SJ，Delphi 创始人）在 2025 年写道：

> Snowflake 和 Databricks 最初也用 text-to-SQL 方法，给 LLM 喂 YAML 上下文来提升质量。结果确实好一些，但永远不可能接近 text-to-Semantic-Layer 的准确率。这就是为什么它们在 2025 年都转向了内置语义层——这是方向性的承认，不是功能迭代。

这个争论的结论影响深远：如果 text-to-Semantic-Layer 确实在准确率上有结构性优势，那么没有语义层的 NL2SQL 工具（无论 RAG 多精致）都面临天花板。

---

## 九、新的隐患：语义层碎片化

当每个工具都开始定义"语义层"，术语本身开始失去意义。

- Snowflake 有 Semantic Views
- Databricks 有 Metric Views
- dbt 有 MetricFlow
- Cube 有自己的 schema
- Lightdash 有 YAML 指标定义
- WrenAI 有 MDL
- LookML 有 Explores

这些定义不兼容。一个在 dbt 里定义的指标，不能直接在 Snowflake Semantic Views 里复用。语义模型成了数据团队的新孤岛。

Snowflake 发起的 **OSI（Open Semantic Interchange）** 倡议试图解决这个问题，但目前仍是早期阶段。这是 2026 年之后最值得关注的架构演变方向：谁能定义语义层的互操作标准，谁就掌握了下一代数据基础设施的话语权。

---

## 十、研究结论：三个被低估的判断

**判断一：仓库原生语义层是对独立语义层工具最大的威胁，但它有边界约束。**
Snowflake 和 Databricks 内置语义层会吸走大量原本可能使用 MetricFlow/Cube 的用户，但只适用于单一仓库场景。多云/多仓库的企业（这是大多数大型企业的现实）仍然需要独立的语义层。

**判断二：text-to-SQL 路线有准确率天花板，text-to-Semantic-Layer 是正确方向，但语义层建设本身是瓶颈。**
WrenAI 的技术路线是对的，但它依赖的前置条件——完善的语义层定义——在大多数企业里还没有。这不是 WrenAI 的问题，是行业的整体成熟度问题。

**判断三：数据质量是 AI 问数的隐形杀手，但几乎没有工具在这个链路上认真对待它。**
如果输入语义层的数据本身有质量问题（缺值、口径不一、延迟），AI Agent 会给出自信但错误的答案。Elementary Data、Soda Core 等数据质量工具应该成为 AI 问数链路的标配，但目前很少有团队把它们串联起来。

---

*研究日期：2025 年 5 月 | 信息来源：GitHub、学术论文（arXiv、ICSOC 2024）、行业分析（AppsRunTheWorld、Typedef、Paradime）、厂商文档（Snowflake、Databricks）*
