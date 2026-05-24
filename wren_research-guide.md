# WrenAI Research Guide

本文档用于系统性研究 WrenAI 项目，目标是：

1. 彻底理解 WrenAI 的定位、核心概念和运行链路。
2. 看懂核心源码结构，能定位 CLI、Engine、MDL、Memory、Connector、SDK 的关键实现。
3. 最终能够把 WrenAI 作为上下文层接入自己的 Agent。

建议按阶段推进。每个阶段都包含目标、重点文件、验证任务和产出物。

## 0. 项目定位

WrenAI 当前主线不是传统 BI 应用，而是面向 AI Agent 的业务数据上下文层。它把数据库 schema、业务语义、查询记忆、治理规则和 Agent 工具封装成可组合的 primitives。

核心路径：

```text
用户问题
  -> Agent 规划
  -> Wren memory 获取相关 schema / 历史 query
  -> LLM 生成 SQL
  -> Wren dry-plan / dry-run 校验
  -> Wren query 执行
  -> Agent 总结结果
  -> 可选：store 成功的 NL-SQL 到 memory
```

WrenAI 不是替 Agent 做完整推理，而是给 Agent 提供：

- MDL 语义模型
- SQL planning / validation / execution
- 数据源 connector
- schema memory 和 query memory
- LangChain / Pydantic AI SDK
- 给编码 Agent 使用的 skills

## 1. 第一阶段：建立全局地图

### 目标

理解仓库为什么这样拆分，每个模块承担什么职责。

### 重点阅读

- `README.md`
- `docs/core/concepts/what_is_context.md`
- `docs/core/concepts/architecture.md`
- `docs/core/sdk/overview.md`
- `docs/core/reference/cli.md`

### 顶层结构

```text
core/
  wren/              Python CLI + Python SDK，PyPI 包名 wren-engine
  wren-core/         Rust semantic engine，基于 Apache DataFusion
  wren-core-base/    Manifest / MDL 基础类型与 builder
  wren-core-py/      Rust core 的 Python binding
  wren-core-wasm/    WebAssembly 构建
  wren-mdl/          MDL JSON Schema
sdk/
  wren-langchain/    LangChain / LangGraph Agent 适配
  wren-pydantic/     Pydantic AI Agent 适配
skills/
  wren-onboarding/       项目初始化流程
  wren-usage/            日常查询流程
  wren-generate-mdl/     MDL 生成流程
  wren-dlt-connector/    dlt connector 流程
docs/
  core/             文档源码
```

### 需要回答的问题

- WrenAI 和传统 semantic layer 的区别是什么？
- MDL、memory、profile、connector 分别解决什么问题？
- WrenEngine 在整个链路中处于什么位置？
- SDK 为什么要求先通过 CLI 准备 Wren project？

### 阶段产出

写一页自己的架构笔记，画出：

```text
Agent -> SDK/CLI -> WrenEngine -> SQL planning -> Connector -> Database
                    |
                    -> Memory
                    |
                    -> MDL / Manifest
```

## 2. 第二阶段：跑通最小闭环

### 目标

不要先陷入源码细节。先跑通一个可以查询的 Wren project，建立体感。

### 推荐实验

优先使用 DuckDB 或项目内 sample，避免先被数据库部署阻塞。

可按文档流程执行：

```bash
pip install -e "core/wren[memory,main]"
wren context init
wren context build
wren profile add local --interactive
wren dry-plan --sql 'SELECT * FROM "orders" LIMIT 5'
wren --sql 'SELECT * FROM "orders" LIMIT 5'
wren memory index
wren memory fetch -q "top customers by revenue"
```

实际命令需要根据本机环境和样例数据调整。

### 重点观察

- `wren_project.yml` 如何描述项目。
- `models/`、`views/`、`relationships.yml` 如何被编译。
- `target/mdl.json` 的结构。
- `~/.wren/profiles.yml` 的 profile 存储格式。
- `.wren/memory/` 或默认 memory path 中生成了什么。

### 阶段产出

保存一份最小可运行 Wren project，并记录：

