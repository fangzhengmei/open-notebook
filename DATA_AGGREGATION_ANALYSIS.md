# Open Notebook 数据聚合与列视图驱动分析报告

> 分析日期：2026-05-01  
> 分析范围：笔记本、信息源、笔记三类数据的聚合逻辑与前端列视图驱动机制

---

## 目录

1. [架构概览](#1-架构概览)
2. [后端数据聚合逻辑](#2-后端数据聚合逻辑)
3. [API 响应结构](#3-api-响应结构)
4. [前端状态管理](#4-前端状态管理)
5. [前端列视图渲染](#5-前端列视图渲染)
6. [完整数据流图](#6-完整数据流图)

---

## 1. 架构概览

### 1.1 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **数据库** | SurrealDB | 图数据库，支持边关系和图遍历查询 |
| **后端** | Python + FastAPI | 异步 Web 框架，Pydantic 数据验证 |
| **前端** | Next.js + React | 服务端渲染，React 18 |
| **状态管理** | TanStack Query (React Query) + Zustand | 服务端状态 + 客户端状态 |
| **UI 组件** | shadcn/ui + Tailwind CSS | 组件库 + 样式方案 |

### 1.2 三层数据模型

```
┌─────────────────────────────────────────────────────────────┐
│                      NOTEBOOK (容器层)                        │
│  作用：研究项目的隔离容器，提供上下文边界                        │
│  特性：隔离性、可存档、独立的 AI 上下文                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │   SOURCES       │         │    NOTES        │           │
│  │   (信息源)       │◄───────►│    (笔记)        │           │
│  │                 │  引用    │                 │           │
│  │  原始材料：       │         │  处理结果：       │           │
│  │  - PDF 文档      │         │  - 手动记录       │           │
│  │  - Web 链接      │         │  - AI 摘要       │           │
│  │  - 音视频文件    │         │  - 转换结果       │           │
│  │  - 纯文本        │         │  - 聊天保存       │           │
│  └─────────────────┘         └─────────────────┘           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 后端数据聚合逻辑

### 2.1 核心数据模型定义

**文件位置**: `open_notebook/domain/notebook.py`

#### 2.1.1 Notebook 模型

```python
class Notebook(ObjectModel):
    table_name: ClassVar[str] = "notebook"
    name: str                          # 笔记本名称
    description: str                   # 描述（提供 AI 上下文）
    archived: Optional[bool] = False  # 归档状态
    
    # 关联查询方法
    async def get_sources(self) -> List["Source"]:
        # 通过 reference 边关系查询关联的信息源
        srcs = await repo_query(
            """
            select * omit source.full_text from (
            select in as source from reference where out=$id
            fetch source
        ) order by source.updated desc
        """, {"id": ensure_record_id(self.id)},
        )
        return [Source(**src["source"]) for src in srcs] if srcs else []

    async def get_notes(self) -> List["Note"]:
        # 通过 artifact 边关系查询关联的笔记
        srcs = await repo_query(
            """
            select * omit note.content, note.embedding from (
                select in as note from artifact where out=$id
                fetch note
            ) order by note.updated desc
        """, {"id": ensure_record_id(self.id)},
        )
        return [Note(**src["note"]) for src in srcs] if srcs else []
```

#### 2.1.2 Source 模型

```python
class Source(ObjectModel):
    table_name: ClassVar[str] = "source"
    asset: Optional[Asset] = None      # 资源信息（文件路径或 URL）
    title: Optional[str] = None         # 标题
    topics: Optional[List[str]] = Field(default_factory=list)  # 主题标签
    full_text: Optional[str] = None     # 提取的完整文本
    command: Optional[Union[str, RecordID]]  # 关联的处理命令（异步处理）

    # 关联方法
    async def add_to_notebook(self, notebook_id: str) -> Any:
        # 创建 reference 边关系
        return await self.relate("reference", notebook_id)
    
    async def get_insights(self) -> List[SourceInsight]:
        # 获取 AI 生成的洞察
        result = await repo_query(
            "SELECT * FROM source_insight WHERE source=$id",
            {"id": ensure_record_id(self.id)},
        )
        return [SourceInsight(**insight) for insight in result]
```

#### 2.1.3 Note 模型

```python
class Note(ObjectModel):
    table_name: ClassVar[str] = "note"
    title: Optional[str] = None                    # 笔记标题
    note_type: Optional[Literal["human", "ai"]] = None  # 类型区分
    content: Optional[str] = None                  # 内容
    
    async def add_to_notebook(self, notebook_id: str) -> Any:
        # 创建 artifact 边关系
        return await self.relate("artifact", notebook_id)
```

### 2.2 图数据库关系模型

SurrealDB 使用边（Edge）表来建立实体间的关系：

| 边表名 | 源实体 | 目标实体 | 关系含义 |
|--------|--------|----------|----------|
| `reference` | Source | Notebook | 信息源属于/引用笔记本 |
| `artifact` | Note | Notebook | 笔记属于笔记本 |
| `refers_to` | ChatSession | Notebook | 对话会话引用笔记本 |

**关系图**:

```
Source ──(reference)──► Notebook ◄──(artifact)── Note
   │                                           ▲
   │                                           │
   └──(source_embedding)──► SourceEmbedding   │
   │                                           │
   └──(source_insight)──► SourceInsight ──────┘
                                              (可保存为 Note)
```

### 2.3 聚合查询实现

**文件位置**: `api/routers/notebooks.py` 和 `api/routers/sources.py`

#### 2.3.1 笔记本列表聚合查询

```python
@router.get("/notebooks", response_model=List[NotebookResponse])
async def get_notebooks(...):
    # 使用 SurrealQL 的图遍历语法一次性聚合计数
    query = f"""
        SELECT *,
        count(<-reference.in) as source_count,  -- 反向遍历 reference 边
        count(<-artifact.in) as note_count       -- 反向遍历 artifact 边
        FROM notebook
        ORDER BY {validated_order_by}
    """
    result = await repo_query(query)
    
    return [
        NotebookResponse(
            id=str(nb.get("id", "")),
            name=nb.get("name", ""),
            description=nb.get("description", ""),
            archived=nb.get("archived", False),
            created=str(nb.get("created", "")),
            updated=str(nb.get("updated", "")),
            source_count=nb.get("source_count", 0),  # 聚合后的计数
            note_count=nb.get("note_count", 0),      # 聚合后的计数
        )
        for nb in result
    ]
```

#### 2.3.2 信息源列表聚合查询

**文件位置**: `api/routers/sources.py`

```python
@router.get("/sources", response_model=List[SourceListResponse])
async def get_sources(
    notebook_id: Optional[str] = Query(None),
    limit: int = Query(50, ge=1, le=100),
    offset: int = Query(0),
    sort_by: str = Query("updated"),
    sort_order: str = Query("desc"),
):
    order_clause = f"ORDER BY {sort_by} {sort_order.upper()}"
    
    if notebook_id:
        # 特定笔记本的信息源查询 - 通过 reference 边
        query = f"""
            SELECT id, asset, created, title, updated, topics, command,
            -- 子查询聚合：洞察数量
            (SELECT VALUE count() FROM source_insight 
             WHERE source = $parent.id GROUP ALL)[0].count OR 0 AS insights_count,
            -- 子查询聚合：是否已嵌入
            (SELECT VALUE id FROM source_embedding 
             WHERE source = $parent.id LIMIT 1) != [] AS embedded
            FROM (select value in from reference where out=$notebook_id)
            {order_clause}
            LIMIT $limit START $offset
            FETCH command  -- 关联获取命令状态
        """
        result = await repo_query(
            query,
            {
                "notebook_id": ensure_record_id(notebook_id),
                "limit": limit,
                "offset": offset,
            },
        )
    else:
        # 全量信息源查询
        query = f"""
            SELECT id, asset, created, title, updated, topics, command,
            (SELECT VALUE count() FROM source_insight 
             WHERE source = $parent.id GROUP ALL)[0].count OR 0 AS insights_count,
            (SELECT VALUE id FROM source_embedding 
             WHERE source = $parent.id LIMIT 1) != [] AS embedded
            FROM source
            {order_clause}
            LIMIT $limit START $offset
            FETCH command
        """
        result = await repo_query(query, {"limit": limit, "offset": offset})
    
    # 处理命令状态（异步处理支持）
    response_list = []
    for row in result:
        command = row.get("command")
        command_id = None
        status = None
        processing_info = None
        
        if command and isinstance(command, dict):
            command_id = str(command.get("id"))
            status = command.get("status")
            result_data = command.get("result")
            execution_metadata = (
                result_data.get("execution_metadata", {})
                if isinstance(result_data, dict) else {}
            )
            processing_info = {
                "started_at": execution_metadata.get("started_at"),
                "completed_at": execution_metadata.get("completed_at"),
                "error": command.get("error_message"),
            }
        
        response_list.append(
            SourceListResponse(
                id=row["id"],
                title=row.get("title"),
                topics=row.get("topics") or [],
                asset=AssetModel(...) if row.get("asset") else None,
                embedded=row.get("embedded", False),
                embedded_chunks=0,
                insights_count=row.get("insights_count", 0),
                created=str(row["created"]),
                updated=str(row["updated"]),
                command_id=command_id,
                status=status,
                processing_info=processing_info,
            )
        )
    
    return response_list
```

#### 2.3.3 笔记列表查询

**文件位置**: `api/routers/notes.py`

```python
@router.get("/notes", response_model=List[NoteResponse])
async def get_notes(
    notebook_id: Optional[str] = Query(None),
):
    if notebook_id:
        # 通过 Notebook 对象的 get_notes 方法查询
        notebook = await Notebook.get(notebook_id)
        if not notebook:
            raise HTTPException(status_code=404, detail="Notebook not found")
        notes = await notebook.get_notes()
    else:
        # 全量查询
        notes = await Note.get_all(order_by="updated desc")
    
    return [
        NoteResponse(
            id=note.id or "",
            title=note.title,
            content=note.content,
            note_type=note.note_type,
            created=str(note.created),
            updated=str(note.updated),
        )
        for note in notes
    ]
```

### 2.4 关联创建与删除

#### 2.4.1 创建关联

```python
# Source.add_to_notebook
async def add_to_notebook(self, notebook_id: str) -> Any:
    return await self.relate("reference", notebook_id)

# Note.add_to_notebook
async def add_to_notebook(self, notebook_id: str) -> Any:
    return await self.relate("artifact", notebook_id)

# ObjectModel.relate (基类方法)
async def relate(self, edge_table: str, target_id: str) -> Any:
    return await repo_query(
        f"RELATE $self_id->{edge_table}->$target_id",
        {
            "self_id": ensure_record_id(self.id),
            "target_id": ensure_record_id(target_id),
        },
    )
```

#### 2.4.2 级联删除逻辑

**文件位置**: `open_notebook/domain/notebook.py` - `Notebook.delete()`

```python
async def delete(self, delete_exclusive_sources: bool = False) -> Dict[str, int]:
    notebook_id = ensure_record_id(self.id)
    deleted_notes = 0
    deleted_sources = 0
    unlinked_sources = 0
    
    # 1. 删除所有关联的笔记
    notes = await self.get_notes()
    for note in notes:
        await note.delete()
        deleted_notes += 1
    
    # 删除 artifact 边关系
    await repo_query(
        "DELETE artifact WHERE out = $notebook_id",
        {"notebook_id": notebook_id},
    )
    
    # 2. 处理信息源
    if delete_exclusive_sources:
        # 查询只属于当前笔记本的信息源
        source_counts = await repo_query(
            """
            SELECT
                id,
                count(->reference[WHERE out != $notebook_id].out) as assigned_others
            FROM (SELECT VALUE <-reference.in AS sources FROM $notebook_id)[0]
            """,
            {"notebook_id": notebook_id},
        )
        
        for src in source_counts:
            if src.get("assigned_others", 0) == 0:
                # 独占信息源 - 完全删除
                source = await Source.get(str(src.get("id")))
                await source.delete()
                deleted_sources += 1
            else:
                # 共享信息源 - 只解除关联
                unlinked_sources += 1
    else:
        # 只计数，所有信息源都只解除关联
        source_result = await repo_query(
            "SELECT count() as count FROM reference WHERE out = $notebook_id GROUP ALL",
            {"notebook_id": notebook_id},
        )
        unlinked_sources = source_result[0]["count"] if source_result else 0
    
    # 删除 reference 边关系
    await repo_query(
        "DELETE reference WHERE out = $notebook_id",
        {"notebook_id": notebook_id},
    )
    
    # 3. 删除笔记本本身
    await super().delete()
    
    return {
        "deleted_notes": deleted_notes,
        "deleted_sources": deleted_sources,
        "unlinked_sources": unlinked_sources,
    }
```

---

## 3. API 响应结构

### 3.1 响应模型定义

**文件位置**: `api/models.py` (后端) 和 `frontend/src/lib/types/api.ts` (前端)

#### 3.1.1 NotebookResponse

```typescript
// frontend/src/lib/types/api.ts
export interface NotebookResponse {
  id: string
  name: string
  description: string
  archived: boolean
  created: string
  updated: string
  source_count: number   // 聚合：关联的信息源数量
  note_count: number     // 聚合：关联的笔记数量
}
```

#### 3.1.2 SourceListResponse

```typescript
export interface SourceListResponse {
  id: string
  title: string | null
  topics?: string[]
  asset: {
    file_path?: string
    url?: string
  } | null
  embedded: boolean           // 是否已向量化
  embedded_chunks: number     // 向量化块数量
  insights_count: number      // 聚合：AI 洞察数量
  created: string
  updated: string
  file_available?: boolean
  // 异步处理字段
  command_id?: string
  status?: string
  processing_info?: Record<string, unknown>
}
```

#### 3.1.3 NoteResponse

```typescript
export interface NoteResponse {
  id: string
  title: string | null
  content: string | null
  note_type: string | null    // 'human' | 'ai'
  created: string
  updated: string
}
```

### 3.2 API 端点汇总

| 端点 | 方法 | 功能 | 分页支持 |
|------|------|------|----------|
| `/notebooks` | GET | 获取笔记本列表 | 否 |
| `/notebooks/{id}` | GET | 获取单个笔记本 | 否 |
| `/notebooks` | POST | 创建笔记本 | - |
| `/sources` | GET | 获取信息源列表 | 是 (limit/offset) |
| `/sources/{id}` | GET | 获取单个信息源详情 | 否 |
| `/sources` | POST | 创建信息源 | - |
| `/sources/{id}/status` | GET | 获取处理状态 | 否 |
| `/notes` | GET | 获取笔记列表 | 否 |
| `/notes/{id}` | GET | 获取单个笔记 | 否 |
| `/notes` | POST | 创建笔记 | - |

### 3.3 分页实现

**后端**: `api/routers/sources.py`

```python
@router.get("/sources", response_model=List[SourceListResponse])
async def get_sources(
    notebook_id: Optional[str] = Query(None),
    limit: int = Query(50, ge=1, le=100),   # 每页数量
    offset: int = Query(0, ge=0),            # 偏移量
    sort_by: str = Query("updated"),
    sort_order: str = Query("desc"),
):
    # ...
    query = f"""
        SELECT ...
        FROM ...
        {order_clause}
        LIMIT $limit START $offset  -- SurrealQL 分页语法
        FETCH command
    """
```

**前端**: `frontend/src/lib/hooks/use-sources.ts`

```typescript
const NOTEBOOK_SOURCES_PAGE_SIZE = 30

export function useNotebookSources(notebookId: string) {
  const queryClient = useQueryClient()

  const query = useInfiniteQuery({
    queryKey: QUERY_KEYS.sourcesInfinite(notebookId),
    queryFn: async ({ pageParam = 0 }) => {
      const data = await sourcesApi.list({
        notebook_id: notebookId,
        limit: NOTEBOOK_SOURCES_PAGE_SIZE,
        offset: pageParam,
        sort_by: 'updated',
        sort_order: 'desc',
      })
      return {
        sources: data,
        // 计算下一页偏移量
        nextOffset: data.length === NOTEBOOK_SOURCES_PAGE_SIZE 
          ? pageParam + data.length 
          : undefined,
      }
    },
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextOffset,
    enabled: !!notebookId,
    staleTime: 5 * 1000,
    refetchOnWindowFocus: true,
  })

  // 扁平化所有页面数据
  const sources: SourceListResponse[] = useMemo(
    () => query.data?.pages.flatMap(page => page.sources) ?? [],
    [query.data?.pages]
  )

  return {
    sources,
    isLoading: query.isLoading,
    isFetchingNextPage: query.isFetchingNextPage,
    hasNextPage: query.hasNextPage,
    fetchNextPage: query.fetchNextPage,
    // ...
  }
}
```

---

## 4. 前端状态管理

### 4.1 状态管理架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端状态管理架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              TanStack Query (React Query)                │   │
│  │              服务端状态缓存层                              │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  • 数据获取与缓存                                          │   │
│  │  • 后台自动刷新 (staleTime, refetchOnWindowFocus)        │   │
│  │  • 无限滚动分页 (useInfiniteQuery)                       │   │
│  │  • 乐观更新与失效 (invalidateQueries)                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Zustand Store                           │   │
│  │                客户端 UI 状态层                           │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  useNotebookColumnsStore:                                │   │
│  │  • sourcesCollapsed: 信息源列折叠状态                    │   │
│  │  • notesCollapsed: 笔记列折叠状态                        │   │
│  │  • 持久化到 localStorage                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               React 本地状态 (useState)                  │   │
│  │                   组件级临时状态                           │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  contextSelections: 上下文选择模式 (off/insights/full)  │   │
│  │  mobileActiveTab: 移动端当前激活标签页                    │   │
│  │  对话框开关状态 (addDialogOpen, deleteDialogOpen 等)    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 TanStack Query 配置

#### 4.2.1 Query Keys 定义

**文件位置**: `frontend/src/lib/api/query-client.ts`

```typescript
export const QUERY_KEYS = {
  notebooks: () => ['notebooks'],
  notebook: (id?: string) => ['notebooks', id],
  
  sources: (notebookId?: string) => 
    notebookId ? ['sources', { notebookId }] : ['sources'],
  sourcesInfinite: (notebookId: string) => 
    ['sources', 'infinite', { notebookId }],
  source: (id: string) => ['sources', id],
  
  notes: (notebookId?: string) => 
    notebookId ? ['notes', { notebookId }] : ['notes'],
  note: (id: string) => ['notes', id],
  
  // ... 其他 keys
}
```

#### 4.2.2 信息源查询 Hook

**文件位置**: `frontend/src/lib/hooks/use-sources.ts`

```typescript
// 基础查询（非分页）
export function useSources(notebookId?: string) {
  return useQuery({
    queryKey: QUERY_KEYS.sources(notebookId),
    queryFn: () => sourcesApi.list({ notebook_id: notebookId }),
    enabled: !!notebookId,
    staleTime: 5 * 1000,           // 5 秒后视为过期
    refetchOnWindowFocus: true,    // 窗口聚焦时刷新
  })
}

// 无限滚动分页查询
export function useNotebookSources(notebookId: string) {
  const queryClient = useQueryClient()

  const query = useInfiniteQuery({
    queryKey: QUERY_KEYS.sourcesInfinite(notebookId),
    queryFn: async ({ pageParam = 0 }) => {
      const data = await sourcesApi.list({
        notebook_id: notebookId,
        limit: NOTEBOOK_SOURCES_PAGE_SIZE,
        offset: pageParam,
        sort_by: 'updated',
        sort_order: 'desc',
      })
      return {
        sources: data,
        nextOffset: data.length === NOTEBOOK_SOURCES_PAGE_SIZE 
          ? pageParam + data.length 
          : undefined,
      }
    },
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextOffset,
    enabled: !!notebookId,
    staleTime: 5 * 1000,
    refetchOnWindowFocus: true,
  })

  // 扁平化数据
  const sources: SourceListResponse[] = useMemo(
    () => query.data?.pages.flatMap(page => page.sources) ?? [],
    [query.data?.pages]
  )

  // 自定义刷新（重置分页）
  const refetch = useCallback(() => {
    queryClient.invalidateQueries({ 
      queryKey: QUERY_KEYS.sourcesInfinite(notebookId) 
    })
  }, [queryClient, notebookId])

  return {
    sources,
    isLoading: query.isLoading,
    isFetchingNextPage: query.isFetchingNextPage,
    hasNextPage: query.hasNextPage,
    fetchNextPage: query.fetchNextPage,
    refetch,
    error: query.error,
  }
}
```

#### 4.2.3 变更操作与缓存失效

```typescript
// 创建信息源后的缓存失效
export function useCreateSource() {
  const queryClient = useQueryClient()
  const { toast } = useToast()
  const { t } = useTranslation()

  return useMutation({
    mutationFn: (data: CreateSourceRequest) => sourcesApi.create(data),
    onSuccess: (result: SourceResponse, variables) => {
      // 失效所有相关笔记本的查询
      if (variables.notebooks) {
        variables.notebooks.forEach(notebookId => {
          queryClient.invalidateQueries({
            queryKey: QUERY_KEYS.sources(notebookId),
            refetchType: 'active'
          })
          queryClient.invalidateQueries({
            queryKey: QUERY_KEYS.sourcesInfinite(notebookId),
            refetchType: 'active'
          })
        })
      }
      // 失效全量查询
      queryClient.invalidateQueries({
        queryKey: QUERY_KEYS.sources(),
        refetchType: 'active'
      })
      // 显示成功提示
      toast({ title: t('common.success'), ... })
    },
    onError: (error) => {
      toast({ 
        title: t('common.error'), 
        variant: 'destructive',
        ... 
      })
    },
  })
}
```

### 4.3 Zustand Store

**文件位置**: `frontend/src/lib/stores/notebook-columns-store.ts`

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface NotebookColumnsState {
  sourcesCollapsed: boolean
  notesCollapsed: boolean
  toggleSources: () => void
  toggleNotes: () => void
  setSources: (collapsed: boolean) => void
  setNotes: (collapsed: boolean) => void
}

export const useNotebookColumnsStore = create<NotebookColumnsState>()(
  persist(
    (set) => ({
      sourcesCollapsed: false,
      notesCollapsed: false,
      toggleSources: () => set((state) => ({ 
        sourcesCollapsed: !state.sourcesCollapsed 
      })),
      toggleNotes: () => set((state) => ({ 
        notesCollapsed: !state.notesCollapsed 
      })),
      setSources: (collapsed) => set({ sourcesCollapsed: collapsed }),
      setNotes: (collapsed) => set({ notesCollapsed: collapsed }),
    }),
    {
      name: 'notebook-columns-storage',  // localStorage key
    }
  )
)
```

### 4.4 React 本地状态

**文件位置**: `frontend/src/app/(dashboard)/notebooks/[id]/page.tsx`

```typescript
export type ContextMode = 'off' | 'insights' | 'full'

export interface ContextSelections {
  sources: Record<string, ContextMode>
  notes: Record<string, ContextMode>
}

export default function NotebookPage() {
  const params = useParams()
  const notebookId = params?.id ? decodeURIComponent(params.id as string) : ''

  // 从 React Query 获取数据
  const { data: notebook, isLoading: notebookLoading } = useNotebook(notebookId)
  const {
    sources,
    isLoading: sourcesLoading,
    refetch: refetchSources,
    hasNextPage,
    isFetchingNextPage,
    fetchNextPage,
  } = useNotebookSources(notebookId)
  const { data: notes, isLoading: notesLoading } = useNotes(notebookId)

  // 从 Zustand 获取 UI 状态
  const { sourcesCollapsed, notesCollapsed } = useNotebookColumnsStore()

  // 响应式检测
  const isDesktop = useIsDesktop()

  // 本地状态：移动端标签页
  const [mobileActiveTab, setMobileActiveTab] = useState<
    'sources' | 'notes' | 'chat'
  >('chat')

  // 本地状态：上下文选择
  const [contextSelections, setContextSelections] = useState<ContextSelections>({
    sources: {},
    notes: {}
  })

  // 初始化上下文选择（当 sources 加载时）
  useEffect(() => {
    if (sources && sources.length > 0) {
      setContextSelections(prev => {
        const newSourceSelections = { ...prev.sources }
        sources.forEach(source => {
          const currentMode = newSourceSelections[source.id]
          const hasInsights = source.insights_count > 0

          if (currentMode === undefined) {
            // 首次加载：根据是否有洞察设置默认值
            newSourceSelections[source.id] = hasInsights ? 'insights' : 'full'
          } else if (currentMode === 'full' && hasInsights) {
            // 有洞察时自动切换到 insights 模式
            newSourceSelections[source.id] = 'insights'
          }
        })
        return { ...prev, sources: newSourceSelections }
      })
    }
  }, [sources])

  // 笔记上下文选择初始化
  useEffect(() => {
    if (notes && notes.length > 0) {
      setContextSelections(prev => {
        const newNoteSelections = { ...prev.notes }
        notes.forEach(note => {
          if (!(note.id in newNoteSelections)) {
            newNoteSelections[note.id] = 'full'  // 笔记默认为 full
          }
        })
        return { ...prev, notes: newNoteSelections }
      })
    }
  }, [notes])

  // 上下文模式变更处理
  const handleContextModeChange = (
    itemId: string, 
    mode: ContextMode, 
    type: 'source' | 'note'
  ) => {
    setContextSelections(prev => ({
      ...prev,
      [type === 'source' ? 'sources' : 'notes']: {
        ...(type === 'source' ? prev.sources : prev.notes),
        [itemId]: mode
      }
    }))
  }

  // ... 渲染逻辑
}
```

---

## 5. 前端列视图渲染

### 5.1 整体布局架构

**文件位置**: `frontend/src/app/(dashboard)/notebooks/[id]/page.tsx`

```typescript
return (
  <AppShell>
    <div className="flex flex-col flex-1 min-h-0">
      {/* 笔记本头部 */}
      <div className="flex-shrink-0 p-6 pb-0">
        <NotebookHeader notebook={notebook} />
      </div>

      <div className="flex-1 p-6 pt-6 overflow-x-auto flex flex-col">
        
        {/* 移动端：标签页布局 */}
        {!isDesktop && (
          <>
            {/* 标签切换按钮 */}
            <div className="lg:hidden mb-4">
              <Tabs value={mobileActiveTab} onValueChange={...}>
                <TabsList className="grid w-full grid-cols-3">
                  <TabsTrigger value="sources">
                    <FileText /> {t('navigation.sources')}
                  </TabsTrigger>
                  <TabsTrigger value="notes">
                    <StickyNote /> {t('common.notes')}
                  </TabsTrigger>
                  <TabsTrigger value="chat">
                    <MessageSquare /> {t('common.chat')}
                  </TabsTrigger>
                </TabsList>
              </Tabs>
            </div>

            {/* 移动端：只显示激活的标签页 */}
            <div className="flex-1 overflow-hidden lg:hidden">
              {mobileActiveTab === 'sources' && (
                <SourcesColumn
                  sources={sources}
                  isLoading={sourcesLoading}
                  notebookId={notebookId}
                  contextSelections={contextSelections.sources}
                  onContextModeChange={...}
                  hasNextPage={hasNextPage}
                  isFetchingNextPage={isFetchingNextPage}
                  fetchNextPage={fetchNextPage}
                />
              )}
              {mobileActiveTab === 'notes' && (
                <NotesColumn
                  notes={notes}
                  isLoading={notesLoading}
                  notebookId={notebookId}
                  contextSelections={contextSelections.notes}
                  onContextModeChange={...}
                />
              )}
              {mobileActiveTab === 'chat' && (
                <ChatColumn
                  notebookId={notebookId}
                  contextSelections={contextSelections}
                  sources={sources}
                  sourcesLoading={sourcesLoading}
                />
              )}
            </div>
          </>
        )}

        {/* 桌面端：三列可折叠布局 */}
        <div className={cn(
          'hidden lg:flex h-full min-h-0 gap-6 transition-all duration-150',
          'flex-row'
        )}>
          {/* 信息源列 */}
          <div className={cn(
            'transition-all duration-150',
            sourcesCollapsed ? 'w-12 flex-shrink-0' : 'flex-none basis-1/3'
          )}>
            <SourcesColumn ... />
          </div>

          {/* 笔记列 */}
          <div className={cn(
            'transition-all duration-150',
            notesCollapsed ? 'w-12 flex-shrink-0' : 'flex-none basis-1/3'
          )}>
            <NotesColumn ... />
          </div>

          {/* 聊天列（始终展开） */}
          <div className="transition-all duration-150 flex-1 min-w-0">
            <ChatColumn ... />
          </div>
        </div>
      </div>
    </div>
  </AppShell>
)
```

### 5.2 SourcesColumn 组件

**文件位置**: `frontend/src/app/(dashboard)/notebooks/components/SourcesColumn.tsx`

```typescript
interface SourcesColumnProps {
  sources?: SourceListResponse[]
  isLoading: boolean
  notebookId: string
  notebookName?: string
  onRefresh?: () => void
  contextSelections?: Record<string, ContextMode>
  onContextModeChange?: (sourceId: string, mode: ContextMode) => void
  // 分页属性
  hasNextPage?: boolean
  isFetchingNextPage?: boolean
  fetchNextPage?: () => void
}

export function SourcesColumn({
  sources,
  isLoading,
  notebookId,
  onRefresh,
  contextSelections,
  onContextModeChange,
  hasNextPage,
  isFetchingNextPage,
  fetchNextPage,
}: SourcesColumnProps) {
  const { t } = useTranslation()
  const [dropdownOpen, setDropdownOpen] = useState(false)
  const [addDialogOpen, setAddDialogOpen] = useState(false)
  // ... 其他对话框状态

  // 从 Zustand 获取折叠状态
  const { sourcesCollapsed, toggleSources } = useNotebookColumnsStore()
  const collapseButton = useMemo(
    () => createCollapseButton(toggleSources, t('navigation.sources')),
    [toggleSources, t('navigation.sources')]
  )

  // 滚动容器引用（用于无限滚动）
  const scrollContainerRef = useRef<HTMLDivElement>(null)

  // 无限滚动处理
  const handleScroll = useCallback(() => {
    const container = scrollContainerRef.current
    if (!container || !hasNextPage || isFetchingNextPage || !fetchNextPage) return

    const { scrollTop, scrollHeight, clientHeight } = container
    // 距离底部 200px 时加载更多
    if (scrollHeight - scrollTop - clientHeight < 200) {
      fetchNextPage()
    }
  }, [hasNextPage, isFetchingNextPage, fetchNextPage])

  // 绑定滚动事件
  useEffect(() => {
    const container = scrollContainerRef.current
    if (!container) return
    container.addEventListener('scroll', handleScroll)
    return () => container.removeEventListener('scroll', handleScroll)
  }, [handleScroll])

  // 事件处理函数
  const handleSourceClick = (sourceId: string) => {
    openModal('source', sourceId)  // 打开详情对话框
  }

  const handleDeleteClick = (sourceId: string) => {
    setSourceToDelete(sourceId)
    setDeleteDialogOpen(true)
  }

  const handleDeleteConfirm = async () => {
    if (!sourceToDelete) return
    await deleteSource.mutateAsync(sourceToDelete)
    setDeleteDialogOpen(false)
    onRefresh?.()
  }

  return (
    <>
      <CollapsibleColumn
        isCollapsed={sourcesCollapsed}
        onToggle={toggleSources}
        collapsedIcon={FileText}
        collapsedLabel={t('navigation.sources')}
      >
        <Card className="h-full flex flex-col flex-1 overflow-hidden">
          {/* 列头部 */}
          <CardHeader className="pb-3 flex-shrink-0">
            <div className="flex items-center justify-between gap-2">
              <CardTitle className="text-lg">{t('navigation.sources')}</CardTitle>
              <div className="flex items-center gap-2">
                {/* 添加信息源下拉菜单 */}
                <DropdownMenu open={dropdownOpen} onOpenChange={setDropdownOpen}>
                  <DropdownMenuTrigger asChild>
                    <Button size="sm">
                      <Plus className="h-4 w-4 mr-2" />
                      {t('sources.addSource')}
                      <ChevronDown className="h-4 w-4 ml-2" />
                    </Button>
                  </DropdownMenuTrigger>
                  <DropdownMenuContent align="end">
                    <DropdownMenuItem onClick={() => {
                      setDropdownOpen(false)
                      setAddDialogOpen(true)
                    }}>
                      <Plus /> {t('sources.addSource')}
                    </DropdownMenuItem>
                    <DropdownMenuItem onClick={() => {
                      setDropdownOpen(false)
                      setAddExistingDialogOpen(true)
                    }}>
                      <Link2 /> {t('sources.addExistingTitle')}
                    </DropdownMenuItem>
                  </DropdownMenuContent>
                </DropdownMenu>
                {collapseButton}
              </div>
            </div>
          </CardHeader>

          {/* 可滚动内容区域 */}
          <CardContent 
            ref={scrollContainerRef} 
            className="flex-1 overflow-y-auto min-h-0"
          >
            {isLoading ? (
              <div className="flex items-center justify-center py-8">
                <LoadingSpinner />
              </div>
            ) : !sources || sources.length === 0 ? (
              <EmptyState
                icon={FileText}
                title={t('sources.noSourcesYet')}
                description={t('sources.createFirstSource')}
              />
            ) : (
              <div className="space-y-3">
                {sources.map((source) => (
                  <SourceCard
                    key={source.id}
                    source={source}
                    onClick={handleSourceClick}
                    onDelete={handleDeleteClick}
                    onRetry={handleRetry}
                    onRemoveFromNotebook={handleRemoveFromNotebook}
                    onRefresh={onRefresh}
                    showRemoveFromNotebook={true}
                    contextMode={contextSelections?.[source.id]}
                    onContextModeChange={onContextModeChange
                      ? (mode) => onContextModeChange(source.id, mode)
                      : undefined
                    }
                  />
                ))}
                {/* 无限滚动加载指示器 */}
                {isFetchingNextPage && (
                  <div className="flex items-center justify-center py-4">
                    <Loader2 className="h-5 w-5 animate-spin text-muted-foreground" />
                  </div>
                )}
              </div>
            )}
          </CardContent>
        </Card>
      </CollapsibleColumn>

      {/* 对话框组件 */}
      <AddSourceDialog
        open={addDialogOpen}
        onOpenChange={setAddDialogOpen}
        defaultNotebookId={notebookId}
      />
      {/* ... 其他对话框 */}
    </>
  )
}
```

### 5.3 NotesColumn 组件

**文件位置**: `frontend/src/app/(dashboard)/notebooks/components/NotesColumn.tsx`

```typescript
interface NotesColumnProps {
  notes?: NoteResponse[]
  isLoading: boolean
  notebookId: string
  contextSelections?: Record<string, ContextMode>
  onContextModeChange?: (noteId: string, mode: ContextMode) => void
}

export function NotesColumn({
  notes,
  isLoading,
  notebookId,
  contextSelections,
  onContextModeChange
}: NotesColumnProps) {
  const { t, language } = useTranslation()
  const [showAddDialog, setShowAddDialog] = useState(false)
  const [editingNote, setEditingNote] = useState<NoteResponse | null>(null)
  const [deleteDialogOpen, setDeleteDialogOpen] = useState(false)
  const [noteToDelete, setNoteToDelete] = useState<string | null>(null)

  const deleteNote = useDeleteNote()

  // 折叠状态
  const { notesCollapsed, toggleNotes } = useNotebookColumnsStore()
  const collapseButton = useMemo(
    () => createCollapseButton(toggleNotes, t('common.notes')),
    [toggleNotes, t('common.notes')]
  )

  return (
    <>
      <CollapsibleColumn
        isCollapsed={notesCollapsed}
        onToggle={toggleNotes}
        collapsedIcon={StickyNote}
        collapsedLabel={t('common.notes')}
      >
        <Card className="h-full flex flex-col flex-1 overflow-hidden">
          <CardHeader className="pb-3 flex-shrink-0">
            <div className="flex items-center justify-between gap-2">
              <CardTitle className="text-lg">{t('common.notes')}</CardTitle>
              <div className="flex items-center gap-2">
                <Button
                  size="sm"
                  onClick={() => {
                    setEditingNote(null)
                    setShowAddDialog(true)
                  }}
                >
                  <Plus className="h-4 w-4 mr-2" />
                  {t('common.writeNote')}
                </Button>
                {collapseButton}
              </div>
            </div>
          </CardHeader>

          <CardContent className="flex-1 overflow-y-auto min-h-0">
            {isLoading ? (
              <div className="flex items-center justify-center py-8">
                <LoadingSpinner />
              </div>
            ) : !notes || notes.length === 0 ? (
              <EmptyState
                icon={StickyNote}
                title={t('notebooks.noNotesYet')}
                description={t('sources.createFirstNote')}
              />
            ) : (
              <div className="space-y-3">
                {notes.map((note) => (
                  <div
                    key={note.id}
                    className="p-3 border rounded-lg card-hover group relative cursor-pointer"
                    onClick={() => setEditingNote(note)}
                  >
                    <div className="flex items-start justify-between mb-2">
                      <div className="flex items-center gap-2">
                        {/* 笔记类型图标 */}
                        {note.note_type === 'ai' ? (
                          <Bot className="h-4 w-4 text-primary" />
                        ) : (
                          <User className="h-4 w-4 text-muted-foreground" />
                        )}
                        <Badge variant="secondary" className="text-xs">
                          {note.note_type === 'ai' 
                            ? t('common.aiGenerated') 
                            : t('common.human')}
                        </Badge>
                      </div>

                      <div className="flex items-center gap-2">
                        {/* 更新时间 */}
                        <span className="text-xs text-muted-foreground">
                          {formatDistanceToNow(new Date(note.updated), { 
                            addSuffix: true,
                            locale: getDateLocale(language)
                          })}
                        </span>

                        {/* 上下文选择器 */}
                        {onContextModeChange && contextSelections?.[note.id] && (
                          <div onClick={(event) => event.stopPropagation()}>
                            <ContextToggle
                              mode={contextSelections[note.id]}
                              hasInsights={false}
                              onChange={(mode) => onContextModeChange(note.id, mode)}
                            />
                          </div>
                        )}

                        {/* 删除菜单 */}
                        <DropdownMenu>
                          <DropdownMenuTrigger asChild>
                            <Button
                              variant="ghost"
                              size="sm"
                              className="h-8 w-8 p-0 opacity-0 group-hover:opacity-100 transition-opacity"
                              onClick={(e) => e.stopPropagation()}
                            >
                              <MoreVertical className="h-4 w-4" />
                            </Button>
                          </DropdownMenuTrigger>
                          <DropdownMenuContent align="end" className="w-48">
                            <DropdownMenuItem
                              onClick={(e) => {
                                e.stopPropagation()
                                handleDeleteClick(note.id)
                              }}
                              className="text-red-600 focus:text-red-600"
                            >
                              <Trash2 className="h-4 w-4 mr-2" />
                              {t('notebooks.deleteNote')}
                            </DropdownMenuItem>
                          </DropdownMenuContent>
                        </DropdownMenu>
                      </div>
                    </div>

                    {/* 笔记标题和内容 */}
                    {note.title && (
                      <h4 className="text-sm font-medium mb-2 break-all">
                        {note.title}
                      </h4>
                    )}
                    {note.content && (
                      <p className="text-sm text-muted-foreground line-clamp-3 break-all">
                        {note.content}
                      </p>
                    )}
                  </div>
                ))}
              </div>
            )}
          </CardContent>
        </Card>
      </CollapsibleColumn>

      {/* 笔记编辑器对话框 */}
      <NoteEditorDialog
        open={showAddDialog || Boolean(editingNote)}
        onOpenChange={(open) => {
          if (!open) {
            setShowAddDialog(false)
            setEditingNote(null)
          } else {
            setShowAddDialog(true)
          }
        }}
        notebookId={notebookId}
        note={editingNote ?? undefined}
      />

      {/* 删除确认对话框 */}
      <ConfirmDialog ... />
    </>
  )
}
```

### 5.4 CollapsibleColumn 组件

**文件位置**: `frontend/src/components/notebooks/CollapsibleColumn.tsx`

```typescript
interface CollapsibleColumnProps {
  isCollapsed: boolean
  onToggle: () => void
  collapsedIcon: LucideIcon
  collapsedLabel: string
  children: React.ReactNode
}

export function CollapsibleColumn({
  isCollapsed,
  onToggle,
  collapsedIcon: Icon,
  collapsedLabel,
  children,
}: CollapsibleColumnProps) {
  if (isCollapsed) {
    // 折叠状态：只显示图标按钮
    return (
      <div className="flex flex-col items-center py-4 border-r">
        <TooltipProvider>
          <Tooltip>
            <TooltipTrigger asChild>
              <Button
                variant="ghost"
                size="icon"
                onClick={onToggle}
                className="h-10 w-10"
              >
                <Icon className="h-5 w-5" />
              </Button>
            </TooltipTrigger>
            <TooltipContent side="right">
              {collapsedLabel}
            </TooltipContent>
          </Tooltip>
        </TooltipProvider>
      </div>
    )
  }

  // 展开状态：渲染完整内容
  return <>{children}</>
}

// 折叠按钮创建函数
export function createCollapseButton(
  onToggle: () => void,
  label: string
) {
  return (
    <TooltipProvider>
      <Tooltip>
        <TooltipTrigger asChild>
          <Button
            variant="ghost"
            size="icon"
            onClick={onToggle}
            className="h-8 w-8"
          >
            <ChevronRight className="h-4 w-4" />
          </Button>
        </TooltipTrigger>
        <TooltipContent>
          收起 {label}
        </TooltipContent>
      </Tooltip>
    </TooltipProvider>
  )
}
```

---

## 6. 完整数据流图

### 6.1 端到端数据流

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           完整数据流架构                                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         SurrealDB 数据库                                │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │  Tables:                                                              │  │
│  │  • notebook          (笔记本实体)                                       │  │
│  │  • source            (信息源实体)                                       │  │
│  │  • note              (笔记实体)                                         │  │
│  │  • source_embedding  (向量化块)                                         │  │
│  │  • source_insight    (AI 洞察)                                         │  │
│  │  • command           (异步处理任务)                                     │  │
│  │                                                                         │  │
│  │  Edges (关系):                                                         │  │
│  │  • reference:  source ──► notebook                                     │  │
│  │  • artifact:   note ──► notebook                                       │  │
│  │  • refers_to:  chat_session ──► notebook/source                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         Python 后端层                                  │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                         │  │
│  │  Domain Layer (open_notebook/domain/):                                │  │
│  │  • Notebook: get_sources(), get_notes(), delete()                    │  │
│  │  • Source: add_to_notebook(), get_insights(), get_embedded_chunks()  │  │
│  │  • Note: add_to_notebook()                                             │  │
│  │                                                                         │  │
│  │  API Routers (api/routers/):                                          │  │
│  │  • notebooks.py: 聚合查询 (count(<-reference.in))                     │  │
│  │  • sources.py:   分页查询 + 异步状态聚合                               │  │
│  │  • notes.py:     笔记 CRUD                                             │  │
│  │                                                                         │  │
│  │  Response Models (api/models.py):                                      │  │
│  │  • NotebookResponse: {source_count, note_count}                       │  │
│  │  • SourceListResponse: {embedded, insights_count, status}             │  │
│  │  • NoteResponse: {note_type}                                           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼ HTTP / JSON                           │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                       Next.js 前端层                                    │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                         │  │
│  │  API Client (frontend/src/lib/api/):                                  │  │
│  │  • sources.ts: list(), create(), get(), delete()                      │  │
│  │  • notes.ts:   list(), create(), get(), delete()                      │  │
│  │  • notebooks.ts: list(), get(), create(), update()                    │  │
│  │                                                                         │  │
│  │  TanStack Query (frontend/src/lib/hooks/):                            │  │
│  │  • useNotebookSources(): useInfiniteQuery 无限滚动                    │  │
│  │  • useNotes(): useQuery 普通查询                                       │  │
│  │  • useCreateSource(): useMutation 变更操作                             │  │
│  │                                                                         │  │
│  │  Zustand Store (frontend/src/lib/stores/):                            │  │
│  │  • useNotebookColumnsStore: sourcesCollapsed, notesCollapsed          │  │
│  │                                                                         │  │
│  │  Page Component (frontend/src/app/(dashboard)/notebooks/[id]/page.tsx)│  │
│  │  • contextSelections: 本地状态管理上下文选择                            │  │
│  │  • mobileActiveTab: 移动端标签页状态                                    │  │
│  │  • 数据聚合：sources (无限滚动) + notes (普通查询)                      │  │
│  │                                                                         │  │
│  │  Column Components:                                                    │  │
│  │  • SourcesColumn: 渲染 SourceCard 列表，无限滚动触发                   │  │
│  │  • NotesColumn: 渲染 Note 卡片，显示 note_type 区分                   │  │
│  │  • ChatColumn: 使用 contextSelections 构建 AI 上下文                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 数据聚合时序

```
用户打开笔记本详情页
         │
         ▼
┌─────────────────┐
│  Page Component │
│  (notebooks/[id])│
└────────┬────────┘
         │
         ├───► useNotebook(notebookId) ──► GET /notebooks/{id}
         │                                     │
         │                                     ▼
         │                              NotebookResponse
         │                              {id, name, description,
         │                               source_count, note_count}
         │
         ├───► useNotebookSources(notebookId) ──► GET /sources?notebook_id=...
         │                                              │
         │                                              ▼
         │                              第 1 页 (pageParam=0, limit=30)
         │                              SourceListResponse[]
         │                              {id, title, embedded, insights_count,
         │                               status, command_id, processing_info}
         │
         └───► useNotes(notebookId) ──► GET /notes?notebook_id=...
                                              │
                                              ▼
                                       NoteResponse[]
                                       {id, title, content, note_type}


用户滚动 SourcesColumn
         │
         ▼
┌─────────────────┐
│  handleScroll() │
│  (滚动事件监听)   │
└────────┬────────┘
         │
         ▼ (距离底部 < 200px)
┌─────────────────┐
│ fetchNextPage() │
└────────┬────────┘
         │
         ▼
GET /sources?notebook_id=...&offset=30&limit=30
         │
         ▼
  第 2 页数据追加到 sources 数组
```

---

## 7. 关键设计决策

### 7.1 数据库层面

| 决策 | 实现方式 | 优势 |
|------|----------|------|
| **图关系建模** | SurrealDB Edge 表 (`reference`, `artifact`) | 支持图遍历查询，一次性聚合计数 |
| **反向遍历语法** | `count(<-reference.in)` | 无需 JOIN，直接在查询中聚合计数 |
| **异步处理状态** | `command` 字段 + `FETCH` | 实时获取处理进度，轮询更新 |
| **子查询聚合** | `(SELECT ... WHERE source = $parent.id)` | 单查询获取 `embedded`、`insights_count` |

### 7.2 后端层面

| 决策 | 实现方式 | 优势 |
|------|----------|------|
| **领域模型封装** | `Notebook.get_sources()` 等方法 | 业务逻辑内聚，复用性强 |
| **API 响应聚合** | Router 层组装聚合字段 | 前端无需多次请求 |
| **异步处理** | `surreal-commands` 任务队列 | 大文件处理不阻塞 HTTP |
| **级联删除策略** | 独占/共享信息源区分 | 防止误删共享数据 |

### 7.3 前端层面

| 决策 | 实现方式 | 优势 |
|------|----------|------|
| **服务端状态** | TanStack Query | 自动缓存、后台刷新、乐观更新 |
| **无限滚动** | `useInfiniteQuery` + 滚动监听 | 大数据量平滑体验 |
| **UI 状态持久化** | Zustand `persist` middleware | 刷新页面保留折叠状态 |
| **上下文选择** | 本地状态 `contextSelections` | 实时响应，不触发 API |
| **响应式布局** | `useIsDesktop` + 条件渲染 | 移动端/桌面端一套代码 |

---

## 8. 代码引用索引

### 后端文件

| 文件路径 | 说明 |
|----------|------|
| `open_notebook/domain/notebook.py` | 核心数据模型定义 |
| `open_notebook/domain/base.py` | ObjectModel 基类（CRUD、关系操作） |
| `api/routers/notebooks.py` | 笔记本 API 路由（聚合查询） |
| `api/routers/sources.py` | 信息源 API 路由（分页、状态） |
| `api/routers/notes.py` | 笔记 API 路由 |
| `api/models.py` | Pydantic 响应模型定义 |

### 前端文件

| 文件路径 | 说明 |
|----------|------|
| `frontend/src/app/(dashboard)/notebooks/[id]/page.tsx` | 笔记本详情页（数据聚合） |
| `frontend/src/app/(dashboard)/notebooks/components/SourcesColumn.tsx` | 信息源列组件 |
| `frontend/src/app/(dashboard)/notebooks/components/NotesColumn.tsx` | 笔记列组件 |
| `frontend/src/components/notebooks/CollapsibleColumn.tsx` | 可折叠列组件 |
| `frontend/src/lib/hooks/use-sources.ts` | 信息源查询 Hooks |
| `frontend/src/lib/hooks/use-notes.ts` | 笔记查询 Hooks |
| `frontend/src/lib/stores/notebook-columns-store.ts` | 列状态 Zustand Store |
| `frontend/src/lib/types/api.ts` | TypeScript API 类型定义 |
| `frontend/src/lib/api/sources.ts` | 信息源 API 客户端 |
| `frontend/src/lib/api/notes.ts` | 笔记 API 客户端 |

---

## 9. 总结

Open Notebook 通过以下机制实现三类数据的聚合与列视图驱动：

### 后端聚合

1. **图数据库关系建模**：使用 `reference` 和 `artifact` 边表建立多对多关系
2. **SurrealQL 图遍历聚合**：通过 `<-reference.in` 反向遍历语法一次性获取关联计数
3. **子查询字段聚合**：在单条查询中计算 `embedded`、`insights_count` 等衍生字段
4. **异步状态聚合**：通过 `FETCH command` 关联获取处理任务状态

### 前端状态

1. **分层状态管理**：
   - 服务端状态：TanStack Query（缓存、分页、自动刷新）
   - UI 状态：Zustand（折叠状态持久化）
   - 临时状态：React `useState`（上下文选择、对话框）

2. **数据流向**：
   ```
   API → TanStack Query Cache → Custom Hooks → Page → Column Components
   ```

3. **无限滚动实现**：
   - `useInfiniteQuery` 管理分页状态
   - 滚动监听触发 `fetchNextPage`
   - `useMemo` 扁平化多页数据

### 列视图驱动

1. **响应式布局**：
   - 桌面端：三列可折叠布局（通过 `sourcesCollapsed`/`notesCollapsed` 控制宽度）
   - 移动端：三标签页切换（通过 `mobileActiveTab` 控制显示）

2. **上下文选择机制**：
   - `contextSelections` 本地状态记录每一项的上下文模式
   - `ContextToggle` 组件允许用户切换 `off`/`insights`/`full`
   - 变更后即时传递给 `ChatColumn` 构建 AI 上下文

3. **用户交互反馈**：
   - 加载状态：`LoadingSpinner`、`Loader2` 动画
   - 空状态：`EmptyState` 引导操作
   - 操作确认：`ConfirmDialog` 防止误删

这种设计实现了：
- **查询高效性**：后端单条 SQL 完成聚合，避免 N+1 查询
- **用户体验性**：前端分页加载、状态持久化、响应式适配
- **架构清晰性**：领域模型 → API 路由 → 状态管理 → UI 组件的清晰分层
