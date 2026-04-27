# Open-Notebook Checkpoint 泄漏缺陷深度分析报告

## 1. 缺陷概述

### 1.1 问题描述

在 R3 报告中发现的关键设计缺陷：

> **ChatSession 删除时，SurrealDB 记录被清理但 SqliteSaver checkpoint 残留不清除。**

这意味着：
- 用户删除聊天会话后，SurrealDB 中的元数据记录被删除
- 但 SQLite 数据库中的 LangGraph checkpoint（包含完整消息历史）**仍然存在**
- 这可能导致数据泄漏、隐私问题和不一致的用户体验

### 1.2 涉及的代码位置

| 组件 | 文件路径 | 问题 |
|------|---------|------|
| 会话删除 | `api/routers/chat.py:306-327` | 只删除 SurrealDB，不清理 SQLite |
| 源会话删除 | `api/routers/source_chat.py:362-414` | 只删除 SurrealDB，不清理 SQLite |
| Checkpoint 存储 | `open_notebook/graphs/chat.py:88-92` | 全局 SQLite 连接 |
| 工具函数 | `open_notebook/utils/graph_utils.py:7-23` | 只读操作，无清理能力 |

---

## 2. Session ID 生成机制分析

### 2.1 SurrealDB 自动 ID 生成

项目使用 SurrealDB 的自动 ID 生成机制：

