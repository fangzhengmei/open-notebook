# Open-Notebook 混合检索方案分析报告

## 1. 概述

Open-Notebook 使用 SurrealDB 实现了一个结合向量检索、全文检索与图关系数据建模的混合检索方案。本文档详细追踪一次完整搜索请求的实现链路，分析三种检索方式的触发机制、结果合并策略，以及向量嵌入数据的写入时机与一致性保障。

---

## 2. 整体架构

### 2.1 服务端搜索链路（核心）

服务端搜索请求不经过 `api/search_service.py`，路由层直接调用业务域层：

```
┌─────────────────────────────────────────────────────────────────┐
│                   API Router Layer                                │
│  api/routers/search.py (FastAPI 端点)                            │
│  - POST /search                                                   │
│  - POST /search/ask                                               │
│  - POST /search/ask/simple                                        │
├─────────────────────────────────────────────────────────────────┤
│                      Domain Layer                                 │
│  open_notebook/domain/notebook.py (核心业务逻辑)                  │
│  - text_search() 函数: 调用 fn::text_search()                    │
│  - vector_search() 函数: 生成查询向量 + 调用 fn::vector_search() │
├─────────────────────────────────────────────────────────────────┤
│                      Database Layer                               │
│  open_notebook/database/repository.py (数据库操作)                │
│  - repo_query(): 执行 SurrealQL 查询                              │
│  - db_connection(): 数据库连接管理                                │
│                                                         │
│  open_notebook/database/migrations/*.surrealql (SurrealQL 函数) │
│  - fn::text_search(): 全文检索 + 合并去重排序                     │
│  - fn::vector_search(): 向量检索 + 合并去重排序                   │
├─────────────────────────────────────────────────────────────────┤
│                      Embedding Layer                              │
│  commands/embedding_commands.py (异步嵌入命令)                    │
│  - embed_source_command: Source 分块嵌入                          │
│  - embed_note_command: Note 嵌入                                  │
│  - embed_insight_command: Insight 嵌入                            │
│                                                         │
│  open_notebook/utils/embedding.py (嵌入生成逻辑)                  │
│  - generate_embedding(): 单个文本嵌入（自动分块+平均池化）         │
│  - generate_embeddings(): 批量嵌入                                │
│  - mean_pool_embeddings(): 平均池化                               │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 客户端组件（不参与服务端搜索链路）

以下组件是**客户端封装**，用于外部调用后端 API，**不参与服务端的搜索执行链路**：

| 组件 | 路径 | 定位 | 使用者 |
|-----|------|------|--------|
| `APIClient` | `api/client.py` | HTTP 客户端封装，基于 `httpx` | 命令行工具、外部脚本 |
| `SearchService` | `api/search_service.py` | 对 `APIClient` 的搜索相关封装 | 命令行工具、外部脚本 |
| `searchApi` | `frontend/src/lib/api/search.ts` | 前端 API 调用封装 | Next.js 前端应用 |

**客户端调用链路示例**：
```
前端/命令行工具
    ↓
api/search_service.py → api/client.py (httpx)
    ↓
HTTP POST /api/search
    ↓
进入服务端搜索链路（见上图）
```

**关键修正说明**：
- 原报告误将 `api/search_service.py` 标注为服务层组件，但实际上它是**客户端封装**
- 服务端的真实入口是 `api/routers/search.py`，直接调用 `open_notebook/domain/notebook.py`
- 前端通过 `frontend/src/lib/api/search.ts` 直接调用 `/api/search` 端点，不经过 `api/search_service.py`

---

## 3. 三种检索方式的实现链路

### 3.1 API 入口

**文件**: `api/routers/search.py:17-59`

搜索请求通过 `/search` POST 端点进入，根据 `search_request.type` 字段决定检索类型：

```python
@router.post("/search", response_model=SearchResponse)
async def search_knowledge_base(search_request: SearchRequest):
    if search_request.type == "vector":
        # 向量检索
        results = await vector_search(
            keyword=search_request.query,
            results=search_request.limit,
            source=search_request.search_sources,
            note=search_request.search_notes,
            minimum_score=search_request.minimum_score,
        )
    else:
        # 全文检索 (默认)
        results = await text_search(
            keyword=search_request.query,
            results=search_request.limit,
            source=search_request.search_sources,
            note=search_request.search_notes,
        )
```

**SearchRequest 模型** (`api/models.py:32-41`):
- `query`: 搜索关键词
- `type`: `"text"` 或 `"vector"`，默认 `"text"`
- `limit`: 结果数量限制，最大 1000
- `search_sources`: 是否搜索 sources
- `search_notes`: 是否搜索 notes
- `minimum_score`: 向量检索的最小相似度阈值 (0.0-1.0)

---

### 3.2 全文检索 (Text Search) 链路

#### 3.2.1 触发条件
当 `search_request.type == "text"` 或未指定类型时触发。

#### 3.2.2 调用链路

```
POST /search
    ↓
api/routers/search.py: search_knowledge_base()
    ↓ (type != "vector")
open_notebook/domain/notebook.py: text_search()
    ↓
open_notebook/database/repository.py: repo_query()
    ↓
SurrealDB: fn::text_search()
    ↓
返回合并去重后的结果
```

#### 3.2.3 核心实现

**业务层函数**: `open_notebook/domain/notebook.py:630-647`

```python
async def text_search(
    keyword: str, results: int, source: bool = True, note: bool = True
):
    search_results = await repo_query(
        """
        select *
        from fn::text_search($keyword, $results, $source, $note)
        """,
        {"keyword": keyword, "results": results, "source": source, "note": note},
    )
    return search_results
```

**数据库层函数**: `open_notebook/database/migrations/9.surrealql:2-70` (最新版本)

全文检索使用 SurrealDB 的 **BM25 全文索引**，搜索范围覆盖：

| 搜索目标 | 表名 | 字段 | 索引名称 |
|---------|------|------|---------|
| Source 标题 | `source` | `title` | `idx_source_title` |
| Source 全文 | `source` | `full_text` | `idx_source_full_text` |
| Source 分块内容 | `source_embedding` | `content` | `idx_source_embed_chunk` |
| Source 洞察 | `source_insight` | `content` | `idx_source_insight` |
| Note 标题 | `note` | `title` | `idx_note_title` |
| Note 内容 | `note` | `content` | `idx_note` |

**索引定义** (`migrations/1.surrealql:67-72`):
```surrealql
DEFINE INDEX IF NOT EXISTS idx_source_title ON TABLE source COLUMNS title SEARCH ANALYZER my_analyzer BM25 HIGHLIGHTS;
DEFINE INDEX IF NOT EXISTS idx_source_full_text ON TABLE source COLUMNS full_text SEARCH ANALYZER my_analyzer BM25 HIGHLIGHTS;
-- ... 其他索引类似
```

**全文检索函数逻辑** (`migrations/9.surrealql:5-70`):

```surrealql
DEFINE FUNCTION IF NOT EXISTS fn::text_search($query_text: string, $match_count: int, $sources:bool, $show_notes:bool) {
    -- 1. Source 标题搜索
    let $source_title_search = 
        IF $sources {(
            SELECT id, title, 
            search::highlight('`', '`', 1) as content,
            id as parent_id,
            math::max(search::score(1)) AS relevance
            FROM source
            WHERE title @1@ $query_text
            GROUP BY id)}
        ELSE { [] };
    
    -- 2. Source 分块内容搜索
    let $source_embedding_search = 
         IF $sources {(
            SELECT source.id as id, source.title as title, 
            search::highlight('`', '`', 1) as content, 
            source.id as parent_id, 
            math::max(search::score(1)) AS relevance
            FROM source_embedding
            WHERE content @1@ $query_text
            GROUP BY id)}
        ELSE { [] };

    -- 3. Source 全文搜索
    let $source_full_search = ...
    
    -- 4. Source 洞察搜索
    let $source_insight_search = ...
    
    -- 5. Note 标题搜索
    let $note_title_search = ...
    
    -- 6. Note 内容搜索
    let $note_content_search = ...
    
    -- 合并结果
    let $source_chunk_results = array::union($source_embedding_search, $source_full_search);
    let $source_asset_results = array::union($source_title_search, $source_insight_search);
    let $source_results = array::union($source_chunk_results, $source_asset_results);
    let $note_results = array::union($note_title_search, $note_content_search);
    let $final_results = array::union($source_results, $note_results);

    -- 去重排序
    RETURN (select id, parent_id, title, math::max(relevance) as relevance
    from $final_results where id is not None
    group by id, parent_id, title ORDER BY relevance DESC LIMIT $match_count);
};
```

**关键要点**:
- 使用 `@1@` 操作符进行全文搜索（单字精度）
- 使用 `search::score(1)` 获取 BM25 相关性分数
- 使用 `search::highlight()` 对匹配内容进行高亮
- 多层 `array::union` 合并不同搜索目标的结果

---

### 3.3 向量检索 (Vector Search) 链路

#### 3.3.1 触发条件
当 `search_request.type == "vector"` 时触发，需要配置 embedding model。

#### 3.3.2 调用链路

```
POST /search (type="vector")
    ↓
api/routers/search.py: search_knowledge_base()
    ↓ (检查 embedding model 是否可用)
