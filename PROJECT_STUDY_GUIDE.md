# WrenAI 项目研究指南

这份文档用于帮助你系统性研究 WrenAI。它不是简单的目录清单，而是按“先理解产品目标，再跑通主链路，再深入核心模块，最后形成可修改能力”的顺序组织。

## 1. 先建立全局认知

WrenAI 的定位是面向 AI Agent 的开放上下文层。它把数据库 schema、业务语义、示例查询、记忆、权限和 SQL 执行能力包装成 Agent 可用的能力，让 Agent 不直接猜测真实数据库结构，而是通过 MDL 语义层来写 SQL、校验 SQL、执行 SQL。

从仓库结构看，它不是一个传统单体 Web 应用，而是几个组件组合起来的系统：

```text
core/
  wren-core/         Rust 语义引擎，基于 Apache DataFusion
  wren-core-base/    Rust 共享类型、Manifest/MDL 基础结构
  wren-core-py/      Rust 引擎的 Python 绑定，包名 wren-core-py / 模块名 wren_core
  wren-core-wasm/    Rust 引擎的 WebAssembly 构建，给浏览器/Node 使用
  wren/              Python CLI 和 SDK，包名 wrenai，命令行入口 wren
  wren-mdl/          MDL JSON schema
sdk/
  wren-langchain/    LangChain / LangGraph 集成
  wren-pydantic/     Pydantic AI 集成
skills/
  Agent skills，指导 Cursor、Claude Code 等 Agent 使用 wren CLI
docs/
  文档站源文件，尤其是 docs/core
```

建议先把 WrenAI 理解成四层：

1. **语义建模层**：用 MDL 描述模型、字段、关系、视图、指标、访问控制等。
2. **SQL 规划层**：把用户写给“模型”的 SQL 转换成真实数据源能执行的 SQL。
3. **连接执行层**：通过不同 connector 连接 Postgres、MySQL、BigQuery、Snowflake、DuckDB 等数据源。
4. **Agent 接入层**：CLI、SDK、skills、memory，把语义层暴露给 AI Agent 使用。

## 2. 第一轮阅读顺序

第一轮目标不是看懂每行代码，而是回答三个问题：

- 用户如何使用 WrenAI？
- 一条 SQL 从输入到执行经过哪些模块？
- 如果我要改一个功能，应该从哪里下手？

推荐顺序如下。

### 2.1 从产品和概念开始

先读：

- `README.md`
- `docs/core/README.md`
- `docs/core/introduction.mdx`
- `docs/core/concepts/what_is_context.md`
- `docs/core/concepts/what_is_mdl.md`
- `docs/core/concepts/architecture.md`

重点关注这些概念：

- Context：为什么 AI Agent 需要业务上下文。
- MDL：WrenAI 的核心语义描述格式。
- Memory：如何存储和召回 schema 上下文、历史 NL-to-SQL 查询。
- Correctness primitives：dry-plan、访问控制、结构化错误、row limit 等正确性手段。

### 2.2 从 CLI 使用链路开始

然后读 `core/wren/README.md`，并把用户路径记下来：

```bash
wren context init
wren profile add my-db --ui
wren context build
wren --sql 'SELECT ...'
wren dry-plan --sql 'SELECT ...'
wren memory index
wren memory fetch -q "customer order price"
wren memory recall -q "top customers"
```

这一层的关键文件：

- `core/wren/src/wren/cli.py`：主 CLI 入口，Typer app，负责解析 `wren` 命令。
- `core/wren/src/wren/context_cli.py`：`wren context ...` 命令。
- `core/wren/src/wren/context.py`：YAML 项目和 MDL JSON 的转换、校验、构建。
- `core/wren/src/wren/profile_cli.py`：连接 profile 的命令。
- `core/wren/src/wren/profile.py`：profile 解析、环境变量 secret 展开。
- `core/wren/src/wren/memory/cli.py`：memory 子命令。

这一步的研究产物应该是一张“CLI 命令到代码文件”的映射表。