- 输入 SQL
- `dry-plan` 输出 SQL
- 实际查询结果
- memory fetch 返回的上下文

## 3. 第三阶段：研究 Python CLI 和 Engine

### 目标

搞清用户命令如何进入 WrenEngine，以及 plan / execute 的核心链路。

### 重点文件

- `core/wren/src/wren/cli.py`
- `core/wren/src/wren/engine.py`
- `core/wren/src/wren/context.py`
- `core/wren/src/wren/context_cli.py`
- `core/wren/src/wren/profile.py`
- `core/wren/src/wren/profile_cli.py`
- `core/wren/src/wren/model/error.py`
- `core/wren/src/wren/policy.py`
- `core/wren/src/wren/sql_classify.py`

### 阅读顺序

1. 从 `cli.py` 看 Typer app 如何注册命令。
2. 跟踪 `wren --sql` / `wren query` 到 `WrenEngine`。
3. 看 `engine.py` 中 plan、dry-plan、dry-run、execute 的区别。
4. 看 profile 是如何加载并解析环境变量的。
5. 看 policy 如何限制 strict mode 和 denied functions。

### 需要回答的问题

- CLI 如何自动发现当前 Wren project？
- `target/mdl.json` 在什么时候读取？
- `dry-plan` 是否需要数据库连接？
- `dry-run` 和 `query` 的差别是什么？
- Wren 的错误如何被结构化并返回给 SDK / Agent？

### 推荐调试方式

在本地最小项目中对以下命令加断点或打印：

```bash
wren dry-plan --sql 'SELECT * FROM "orders" LIMIT 5'
wren dry-run --sql 'SELECT * FROM "orders" LIMIT 5'
wren --sql 'SELECT * FROM "orders" LIMIT 5'
```

## 4. 第四阶段：研究 MDL 和 Manifest

### 目标

理解 Wren 的业务语义如何从 YAML / JSON 进入 Rust core。

### 重点文件

- `core/wren-mdl/mdl.schema.json`
- `core/wren-core-base/src/mdl/manifest.rs`
- `core/wren-core-base/src/mdl/builder.rs`
- `core/wren-core-base/src/mdl/migration.rs`
- `core/wren-core-base/src/mdl/cls.rs`
- `core/wren/src/wren/context.py`

### 重点概念

- Model
- Column
- Relationship
- View
- Cube
- Metric
- Row-level / column-level access control
- `table_reference`
- `ref_sql`
- calculated fields

### 需要回答的问题

- YAML 项目格式和最终 `mdl.json` 是如何对应的？
- snake_case YAML 如何转换为 camelCase JSON？
- model、view、relationship 的引用关系在哪里校验？
- migration 代码支持哪些 schema version？
- MDL 中哪些字段对 Agent 最重要？

### 阶段产出

手动写一个最小 MDL，至少包含：

- 2 个 model
- 1 个 relationship
- 1 个 calculated column
- 1 个 view 或 cube

然后运行：

```bash
wren context validate
wren context build
wren dry-plan --sql '...'
```

## 5. 第五阶段：研究 Rust Core

### 目标

理解 Wren 如何把语义模型展开成可执行 SQL。这是项目最核心的部分。

### 重点文件

- `core/wren-core/core/src/lib.rs`
- `core/wren-core/core/src/mdl/mod.rs`
- `core/wren-core/core/src/mdl/context.rs`
- `core/wren-core/core/src/mdl/dataset.rs`
- `core/wren-core/core/src/mdl/cube.rs`
- `core/wren-core/core/src/mdl/type_planner.rs`
- `core/wren-core/core/src/mdl/lineage.rs`
- `core/wren-core/core/src/logical_plan/mod.rs`
- `core/wren-core/core/src/logical_plan/unparser.rs`
- `core/wren-core-py/`

### 阅读顺序

1. 看 `lib.rs` 暴露了哪些核心 API。
2. 找到 `SessionContext` 和 `transform_sql()`。
3. 跟踪 Manifest 如何被加载进 Rust core。
4. 跟踪一个 model SQL 如何被展开成物理 SQL。
5. 看 relationship join、calculated field、cube 的处理。
6. 看 `wren-core-py` 如何把 Rust API 暴露给 Python。