open_notebook/domain/notebook.py: vector_search()
    ├─→ open_notebook/utils/embedding.py: generate_embedding()
    │       └─→ 调用 embedding model 生成查询向量
    └─→ open_notebook/database/repository.py: repo_query()
            ↓
SurrealDB: fn::vector_search()
    ↓
返回余弦相似度匹配结果
```

#### 3.3.3 核心实现

**前置检查**: `api/routers/search.py:21-27`

```python
if search_request.type == "vector":
    # 检查 embedding model 是否可用
    if not await model_manager.get_embedding_model():
        raise HTTPException(
            status_code=400,
            detail="Vector search requires an embedding model...",
        )
```

**业务层函数**: `open_notebook/domain/notebook.py:650-680`

```python
async def vector_search(
    keyword: str,
    results: int,
    source: bool = True,
    note: bool = True,
    minimum_score=0.2,
):
    # 1. 生成查询向量
    from open_notebook.utils.embedding import generate_embedding
    embed = await generate_embedding(keyword)
    
    # 2. 调用数据库向量搜索函数
    search_results = await repo_query(
        """
        SELECT * FROM fn::vector_search($embed, $results, $source, $note, $minimum_score);
        """,
        {
            "embed": embed,
            "results": results,
            "source": source,
            "note": note,
            "minimum_score": minimum_score,
        },
    )
    return search_results
```

**向量生成函数**: `open_notebook/utils/embedding.py:209-274`

```python
async def generate_embedding(
    text: str,
    content_type: Optional[ContentType] = None,
    file_path: Optional[str] = None,
    command_id: Optional[str] = None,
) -> List[float]:
    """
    生成单个文本的嵌入向量，处理大文本通过分块和平均池化。
    
    对于短文本 (<= CHUNK_SIZE tokens):
        - 直接嵌入并返回向量
    
    对于长文本 (> CHUNK_SIZE tokens):
        - 使用合适的分割器分块
        - 批量嵌入所有块
        - 通过平均池化合并嵌入
    """
    text_tokens = token_count(text)
    
    if text_tokens <= CHUNK_SIZE:
        # 短文本 - 直接嵌入
        embeddings = await generate_embeddings([text], command_id=command_id)
        return embeddings[0]
    
    # 长文本 - 分块 + 平均池化
    chunks = chunk_text(text, content_type=content_type, file_path=file_path)
    embeddings = await generate_embeddings(chunks, command_id=command_id)
    pooled = await mean_pool_embeddings(embeddings)
    return pooled
```

**平均池化算法**: `open_notebook/utils/embedding.py:55-108`

```python
async def mean_pool_embeddings(embeddings: List[List[float]]) -> List[float]:
    """
    使用平均池化将多个嵌入向量合并为一个。
    
    算法:
    1. 将每个嵌入归一化为单位长度
    2. 计算逐元素均值
    3. 将结果归一化为单位长度
    """
    if len(embeddings) == 1:
        return embeddings[0]
    
    arr = np.array(embeddings, dtype=np.float64)
    
    # 归一化每个嵌入到单位长度
    norms = np.linalg.norm(arr, axis=1, keepdims=True)
    norms = np.where(norms > 0, norms, 1.0)
    normalized = arr / norms
    
    # 计算均值
    mean = np.mean(normalized, axis=0)
    
    # 归一化结果
    mean_norm = np.linalg.norm(mean)
    if mean_norm > 0:
        mean = mean / mean_norm
    
    return mean.tolist()
```

**数据库层函数**: `open_notebook/database/migrations/9.surrealql:2-66`

```surrealql
DEFINE FUNCTION IF NOT EXISTS fn::vector_search($query: array<float>, $match_count: int, $sources: bool, $show_notes: bool, $min_similarity: float) {
    -- 1. Source 分块向量搜索
    let $source_embedding_search = 
        IF $sources {(
            SELECT 
                source.id as id,
                source.title as title,
                content,
                source.id as parent_id,
                vector::similarity::cosine(embedding, $query) as similarity
            FROM source_embedding 
            WHERE embedding != none 
              AND array::len(embedding)=array::len($query) 
              AND vector::similarity::cosine(embedding, $query) >= $min_similarity
            ORDER BY similarity DESC
            LIMIT $match_count
        )}
        ELSE { [] };

    -- 2. Source 洞察向量搜索
    let $source_insight_search = 
        IF $sources {(
            SELECT 
                id,
                insight_type + ' - ' + (source.title OR '') as title,
                content,
                source.id as parent_id,
                vector::similarity::cosine(embedding, $query) as similarity
            FROM source_insight
            WHERE embedding != none 
              AND array::len(embedding)=array::len($query) 
              AND vector::similarity::cosine(embedding, $query) >= $min_similarity
            ORDER BY similarity DESC
            LIMIT $match_count
        )}
        ELSE { [] };

    -- 3. Note 向量搜索
    let $note_content_search = 
        IF $show_notes {(
            SELECT 
                id,
                title,
                content,
                id as parent_id,
                vector::similarity::cosine(embedding, $query) as similarity
            FROM note
            WHERE embedding != none 
              AND array::len(embedding)=array::len($query) 
              AND vector::similarity::cosine(embedding, $query) >= $min_similarity
            ORDER BY similarity DESC
            LIMIT $match_count
        )}
        ELSE { [] };

    -- 合并结果
    let $all_results = array::union(
        array::union($source_embedding_search, $source_insight_search),
        $note_content_search
    );

    -- 去重排序
    RETURN (select id, parent_id, title, math::max(similarity) as similarity,
    array::flatten(content) as matches
    from $all_results where id is not None
    group by id, parent_id, title ORDER BY similarity DESC LIMIT $match_count);
};
```

**关键要点**:
- 使用 `vector::similarity::cosine()` 计算余弦相似度
- 增加维度检查 `array::len(embedding)=array::len($query)` 确保兼容性
- 支持 `minimum_score` 阈值过滤低相似度结果
- 向量搜索覆盖: `source_embedding`、`source_insight`、`note` 表

---

### 3.4 图关系查询 (Graph Relation Query)

#### 3.4.1 数据模型中的图关系

Open-Notebook 使用 SurrealDB 的 **RELATE** 机制建立实体间的图关系，但当前搜索实现**并未直接在搜索时使用图遍历扩展结果**。

**定义的关系表**:

| 关系名 | 源表 | 目标表 | 用途 |
|-------|------|--------|------|
| `reference` | `source` | `notebook` | Source 归属到 Notebook |
| `artifact` | `note` | `notebook` | Note 归属到 Notebook |
| `refers_to` | `chat_session` | `notebook` | 聊天会话引用 Notebook |

**关系定义** (`migrations/1.surrealql:54-60`):
```surrealql
DEFINE TABLE IF NOT EXISTS reference
TYPE RELATION 
FROM source TO notebook;

DEFINE TABLE IF NOT EXISTS artifact
TYPE RELATION 
FROM note TO notebook;
```

**关系使用示例**: `open_notebook/domain/base.py:217-229`

```python
async def relate(
    self, relationship: str, target_id: str, data: Optional[Dict] = {}
) -> Any:
    return await repo_relate(
        source=self.id, relationship=relationship, target=target_id, data=data
    )
```

**底层实现**: `open_notebook/database/repository.py:106-120`

```python
async def repo_relate(
    source: str, relationship: str, target: str, data: Optional[Dict[str, Any]] = None
) -> List[Dict[str, Any]]:
    """创建两个记录之间的关系，可选数据"""
    query = f"RELATE {source}->{relationship}->{target} CONTENT $data;"
    return await repo_query(query, {"data": data})
```

#### 3.4.2 当前搜索中的图关系使用

**重要发现**: 当前版本的 `fn::text_search` 和 `fn::vector_search` **不包含图遍历逻辑**。搜索结果仅来自：
- 直接表扫描（向量搜索）
- 全文索引（文本搜索）

**图关系的实际用途**:
1. **数据组织**: 通过 Notebook 组织 Source 和 Note
2. **级联操作**: 删除 Notebook 时级联处理关联数据
3. **上下文构建**: 构建 AI 上下文时获取关联的 Source/Note

**级联删除示例**: `open_notebook/domain/notebook.py:138-230`

```python
async def delete(self, delete_exclusive_sources: bool = False) -> Dict[str, int]:
    # 1. 删除关联的所有 notes
    notes = await self.get_notes()
    for note in notes:
        await note.delete()
    
    # 2. 处理 sources (根据是否独占决定删除或取消关联)
    if delete_exclusive_sources:
        # 查询仅属于此 notebook 的 sources
        source_counts = await repo_query("""
            SELECT id, count(->reference[WHERE out != $notebook_id].out) as assigned_others
            FROM (SELECT VALUE <-reference.in AS sources FROM $notebook_id)[0]
        """, {"notebook_id": notebook_id})
    
    # 3. 删除关系记录
    await repo_query("DELETE reference WHERE out = $notebook_id", ...)
    await repo_query("DELETE artifact WHERE out = $notebook_id", ...)