```python
# open_notebook/database/repository.py:85-103
async def repo_create(table: str, data: Dict[str, Any]) -> Dict[str, Any]:
    """Create a new record in the specified table"""
    # Remove 'id' attribute if it exists in data
    data.pop("id", None)  # ← 强制移除 ID，让 SurrealDB 自动生成
    data["created"] = datetime.now(timezone.utc)
    data["updated"] = datetime.now(timezone.utc)
    
    async with db_connection() as connection:
        result = parse_record_ids(await connection.insert(table, data))
        # SurrealDB 的 insert 方法会自动生成 ID
        # 格式为：table:random_id
        # 例如：chat_session:8x7f3z9k2y1w
```
**[repository.py:85-103](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/open_notebook/database/repository.py#L85-L103)**

### 2.2 ID 格式与碰撞概率

**SurrealDB ID 格式**：
```
chat_session:8x7f3z9k2y1w
└──────┬──────┘ └─────┬─────┘
   表名前缀        随机部分
```

**随机 ID 特性**：
- SurrealDB 使用类似 **nanoid** 的随机字符串生成算法
- 默认长度通常为 12-16 个字符
- 字符集：`A-Za-z0-9_-` (约 64 个字符)
- 碰撞概率：**极低**（12 字符的碰撞概率约为 1/2^72）

### 2.3 代码中无手动指定 ID 的逻辑

检查所有会话创建代码：

```python
# api/routers/chat.py:137-172
@router.post("/chat/sessions", response_model=ChatSessionResponse)
async def create_session(request: CreateSessionRequest):
    # ...
    session = ChatSession(
        title=request.title or f"Chat Session {asyncio.get_event_loop().time():.0f}",
        model_override=request.model_override,
        # ← 注意：没有指定 id 字段！
    )
    await session.save()  # 让 SurrealDB 自动生成 ID
    # ...
```
**[chat.py:137-172](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/api/routers/chat.py#L137-L172)**

**结论**：
- ✅ 正常情况下，SurrealDB 生成的随机 ID 不会重复
- ✅ 代码中没有手动指定 ID 的逻辑
- ⚠️ 但这并不意味着缺陷不存在，只是表现形式不同

---

## 3. 相同 Session ID 重建的风险分析

### 3.1 正常场景：ID 不会重复

在正常使用流程中：

```
时间线：
T0: 用户创建会话 A
    → SurrealDB 生成 ID: chat_session:abc123
    → SQLite 无操作（惰性创建）

T1: 用户发送第一条消息
    → graph.invoke() 执行
    → SQLite 创建 checkpoint: thread_id = chat_session:abc123
    → 消息历史被保存

T2: 用户删除会话
    → SurrealDB 删除 chat_session:abc123 记录
    → SQLite 中的 checkpoint **仍然存在** ⚠️

T3: 用户创建新会话 B
    → SurrealDB 生成新 ID: chat_session:xyz789 (新随机 ID)
    → SQLite 无操作（新的惰性创建）

T4: 用户发送消息给会话 B
    → 使用新 ID chat_session:xyz789
    → 不会访问旧的 checkpoint
```

**正常场景结论**：
- ✅ 由于 SurrealDB 生成的随机 ID 不会重复
- ✅ 新会话不会意外访问旧的 checkpoint

### 3.2 风险场景 1：手动操作数据库

如果管理员或开发者**手动操作** SurrealDB：

```
风险场景：
T0: 用户创建会话，ID = chat_session:test_001
    → (假设通过某种方式手动指定了 ID)

T1: 用户删除会话
    → SurrealDB 删除 chat_session:test_001
    → SQLite checkpoint 残留

T2: 管理员手动创建相同 ID 的记录
    → 可能通过直接 SQL：CREATE chat_session:test_001 CONTENT {...}
    → 或通过 API 重新创建同名会话

T3: 新会话发送消息
    → graph.invoke(config={"thread_id": "chat_session:test_001"})
    → SqliteSaver 加载**旧的 checkpoint**！
    → 旧消息历史被意外恢复！⚠️
```

**代码证据**：API 层确实支持传入特定 ID 的场景（虽然不是在创建时）：

```python
# api/routers/chat.py:175-247
@router.get("/chat/sessions/{session_id}", ...)
async def get_session(session_id: str):
    # session_id 来自 URL 路径参数
    # 可以是任何字符串，包括手动构造的
    full_session_id = (
        session_id
        if session_id.startswith("chat_session:")
        else f"chat_session:{session_id}"  # ← 支持任意 ID
    )
    
    # SqliteSaver 直接使用这个 ID
    thread_state = await asyncio.to_thread(
        chat_graph.get_state,
        config=RunnableConfig(configurable={"thread_id": full_session_id}),
    )
```
**[chat.py:175-247](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/api/routers/chat.py#L175-L247)**

### 3.3 风险场景 2：数据恢复/备份

在数据备份和恢复场景中：

```
风险场景：
T0: 系统备份
    → SurrealDB 备份包含 chat_session:abc123
    → SQLite 备份包含对应的 checkpoint

T1: 用户删除会话 chat_session:abc123
    → SurrealDB 记录删除
    → SQLite checkpoint 残留（但可能被忽略）

T2: 系统从备份恢复
    → SurrealDB 恢复 chat_session:abc123 记录
    → SQLite 可能也恢复，或残留的 checkpoint 仍然存在

T3: 用户访问"恢复的"会话
    → 访问的实际上是**删除前的旧消息**
    → 用户可能以为这是新会话，但看到的是旧数据
```

### 3.4 风险场景 3：ID 枚举攻击

虽然概率极低，但理论上存在：

```
风险场景（理论上）：
攻击者猜测或枚举 session_id
    → 尝试访问 /api/chat/sessions/{guessed_id}
    
如果 SurrealDB 中不存在该 ID：
    → API 返回 404
    
但 SQLite 中可能存在残留的 checkpoint：
    → 虽然 API 不会直接返回这些消息
    → 但如果存在其他漏洞，可能被利用
```

---

## 4. Graph Utils 工具函数分析

### 4.1 函数定义

```python
# open_notebook/utils/graph_utils.py:7-23
async def get_session_message_count(graph, session_id: str) -> int:
    """Get message count from LangGraph state, returns 0 on error."""
    try:
        # Use sync get_state() in a thread (SqliteSaver doesn't support async)
        thread_state = await asyncio.to_thread(
            graph.get_state,
            config=RunnableConfig(configurable={"thread_id": session_id}),
        )
        if (
            thread_state
            and thread_state.values
            and "messages" in thread_state.values
        ):
            return len(thread_state.values["messages"])
    except Exception as e:
        logger.warning(f"Could not fetch message count for session {session_id}: {e}")
    return 0  # ← 出错时返回 0，静默失败
```
**[graph_utils.py:7-23](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/open_notebook/utils/graph_utils.py#L7-L23)**

### 4.2 调用位置与时机

| 调用位置 | 文件路径 | 调用场景 | 目的 |
|---------|---------|---------|------|
| 会话列表 | `api/routers/chat.py:113` | `GET /api/chat/sessions` | 显示每个会话的消息数 |
| 会话详情 | `api/routers/chat.py:288` | `GET /api/chat/sessions/{id}` | 显示消息数 |
| 源会话列表 | `api/routers/source_chat.py:165` | `GET /api/sources/{id}/chat/sessions` | 显示消息数 |
| 源会话更新 | `api/routers/source_chat.py:342` | `PUT /api/sources/{id}/chat/sessions/{id}` | 更新后显示消息数 |

### 4.3 双存储协调机制

**查询会话列表时的数据流**：

```
GET /api/chat/sessions?notebook_id=nb_001
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 1 步：SurrealDB 查询                                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ session_list = await notebook.get_chat_sessions()       ││
│  │                                                          ││
│  │ SurrealQL:                                                ││
│  │ SELECT * FROM (                                           ││
│  │   SELECT <-chat_session AS chat_session                  ││
│  │   FROM refers_to WHERE out = $notebook_id                ││
│  │   FETCH chat_session                                      ││
│  │ ) ORDER BY chat_session.updated DESC                      ││
│  └─────────────────────────────────────────────────────────┘│
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 2 步：对每个会话，查询 SqliteSaver                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ for session in session_list:                             ││
│  │     msg_count = await get_session_message_count(         ││
│  │         chat_graph,                                       ││
│  │         session.id  # ← 使用 SurrealDB 的 ID            ││
│  │     )                                                     ││
│  │                                                          ││
│  │ 内部实现：                                                 ││
│  │ thread_state = await asyncio.to_thread(                  ││
│  │     chat_graph.get_state,                                 ││
│  │     config=RunnableConfig(                                ││
│  │         configurable={"thread_id": session.id}           ││
│  │     )                                                      ││
│  │ )                                                          ││
│  └─────────────────────────────────────────────────────────┘│
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 3 步：整合响应                                           │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ ChatSessionResponse(                                     ││
│  │     id=session.id,              ← SurrealDB             ││
│  │     title=session.title,          ← SurrealDB             ││
│  │     created=session.created,      ← SurrealDB             ││
│  │     updated=session.updated,      ← SurrealDB             ││
│  │     message_count=msg_count,      ← SqliteSaver ⚠️       ││
│  │     model_override=session.model_override  ← SurrealDB   ││
│  │ )                                                          ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 4.4 静默失败的问题

`get_session_message_count` 有一个**潜在问题**：

```python
# open_notebook/utils/graph_utils.py:7-23
async def get_session_message_count(graph, session_id: str) -> int:
    try:
        # ... 尝试获取状态 ...
        return len(thread_state.values["messages"])
    except Exception as e:
        logger.warning(f"Could not fetch message count for session {session_id}: {e}")
    return 0  # ← 任何错误都返回 0
```
**[graph_utils.py:7-23](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/open_notebook/utils/graph_utils.py#L7-L23)**

**问题场景**：

| 实际情况 | 返回值 | 用户看到的 |
|---------|-------|-----------|
| SQLite 中没有 checkpoint（新会话） | 0 | ✅ 正确：0 条消息 |
| SQLite 连接失败 | 0 | ⚠️ 错误：显示 0，但实际可能有消息 |
| 其他异常 | 0 | ⚠️ 静默失败 |

**更严重的问题**：如果 SurrealDB 中存在但 SQLite 中不存在（或相反），没有一致性检查。

---

## 5. 并发请求竞态风险分析

### 5.1 SurrealDB 层的并发处理

代码中已经有事务冲突处理：

```python
# open_notebook/database/repository.py:65-82
async def repo_query(
    query_str: str, vars: Optional[Dict[str, Any]] = None
) -> List[Dict[str, Any]]:
    async with db_connection() as connection:
        try:
            result = parse_record_ids(await connection.query(query_str, vars))
            # ...
        except RuntimeError as e:
            # RuntimeError is raised for retriable transaction conflicts
            # - log at debug to avoid noise
            logger.debug(str(e))  # ← 事务冲突只记录 debug
            raise  # ← 重新抛出，让上层处理
        except Exception as e:
            logger.exception(e)
            raise
```
**[repository.py:65-82](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/open_notebook/database/repository.py#L65-L82)**

```python
# open_notebook/domain/base.py:188-190
except RuntimeError:
    # Transaction conflicts should propagate for retry
    raise  # ← 让调用者处理重试
```
**[base.py:188-190](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/open_notebook/domain/base.py#L188-L190)**

### 5.2 SqliteSaver 层的并发风险

**SQLite 的特性**：
- SQLite 是文件级数据库
- 默认情况下，多线程访问需要特殊配置
- 写入操作会锁定整个数据库

**代码中的连接配置**：

```python
# open_notebook/graphs/chat.py:88-92
conn = sqlite3.connect(
    LANGGRAPH_CHECKPOINT_FILE,
    check_same_thread=False,  # ← 允许跨线程访问
)
memory = SqliteSaver(conn)
```
**[chat.py:88-92](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/open_notebook/graphs/chat.py#L88-L92)**

```python
# open_notebook/graphs/source_chat.py:244-248
conn = sqlite3.connect(
    LANGGRAPH_CHECKPOINT_FILE,
    check_same_thread=False,  # ← 另一个连接！
)
memory = SqliteSaver(conn)
```
**[source_chat.py:244-248](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/open_notebook/graphs/source_chat.py#L244-L248)**

### 5.3 潜在的并发问题

**问题 1：多个独立的 SQLite 连接**

```
┌─────────────────────────────────────────────────────────────┐
│  模块 1：chat.py                                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ conn = sqlite3.connect(                                   ││
│  │     LANGGRAPH_CHECKPOINT_FILE,                            ││
│  │     check_same_thread=False                               ││
│  │ )  ← 连接 A                                               ││
│  │                                                            ││
│  │ graph = agent_state.compile(checkpointer=SqliteSaver(conn))││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  模块 2：source_chat.py                                      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ conn = sqlite3.connect(                                   ││
│  │     LANGGRAPH_CHECKPOINT_FILE,                            ││
│  │     check_same_thread=False                               ││
│  │ )  ← 连接 B（独立于连接 A）                               ││
│  │                                                            ││
│  │ source_chat_graph = ...compile(checkpointer=SqliteSaver(conn))││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

**风险**：
- SQLite 在多连接写入时可能出现数据库锁
- `check_same_thread=False` 允许跨线程，但不解决多连接的锁问题
- 没有 WAL (Write-Ahead Logging) 模式的配置

**问题 2：没有事务隔离**

```python
# 典型的消息执行流程
@router.post("/chat/execute")
async def execute_chat(request: ExecuteChatRequest):
    # 1. 读取状态（可能并发）
    current_state = await asyncio.to_thread(
        chat_graph.get_state,  # ← 读取
        config=RunnableConfig(configurable={"thread_id": full_session_id}),
    )
    
    # 2. 修改状态（内存中）
    state_values["messages"].append(user_message)
    
    # 3. 执行并保存（可能并发）
    result = chat_graph.invoke(  # ← 写入
        input=state_values,
        config=RunnableConfig(configurable={"thread_id": full_session_id}),
    )
```

**竞态场景**：

```
时间线（同一会话的并发请求）：

T0: 请求 A 读取状态 → messages = [msg1]
T1: 请求 B 读取状态 → messages = [msg1]  ← 相同的起始状态
T2: 请求 A 添加 msg2 → messages = [msg1, msg2]
T3: 请求 B 添加 msg3 → messages = [msg1, msg3]  ← 丢失了 msg2！
T4: 请求 A 执行并保存 → checkpoint 保存 [msg1, msg2]
T5: 请求 B 执行并保存 → checkpoint 覆盖为 [msg1, msg3]  ⚠️

结果：msg2 永久丢失！
```

### 5.4 实际风险评估

| 风险类型 | 可能性 | 严重程度 | 说明 |
|---------|-------|---------|------|
| SurrealDB 事务冲突 | 中 | 低 | 已有处理机制，会重新抛出 |
| SQLite 数据库锁 | 中 | 中 | 多连接写入可能超时或失败 |
| 同一会话并发写入 | 低 | 高 | 消息丢失，但前端通常会阻止 |
| 双存储不一致 | 高 | 中 | 本报告的核心问题 |

---

## 6. 数据不一致的具体表现

### 6.1 不一致场景矩阵

| 场景 | SurrealDB | SqliteSaver | 用户体验 |
|------|-----------|-------------|---------|
| **正常新会话** | 有记录 | 无 checkpoint | 显示 0 条消息 ✅ |
| **发送消息后** | 有记录 | 有 checkpoint | 显示 N 条消息 ✅ |
| **删除会话后** | 无记录 | 有 checkpoint | 会话消失 ✅（但数据泄漏） |
| **SQLite 损坏/丢失** | 有记录 | 无 checkpoint | 显示 0 条消息 ⚠️（数据不一致） |
| **SurrealDB 恢复** | 有记录 | 有旧 checkpoint | 显示旧消息 ⚠️（意外恢复） |
| **并发写入** | 有记录 | 可能覆盖 | 消息丢失 ⚠️ |

### 6.2 删除后的数据残留

**会话删除时的实际操作**：

```python
# api/routers/chat.py:306-327
@router.delete("/chat/sessions/{session_id}", response_model=SuccessResponse)
async def delete_session(session_id: str):
    try:
        # ... 验证会话存在 ...
        
        # 只删除 SurrealDB
        await session.delete()  # ← repo_delete(record_id)
        
        # ⚠️ 没有清理 SQLite！
        
        return SuccessResponse(success=True, message="Session deleted successfully")
    except Exception as e:
        # ...
```
**[chat.py:306-327](file:///g:/fangzheng/solo-dogfeeding/code/open-notebook-7069/api/routers/chat.py#L306-L327)**

**SQLite 中残留的数据**：

```
SQLite checkpoint 表（简化）：
┌─────────────────────┬──────────────────────────────────────────┐
│ thread_id (PK)      │ state (JSON)                             │
├─────────────────────┼──────────────────────────────────────────┤
│ chat_session:abc123 │ {"messages": [...], "context": ...}    │ ← 删除后残留！
│ chat_session:xyz789 │ {"messages": [...], "context_indicators": ...} │
│ ...                 │ ...                                      │
└─────────────────────┴──────────────────────────────────────────┘
```

### 6.3 隐私影响

**残留数据包含**：
- ✅ 完整的消息历史（用户和 AI 的所有对话）
- ✅ 上下文数据（引用的源文件、笔记、洞察）
- ✅ 上下文指示器（哪些源文件被引用）
- ✅ 模型覆盖配置
- ⚠️ 可能包含敏感信息（用户的私人笔记、研究内容等）

**合规风险**：
- 如果用户要求删除数据（GDPR "被遗忘权"）
- 实际上消息历史仍然存在于 SQLite 中
- 这可能导致合规问题

---

## 7. 修复建议

### 7.1 立即修复：删除时清理 Checkpoint

**方案 A：添加 SQLite 清理逻辑**

```python
# 建议的修复代码
import sqlite3
from open_notebook.config import LANGGRAPH_CHECKPOINT_FILE

async def delete_session(session_id: str):
    try:
        # 1. 规范化 ID
        full_session_id = (
            session_id
            if session_id.startswith("chat_session:")
            else f"chat_session:{session_id}"
        )
        
        # 2. 验证并获取 SurrealDB 记录
        session = await ChatSession.get(full_session_id)
        if not session:
            raise HTTPException(status_code=404, detail="Session not found")
        
        # 3. 删除 SurrealDB 记录
        await session.delete()
        
        # 4. 新增：清理 SQLite checkpoint ⚠️
        try:
            conn = sqlite3.connect(LANGGRAPH_CHECKPOINT_FILE)
            cursor = conn.cursor()
            
            # LangGraph SqliteSaver 的表结构：
            # - checkpoint_store: 主表
            # - writes: 写入日志
            # 或者查看实际的表名
            
            # 方案 1：直接删除（如果知道表结构）
            # cursor.execute("DELETE FROM checkpoint_store WHERE thread_id = ?", (full_session_id,))
            
            # 方案 2：使用 LangGraph 官方 API（如果可用）
            # 注意：SqliteSaver 可能没有公开的删除 API
            
            # 方案 3：执行原始 SQL（需要确认表结构）
            # 先查询表名
            cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
            tables = cursor.fetchall()
            
            for (table_name,) in tables:
                # 检查是否有 thread_id 列
                cursor.execute(f"PRAGMA table_info({table_name})")
                columns = cursor.fetchall()
                column_names = [col[1] for col in columns]
                
                if 'thread_id' in column_names:
                    cursor.execute(
                        f"DELETE FROM {table_name} WHERE thread_id = ?",
                        (full_session_id,)
                    )
            
            conn.commit()
            conn.close()
            logger.info(f"Cleaned up checkpoint for session {full_session_id}")
            
        except Exception as e:
            logger.error(f"Failed to clean up checkpoint for session {full_session_id}: {e}")
            # 注意：不要因为 SQLite 清理失败而回滚 SurrealDB 删除
            # 但应该记录日志以便后续处理
        
        return SuccessResponse(success=True, message="Session deleted successfully")
        
    except NotFoundError:
        raise HTTPException(status_code=404, detail="Session not found")
    except Exception as e:
        logger.error(f"Error deleting session: {str(e)}")
        raise HTTPException(status_code=500, detail=f"Error deleting session: {str(e)}")
```

### 7.2 方案 B：封装 Checkpoint 管理

创建一个统一的 checkpoint 管理模块：

```python
# open_notebook/utils/checkpoint_manager.py
import sqlite3
from contextlib import contextmanager
from typing import Optional

from langgraph.checkpoint.sqlite import SqliteSaver
from loguru import logger

from open_notebook.config import LANGGRAPH_CHECKPOINT_FILE


class CheckpointManager:
    """统一管理 LangGraph checkpoint 的创建、读取、删除"""
    
    _instance: Optional["CheckpointManager"] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance
    
    def __init__(self):
        if self._initialized:
            return
        self._initialized = True
        self._db_path = LANGGRAPH_CHECKPOINT_FILE
    
    @contextmanager
    def _get_connection(self):
        """获取数据库连接（使用 WAL 模式）"""
        conn = sqlite3.connect(self._db_path, check_same_thread=False)
        # 启用 WAL 模式以提高并发性
        conn.execute("PRAGMA journal_mode=WAL")
        conn.execute("PRAGMA synchronous=NORMAL")
        conn.execute("PRAGMA busy_timeout=30000")  # 30 秒超时
        try:
            yield conn
        finally:
            conn.close()
    
    def delete_checkpoint(self, thread_id: str) -> bool:
        """
        删除指定 thread_id 的所有 checkpoint 数据
        
        Args:
            thread_id: 会话 ID（格式：chat_session:xxx）
        
        Returns:
            bool: 是否成功删除
        """
        try:
            with self._get_connection() as conn:
                cursor = conn.cursor()
                
                # 获取所有表名
                cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
                tables = cursor.fetchall()
                
                deleted_count = 0
                for (table_name,) in tables:
                    # 检查表结构
                    cursor.execute(f"PRAGMA table_info({table_name})")
                    columns = cursor.fetchall()
                    column_names = [col[1] for col in columns]
                    
                    # 删除包含 thread_id 列的表中的相关记录
                    if 'thread_id' in column_names:
                        cursor.execute(
                            f"DELETE FROM {table_name} WHERE thread_id = ?",
                            (thread_id,)
                        )
                        deleted_count += cursor.rowcount
                
                conn.commit()
                
                if deleted_count > 0:
                    logger.info(f"Deleted {deleted_count} checkpoint records for thread_id: {thread_id}")
                else:
                    logger.debug(f"No checkpoint records found for thread_id: {thread_id}")
                
                return True
                
        except Exception as e:
            logger.error(f"Failed to delete checkpoint for thread_id {thread_id}: {e}")
            return False
    
    def get_checkpoint_exists(self, thread_id: str) -> bool:
        """检查指定 thread_id 的 checkpoint 是否存在"""
        try:
            with self._get_connection() as conn:
                cursor = conn.cursor()
                
                cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
                tables = cursor.fetchall()
                
                for (table_name,) in tables:
                    cursor.execute(f"PRAGMA table_info({table_name})")
                    columns = cursor.fetchall()
                    column_names = [col[1] for col in columns]
                    
                    if 'thread_id' in column_names:
                        cursor.execute(
                            f"SELECT 1 FROM {table_name} WHERE thread_id = ? LIMIT 1",
                            (thread_id,)
                        )
                        if cursor.fetchone():
                            return True
                
                return False
                
        except Exception as e:
            logger.error(f"Failed to check checkpoint existence for {thread_id}: {e}")
            return False
    
    def verify_consistency(self, session_id: str) -> dict:
        """
        验证 SurrealDB 和 SqliteSaver 的一致性
        
        Returns:
            dict: 包含一致性状态的字典
        """
        from open_notebook.domain.notebook import ChatSession
        
        result = {
            "session_id": session_id,
            "surrealdb_exists": False,
            "sqlite_exists": False,
            "consistent": False,
            "message_count": None,
        }
        
        try:
            # 检查 SurrealDB
            try:
                full_id = session_id if session_id.startswith("chat_session:") else f"chat_session:{session_id}"
                session = await ChatSession.get(full_id)
                result["surrealdb_exists"] = True
            except Exception:
                result["surrealdb_exists"] = False
            
            # 检查 SQLite
            result["sqlite_exists"] = self.get_checkpoint_exists(full_id)
            
            # 判断一致性
            # 正常情况：都存在 或 都不存在
            # 不一致：一个存在另一个不存在
            result["consistent"] = (result["surrealdb_exists"] == result["sqlite_exists"])
            
            return result
            
        except Exception as e:
            logger.error(f"Consistency check failed for {session_id}: {e}")
            result["error"] = str(e)
            return result


# 全局单例
checkpoint_manager = CheckpointManager()
```

### 7.3 方案 C：定期清理任务

创建定期任务清理孤立的 checkpoint：

```python
# open_notebook/tasks/checkpoint_cleanup.py
import asyncio
from loguru import logger

from open_notebook.database.repository import repo_query
from open_notebook.utils.checkpoint_manager import checkpoint_manager


async def cleanup_orphaned_checkpoints():
    """
    清理孤立的 checkpoint（SurrealDB 中不存在但 SQLite 中存在）
    
    建议定期执行（例如每天一次）
    """
    logger.info("Starting orphaned checkpoint cleanup...")
    
    try:
        # 1. 获取所有现存的会话 ID（从 SurrealDB）
        # 注意：需要查询所有 chat_session 记录
        from open_notebook.domain.notebook import ChatSession
        
        # 获取所有会话
        all_sessions = await repo_query("SELECT id FROM chat_session")
        existing_session_ids = set()
        
        for record in all_sessions:
            session_id = record.get("id")
            if session_id:
                # 确保是完整格式
                full_id = str(session_id) if str(session_id).startswith("chat_session:") else f"chat_session:{session_id}"
                existing_session_ids.add(full_id)
        
        logger.info(f"Found {len(existing_session_ids)} active sessions in SurrealDB")
        
        # 2. 获取 SQLite 中的所有 thread_id
        # 注意：这需要直接查询 SQLite
        import sqlite3
        from open_notebook.config import LANGGRAPH_CHECKPOINT_FILE
        
        conn = sqlite3.connect(LANGGRAPH_CHECKPOINT_FILE)
        cursor = conn.cursor()
        
        # 获取所有表
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
        tables = cursor.fetchall()
        
        checkpoint_thread_ids = set()
        
        for (table_name,) in tables:
            cursor.execute(f"PRAGMA table_info({table_name})")
            columns = cursor.fetchall()
            column_names = [col[1] for col in columns]
            
            if 'thread_id' in column_names:
                cursor.execute(f"SELECT DISTINCT thread_id FROM {table_name}")
                for (thread_id,) in cursor.fetchall():
                    if thread_id and thread_id.startswith("chat_session:"):
                        checkpoint_thread_ids.add(thread_id)
        
        conn.close()
        
        logger.info(f"Found {len(checkpoint_thread_ids)} checkpoints in SQLite")
        
        # 3. 找出孤立的 checkpoint
        orphaned_ids = checkpoint_thread_ids - existing_session_ids
        
        if orphaned_ids:
            logger.warning(f"Found {len(orphaned_ids)} orphaned checkpoints to clean up")
            
            # 4. 删除孤立的 checkpoint
            deleted_count = 0
            for thread_id in orphaned_ids:
                if checkpoint_manager.delete_checkpoint(thread_id):
                    deleted_count += 1
            
            logger.info(f"Cleaned up {deleted_count} orphaned checkpoints")
        else:
            logger.info("No orphaned checkpoints found")
        
        return {
            "active_sessions": len(existing_session_ids),
            "total_checkpoints": len(checkpoint_thread_ids),
            "orphaned_checkpoints": len(orphaned_ids),
            "cleaned_checkpoints": deleted_count if orphaned_ids else 0,
        }
        
    except Exception as e:
        logger.error(f"Orphaned checkpoint cleanup failed: {e}")
        logger.exception(e)
        raise


if __name__ == "__main__":
    # 手动运行清理
    asyncio.run(cleanup_orphaned_checkpoints())
```

### 7.4 修复方案总结

| 方案 | 优点 | 缺点 | 优先级 |
|------|------|------|--------|
| **A: 删除时清理** | 即时生效，防止泄漏 | 需要修改多个路由文件 | 🔴 高 |
| **B: CheckpointManager** | 统一管理，可扩展 | 需要新增模块 | 🟡 中 |
| **C: 定期清理任务** | 兜底方案，处理历史数据 | 不能实时防止泄漏 | 🟢 低 |

**建议实施顺序**：
1. 首先实施方案 A（删除时清理）
2. 然后实施方案 C（定期清理）处理历史数据
3. 最后实施方案 B（统一管理）作为长期架构改进

---

## 8. 关键文件索引

| 文件路径 | 职责 | 问题点 |
|---------|------|--------|
| `api/routers/chat.py:306-327` | 会话删除 | 只删 SurrealDB，不删 SQLite |
| `api/routers/source_chat.py:362-414` | 源会话删除 | 只删 SurrealDB，不删 SQLite |
| `open_notebook/graphs/chat.py:88-92` | Chat Graph | 全局 SQLite 连接 |
| `open_notebook/graphs/source_chat.py:244-248` | Source Chat Graph | 另一个独立的 SQLite 连接 |
| `open_notebook/utils/graph_utils.py:7-23` | 消息计数工具 | 静默失败，返回 0 |
| `open_notebook/database/repository.py:65-82` | 数据库查询 | 事务冲突处理 |
| `open_notebook/domain/base.py:188-190` | 模型基类 | 事务冲突传播 |

---

## 9. 结论

### 9.1 缺陷严重性评估

| 维度 | 评估 | 说明 |
|------|------|------|
| **功能正确性** | 🟡 中 | 用户看不到残留数据，功能上"看起来正常" |
| **数据一致性** | 🔴 高 | 双存储不一致，可能导致意外恢复旧数据 |
| **隐私安全** | 🔴 高 | 删除后消息仍然存在，违反"被遗忘权" |
| **合规风险** | 🟡 中 | 如果有 GDPR 等合规要求，可能存在问题 |
| **用户体验** | 🟢 低 | 用户通常不会察觉 |

### 9.2 关键发现

1. **正常情况下不会意外恢复**：
   - SurrealDB 生成的随机 ID 碰撞概率极低
   - 新会话不会意外访问旧 checkpoint

2. **但存在特定风险场景**：
   - 手动操作数据库（DBA 或开发者）
   - 数据备份恢复
   - 未来可能的 ID 枚举攻击

3. **并发风险**：
   - SQLite 多连接写入可能导致锁问题
   - 同一会话并发写入可能导致消息丢失（但前端通常阻止）

4. **核心问题**：
   - **数据泄漏**：删除后消息历史仍然存在
   - **不一致性**：两套存储之间没有协调机制
   - **无清理**：没有任何机制清理过期/删除的 checkpoint

### 9.3 最终建议

**立即行动**：
1. ✅ 在会话删除时添加 SQLite checkpoint 清理逻辑
2. ✅ 添加定期清理任务处理历史数据

**长期改进**：
1. 考虑使用支持异步和更好并发的 checkpoint 后端（PostgreSQL, Redis）
2. 实现双存储的一致性检查机制
3. 添加数据过期策略（自动清理旧会话）

---

*报告生成时间：2026-04-27*
*基于 open-notebook 项目代码分析*