### 推荐实验

在 `core/wren-core` 下运行测试：

```bash
cargo test
```

重点研究：

- unit test 中的输入 MDL
- 输入 SQL
- 期望输出 SQL
- sqllogictest 覆盖的端到端场景

### 需要回答的问题

- Rust core 和 Python `sqlglot` 各自负责哪部分 SQL planning？
- ManifestExtractor 为什么要过滤 referenced models？
- CTE 注入发生在 Python 还是 Rust？
- DataFusion 在这里主要承担什么角色？
- 如果 SQL 转换失败，错误如何传递到 Python？

## 6. 第六阶段：研究 Connector 层

### 目标

理解 Wren 如何连接不同数据库，以及如何把 planned SQL 执行成 PyArrow table。

### 重点文件

- `core/wren/src/wren/connector/base.py`
- `core/wren/src/wren/connector/factory.py`
- `core/wren/src/wren/connector/duckdb.py`
- `core/wren/src/wren/connector/postgres.py`
- `core/wren/src/wren/connector/mysql.py`
- `core/wren/src/wren/connector/bigquery.py`
- `core/wren/src/wren/type_mapping.py`
- `core/wren/src/wren/model/data_source.py`

### 需要回答的问题

- Connector 的共同接口是什么？
- 每个 connector 如何处理 dialect、limit、dry-run？
- 数据库结果如何转换为 PyArrow？
- connection profile 中的字段如何映射到 connector 参数？
- 新增一个数据库 connector 需要改哪些地方？

### 阶段产出

写一份“新增 connector checklist”，至少包含：

- profile schema
- connector class
- factory 注册
- datasource enum
- type mapping
- tests
- docs

## 7. 第七阶段：研究 Memory

### 目标

理解 Wren 如何为 Agent 提供 schema retrieval 和 NL-SQL few-shot recall。

### 重点文件

- `core/wren/src/wren/memory/cli.py`
- `core/wren/src/wren/memory/store.py`
- `core/wren/src/wren/memory/schema_indexer.py`
- `core/wren/src/wren/memory/seed_queries.py`
- `core/wren/src/wren/memory/embeddings.py`
- `docs/core/concepts/memory_system.md`
- `docs/core/guides/memory.md`

### 重点命令

```bash
wren memory describe
wren memory index
wren memory fetch -q "..."
wren memory store --nl "..." --sql "..."
wren memory recall -q "..."
wren memory status
```

### 需要回答的问题

- 小 schema 为什么直接返回全文，而不是 embedding search？
- 大 schema 的 schema_items 如何切分？
- query_history 存哪些字段？
- memory 的 datasource filter 如何工作？
- Agent 应该何时 fetch context，何时 recall query？
- 成功查询应不应该自动 store？

### Agent 设计建议

Agent 生成 SQL 前：

1. 调用 `wren_list_models` 或 `wren_fetch_context`。
2. 调用 `wren_recall_queries` 获取类似问题。
3. 生成候选 SQL。
4. 调用 `wren_dry_plan`。
5. 必要时调用 `wren_dry_run`。
6. 执行 `wren_query`。
7. 用户确认后调用 `wren_store_query`。

## 8. 第八阶段：研究 SDK 对接 Agent

### 目标

理解现有 SDK 如何把 Wren 项目变成 Agent tools，并据此设计自己的 Agent 接入方式。

### LangChain / LangGraph

重点文件：

- `sdk/wren-langchain/README.md`
- `sdk/wren-langchain/src/wren_langchain/`
- `sdk/wren-langchain/examples/langchain_demo.py`
- `sdk/wren-langchain/examples/langgraph_demo.py`

核心入口：

```python
from wren_langchain import WrenToolkit

toolkit = WrenToolkit.from_project("./analytics_db")
tools = toolkit.get_tools()
prompt = toolkit.system_prompt()
```

### Pydantic AI

