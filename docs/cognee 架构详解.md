# Cognee 项目架构详解

> 文档版本：0.5.6  
> 整理日期：2026-04-02

---

## 一、项目概述

### 1.1 Cognee 是什么？

Cognee 是一个**AI 记忆平台**，它将原始数据转换为持久化的知识图谱，供 AI 代理使用。它不是传统的 RAG（检索增强生成），而是采用 **ECL（Extract, Cognify, Load）流水线**，结合了：
- 向量搜索（Vector Search）
- 图数据库（Graph Database）
- LLM 驱动的实体提取（LLM-powered Entity Extraction）

### 1.2 核心工作流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   add()     │ ──► │  cognify()  │ ──► │   search()  │
│ 数据 ingestion │     │ 构建知识图谱  │     │  查询知识   │
└─────────────┘     └─────────────┘     └─────────────┘
```

**简单比喻**：
- `add()` 像是把书放进图书馆
- `cognify()` 像是图书管理员读书、提取关键信息、建立索引卡片和交叉引用
- `search()` 像是读者来询问问题，图书管理员根据索引和知识来回答

---

## 二、项目目录结构

```
cognee/
├── cognee/                      # 核心 Python 包
│   ├── __init__.py              # 导出主要 API 函数
│   ├── api/v1/                  # REST API 接口层
│   │   ├── add/                 # 数据添加接口
│   │   ├── cognify/             # 知识图谱构建接口
│   │   ├── search/              # 搜索接口
│   │   ├── config/              # 配置接口
│   │   └── datasets/            # 数据集管理接口
│   │
│   ├── cli/                     # 命令行工具
│   │   └── commands/
│   │
│   ├── modules/                 # 业务模块（核心逻辑）
│   │   ├── pipelines/           # 流水线引擎
│   │   ├── data/                # 数据处理
│   │   ├── chunking/            # 文本分块
│   │   ├── graph/               # 图谱操作
│   │   ├── retrieval/           # 检索器
│   │   ├── search/              # 搜索类型
│   │   ├── storage/             # 存储抽象
│   │   ├── users/               # 用户与权限
│   │   ├── observability/       # 可观测性
│   │   └── ontologies/          # 本体论支持
│   │
│   ├── tasks/                   # 具体任务实现
│   │   ├── ingestion/           # 数据摄入任务
│   │   ├── documents/           # 文档处理任务
│   │   ├── graph/               # 图谱提取任务
│   │   ├── storage/             # 数据存储任务
│   │   ├── summarization/       # 摘要生成任务
│   │   └── completion/          # LLM 完成
│   │
│   ├── infrastructure/          # 基础设施层（数据库、LLM 适配）
│   │   ├── databases/
│   │   │   ├── graph/           # 图数据库适配器 (Kuzu, Neo4j)
│   │   │   ├── vector/          # 向量数据库适配器 (LanceDB, ChromaDB)
│   │   │   └── relational/      # 关系数据库适配器 (SQLite, PostgreSQL)
│   │   ├── llm/                 # LLM 网关（支持 OpenAI, Anthropic, Gemini 等）
│   │   ├── files/               # 文件加载器
│   │   └── engine/              # 引擎核心
│   │
│   ├── shared/                  # 共享工具和数据模型
│   │   ├── data_models.py       # 核心数据模型（KnowledgeGraph, Node, Edge）
│   │   └── logging_utils.py     # 日志工具
│   │
│   └── exceptions/              # 异常定义
│
├── cognee-frontend/             # React 前端（可选）
├── cognee-mcp/                  # MCP 服务器
├── examples/                    # 示例代码
├── tests/                       # 测试套件
├── docker-compose.yml           # Docker 部署配置
└── pyproject.toml               # Python 依赖配置
```

---

## 三、核心模块实现原理

### 3.1 API 层（api/v1/）

#### 3.1.1 add() - 数据添加

**文件**: `cognee/api/v1/add/add.py`

**作用**: 接收各种格式的数据（文本、文件、URL），存储到数据集中

**实现思路**:
```python
async def add(data, dataset_name="main_dataset", user=None, ...):
    # 1. 定义流水线任务
    tasks = [
        Task(resolve_data_directories),     # 解析目录
        Task(ingest_data, dataset_name)     # 摄入数据
    ]
    
    # 2. 设置数据库
    await setup()
    
    # 3. 验证用户和数据集权限
    user, authorized_dataset = await resolve_authorized_user_dataset(...)
    
    # 4. 运行流水线
    async for run_info in run_pipeline(tasks, data, user, ...):
        pipeline_run_info = run_info
    
    return pipeline_run_info
