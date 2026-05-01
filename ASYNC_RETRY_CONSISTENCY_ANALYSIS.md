# Open Notebook 异步重试与一致性深度分析报告

> 分析日期：2026-05-01  
> 深度主题：重试参数语义、并发幂等性风险、尽力回收策略

---

## 目录

1. [重试时的参数语义深度分析](#1-重试时的参数语义深度分析)
2. [状态检查失败时的并发幂等性风险](#2-状态检查失败时的并发幂等性风险)
3. [尽力回收而非强一致回滚的设计分析](#3-尽力回收而非强一致回滚的设计分析)
4. [代码缺陷与改进建议](#4-代码缺陷与改进建议)

---

## 1. 重试时的参数语义深度分析

### 1.1 问题：注释与实现的不一致

**发现**：重试接口的代码注释与实际实现存在不一致。

**代码位置**: `api/routers/sources.py:882-889`

```python
# 实际代码
command_input = SourceProcessingInput(
    source_id=str(source.id),
    content_state=content_state,
    notebook_ids=notebook_ids,
    transformations=[],  # 空列表
    embed=True,          # 总是 true
)
```

**代码中的注释**:
```python
# Use default transformations on retry (L887)
# Always embed on retry (L888)
```

### 1.2 `transformations=[]` 的实际语义

让我们追踪 `transformations` 参数在工作流中的实际行为。

#### 1.2.1 初始创建时的 transformations

**代码位置**: `api/routers/sources.py:130-136`

```python
# 解析用户选择的转换
transformations_list = []
if transformations:
    try:
        transformations_list = json.loads(transformations)
    except json.JSONDecodeError:
        # ...
```

**用户交互流程**：
```
用户在添加信息源时
         │
         ▼
选择是否应用转换（可选）
         │
         ▼
前端发送 transformations: ["transformation:abc", "transformation:xyz"]
         │
         ▼
后端加载这些 Transformation 对象
         │
         ▼
source_graph 并行运行所有转换
```

#### 1.2.2 source_graph 中的转换处理

**代码位置**: `open_notebook/graphs/source.py:130-146`

```python
def trigger_transformations(state: SourceState, config: RunnableConfig) -> List[Send]:
    """
    决定是否运行转换。
    这是 LangGraph 的条件边函数。
    """
    if len(state["apply_transformations"]) == 0:
        # 关键：空列表直接返回空列表，不运行任何转换
        return []

    to_apply = state["apply_transformations"]
    logger.debug(f"Applying transformations {to_apply}")

    # 为每个转换创建并行的 Send
    return [
        Send(
            "transform_content",
            {
                "source": state["source"],
                "transformation": t,
            },
        )
        for t in to_apply
    ]
```

#### 1.2.3 语义分析

| 场景 | transformations 值 | 实际行为 |
|------|-------------------|----------|
| **初始创建 - 用户选择转换** | `["trans:abc", "trans:xyz"]` | 并行运行所有选中的转换 |
| **初始创建 - 用户不选转换** | `[]` (默认值) | 不运行任何转换 |
| **重试时** | `[]` (硬编码) | 不运行任何转换 |

**关键发现**：

```
注释所说："Use default transformations on retry"
实际行为："No transformations on retry"
```

这是一个**注释与实现不一致**的缺陷！

#### 1.2.4 为什么会这样？

让我们检查 `apply_default` 字段：

**代码位置**: `open_notebook/database/migrations/5.surrealql:14`

```sql
DEFINE FIELD IF NOT EXISTS apply_default ON TABLE transformation TYPE bool DEFAULT False;
```

**数据库中的默认转换** (同文件)：
```sql
insert into transformation  [
   {
      "id": "transformation:summary",
      "name": "summary",
      "title": "Summary",
      "description": "Creates a concise summary of the document...",
      "prompt": "...",
      "apply_default": False    -- 注意：默认是 False
   },
   {
      "id": "transformation:key_insights",
      "name": "key_insights",
      "title": "Key Insights",
      "apply_default": False    -- 也是 False
   },
   -- ... 其他转换
];
```

**结论**：

1. 数据库中所有默认转换的 `apply_default=False`
2. 没有代码逻辑会自动应用 `apply_default=True` 的转换
3. 因此，`transformations=[]` 实际上意味着**不运行任何转换**
4. 注释中的 "default transformations" 是误导性的

### 1.3 `embed=True` 的实际语义

#### 1.3.1 save_source 中的处理

**代码位置**: `open_notebook/graphs/source.py:118-127`

```python
async def save_source(state: SourceState) -> dict:
    # ... 保存源 ...

    if state["embed"]:
        if source.full_text and source.full_text.strip():
            logger.debug("Embedding content for vector search")
            await source.vectorize()  # 调用向量化
        else:
            logger.warning(
                f"Source {source.id} has no text content to embed, skipping vectorization"
            )

    return {"source": source}
```

#### 1.3.2 Source.vectorize() 的实现

**代码位置**: `open_notebook/domain/notebook.py` (需要确认)

从 `embed_source_command` 可以推断：

**代码位置**: `commands/embedding_commands.py:319-440`

```python
@command("embed_source", ...)
async def embed_source_command(input_data: EmbedSourceInput) -> EmbedSourceOutput:
    # ...
    
    # 1. 加载源
    source = await Source.get(input_data.source_id)
    
    # 2. 先 DELETE 已有嵌入（幂等性关键）
    logger.debug(f"Deleting existing embeddings for source {input_data.source_id}")
    await repo_query(
        "DELETE source_embedding WHERE source = $source_id",
        {"source_id": ensure_record_id(input_data.source_id)},
    )
    
    # 3. 分块
    chunks = chunk_text(source.full_text, content_type=content_type)
    
    # 4. 生成嵌入
    embeddings = await generate_embeddings(chunks, command_id=cmd_id)
    
    # 5. 批量 INSERT
    records = [
        {"source": ensure_record_id(input_data.source_id), ...}
        for idx, (chunk, embedding) in enumerate(zip(chunks, embeddings))
    ]
    await repo_insert("source_embedding", records)
```

#### 1.3.3 幂等性分析

| 操作 | 是否幂等 | 说明 |
|------|----------|------|
| `DELETE source_embedding WHERE source = $id` | ✅ 幂等 | 多次删除结果相同 |
| `INSERT source_embedding` | ❌ 非幂等 | 会重复插入 |
| **DELETE + INSERT** | ✅ 幂等 | 先删除再插入，结果唯一 |

**关键设计**：

```
embed_source_command 的流程：

1. DELETE FROM source_embedding WHERE source = $id
         │
         ▼ (清除所有旧嵌入)
         │
2. 分块 + 生成嵌入
         │
         ▼
3. INSERT INTO source_embedding
         │
         ▼
最终结果：只有新的嵌入，无重复
```

#### 1.3.4 语义总结

**`embed=True` 的含义**：

| 维度 | 说明 |
|------|------|
| **行为** | 总是重新向量化 |
| **幂等性** | 通过 DELETE + INSERT 保证幂等 |
| **成本** | 如果源很大，重复向量化有 API 成本 |
| **一致性** | 确保嵌入与当前 `full_text` 同步 |

**对比**：

| 场景 | embed 值 | 行为 |
|------|-----------|------|
| 初始创建 - 用户选择向量化 | `True` | 向量化 |
| 初始创建 - 用户不选择向量化 | `False` | 不向量化 |
| 重试时 | `True` (硬编码) | **总是重新向量化** |

### 1.4 参数语义对照表

| 参数 | 初始创建 | 重试时 | 实际语义 |
|------|----------|--------|----------|
| `transformations` | 用户选择的 ID 列表 | `[]` (硬编码) | **不运行任何转换** |
| `embed` | 用户选择的布尔值 | `True` (硬编码) | **总是重新向量化** |
| `delete_source` | 可能为 True | 不传递 | 不删除文件 |

### 1.5 设计意图与实际效果

#### 1.5.1 可能的设计意图

从代码模式推断，开发者可能想要：

```
重试时：
├── 不重复运行转换（因为转换可能在之前部分成功）
└── 总是向量化（确保嵌入是最新的）
```

这是合理的：
- 转换可能在失败前已经成功运行了部分
- 重复运行转换可能导致重复的 `source_insight` 记录
- 向量化通过 DELETE + INSERT 是幂等的

#### 1.5.2 但注释是错误的

注释说 "Use default transformations"，但实际是 "No transformations"。

如果意图是"使用默认转换"，代码应该：

```python
# 伪代码：查询 apply_default=True 的转换
default_transformations = await repo_query(
    "SELECT id FROM transformation WHERE apply_default = true"
)

command_input = SourceProcessingInput(
    # ...
    transformations=[t["id"] for t in default_transformations],
    # ...
)
```

但当前代码是硬编码的 `[]`。

---

## 2. 状态检查失败时的并发幂等性风险

### 2.1 问题代码

**代码位置**: `api/routers/sources.py:831-843`

```python
# Check if source already has a running command
if source.command:
    try:
        status = await source.get_status()
        if status in ["running", "queued"]:
            raise HTTPException(
                status_code=400,
                detail="Source is already processing. Cannot retry while processing is active.",
            )
    except Exception as e:
        logger.warning(
            f"Failed to check current status for source {source_id}: {e}"
        )
        # 关键：继续重试，即使状态检查失败
        # Continue with retry if we can't check status
```

### 2.2 风险场景分析

#### 2.2.1 场景描述

```
时间线：

T1: 用户创建源，提交 process_source 命令
    ├── command:1 进入 running 状态
    └── source.command = "command:1"

T2: 数据库暂时不可用（网络波动、连接池耗尽等）

T3: 用户点击"重试"按钮
    ├── 进入 retry_source_processing
    ├── source.command = "command:1" 存在
    ├── 尝试调用 source.get_status()
    └── 数据库不可用，抛出异常
        ├── logger.warning 记录警告
        └── 代码选择 CONTINUE 而不是 ABORT

T4: 新命令 command:2 被提交
    ├── source.command 更新为 "command:2"
    └── 进入 queued 状态

T5: 数据库恢复，command:1 和 command:2 都在运行
    ├── 两个命令同时处理同一个源
    └── 可能导致数据不一致
```

#### 2.2.2 为什么状态检查会失败？

让我们看 `source.get_status()` 的实现：

**代码位置**: `open_notebook/domain/notebook.py` (推断)

```python
async def get_status(self) -> Optional[str]:
    """Get the processing status of the associated command"""
    if not self.command:
        return None

    try:
        from surreal_commands import get_command_status

        # 这会查询数据库
        status = await get_command_status(str(self.command))
        return status.status if status else "unknown"
    except Exception as e:
        logger.warning(f"Failed to get command status for {self.command}: {e}")
        return "unknown"
```

可能的失败原因：

| 失败原因 | 是否临时 | 当前处理 |
|----------|----------|----------|
| 数据库连接超时 | ✅ 临时 | 继续 |
| SurrealDB 事务冲突 | ✅ 临时 | 继续 |
| command 记录不存在 | ❌ 永久 | 继续 |
| 网络分区 | ✅ 临时 | 继续 |

### 2.3 并发执行的幂等性分析

让我们分析如果两个命令同时运行会发生什么。

#### 2.3.1 process_source 命令的执行流程

```
process_source_command:
│
├── 1. 加载 Transformation 列表
│
├── 2. 加载 Source，更新 source.command
│
├── 3. 调用 source_graph.ainvoke()
│       │
│       ├── content_process: 提取文本
│       │
│       ├── save_source: 更新 source.full_text
│       │                     调用 source.vectorize() (如果 embed=True)
│       │
│       └── transform_content (并行): 创建 source_insight
│
└── 4. 返回结果
```

#### 2.3.2 各步骤的幂等性

| 步骤 | 操作 | 幂等性 | 并发风险 |
|------|------|--------|----------|
| **content_process** | 从 URL/文件提取文本 | ✅ 只读 | 无风险 |
| **save_source** | UPDATE source SET full_text=... | ✅ 幂等 | 可能覆盖，但值相同 |
| **vectorize** | DELETE + INSERT source_embedding | ✅ 幂等 | 后执行的覆盖先执行的 |
| **transform_content** | INSERT INTO source_insight | ❌ 非幂等 | **重复创建洞察** |

#### 2.3.3 风险最高的操作：创建洞察

**代码位置**: `open_notebook/graphs/source.py:149-168`

```python
async def transform_content(state: TransformationState) -> Optional[dict]:
    source = state["source"]
    content = source.full_text
    if not content:
        return None
    transformation: Transformation = state["transformation"]

    logger.debug(f"Applying transformation {transformation.name}")
    result = await transform_graph.ainvoke(
        dict(input_text=content, transformation=transformation)
    )
    
    # 关键：直接创建 insight，不检查是否已存在
    await source.add_insight(transformation.title, result["output"])
    
    return {
        "transformation": [
            {
                "output": result["output"],
                "transformation_name": transformation.name,
            }
        ]
    }
```

**Source.add_insight()** (推断)：

```python
async def add_insight(self, insight_type: str, content: str):
    # CREATE，不是 UPSERT
    await repo_query(
        """
        CREATE source_insight CONTENT {
            "source": $source_id,
            "insight_type": $insight_type,
            "content": $content
        };
        """,
        {
            "source_id": ensure_record_id(self.id),
            "insight_type": insight_type,
            "content": content,
        },
    )
```

**并发执行的结果**：

```
时间线：

Command A (先开始)        Command B (后开始)
         │                        │
         ▼                        ▼
   提取文本完成              提取文本完成
         │                        │
         ▼                        ▼
   UPDATE source              UPDATE source
   (full_text)                (full_text)
         │                        │
         ▼                        ▼
   CREATE source_insight    CREATE source_insight
   (Summary 类型)            (Summary 类型)
         │                        │
         ▼                        ▼
   结果：两个 "Summary" 类型的 insight
         │
         ▼
   用户在 SourceCard 中看到 insights_count 翻倍
```

#### 2.3.4 向量化的幂等性（安全的）

```python
# commands/embedding_commands.py:352-357

# 先删除所有旧嵌入
await repo_query(
    "DELETE source_embedding WHERE source = $source_id",
    {"source_id": ensure_record_id(input_data.source_id)},
)

# 再插入新嵌入
await repo_insert("source_embedding", records)
```

**并发场景**：

```
Command A: DELETE (空) ──► INSERT (10 chunks)
Command B:              ──► DELETE (删除 A 的 10 个) ──► INSERT (10 chunks)

结果：Command B 的 10 个 chunk，无重复
```

向量化是安全的，但洞察创建有风险。

### 2.4 设计决策分析

#### 2.4.1 为什么选择"继续"？

开发者可能的考虑：

1. **状态检查失败是临时的**
   - 数据库连接可能很快恢复
   - 拒绝用户操作可能更令人沮丧

2. **并发风险低**
   - 两个命令同时运行的概率低
   - 大多数操作是幂等的

3. **可用性 > 一致性**
   - 优先保证用户可以操作
   - 数据不一致可以事后修复

#### 2.4.2 风险评估

| 风险项 | 概率 | 影响 | 严重程度 |
|--------|------|------|----------|
| 状态检查失败 | 中（数据库波动） | - | - |
| 状态检查失败时原命令实际在运行 | 低 | 高 | **高** |
| 重复创建洞察 | 低 | 中（可手动删除） | 中 |
| 重复向量化 | 低 | 低（API 成本） | 低 |

#### 2.4.3 替代设计

**方案 1：失败时拒绝（保守）**

```python
if source.command:
    try:
        status = await source.get_status()
        if status in ["running", "queued"]:
            raise HTTPException(400, "Already processing")
    except Exception as e:
        # 改变：状态检查失败时拒绝
        raise HTTPException(
            status_code=503,
            detail="Cannot verify processing status. Please try again later."
        )
```

**方案 2：乐观锁（推荐）**

使用数据库的条件更新：

```python
# 伪代码：使用乐观锁

# 1. 先查询当前 command ID
current_command_id = source.command

# 2. 尝试更新，仅当 command 仍为原值时
# （使用 SurrealDB 的条件更新）
result = await repo_query(
    """
    UPDATE source 
    SET command = $new_command_id 
    WHERE id = $source_id 
      AND command = $expected_command_id
    """,
    {
        "source_id": source.id,
        "expected_command_id": current_command_id,
        "new_command_id": new_command_id,
    }
)

# 3. 如果没有行被更新，说明并发修改
if not result:
    raise HTTPException(
        status_code=409,
        detail="Source status changed. Please refresh and try again."
    )
```

**方案 3：状态缓存**

```python
# 如果 status 查询失败，使用上次已知的状态
# 但这需要额外的缓存存储
```

---

## 3. 尽力回收而非强一致回滚的设计分析

### 3.1 问题代码模式

在代码中发现多处相同的模式：

**代码位置**: `api/routers/sources.py:438-448` (异步路径命令提交失败)

```python
except Exception as e:
    logger.error(f"Failed to submit async processing command: {e}")
    # Clean up source record on command submission failure
    try:
        await source.delete()
    except Exception:
        pass  # 静默忽略
    # Clean up uploaded file if we created it
    if file_path and upload_file:
        try:
            os.unlink(file_path)
        except Exception:
            pass  # 静默忽略
    raise HTTPException(
        status_code=500, detail=f"Failed to queue processing: {str(e)}"
    )
```

**代码位置**: `api/routers/sources.py:496-505` (同步路径失败)

```python
if not result.is_success():
    logger.error(f"Sync processing failed: {result.error_message}")
    # Clean up source record
    try:
        await source.delete()
    except Exception:
        pass  # 静默忽略
    # Clean up uploaded file if we created it
    if file_path and upload_file:
        try:
            os.unlink(file_path)
        except Exception:
            pass  # 静默忽略
    raise HTTPException(...)
```

**代码位置**: `api/routers/sources.py:553-577` (其他异常)

```python
except HTTPException:
    # Clean up uploaded file on HTTP exceptions if we created it
    if file_path and upload_file:
        try:
            os.unlink(file_path)
        except Exception:
            pass
    raise

except InvalidInputError as e:
    # Clean up uploaded file on validation errors if we created it
    if file_path and upload_file:
        try:
            os.unlink(file_path)
        except Exception:
            pass
    raise HTTPException(status_code=400, detail=str(e))

except Exception as e:
    logger.error(f"Error creating source: {str(e)}")
    # Clean up uploaded file on unexpected errors if we created it
    if file_path and upload_file:
        try:
            os.unlink(file_path)
        except Exception:
            pass
    raise HTTPException(status_code=500, detail=f"Error creating source: {str(e)}")
```

### 3.2 模式总结

```python
# 通用模式
try:
    # 清理操作 1
    await some_cleanup_operation()
except Exception:
    pass  # 静默忽略

# 或者
if condition:
    try:
        # 清理操作 2
        another_cleanup()
    except Exception:
        pass  # 静默忽略
```

### 3.3 为什么是"尽力回收"而非"强一致回滚"？

让我们分析可能的原因。

#### 3.3.1 清理操作的性质

| 清理操作 | 失败可能原因 | 失败后果 |
|----------|--------------|----------|
| `source.delete()` | 数据库连接问题、事务冲突 | 孤立的 Source 记录 |
| `os.unlink(file_path)` | 文件被锁定、权限问题、文件已删除 | 孤立的文件 |
| 删除边关系 | 数据库问题 | 孤立的边记录 |

#### 3.3.2 强一致回滚的挑战

**挑战 1：清理操作可能失败**

```
强一致回滚的期望：
┌─────────────────────────────────────────────────────────────┐
│  如果任何一步失败，整个回滚失败，保持一致性                   │
└─────────────────────────────────────────────────────────────┘

但实际：
┌─────────────────────────────────────────────────────────────┐
│  1. 尝试删除 Source 记录                                      │
│     └── 失败（数据库连接问题）                                │
│                                                              │
│  强一致回滚要求：                                             │
│     └── 保留 Source，但删除文件？                             │
│     或                                                        │
│     └── 保留文件，但删除 Source？                             │
│     或                                                        │
│     └── 都保留？                                              │
└─────────────────────────────────────────────────────────────┘
```

**挑战 2：部分成功的回滚**

```
场景：
1. 创建 Source 记录 ✓
2. 创建 reference 边 ✓
3. 提交命令失败 ✗

回滚尝试：
1. 删除 Source 记录 ✓
2. 删除 reference 边 ✗ (失败)

结果：
├── Source 记录已删除
└── reference 边指向不存在的 Source
```

这比都保留更糟糕：**悬空引用**。

**挑战 3：异步环境中的回滚**

对于异步处理路径，命令执行失败时：

```
异步路径失败场景：

1. POST /sources (同步)
   ├── 创建 Source 记录 ✓
   ├── 创建 reference 边 ✓
   ├── 保存上传文件 ✓
   └── 提交命令 ✓

2. process_source 命令执行 (异步)
   ├── content_process ✗ (失败)
   └── 返回 success=False

此时：
├── Source 记录存在 (含 command 引用)
├── reference 边存在
├── 上传文件存在
└── 命令记录存在 (status=failed)
```

**回滚选项**：

| 选项 | 实现难度 | 用户体验 | 一致性 |
|------|----------|----------|--------|
| 自动删除所有 | 高（需要事务） | 用户源消失 | 好 |
| 保留所有 | 低 | 用户看到失败 + 可重试 | 可接受 |
| 标记为"需要清理" | 中 | 用户看到失败 | 好 |

#### 3.3.3 当前设计的选择

```
当前选择：保留所有 + 显示失败状态

用户视角：
├── 看到 Source 卡片显示 "失败"
├── 看到错误消息
├── 可以点击"重试"
└── 或点击"删除"彻底清理
```

**优势**：
1. 用户知道发生了什么
2. 用户有控制权（重试或删除）
3. 实现简单

**劣势**：
1. 可能留下孤立资源
2. 如果用户忘记清理，资源会累积

### 3.4 孤立资源分析

#### 3.4.1 可能的孤立资源类型

| 资源类型 | 存储位置 | 累积场景 | 检测难度 |
|----------|----------|----------|----------|
| Source 记录 | SurrealDB | 命令提交后用户未重试也未删除 | 低 |
| reference 边 | SurrealDB | Source 删除但边未删除 | 中 |
| 上传文件 | 文件系统 | 异步失败后用户删除 Source 但文件未删 | 高 |
| command 记录 | SurrealDB | 命令执行后保留 | 低 |
| source_embedding | SurrealDB | Source 删除但嵌入未删 | 中 |
| source_insight | SurrealDB | Source 删除但洞察未删 | 中 |

#### 3.4.2 Source.delete() 的级联清理

让我们检查 `Source.delete()` 是否级联清理。

**代码位置**: `open_notebook/domain/notebook.py` (需要确认)

从 `Notebook.delete()` 的模式推断：

```python
async def delete(self, delete_exclusive_sources: bool = False) -> Dict[str, int]:
    # ...
    for note in notes:
        await note.delete()  # 级联删除笔记
    
    # 删除边关系
    await repo_query("DELETE artifact WHERE out = $notebook_id", ...)
    await repo_query("DELETE reference WHERE out = $notebook_id", ...)
    # ...
```

但 `Source.delete()` 的实现？

让我们检查 `Notebook.delete()` 中的源删除：

**代码位置**: `open_notebook/domain/notebook.py:189-196`

```python
if source_id and src.get("assigned_others", 0) == 0:
    # Exclusive source - delete it
    try:
        source = await Source.get(str(source_id))
        await source.delete()
        deleted_sources += 1
    except Exception as e:
        logger.warning(
            f"Failed to delete exclusive source {source_id}: {e}"
        )
```

注意：`source.delete()` 被 `try/except` 包裹，失败时静默忽略。

**问题**：`Source.delete()` 是否级联清理关联资源？

- `source_embedding`？
- `source_insight`？
- `reference` 边？

如果没有级联，删除 Source 会留下孤立的 `source_embedding`、`source_insight`、`reference` 边。

### 3.5 设计评估

#### 3.5.1 当前设计的合理性

| 维度 | 评估 |
|------|------|
| **实现复杂度** | 低（try/except pass 模式） |
| **用户体验** | 好（用户看到失败，可操作） |
| **资源清理** | 依赖用户手动删除 |
| **数据一致性** | 可能有孤立资源 |

#### 3.5.2 改进方向

**方案 1：显式级联删除**

```python
# Source.delete() 应该级联清理

async def delete(self):
    # 1. 删除嵌入
    await repo_query(
        "DELETE source_embedding WHERE source = $source_id",
        {"source_id": ensure_record_id(self.id)},
    )
    
    # 2. 删除洞察
    await repo_query(
        "DELETE source_insight WHERE source = $source_id",
        {"source_id": ensure_record_id(self.id)},
    )
    
    # 3. 删除边关系
    await repo_query(
        "DELETE reference WHERE in = $source_id",
        {"source_id": ensure_record_id(self.id)},
    )
    
    # 4. 删除自己
    await super().delete()
```

**方案 2：后台清理任务**

```python
# 定期清理孤立资源
# 例如：删除没有对应 Source 的 source_embedding

async def cleanup_orphaned_resources():
    # 删除没有 source 的嵌入
    await repo_query("""
        DELETE source_embedding 
        WHERE source NOT IN (SELECT id FROM source)
    """)
    
    # 删除没有 source 的洞察
    await repo_query("""
        DELETE source_insight 
        WHERE source NOT IN (SELECT id FROM source)
    """)
    
    # 删除悬空的 reference 边
    await repo_query("""
        DELETE reference 
        WHERE in NOT IN (SELECT id FROM source)
           OR out NOT IN (SELECT id FROM notebook)
    """)
```

**方案 3：事务性回滚**

使用 SurrealDB 的事务：

```python
# 伪代码：使用事务

async def create_source_atomic(source_data, upload_file):
    async with SurrealDB.transaction() as tx:
        try:
            # 1. 保存文件
            file_path = await save_uploaded_file(upload_file)
            
            # 2. 创建 Source 记录
            source = Source(...)
            await tx.save(source)
            
            # 3. 创建边关系
            for notebook_id in source_data.notebooks:
                await source.add_to_notebook(notebook_id)
            
            # 4. 提交命令
            command_id = await submit_command(...)
            
            # 5. 提交事务
            await tx.commit()
            
        except Exception as e:
            # 6. 回滚事务
            await tx.rollback()
            
            # 7. 清理文件（不在事务中）
            if file_path and os.path.exists(file_path):
                os.unlink(file_path)
            
            raise
```

但文件系统操作无法参与数据库事务。

### 3.6 设计权衡总结

| 权衡项 | 当前选择 | 理由 |
|--------|----------|------|
| **回滚一致性** | 尽力回收 | 清理操作可能失败，强一致难以实现 |
| **用户可见性** | 保留失败记录 | 用户需要知道发生了什么 |
| **用户控制权** | 显示重试/删除按钮 | 用户决定如何处理 |
| **资源清理** | 依赖手动删除 | 实现简单，用户可控 |
| **孤立资源** | 可能存在 | 低概率，可通过监控发现 |

---

## 4. 代码缺陷与改进建议

### 4.1 缺陷 1：重试参数注释错误

**问题**：`transformations=[]` 的注释说 "Use default transformations"，实际是 "No transformations"。

**代码位置**: `api/routers/sources.py:887`

```python
# 当前代码
transformations=[],  # Use default transformations on retry
```

**建议修复**：

**方案 A：修正注释**

```python
transformations=[],  # Do not run transformations on retry
                      # (partial transformations may have succeeded before)
```

**方案 B：实现默认转换**

如果意图是使用默认转换，需要实现：

```python
# 查询 apply_default=True 的转换
default_transformations = await repo_query(
    "SELECT id FROM transformation WHERE apply_default = true"
)

command_input = SourceProcessingInput(
    # ...
    transformations=[t["id"] for t in default_transformations],
    # ...
)
```

但需要考虑：
- 默认转换可能在失败前已执行
- 重复执行可能创建重复洞察

**推荐**：方案 A（修正注释），因为当前行为是合理的。

### 4.2 缺陷 2：状态检查失败时继续入队的并发风险

**问题**：`source.get_status()` 失败时继续入队，可能导致并发执行。

**代码位置**: `api/routers/sources.py:839-843`

```python
except Exception as e:
    logger.warning(f"Failed to check current status...")
    # Continue with retry if we can't check status  ← 风险
```

**建议修复**：

**方案 A：失败时拒绝（简单但保守）**

```python
except Exception as e:
    logger.error(f"Failed to check status: {e}")
    raise HTTPException(
        status_code=503,
        detail="Cannot verify processing status due to a temporary error. "
               "Please refresh the page and try again."
    )
```

**方案 B：乐观锁（推荐）**

```python
# 在提交新命令前，验证 source.command 未变化

# 1. 记录当前 command
old_command_id = source.command

# 2. 提交新命令
new_command_id = await CommandService.submit_command_job(...)

# 3. 条件更新：仅当 command 仍为原值时更新
result = await repo_query(
    """
    UPDATE source 
    SET command = $new_command_id 
    WHERE id = $source_id 
      AND command = $old_command_id
    """,
    {
        "source_id": ensure_record_id(source.id),
        "old_command_id": old_command_id,
        "new_command_id": ensure_record_id(new_command_id),
    }
)

# 4. 如果没有行被更新，说明并发修改
if not result:
    raise HTTPException(
        status_code=409,
        detail="Source status changed while processing your request. "
               "Please refresh and try again."
    )
```

### 4.3 缺陷 3：Source.delete() 缺少级联清理

**问题**：删除 Source 时可能留下孤立的 `source_embedding`、`source_insight`、`reference` 边。

**建议**：在 `Source.delete()` 中添加级联清理。

```python
# open_notebook/domain/notebook.py

class Source(ObjectModel):
    # ...
    
    async def delete(self):
        # 级联清理关联资源
        
        # 1. 删除嵌入
        await repo_query(
            "DELETE source_embedding WHERE source = $source_id",
            {"source_id": ensure_record_id(self.id)},
        )
        
        # 2. 删除洞察
        await repo_query(
            "DELETE source_insight WHERE source = $source_id",
            {"source_id": ensure_record_id(self.id)},
        )
        
        # 3. 删除边关系
        await repo_query(
            "DELETE reference WHERE in = $source_id",
            {"source_id": ensure_record_id(self.id)},
        )
        
        # 4. 删除自己
        await super().delete()
```

### 4.4 缺陷 4：清理失败时静默忽略

**问题**：`try/except pass` 模式使得清理失败不可见。

```python
try:
    await source.delete()
except Exception:
    pass  # 静默忽略
```

**建议**：至少记录日志。

```python
try:
    await source.delete()
except Exception as e:
    logger.warning(f"Failed to delete source {source.id} during cleanup: {e}")
    # 继续执行，不影响主流程
```

### 4.5 改进优先级

| 优先级 | 缺陷 | 建议方案 |
|--------|------|----------|
| P0 (高) | 注释错误 | 修正注释 |
| P1 (中) | 状态检查失败继续入队 | 方案 A（失败时拒绝）或 B（乐观锁） |
| P2 (低) | 缺少级联删除 | 添加级联清理 |
| P2 (低) | 清理失败静默 | 添加日志 |

---

## 附录：代码引用索引

| 文件路径 | 说明 |
|----------|------|
| `api/routers/sources.py:882-889` | 重试时的参数设置 |
| `api/routers/sources.py:831-843` | 状态检查失败时继续入队 |
| `api/routers/sources.py:438-448` | 异步路径清理模式 |
| `api/routers/sources.py:496-505` | 同步路径清理模式 |
| `open_notebook/graphs/source.py:130-146` | trigger_transformations 条件边 |
| `open_notebook/graphs/source.py:149-168` | transform_content 非幂等 |
| `commands/embedding_commands.py:352-357` | embed_source 的 DELETE+INSERT |
| `open_notebook/database/migrations/5.surrealql` | apply_default 字段定义 |
