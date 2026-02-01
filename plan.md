这是一个为您定制的 **TrendRadar 2.0 重构执行方案文档**。该文档严格遵循“架构师级”标准，将复杂的重构任务拆解为四个独立的原子化阶段 (Phases)。

您可以直接将每个阶段的描述复制给 AI 编程助手（如 Cursor, Windsurf, 或 Vibe Coding），它们即可按图索骥，精准执行。

---

# TrendRadar 2.0 重构执行方案 (Refactoring Execution Plan)

**项目目标**：将原本单次运行的脚本工具，重构为支持“守护进程调度”、“多源数据聚合”、“统一数据存储”及“Web 可视化”的企业级自动化情报系统。

---

## Phase 1: 统一存储与核心调度 (Unified Storage & Daemon Scheduling)

### 🎯 核心问题解决

* **碎片化存储**：废除按日期生成 `2025-12-21.db` 文件的模式，改为统一的 `trendradar.db` (SQLite) 或 PostgreSQL，为聚合分析和历史查询打下地基。
* **生命周期管理**：将 CLI 一次性脚本改造为支持 CRON 调度的常驻守护进程 (Daemon)。

### 📋 待办事项拆解 (TODOs)

#### 1.1 数据库模型重构 (`trendradar/storage/`)

* [ ] **修改 `schema.sql**`：
* 创建一个单一的 `schema.sql`，废除“按日期建表”的逻辑。
* 表结构设计：
* `raw_news` 表：增加 `created_at` (TIMESTAMP), `source_type` (RSS/Twitter/Web), `domain_tag` (Tech/Finance)。
* `trends` 表：用于存储 AI 聚合后的热点。




* [ ] **重写 `manager.py**`：
* 移除 `_get_db_path(date)` 逻辑，将数据库路径固定为 `config.yaml` 中配置的路径（如 `data/trendradar.db`）。
* 实现 `SQLAlchemy` 或优化原生 SQL 查询，支持“按时间范围” (`start_date`, `end_date`) 查询数据，而不再是只查当天。



#### 1.2 引入调度器 (`trendradar/core/`)

* [ ] **新增 `scheduler.py**`：
* 引入 `APScheduler` 库 (`BackgroundScheduler`)。
* 实现 `start_scheduler()` 函数，读取 `config.yaml` 中的 cron 表达式。
* 定义 `scheduled_job()`：封装原有的“抓取 -> 分析 -> 推送”流程。



#### 1.3 改造入口文件 (`trendradar/__main__.py`)

* [ ] **参数解析增强**：使用 `argparse` 添加 `--daemon` 参数。
* [ ] **模式分离**：
* 如果无参数：执行一次性逻辑（保持兼容）。
* 如果带 `--daemon`：初始化数据库 -> 启动 `APScheduler` -> 使用 `time.sleep` 或 `signal.pause()` 阻塞主线程，防止程序退出。



---

## Phase 2: 智能聚合引擎 (Intelligence Fusion Engine)

### 🎯 核心问题解决

* **信息冗余**：目前 10 个数据源可能报道同一个新闻，导致推送重复。
* **缺乏深度**：单一来源的信息往往片面，需要将多源信息融合为一个完整的“热点事件”。

### 📋 待办事项拆解 (TODOs)

#### 2.1 创建清洗与融合层 (`trendradar/core/fusion.py`)

* [ ] **实现 `FusionEngine` 类**：
* 输入：一批 `RawNews` 对象。
* 处理：使用 `difflib.SequenceMatcher` 或 `TF-IDF` 计算标题相似度。
* 输出：聚类后的 `TopicCluster` 列表（每个 Cluster 包含多条相关 News）。



#### 2.2 改造分析流程 (`trendradar/ai/analyzer.py`)

* [ ] **修改 Prompt 逻辑**：
* 不再是对单条新闻进行总结。
* 改为：将 `TopicCluster` 中的所有相关新闻标题和摘要合并，发送给 LLM。
* Prompt 指令升级：“请根据以下来自不同来源的关于同一事件的报道，生成一份综合情报摘要，并标注来源多样性。”



#### 2.3 数据流对接

* [ ] **修改 `__main__.py` 逻辑流**：
* 流程变更为：`Collector (多源并行)` -> `Storage (Raw Data)` -> **`Fusion Engine (聚合)`** -> `AI Analyzer` -> `Notification`。



---

## Phase 3: 感知扩展与配置驱动 (Domain Expansion)

### 🎯 核心问题解决

* **领域单一**：目前仅关注默认配置，缺乏对经济、科技、政治的分类感知。
* **配置硬编码**：数据源与领域标签没有解耦。

### 📋 待办事项拆解 (TODOs)

#### 3.1 增强配置结构 (`config/config.yaml`)

* [ ] **重构 YAML 结构**：
```yaml
sources:
  - name: "TechCrunch"
    url: "..."
    type: "rss"
    category: "technology"  <-- 新增字段
  - name: "Bloomberg"
    url: "..."
    type: "rss"
    category: "finance"

```


* [ ] **更新 `trendradar/core/config.py**`：确保 `Config` 类能正确解析新的嵌套结构，并提供 `get_sources_by_category(category)` 方法。

#### 3.2 抽象采集器 (`trendradar/crawler/`)

* [ ] **统一接口 `BaseFetcher**`：所有抓取器（RSS, Twitter, Web）必须在抓取结果中打上 `category` 标签。
* [ ] **入库逻辑更新**：确保 `category` 字段正确写入数据库 `raw_news` 表。

---

## Phase 4: 现代化 Web 看板 (Modern Visualization)

### 🎯 核心问题解决

* **展示简陋**：静态 HTML 无法筛选、排序，体验差。
* **数据孤岛**：无法查看历史数据。

### 📋 待办事项拆解 (TODOs)

#### 4.1 构建 API 后端 (`trendradar/web/api.py`)

* [ ] **引入 FastAPI**：在项目中集成 `FastAPI`。
* [ ] **开发核心接口**：
* `GET /api/trends`：支持参数 `date_range`, `category`, `sort_by (hot/time)`。
* `GET /api/trends/{id}`：获取热点详情及原始链接。



#### 4.2 开发前端 SPA (`trendradar/web/ui/`)

* [ ] **极简前端栈**：创建一个 `index.html`，使用 Vue 3 (CDN引入) + Element Plus。
* [ ] **实现数据表格**：
* 列显示：热度评分、标题、分类、来源数、时间。
* 功能：按热度排序，按分类筛选。


* [ ] **集成启动**：
* 修改 `__main__.py`，增加 `--web` 参数。
* 当运行时，启动 `uvicorn` 服务器，挂载 API 和静态资源。



---

### 🚀 执行建议 (Architect's Note)

建议您 **按顺序** 将这些 Phase 发送给 AI 编程助手。

* **Phase 1 是地基**，必须最先完成且测试通过（保证数据能存、能查、能自动跑）。
* **Phase 2 是大脑**，可以独立调试效果。
* **Phase 3 是四肢**，随时可以加新的源。
* **Phase 4 是面子**，依赖于 Phase 1 的统一数据库接口。
