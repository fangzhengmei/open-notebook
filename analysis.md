# Open-Notebook 混合检索方案分析报告

## 1. 概述

Open-Notebook 使用 SurrealDB 实现了一个结合向量检索、全文检索与图关系数据建模的混合检索方案。本文档详细追踪一次完整搜索请求的实现链路，分析三种检索方式的触发机制、结果合并策略，以及向量嵌入数据的写入时机与一致性保障。

---

## 2. 整体架构

### 2.1 模块层级

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Layer                                  │
│  api/routers/search.py (FastAPI 端点)                            │
│  api/search_service.py (服务层封装)                               │
├─────────────────────────────────────────────────────────────────┤
│                      Domain Layer                                 │
│  open_notebook/domain/notebook.py (核心业务逻辑)                  │
│  - text_search() 函数                                             │
│  - vector_search() 函数                                           │
├─────────────────────────────────────────────────────────────────┤
│                      Database Layer                               │
│  open_notebook/database/repository.py (数据库操作)                │
│  open_notebook/database/migrations/*.surrealql (SurrealQL 函数) │
│  - fn::text_search()                                              │
│  - fn::vector_search()                                            │
├─────────────────────────────────────────────────────────────────┤
│                      Embedding Layer                              │
│  commands/embedding_commands.py (异步嵌入命令)                    │
│  open_notebook/utils/embedding.py (嵌入生成逻辑)                  │
└─────────────────────────────────────────────────────────────────┘
```

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

### 7.5 潜在改进点

1. **图关系搜索**: 当前未使用图遍历扩展搜索结果，可考虑实现:
   - 通过共享 Notebook 查找相关 Source
   - 通过引用关系扩展搜索范围

2. **混合分数融合**: 向量检索和全文检索是独立的两种模式，可考虑实现:
   - 同时执行两种检索
   - 分数归一化后融合排序

3. **嵌入状态跟踪**: 异步嵌入缺少明确的状态反馈机制，用户无法知道嵌入是否完成

---

## 8. 关键代码位置索引

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

---

*报告生成日期: 2026-04-27*