```

#### 3.4.3 潜在的图扩展搜索

虽然当前实现未使用，但 SurrealDB 支持图遍历语法：

```surrealql
-- 示例: 查找与某个 Source 关联的其他 Source (通过共享 Notebook)
SELECT ->reference->notebook<-reference<-source FROM source:xxx
```

#### 3.4.4 设计权衡：为什么当前方案不做图遍历扩展

当前搜索实现选择**不使用图遍历扩展搜索结果**，这是一个经过多维度权衡的设计决策。以下从三个核心维度分析：

##### 3.4.4.1 查询性能考量

| 考量点 | 具体分析 |
|-------|---------|
| **多跳查询复杂度** | 图遍历（如 `->reference->notebook<-reference<-source`）需要至少 3 跳：Source → 关系表 → Notebook → 关系表 → 其他 Source。每一跳都涉及一次表查询或索引查找。 |
| **结果爆炸风险** | 一个 Notebook 可能关联数十个 Source，每个 Source 又可能关联多个 Notebook。2 跳查询可能导致结果数量呈 O(n²) 级增长。例如：10 个匹配的 Source，每个关联 3 个 Notebook，每个 Notebook 再关联 10 个 Source → 可能产生 300+ 个扩展结果。 |
| **延迟不可预测** | 直接搜索（全文/向量）的延迟相对稳定，因为主要依赖索引查找。而图遍历的延迟取决于：<br>1. 初始结果集大小<br>2. 每个节点的出边/入边数量<br>3. 关系表的索引效率<br>在数据量大时，图遍历可能成为性能瓶颈。 |
| **资源消耗** | 图遍历需要在内存中构建和处理中间结果集。高并发场景下，多个图遍历查询可能导致：<br>- CPU 使用率飙升（处理大量关系边）<br>- 内存压力增大（存储中间结果）<br>- 数据库连接池耗尽（长查询持有连接） |

**SurrealDB 图遍历的底层实现**：

SurrealDB 的图遍历语法（`->`/`<-` 操作符）底层实际上是通过关系表查询实现的，而非专门的图数据库优化。例如：

```surrealql
-- 语法糖写法
SELECT ->reference->notebook FROM source:xxx

-- 等价的展开写法
SELECT (SELECT out FROM reference WHERE in = source:xxx).out AS notebooks
FROM source:xxx
```

这意味着每一跳都涉及一次 `SELECT ... FROM reference WHERE in/out = ...` 查询。如果关系表没有合适的索引，性能会很差。

##### 3.4.4.2 结果可预测性与用户体验

| 问题维度 | 具体分析 |
|---------|---------|
| **相关性衰减** | 直接匹配的文档（通过关键词或向量相似度）相关性是明确的。但通过 2 跳、3 跳图路径找到的文档，相关性会急剧衰减。<br><br>**示例**：<br>- 用户搜索"机器学习"<br>- 直接找到 Source A（标题包含"机器学习"）<br>- Source A 属于 Notebook X<br>- Notebook X 中还有 Source B（关于"深度学习"）<br>- Source B 属于 Notebook X，但也属于 Notebook Y<br>- Notebook Y 中还有 Source C（关于"Python 编程"）<br><br>问题：Source C 与"机器学习"的相关性有多高？应该排在什么位置？ |
| **分数融合困难** | 全文检索的 `relevance`（BM25 分数）和向量检索的 `similarity`（余弦相似度）都是成熟的、可解释的分数。<br><br>但图扩展的结果如何评分？<br>- 方案 A：使用原始搜索中关联节点的分数 × 衰减系数<br>- 方案 B：对扩展结果重新执行一次搜索验证<br>- 方案 C：使用图路径长度作为评分依据<br><br>每种方案都有缺陷：方案 A 可能引入大量低相关结果；方案 B 增加额外查询开销；方案 C 完全忽略语义相关性。 |
| **去重策略复杂** | 同一文档可能通过多条不同的图路径被找到。例如：<br>- Source A → Notebook X → Source D<br>- Source B → Notebook X → Source D<br><br>问题：当 Source D 通过两条路径被找到时，应该：<br>1. 保留第一次出现的分数？<br>2. 取两条路径中的最高分数？<br>3. 加权求和？<br>4. 考虑路径多样性（通过不同 Notebook 找到的应该加分？） |
| **用户预期管理** | 用户搜索时通常有明确的信息需求。如果搜索结果中混入大量通过图关系间接关联的文档，用户可能会困惑：<br>- "为什么这篇文档会出现在这里？"<br>- "这和我搜索的关键词有什么关系？"<br><br>缺乏可解释性会降低用户对搜索系统的信任。 |

##### 3.4.4.3 语义可信度与数据模型限制

当前的图关系模型存在**语义局限性**，这是不做图扩展搜索的更深层原因：

| 维度 | 分析 |
|-----|------|
| **关系语义单一** | 当前定义的关系只有：<br>- `reference`: source → notebook（"归属"）<br>- `artifact`: note → notebook（"归属"）<br><br>这些关系只表示"组织归属"，**不表示语义相似或主题相关**。 |
| **组织 vs 语义的混淆** | 两个 Source 共享同一个 Notebook，可能只是因为：<br>1. 用户手动组织（"我把这两个文档放在同一个文件夹"）<br>2. 上传时的默认分组<br>3. 批量导入时的关联<br><br>**这并不意味着这两个 Source 在语义上相关**。<br><br>**反例**：<br>- Notebook "我的学习资料" 中可能同时包含：<br>  - Source A: "Python 机器学习入门"<br>  - Source B: "如何烤蛋糕"<br><br>如果用户搜索"机器学习"，通过图扩展会找到 Source B，这显然是噪音。 |
| **缺乏细粒度关系类型** | 真正有效的图检索需要更丰富的关系语义，例如：<br>- `cites`/`cited_by`: 引用关系（学术文献）<br>- `related_to`: 显式标记的相关关系<br>- `contradicts`: 对立/相反观点<br>- `extends`: 扩展/续作<br><br>当前模型只有"归属"关系，缺乏这些语义信息。 |
| **对比：向量检索的优势** | 向量检索天然擅长发现**隐式语义关联**。即使两个文档没有显式关系，只要它们的内容语义相似，向量相似度就会高。<br><br>这比基于显式图关系的扩展更可靠，因为：<br>1. 不依赖用户手动标注关系<br>2. 基于实际内容而非组织方式<br>3. 可以发现跨 Notebook 的语义关联 |

##### 3.4.4.4 现有方案的优势总结

当前"直接搜索 + 合并去重"方案的优势：

| 优势 | 说明 |
|-----|------|
| **性能可预测** | 单表查询 + 索引查找，延迟稳定可控。即使数据量增长，性能下降也是线性的。 |
| **实现简单** | 无需处理图遍历的边界条件：<br>- 循环路径检测<br>- 无限递归防护<br>- 路径数量爆炸 |
| **调试方便** | 可以直接在 SurrealDB 控制台执行 `fn::text_search()` 或 `fn::vector_search()` 验证搜索逻辑，返回的 SQL 结果易于理解。 |
| **结果可解释** | 用户可以理解：<br>- "这篇文档被返回是因为标题包含我的搜索词"<br>- "这篇文档被返回是因为内容与我的查询语义相似" |
| **分数直观** | BM25 和余弦相似度都是成熟的、可解释的分数，用户可以通过调整阈值来控制结果质量。 |

---

#### 3.4.5 扩展方案：如何将图关系纳入混合检索

如果未来确实需要将图关系纳入搜索（例如：添加了更丰富的关系语义、用户明确需要"查找相关文档"功能），可以在现有分层架构上进行扩展。以下分析两种可行方案及其优劣对比。

##### 3.4.5.1 架构扩展点概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        扩展后的搜索架构                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ API Router Layer (api/routers/search.py)                              │   │
│  │                                                                          │   │
│  │  新增请求参数:                                                          │   │
│  │  - expand_graph: bool (是否启用图扩展)                                  │   │
│  │  - graph_hops: int (最大扩展跳数，建议 1-2)                            │   │
│  │  - graph_decay: float (每跳分数衰减系数，如 0.7)                       │   │
│  │  - notebook_id: Optional[str] (可选：仅在特定 Notebook 内扩展)         │   │
│  │                                                                          │   │
│  │  新增端点:                                                              │   │
│  │  - POST /search/related (专门的"查找相关文档"接口)                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ Domain Layer (open_notebook/domain/notebook.py)                      │   │
│  │                                                                          │   │
│  │  方案 A: 应用层图扩展（推荐）                                            │   │
│  │  ┌────────────────────────────────────────────────────────────────┐   │   │
│  │  │ 新增函数: hybrid_search_with_graph()                             │   │   │
│  │  │                                                                  │   │   │
│  │  │ 执行流程:                                                         │   │   │
│  │  │ 1. 基础搜索: text_search() / vector_search()                    │   │   │
│  │  │ 2. 结果收集: 提取匹配的 Source/Note ID                           │   │   │
│  │  │ 3. 图遍历: 查询这些记录关联的 Notebook                            │   │   │
│  │  │ 4. 扩展发现: 查询 Notebook 关联的其他 Source/Note                 │   │   │
│  │  │ 5. 二次检索: 对扩展结果执行轻量搜索（验证相关性）                   │   │   │
│  │  │ 6. 分数融合: 基础分数 × 衰减系数，与二次检索分数融合               │   │   │
│  │  │ 7. 重新排序: 按最终分数降序排列                                   │   │   │
│  │  └────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                          │   │
│  │  方案 B: 数据库层图扩展                                                 │   │
│  │  ┌────────────────────────────────────────────────────────────────┐   │   │
│  │  │ 新增 SurrealQL 函数: fn::graph_expand_search()                  │   │   │
│  │  │                                                                  │   │   │
│  │  │ 执行流程:                                                         │   │   │
│  │  │ 1. 基础搜索（同现有 fn::text_search/fn::vector_search）          │   │   │
│  │  │ 2. 图遍历: 使用 SurrealDB 原生图语法（->/<-）                     │   │   │
│  │  │ 3. 合并: array::union() 基础结果 + 扩展结果                       │   │   │
│  │  │ 4. 排序: ORDER BY relevance/similarity DESC                      │   │   │
│  │  └────────────────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ Database Layer (open_notebook/database/)                              │   │
│  │                                                                          │   │
│  │  方案 A 无需改动数据库层（除了可能需要的索引优化）                       │   │
│  │                                                                          │   │
│  │  方案 B 可能需要:                                                        │   │
│  │  - 新增关系表索引: CREATE INDEX idx_reference_in/out                   │   │
│  │  - 新增 SurrealQL 函数: migrations/15.surrealql 等                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

##### 3.4.5.2 方案 A：应用层图扩展（推荐）

**核心思想**：在 Python 应用层（Domain Layer）控制图扩展逻辑，而不是委托给 SurrealDB。

**实现步骤**：

```python
# open_notebook/domain/notebook.py 中新增