```

**处理流程**:
```
用户数据 → resolve_data_directories → ingest_data → save_data_item_to_storage → 创建 Dataset + Data 记录
```

**支持的数据类型**:
- 文本字符串：`"Your text content"`
- 文件路径：`"/path/to/file.pdf"` 或 `"file://relative/path.txt"`
- S3 路径：`"s3://bucket/file.pdf"`
- URL: `"https://example.com"`
- 二进制文件对象：`open("file.txt", "rb")`

**支持的文件格式**:
- 文本：`.txt`, `.md`, `.csv`
- PDF: `.pdf`
- 图片：`.png`, `.jpg`, `.jpeg` (OCR 提取)
- 音频：`.mp3`, `.wav` (转录)
- 代码：`.py`, `.js`, `.ts`
- Office: `.docx`, `.pptx`

---

#### 3.1.2 cognify() - 知识图谱构建

**文件**: `cognee/api/v1/cognify/cognify.py`

**作用**: 将原始文本转换为结构化的知识图谱

**核心处理流水线**:
```python
async def get_default_tasks():
    tasks = [
        Task(classify_documents),              # 1. 文档分类
        Task(extract_chunks_from_documents),   # 2. 文本分块
        Task(extract_graph_from_data),         # 3. 实体关系提取 (LLM)
        Task(summarize_text),                  # 4. 文本摘要
        Task(add_data_points),                 # 5. 存储到数据库
        Task(extract_dlt_fk_edges),            # 6. 提取 DLT 外键边
    ]
    return tasks
