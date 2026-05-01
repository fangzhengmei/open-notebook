# Open Notebook 向量重建与搜索索引刷新流程分析

## 目录

1. [概述](#1-概述)
2. [触发条件](#2-触发条件)
3. [后台任务调度](#3-后台任务调度)
4. [向量写入数据库](#4-向量写入数据库)
5. [索引更新机制](#5-索引更新机制)
6. [增量更新与全量重建的路径差异](#6-增量更新与全量重建的路径差异)
7. [完整流程图](#7-完整流程图)
8. [关键代码位置汇总](#8-关键代码位置汇总)

---

## 1. 概述

Open Notebook 项目使用 **SurrealDB** 作为向量数据库，通过 **surreal-commands** 库实现后台任务调度。向量重建和搜索索引刷新是一个多层次、异步执行的流程，涉及前端 UI、API 层、命令调度层、向量生成层和数据库层。

### 核心数据模型

| 数据类型 | 存储位置 | 向量字段 |
|---------|---------|---------|
| Source (文档) | `source_embedding` 表 | 每条 chunk 一个 embedding |
| Note (笔记) | `note` 表 | `embedding` 字段 (array<float>) |
| SourceInsight (洞察) | `source_insight` 表 | `embedding` 字段 (array<float>) |

---

## 2. 触发条件

向量重建和索引刷新有两种主要触发方式：**手动触发** 和 **自动触发**。

### 2.1 手动触发

#### 前端 UI 触发

**模块位置**: `frontend/src/app/(dashboard)/advanced/components/RebuildEmbeddings.tsx`

用户可以在 Advanced 页面手动触发向量重建，可配置以下选项：

| 配置项 | 选项 | 说明 |
|-------|------|------|
| **Mode** | `existing` / `all` | 重建模式（增量/全量） |
| **Include** | Sources / Notes / Insights | 选择重建的数据类型 |

**API 调用链**:

```typescript
// frontend/src/lib/api/embedding.ts
embeddingApi.rebuildEmbeddings(request)  // POST /embeddings/rebuild
embeddingApi.getRebuildStatus(commandId)  // GET /embeddings/rebuild/{id}/status
```

#### API 直接触发

**模块位置**: `api/routers/embedding_rebuild.py`

| 端点 | 方法 | 功能 |
|-----|------|------|
| `/embeddings/rebuild` | POST | 启动重建任务 |
| `/embeddings/rebuild/{command_id}/status` | GET | 查询重建进度 |

### 2.2 自动触发

以下操作会自动触发向量嵌入任务：

| 操作 | 触发位置 | 触发命令 |
|-----|---------|---------|
| **保存 Note** | `open_notebook/domain/notebook.py:570-593` | `embed_note` |
| **Source 向量化** | `open_notebook/domain/notebook.py:411-457` | `embed_source` |
| **创建 Insight** | `commands/embedding_commands.py:443-546` | `embed_insight` |
| **Source 添加 Insight** | `open_notebook/domain/notebook.py:459-504` | `create_insight` → `embed_insight` |

**关键代码示例**:

```python
# open_notebook/domain/notebook.py:570-593 - Note.save() 自动触发
async def save(self) -> Optional[str]:
    await super().save()
    if self.id and self.content and self.content.strip():
        command_id = submit_command(
            "open_notebook",
            "embed_note",
            {"note_id": str(self.id)},
        )
        return command_id
    return None
```

---

## 3. 后台任务调度

### 3.1 调度架构

项目使用 **surreal-commands** 库实现分布式任务调度，核心组件包括：

| 组件 | 位置 | 功能 |
|-----|------|------|
| `CommandService` | `api/command_service.py` | 命令提交服务层 |
| `submit_command` | `surreal_commands` 库 | 提交任务到队列 |
| `@command` 装饰器 | `surreal_commands` 库 | 定义命令处理器 |
| `get_command_status` | `surreal_commands` 库 | 查询任务状态 |

### 3.2 任务提交流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        任务提交流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. 用户操作 / 自动触发                                           │
│         │                                                         │
│         ▼                                                         │
│  2. CommandService.submit_command_job()                          │
│     ├── 导入命令模块 (确保注册)                                    │
│     └── 调用 surreal_commands.submit_command()                   │
│         │                                                         │
│         ▼                                                         │
│  3. SurrealDB 存储命令记录 (command 表)                           │
│     ├── app: "open_notebook"                                     │
│     ├── name: "rebuild_embeddings" / "embed_source" 等          │
│     ├── status: "queued" → "running" → "completed"/"failed"    │
│     └── result: 存储执行结果和统计信息                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 命令定义

所有嵌入相关命令定义在 `commands/embedding_commands.py`：

| 命令名 | 输入模型 | 输出模型 | 功能 |
|-------|---------|---------|------|
| `rebuild_embeddings` | `RebuildEmbeddingsInput` | `RebuildEmbeddingsOutput` | 协调器命令，提交子任务 |
| `embed_source` | `EmbedSourceInput` | `EmbedSourceOutput` | 嵌入单个 Source |
| `embed_note` | `EmbedNoteInput` | `EmbedNoteOutput` | 嵌入单个 Note |
| `embed_insight` | `EmbedInsightInput` | `EmbedInsightOutput` | 嵌入单个 Insight |
| `create_insight` | `CreateInsightInput` | `CreateInsightOutput` | 创建 Insight 并嵌入 |

### 3.4 重试策略

所有 `embed_*` 命令都配置了重试策略：

```python
# commands/embedding_commands.py:121-131
@command(
    "embed_note",
    app="open_notebook",
    retry={
        "max_attempts": 5,
        "wait_strategy": "exponential_jitter",
        "wait_min": 1,
        "wait_max": 60,
        "stop_on": [ValueError, ConfigurationError],  # 永久错误不重试
        "retry_log_level": "debug",
    },
)
```

| 重试参数 | 值 | 说明 |
|---------|---|------|
| `max_attempts` | 5 | 最多重试 5 次 |
| `wait_strategy` | `exponential_jitter` | 指数退避 + 抖动 |
| `wait_min` | 1s | 最小等待时间 |
| `wait_max` | 60s | 最大等待时间 |
| `stop_on` | ValueError, ConfigurationError | 这些错误类型直接失败，不重试 |

**注意**: `rebuild_embeddings` 命令**禁用重试** (`retry=None`)，因为它只是协调器，实际的嵌入逻辑由子命令处理，子命令有自己的重试策略。

---

## 4. 向量写入数据库

### 4.1 向量生成流程

**核心模块**: `open_notebook/utils/embedding.py`

#### 4.1.1 单个文本嵌入 (`generate_embedding`)

```python
# open_notebook/utils/embedding.py:209-274
async def generate_embedding(
    text: str,
    content_type: Optional[ContentType] = None,
    file_path: Optional[str] = None,
    command_id: Optional[str] = None,
) -> List[float]:
```

**处理逻辑**:

```
┌────────────────────────────────────────────────────────────────┐
│                   generate_embedding 流程                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. 检查文本是否为空 → ValueError                              │
│         │                                                       │
│         ▼                                                       │
│  2. 计算 token 数量                                            │
│         │                                                       │
│         ▼                                                       │
│  3. 判断是否需要分块                                            │
│         ├── token_count <= CHUNK_SIZE → 直接嵌入               │
│         │                          │                            │
│         │                          ▼                            │
│         │                    generate_embeddings([text])       │
│         │                          │                            │
│         │                          ▼                            │
│         │                    返回 embeddings[0]                │
│         │                                                       │
│         └── token_count > CHUNK_SIZE → 分块处理                │
│                                    │                             │
│                                    ▼                             │
│                              chunk_text()                       │
│                                    │                             │
│                                    ▼                             │
│                              generate_embeddings(chunks)        │
│                                    │                             │
│                                    ▼                             │
│                              mean_pool_embeddings()             │
│                                    │                             │
│                                    ▼                             │
│                              返回合并后的向量                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 批量嵌入 (`generate_embeddings`)

```python
# open_notebook/utils/embedding.py:111-206
async def generate_embeddings(
    texts: List[str], 
    command_id: Optional[str] = None
) -> List[List[float]]:
```

**处理逻辑**:

| 步骤 | 说明 |
|-----|------|
| 1 | 检查嵌入模型是否配置 |
| 2 | 计算批次数 = `ceil(len(texts) / EMBEDDING_BATCH_SIZE)` |
| 3 | 逐批调用 `embedding_model.aembed(batch)` |
| 4 | 每批最多重试 `EMBEDDING_MAX_RETRIES` (3次) |
| 5 | 重试间隔 `EMBEDDING_RETRY_DELAY` (2秒) |

**配置参数**:

| 参数 | 环境变量 | 默认值 | 说明 |
|-----|---------|-------|------|
| `EMBEDDING_BATCH_SIZE` | `OPEN_NOTEBOOK_EMBEDDING_BATCH_SIZE` | 50 | 每批处理的文本数量 |
| `EMBEDDING_MAX_RETRIES` | - | 3 | 最大重试次数 |
| `EMBEDDING_RETRY_DELAY` | - | 2s | 重试等待时间 |

#### 4.1.3 均值池化 (`mean_pool_embeddings`)

当文本超过 chunk size 时，需要将多个 chunk 的 embedding 合并为一个：

```python
# open_notebook/utils/embedding.py:55-108
async def mean_pool_embeddings(embeddings: List[List[float]]) -> List[float]:
```

**算法**:

1. **归一化**: 每个 embedding 归一化到单位长度
2. **求均值**: 计算所有 embedding 的元素级均值
3. **再次归一化**: 确保结果也是单位长度

```python
# 伪代码
arr = np.array(embeddings, dtype=np.float64)
norms = np.linalg.norm(arr, axis=1, keepdims=True)
normalized = arr / norms  # 归一化
mean = np.mean(normalized, axis=0)  # 求均值
mean = mean / np.linalg.norm(mean)  # 再次归一化
```

### 4.2 分块策略

**核心模块**: `open_notebook/utils/chunking.py`

#### 4.2.1 内容类型检测

```python
# open_notebook/utils/chunking.py
def detect_content_type(text: str, file_path: Optional[str] = None) -> ContentType:
```

| ContentType | 说明 | 分块策略 |
|------------|------|---------|
| `MARKDOWN` | Markdown 文档 | 按标题分块 |
| `CODE` | 代码文件 | 按函数/类分块 |
| `PLAIN_TEXT` | 普通文本 | 按段落分块 |
| `TRANSCRIPT` | 转录文本 | 特殊处理 |

#### 4.2.2 分块参数

| 参数 | 值 | 说明 |
|-----|---|------|
| `CHUNK_SIZE` | 8191 tokens | 最大 chunk 大小 |
| `CHUNK_OVERLAP` | 20 tokens | chunk 重叠大小 |

### 4.3 不同数据类型的写入逻辑

#### 4.3.1 Source 嵌入 (`embed_source_command`)

**位置**: `commands/embedding_commands.py:307-440`

**处理流程**:

```
┌──────────────────────────────────────────────────────────────┐
│                    embed_source_command 流程                    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. 加载 Source 记录                                          │
│         │                                                      │
│         ▼                                                      │
│  2. 检查 full_text 是否为空 → ValueError                      │
│         │                                                      │
│         ▼                                                      │
│  3. DELETE 现有 source_embedding 记录 (幂等性)                 │
│         │                                                      │
│         ▼                                                      │
│  4. 检测内容类型 (detect_content_type)                         │
│         │                                                      │
│         ▼                                                      │
│  5. 分块 (chunk_text)                                          │
│         │                                                      │
│         ▼                                                      │
│  6. 批量生成 embedding (generate_embeddings)                   │
│         │                                                      │
│         ▼                                                      │
│  7. 批量 INSERT source_embedding 记录                          │
│         │                                                      │
│         ▼                                                      │
│  8. 返回 EmbedSourceOutput (包含 chunks_created)               │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**数据库操作**:

```python
# 删除现有嵌入
await repo_query(
    "DELETE source_embedding WHERE source = $source_id",
    {"source_id": ensure_record_id(input_data.source_id)},
)

# 批量插入
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

#### 4.3.2 Note 嵌入 (`embed_note_command`)

**位置**: `commands/embedding_commands.py:121-210`

**处理流程**:

```
┌──────────────────────────────────────────────────────────────┐
│                     embed_note_command 流程                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. 加载 Note 记录                                            │
│         │                                                      │
│         ▼                                                      │
│  2. 检查 content 是否为空 → ValueError                         │
│         │                                                      │
│         ▼                                                      │
│  3. 生成 embedding (generate_embedding)                        │
│         │                                                      │
│         ▼                                                      │
│  4. UPDATE note SET embedding = $embedding                     │
│         │                                                      │
│         ▼                                                      │
│  5. 返回 EmbedNoteOutput                                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**数据库操作**:

```python
await repo_query(
    "UPDATE $note_id SET embedding = $embedding",
    {
        "note_id": ensure_record_id(input_data.note_id),
        "embedding": embedding,
    },
)
```

#### 4.3.3 Insight 嵌入 (`embed_insight_command`)

**位置**: `commands/embedding_commands.py:213-304`

**处理流程与 Note 基本相同**，只是操作的是 `source_insight` 表。

---

## 5. 索引更新机制

### 5.1 数据库索引定义

**位置**: `open_notebook/database/migrations/`

#### 5.1.1 迁移文件 10 (`10.surrealql`)

```surrealql
-- 为 source_insight 和 source_embedding 的 source 字段添加索引
DEFINE INDEX IF NOT EXISTS idx_source_insight_source ON source_insight FIELDS source CONCURRENTLY;
DEFINE INDEX IF NOT EXISTS idx_source_embedding_source ON source_embedding FIELDS source CONCURRENTLY;

-- 定义 embedding 字段类型
DEFINE FIELD OVERWRITE embedding ON TABLE source_insight TYPE option<array<float>>;
DEFINE FIELD OVERWRITE embedding ON TABLE note TYPE option<array<float>>;
```

#### 5.1.2 索引说明

| 索引名 | 表 | 字段 | 用途 |
|-------|---|------|------|
| `idx_source_insight_source` | `source_insight` | `source` | 加速按 Source 查询 Insight |
| `idx_source_embedding_source` | `source_embedding` | `source` | 加速按 Source 查询嵌入 chunks |

### 5.2 向量搜索函数

**位置**: `open_notebook/database/migrations/4.surrealql` 和 `9.surrealql`

#### 5.2.1 `fn::vector_search` 定义

```surrealql
DEFINE FUNCTION IF NOT EXISTS fn::vector_search(
    $query: array<float>, 
    $match_count: int, 
    $sources: bool, 
    $show_notes: bool, 
    $min_similarity: float
) {
    -- 1. 搜索 source_embedding (Source 分块)
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

    -- 2. 搜索 source_insight
    let $source_insight_search = ...;

    -- 3. 搜索 note
    let $note_content_search = ...;

    -- 4. 合并结果，按 similarity 排序
    let $all_results = array::union(
        array::union($source_embedding_search, $source_insight_search),
        $note_content_search
    );

    RETURN (
        select id, parent_id, title, 
               math::max(similarity) as similarity,
               array::flatten(content) as matches
        from $all_results 
        where id is not None
        group by id, parent_id, title 
        ORDER BY similarity DESC 
        LIMIT $match_count
    );
};
```

### 5.3 搜索时的索引使用

**位置**: `open_notebook/domain/notebook.py:650-680`

```python
async def vector_search(
    keyword: str,
    results: int,
    source: bool = True,
    note: bool = True,
    minimum_score=0.2,
):
    # 1. 生成查询向量
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

### 5.4 索引更新时机

**重要说明**: SurrealDB 的 `vector::similarity::cosine` 是**运行时计算**的，不是预构建的向量索引。

| 操作 | 索引影响 |
|-----|---------|
| INSERT embedding | 新记录立即可用于相似度计算 |
| UPDATE embedding | 更新后立即可用于相似度计算 |
| DELETE embedding | 记录从搜索池中移除 |

**当前实现的特点**:
- 没有使用专门的向量索引（如 HNSW、IVF 等）
- 每次搜索都遍历所有符合条件的记录计算余弦相似度
- 适用于中小规模数据，大规模数据可能需要优化

---

## 6. 增量更新与全量重建的路径差异

### 6.1 两种模式的定义

| 模式 | 参数值 | 语义 |
|-----|-------|------|
| **增量更新** | `mode="existing"` | 只重建**已有 embedding** 的记录 |
| **全量重建** | `mode="all"` | 重建**所有有内容**的记录 |

### 6.2 数据收集逻辑差异

**位置**: `commands/embedding_commands.py:549-619` (`collect_items_for_rebuild`)

#### 6.2.1 Source 收集差异

```python
# 增量更新 (mode="existing")
if mode == "existing":
    result = await repo_query(
        """
        RETURN array::distinct(
            SELECT VALUE source.id
            FROM source_embedding
            WHERE embedding != none AND array::len(embedding) > 0
        )
        """
    )

# 全量重建 (mode="all")
else:
    result = await repo_query(
        "SELECT id FROM source WHERE full_text != none AND string::trim(full_text) != ''"
    )
```

#### 6.2.2 Note 收集差异

```python
# 增量更新
if mode == "existing":
    result = await repo_query(
        "SELECT id FROM note WHERE embedding != none AND array::len(embedding) > 0"
    )

# 全量重建
else:
    result = await repo_query(
        "SELECT id FROM note WHERE content != none AND string::trim(content) != ''"
    )
```

#### 6.2.3 Insight 收集差异

```python
# 增量更新
if mode == "existing":
    result = await repo_query(
        "SELECT id FROM source_insight WHERE embedding != none AND array::len(embedding) > 0"
    )

# 全量重建
else:
    result = await repo_query(
        "SELECT id FROM source_insight WHERE content != none AND string::trim(content) != ''"
    )
```

### 6.3 对比总结

| 维度 | 增量更新 (`existing`) | 全量重建 (`all`) |
|-----|---------------------|------------------|
| **目标数据** | 已有 embedding 的记录 | 所有有内容的记录 |
| **Source 查询条件** | `source_embedding` 中存在有效 embedding | `source.full_text` 非空 |
| **Note 查询条件** | `note.embedding` 非空且长度 > 0 | `note.content` 非空 |
| **Insight 查询条件** | `source_insight.embedding` 非空 | `source_insight.content` 非空 |
| **使用场景** | 切换嵌入模型后刷新、修复损坏向量 | 首次导入数据、新增内容从未嵌入 |
| **数据范围** | 较小（仅已有向量） | 较大（所有有效内容） |

### 6.4 重建协调器流程

**位置**: `commands/embedding_commands.py:622-787` (`rebuild_embeddings_command`)

```
┌────────────────────────────────────────────────────────────────┐
│                 rebuild_embeddings_command 流程                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. 检查嵌入模型是否配置 → 无配置则失败                          │
│         │                                                       │
│         ▼                                                       │
│  2. collect_items_for_rebuild(mode, include_*)                 │
│         │                                                       │
│         ├── 增量更新: 收集已有 embedding 的记录 ID               │
│         └── 全量重建: 收集所有有内容的记录 ID                    │
│         │                                                       │
│         ▼                                                       │
│  3. 遍历所有 ID，提交子命令                                      │
│         │                                                       │
│         ├── Source → submit_command("embed_source")            │
│         ├── Note → submit_command("embed_note")                │
│         └── Insight → submit_command("embed_insight")          │
│         │                                                       │
│         ▼                                                       │
│  4. 返回统计信息                                                 │
│     ├── total_items: 总项目数                                   │
│     ├── jobs_submitted: 成功提交的任务数                        │
│     ├── failed_submissions: 提交失败的任务数                    │
│     └── 按类型统计: sources_submitted, notes_submitted, etc.   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**重要**: `rebuild_embeddings` 只是**协调器**，它不实际执行嵌入操作，而是：
1. 收集需要处理的 ID 列表
2. 为每个 ID 提交一个 `embed_*` 子命令
3. 立即返回，不等待子命令完成

子命令由 `surreal-commands` 调度器异步执行，每个子命令有自己的重试策略。

---

## 7. 完整流程图

### 7.1 手动触发重建流程

```
┌──────────┐     ┌──────────────┐     ┌────────────────────┐
│ 前端 UI  │────▶│  API Router  │────▶│  CommandService    │
│          │     │              │     │                    │
│Rebuild-  │     │embedding_    │     │ submit_command_job │
│Embeddings│     │rebuild.py    │     │                    │
└──────────┘     └──────────────┘     └────────────────────┘
                                                  │
                                                  ▼
┌──────────────────────────────────────────────────────────────┐
│                    surreal_commands 调度器                     │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. 创建 command 记录 (status=queued)                   │  │
│  │  2. 调度器拾取任务 (status=running)                      │  │
│  │  3. 执行 rebuild_embeddings_command                      │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
┌──────────────────────────────────────────────────────────────┐
│              rebuild_embeddings_command (协调器)               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. 检查嵌入模型配置                                       │  │
│  │  2. collect_items_for_rebuild()                          │  │
│  │     ├── 增量: 已有 embedding 的记录                       │  │
│  │     └── 全量: 所有有内容的记录                            │  │
│  │  3. 为每个 ID 提交子命令                                   │  │
│  │     ├── embed_source                                      │  │
│  │     ├── embed_note                                        │  │
│  │     └── embed_insight                                     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                                                  │
                    ┌─────────────────────────────┼─────────────────────────────┐
                    ▼                             ▼                             ▼
           ┌──────────────┐             ┌──────────────┐             ┌──────────────┐
           │ embed_source │             │  embed_note  │             │embed_insight │
           │  命令        │             │   命令        │             │   命令        │
           └──────────────┘             └──────────────┘             └──────────────┘
                    │                             │                             │
                    ▼                             ▼                             ▼
           ┌─────────────────────────────────────────────────────────────────────┐
           │                    generate_embedding / generate_embeddings          │
           │  ┌──────────────────────────────────────────────────────────────┐  │
           │  │  1. 检测是否需要分块 (> CHUNK_SIZE tokens)                    │  │
           │  │  2. 需要分块: chunk_text() → generate_embeddings() → 均值池化  │  │
           │  │  3. 不需要分块: 直接 generate_embeddings([text])               │  │
           │  └──────────────────────────────────────────────────────────────┘  │
           └─────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
           ┌─────────────────────────────────────────────────────────────────────┐
           │                         数据库写入                                    │
           │  ┌──────────────────────────────────────────────────────────────┐  │
           │  │  Source:  DELETE + INSERT source_embedding (分块存储)         │  │
           │  │  Note:    UPDATE note SET embedding = $embedding              │  │
           │  │  Insight: UPDATE source_insight SET embedding = $embedding    │  │
           │  └──────────────────────────────────────────────────────────────┘  │
           └─────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
           ┌─────────────────────────────────────────────────────────────────────┐
           │                         搜索时索引使用                                │
           │  ┌──────────────────────────────────────────────────────────────┐  │
           │  │  vector_search() → fn::vector_search()                        │  │
           │  │  使用 vector::similarity::cosine() 计算余弦相似度              │  │
           │  │  按相似度排序返回结果                                            │  │
           │  └──────────────────────────────────────────────────────────────┘  │
           └─────────────────────────────────────────────────────────────────────┘
```

### 7.2 自动触发嵌入流程 (以 Note.save() 为例)

```
┌────────────────────────────────────────────────────────────────┐
│                      Note.save() 自动触发                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  User saves Note in frontend                                   │
│         │                                                       │
│         ▼                                                       │
│  API: POST /notes 或 PUT /notes/{id}                           │
│         │                                                       │
│         ▼                                                       │
│  Note.save()                                                    │
│  ├── 1. await super().save()  # 保存到数据库                    │
│  └── 2. 有内容?                                                 │
│       ├── 是 → submit_command("embed_note", {"note_id": id})  │
│       └── 否 → 不提交嵌入任务                                    │
│         │                                                       │
│         ▼                                                       │
│  surreal_commands 调度器异步执行 embed_note_command             │
│         │                                                       │
│         ▼                                                       │
│  生成 embedding → UPDATE note SET embedding                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 8. 状态查询与进度追踪

### 8.1 前端轮询机制

**位置**: `frontend/src/app/(dashboard)/advanced/components/RebuildEmbeddings.tsx`

#### 8.1.1 轮询触发与停止

```typescript
// 启动轮询
const startPolling = (cmdId: string) => {
  const interval = setInterval(async () => {
    const statusData = await embeddingApi.getRebuildStatus(cmdId)
    setStatus(statusData)

    // 任务完成或失败时停止轮询
    if (statusData.status === 'completed' || statusData.status === 'failed') {
      stopPolling()
    }
  }, 5000)  // 每 5 秒轮询一次

  setPollingInterval(interval)
}

// 停止轮询
const stopPolling = useCallback(() => {
  if (pollingInterval) {
    clearInterval(pollingInterval)
    setPollingInterval(null)
  }
}, [pollingInterval])
```

#### 8.1.2 轮询配置

| 配置项 | 值 | 说明 |
|-------|---|------|
| 轮询间隔 | 5000ms (5秒) | 状态查询频率 |
| 停止条件 | `status === 'completed'` 或 `status === 'failed'` | 任务结束后停止 |
| 组件卸载 | `useEffect` cleanup | 防止内存泄漏 |

### 8.2 API 状态查询接口

**位置**: `api/routers/embedding_rebuild.py:123-192`

#### 8.2.1 接口定义

```python
@router.get("/rebuild/{command_id}/status", response_model=RebuildStatusResponse)
async def get_rebuild_status(command_id: str):
    """
    Get the status of a rebuild operation.
    
    Returns:
    - **status**: queued, running, completed, failed
    - **progress**: processed count, total count, percentage
    - **stats**: breakdown by type (sources, notes, insights, failed)
    - **timestamps**: started_at, completed_at
    """
```

#### 8.2.2 状态数据结构

```python
# api/models.py:248-256
class RebuildStatusResponse(BaseModel):
    command_id: str                    # 命令 ID
    status: str                        # 状态: queued, running, completed, failed
    progress: Optional[RebuildProgress] = None  # 进度信息
    stats: Optional[RebuildStats] = None        # 统计信息
    started_at: Optional[str] = None   # 开始时间
    completed_at: Optional[str] = None # 完成时间
    error_message: Optional[str] = None # 错误信息

class RebuildProgress(BaseModel):
    processed: int       # 已处理数
    total: int           # 总数
    percentage: float    # 百分比

class RebuildStats(BaseModel):
    sources: int = 0     # Sources 数
    notes: int = 0       # Notes 数
    insights: int = 0    # Insights 数
    failed: int = 0      # 失败数
```

#### 8.2.3 状态查询流程

```
┌────────────────────────────────────────────────────────────────────────┐
│                        状态查询完整流程                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  前端 (每 5 秒轮询)                                                      │
│       │                                                                 │
│       ▼                                                                 │
│  GET /embeddings/rebuild/{command_id}/status                           │
│       │                                                                 │
│       ▼                                                                 │
│  api/routers/embedding_rebuild.py:get_rebuild_status()                 │
│       │                                                                 │
│       ├── 1. 调用 surreal_commands.get_command_status(command_id)       │
│       │         │                                                        │
│       │         ▼                                                        │
│       │    从 SurrealDB 的 command 表查询记录                            │
│       │    返回: status, result, created, updated, error_message        │
│       │                                                                 │
│       ├── 2. 构建响应对象                                                │
│       │    ├── response.status = status.status                          │
│       │    ├── response.progress = 从 result 提取                       │
│       │    ├── response.stats = 从 result 提取                          │
│       │    ├── response.started_at = status.created                     │
│       │    ├── response.completed_at = status.updated                   │
│       │    └── response.error_message = result.error_message (if failed)│
│       │                                                                 │
│       └── 3. 返回 RebuildStatusResponse                                  │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### 8.3 父任务与子任务的状态关系

#### 8.3.1 任务层级结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        任务层级结构                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  父任务: rebuild_embeddings (command_id = "parent_123")             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  status: queued → running → completed/failed                  │  │
│  │  result: {                                                     │  │
│  │    total_items: 100,                                           │  │
│  │    jobs_submitted: 100,    ← 已提交的子任务数                  │  │
│  │    failed_submissions: 0,   ← 提交失败的数量                   │  │
│  │    sources_submitted: 50,                                       │  │
│  │    notes_submitted: 30,                                         │  │
│  │    insights_submitted: 20                                       │  │
│  │  }                                                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                        │
│                              ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  子任务 (独立的 command_id)                                    │  │
│  │                                                               │  │
│  │  embed_source:1  │ embed_source:2  │ ... │ embed_source:50   │  │
│  │  embed_note:1    │ embed_note:2    │ ... │ embed_note:30     │  │
│  │  embed_insight:1 │ embed_insight:2 │ ... │ embed_insight:20  │  │
│  │                                                               │  │
│  │  每个子任务都有自己的:                                          │  │
│  │  ├── status: queued/running/completed/failed                 │  │
│  │  ├── result: {success, error_message, processing_time, ...} │  │
│  │  └── created/updated 时间戳                                   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ⚠️ 重要: 父任务不追踪子任务的执行状态!                               │
│     - 父任务 completed ≠ 所有子任务 completed                        │
│     - 子任务的失败信息不会自动汇总到父任务                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.3.2 父任务的状态流转

**位置**: `commands/embedding_commands.py:622-787`

```
┌─────────────────────────────────────────────────────────────────────┐
│                 rebuild_embeddings 父任务状态流转                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  初始状态: queued                                                    │
│       │                                                              │
│       ▼ (调度器拾取任务)                                              │
│  running                                                             │
│       │                                                              │
│       ├── 1. 检查嵌入模型配置                                        │
│       ├── 2. collect_items_for_rebuild()                            │
│       ├── 3. 遍历所有 ID，提交子命令                                   │
│       │    ├── submit_command("embed_source")                       │
│       │    ├── submit_command("embed_note")                         │
│       │    └── submit_command("embed_insight")                      │
│       │                                                              │
│       ├── 4. 统计提交结果                                            │
│       │    ├── jobs_submitted: 成功提交的数量                        │
│       │    └── failed_submissions: 提交失败的数量                     │
│       │                                                              │
│       └── 5. 返回 RebuildEmbeddingsOutput                           │
│            (此时父任务状态变为 completed)                              │
│                                                                      │
│       ▼                                                              │
│  completed / failed                                                  │
│                                                                      │
│  注意: 父任务 completed 只表示"所有子任务都已提交"，                   │
│       不表示"所有子任务都已执行完成"!                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 8.3.3 进度计算逻辑

**位置**: `api/routers/embedding_rebuild.py:151-159`

```python
# 从父任务的 result 中提取进度信息
if "total_items" in result and "jobs_submitted" in result:
    total = result["total_items"]
    submitted = result["jobs_submitted"]
    response.progress = RebuildProgress(
        processed=submitted,      # 已提交的任务数
        total=total,              # 总任务数
        percentage=round((submitted / total * 100) if total > 0 else 0, 2),
    )
```

**关键发现**:

| 进度字段 | 实际含义 | 潜在问题 |
|---------|---------|---------|
| `processed` | `jobs_submitted` (已提交的任务数) | 不是已完成的任务数 |
| `percentage` | `jobs_submitted / total_items * 100` | 可能误导用户 |

**示例场景**:
- 父任务提交了 100 个子任务
- 父任务状态变为 `completed`，进度显示 100%
- 但实际上可能只有 10 个子任务真正执行完成
- 用户看到进度 100% 以为全部完成，但实际还有 90 个子任务在执行中

### 8.4 子任务的独立状态追踪

#### 8.4.1 子任务状态管理

**位置**: `open_notebook/domain/notebook.py:318-359`

Source 对象可以追踪与其关联的命令状态:

```python
# 获取状态
async def get_status(self) -> Optional[str]:
    if not self.command:
        return None
    try:
        from surreal_commands import get_command_status
        status = await get_command_status(str(self.command))
        return status.status if status else "unknown"
    except Exception as e:
        logger.warning(f"Failed to get command status for {self.command}: {e}")
        return "unknown"

# 获取详细进度
async def get_processing_progress(self) -> Optional[Dict[str, Any]]:
    if not self.command:
        return None
    try:
        from surreal_commands import get_command_status
        status_result = await get_command_status(str(self.command))
        if not status_result:
            return None
        
        result = getattr(status_result, "result", None)
        return {
            "status": status_result.status,
            "started_at": execution_metadata.get("started_at"),
            "completed_at": execution_metadata.get("completed_at"),
            "error": getattr(status_result, "error_message", None),
            "result": result,
        }
    except Exception as e:
        logger.warning(f"Failed to get command progress for {self.command}: {e}")
        return None
```

#### 8.4.2 子任务与父任务的关联问题

| 问题 | 说明 |
|-----|------|
| **没有关联** | 父任务不知道自己提交了哪些子任务的 command_id |
| **无法追踪** | 无法通过父任务查询所有子任务的状态 |
| **没有汇总** | 子任务的成功/失败不会影响父任务的状态 |
| **进度不准确** | 父任务的进度基于"已提交"而非"已完成" |

### 8.5 失败信息的收集与展示

#### 8.5.1 错误层级分类

```
┌────────────────────────────────────────────────────────────────────────┐
│                        错误层级分类                                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  层级 1: 父任务级别错误 (fatal)                                          │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  触发位置: rebuild_embeddings_command                               │ │
│  │  错误类型:                                                          │ │
│  │  ├── 嵌入模型未配置                                                  │ │
│  │  ├── 数据库查询失败 (收集项目时)                                      │ │
│  │  └── 其他系统级错误                                                  │ │
│  │                                                                     │ │
│  │  结果: 父任务 status = "failed"                                      │ │
│  │       error_message 包含具体错误信息                                  │ │
│  │       不会提交任何子任务                                              │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  层级 2: 子任务提交失败 (submission error)                              │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  触发位置: rebuild_embeddings_command 中的 submit_command() 调用    │ │
│  │  错误类型:                                                          │ │
│  │  ├── surreal_commands 注册问题                                      │ │
│  │  ├── 数据库写入失败                                                  │ │
│  │  └── 其他提交时异常                                                  │ │
│  │                                                                     │ │
│  │  结果: 统计到 failed_submissions                                     │ │
│  │       父任务继续执行，提交其他子任务                                   │ │
│  │       父任务最终 status 仍为 "completed"                             │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  层级 3: 子任务执行失败 (execution error)                               │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  触发位置: embed_source/note/insight_command 执行过程中              │ │
│  │  错误类型:                                                          │ │
│  │  ├── 永久错误 (ValueError, ConfigurationError)                      │ │
│  │  │   ├── 记录不存在                                                  │ │
│  │  │   ├── 内容为空                                                    │ │
│  │  │   └── 配置错误                                                    │ │
│  │  │   结果: 立即返回 success=False + error_message                    │ │
│  │  │         不重试                                                    │ │
│  │  │                                                                   │ │
│  │  └── 瞬时错误 (其他 Exception)                                       │ │
│  │      ├── 网络超时                                                    │ │
│  │      ├── API 限流                                                    │ │
│  │      └── 数据库冲突                                                  │ │
│  │      结果: 由 surreal-commands 重试最多 5 次                         │ │
│  │            最终失败后记录到 command 表                                │ │
│  │                                                                     │ │
│  │  ⚠️ 关键: 这些失败信息**不会**自动汇总到父任务!                       │ │
│  │     父任务不知道子任务是否执行成功                                    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

#### 8.5.2 父任务级错误处理

**位置**: `commands/embedding_commands.py:775-787`

```python
except Exception as e:
    processing_time = time.time() - start_time
    logger.error(f"Rebuild embeddings failed: {e}")
    logger.exception(e)
    
    return RebuildEmbeddingsOutput(
        success=False,
        total_items=0,
        jobs_submitted=0,
        failed_submissions=0,
        processing_time=processing_time,
        error_message=str(e),  # 错误信息会存储到 result 中
    )
```

#### 8.5.3 子任务级错误处理 (永久错误)

**位置**: `commands/embedding_commands.py:190-202` (以 embed_note 为例)

```python
except ValueError as e:
    # 永久失败 - 不重试
    processing_time = time.time() - start_time
    cmd_id = get_command_id(input_data)
    logger.error(
        f"Failed to embed note {input_data.note_id} (command: {cmd_id}): {e}"
    )
    return EmbedNoteOutput(
        success=False,
        note_id=input_data.note_id,
        processing_time=processing_time,
        error_message=str(e),  # 错误信息存储到子任务的 result 中
    )
```

#### 8.5.4 子任务级错误处理 (瞬时错误)

**位置**: `commands/embedding_commands.py:203-210`

```python
except Exception as e:
    # 瞬时失败 - 会被重试 (surreal-commands 记录最终失败)
    cmd_id = get_command_id(input_data)
    logger.debug(
        f"Transient error embedding note {input_data.note_id} "
        f"(command: {cmd_id}): {e}"
    )
    raise  # 抛出异常让 surreal-commands 处理重试
```

### 8.6 前端展示的状态与错误信号

#### 8.6.1 状态图标与颜色

**位置**: `frontend/src/app/(dashboard)/advanced/components/RebuildEmbeddings.tsx:229-232`

```typescript
{status.status === 'queued' && <Clock className="h-5 w-5 text-yellow-500" />}
{status.status === 'running' && <Loader2 className="h-5 w-5 text-blue-500 animate-spin" />}
{status.status === 'completed' && <CheckCircle2 className="h-5 w-5 text-green-500" />}
{status.status === 'failed' && <XCircle className="h-5 w-5 text-red-500" />}
```

#### 8.6.2 状态展示汇总

| 状态 | 图标 | 颜色 | 说明 |
|-----|------|------|------|
| `queued` | Clock | 黄色 | 任务已排队，等待执行 |
| `running` | Loader2 (旋转) | 蓝色 | 任务正在执行中 |
| `completed` | CheckCircle2 | 绿色 | 父任务已完成 (子任务可能还在执行) |
| `failed` | XCircle | 红色 | 父任务执行失败 |

#### 8.6.3 进度条展示

**位置**: `frontend/src/app/(dashboard)/advanced/components/RebuildEmbeddings.tsx:254-271`

```typescript
{progressData && (
  <div className="space-y-2">
    <div className="flex justify-between text-sm">
      <span>{t('common.progress')}</span>
      <span className="font-medium">
        {t('advanced.rebuild.itemsProcessed')
          .replace('{processed}', processedItems.toString())
          .replace('{total}', totalItems.toString())
          .replace('{percent}', progressPercent.toFixed(1))}
      </span>
    </div>
    <Progress value={progressPercent} className="h-2" />
    {failedItems > 0 && (
      <p className="text-sm text-yellow-600">
        ⚠️ {t('advanced.rebuild.failedItems').replace('{count}', failedItems.toString())}
      </p>
    )}
  </div>
)}
```

#### 8.6.4 统计面板

**位置**: `frontend/src/app/(dashboard)/advanced/components/RebuildEmbeddings.tsx:274-295`

```typescript
{stats && (
  <div className="grid grid-cols-4 gap-4">
    <div className="space-y-1">
      <p className="text-sm text-muted-foreground">{t('navigation.sources')}</p>
      <p className="text-2xl font-bold">{sourcesProcessed}</p>
    </div>
    <div className="space-y-1">
      <p className="text-sm text-muted-foreground">{t('common.notes')}</p>
      <p className="text-2xl font-bold">{notesProcessed}</p>
    </div>
    <div className="space-y-1">
      <p className="text-sm text-muted-foreground">{t('common.insights')}</p>
      <p className="text-2xl font-bold">{insightsProcessed}</p>
    </div>
    <div className="space-y-1">
      <p className="text-sm text-muted-foreground">{t('advanced.rebuild.time')}</p>
      <p className="text-2xl font-bold">
        {processingTimeSeconds !== undefined ? `${processingTimeSeconds.toFixed(1)}s` : '—'}
      </p>
    </div>
  </div>
)}
```

#### 8.6.5 错误信息展示

**位置**: `frontend/src/app/(dashboard)/advanced/components/RebuildEmbeddings.tsx:297-302`

```typescript
{status.error_message && (
  <Alert variant="destructive">
    <AlertCircle className="h-4 w-4" />
    <AlertDescription>{status.error_message}</AlertDescription>
  </Alert>
)}
```

### 8.7 用户能看到的完整状态信号

#### 8.7.1 信号汇总表

| 信号类型 | 展示位置 | 数据来源 | 含义 |
|---------|---------|---------|------|
| **状态图标** | 顶部状态栏 | `status.status` | 父任务的执行状态 |
| **状态文本** | 图标旁边 | `status.status` | queued/running/completed/failed |
| **进度条** | 进度区域 | `jobs_submitted / total_items` | 已提交任务的比例 |
| **进度文本** | 进度条上方 | `processed / total (percentage)` | 具体数值 |
| **警告提示** | 进度条下方 | `failedItems > 0` | 有提交失败的任务 |
| **统计面板** | 进度下方 | `stats` 对象 | 各类型已提交数量、耗时 |
| **错误提示** | 统计面板下方 | `status.error_message` | 父任务级别的错误 |
| **时间戳** | 最底部 | `started_at`, `completed_at` | 开始和完成时间 |

#### 8.7.2 不同场景下的用户体验

**场景 1: 父任务执行中**
- 状态: 蓝色旋转图标 + "running"
- 进度条: 逐渐增加 (基于已提交的任务数)
- 用户感知: 知道任务正在进行

**场景 2: 父任务完成，但子任务还在执行**
- 状态: 绿色勾选图标 + "completed"
- 进度条: 100%
- ⚠️ 用户感知: 以为全部完成，但实际子任务可能还在执行

**场景 3: 父任务失败**
- 状态: 红色叉号图标 + "failed"
- 错误信息: 显示 `error_message`
- 用户感知: 知道任务失败，可以看到具体原因

**场景 4: 有提交失败的任务**
- 状态: 绿色勾选 (父任务成功)
- 警告提示: 黄色 "⚠️ X 个项目失败"
- 用户感知: 知道部分任务提交失败，但不知道具体是哪些

### 8.8 当前设计的局限性

#### 8.8.1 问题清单

| 问题 | 影响 | 严重程度 |
|-----|------|---------|
| 进度基于"已提交"而非"已完成" | 用户可能在子任务还在执行时就以为完成了 | 高 |
| 父任务不追踪子任务状态 | 无法知道有多少子任务实际成功/失败 | 高 |
| 子任务错误不汇总 | 子任务执行失败用户看不到 | 高 |
| `failed_submissions` 含义模糊 | 用户以为是执行失败，实际是提交失败 | 中 |
| 没有重试状态展示 | 用户不知道哪些任务在重试 | 低 |

#### 8.8.2 潜在的改进方向

```
┌────────────────────────────────────────────────────────────────────────┐
│                        潜在改进方向                                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 父任务追踪子任务                                                     │
│     ├── 存储所有子任务的 command_id 列表                                │
│     ├── 轮询时查询所有子任务的状态                                       │
│     └── 计算真正的完成进度 (已完成子任务数 / 总子任务数)                 │
│                                                                         │
│  2. 子任务状态汇总                                                       │
│     ├── 统计各状态的子任务数量 (queued/running/completed/failed)        │
│     ├── 汇总所有子任务的错误信息                                         │
│     └── 展示给用户真正的执行状态                                         │
│                                                                         │
│  3. 进度计算改进                                                         │
│     ├── 区分"提交进度"和"执行进度"                                      │
│     ├── 或者只展示"执行进度"                                             │
│     └── 添加更详细的进度信息                                             │
│                                                                         │
│  4. 错误信息改进                                                         │
│     ├── 区分"提交失败"和"执行失败"                                      │
│     ├── 展示失败的具体子任务 ID 和类型                                   │
│     └── 提供重试按钮让用户可以重试失败的任务                              │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### 8.9 漏点分析: 启动阶段预估总量与实际重建收集总量的口径差异

#### 8.9.1 问题概述

向量重建流程存在**两个阶段的总量统计**，但两者的**查询口径不一致**，导致用户看到的预估总量与实际处理数量存在差异。

| 阶段 | 触发时机 | 位置 | 用途 |
|-----|---------|------|------|
| **启动阶段预估** | 用户点击"开始重建"时 | `api/routers/embedding_rebuild.py:start_rebuild` | 立即返回给用户的预估数量 |
| **实际重建收集** | 父任务 `rebuild_embeddings_command` 执行时 | `commands/embedding_commands.py:collect_items_for_rebuild` | 实际提交子任务的数量 |

#### 8.9.2 代码级对比分析

##### Source 统计对比

**启动阶段 (`embedding_rebuild.py:40-61`)**:

```python
if request.include_sources:
    if request.mode == "existing":
        # 从 source_embedding 表统计有有效 embedding 的 Source
        result = await repo_query(
            """
            SELECT VALUE count(array::distinct(
                SELECT VALUE source.id
                FROM source_embedding
                WHERE embedding != none AND array::len(embedding) > 0
            )) as count FROM {}
            """
        )
    else:  # mode == "all"
        # ⚠️ 只检查 full_text != none
        result = await repo_query(
            "SELECT VALUE count() as count FROM source WHERE full_text != none GROUP ALL"
        )
```

**实际重建阶段 (`embedding_commands.py:563-587`)**:

```python
if include_sources:
    if mode == "existing":
        # 与启动阶段一致
        result = await repo_query(
            """
            RETURN array::distinct(
                SELECT VALUE source.id
                FROM source_embedding
                WHERE embedding != none AND array::len(embedding) > 0
            )
            """
        )
    else:  # mode == "all"
        # ⚠️ 多了 string::trim(full_text) != '' 条件!
        result = await repo_query(
            "SELECT id FROM source WHERE full_text != none AND string::trim(full_text) != ''"
        )
```

**差异分析**:

| 模式 | 启动阶段 | 实际重建阶段 | 差异 |
|-----|---------|-------------|------|
| `existing` | `source_embedding.embedding` 非空且长度>0 | 相同 | 无差异 |
| `all` | `source.full_text != none` | `full_text != none AND string::trim(full_text) != ''` | **实际阶段排除空字符串** |

##### Note 统计对比

**启动阶段 (`embedding_rebuild.py:63-76`)**:

```python
if request.include_notes:
    if request.mode == "existing":
        result = await repo_query(
            "SELECT VALUE count() as count FROM note WHERE embedding != none AND array::len(embedding) > 0 GROUP ALL"
        )
    else:  # mode == "all"
        # ⚠️ 只检查 content != none
        result = await repo_query(
            "SELECT VALUE count() as count FROM note WHERE content != none GROUP ALL"
        )
```

**实际重建阶段 (`embedding_commands.py:589-602`)**:

```python
if include_notes:
    if mode == "existing":
        # 与启动阶段一致
        result = await repo_query(
            "SELECT id FROM note WHERE embedding != none AND array::len(embedding) > 0"
        )
    else:  # mode == "all"
        # ⚠️ 多了 string::trim(content) != '' 条件!
        result = await repo_query(
            "SELECT id FROM note WHERE content != none AND string::trim(content) != ''"
        )
```

**差异分析**:

| 模式 | 启动阶段 | 实际重建阶段 | 差异 |
|-----|---------|-------------|------|
| `existing` | `note.embedding` 非空且长度>0 | 相同 | 无差异 |
| `all` | `note.content != none` | `content != none AND string::trim(content) != ''` | **实际阶段排除空字符串** |

##### Insight 统计对比 (差异最大!)

**启动阶段 (`embedding_rebuild.py:78-91`)**:

```python
if request.include_insights:
    if request.mode == "existing":
        result = await repo_query(
            "SELECT VALUE count() as count FROM source_insight WHERE embedding != none AND array::len(embedding) > 0 GROUP ALL"
        )
    else:  # mode == "all"
        # ⚠️ 没有任何条件! 直接统计所有 source_insight 记录
        result = await repo_query(
            "SELECT VALUE count() as count FROM source_insight GROUP ALL"
        )
```

**实际重建阶段 (`embedding_commands.py:604-617`)**:

```python
if include_insights:
    if mode == "existing":
        # 与启动阶段一致
        result = await repo_query(
            "SELECT id FROM source_insight WHERE embedding != none AND array::len(embedding) > 0"
        )
    else:  # mode == "all"
        # ⚠️ 有条件: content != none AND string::trim(content) != ''
        result = await repo_query(
            "SELECT id FROM source_insight WHERE content != none AND string::trim(content) != ''"
        )
```

**差异分析 (Critical!)**:

| 模式 | 启动阶段 | 实际重建阶段 | 差异 |
|-----|---------|-------------|------|
| `existing` | `source_insight.embedding` 非空且长度>0 | 相同 | 无差异 |
| `all` | **无任何条件** (`GROUP ALL`) | `content != none AND string::trim(content) != ''` | **差异极大!** |

#### 8.9.3 差异汇总表

| 数据类型 | 模式 | 启动阶段条件 | 实际重建阶段条件 | 差异类型 |
|---------|------|-------------|-----------------|---------|
| Source | `existing` | `embedding != none AND len > 0` | 相同 | 无 |
| Source | `all` | `full_text != none` | `full_text != none AND trim(full_text) != ''` | 排除空字符串 |
| Note | `existing` | `embedding != none AND len > 0` | 相同 | 无 |
| Note | `all` | `content != none` | `content != none AND trim(content) != ''` | 排除空字符串 |
| Insight | `existing` | `embedding != none AND len > 0` | 相同 | 无 |
| Insight | `all` | **无任何条件** | `content != none AND trim(content) != ''` | **差异极大** |

#### 8.9.4 用户可见影响

```
┌────────────────────────────────────────────────────────────────────────┐
│                    口径差异的用户可见影响                                 │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景 1: 用户选择 mode="all" 开始重建                                    │
│                                                                         │
│  启动阶段 (用户看到):                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  Rebuild operation started.                                       │  │
│  │  Estimated 150 items to process.                                  │  │
│  │  ├── Sources: 50 (full_text != none)                              │  │
│  │  ├── Notes: 50 (content != none)                                  │  │
│  │  └── Insights: 50 (无任何条件 - 所有记录)                          │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  实际重建阶段 (实际处理):                                                │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  实际只处理了 100 个项目                                           │  │
│  │  ├── Sources: 40 (full_text 非空且非空白 - 排除了 10 个空字符串)   │  │
│  │  ├── Notes: 40 (content 非空且非空白 - 排除了 10 个空字符串)        │  │
│  │  └── Insights: 20 (content 非空且非空白 - 排除了 30 个无内容记录)  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  进度展示问题:                                                          │
│                                                                         │
│  进度计算: percentage = jobs_submitted / total_items * 100            │
│                                                                         │
│  问题 1: 分母不一致                                                      │
│  ├── 启动阶段: total_estimate = 150 (返回给用户的 initial total_items) │
│  └── 实际阶段: total_items = 100 (存储在父任务 result 中)              │
│                                                                         │
│  问题 2: 用户困惑                                                        │
│  ├── 用户以为要处理 150 个                                              │
│  ├── 实际只处理了 100 个                                                │
│  └── 进度条可能显示异常 (超过 100% 或永远达不到)                         │
│                                                                         │
│  问题 3: Insight 差异最大                                                │
│  ├── 启动阶段: 统计所有 source_insight 记录 (包括无 content 的)         │
│  ├── 实际阶段: 只统计有 content 且非空白的记录                           │
│  └── 可能导致预估与实际差异超过 50%                                      │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

#### 8.9.5 根本原因分析

| 问题 | 根本原因 |
|-----|---------|
| **重复代码** | 两个阶段有几乎相同的查询逻辑，但写在不同文件中 |
| **无统一常量** | 没有定义统一的查询条件常量或函数 |
| **测试不足** | 边界情况（空字符串、null 值）没有统一处理 |
| **Insight 特殊问题** | 启动阶段 `mode="all"` 时忘记加 `content != none` 条件 |

### 8.10 漏点分析: 子任务 command_id 的保存与丢失点

#### 8.10.1 问题概述

父任务 `rebuild_embeddings_command` 作为协调器，提交了大量子任务（`embed_source`, `embed_note`, `embed_insight`），但**没有保存这些子任务的 command_id**，导致：

1. 无法追踪子任务的执行状态
2. 无法汇总子任务的成功/失败情况
3. 子任务执行失败的错误信息**完全不可见**

#### 8.10.2 代码级证据: command_id 的丢失点

##### 提交子任务的代码 (`embedding_commands.py:690-748`)

```python
# Submit embed_source commands for sources
logger.info(f"\nSubmitting {len(items['sources'])} source embedding jobs...")
for idx, source_id in enumerate(items["sources"], 1):
    try:
        # ⚠️ submit_command() 返回 command_id，但被直接丢弃!
        submit_command(
            "open_notebook",
            "embed_source",
            {"source_id": source_id},
        )
        sources_submitted += 1  # 只是计数器，不保存 command_id

        if idx % 50 == 0 or idx == len(items["sources"]):
            logger.info(
                f"  Progress: {idx}/{len(items['sources'])} source jobs submitted"
            )

    except Exception as e:
        logger.error(f"Failed to submit embed_source for {source_id}: {e}")
        failed_submissions += 1  # 只是计数器

# Submit embed_note commands for notes (相同模式)
for idx, note_id in enumerate(items["notes"], 1):
    try:
        submit_command(  # ⚠️ 返回值被丢弃
            "open_notebook",
            "embed_note",
            {"note_id": note_id},
        )
        notes_submitted += 1
    except Exception as e:
        failed_submissions += 1

# Submit embed_insight commands for insights (相同模式)
for idx, insight_id in enumerate(items["insights"], 1):
    try:
        submit_command(  # ⚠️ 返回值被丢弃
            "open_notebook",
            "embed_insight",
            {"insight_id": insight_id},
        )
        insights_submitted += 1
    except Exception as e:
        failed_submissions += 1
```

**关键问题**: `submit_command()` 函数会返回 `command_id`，但代码中**没有接收这个返回值**。

##### 返回结果的数据结构 (`embedding_commands.py:764-773`)

```python
return RebuildEmbeddingsOutput(
    success=True,
    total_items=total_items,
    jobs_submitted=jobs_submitted,
    failed_submissions=failed_submissions,
    sources_submitted=sources_submitted,  # 只是数字
    notes_submitted=notes_submitted,      # 只是数字
    insights_submitted=insights_submitted, # 只是数字
    processing_time=processing_time,
    # ⚠️ 没有任何字段保存子任务的 command_id 列表!
)
```

#### 8.10.3 完整的丢失流程图

```
┌────────────────────────────────────────────────────────────────────────┐
│                    子任务 command_id 的完整丢失流程                        │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. submit_command() 被调用                                             │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  def submit_command(app_id, command_name, args):            │   │
│     │      # 1. 在 SurrealDB 创建 command 记录                      │   │
│     │      # 2. 生成唯一的 command_id                               │   │
│     │      # 3. 返回 command_id                                    │   │
│     │      return command_id  # ⚠️ 这个值被丢弃!                  │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  2. 调用处没有接收返回值                                                  │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  for source_id in items["sources"]:                          │   │
│     │      try:                                                     │   │
│     │          submit_command(      # ⚠️ 没有赋值!                │   │
│     │              "open_notebook",                                 │   │
│     │              "embed_source",                                  │   │
│     │              {"source_id": source_id},                        │   │
│     │          )                                                    │   │
│     │          sources_submitted += 1  # 只是计数器                 │   │
│     │      except Exception as e:                                   │   │
│     │          failed_submissions += 1  # 只是计数器                │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  3. command_id 永远丢失                                                  │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  ┌─────────────┐      ┌─────────────┐                        │   │
│     │  │ command_id  │ ───▶ │  内存临时   │ ───▶ │  完全丢失! │   │
│     │  │ (返回值)    │      │  变量(无)   │      │             │   │
│     │  └─────────────┘      └─────────────┘                        │   │
│     │                                                                 │   │
│     │  只有计数器被保存:                                              │   │
│     │  ├── sources_submitted: 50 (数字)                              │   │
│     │  ├── notes_submitted: 30 (数字)                                │   │
│     │  └── insights_submitted: 20 (数字)                             │   │
│     │                                                                 │   │
│     │  没有任何地方保存:                                               │   │
│     │  ├── sub_commands: ["embed_source:abc123", "embed_note:def456", ...]│   │
│     │  ├── failed_command_ids: [...]                                  │   │
│     │  └── 任何 command_id 相关的数据                                  │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

#### 8.10.4 导致失败不可观测的原因分析

```
┌────────────────────────────────────────────────────────────────────────┐
│                    失败不可观测的完整链路                                 │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  假设场景: 100 个子任务被提交，其中 5 个执行失败                          │
│                                                                         │
│  子任务执行流程:                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  子任务 1: embed_source (command_id: "abc123")                   │  │
│  │  └── 执行成功: result = {"success": true, ...}                   │  │
│  │                                                                   │  │
│  │  子任务 2: embed_source (command_id: "def456")                   │  │
│  │  └── 执行失败: result = {"success": false,                       │  │
│  │                           "error_message": "API 限流",           │  │
│  │                           ...}                                    │  │
│  │                                                                   │  │
│  │  ...                                                              │  │
│  │                                                                   │  │
│  │  子任务 100: embed_insight (command_id: "xyz789")                │  │
│  │  └── 执行成功                                                     │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              ▼                                          │
│  父任务状态查询时的情况:                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  父任务 status = "completed" (因为所有子任务都提交了)              │  │
│  │                                                                   │  │
│  │  父任务 result = {                                                 │  │
│  │      "total_items": 100,                                          │  │
│  │      "jobs_submitted": 100,     ← 看起来都成功了!                 │  │
│  │      "failed_submissions": 0,    ← 0 个提交失败                   │  │
│  │      "sources_submitted": 50,                                    │  │
│  │      "notes_submitted": 30,                                      │  │
│  │      "insights_submitted": 20                                    │  │
│  │  }                                                                 │  │
│  │                                                                   │  │
│  │  ⚠️ 问题:                                                         │  │
│  │  ├── 没有 sub_command_ids 字段                                    │  │
│  │  ├── 无法查询这 100 个子任务的实际状态                             │  │
│  │  └── 5 个执行失败的子任务**完全不可见**!                           │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              ▼                                          │
│  前端展示给用户的情况:                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  ✅ 完成 (绿色勾选)                                                │  │
│  │                                                                   │  │
│  │  进度: 100 / 100 (100%)                                          │  │
│  │  ┌████████████████████████████████████████████████████████████┐  │  │
│  │  └████████████████████████████████████████████████████████████┘  │  │
│  │                                                                   │  │
│  │  统计:                                                            │  │
│  │  ├── Sources: 50                                                  │  │
│  │  ├── Notes: 30                                                    │  │
│  │  ├── Insights: 20                                                 │  │
│  │  └── 失败: 0 (⚠️ 这是提交失败数，不是执行失败数!)                   │  │
│  │                                                                   │  │
│  │  用户感知: 所有 100 个都成功了!                                   │  │
│  │  实际情况: 95 个成功，5 个失败 (用户完全不知道!)                  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

#### 8.10.5 失败类型的混淆问题

| 失败类型 | 计数器字段 | 含义 | 用户可见性 |
|---------|-----------|------|-----------|
| **提交失败** | `failed_submissions` | 调用 `submit_command()` 时抛出异常 | ✅ 可见 (黄色警告) |
| **执行失败** | **无任何字段** | 子任务实际执行时失败 | ❌ **完全不可见** |

**代码证据** (`embedding_commands.py:706-708`):

```python
except Exception as e:
    logger.error(f"Failed to submit embed_source for {source_id}: {e}")
    failed_submissions += 1  # 只统计提交失败
```

这里的 `failed_submissions` 只统计**提交时**的异常（如 `submit_command()` 本身抛出异常），不统计**执行时**的失败。

#### 8.10.6 子任务独立状态追踪的限制

虽然 `Source`, `Note`, `Insight` 对象有自己的状态追踪方法（见 `8.4 子任务的独立状态追踪`），但存在以下限制：

| 限制 | 说明 |
|-----|------|
| **没有关联** | 父任务不知道子任务的 command_id 列表 |
| **只能单个查询** | 必须知道具体的 `source_id/note_id/insight_id` 才能查询 |
| **没有批量查询** | 无法一次性查询所有子任务的状态 |
| **重建场景不适用** | 用户触发重建时，不知道具体哪些 ID 会被处理 |

**代码证据** (`open_notebook/domain/notebook.py:318-359`):

```python
async def get_status(self) -> Optional[str]:
    """Get the processing status of the associated command"""
    if not self.command:  # ⚠️ 需要知道具体对象的 command 字段
        return None
    # ...
```

这需要先获取 `Source` 对象，然后访问其 `command` 属性，才能查询状态。但在重建场景下：
1. 父任务不知道提交了哪些 `source_id`
2. 即使知道，也需要逐个加载对象才能查询
3. 没有一个统一的入口可以汇总所有子任务状态

#### 8.10.7 完整的不可观测性链路总结

```
┌────────────────────────────────────────────────────────────────────────┐
│                    完整的不可观测性链路                                   │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 子任务 command_id 丢失                                              │
│     ├── 位置: commands/embedding_commands.py:692-748                  │
│     └── 原因: submit_command() 返回值未被接收                          │
│                                                                         │
│  2. 父任务 result 中无子任务信息                                         │
│     ├── 位置: commands/embedding_commands.py:764-773                  │
│     └── 原因: RebuildEmbeddingsOutput 无 command_id 列表字段           │
│                                                                         │
│  3. 状态查询 API 无法获取子任务                                          │
│     ├── 位置: api/routers/embedding_rebuild.py:123-192                │
│     └── 原因: 只能查询父任务状态，无法查询子任务                         │
│                                                                         │
│  4. 前端无法展示子任务失败                                               │
│     ├── 位置: frontend/src/app/(dashboard)/advanced/components/        │
│     │          RebuildEmbeddings.tsx                                   │
│     └── 原因: 没有子任务状态数据来源                                    │
│                                                                         │
│  5. 用户被误导                                                          │
│     ├── 看到: 绿色勾选 + 100% 进度 + 0 失败                           │
│     └── 实际: 可能有多个子任务执行失败                                  │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 关键代码位置汇总

### 8.1 按功能模块分类

#### 触发层

| 功能 | 文件路径 | 关键函数/组件 |
|-----|---------|--------------|
| 前端重建 UI | `frontend/src/app/(dashboard)/advanced/components/RebuildEmbeddings.tsx` | `RebuildEmbeddings` 组件 |
| 前端 API 封装 | `frontend/src/lib/api/embedding.ts` | `embeddingApi` 对象 |
| 重建 API 路由 | `api/routers/embedding_rebuild.py` | `start_rebuild`, `get_rebuild_status` |
| 搜索 API 路由 | `api/routers/search.py` | `search_knowledge_base`, `vector_search` |

#### 调度层

| 功能 | 文件路径 | 关键函数/组件 |
|-----|---------|--------------|
| 命令服务 | `api/command_service.py` | `CommandService.submit_command_job` |
| 重建协调器 | `commands/embedding_commands.py` | `rebuild_embeddings_command` |
| 数据收集 | `commands/embedding_commands.py` | `collect_items_for_rebuild` |
| Source 嵌入命令 | `commands/embedding_commands.py` | `embed_source_command` |
| Note 嵌入命令 | `commands/embedding_commands.py` | `embed_note_command` |
| Insight 嵌入命令 | `commands/embedding_commands.py` | `embed_insight_command` |
| 创建 Insight 命令 | `commands/embedding_commands.py` | `create_insight_command` |

#### 向量生成层

| 功能 | 文件路径 | 关键函数/组件 |
|-----|---------|--------------|
| 单个文本嵌入 | `open_notebook/utils/embedding.py` | `generate_embedding` |
| 批量嵌入 | `open_notebook/utils/embedding.py` | `generate_embeddings` |
| 均值池化 | `open_notebook/utils/embedding.py` | `mean_pool_embeddings` |
| 文本分块 | `open_notebook/utils/chunking.py` | `chunk_text`, `detect_content_type` |

#### 数据模型层

| 功能 | 文件路径 | 关键函数/组件 |
|-----|---------|--------------|
| Source 模型 | `open_notebook/domain/notebook.py` | `Source.vectorize`, `Source.add_insight` |
| Note 模型 | `open_notebook/domain/notebook.py` | `Note.save` (自动触发嵌入) |
| 向量搜索 | `open_notebook/domain/notebook.py` | `vector_search` |
| 文本搜索 | `open_notebook/domain/notebook.py` | `text_search` |

#### 数据库层

| 功能 | 文件路径 | 关键函数/组件 |
|-----|---------|--------------|
| 数据库操作 | `open_notebook/database/repository.py` | `repo_query`, `repo_insert`, `repo_update` |
| 向量搜索函数 | `open_notebook/database/migrations/9.surrealql` | `fn::vector_search` |
| 文本搜索函数 | `open_notebook/database/migrations/4.surrealql` | `fn::text_search` |
| 索引定义 | `open_notebook/database/migrations/10.surrealql` | `idx_source_insight_source`, `idx_source_embedding_source` |

### 9.2 配置参数汇总

| 参数 | 位置 | 默认值 | 说明 |
|-----|------|-------|------|
| `EMBEDDING_BATCH_SIZE` | `open_notebook/utils/embedding.py` | 50 | 批量嵌入大小 |
| `EMBEDDING_MAX_RETRIES` | `open_notebook/utils/embedding.py` | 3 | 嵌入重试次数 |
| `EMBEDDING_RETRY_DELAY` | `open_notebook/utils/embedding.py` | 2s | 重试间隔 |
| `CHUNK_SIZE` | `open_notebook/utils/chunking.py` | 8191 | 分块大小 (tokens) |
| `CHUNK_OVERLAP` | `open_notebook/utils/chunking.py` | 20 | 分块重叠 (tokens) |
| 命令重试次数 | `commands/embedding_commands.py` | 5 | `embed_*` 命令最大重试 |
| 命令退避策略 | `commands/embedding_commands.py` | `exponential_jitter` | 指数退避 + 抖动 |
| 最小等待时间 | `commands/embedding_commands.py` | 1s | 命令重试最小等待 |
| 最大等待时间 | `commands/embedding_commands.py` | 60s | 命令重试最大等待 |

---

## 附录: 数据流向图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据流总览                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   触发点                                                                      │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐       │
│   │  前端手动触发    │    │  Note.save()    │    │  其他自动触发    │       │
│   │  Rebuild UI     │    │  Source.        │    │  (转换、洞察等)  │       │
│   └────────┬────────┘    │  vectorize()    │    └────────┬────────┘       │
│            │              └────────┬────────┘             │                │
│            │                       │                      │                │
│            └───────────────────────┼──────────────────────┘                │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      surreal_commands 调度层                          │  │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │  │
│   │  │rebuild_     │  │  embed_     │  │  embed_     │                 │  │
│   │  │embeddings   │  │  source     │  │  note       │  ...            │  │
│   │  │(协调器)      │  │             │  │             │                 │  │
│   │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │  │
│   │         │                │                │                          │  │
│   │         └────────────────┼────────────────┘                          │  │
│   │                          │                                           │  │
│   └──────────────────────────┼────────────────────────────────────────────┘  │
│                              │                                                │
│                              ▼                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      向量生成层 (utils/embedding.py)                  │  │
│   │  ┌───────────────────────────────────────────────────────────────┐  │  │
│   │  │  generate_embedding()                                           │  │  │
│   │  │  ├── 短文本: 直接嵌入                                            │  │  │
│   │  │  └── 长文本: 分块 → 批量嵌入 → 均值池化                           │  │  │
│   │  └───────────────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                              │                                                │
│                              ▼                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         数据库层 (SurrealDB)                          │  │
│   │                                                                       │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │  │
│   │  │source_       │  │    note      │  │source_       │             │  │
│   │  │embedding     │  │              │  │insight       │             │  │
│   │  │ (分块存储)    │  │ embedding    │  │ embedding    │             │  │
│   │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │  │
│   │         │                  │                  │                      │  │
│   │         └──────────────────┼──────────────────┘                      │  │
│   │                            │                                           │  │
│   │                            ▼                                           │  │
│   │              ┌──────────────────────────────┐                         │  │
│   │              │    fn::vector_search()       │                         │  │
│   │              │  vector::similarity::cosine  │                         │  │
│   │              │    (搜索时实时计算相似度)      │                         │  │
│   │              └──────────────────────────────┘                         │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

**报告生成时间**: 2026-05-01  
**分析范围**: open-notebook 项目向量重建与搜索索引刷新流程  
**数据来源**: 代码库静态分析