from typing import List, Dict, Any, Optional
from enum import Enum
from dataclasses import dataclass


class GraphSearchMode(Enum):
    """图扩展搜索的模式"""
    SAME_NOTEBOOK = "same_notebook"      # 仅扩展同一 Notebook 内的文档
    SHARED_NOTEBOOK = "shared_notebook"   # 扩展共享 Notebook 的文档
    ALL_NEIGHBORS = "all_neighbors"       # 扩展所有关联邻居（谨慎使用）


@dataclass
class GraphSearchConfig:
    """图扩展搜索的配置"""
    enabled: bool = False
    max_hops: int = 1                    # 最大跳数，建议 1-2
    decay_factor: float = 0.7            # 每跳的分数衰减系数
    mode: GraphSearchMode = GraphSearchMode.SAME_NOTEBOOK
    min_secondary_score: float = 0.3     # 二次检索的最小分数阈值
    limit_per_hop: int = 20               # 每跳最多扩展的结果数


async def hybrid_search_with_graph(
    keyword: str,
    search_type: str = "text",           # "text" or "vector"
    results: int = 100,
    source: bool = True,
    note: bool = True,
    minimum_score: float = 0.2,           # 仅对 vector_search 有效
    graph_config: Optional[GraphSearchConfig] = None,
) -> List[Dict[str, Any]]:
    """
    支持图扩展的混合搜索。
    
    核心设计原则:
    1. 基础搜索优先（保证核心质量）
    2. 图扩展作为补充（需要验证相关性）
    3. 分数衰减明确（用户可感知）
    """
    
    # 默认配置：不启用图扩展
    if graph_config is None:
        graph_config = GraphSearchConfig(enabled=False)
    
    # ========== 步骤 1: 执行基础搜索 ==========
    logger.info(f"[hybrid_search] 基础搜索: type={search_type}, keyword={keyword}")
    
    if search_type == "vector":
        base_results = await vector_search(keyword, results, source, note, minimum_score)
        score_key = "similarity"
    else:
        base_results = await text_search(keyword, results, source, note)
        score_key = "relevance"
    
    # 如果不启用图扩展，直接返回基础结果
    if not graph_config.enabled:
        return base_results
    
    # 标记基础结果的来源和原始分数
    for r in base_results:
        r["_source_type"] = "base"
        r["_original_score"] = r[score_key]
        r["_hop_count"] = 0
    
    # 如果基础结果为空，直接返回
    if not base_results:
        return []
    
    # ========== 步骤 2: 收集基础结果中的相关实体 ==========
    # 提取所有匹配的 source_embedding 对应的 source ID
    # 注意：source_embedding 的 parent_id 指向 source
    base_source_ids = set()
    base_note_ids = set()
    
    for r in base_results:
        record_id = str(r.get("id", ""))
        parent_id = str(r.get("parent_id", ""))
        
        # 判断记录类型
        if record_id.startswith("source_embedding:"):
            # source_embedding → parent_id 是 source
            if parent_id:
                base_source_ids.add(parent_id)
        elif record_id.startswith("source:"):
            base_source_ids.add(record_id)
        elif record_id.startswith("note:"):
            base_note_ids.add(record_id)
    
    logger.info(f"[hybrid_search] 基础结果包含: {len(base_source_ids)} 个 sources, {len(base_note_ids)} 个 notes")
    
    # ========== 步骤 3: 图扩展发现 ==========
    expanded_candidates = []
    
    if graph_config.mode in [GraphSearchMode.SAME_NOTEBOOK, GraphSearchMode.SHARED_NOTEBOOK]:
        # 通过 Notebook 关系扩展
        
        # 查询这些 source/note 关联的 notebook
        # 注意：需要通过 reference/artifact 关系反向查询
        related_notebooks = set()
        
        # 查询 source → reference → notebook
        if base_source_ids:
            source_notebooks = await repo_query("""
                SELECT VALUE DISTINCT out 
                FROM reference 
                WHERE in IN $source_ids
            """, {"source_ids": list(base_source_ids)})
            related_notebooks.update(source_notebooks)
        
        # 查询 note → artifact → notebook
        if base_note_ids:
            note_notebooks = await repo_query("""
                SELECT VALUE DISTINCT out 
                FROM artifact 
                WHERE in IN $note_ids
            """, {"note_ids": list(base_note_ids)})
            related_notebooks.update(note_notebooks)
        
        logger.info(f"[hybrid_search] 关联的 notebooks: {len(related_notebooks)}")
        
        if related_notebooks:
            # 查询这些 notebook 中的其他 source/note
            # 根据模式决定扩展范围
            
            if graph_config.mode == GraphSearchMode.SAME_NOTEBOOK:
                # 仅获取这些 notebook 中的记录
                # （但排除已经在基础结果中的）
                
                # 查询 notebook 中的其他 sources
                notebook_sources = await repo_query("""
                    SELECT VALUE DISTINCT in 
                    FROM reference 
                    WHERE out IN $notebook_ids
                """, {"notebook_ids": list(related_notebooks)})
                
                # 查询 notebook 中的其他 notes
                notebook_notes = await repo_query("""
                    SELECT VALUE DISTINCT in 
                    FROM artifact 
                    WHERE out IN $notebook_ids
                """, {"notebook_ids": list(related_notebooks)})
                
                # 排除已在基础结果中的
                expand_source_ids = [s for s in notebook_sources if s not in base_source_ids]
                expand_note_ids = [n for n in notebook_notes if n not in base_note_ids]
                
                logger.info(f"[hybrid_search] 待扩展: {len(expand_source_ids)} sources, {len(expand_note_ids)} notes")
                
                # 收集为候选
                for sid in expand_source_ids[:graph_config.limit_per_hop]:
                    expanded_candidates.append({
                        "id": sid,
                        "_source_type": "expanded",
                        "_hop_count": 1,
                        "_expansion_reason": f"shared_notebook",
                    })
                
                for nid in expand_note_ids[:graph_config.limit_per_hop]:
                    expanded_candidates.append({
                        "id": nid,
                        "_source_type": "expanded",
                        "_hop_count": 1,
                        "_expansion_reason": f"shared_notebook",
                    })
    
    # ========== 步骤 4: 对扩展候选执行二次检索（验证相关性） ==========
    # 关键：不直接接受图扩展的结果，而是对候选执行一次轻量搜索验证
    # 这样可以过滤掉"同属一个 Notebook 但语义不相关"的噪音
    
    validated_results = []
    
    if expanded_candidates:
        logger.info(f"[hybrid_search] 执行二次检索验证，候选数: {len(expanded_candidates)}")
        
        # 提取候选的 ID 列表
        candidate_ids = [c["id"] for c in expanded_candidates]
        
        if search_type == "vector":
            # 向量检索：需要先生成查询向量
            from open_notebook.utils.embedding import generate_embedding
            query_embedding = await generate_embedding(keyword)
            
            # 对候选执行向量相似度计算
            # 注意：这里只在候选范围内搜索
            secondary_results = await repo_query("""
                SELECT 
                    id,
                    parent_id,
                    title,
                    vector::similarity::cosine(embedding, $query_embedding) as similarity
                FROM (
                    SELECT * FROM source_embedding WHERE source IN $candidate_ids
                    UNION ALL
                    SELECT * FROM source_insight WHERE source IN $candidate_ids
                    UNION ALL
                    SELECT * FROM note WHERE id IN $candidate_ids
                )
                WHERE embedding != NONE
                  AND vector::similarity::cosine(embedding, $query_embedding) >= $min_score
                ORDER BY similarity DESC
                LIMIT $limit
            """, {
                "query_embedding": query_embedding,
                "candidate_ids": candidate_ids,
                "min_score": graph_config.min_secondary_score,
                "limit": results,
            })
            
            # 处理 secondary_results，添加元数据
            for r in secondary_results:
                r["_source_type"] = "validated_expansion"
                r["_hop_count"] = 1
                r["_original_score"] = r["similarity"]
                # 应用衰减
                r["similarity"] = r["similarity"] * graph_config.decay_factor
                validated_results.append(r)
                
        else:
            # 全文检索：对候选执行关键词匹配
            secondary_results = await repo_query("""
                SELECT 
                    id,
                    parent_id,
                    title,
                    math::max(search::score(1)) as relevance
                FROM (
                    SELECT id, parent_id, title FROM source WHERE id IN $candidate_ids AND title @1@ $keyword
                    UNION ALL
                    SELECT source.id as id, source.id as parent_id, source.title as title 
                    FROM source_embedding WHERE source IN $candidate_ids AND content @1@ $keyword
                    UNION ALL
                    SELECT id, id as parent_id, title FROM note WHERE id IN $candidate_ids 
                    AND (title @1@ $keyword OR content @1@ $keyword)
                )
                GROUP BY id, parent_id, title
                HAVING relevance >= $min_score
                ORDER BY relevance DESC
                LIMIT $limit
            """, {
                "keyword": keyword,
                "candidate_ids": candidate_ids,
                "min_score": graph_config.min_secondary_score,
                "limit": results,
            })
            
            # 处理 secondary_results
            for r in secondary_results:
                r["_source_type"] = "validated_expansion"
                r["_hop_count"] = 1
                r["_original_score"] = r["relevance"]
                # 应用衰减
                r["relevance"] = r["relevance"] * graph_config.decay_factor
                validated_results.append(r)
    
    logger.info(f"[hybrid_search] 二次验证通过: {len(validated_results)} 个结果")
    
    # ========== 步骤 5: 合并、去重、重新排序 ==========
    
    # 合并基础结果和验证后的扩展结果
    all_results = base_results + validated_results
    
    # 去重：保留最高分数
    # 使用字典，key 为 (id, parent_id)
    deduped = {}
    
    for r in all_results:
        key = (str(r.get("id", "")), str(r.get("parent_id", "")))
        
        if key not in deduped:
            deduped[key] = r
        else:
            # 比较分数，保留较高的
            existing_score = deduped[key].get(score_key, 0)
            current_score = r.get(score_key, 0)
            
            if current_score > existing_score:
                # 如果扩展结果的分数更高（经过衰减后），保留它
                # 但保留 _source_type 为 "base" 以指示来源
                r["_source_type"] = deduped[key].get("_source_type", "base")
                deduped[key] = r
    
    # 转换为列表并排序
    final_results = list(deduped.values())
    final_results.sort(key=lambda x: x.get(score_key, 0), reverse=True)
    
    # 限制数量
    final_results = final_results[:results]
    
    logger.info(f"[hybrid_search] 最终结果: {len(final_results)} 个 (基础 {len(base_results)}, 扩展 {len(validated_results)})")
    
    return final_results