```

**详细步骤说明**:

1. **文档分类** (`classify_documents`)
   - 识别文档类型（文章、代码、表格等）
   - 确定后续处理方式

2. **文本分块** (`extract_chunks_from_documents`)
   - 使用 TextChunker 按段落/句子切分
   - 自动计算最优 chunk 大小（基于 LLM 和 embedding 模型限制）
   - 公式：`min(embedding_max_tokens, llm_max_tokens // 2)`

3. **图谱提取** (`extract_graph_from_data`)
   - **核心步骤**: 使用 LLM 从每个 chunk 中提取实体和关系
   - 使用 Instructor 进行结构化输出提取
   - 支持自定义 graph_model 和 ontology

4. **文本摘要** (`summarize_text`)
   - 为文档块生成层次化摘要
   - 支持快速浏览和导航

5. **数据存储** (`add_data_points`)
   - 将节点存储到图数据库
   - 创建向量索引用于语义搜索
   - 建立边关系

---

#### 3.1.3 search() - 知识查询

**文件**: `cognee/api/v1/search/search.py`

**支持的搜索类型**（SearchType 枚举）:

| 搜索类型 | 说明 | 适用场景 |
|---------|------|---------|
| `GRAPH_COMPLETION` (默认) | 基于图谱上下文的 LLM 回答 | 复杂问题、分析、洞察 |
| `RAG_COMPLETION` | 传统 RAG，基于文档块 | 直接文档检索 |
| `CHUNKS` | 原始文本块语义搜索 | 查找特定段落 |
| `SUMMARIES` | 预生成的摘要搜索 | 快速概览 |
| `CODE` | 代码专用搜索 | 查找函数、类 |
| `CYPHER` | 直接执行 Cypher 查询 | 高级用户调试 |
| `FEELING_LUCKY` | 智能选择最佳搜索类型 | 通用查询 |

**实现架构**:
```python
# 检索器模式（Retriever Pattern）
class GraphCompletionRetriever(BaseRetriever):
    async def get_completion(self, query):
        # 1. 获取相关三元组
        retrieved_objects = await self.get_retrieved_objects(query)
        
        # 2. 将图谱边转换为可读文本
        context = await self.get_context_from_objects(retrieved_objects)
        
        # 3. 使用 LLM 生成回答
        completion = await self.get_completion_from_context(query, context)
        
        return completion
```

**检索流程**:
```
用户查询 → 向量搜索三元组 → 解析为文本上下文 → LLM 生成回答
```

---

### 3.2 流水线模块（modules/pipelines/）

#### 3.2.1 Task 模型

**文件**: `cognee/modules/pipelines/models/Task.py`

```python
@dataclass
class Task:
    """任务定义"""
    func: Callable          # 任务函数
    task_config: dict       # 任务配置（如 batch_size）
    **kwargs                # 传递给函数的参数
```

#### 3.2.2 Pipeline 执行引擎

**文件**: `cognee/modules/pipelines/operations/run_tasks.py`

**核心逻辑**:
```python
async def run_pipeline(tasks, data, user, datasets, ...):
    """
    执行流水线任务
    支持：
    - 批量处理（batch processing）
    - 增量加载（incremental loading）
    - 并行执行（parallel execution）
    - 缓存（pipeline cache）
    """
    for task in tasks:
        # 1. 数据批处理
        batches = create_batches(data, batch_size=task.task_config.get("batch_size"))
        
        # 2. 执行任务
        for batch in batches:
            results = await task.func(batch, context=context)
            
            # 3. 管道传递结果到下一个任务
            data = results
```

**执行模式**:
- **阻塞模式**: 等待所有任务完成
- **后台模式**: 异步执行，返回 pipeline_run_id 用于追踪

---

### 3.3 图谱提取模块（tasks/graph/）

#### 3.3.1 实体关系提取

**文件**: `cognee/tasks/graph/extract_graph_from_data.py`

**实现思路**:
```python
async def extract_graph_from_data(data_chunks, graph_model, config, ...):
    # 1. 并行从每个 chunk 提取图谱
    chunk_graphs = await asyncio.gather(*[
        extract_content_graph(
            chunk.text, 
            graph_model, 
            custom_prompt=custom_prompt
        )
        for chunk in data_chunks
    ])
    
    # 2. 整合图谱（去重、本体验证）
    integrated_chunks = await integrate_chunk_graphs(
        data_chunks, 
        chunk_graphs, 
        graph_model, 
        ontology_resolver
    )
    
    return integrated_chunks
```

**关键函数 `extract_content_graph`**:
```python
# 文件：cognee/infrastructure/llm/extraction.py
async def extract_content_graph(text, response_model, custom_prompt=None):
    """使用 LLM 从文本提取知识图谱"""
    
    system_prompt = """
    从以下文本中提取实体和关系，构建知识图谱。
    
    实体（Node）:
    - id: 唯一标识符
    - name: 实体名称
    - type: 实体类型
    - description: 实体描述
    
    关系（Edge）:
    - source_node_id: 源实体 ID
    - target_node_id: 目标实体 ID
    - relationship_name: 关系名称
    """
    
    # 使用 LLMGateway 进行结构化输出
    graph = await LLMGateway.acreate_structured_output(
        text_input=text,
        system_prompt=system_prompt,
        response_model=KnowledgeGraph  # Pydantic 模型
    )
    
    return graph
```

**本体集成**（Ontology Integration）:
- 支持 OWL 本体文件
- 使用 `rdflib` 解析本体
- 实体与本体概念模糊匹配（80% 相似度）

---

### 3.4 数据存储模块（tasks/storage/）

#### 3.4.1 添加数据点

**文件**: `cognee/tasks/storage/add_data_points.py`

```python
async def add_data_points(data_points, context, embed_triplets=False):
    """
    将数据点存储到图数据库和向量数据库
    """
    # 1. 从数据点提取节点和边
    nodes, edges = await asyncio.gather(*[
        get_graph_from_model(dp) for dp in data_points
    ])
    
    # 2. 去重
    nodes, edges = deduplicate_nodes_and_edges(nodes, edges)
    
    # 3. 存储到图数据库
    await graph_engine.add_nodes(nodes)
    await graph_engine.add_edges(edges)
    
    # 4. 创建向量索引
    await index_data_points(nodes, vector_engine)
    await index_graph_edges(edges, vector_engine)
    
    # 5. 可选：创建三元组嵌入
    if embed_triplets:
        triplets = _create_triplets_from_graph(nodes, edges)
        await index_data_points(triplets, vector_engine)
    
    return data_points
```

**三元组嵌入**（Triplet Embeddings）:
- 将 `(subject, predicate, object)` 转换为可嵌入文本
- 格式：`"subject_text -> relationship_text -> object_text"`
- 用于增强语义搜索

---

### 3.5 检索模块（modules/retrieval/）

#### 3.5.1 基础检索器

**文件**: `cognee/modules/retrieval/base_retriever.py`

```python
class BaseRetriever:
    """所有检索器的基类"""
    
    async def get_retrieved_objects(self, query) -> Any:
        """获取相关对象"""
        pass
    
    async def get_context_from_objects(self, objects) -> str:
        """将对象转换为文本上下文"""
        pass
    
    async def get_completion_from_context(self, query, context) -> Any:
        """从上下文生成回答"""
        pass
    
    async def get_completion(self, query) -> Any:
        """完整检索流程"""
        objects = await self.get_retrieved_objects(query)
        context = await self.get_context_from_objects(objects)
        completion = await self.get_completion_from_context(query, context)
        return completion
```

#### 3.5.2 图谱完成检索器

**文件**: `cognee/modules/retrieval/graph_completion_retriever.py`

```python
class GraphCompletionRetriever(BaseRetriever):
    """默认的图谱完成检索器"""
    
    async def get_retrieved_objects(self, query):
        # 使用暴力三元组搜索
        return await brute_force_triplet_search(
            query,
            top_k=self.top_k,
            wide_search_top_k=self.wide_search_top_k,
            triplet_distance_penalty=self.triplet_distance_penalty,
        )
    
    async def get_context_from_objects(self, retrieved_edges):
        # 将边转换为可读文本
        return await resolve_edges_to_text(retrieved_edges)
    
    async def get_completion_from_context(self, query, context):
        # 使用 LLM 生成回答
        return await generate_completion(
            query=query,
            context=context,
            system_prompt_path=self.system_prompt_path,
            user_prompt_path=self.user_prompt_path,
        )
```

**三元组搜索算法**:
```python
# 文件：cognee/modules/retrieval/utils/brute_force_triplet_search.py
async def brute_force_triplet_search(query, top_k=10, ...):
    """
    在向量数据库中搜索相关三元组
    支持：
    - 多集合搜索（节点、边、三元组）
    - 距离惩罚（避免过远关联）
    - 反馈影响（用户反馈加权）
    """
    # 1. 获取所有向量集合
    collections = get_vector_index_collections()
    
    # 2. 查询每个集合
    results = await asyncio.gather(*[
        vector_engine.search(collection, query, top_k=wide_search_top_k)
        for collection in collections
    ])
    
    # 3. 合并和重新排序
    combined_results = rerank_results(results, distance_penalty)
    
    # 4. 返回 top_k
    return combined_results[:top_k]
```

---

### 3.6 基础设施层（infrastructure/）

#### 3.6.1 数据库适配器

**统一引擎**（Unified Engine）:
```python
# 文件：cognee/infrastructure/databases/unified/get_unified_engine.py
async def get_unified_engine():
    """
    获取统一的数据库引擎，封装：
    - graph: 图数据库操作
    - vector: 向量数据库操作
    - relational: 关系数据库操作
    """
    return UnifiedStoreEngine()
```

**图数据库接口**（GraphDBInterface）:
```python
# 文件：cognee/infrastructure/databases/graph/graph_db_interface.py
class GraphDBInterface(ABC):
    @abstractmethod
    async def add_nodes(self, nodes): pass
    
    @abstractmethod
    async def add_edges(self, edges): pass
    
    @abstractmethod
    async def query(self, cypher_query): pass
    
    @abstractmethod
    async def is_empty(self): pass
```

**支持的图数据库**:
- **Kuzu** (默认): 嵌入式图数据库，零配置
- **Neo4j**: 生产级图数据库
- **Neptune**: AWS 托管图数据库

**向量数据库接口**（VectorDBInterface）:
```python
# 文件：cognee/infrastructure/databases/vector/vector_db_interface.py
class VectorDBInterface(ABC):
    @abstractmethod
    async def create_collection(self, name, config): pass
    
    @abstractmethod
    async def upsert(self, collection, vectors, payloads): pass
    
    @abstractmethod
    async def search(self, collection, query, top_k): pass
```

**支持的向量数据库**:
- **LanceDB** (默认): 本地向量数据库
- **ChromaDB**: 轻量级向量数据库
- **PGVector**: PostgreSQL 向量扩展

---

#### 3.6.2 LLM 网关

**文件**: `cognee/infrastructure/llm/LLMGateway.py`

```python
class LLMGateway:
    """统一的 LLM 接口"""
    
    @staticmethod
    async def acreate_structured_output(text_input, system_prompt, response_model):
        """
        创建结构化输出
        
        支持两种框架:
        1. BAML: 高性能结构化输出
        2. Instructor (litellm): 通用的结构化输出
        """
        if STRUCTURED_OUTPUT_FRAMEWORK == "BAML":
            return await baml.acreate_structured_output(...)
        else:
            llm_client = get_llm_client()
            return await llm_client.acreate_structured_output(...)
```

**支持的 LLM 提供商**:
- OpenAI (默认)
- Anthropic Claude
- Google Gemini
- AWS Bedrock
- Ollama (本地)
- 自定义 OpenAI 兼容 API

**Instructor 模式**:
```python
# 不同的 LLM 使用不同的结构化输出模式
LLM_INSTRUCTOR_MODE = {
    "openai/gpt-4o": "json_schema_mode",      # GPT-4o 使用 JSON Schema
    "anthropic/claude-3": "tool_call",        # Claude 使用 Tool Call
    "gemini/gemini-pro": "md_json",           # Gemini 使用 Markdown JSON
}
```

---

### 3.7 数据模型（shared/data_models.py）

#### 3.7.1 知识图谱模型

```python
class Node(BaseModel):
    """图谱节点"""
    id: str                  # 唯一标识符
    name: str                # 节点名称
    type: str                # 节点类型
    description: str         # 节点描述

class Edge(BaseModel):
    """图谱边"""
    source_node_id: str      # 源节点 ID
    target_node_id: str      # 目标节点 ID
    relationship_name: str   # 关系名称

class KnowledgeGraph(BaseModel):
    """知识图谱容器"""
    nodes: List[Node] = []   # 节点列表
    edges: List[Edge] = []   # 边列表
```

#### 3.7.2 DataPoint 基类

```python
# 文件：cognee/infrastructure/engine/models/data_point.py
class DataPoint(BaseModel):
    """所有图节点的基类"""
    
    id: UUID = Field(default_factory=uuid4)
    metadata: dict = {
        "index_fields": [],  # 用于向量索引的字段
        "edge_fields": []    # 用于建立边的字段
    }
    
    # 版本控制
    version: str = "1.0"
    created_at: datetime = Field(default_factory=datetime.now)
```

---

## 四、架构设计模式

### 4.1 分层架构

```
┌─────────────────────────────────────────┐
│           API Layer (FastAPI)           │  ← HTTP 请求入口
├─────────────────────────────────────────┤
│         Main Functions (SDK)            │  ← add, cognify, search
├─────────────────────────────────────────┤
│       Pipeline Orchestrator             │  ← 任务编排和调度
├─────────────────────────────────────────┤
│        Task Execution Layer             │  ← 具体任务实现
├─────────────────────────────────────────┤
│       Domain Modules                    │  ← 业务领域逻辑
├─────────────────────────────────────────┤
│     Infrastructure Adapters             │  ← 数据库、LLM 适配
├─────────────────────────────────────────┤
│      External Services                  │  ← OpenAI, Kuzu, LanceDB
└─────────────────────────────────────────┘
```

### 4.2 接口适配模式

```python
# 统一的数据库接口，便于切换后端
class GraphDBInterface(ABC): ...
class KuzuAdapter(GraphDBInterface): ...
class Neo4jAdapter(GraphDBInterface): ...

# 使用工厂模式获取适配器
async def get_graph_engine():
    provider = get_graph_config().provider
    if provider == "kuzu":
        return KuzuAdapter()
    elif provider == "neo4j":
        return Neo4jAdapter()
```

### 4.3 检索器模式

```python
# 所有搜索类型都实现相同的接口
class BaseRetriever:
    async def get_completion(self, query) -> Any: ...

class GraphCompletionRetriever(BaseRetriever): ...
class ChunksRetriever(BaseRetriever): ...
class CodeRetriever(BaseRetriever): ...

# 根据 SearchType 动态选择检索器
retriever = get_search_type_retriever_instance(query_type)
result = await retriever.get_completion(query)
```

### 4.4 流水线模式

```python
# 任务可组合、可重用
tasks = [
    Task(classify_documents),
    Task(extract_chunks_from_documents),
    Task(extract_graph_from_data),
    Task(add_data_points),
]

# 支持并行、批量、增量执行
async for run_info in run_pipeline(
    tasks=tasks,
    data=data,
    incremental_loading=True,  # 增量加载
    use_pipeline_cache=True,   # 使用缓存
):
    print(f"Progress: {run_info}")
```

---

## 五、多租户与权限控制

### 5.1 用户 - 数据集层次结构

```
User (用户)
  └── Datasets (数据集)
        └── Data (数据项)
              └── Knowledge Graph (知识图谱)
```

### 5.2 权限模型

```python
class PermissionType(Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    SHARE = "share"

# 每个用户对数据集有独立的权限
class ACL(BaseModel):
    user_id: UUID
    dataset_id: UUID
    permissions: List[PermissionType]
```

### 5.3 数据库隔离

当启用 `ENABLE_BACKEND_ACCESS_CONTROL=True` 时：
- 每个用户 + 数据集组合可以有独立的图数据库
- 每个用户 + 数据集组合可以有独立的向量数据库
- 支持 Kuzu、LanceDB、SQLite、Postgres 的隔离模式

---

## 六、配置系统

### 6.1 环境变量层次结构

```python
# 优先级：运行时参数 > 环境变量 > 默认值

# 示例：LLM 配置
llm_config = get_llm_config()  # 从环境变量读取

# 可以在运行时覆盖
await cognify(
    graph_model=CustomGraphModel,  # 运行时参数
    chunk_size=512,                # 运行时参数
)
```

### 6.2 核心配置项

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `LLM_API_KEY` | - | LLM API 密钥 |
| `LLM_MODEL` | `openai/gpt-4o-mini` | LLM 模型 |
| `LLM_PROVIDER` | `openai` | LLM 提供商 |
| `EMBEDDING_PROVIDER` | `openai` | 嵌入提供商 |
| `VECTOR_DB_PROVIDER` | `lancedb` | 向量数据库 |
| `GRAPH_DATABASE_PROVIDER` | `kuzu` | 图数据库 |
| `DB_PROVIDER` | `sqlite` | 关系数据库 |
| `ENABLE_BACKEND_ACCESS_CONTROL` | `true` | 多租户隔离 |

---

## 七、扩展点

### 7.1 添加新任务类型

```python
# 1. 创建任务函数
from cognee.modules.pipelines.tasks.Task import Task

async def my_custom_task(data, context):
    """自定义任务"""
    processed = process(data)
    return processed

# 2. 注册到流水线
tasks = [
    Task(classify_documents),
    Task(my_custom_task),  # 添加自定义任务
    Task(extract_graph_from_data),
]
```

### 7.2 添加新的数据库后端

```python
# 1. 实现图数据库接口
class MyGraphAdapter(GraphDBInterface):
    async def add_nodes(self, nodes):
        # 实现节点添加逻辑
        pass
    
    async def add_edges(self, edges):
        # 实现边添加逻辑
        pass

# 2. 注册到工厂
# 修改 get_graph_engine() 函数
```

### 7.3 添加新的搜索类型

```python
# 1. 添加到 SearchType 枚举
class SearchType(Enum):
    MY_CUSTOM_SEARCH = "my_custom_search"

# 2. 创建检索器
class MyCustomRetriever(BaseRetriever):
    async def get_completion(self, query):
        # 实现自定义检索逻辑
        pass

# 3. 注册检索器映射
register_retriever(SearchType.MY_CUSTOM_SEARCH, MyCustomRetriever)
```

### 7.4 自定义图谱模型

```python
from cognee.infrastructure.engine import DataPoint

# 定义领域特定的节点类型
class ScientificPaper(DataPoint):
    title: str
    authors: List[str]
    abstract: str
    publication_date: datetime
    
    class Metadata:
        index_fields = ["title", "abstract"]

# 使用自定义模型
await cognify(
    graph_model=ScientificPaper,
    datasets=["research_papers"]
)
```

---

## 八、错误处理与日志

### 8.1 日志系统

使用 `structlog` 结构化日志：

```python
from cognee.shared.logging_utils import get_logger

logger = get_logger("my_module")

logger.info("Processing started", data_count=len(data))
logger.warning("Empty graph detected", dataset_id=dataset.id)
logger.error("LLM connection failed", exception=str(error))
```

### 8.2 异常层次结构

```python
# 基础异常
CogneeException
├── CogneeValidationError
├── DatabaseNotCreatedError
├── DatasetNotFoundError
├── UserNotFoundError
├── InvalidGraphModelError
├── InvalidDataChunksError
└── ...
```

---

## 九、性能优化

### 9.1 批量处理

```python
# 所有任务都支持批量处理
Task(extract_graph_from_data, task_config={"batch_size": 100})
```

### 9.2 并行执行

```python
# 使用 asyncio.gather 并行处理
results = await asyncio.gather(*[
    process_chunk(chunk) for chunk in chunks
])
```

### 9.3 缓存

```python
# 流水线缓存
await run_pipeline(
    tasks=tasks,
    use_pipeline_cache=True,  # 启用缓存
    pipeline_name="cognify_pipeline"
)

# 会话缓存（Q&A 历史）
sm = get_session_manager()
completion = await sm.generate_completion_with_session(
    session_id=session_id,
    query=query,
    context=context,
)
```

### 9.4 增量加载

```python
# 只处理新增或修改的数据
await cognify(
    datasets=["my_dataset"],
    incremental_loading=True  # 默认启用
)
```

---

## 十、调试技巧

### 10.1 启用调试日志

```bash
# 设置环境变量
export LITELLM_LOG="DEBUG"
export ENV="debug"
export LOG_LEVEL="DEBUG"
```

### 10.2 跳过 LLM 连接测试

```bash
# 遇到 LLM 连接超时时
export COGNEE_SKIP_CONNECTION_TEST=true
```

### 10.3 禁用多租户（访问旧数据）

```bash
# 如果需要访问多租户启用前的数据
export ENABLE_BACKEND_ACCESS_CONTROL=false
```

### 10.4 检查数据库状态

```python
from cognee.infrastructure.databases.unified import get_unified_engine

engine = await get_unified_engine()
is_empty = await engine.graph.is_empty()
print(f"Graph empty: {is_empty}")
```

---

## 十一、常见问题与解决方案

### Q1: LLM 连接超时

**症状**: `LLM connection test timed out after 30s`

**解决方案**:
1. 检查 API 密钥是否正确
2. 检查端点 URL 是否可访问
3. 设置 `COGNEE_SKIP_CONNECTION_TEST=true` 跳过测试

### Q2: 图谱为空

**症状**: `Search attempt on an empty knowledge graph`

**解决方案**:
1. 确认已运行 `cognify()`
2. 检查 LLM_API_KEY 是否配置
3. 查看日志中的 cognify 错误信息

### Q3: 权限错误

**症状**: 搜索返回空列表

**解决方案**:
1. 检查用户是否有数据集读权限
2. 确认 `ENABLE_BACKEND_ACCESS_CONTROL` 设置
3. 使用 `datasets()` API 查看可访问的数据集

### Q4: 嵌入维度不匹配

**症状**: 向量搜索失败

**解决方案**:
1. 确保嵌入模型维度与配置一致
2. 检查 `EMBEDDING_DIMENSIONS` 设置
3. 重建向量索引

---

## 十二、总结

Cognee 的核心设计理念：

1. **ECL 流水线**: Extract → Cognify → Load 的清晰数据流
2. **适配器模式**: 统一的数据库和 LLM 接口，支持多种后端
3. **检索器模式**: 不同搜索类型共享相同的接口
4. **多租户支持**: 用户级别的数据隔离和权限控制
5. **可扩展性**: 易于添加新任务、新数据库、新搜索类型

通过学习这个架构，你可以：
- 理解现代 AI 应用的分层设计
- 学习如何构建可扩展的 LLM 应用
- 掌握知识图谱和向量搜索的集成模式
