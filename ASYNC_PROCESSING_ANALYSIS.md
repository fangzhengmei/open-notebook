# Open Notebook 异步处理与一致性恢复机制分析报告

> 分析日期：2026-05-01  
> 补充分析：数据聚合链路中的异步处理、重试策略、回滚机制与前端一致性

---

## 目录

1. [异步处理架构概览](#1-异步处理架构概览)
2. [自动重试策略](#2-自动重试策略)
3. [手动重试机制](#3-手动重试机制)
4. [失败分类与回滚策略](#4-失败分类与回滚策略)
5. [状态同步链路](#5-状态同步链路)
6. [前端列视图一致性保障](#6-前端列视图一致性保障)
7. [关键设计决策总结](#7-关键设计决策总结)

---

## 1. 异步处理架构概览

### 1.1 架构分层

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          异步处理架构分层                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Frontend (Next.js / React)                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ • useSourceStatus: 状态轮询 Hook                            │    │   │
│  │  │ • SourceCard: 状态显示 + 重试按钮                            │    │   │
│  │  │ • wasProcessing: 状态跟踪确保捕获完成                         │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼ HTTP / JSON                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Backend API (FastAPI)                             │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ • POST /sources: 同步/异步模式创建                            │    │   │
│  │  │ • POST /sources/{id}/retry: 手动重试接口                     │    │   │
│  │  │ • GET /sources/{id}/status: 状态查询接口                     │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼ Command Submission                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │               surreal-commands (任务队列系统)                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ • @command 装饰器定义命令                                     │    │   │
│  │  │ • command 表存储任务状态 (new/queued/running/completed/failed)│    │   │
│  │  │ • 自动重试机制 (max_attempts, exponential_jitter)            │    │   │
│  │  │ • stop_on: 不重试的错误类型                                   │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼ Command Execution                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Command Handlers (业务逻辑)                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ • process_source: 内容提取 + 转换 + 向量化                   │    │   │
│  │  │ • embed_source: 单独向量化任务                                │    │   │
│  │  │ • embed_note: 笔记向量化                                     │    │   │
│  │  │ • create_insight: 创建洞察                                   │    │   │
│  │  │ • run_transformation: 运行转换                               │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼ LangGraph Workflow                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                   source_graph (LangGraph 工作流)                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  START → content_process → save_source → (conditional)      │    │   │
│  │  │  → transform_content (parallel) → END                        │    │   │
│  │  │                                                               │    │   │
│  │  │  • content_process: 调用 content_core 提取文本               │    │   │
│  │  │  • save_source: 更新 Source 记录 + 提交 embed_source        │    │   │
│  │  │  • transform_content: 并行运行所有转换 (Send)                │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 命令类型与职责

| 命令名称 | 职责 | 触发时机 |
|----------|------|----------|
| `process_source` | 完整处理流程：提取文本 → 保存源 → 运行转换 → 提交向量化 | 新信息源创建时 |
| `embed_source` | 仅向量化：分块 → 生成嵌入 → 批量插入 | process_source 完成后 / 手动重建时 |
| `embed_note` | 笔记向量化：生成嵌入 → UPSERT 到 note 记录 | 笔记保存后 |
| `embed_insight` | 洞察向量化 | 洞察创建后 |
| `create_insight` | 创建洞察记录 + 提交 embed_insight | 转换完成后 |
| `run_transformation` | 对已有源运行单个转换 | UI 触发洞察生成 |
| `rebuild_embeddings` | 批量重建嵌入（协调器命令） | 管理员手动触发 |

### 1.3 状态机定义

**命令状态流转** (由 surreal-commands 管理)：

```
     ┌──────────┐
     │   new    │  刚创建
     └────┬─────┘
          │
          ▼
     ┌──────────┐
     │  queued  │  队列中等待
     └────┬─────┘
          │
          ▼
    ┌─────────────┐
    │   running   │  执行中
    └──────┬──────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
┌─────────┐ ┌──────────┐
│completed│ │  failed  │
└─────────┘ └──────────┘
     │
     │ 失败重试循环 (max_attempts 次)
     │
     └──► queued ──► running ──► (如果重试)
```

**Source 记录状态字段**：
- `source.command`: 关联的 command 记录 ID
- 通过 `FETCH command` 获取实时状态

---

## 2. 自动重试策略

### 2.1 各命令重试配置

**文件位置**: `commands/source_commands.py` 和 `commands/embedding_commands.py`

#### 2.1.1 process_source 命令

```python
@command(
    "process_source",
    app="open_notebook",
    retry={
        "max_attempts": 15,                    # 最多 15 次重试
        "wait_strategy": "exponential_jitter", # 指数抖动退避
        "wait_min": 1,                         # 最小等待 1 秒
        "wait_max": 120,                       # 最大等待 120 秒
        "stop_on": [ValueError, ConfigurationError],  # 不重试的错误类型
        "retry_log_level": "debug",            # 重试日志级别 (减少噪音)
    },
)
```

**设计考量**：
- **15 次重试**：应对 SurrealDB v2 的事务冲突问题（深度队列场景）
- **指数抖动退避**：避免惊群效应，等待时间 = 随机(0, min(wait_max, wait_min * 2^attempt))
- **120 秒最大等待**：给队列足够时间排空
- **debug 级别日志**：事务冲突重试频繁，避免日志噪音

#### 2.1.2 embed_source / embed_note / embed_insight 命令

```python
@command(
    "embed_source",
    app="open_notebook",
    retry={
        "max_attempts": 5,           # 最多 5 次重试
        "wait_strategy": "exponential_jitter",
        "wait_min": 1,
        "wait_max": 60,              # 最大等待 60 秒
        "stop_on": [ValueError, ConfigurationError],
        "retry_log_level": "debug",
    },
)
```

**设计考量**：
- **5 次重试**：向量化主要是网络调用 (API 调用)，不需要太多重试
- **60 秒最大等待**：API 限流场景下的等待

#### 2.1.3 run_transformation 命令

```python
@command(
    "run_transformation",
    app="open_notebook",
    retry={
        "max_attempts": 5,
        "wait_strategy": "exponential_jitter",
        "wait_min": 1,
        "wait_max": 60,
        "stop_on": [ValueError, ConfigurationError],
        "retry_log_level": "debug",
    },
)
```

### 2.2 指数抖动退避算法

```
等待时间计算 (surreal-commands 实现):

第 1 次重试: 随机(0, 1) 秒
第 2 次重试: 随机(0, 2) 秒
第 3 次重试: 随机(0, 4) 秒
第 4 次重试: 随机(0, 8) 秒
...
第 N 次重试: 随机(0, min(wait_max, wait_min * 2^(N-1))) 秒

process_source (wait_max=120):
  第 1-7 次: 指数增长 (1→2→4→8→16→32→64)
  第 8-15 次: 上限 120 秒

embed_source (wait_max=60):
  第 1-6 次: 指数增长 (1→2→4→8→16→32)
  第 7+ 次: 上限 60 秒
```

**优势**：
1. **避免惊群**：随机化防止多个失败任务同时重试
2. **指数退避**：给系统恢复时间
3. **上限保护**：防止无限等待

### 2.3 重试触发条件

#### 2.3.1 自动重试的错误类型

命令处理函数中通过异常类型区分：

```python
async def process_source_command(
    input_data: SourceProcessingInput,
) -> SourceProcessingOutput:
    try:
        # ... 业务逻辑 ...
        
    except ValueError as e:
        # 永久性失败 - 不重试
        # 例如：源不存在、转换不存在、配置错误
        processing_time = time.time() - start_time
        logger.error(f"Source processing failed: {e}")
        return SourceProcessingOutput(
            success=False,
            source_id=input_data.source_id,
            processing_time=processing_time,
            error_message=str(e),
        )
    except Exception as e:
        # 临时性失败 - 将会自动重试
        # 例如：网络超时、数据库事务冲突、API 限流
        logger.debug(
            f"Transient error processing source {input_data.source_id}: {e}"
        )
        raise  # 重新抛出，触发 surreal-commands 重试
```

#### 2.3.2 错误分类表

| 错误类型 | 异常类 | 处理方式 | 示例场景 |
|----------|--------|----------|----------|
| **永久性失败** | `ValueError` | 返回 success=False，不重试 | 源 ID 不存在、转换不存在、内容为空、YouTube 无字幕 |
| **配置错误** | `ConfigurationError` | `stop_on` 配置，不重试 | API 密钥缺失、模型配置错误 |
| **临时性失败** | `Exception` (其他) | 自动重试 (max_attempts 次) | 网络超时、数据库事务冲突、API 限流、临时服务不可用 |

#### 2.3.3 source_graph 中的失败传播

```python
# open_notebook/graphs/source.py

async def content_process(state: SourceState) -> dict:
    # 调用 content_core 提取内容
    processed_state = await extract_content(content_state)

    if not processed_state.content or not processed_state.content.strip():
        url = processed_state.url or ""
        if url and ("youtube.com" in url or "youtu.be" in url):
            raise ValueError(
                "Could not extract content from this YouTube video. "
                "No transcript or subtitles are available. "
                "Try configuring a Speech-to-Text model in Settings "
                "to transcribe the audio instead."
            )
        raise ValueError(
            "Could not extract any text content from this source. "
            "The content may be empty, inaccessible, or in an unsupported format."
        )

    return {"content_state": processed_state}
```

**注意**：`content_process` 中的失败会抛出 `ValueError`，属于永久性失败，不会自动重试。这是因为：
- 内容提取失败通常是根本性问题（格式不支持、无权限等）
- 重试不太可能解决问题

---

## 3. 手动重试机制

### 3.1 后端重试接口

**文件位置**: `api/routers/sources.py` - `retry_source_processing`

```python
@router.post("/sources/{source_id}/retry", response_model=SourceResponse)
async def retry_source_processing(source_id: str):
    """Retry processing for a failed or stuck source."""
    try:
        # 1. 验证源存在
        source = await Source.get(source_id)
        if not source:
            raise HTTPException(status_code=404, detail="Source not found")

        # 2. 检查是否已有运行中的命令
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
                # 无法检查状态时，继续重试

        # 3. 获取源所属的笔记本
        query = "SELECT notebook FROM reference WHERE source = $source_id"
        references = await repo_query(query, {"source_id": source_id})
        notebook_ids = [str(ref["notebook"]) for ref in references]

        if not notebook_ids:
            raise HTTPException(
                status_code=400, detail="Source is not associated with any notebooks"
            )

        # 4. 根据 asset 重建 content_state
        content_state = {}
        if source.asset:
            if source.asset.file_path:
                content_state = {
                    "file_path": source.asset.file_path,
                    "delete_source": False,  # 重试时不删除文件
                }
            elif source.asset.url:
                content_state = {"url": source.asset.url}
            else:
                raise HTTPException(
                    status_code=400, detail="Source asset has no file_path or url"
                )
        else:
            # text 类型源：使用 full_text
            if source.full_text:
                content_state = {"content": source.full_text}
            else:
                raise HTTPException(
                    status_code=400, detail="Cannot determine source content for retry"
                )

        # 5. 提交新的 process_source 命令
        try:
            import commands.source_commands  # noqa: F401

            command_input = SourceProcessingInput(
                source_id=str(source.id),
                content_state=content_state,
                notebook_ids=notebook_ids,
                transformations=[],  # 重试时使用默认转换
                embed=True,           # 重试时总是向量化
            )

            command_id = await CommandService.submit_command_job(
                "open_notebook",
                "process_source",
                command_input.model_dump(),
            )

            # 6. 更新源的 command 引用
            source.command = ensure_record_id(f"command:{command_id}")
            await source.save()

            # 7. 返回更新后的源信息
            embedded_chunks = await source.get_embedded_chunks()
            return SourceResponse(
                id=source.id or "",
                title=source.title,
                topics=source.topics or [],
                asset=AssetModel(
                    file_path=source.asset.file_path if source.asset else None,
                    url=source.asset.url if source.asset else None,
                )
                if source.asset
                else None,
                full_text=source.full_text,
                embedded=embedded_chunks > 0,
                embedded_chunks=embedded_chunks,
                created=str(source.created),
                updated=str(source.updated),
                command_id=command_id,
                status="queued",
                processing_info={"retry": True, "queued": True},
            )

        except Exception as e:
            logger.error(
                f"Failed to submit retry processing command for source {source_id}: {e}"
            )
            raise HTTPException(
                status_code=500, detail=f"Failed to queue retry processing: {str(e)}"
            )

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error retrying source processing for {source_id}: {str(e)}")
        raise HTTPException(
            status_code=500, detail=f"Error retrying source processing: {str(e)}"
        )
```

### 3.2 手动重试流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          手动重试流程                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  前端用户点击 "重试" 按钮                                                     │
│           │                                                                  │
│           ▼                                                                  │
│  POST /sources/{id}/retry                                                    │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐           │
│  │  1. 验证源是否存在                                            │           │
│  │  2. 检查当前命令状态                                          │           │
│  │     ├── 如果是 running/queued → 返回 400 错误               │           │
│  │     └── 其他状态 (completed/failed/unknown) → 继续          │           │
│  │  3. 查询源关联的笔记本 (reference 边)                         │           │
│  │  4. 根据 asset 重建 content_state:                           │           │
│  │     ├── file_path → {"file_path": ..., "delete_source": false}│         │
│  │     ├── url → {"url": ...}                                  │           │
│  │     └── full_text (text 类型) → {"content": ...}           │           │
│  │  5. 提交新的 process_source 命令                              │           │
│  │  6. 更新 source.command 字段指向新命令                       │           │
│  │  7. 返回 status="queued"                                     │           │
│  └─────────────────────────────────────────────────────────────┘           │
│           │                                                                  │
│           ▼                                                                  │
│  前端：触发缓存失效 → 重新查询状态 → 显示 "queued" 状态                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 手动重试与自动重试的区别

| 维度 | 自动重试 (surreal-commands) | 手动重试 (POST /retry) |
|------|-----------------------------|-------------------------|
| **触发方式** | 命令失败时自动触发 | 用户手动点击按钮 |
| **重试次数** | 受 max_attempts 限制 (5-15 次) | 无限次 (用户可反复重试) |
| **状态检查** | 无 (由队列管理) | 检查是否已有运行中任务 |
| **content_state** | 使用原始参数 | 从 Source.asset/Source.full_text 重建 |
| **transformations** | 使用原始参数 | 空列表 (使用默认转换) |
| **embed** | 使用原始参数 | 总是 true |
| **适用场景** | 临时性故障 (网络、事务冲突) | 永久性故障修复后 |

---

## 4. 失败分类与回滚策略

### 4.1 失败场景分类

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          失败场景分类                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    同步处理路径 (已废弃，仅向后兼容)                  │   │
│  │  失败策略：完全回滚                                                    │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                       │   │
│  │  同步路径失败时：                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  1. 删除 Source 记录 (await source.delete())                │    │   │
│  │  │  2. 删除上传的文件 (os.unlink(file_path))                    │    │   │
│  │  │  3. 返回 500 错误                                            │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                       │   │
│  │  代码位置: api/routers/sources.py:493-509                          │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    异步处理路径 (默认)                                │   │
│  │  失败策略：分阶段回滚                                                 │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                       │   │
│  │  阶段 1: 命令提交前失败                                               │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  触发时机：POST /sources 时，提交命令前                        │    │   │
│  │  │  原因：命令服务不可用、数据库连接失败                          │    │   │
│  │  │  回滚操作：                                                     │    │   │
│  │  │  1. 删除 Source 记录 (await source.delete())                 │    │   │
│  │  │  2. 删除上传的文件 (os.unlink(file_path))                     │    │   │
│  │  │  3. 返回 500 错误                                             │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                       │   │
│  │  阶段 2: 命令执行中失败 (永久性错误)                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  触发时机：process_source 命令执行中抛出 ValueError           │    │   │
│  │  │  原因：内容提取失败、转换不存在、配置错误                      │    │   │
│  │  │  回滚策略：**不回滚** (保留数据用于手动重试)                  │    │   │
│  │  │                                                                 │    │   │
│  │  │  保留的数据：                                                  │    │   │
│  │  │  • Source 记录 (含 asset 信息)                                │    │   │
│  │  │  • 上传的文件 (file_path)                                     │    │   │
│  │  │  • reference 边关系 (与笔记本的关联)                           │    │   │
│  │  │                                                                 │    │   │
│  │  │  设计理由：                                                    │    │   │
│  │  │  • 用户可以看到失败的源                                        │    │   │
│  │  │  • 可以手动重试 (retry 接口)                                  │    │   │
│  │  │  • 上传的大文件不需要重新上传                                   │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                       │   │
│  │  阶段 3: 命令执行中失败 (临时性错误)                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  触发时机：process_source 命令执行中抛出其他 Exception        │    │   │
│  │  │  原因：网络超时、事务冲突、API 限流                            │    │   │
│  │  │  处理策略：自动重试 (max_attempts 次)                         │    │   │
│  │  │                                                                 │    │   │
│  │  │  重试期间：                                                    │    │   │
│  │  │  • 状态保持 running/queued                                    │    │   │
│  │  │  • 前端持续轮询                                                │    │   │
│  │  │  • 成功后正常完成                                              │    │   │
│  │  │  • 重试耗尽后进入 failed 状态 (如阶段 2)                      │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 同步路径完全回滚代码

**文件位置**: `api/routers/sources.py:493-509`

```python
# 同步路径失败处理
if not result.is_success():
    logger.error(f"Sync processing failed: {result.error_message}")
    
    # 1. 删除 Source 记录
    try:
        await source.delete()
    except Exception:
        pass
    
    # 2. 删除上传的文件
    if file_path and upload_file:
        try:
            os.unlink(file_path)
        except Exception:
            pass
    
    # 3. 返回错误
    raise HTTPException(
        status_code=500,
        detail=f"Processing failed: {result.error_message}",
    )
```

### 4.3 异步路径命令提交失败回滚

**文件位置**: `api/routers/sources.py:436-450`

```python
# 异步路径：命令提交失败处理
except Exception as e:
    logger.error(f"Failed to submit async processing command: {e}")
    
    # 1. 删除 Source 记录
    try:
        await source.delete()
    except Exception:
        pass
    
    # 2. 删除上传的文件
    if file_path and upload_file:
        try:
            os.unlink(file_path)
        except Exception:
            pass
    
    raise HTTPException(
        status_code=500, detail=f"Failed to queue processing: {str(e)}"
    )
```

### 4.4 异步路径命令执行失败：不回滚

**设计决策分析**：

| 回滚项 | 同步路径 | 异步路径 (命令执行失败) | 理由 |
|--------|----------|--------------------------|------|
| **Source 记录** | 删除 | 保留 | 用户需要看到失败项 |
| **上传文件** | 删除 | 保留 | 避免重新上传大文件 |
| **reference 边** | 删除 | 保留 | 保持笔记本关联 |
| **用户体验** | 完全消失 | 显示失败 + 重试按钮 | 可控性更好 |

**失败后用户操作**：
1. 查看失败原因 (`error_message`)
2. 点击"重试"按钮 (手动重试)
3. 或删除源 (清理资源)

### 4.5 部分失败处理

#### 4.5.1 source_graph 中的部分失败

```
source_graph 工作流:

START → content_process → save_source → (conditional) → transform_content → END
                                    │
                                    │ 如果 embed=True
                                    ▼
                              submit embed_source
                              (fire-and-forget)
```

**部分失败场景**：

| 阶段 | 失败类型 | 影响 | 处理方式 |
|------|----------|------|----------|
| `content_process` | 永久性失败 | 整个命令失败 | 返回 success=False |
| `content_process` | 临时性失败 | 自动重试 | surreal-commands 管理 |
| `save_source` | 任何失败 | 整个命令失败 | 自动重试 |
| `transform_content` | 单个转换失败 | 不影响其他转换 | 并行 Send，独立失败 |
| `embed_source` (异步) | 任何失败 | 不影响主流程 | 独立命令，独立重试 |

#### 4.5.2 并行转换的独立性

```python
# open_notebook/graphs/source.py:130-146

def trigger_transformations(state: SourceState, config: RunnableConfig) -> List[Send]:
    if len(state["apply_transformations"]) == 0:
        return []

    to_apply = state["apply_transformations"]
    logger.debug(f"Applying transformations {to_apply}")

    # 为每个转换创建独立的 Send
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

**设计**：每个转换通过 `Send` 并行执行，一个转换失败不会影响其他转换。

---

## 5. 状态同步链路

### 5.1 后端状态存储

#### 5.1.1 状态数据结构

```
SurrealDB 中的状态存储:

┌─────────────────┐         ┌─────────────────┐
│     source      │         │     command     │
├─────────────────┤         ├─────────────────┤
│ id              │         │ id              │
│ title           │         │ name            │
│ asset           │         │ app             │
│ full_text       │         │ status          │ ◄── 主状态字段
│ topics          │         │ input           │
│ command ────────┼────────►│ output          │
│ created         │         │ error_message   │ ◄── 错误信息
│ updated         │         │ created         │
│                 │         │ updated         │
│                 │         │ attempts        │ ◄── 已重试次数
│                 │         │ max_attempts    │
└─────────────────┘         │ result          │ ◄── 执行元数据
                              │  ├── started_at │
                              │  ├── completed_at │
                              │  └── execution_metadata │
                              └─────────────────┘
```

#### 5.1.2 状态查询实现

**文件位置**: `open_notebook/domain/notebook.py` - `Source.get_status()`

```python
async def get_status(self) -> Optional[str]:
    """Get the processing status of the associated command"""
    if not self.command:
        return None

    try:
        from surreal_commands import get_command_status

        status = await get_command_status(str(self.command))
        return status.status if status else "unknown"
    except Exception as e:
        logger.warning(f"Failed to get command status for {self.command}: {e}")
        return "unknown"

async def get_processing_progress(self) -> Optional[Dict[str, Any]]:
    """Get detailed processing information for the associated command"""
    if not self.command:
        return None

    try:
        from surreal_commands import get_command_status

        status_result = await get_command_status(str(self.command))
        if not status_result:
            return None

        # 从嵌套的 result 结构提取执行元数据
        result = getattr(status_result, "result", None)
        execution_metadata = (
            result.get("execution_metadata", {})
            if isinstance(result, dict)
            else {}
        )

        return {
            "status": status_result.status,
            "started_at": execution_metadata.get("started_at"),
            "completed_at": execution_metadata.get("completed_at"),
            "error": getattr(status_result, "error_message", None),
        }
    except Exception as e:
        logger.warning(f"Failed to get command progress for {self.command}: {e}")
        return None
```

### 5.2 API 层状态聚合

**文件位置**: `api/routers/sources.py` - `get_sources()`

```python
@router.get("/sources", response_model=List[SourceListResponse])
async def get_sources(...):
    # 使用 FETCH 语法一次性获取 command 数据
    if notebook_id:
        query = f"""
            SELECT id, asset, created, title, updated, topics, command,
            -- 子查询聚合
            (SELECT VALUE count() FROM source_insight 
             WHERE source = $parent.id GROUP ALL)[0].count OR 0 AS insights_count,
            (SELECT VALUE id FROM source_embedding 
             WHERE source = $parent.id LIMIT 1) != [] AS embedded
            FROM (select value in from reference where out=$notebook_id)
            {order_clause}
            LIMIT $limit START $offset
            FETCH command  -- 关键：关联获取 command 记录
        """
    else:
        query = f"""
            SELECT ..., command, ...
            FROM source
            ...
            FETCH command
        """
    
    result = await repo_query(...)
    
    # 处理 FETCH 返回的 command 数据
    response_list = []
    for row in result:
        command = row.get("command")
        command_id = None
        status = None
        processing_info = None

        # command 已经通过 FETCH 解析为字典
        if command and isinstance(command, dict):
            command_id = str(command.get("id"))
            status = command.get("status")
            
            # 提取执行元数据
            result_data = command.get("result")
            execution_metadata = (
                result_data.get("execution_metadata", {})
                if isinstance(result_data, dict)
                else {}
            )
            processing_info = {
                "started_at": execution_metadata.get("started_at"),
                "completed_at": execution_metadata.get("completed_at"),
                "error": command.get("error_message"),
            }
        elif command:
            # FETCH 失败，command 仍是 RecordID
            command_id = str(command)
            status = "unknown"

        response_list.append(
            SourceListResponse(
                id=row["id"],
                # ... 其他字段
                command_id=command_id,
                status=status,
                processing_info=processing_info,
            )
        )
```

### 5.3 前端状态轮询

#### 5.3.1 useSourceStatus Hook

**文件位置**: `frontend/src/lib/hooks/use-sources.ts`

```typescript
export function useSourceStatus(sourceId: string, enabled = true) {
  return useQuery({
    queryKey: ['sources', sourceId, 'status'],
    queryFn: () => sourcesApi.status(sourceId),
    enabled: !!sourceId && enabled,
    
    // 智能轮询：仅在处理中时轮询
    refetchInterval: (query) => {
      const data = query.state.data as SourceStatusResponse | undefined
      if (data?.status === 'running' || 
          data?.status === 'queued' || 
          data?.status === 'new') {
        return 2000  // 处理中：每 2 秒轮询
      }
      return false  // 完成/失败：停止轮询
    },
    
    staleTime: 0,  // 状态数据总是过期，需要实时更新
    retry: (failureCount, error) => {
      // 404 时不重试
      const axiosError = error as { response?: { status?: number } }
      if (axiosError?.response?.status === 404) {
        return false
      }
      return failureCount < 3
    },
  })
}
```

#### 5.3.2 SourceCard 中的状态跟踪

**文件位置**: `frontend/src/components/sources/SourceCard.tsx`

```typescript
export function SourceCard({ source, ... }: SourceCardProps) {
  const { t } = useTranslation()
  const statusConfigMap = getStatusConfig(t)
  
  const sourceWithStatus = source as SourceListResponse & { 
    command_id?: string; 
    status?: string 
  }

  // 关键：跟踪是否曾经处于处理状态
  // 用于在状态从 running → completed/failed 时继续轮询一小段时间
  const [wasProcessing, setWasProcessing] = useState(false)

  // 决定是否应该轮询状态
  const shouldFetchStatus = !!sourceWithStatus.command_id ||
    sourceWithStatus.status === 'new' ||
    sourceWithStatus.status === 'queued' ||
    sourceWithStatus.status === 'running' ||
    wasProcessing  // 即使当前状态已完成，如果曾经在处理，继续轮询

  const { data: statusData, isLoading: statusLoading } = useSourceStatus(
    source.id,
    shouldFetchStatus
  )

  // 确定当前状态
  const rawStatus = statusData?.status || sourceWithStatus.status
  const currentStatus: SourceStatus = isSourceStatus(rawStatus)
    ? rawStatus
    : (sourceWithStatus.command_id ? 'new' : 'completed')

  // 状态流转跟踪与自动刷新
  useEffect(() => {
    const currentStatusFromData = statusData?.status || sourceWithStatus.status

    // 如果当前在处理中，标记 wasProcessing
    if (currentStatusFromData === 'new' || 
        currentStatusFromData === 'running' || 
        currentStatusFromData === 'queued') {
      setWasProcessing(true)
    }

    // 如果曾经在处理，现在完成/失败了
    if (wasProcessing &&
        (currentStatusFromData === 'completed' || currentStatusFromData === 'failed')) {
      setWasProcessing(false)  // 停止轮询

      // 延迟刷新列表，确保 API 数据已更新
      if (onRefresh) {
        setTimeout(() => onRefresh(), 500)
      }
    }
  }, [statusData, sourceWithStatus.status, wasProcessing, onRefresh, source.id])
  
  // ... 渲染逻辑
}
```

### 5.4 状态同步流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          状态同步完整链路                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  后端 (命令执行)                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  surreal-commands 执行命令                                            │   │
│  │  ├── 更新 command.status (new → queued → running)                   │   │
│  │  ├── 更新 command.attempts (重试计数)                                │   │
│  │  ├── 更新 command.result.started_at                                  │   │
│  │  └── 完成/失败时:                                                      │   │
│  │      ├── command.status = completed/failed                           │   │
│  │      ├── command.result.completed_at                                 │   │
│  │      └── command.error_message (失败时)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  API 层 (GET /sources 或 GET /sources/{id}/status)                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  • SELECT ... FETCH command: 关联获取 command 记录                    │   │
│  │  • 提取 command.status, command.error_message, command.result         │   │
│  │  • 封装到 SourceListResponse.status / processing_info                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼ HTTP / JSON                           │
│  前端 (React)                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. useSourceStatus Hook:                                             │   │
│  │     ├── queryKey: ['sources', sourceId, 'status']                    │   │
│  │     ├── refetchInterval: 2000ms (仅当 running/queued/new 时)        │   │
│  │     └── staleTime: 0 (总是需要刷新)                                   │   │
│  │                                                                       │   │
│  │  2. SourceCard 组件:                                                   │   │
│  │     ├── wasProcessing: 状态跟踪 (确保捕获完成事件)                    │   │
│  │     ├── currentStatus: 合并 API 返回状态和源初始状态                  │   │
│  │     └── useEffect: 状态流转时触发 onRefresh                           │   │
│  │                                                                       │   │
│  │  3. 状态显示:                                                          │   │
│  │     ├── new/queued/running: 动画图标 + "处理中"                      │   │
│  │     ├── completed: 绿色勾选 + 不显示状态徽章                         │   │
│  │     └── failed: 红色警告 + "失败" + 重试按钮                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 前端列视图一致性保障

### 6.1 状态配置映射

**文件位置**: `frontend/src/components/sources/SourceCard.tsx`

```typescript
const getStatusConfig = (t: TFunction) => ({
  new: {
    icon: Clock,
    color: 'text-blue-600',
    bgColor: 'bg-blue-50',
    borderColor: 'border-blue-200',
    label: t('sources.statusProcessing'),
    description: t('sources.statusPreparingDesc')
  },
  queued: {
    icon: Clock,
    color: 'text-blue-600',
    bgColor: 'bg-blue-50',
    borderColor: 'border-blue-200',
    label: t('sources.statusQueued'),
    description: t('sources.statusQueuedDesc')
  },
  running: {
    icon: Loader2,
    color: 'text-blue-600',
    bgColor: 'bg-blue-50',
    borderColor: 'border-blue-200',
    label: t('sources.statusProcessing'),
    description: t('sources.statusProcessingDesc')
  },
  completed: {
    icon: CheckCircle,
    color: 'text-green-600',
    bgColor: 'bg-green-50',
    borderColor: 'border-green-200',
    label: t('sources.statusCompleted'),
    description: t('sources.statusCompletedDesc')
  },
  failed: {
    icon: AlertTriangle,
    color: 'text-red-600',
    bgColor: 'bg-red-50',
    borderColor: 'border-red-200',
    label: t('sources.statusFailed'),
    description: t('sources.statusFailedDesc')
  }
})
```

### 6.2 状态渲染逻辑

```typescript
// 计算状态
const isProcessing: boolean = 
  currentStatus === 'new' || currentStatus === 'running' || currentStatus === 'queued'
const isFailed: boolean = currentStatus === 'failed'
const isCompleted: boolean = currentStatus === 'completed'

// 渲染状态徽章
{!isCompleted && (
  <div className="flex items-center gap-2 mb-2">
    <div className={cn(
      'flex items-center gap-1.5 px-2 py-1 rounded-md text-xs font-medium',
      statusConfig.bgColor,
      statusConfig.color
    )}>
      <StatusIcon className={cn(
        'h-3 w-3',
        isProcessing && 'animate-spin'  // 处理中时旋转动画
      )} />
      {statusLoading && shouldFetchStatus 
        ? t('sources.checking') 
        : statusConfig.label}
    </div>
    {/* ... */}
  </div>
)}

// 失败消息
{statusData?.message && (isProcessing || isFailed) && (
  <p className="text-xs text-gray-600 mb-2 italic">
    {statusData.message}
  </p>
)}

// 进度条 (如果有 progress 数据)
{isProcessing && statusData?.processing_info?.progress && (
  <div className="mt-3 pt-2 border-t">
    <div className="flex justify-between items-center mb-1">
      <span className="text-xs text-gray-600">{t('common.progress')}</span>
      <span className="text-xs text-gray-600">
        {Math.round(statusData.processing_info.progress as number)}%
      </span>
    </div>
    <div className="w-full bg-gray-200 rounded-full h-1.5">
      <div
        className="bg-blue-600 h-1.5 rounded-full transition-all duration-300"
        style={{ width: `${statusData.processing_info.progress as number}%` }}
      />
    </div>
  </div>
)}
```

### 6.3 重试按钮显示

```typescript
// 下拉菜单中的重试选项
<DropdownMenuContent align="end" className="w-48">
  {/* ... */}
  
  {isFailed && (
    <>
      <DropdownMenuItem
        onClick={(e) => {
          e.stopPropagation()
          handleRetry()
        }}
        disabled={!onRetry}
      >
        <RefreshCw className="h-4 w-4 mr-2" />
        {t('sources.retryProcessing')}
      </DropdownMenuItem>
      <DropdownMenuSeparator />
    </>
  )}
  
  {/* 删除选项 */}
</DropdownMenuContent>

// 卡片底部的重试按钮 (更醒目)
{isFailed && (
  <div className="flex gap-2 pt-2 border-t">
    <Button
      variant="outline"
      size="sm"
      onClick={handleRetry}
      disabled={!onRetry}
      className="h-7 text-xs"
    >
      <RefreshCw className="h-3 w-3 mr-1" />
      {t('sources.retry')}
    </Button>
  </div>
)}
```

### 6.4 完成后的自动刷新

```typescript
// SourceCard.tsx 中的 useEffect
useEffect(() => {
  const currentStatusFromData = statusData?.status || sourceWithStatus.status

  // 标记曾经在处理中
  if (currentStatusFromData === 'new' || 
      currentStatusFromData === 'running' || 
      currentStatusFromData === 'queued') {
    setWasProcessing(true)
  }

  // 关键：从处理中变为完成/失败时
  if (wasProcessing &&
      (currentStatusFromData === 'completed' || currentStatusFromData === 'failed')) {
    setWasProcessing(false)

    // 延迟刷新，确保后端数据已更新
    if (onRefresh) {
      setTimeout(() => onRefresh(), 500)
    }
  }
}, [statusData, sourceWithStatus.status, wasProcessing, onRefresh, source.id])
```

**设计理由**：
- `wasProcessing` 状态确保不会错过短暂的 `running → completed` 状态变化
- 500ms 延迟给数据库更新和 API 缓存留时间
- `onRefresh` 触发 `useNotebookSources` 的缓存失效和重新获取

### 6.5 列视图一致性策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    前端列视图一致性保障策略                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  策略 1: 分层状态管理                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  • 列表级: useNotebookSources (React Query)                          │   │
│  │    ├── staleTime: 5000ms (5秒后视为过期)                             │   │
│  │    ├── refetchOnWindowFocus: true (窗口聚焦时刷新)                   │   │
│  │    └── 数据: sources 数组 (扁平化的无限滚动数据)                       │   │
│  │                                                                       │   │
│  │  • 卡片级: useSourceStatus (React Query)                              │   │
│  │    ├── staleTime: 0 (总是过期)                                        │   │
│  │    ├── refetchInterval: 2000ms (处理中时)                           │   │
│  │    └── 数据: 单个源的实时状态                                         │   │
│  │                                                                       │   │
│  │  设计理由:                                                            │   │
│  │  • 列表数据相对稳定，不需要频繁刷新                                    │   │
│  │  • 状态数据需要实时更新，单独轮询                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  策略 2: 智能轮询                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  轮询触发条件:                                                         │   │
│  │  ├── source.command_id 存在 (有异步处理)                              │   │
│  │  ├── source.status 是 'new'/'queued'/'running'                       │   │
│  │  └── wasProcessing 为 true (曾经在处理中)                             │   │
│  │                                                                       │   │
│  │ 轮询停止条件:                                                          │   │
│  ├── 状态变为 'completed' 或 'failed'                                    │   │
│  └── wasProcessing 重置为 false                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  策略 3: 自动刷新                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  触发场景:                                                             │   │
│  │  1. 状态流转: running → completed/failed                               │   │
│  │     └── SourceCard 中的 useEffect 调用 onRefresh                      │   │
│  │                                                                       │   │
│  │  2. 用户操作: 点击重试按钮                                             │   │
│  │     └── useRetrySource mutation 的 onSuccess 中:                     │   │
│  │         queryClient.invalidateQueries(['sources'])                   │   │
│  │                                                                       │   │
│  │  3. 窗口聚焦: refetchOnWindowFocus: true                              │   │
│  │     └── 用户切回页面时自动刷新列表                                     │   │
│  │                                                                       │   │
│  │  4. 手动刷新: 用户下拉刷新 (未来)                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  策略 4: 乐观更新                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  操作后立即更新 UI，然后让后台刷新:                                    │   │
│  │                                                                       │   │
│  │  例如: useRetrySource                                                  │   │
│  │  onSuccess: (result, sourceId) => {                                  │   │
│  │    queryClient.invalidateQueries({ queryKey: ['sources'] })         │   │
│  │    queryClient.invalidateQueries({                                    │   │
│  │      queryKey: QUERY_KEYS.source(sourceId)                           │   │
│  │    })                                                                  │   │
│  │    toast({ title: '已重新排队...' })                                  │   │
│  │  }                                                                     │   │
│  │                                                                       │   │
│  │  设计理由:                                                            │   │
│  │  • invalidateQueries 比直接设置数据更灵活                            │   │
│  │  • 依赖 React Query 的自动重新获取                                    │   │
│  │  • 多个组件共享同一查询 key，自动同步                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 关键设计决策总结

### 7.1 重试策略决策

| 决策 | 选择 | 理由 |
|------|------|------|
| **自动重试次数** | process_source: 15 次 | SurrealDB v2 事务冲突频繁，需要更多重试 |
| **重试间隔** | 指数抖动退避 | 避免惊群效应，给系统恢复时间 |
| **不重试错误** | ValueError, ConfigurationError | 永久性问题，重试无意义 |
| **日志级别** | debug | 事务冲突重试频繁，减少日志噪音 |

### 7.2 回滚策略决策

| 决策 | 选择 | 理由 |
|------|------|------|
| **同步路径** | 完全回滚 | 同步路径阻塞 HTTP，失败后用户看不到中间状态 |
| **异步路径 - 命令提交前** | 完全回滚 | 命令提交失败，任务不会执行 |
| **异步路径 - 命令执行失败** | **不回滚** | 保留数据让用户可以手动重试，避免重新上传 |
| **上传文件** | 失败时保留 | 大文件上传成本高，重试时可复用 |

### 7.3 状态同步决策

| 决策 | 选择 | 理由 |
|------|------|------|
| **轮询间隔** | 2 秒 | 平衡实时性和服务器负载 |
| **轮询条件** | 仅处理中时轮询 | 完成/失败后不需要持续轮询 |
| **wasProcessing 状态** | 跟踪历史状态 | 确保捕获 running → completed 的短暂状态变化 |
| **FETCH command** | 列表查询时关联获取 | 减少 N+1 查询，批量获取状态 |

### 7.4 前端一致性决策

| 决策 | 选择 | 理由 |
|------|------|------|
| **分层状态** | 列表 + 卡片分离 | 列表数据稳定，状态需要实时 |
| **staleTime** | 列表: 5s, 状态: 0 | 平衡性能和一致性 |
| **自动刷新** | 状态流转 + 窗口聚焦 | 确保用户看到最新状态 |
| **乐观更新** | invalidateQueries | 组件间自动同步 |

### 7.5 代码引用索引

| 文件路径 | 说明 |
|----------|------|
| `commands/source_commands.py:49-157` | process_source 命令定义 + 重试配置 |
| `commands/embedding_commands.py:307-440` | embed_source 命令定义 + 重试配置 |
| `api/routers/sources.py:290-577` | 创建源：同步/异步路径 + 失败回滚 |
| `api/routers/sources.py:822-942` | 手动重试接口实现 |
| `api/routers/sources.py:161-286` | GET /sources: FETCH command 状态聚合 |
| `frontend/src/lib/hooks/use-sources.ts:234-259` | useSourceStatus Hook 实现 |
| `frontend/src/components/sources/SourceCard.tsx` | 状态显示 + wasProcessing 跟踪 |
| `frontend/src/app/(dashboard)/notebooks/components/SourcesColumn.tsx` | 列组件 + 无限滚动触发 |

---

## 附录：状态流转速查表

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          状态流转速查表                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  状态枚举:                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  new       → 刚创建，准备中                                           │   │
│  │  queued    → 队列中等待                                               │   │
│  │  running   → 执行中                                                   │   │
│  │  completed → 成功完成                                                 │   │
│  │  failed    → 失败 (重试已耗尽或永久性错误)                             │   │
│  │  unknown   → 无法获取状态                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  正常流转:                                                                    │
│  new → queued → running → completed                                         │
│                                                                              │
│  重试循环 (max_attempts 次):                                                 │
│  running → failed → queued → running → ... → completed/failed              │
│                                                                              │
│  前端显示:                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  new/queued/running:                                                 │   │
│  │  ├── 图标: 旋转动画 (Loader2/Clock)                                  │   │
│  │  ├── 颜色: 蓝色 (text-blue-600)                                      │   │
│  │  ├── 文字: "处理中" / "排队中"                                        │   │
│  │  └── 轮询: 每 2 秒自动刷新                                            │   │
│  │                                                                       │   │
│  │  completed:                                                           │   │
│  │  ├── 图标: 不显示状态徽章 (正常状态)                                   │   │
│  │  └── 轮询: 停止                                                       │   │
│  │                                                                       │   │
│  │  failed:                                                              │   │
│  │  ├── 图标: AlertTriangle (红色)                                       │   │
│  │  ├── 颜色: 红色 (text-red-600)                                       │   │
│  │  ├── 文字: "失败" + 错误消息                                          │   │
│  │  ├── 按钮: 显示"重试"按钮                                             │   │
│  │  └── 轮询: 停止 (需手动重试)                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