```

**方案 A 的配套数据访问方法**：

```python
# 需要在 Source 和 Notebook 类中补充以下方法

class Source(ObjectModel):
    # ... 现有代码 ...
    
    async def get_linked_notebooks(self) -> List[str]:
        """获取此 Source 关联的所有 Notebook ID"""
        result = await repo_query("""
            SELECT VALUE out 
            FROM reference 
            WHERE in = $source_id
        """, {"source_id": ensure_record_id(self.id)})
        return [str(r) for r in result]
    
    async def get_notebook_context(self) -> Dict[str, Any]:
        """获取此 Source 在各 Notebook 中的上下文信息"""
        notebooks = await self.get_linked_notebooks()
        # 可以扩展：查询 Notebook 中的其他 Source，计算相似度等
        return {"notebooks": notebooks}


class Notebook(ObjectModel):
    # ... 现有代码 ...
    
    async def search_within(
        self, 
        keyword: str, 
        search_type: str = "text",
        limit: int = 50
    ) -> List[Dict[str, Any]]:
        """在此 Notebook 范围内搜索"""
        # 获取此 Notebook 中的所有 source 和 note
        source_ids = await repo_query("""
            SELECT VALUE in FROM reference WHERE out = $notebook_id
        """, {"notebook_id": ensure_record_id(self.id)})
        
        note_ids = await repo_query("""
            SELECT VALUE in FROM artifact WHERE out = $notebook_id
        """, {"notebook_id": ensure_record_id(self.id)})
        
        # 对这些 ID 执行搜索（可以复用现有逻辑）
        # ...
        
        return []
```

**方案 A 的优劣分析**：

| 优势 | 劣势 |
|-----|------|
| **灵活可控**：可以精确控制扩展逻辑、分数融合策略、二次验证机制 | **多次数据库往返**：基础搜索 → 图查询 → 二次检索，涉及多次数据库交互 |
| **业务规则丰富**：可以应用复杂的业务规则，如"仅在特定 Notebook 内扩展"、"排除某些类型的文档"等 | **部分逻辑重复**：二次检索的逻辑与基础搜索有重叠，需要维护两份相似代码 |
| **易于调试**：Python 代码易于添加日志、断点调试，可以清晰地看到每一步的结果 | **需要额外索引**：关系表的 `in` 和 `out` 字段可能需要索引来加速图查询 |
| **渐进式增强**：可以作为现有搜索的**可选增强**，默认关闭，用户显式请求时才启用 | |
| **安全性高**：二次验证机制可以过滤掉大部分噪音，保证扩展结果的质量 | |

---

##### 3.4.5.3 方案 B：数据库层图扩展

**核心思想**：在 SurrealQL 函数中直接使用图遍历语法，利用 SurrealDB 的原生能力。

**实现示例**：

```surrealql
-- migrations/15_graph_search.surrealql

DEFINE FUNCTION IF NOT EXISTS fn::hybrid_search_with_graph(
    $query_text: string,
    $match_count: int,
    $sources: bool,
    $show_notes: bool,
    $expand_graph: bool,
    $max_hops: int,
    $decay_factor: float
) {
    -- ========== 步骤 1: 基础搜索 ==========
    let $base_results = SELECT * FROM fn::text_search(
        $query_text, $match_count * 2, $sources, $show_notes
    );
    
    IF !$expand_graph OR $max_hops <= 0 {
        RETURN $base_results LIMIT $match_count;
    }
    
    -- 标记基础结果
    LET $base_with_meta = SELECT 
        *,
        "base" as _source_type,
        0 as _hop_count
    FROM $base_results;
    
    -- ========== 步骤 2: 图扩展 ==========
    -- 收集基础结果中的 ID
    LET $base_ids = SELECT VALUE id FROM $base_results;
    
    -- 查找这些 ID 关联的 Notebook
    -- 注意：需要区分 source 和 note
    LET $linked_notebooks = SELECT VALUE DISTINCT array::concat(
        (SELECT VALUE out FROM reference WHERE in IN $base_ids),
        (SELECT VALUE out FROM artifact WHERE in IN $base_ids)
    );
    
    LET $flat_notebooks = array::flatten($linked_notebooks);
    
    IF array::len($flat_notebooks) = 0 {
        RETURN $base_with_meta LIMIT $match_count;
    }
    
    -- 查找这些 Notebook 中的其他 Source/Note
    LET $expanded_sources = SELECT VALUE in 
        FROM reference 
        WHERE out IN $flat_notebooks 
          AND in NOT IN $base_ids;
    
    LET $expanded_notes = SELECT VALUE in 
        FROM artifact 
        WHERE out IN $flat_notebooks 
          AND in NOT IN $base_ids;
    
    LET $all_expanded = array::union($expanded_sources, $expanded_notes);
    
    IF array::len($all_expanded) = 0 {
        RETURN $base_with_meta LIMIT $match_count;
    }
    
    -- ========== 步骤 3: 对扩展结果执行关键词匹配（验证） ==========
    LET $expanded_results = SELECT 
        id,
        parent_id,
        title,
        search::score(1) * $decay_factor as relevance,
        "expanded" as _source_type,
        1 as _hop_count
    FROM (
        SELECT id, id as parent_id, title FROM source 
        WHERE id IN $all_expanded AND title @1@ $query_text
        
        UNION ALL
        
        SELECT source.id as id, source.id as parent_id, source.title as title 
        FROM source_embedding 
        WHERE source IN $all_expanded AND content @1@ $query_text
        
        UNION ALL
        
        SELECT id, id as parent_id, title FROM note 
        WHERE id IN $all_expanded AND (title @1@ $query_text OR content @1@ $query_text)
    )
    GROUP BY id, parent_id, title
    ORDER BY relevance DESC
    LIMIT $match_count;
    
    -- ========== 步骤 4: 合并并去重 ==========
    LET $all_results = array::union($base_with_meta, $expanded_results);
    
    RETURN (
        SELECT 
            id, 
            parent_id, 
            title, 
            math::max(relevance) as relevance,
            array::first(_source_type) as _source_type,  -- 优先保留 "base"
            math::min(_hop_count) as _hop_count
        FROM $all_results 
        GROUP BY id, parent_id, title 
        ORDER BY relevance DESC 
        LIMIT $match_count
    );
};
```

**方案 B 的优劣分析**：

| 优势 | 劣势 |
|-----|------|
| **单次查询**：所有逻辑在一个 SurrealQL 函数中完成，减少网络往返 | **SurrealQL 能力限制**：复杂的条件逻辑、循环、数据结构操作在 SurrealQL 中表达困难 |
| **利用数据库优化**：SurrealDB 可能对图遍历语法有内部优化（虽然当前实现主要是语法糖） | **调试困难**：SurrealQL 函数的调试体验远不如 Python，缺乏日志、断点、错误堆栈 |
| **原子性**：整个搜索过程在数据库层面是原子的 | **分数融合受限**：难以实现复杂的分数融合策略（如 RRF、加权求和） |
| | **性能不可控**：图遍历在数据量大时可能很慢，且难以诊断和优化 |
| | **版本依赖**：SurrealDB 的图语法和行为可能在版本间变化 |

---

##### 3.4.5.4 两种方案对比总结

| 维度 | 方案 A（应用层图扩展） | 方案 B（数据库层图扩展） |
|-----|----------------------|-----------------------|
| **推荐度** | ⭐⭐⭐⭐⭐ 强烈推荐 | ⭐⭐ 谨慎考虑 |
| **实现复杂度** | 中等（Python 代码） | 高（SurrealQL 限制多） |
| **调试体验** | 优秀（日志、断点） | 困难（SurrealQL 工具链弱） |
| **性能** | 多次数据库往返 | 单次查询 |
| **灵活性** | 极高（可任意定制） | 有限（受 SurrealQL 能力限制） |
| **业务规则** | 易于集成 | 难以表达复杂业务规则 |
| **渐进式增强** | 支持（可选启用） | 可能需要替换现有函数 |
| **安全性** | 高（二次验证机制） | 中等（依赖数据库查询优化） |

**最终建议**：

1. **短期**：采用**方案 A**，在 `open_notebook/domain/notebook.py` 中实现 `hybrid_search_with_graph()` 函数，作为现有 `text_search()` 和 `vector_search()` 的**可选增强**。

2. **API 设计**：不修改现有 `/search` 端点的默认行为，而是：
   - 添加可选参数 `expand_graph`（默认 `false`）
   - 或者新增专门的端点 `/search/related` 用于显式的图扩展搜索

3. **关键保障**：**必须实现二次验证机制**，即对图扩展发现的候选结果执行一次轻量的全文/向量搜索验证，过滤掉"同属一个 Notebook 但语义不相关"的噪音。

4. **长期**：如果 SurrealDB 未来增强了图数据库能力（如原生图索引、更丰富的图算法），可以考虑迁移到方案 B 或混合方案。

---

## 4. 查询结果的合并、去重和排序

### 4.1 合并策略

合并操作完全在 **SurrealDB 函数内部** 执行，使用 `array::union()` 函数。

#### 4.1.1 全文检索的合并流程

```surrealql
-- 第一层: 合并 Source 相关搜索
let $source_chunk_results = array::union($source_embedding_search, $source_full_search);
let $source_asset_results = array::union($source_title_search, $source_insight_search);