重点文件：

- `sdk/wren-pydantic/README.md`
- `sdk/wren-pydantic/src/wren_pydantic/`
- `sdk/wren-pydantic/examples/pydantic_ai_demo.py`
- `sdk/wren-pydantic/examples/pydantic_ai_structured_demo.py`

核心入口：

```python
from wren_pydantic import WrenToolkit

toolkit = WrenToolkit.from_project("./analytics_db")
toolset = toolkit.toolset()
instructions = toolkit.instructions()
```

### 需要回答的问题

- SDK 是否持有长期状态？
- `target/mdl.json` 是否会热更新？
- profile 如何选择？
- memory tools 什么时候暴露？
- tool 的异常如何反馈给 Agent？
- read-only 模式如何实现？

### 阶段产出

选择一个 Agent 框架，实现最小 demo：

```text
用户输入自然语言问题
  -> Agent 获取 Wren tools
  -> memory fetch / recall
  -> 生成 SQL
  -> dry-plan
  -> query
  -> 自然语言总结
```

## 9. 第九阶段：研究 Skills

### 目标

理解 Wren 如何指导编码 Agent 自动完成 onboarding、MDL 生成和查询流程。

### 重点文件

- `skills/README.md`
- `skills/SKILLS.md`
- `skills/wren-onboarding/SKILL.md`
- `skills/wren-usage/SKILL.md`
- `skills/wren-generate-mdl/SKILL.md`
- `skills/wren-dlt-connector/SKILL.md`

### 需要回答的问题

- skill 和 SDK 的边界是什么？
- skill 中哪些步骤适合自动化，哪些必须让用户确认？
- onboarding 如何检测环境、安装依赖、创建项目？
- generate-mdl 如何从数据库结构推导业务语义？
- 如果你要给自己的 Agent 写 Wren skill，需要包含哪些安全约束？

## 10. 第十阶段：面向自己的 Agent 做集成设计

### 目标

形成可落地的 Agent 接入方案。

### 架构选择

优先考虑三种方式：

1. 直接使用 `wren-langchain`。
2. 直接使用 `wren-pydantic`。
3. 自己封装 `wren-engine` Python SDK 或 CLI。

选择标准：

| 方式 | 适合场景 |
|---|---|
| `wren-langchain` | 已使用 LangChain / LangGraph，需要快速接工具 |
| `wren-pydantic` | 使用 Pydantic AI，希望工具输入输出强类型 |
| 直接封装 `wren-engine` | 自研 Agent 框架，需要完全控制 tool schema、错误处理和执行策略 |
| CLI 调用 | 原型验证、低耦合、跨语言接入 |

### 推荐 Agent 工作流

```text
1. 接收用户问题
2. 判断是否是数据查询类问题
3. 调用 wren_fetch_context 获取相关模型/字段/关系
4. 调用 wren_recall_queries 获取相似 NL-SQL
5. 若问题含糊，先向用户澄清
6. 生成 SQL
7. 调用 wren_dry_plan 检查语义展开
8. dry-plan 失败则修复 SQL
9. 可选调用 wren_dry_run 做数据库侧校验
10. 调用 wren_query 执行
11. 总结结果，附上 SQL 或执行摘要
12. 用户确认结果正确后，调用 wren_store_query
```

### Agent 安全策略

建议默认启用：

- 只允许 SELECT / read-only 查询。
- 每次执行前必须 dry-plan。
- 大查询必须强制 limit。
- 禁止危险函数。
- profile 绑定到项目，不允许 Agent 任意切换生产库。
- memory store 需要用户确认。
- 对错误进行分类：SQL 语法错误、模型不存在、字段不存在、连接失败、权限失败。

### 需要设计的接口

你的 Agent 至少需要抽象以下工具：

```text
wren_list_models()
wren_fetch_context(question)
wren_recall_queries(question)
wren_dry_plan(sql)
wren_dry_run(sql)
wren_query(sql)
wren_store_query(question, sql, tags)
```

每个工具都应该定义：