### 2.3 跑通一条 SQL 的主链路

重点读：

- `core/wren/src/wren/engine.py`
- `core/wren/src/wren/mdl/__init__.py`
- `core/wren/src/wren/mdl/cte_rewriter.py`
- `core/wren/src/wren/connector/factory.py`
- `core/wren/src/wren/connector/base.py`
- `core/wren/src/wren/connector/duckdb.py`
- `core/wren/src/wren/connector/postgres.py`

主链路可以先这样理解：

```text
用户 SQL
  -> wren CLI 解析参数
  -> 加载 target/mdl.json 或显式 --mdl
  -> 解析 profile / connection_info
  -> 创建 WrenEngine
  -> WrenEngine.dry_plan()
  -> sqlglot 解析目标 SQL
  -> 从 MDL 中抽取相关模型
  -> wren-core 创建 SessionContext
  -> CTERewriter 把模型语义展开成目标数据源 SQL
  -> connector.query() 执行
  -> 返回 pyarrow.Table / CLI 输出
```

`WrenEngine` 是 Python 层最重要的门面。它把 Rust core、SQL rewrite、connector 和 policy 串起来。研究时优先看：

- `dry_plan()`：只规划，不访问数据库。
- `query()`：先 dry-plan，再通过 connector 执行。
- `_plan()`：真正处理 SQL 解析、MDL 抽取、SessionContext 和 rewrite。
- `_get_connector()`：按数据源懒加载 connector。

### 2.4 再看 Rust 语义核心

Rust 核心在：

- `core/wren-core/`
- `core/wren-core-base/`
- `core/wren-core-py/`
- `core/wren-core-wasm/`

阅读顺序：

1. `core/wren-core/README.md`
2. `core/wren-core/Cargo.toml`
3. `core/wren-core/core/src/lib.rs`
4. `core/wren-core/core/src/mdl/mod.rs`
5. `core/wren-core/core/src/mdl/context.rs`
6. `core/wren-core/core/src/logical_plan/mod.rs`
7. `core/wren-core/core/src/logical_plan/analyze/`
8. `core/wren-core/core/src/logical_plan/optimize/`
9. `core/wren-core/core/src/logical_plan/unparser.rs`

Rust 层重点不是一开始就追所有 DataFusion 细节，而是先搞清楚：

- Manifest / MDL 在 Rust 里如何表示。
- SessionContext 如何加载 MDL。
- `transform_sql` 或等价逻辑如何把模型 SQL 转成真实 SQL。
- logical plan analyze 做了哪些语义展开。
- 不同 SQL dialect 的函数、类型、unparse 是如何处理的。

### 2.5 看 Python 绑定

`core/wren-core-py` 是 Rust core 和 Python SDK/CLI 之间的桥。

重点文件：

- `core/wren-core-py/README.md`
- `core/wren-core-py/pyproject.toml`
- `core/wren-core-py/src/`
- `core/wren-core-py/tests/`

关注点：

- PyO3 / maturin 如何把 Rust 编译成 Python 模块 `wren_core`。
- Python 层调用的函数和类有哪些。
- `wren_core` 暴露给 `core/wren` 的 API 边界是什么。

### 2.6 看 WASM 构建

`core/wren-core-wasm` 用于浏览器或 Node 场景。

重点文件：

- `core/wren-core-wasm/README.md`
- `core/wren-core-wasm/package.json`
- `core/wren-core-wasm/sdk/`
- `core/wren-core-wasm/examples/`

关注点：

- Rust core 如何编译到 WASM。
- JS/TS 层的 `WrenEngine` API 如何设计。
- 浏览器中如何注册 JSON、CSV、Parquet。
- URL mode 为什么要求 HTTP Range。

### 2.7 最后看 Agent SDK 和 skills

Agent 集成在：

- `sdk/wren-langchain/`
- `sdk/wren-pydantic/`
- `skills/`

优先读：