-- 第二层: 合并 Source 整体
let $source_results = array::union($source_chunk_results, $source_asset_results);

-- 第三层: 合并 Note 相关搜索
let $note_results = array::union($note_title_search, $note_content_search);

-- 第四层: 合并所有结果
let $final_results = array::union($source_results, $note_results);
```

#### 4.1.2 向量检索的合并流程

```surrealql
let $all_results = array::union(
    array::union($source_embedding_search, $source_insight_search),
    $note_content_search
);
```

**`array::union()` 的特性**:
- 自动去重
- 保持元素顺序（保留第一次出现的位置）

### 4.2 去重策略

去重通过 **GROUP BY** 子句实现，同时保留最高分数。

#### 4.2.1 全文检索去重

```surrealql
RETURN (
    SELECT id, parent_id, title, math::max(relevance) as relevance
    FROM $final_results 
    WHERE id is not None
    GROUP BY id, parent_id, title 
    ORDER BY relevance DESC 
    LIMIT $match_count
);
```

#### 4.2.2 向量检索去重

```surrealql
RETURN (
    SELECT id, parent_id, title, math::max(similarity) as similarity,
           array::flatten(content) as matches
    FROM $all_results 
    WHERE id is not None
    GROUP BY id, parent_id, title 
    ORDER BY similarity DESC 
    LIMIT $match_count
);
```

**去重逻辑**:
- **分组键**: `id`, `parent_id`, `title`
- **分数聚合**: `math::max(relevance)` 或 `math::max(similarity)`
- **内容聚合**: `array::flatten(content)` 合并所有匹配内容

**场景说明**:
- 同一 Source 可能在 `source.title`、`source.full_text`、`source_embedding.content` 中多次匹配
- 去重后保留最高的 relevance/similarity 分数
- 合并所有匹配的 content 片段

### 4.3 排序策略

#### 4.3.1 分数类型

| 检索类型 | 分数字段 | 分数来源 | 范围 |
|---------|---------|---------|------|
| 全文检索 | `relevance` | `search::score(1)` (BM25) | 正值，越大越相关 |
| 向量检索 | `similarity` | `vector::similarity::cosine()` | [-1, 1]，越大越相似 |

#### 4.3.2 排序方式

```surrealql
-- 全文检索
ORDER BY relevance DESC

-- 向量检索
ORDER BY similarity DESC
```

**统一降序排序**: 分数越高的结果排在前面。

#### 4.3.3 结果限制

```surrealql
LIMIT $match_count
```

由 API 层的 `search_request.limit` 参数控制，默认 100，最大 1000。

### 4.4 两种检索的结果对比

**全文检索返回字段**:
- `id`: 记录 ID
- `parent_id`: 父记录 ID（如 source_embedding 指向 source）
- `title`: 标题
- `relevance`: BM25 相关性分数

**向量检索返回字段**:
- `id`: 记录 ID
- `parent_id`: 父记录 ID
- `title`: 标题
- `similarity`: 余弦相似度分数
- `matches`: 匹配的内容片段（通过 `array::flatten(content)` 合并）

---

## 5. 向量嵌入数据的写入时机与一致性保障

### 5.1 嵌入数据模型

#### 5.1.1 表结构

| 表名 | 内容 | 嵌入存储位置 |
|-----|------|-------------|
| `source` | 源文档元数据 | 不直接存储嵌入 |
| `source_embedding` | 源文档分块 | **每条记录一个 embedding** |
| `source_insight` | 源文档洞察 | **每条记录一个 embedding** |
| `note` | 笔记 | **每条记录一个 embedding** |

#### 5.1.2 嵌入字段定义

```surrealql
-- source_embedding 表 (migrations/1.surrealql:16-20)
DEFINE TABLE IF NOT EXISTS source_embedding SCHEMAFULL;
DEFINE FIELD IF NOT EXISTS source ON TABLE source_embedding TYPE record<source>;
DEFINE FIELD IF NOT EXISTS order ON TABLE source_embedding TYPE int;
DEFINE FIELD IF NOT EXISTS content ON TABLE source_embedding TYPE string;
DEFINE FIELD IF NOT EXISTS embedding ON TABLE source_embedding TYPE array<float>;

-- source_insight 表 (migrations/1.surrealql:22-26)
DEFINE TABLE IF NOT EXISTS source_insight SCHEMAFULL;
DEFINE FIELD IF NOT EXISTS source ON TABLE source_insight TYPE record<source>;
DEFINE FIELD IF NOT EXISTS insight_type ON TABLE source_insight TYPE string;
DEFINE FIELD IF NOT EXISTS content ON TABLE source_insight TYPE string;
DEFINE FIELD IF NOT EXISTS embedding ON TABLE source_insight TYPE array<float>;

-- note 表 (migrations/1.surrealql:34-42)
DEFINE TABLE IF NOT EXISTS note SCHEMAFULL;
DEFINE FIELD IF NOT EXISTS title ON TABLE note TYPE option<string>;
DEFINE FIELD IF NOT EXISTS content ON TABLE note TYPE option<string>;
DEFINE FIELD IF NOT EXISTS embedding ON TABLE note TYPE array<float>;
```

### 5.2 写入时机

嵌入写入通过 **surreal-commands** 框架**异步执行**，不阻塞主请求流程。

#### 5.2.1 Source 嵌入

**触发时机**: 调用 `Source.vectorize()` 或上传 Source 时

**代码位置**: `open_notebook/domain/notebook.py:411-457`

```python
async def vectorize(self) -> str:
    """
    提交向量化作为后台任务，使用 embed_source 命令。
    
    返回: 可用于跟踪进度的命令/任务 ID
    """
    logger.info(f"Submitting embed_source job for source {self.id}")
    
    # 提交 embed_source 命令
    command_id = submit_command(
        "open_notebook",
        "embed_source",
        {"source_id": str(self.id)},
    )
    return command_id_str