- 输入 schema
- 输出 schema
- 错误类型
- 是否可重试
- 是否需要用户确认
- 是否允许在生产环境执行

## 11. 推荐源码阅读路线总表

```text
README.md
  -> docs/core/concepts/architecture.md
  -> core/wren/src/wren/cli.py
  -> core/wren/src/wren/engine.py
  -> core/wren/src/wren/context.py
  -> core/wren-core-base/src/mdl/manifest.rs
  -> core/wren-core/core/src/lib.rs
  -> core/wren-core/core/src/mdl/
  -> core/wren/src/wren/connector/
  -> core/wren/src/wren/memory/
  -> sdk/wren-langchain/src/wren_langchain/
  -> sdk/wren-pydantic/src/wren_pydantic/
  -> skills/*/SKILL.md
```

## 12. 深入研究检查清单

完成以下项目后，可以认为已经基本搞清 WrenAI：

- [ ] 能解释 WrenAI 的 context layer 定位。
- [ ] 能从 CLI 命令追踪到 `WrenEngine`。
- [ ] 能解释 `dry-plan`、`dry-run`、`query` 的差异。
- [ ] 能手写一个最小 Wren project。
- [ ] 能解释 `wren_project.yml` 到 `target/mdl.json` 的构建流程。
- [ ] 能解释 MDL 中 model、relationship、view、cube 的关系。
- [ ] 能追踪一条 SQL 如何被展开为物理 SQL。
- [ ] 能说明 Python planning、Rust core、DataFusion 的职责边界。
- [ ] 能理解 connector interface 并知道如何新增 connector。
- [ ] 能解释 memory index / fetch / recall / store。
- [ ] 能运行 LangChain 或 Pydantic AI demo。
- [ ] 能设计自己的 Agent tool schema。
- [ ] 能制定 query 安全策略和错误重试策略。
- [ ] 能判断什么时候需要用户澄清，而不是直接生成 SQL。
- [ ] 能将成功的 NL-SQL 流程沉淀进 memory。

## 13. 最终集成里程碑

### Milestone 1：本地 Wren project 可查询

完成标准：

- 有可运行的 `wren_project.yml`
- 有 `target/mdl.json`
- 有可用 profile
- `wren dry-plan` 和 `wren query` 成功

### Milestone 2：Memory 可用

完成标准：

- `wren memory index` 成功
- `wren memory fetch` 能返回相关 schema
- `wren memory store` / `recall` 能保存和召回 NL-SQL

### Milestone 3：Agent 最小接入

完成标准：

- Agent 能调用 Wren tools
- 能完成自然语言到 SQL 到结果总结
- 执行前有 dry-plan
- 查询失败时能根据错误修复 SQL

### Milestone 4：生产化约束

完成标准：

- 只读查询策略生效
- 默认 limit 生效
- profile 权限边界明确
- 记录 SQL、结果摘要、错误和用户确认状态
- memory 写入需要确认或审核

### Milestone 5：可持续优化

完成标准：

- 有 golden questions / expected SQL
- 有回归测试
- 有业务术语和指标维护流程
- 有 MDL 变更 review 流程
- 有 Agent 失败样本分析流程

## 14. 建议的研究节奏

如果时间充足，建议按 2 周节奏：

| 时间 | 重点 |
|---|---|
| Day 1-2 | 阅读 README、architecture、context、CLI 文档 |
| Day 3-4 | 跑通最小 Wren project |
| Day 5-6 | 研究 Python CLI / Engine |
| Day 7-8 | 研究 MDL / Manifest / Rust core |
| Day 9 | 研究 Connector |
| Day 10 | 研究 Memory |
| Day 11 | 研究 SDK |
| Day 12 | 研究 Skills |
| Day 13 | 实现 Agent 最小 demo |
| Day 14 | 补安全策略、测试和集成设计文档 |

如果时间紧，最短路径是：

```text
README
  -> architecture
  -> 跑通 wren context/profile/query/memory
  -> sdk/wren-langchain 或 sdk/wren-pydantic demo
  -> 再回头看 engine.py 和 Rust core
```