- `sdk/wren-langchain/README.md`
- `sdk/wren-langchain/src/wren_langchain/_toolkit.py`
- `sdk/wren-langchain/src/wren_langchain/_tools.py`
- `sdk/wren-langchain/src/wren_langchain/_providers/`
- `sdk/wren-pydantic/README.md`
- `skills/README.md`
- `skills/wren-usage/SKILL.md`
- `skills/wren-generate-mdl/SKILL.md`
- `skills/wren-onboarding/SKILL.md`

这一层重点关注 WrenAI 如何暴露给 Agent：

- `wren_query`
- `wren_dry_plan`
- `wren_list_models`
- `wren_fetch_context`
- `wren_recall_queries`
- `wren_store_query`

## 3. 第二轮：按问题深入

第一轮建立地图后，第二轮建议按问题深入。

### 3.1 MDL 项目是如何构建的？

从这些文件开始：

- `core/wren/src/wren/context.py`
- `core/wren/src/wren/context_cli.py`
- `core/wren/src/wren/docs.py`
- `docs/core/reference/mdl.md`
- `docs/core/guides/model.md`
- `docs/core/guides/cubes.md`

需要搞清楚：

- `wren_project.yml` 的字段含义。
- `models/*/metadata.yml` 如何变成 MDL JSON。
- `views/*`、`relationships.yml`、`cubes/` 如何合并进 manifest。
- `target/mdl.json` 是怎么生成和发现的。
- schema version / layout version 如何转换。

### 3.2 SQL rewrite 是如何工作的？

从这些文件开始：

- `core/wren/src/wren/engine.py`
- `core/wren/src/wren/mdl/cte_rewriter.py`
- `core/wren/src/wren/mdl/wren_dialect.py`
- `core/wren/src/wren/policy.py`
- `core/wren-core/core/src/mdl/`
- `core/wren-core/core/src/logical_plan/`

建议用一个简单 SQL 做跟踪：

```sql
SELECT customer_id, SUM(total)
FROM "orders"
GROUP BY customer_id
```

研究时记录：

- 输入 SQL 用哪个 dialect 解析。
- SQL 里引用了哪些 model。
- 如何从完整 MDL 中抽取相关 model。
- wren-core 如何 transform。
- Python `CTERewriter` 如何把 transform 结果插回原 SQL。
- 最后生成的目标数据源 SQL 长什么样。

### 3.3 Connector 是如何扩展的？

从这些文件开始：

- `core/wren/src/wren/connector/factory.py`
- `core/wren/src/wren/connector/base.py`
- `core/wren/src/wren/model/data_source.py`
- `core/wren/src/wren/model/field_registry.py`
- `core/wren/src/wren/connector/*.py`

重点问题：

- 新增一个 datasource 要改哪些地方。
- connection_info 如何校验。
- connector 的 `query()`、`dry_run()`、`close()` 约定是什么。
- 依赖为什么按 extra 分组，例如 `wrenai[postgres]`、`wrenai[bigquery]`。

### 3.4 Memory 是如何工作的？

从这些文件开始：

- `core/wren/src/wren/memory/`
- `docs/core/concepts/memory_system.md`
- `docs/core/guides/memory.md`
- `skills/wren-usage/references/memory.md`

重点问题：

- schema context 如何 index。
- 自然语言问题如何 fetch 相关 schema。
- NL-to-SQL 示例如何 store / recall。
- memory 文件放在哪里。
- Agent SDK 如何决定是否暴露 memory tools。

### 3.5 Agent SDK 是如何包 CLI/engine 能力的？

从这些文件开始：

- `sdk/wren-langchain/src/wren_langchain/_toolkit.py`
- `sdk/wren-langchain/src/wren_langchain/_tools.py`
- `sdk/wren-langchain/src/wren_langchain/_providers/`
- `sdk/wren-pydantic/src/wren_pydantic/_toolkit.py`
- `sdk/wren-pydantic/src/wren_pydantic/_tools.py`

重点问题：