```

**命令实现**: `commands/embedding_commands.py:307-440`

```python
@command(
    "embed_source",
    app="open_notebook",
    retry={
        "max_attempts": 5,
        "wait_strategy": "exponential_jitter",
        "wait_min": 1,
        "wait_max": 60,
        "stop_on": [ValueError, ConfigurationError],
    },
)
async def embed_source_command(input_data: EmbedSourceInput) -> EmbedSourceOutput:
    """
    生成并存储源文档的嵌入向量。
    
    流程:
    1. 按 ID 加载 Source
    2. 删除此 source 已有的 source_embedding 记录（幂等性）
    3. 从文件路径或内容检测内容类型
    4. 使用合适的分割器分块文本
    5. 批量生成所有块的嵌入
    6. 批量 INSERT source_embedding 记录
    """
    # 1. 加载 source
    source = await Source.get(input_data.source_id)
    
    # 2. 删除现有嵌入（幂等性）
    await repo_query(
        "DELETE source_embedding WHERE source = $source_id",
        {"source_id": ensure_record_id(input_data.source_id)},
    )
    
    # 3. 检测内容类型
    content_type = detect_content_type(source.full_text, file_path)
    
    # 4. 分块
    chunks = chunk_text(source.full_text, content_type=content_type)
    
    # 5. 生成嵌入
    embeddings = await generate_embeddings(chunks, command_id=cmd_id)
    
    # 6. 批量插入
    records = [
        {
            "source": ensure_record_id(input_data.source_id),
            "order": idx,
            "content": chunk,
            "embedding": embedding,
        }
        for idx, (chunk, embedding) in enumerate(zip(chunks, embeddings))
    ]
    await repo_insert("source_embedding", records)
```

#### 5.2.2 Note 嵌入

**触发时机**: 调用 `Note.save()` 时

**代码位置**: `open_notebook/domain/notebook.py:570-593`

```python
async def save(self) -> Optional[str]:
    """
    保存 note 并提交嵌入命令。
    
    重写 ObjectModel.save()，在保存后提交异步 embed_note 命令，
    而不是内联嵌入。
    """
    # 调用父类 save（不包含嵌入）
    await super().save()
    
    # 如果 note 有内容，提交嵌入命令（发射后不管）
    if self.id and self.content and self.content.strip():
        command_id = submit_command(
            "open_notebook",
            "embed_note",
            {"note_id": str(self.id)},
        )
        return command_id
    
    return None
```

**命令实现**: `commands/embedding_commands.py:121-210`

```python
@command("embed_note", app="open_notebook", retry={...})
async def embed_note_command(input_data: EmbedNoteInput) -> EmbedNoteOutput:
    """
    生成并存储单个 note 的嵌入向量。
    
    流程:
    1. 按 ID 加载 Note
    2. 生成嵌入（自动分块 + 平均池化，如果需要）
    3. UPSERT note 记录中的 embedding 字段
    """
    # 1. 加载 note
    note = await Note.get(input_data.note_id)
    
    # 2. 生成嵌入
    embedding = await generate_embedding(
        note.content, content_type=ContentType.MARKDOWN, command_id=cmd_id
    )
    
    # 3. UPSERT 嵌入到 note 记录
    await repo_query(
        "UPDATE $note_id SET embedding = $embedding",
        {
            "note_id": ensure_record_id(input_data.note_id),
            "embedding": embedding,
        },
    )
```

#### 5.2.3 SourceInsight 嵌入

**触发时机**: 调用 `create_insight_command` 创建洞察后

**代码位置**: `commands/embedding_commands.py:443-546`

```python
@command("create_insight", app="open_notebook", retry={...})
async def create_insight_command(input_data: CreateInsightInput) -> CreateInsightOutput:
    """
    创建 source insight，带事务冲突自动重试。
    
    流程:
    1. CREATE source_insight 记录到数据库
    2. 提交 embed_insight 命令（发射后不管）用于异步嵌入
    3. 返回 insight_id
    """
    # 1. 创建 insight 记录
    result = await repo_query("""
        CREATE source_insight CONTENT {
            "source": $source_id,
            "insight_type": $insight_type,
            "content": $content
        };
    """, {...})
    
    insight_id = str(result[0].get("id", ""))
    
    # 2. 提交嵌入命令（发射后不管）
    submit_command(
        "open_notebook",
        "embed_insight",
        {"insight_id": insight_id},
    )
    
    return CreateInsightOutput(success=True, insight_id=insight_id, ...)
```

**embed_insight 命令**: `commands/embedding_commands.py:213-304`

```python
@command("embed_insight", app="open_notebook", retry={...})
async def embed_insight_command(input_data: EmbedInsightInput) -> EmbedInsightOutput:
    """
    生成并存储单个 source insight 的嵌入向量。
    
    流程:
    1. 按 ID 加载 SourceInsight
    2. 生成嵌入（自动分块 + 平均池化，如果需要）
    3. UPSERT insight 记录中的 embedding 字段
    """
    # 1. 加载 insight
    insight = await SourceInsight.get(input_data.insight_id)
    
    # 2. 生成嵌入
    embedding = await generate_embedding(
        insight.content, content_type=ContentType.MARKDOWN, command_id=cmd_id
    )
    
    # 3. UPSERT 嵌入
    await repo_query(
        "UPDATE $insight_id SET embedding = $embedding",
        {
            "insight_id": ensure_record_id(input_data.insight_id),
            "embedding": embedding,
        },
    )
```

### 5.3 一致性保障

#### 5.3.1 重试机制

所有嵌入命令都配置了 **指数退避重试**:

```python
retry={
    "max_attempts": 5,           # 最多重试 5 次
    "wait_strategy": "exponential_jitter",  # 指数抖动退避
    "wait_min": 1,               # 最小等待 1 秒
    "wait_max": 60,              # 最大等待 60 秒
    "stop_on": [ValueError, ConfigurationError],  # 遇到这些错误不重试
}
```

**重试场景**:
- 网络超时
- API 限流
- 数据库事务冲突
- 临时服务不可用

**不重试场景**:
- `ValueError`: 验证错误（如内容为空）
- `ConfigurationError`: 配置错误（如 embedding model 未配置）

#### 5.3.2 幂等性设计

**Source 嵌入的幂等性**:
```python
# embed_source_command 中先删除现有嵌入
await repo_query(
    "DELETE source_embedding WHERE source = $source_id",
    {"source_id": ensure_record_id(input_data.source_id)},
)
# 然后重新插入
await repo_insert("source_embedding", records)
```

**Note/Insight 嵌入的幂等性**:
```python
# 使用 UPSERT 语义
await repo_query(
    "UPDATE $note_id SET embedding = $embedding",
    {"note_id": ensure_record_id(input_data.note_id), "embedding": embedding},
)
```

#### 5.3.3 级联清理

**Source 删除时级联清理嵌入**:

**数据库事件** (`migrations/1.surrealql:29-32`):
```surrealql
DEFINE EVENT IF NOT EXISTS source_delete ON TABLE source WHEN ($after == NONE) THEN {
    delete source_embedding where source == $before.id;
    delete source_insight where source == $before.id;
};
```

**应用层级联** (`open_notebook/domain/notebook.py:516-554`):
```python
async def delete(self) -> bool:
    # 删除关联的嵌入和洞察，防止孤立记录
    try:
        source_id = ensure_record_id(self.id)
        await repo_query(
            "DELETE source_embedding WHERE source = $source_id",
            {"source_id": source_id},
        )
        await repo_query(
            "DELETE source_insight WHERE source = $source_id",
            {"source_id": source_id},
        )
    except Exception as e:
        logger.warning(...)
    
    # 调用父类删除
    return await super().delete()
```

#### 5.3.4 嵌入重建机制

提供 `rebuild_embeddings` 命令用于批量重建嵌入:

**代码位置**: `commands/embedding_commands.py:622-787`

```python
@command("rebuild_embeddings", app="open_notebook", retry=None)
async def rebuild_embeddings_command(input_data: RebuildEmbeddingsInput) -> RebuildEmbeddingsOutput:
    """
    重建 sources、notes 和/或 insights 的嵌入。
    
    模式:
    - "existing": 仅重建已有嵌入的项目
    - "all": 重建所有有内容的项目
    
    此命令为每个项目提交单独的嵌入任务:
    - embed_source 用于 sources
    - embed_note 用于 notes
    - embed_insight 用于 insights
    """
    # 收集需要处理的项目
    items = await collect_items_for_rebuild(
        input_data.mode,
        input_data.include_sources,
        input_data.include_notes,
        input_data.include_insights,
    )
    
    # 提交嵌入命令
    for source_id in items["sources"]:
        submit_command("open_notebook", "embed_source", {"source_id": source_id})
    
    for note_id in items["notes"]:
        submit_command("open_notebook", "embed_note", {"note_id": note_id})
    
    for insight_id in items["insights"]:
        submit_command("open_notebook", "embed_insight", {"insight_id": insight_id})
```

#### 5.3.5 向量搜索时的一致性检查

向量搜索函数包含维度一致性检查:

```surrealql
WHERE embedding != none 
  AND array::len(embedding)=array::len($query)  -- 维度检查
  AND vector::similarity::cosine(embedding, $query) >= $min_similarity