- `WrenToolkit.from_project(path)` 如何找到 MDL、profile 和 `.env`。
- tools 的输入输出格式如何设计。
- 错误如何包装给 Agent。
- system prompt 如何根据项目和 memory 动态生成。

## 4. 建议实际跑通的实验

建议不要只读代码，至少做这些小实验。

### 4.1 跑通 CLI 项目初始化

在临时目录中：

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e /home/zym/WrenAI/core/wren
wren context init
wren context validate
wren context build
```

观察生成：

- `wren_project.yml`
- `models/`
- `views/`
- `target/mdl.json`
- `AGENTS.md`

### 4.2 用 DuckDB / sample 数据跑一条查询

如果项目提供 sample 或你准备一个简单 DuckDB 表，重点验证：

```bash
wren dry-plan --sql 'SELECT * FROM "orders" LIMIT 10'
wren --sql 'SELECT * FROM "orders" LIMIT 10'
```

dry-plan 比 query 更适合研究，因为它能让你先观察 SQL rewrite，不需要把所有数据库连接细节都跑通。

### 4.3 修改一个 model 字段再 build

改 `models/<model>/metadata.yml` 后运行：

```bash
wren context validate
wren context build
```

然后对比 `target/mdl.json` 变化，理解 YAML 到 MDL 的映射。

### 4.4 跟踪一个 connector

建议先看 DuckDB 或 Postgres connector，因为它们比较常见。记录：

- connector 如何建立连接。
- SQL 如何提交给底层库。
- 查询结果如何转成 Arrow table。
- limit 在哪一层处理。

### 4.5 跑 SDK demo

LangChain 方向：

```bash
cd /home/zym/WrenAI/sdk/wren-langchain
pip install -e ".[dev]"
pytest
```

Pydantic 方向：

```bash
cd /home/zym/WrenAI/sdk/wren-pydantic
pip install -e ".[dev]"
pytest
```

如果依赖较重，先只读测试文件，也能快速理解公共 API 的预期行为。

## 5. 测试和质量检查入口

这个仓库有多套技术栈，测试也分散。

### Python CLI / SDK

位置：

- `core/wren/tests/`
- `sdk/wren-langchain/tests/`
- `sdk/wren-pydantic/tests/`

常用命令：

```bash
cd /home/zym/WrenAI/core/wren
pytest
ruff check .
ruff format --check .
```

### Rust core

位置：

- `core/wren-core/`
- `core/wren-core/core/src/`
- `core/wren-core/sqllogictest/`

常用命令：

```bash
cd /home/zym/WrenAI/core/wren-core
cargo test
cargo fmt --check
```

### Python binding

位置：

- `core/wren-core-py/`

常用命令：

```bash
cd /home/zym/WrenAI/core/wren-core-py
just develop
just test
```

如果没有 `just`，先看 `justfile` 中命令展开，手动执行 maturin / pytest。

### WASM

位置：

- `core/wren-core-wasm/`

常用命令：

```bash
cd /home/zym/WrenAI/core/wren-core-wasm
npm install
npm run build
npm test
```

### Skills

位置：

- `skills/`

常用命令：

```bash
cd /home/zym/WrenAI
bash skills/check-versions.sh
```

## 6. 推荐研究节奏

### 第 1 天：建立地图

- 读 `README.md` 和 `docs/core` 的概念文档。
- 画出四层架构：MDL、SQL planning、connector、Agent integration。
- 列出每层入口文件。

### 第 2 天：跑通 CLI

- 安装 `core/wren` 本地 editable 包。
- 跑 `wren context init/build`。
- 看 `target/mdl.json`。
- 用 dry-plan 跟踪一条 SQL。

### 第 3 天：深入 Python 主链路

- 精读 `cli.py`、`engine.py`、`context.py`、`cte_rewriter.py`。
- 写一份“SQL 从 CLI 到 connector”的调用链笔记。
- 看 `tests/unit/test_engine.py`、`tests/unit/test_context.py`。

### 第 4 天：深入 Rust core

- 看 `wren-core` 的 MDL 和 logical_plan 模块。
- 跑 `cargo test`。
- 找一个 sqllogictest 或单元测试，反推 transform 行为。

### 第 5 天：看 connector 和 profile

- 精读一个简单 connector。
- 理解 `DataSource`、`ConnectionInfo`、profile、`.env` 的关系。
- 总结新增 datasource 的步骤。

### 第 6 天：看 memory 和 Agent SDK

- 跑或阅读 memory tests。
- 看 LangChain / Pydantic Toolkit 如何包装 engine。
- 总结 Agent 看到的工具和 system prompt。

### 第 7 天：做一个小改动

任选一个低风险改动：

- 增加 CLI 输出提示。
- 给 context build 增加一条校验。
- 给 connector 增加一个单元测试。
- 给 docs 补一个例子。

目标是完整走一遍：读代码、改代码、跑测试、写说明。

## 7. 研究时容易卡住的点

### 7.1 名字和包边界容易混淆

`wrenai` 是 Python CLI/SDK 包，入口命令是 `wren`。`wren-core-py` 是 Rust core 的 Python binding，Python import 名是 `wren_core`。`wren-core-wasm` 是 npm 包。研究时要区分包名、目录名和 import 名。

### 7.2 SQL rewrite 跨 Python 和 Rust

SQL rewrite 不是单纯 Python 逻辑，也不是全在 Rust。Python 负责 CLI、manifest 抽取、sqlglot、CTE rewrite、connector 执行；Rust core 负责语义核心和 DataFusion 相关转换。追链路时要准备在两个语言之间来回跳。

### 7.3 MDL 有 JSON 和 YAML 两种形态

用户项目里主要是 YAML 文件，例如 `wren_project.yml`、`models/*/metadata.yml`。运行时核心需要的是 `target/mdl.json`。研究时要明确当前讨论的是源文件、构建产物，还是 base64 编码后的 manifest。

### 7.4 profile 和 secret 解析有优先级

连接信息可能来自：

- 命令行参数。
- `~/.wren/connection_info.json`。
- `~/.wren/profiles.yml`。
- 项目里的 `profile:` 字段。
- 项目 `.env` 中的 secret。

调试连接问题时，要先理清优先级。

### 7.5 测试成本不均匀

Python unit tests 通常比较快。Rust + maturin + WASM 构建可能较慢。研究初期优先跑最小测试或 dry-plan，等理解之后再跑全量。

## 8. 建议输出的个人笔记

研究过程中建议维护这些笔记：

```text
notes/
  01-architecture.md
  02-cli-flow.md
  03-mdl-build.md
  04-sql-rewrite.md
  05-connectors.md
  06-memory.md
  07-agent-sdk.md
  questions.md
```

每篇笔记建议固定格式：

```markdown
# 标题

## 我现在理解的结论

## 关键文件

## 调用链

## 还没理解的问题

## 可验证实验
```

这样一周后你不会只留下零散阅读痕迹，而是形成能指导修改代码的项目地图。

## 9. 最小核心文件清单

如果时间有限，优先研究这些：

- `README.md`
- `docs/core/README.md`
- `docs/core/concepts/architecture.md`
- `docs/core/reference/cli.md`
- `docs/core/reference/mdl.md`
- `core/wren/README.md`
- `core/wren/src/wren/cli.py`
- `core/wren/src/wren/engine.py`
- `core/wren/src/wren/context.py`
- `core/wren/src/wren/mdl/cte_rewriter.py`
- `core/wren/src/wren/connector/factory.py`
- `core/wren-core/README.md`
- `core/wren-core/core/src/lib.rs`
- `core/wren-core/core/src/mdl/mod.rs`
- `core/wren-core/core/src/logical_plan/mod.rs`
- `core/wren-core-py/README.md`
- `sdk/wren-langchain/README.md`
- `skills/README.md`

把这些读完并跑通一次 dry-plan，基本就能开始理解普通 issue 或做小改动了。