```

**保护机制**:
- 跳过 `embedding` 为 `none` 的记录
- 跳过嵌入维度与查询向量维度不匹配的记录
- 跳过相似度低于阈值的记录

### 5.4 嵌入写入流程总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           嵌入写入触发点                                   │
├─────────────────────────┬─────────────────────────┬─────────────────────┤
│    Source.vectorize()   │     Note.save()         │ create_insight()    │
│    (手动或上传时)        │    (保存笔记时)          │   (创建洞察时)       │
└───────────┬─────────────┴───────────┬─────────────┴───────────┬─────────┘
            │                         │                         │
            ▼                         ▼                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     surreal-commands 异步任务队列                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐              │
│  │embed_source │  │ embed_note  │  │   embed_insight     │              │
│  │  命令处理    │  │  命令处理   │  │     命令处理         │              │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘              │
└─────────┼────────────────┼────────────────────┼─────────────────────────┘
          │                │                    │
          ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        嵌入生成与存储                                      │
│                                                                           │
│  embed_source 流程:                                                       │
│  1. 加载 Source                                                          │
│  2. DELETE source_embedding WHERE source = $id (幂等)                   │
│  3. chunk_text(full_text) → 多个分块                                     │
│  4. generate_embeddings(chunks) → 多个向量                               │
│  5. INSERT INTO source_embedding (批量)                                  │
│                                                                           │
│  embed_note / embed_insight 流程:                                        │
│  1. 加载 Note/Insight                                                    │
│  2. generate_embedding(content) → 单个向量 (自动分块+平均池化)          │
│  3. UPDATE note/insight SET embedding = $vector                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Ask 功能中的检索

### 6.1 Ask 功能概述

Ask 功能是一个基于 LangGraph 的 AI 问答系统，使用向量检索获取相关上下文。

**入口**: `api/routers/search.py:113-157`

```python
@router.post("/search/ask")
async def ask_knowledge_base(ask_request: AskRequest):
    # 验证模型
    strategy_model = await Model.get(ask_request.strategy_model)
    answer_model = await Model.get(ask_request.answer_model)
    final_answer_model = await Model.get(ask_request.final_answer_model)
    
    # 检查 embedding model
    if not await model_manager.get_embedding_model():
        raise HTTPException(...)
    
    # 流式返回
    return StreamingResponse(
        stream_ask_response(ask_request.question, ...),
        media_type="text/plain",
    )
```

### 6.2 LangGraph 工作流

**图定义**: `open_notebook/graphs/ask.py:146-154`

```python
agent_state = StateGraph(ThreadState)
agent_state.add_node("agent", call_model_with_messages)
agent_state.add_node("provide_answer", provide_answer)
agent_state.add_node("write_final_answer", write_final_answer)
agent_state.add_edge(START, "agent")
agent_state.add_conditional_edges("agent", trigger_queries, ["provide_answer"])
agent_state.add_edge("provide_answer", "write_final_answer")
agent_state.add_edge("write_final_answer", END)

graph = agent_state.compile()
```

### 6.3 工作流中的检索

**步骤 1: 策略生成** (`call_model_with_messages`)

```python
async def call_model_with_messages(state: ThreadState, config: RunnableConfig) -> dict:
    # 使用 strategy_model 生成搜索策略
    # 返回: {"strategy": Strategy}
    # Strategy 包含: reasoning, searches (List[Search])
    # Search 包含: term, instructions
```

**步骤 2: 并行检索触发** (`trigger_queries`)

```python
async def trigger_queries(state: ThreadState, config: RunnableConfig):
    # 为每个搜索策略创建并行的 provide_answer 任务
    return [
        Send(
            "provide_answer",
            {
                "question": state["question"],
                "instructions": s.instructions,
                "term": s.term,
            },
        )
        for s in state["strategy"].searches
    ]
```

**步骤 3: 执行检索** (`provide_answer`)

```python
async def provide_answer(state: SubGraphState, config: RunnableConfig) -> dict:
    # 注意: 这里硬编码使用 vector_search
    results = await vector_search(state["term"], 10, True, True)
    
    if len(results) == 0:
        return {"answers": []}
    
    # 使用检索结果构建 prompt，调用 answer_model 生成答案
    payload["results"] = results
    system_prompt = Prompter(prompt_template="ask/query_process").render(data=payload)
    ai_message = await model.ainvoke(system_prompt)
    
    return {"answers": [clean_thinking_content(ai_content)]}
```

**重要发现**: Ask 功能**仅使用向量检索** (`vector_search`)，不使用全文检索。

---

## 7. 总结

### 7.1 混合检索方案的组成

| 检索类型 | 触发条件 | 核心技术 | 数据来源 |
|---------|---------|---------|---------|
| **全文检索** | `type="text"` 或默认 | BM25 全文索引 | `source`, `source_embedding`, `source_insight`, `note` |
| **向量检索** | `type="vector"` | 余弦相似度 | `source_embedding`, `source_insight`, `note` |
| **图关系** | 未直接用于搜索 | SurrealDB RELATE | 用于数据组织和级联操作 |

### 7.2 结果处理流程

```
多个子查询
    ↓
array::union() 合并 (自动去重)
    ↓
GROUP BY id, parent_id, title
    ↓
math::max(relevance/similarity) 保留最高分数
    ↓
ORDER BY score DESC 排序
    ↓
LIMIT $match_count 限制数量
```

### 7.3 嵌入数据一致性保障

| 保障机制 | 实现方式 |
|---------|---------|
| **异步处理** | surreal-commands 任务队列 |
| **自动重试** | 指数退避，最多 5 次 |
| **幂等性** | 先删除再插入 (source) / UPSERT (note/insight) |
| **级联清理** | 数据库 EVENT + 应用层删除 |
| **重建能力** | rebuild_embeddings 命令 |
| **运行时检查** | 维度匹配 + 空值过滤 |

### 7.4 架构亮点

1. **数据库层聚合**: 合并、去重、排序完全在 SurrealDB 函数内完成，减少数据传输
2. **异步嵌入**: 嵌入生成不阻塞主请求，提升用户体验
3. **统一嵌入接口**: `generate_embedding` 自动处理大文本分块和平均池化
4. **灵活的重试策略**: 区分临时故障和永久错误
5. **多模型支持**: Ask 功能使用三个不同模型 (策略/答案/最终答案)

### 7.5 设计决策总结

| 决策点 | 当前选择 | 理由 |
|-------|---------|------|
| **搜索时不做图遍历** | 直接表查询 + 索引查找 | 性能可预测、实现简单、结果可解释 |
| **向量/全文检索分离** | 两种独立模式，用户选择 | 分数语义不同（BM25 vs 余弦相似度），融合需要额外设计 |
| **异步嵌入** | surreal-commands 后台任务 | 不阻塞主请求，支持重试和幂等 |
| **数据库层聚合** | SurrealQL 函数内合并去重排序 | 减少数据传输，利用数据库优化 |

### 7.6 未来扩展方向

1. **混合分数融合**（向量 + 全文）:
   - 同时执行两种检索
   - 分数归一化（如 min-max scaling、z-score）
   - 加权融合或 RRF (Reciprocal Rank Fusion)

2. **图关系扩展搜索**（详见 3.4.5）:
   - 推荐方案：应用层图扩展（Domain Layer）
   - 新增参数：`expand_graph`, `graph_hops`
   - 分数衰减策略：1 跳 0.7，2 跳 0.4

3. **嵌入状态跟踪**:
   - 为异步嵌入任务添加状态查询 API
   - 在 Source/Note 模型中添加 `embedding_status` 字段

---

## 8. 关键代码位置索引

### 8.1 服务端搜索链路（核心）

| 功能模块 | 文件路径 | 关键函数/类 |
|---------|----------|------------|
| API 端点 | `api/routers/search.py` | `search_knowledge_base()`, `ask_knowledge_base()` |
| 业务层搜索 | `open_notebook/domain/notebook.py` | `text_search()`, `vector_search()` |
| 全文检索函数 | `open_notebook/database/migrations/9.surrealql` | `fn::text_search()` |
| 向量检索函数 | `open_notebook/database/migrations/9.surrealql` | `fn::vector_search()` |
| 嵌入生成 | `open_notebook/utils/embedding.py` | `generate_embedding()`, `generate_embeddings()`, `mean_pool_embeddings()` |
| 嵌入命令 | `commands/embedding_commands.py` | `embed_source_command()`, `embed_note_command()`, `embed_insight_command()` |
| Ask 工作流 | `open_notebook/graphs/ask.py` | `graph` (LangGraph 编译结果) |
| 关系操作 | `open_notebook/database/repository.py` | `repo_relate()` |
| 数据库连接 | `open_notebook/database/repository.py` | `db_connection()`, `repo_query()` |

### 8.2 客户端组件（不参与服务端搜索链路）

| 组件 | 文件路径 | 定位 | 使用者 |
|-----|----------|------|--------|
| HTTP 客户端 | `api/client.py` | `APIClient` 类，基于 `httpx` | 命令行工具、外部脚本 |
| 搜索客户端封装 | `api/search_service.py` | `SearchService` 类，封装 `APIClient` | 命令行工具、外部脚本 |
| 前端 API 调用 | `frontend/src/lib/api/search.ts` | `searchApi` 对象，基于 `fetch` | Next.js 前端应用 |

---

*报告更新日期: 2026-04-27*
*修正内容: 架构图修正（客户端组件定位）、图关系查询设计权衡分析、代码位置索引扩展*
