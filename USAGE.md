# USAGE.md - SuperMew 项目开发使用指南

本指南提供 SuperMew 项目的详细使用说明，涵盖架构解析、核心流程、开发任务和故障排查。

---

## 目录

1. [项目概述](#项目概述)
2. [技术栈](#技术栈)
3. [架构详解](#架构详解)
4. [核心流程](#核心流程)
5. [开发指南](#开发指南)
6. [故障排查](#故障排查)

---

## 项目概述

SuperMew 是一个基于 RAG（检索增强生成）的智能聊天助手应用，具备以下核心能力：

- **混合检索**：稠密向量（BAAI/bge-m3）+ 稀疏向量（BM25），RRF 融合排序
- **Auto-merging 检索**：三级分块（L1/L2/L3），叶子召回后自动合并到父块
- **查询重写**：Step-Back（退步问题）和 HyDE（假设性文档）策略
- **实时流式输出**：SSE 推送，前端打字机效果，RAG 过程可视化
- **JWT 鉴权 + RBAC**：admin（文档管理）/ user（聊天会话）角色分离

---

## 技术栈

### 后端
| 类别 | 技术 | 版本/说明 |
|------|------|----------|
| 框架 | FastAPI | 异步 Web 框架 |
| Agent | LangChain + LangGraph | 状态机 RAG 流程 |
| 向量库 | Milvus | v2.5.14，HNSW + SPARSE_INVERTED_INDEX |
| 关系数据库 | PostgreSQL | v15，存储用户/会话/消息/父块 |
| 缓存 | Redis | v7，会话/父块缓存 |
| 密集向量 | langchain_huggingface | BAAI/bge-m3，1024 维 |
| 稀疏向量 | 手写 BM25 | 中英混合分词，持久化统计 |
| 精排 | Jina Rerank API | 可选，不配置则降级 |
| 认证 | JWT (python-jose) | HS256，PBKDF2-SHA256 密码哈希 |

### 前端
| 类别 | 技术 | 说明 |
|------|------|------|
| 框架 | Vue 3 (CDN) | 无构建，单 HTML 文件 |
| Markdown | marked | 渲染 LLM 输出 |
| 代码高亮 | highlight.js | 支持代码块 |
| 图标 | FontAwesome 6 | UI 图标 |

---

## 架构详解

### 后端模块依赖关系

```
app.py (FastAPI 入口)
├── api.py (REST 路由)
│   ├── auth.py (JWT 认证，@require_admin)
│   ├── agent.py (LangGraph Agent)
│   │   └── rag_pipeline.py (RAG 状态机)
│   │       ├── rag_utils.py (检索/重写/Rerank)
│   │       │   ├── milvus_client.py (混合检索)
│   │       │   ├── parent_chunk_store.py (父块 PostgreSQL+Redis)
│   │       │   └── embedding.py (BM25 统计)
│   │       └── tools.py (emit_rag_step 跨线程推送)
│   ├── milvus_writer.py (向量写入 + BM25 increment_add)
│   └── document_loader.py (3 级分块)
├── database.py (SQLAlchemy engine)
├── models.py (ORM: User, ChatSession, ChatMessage, ParentChunk)
└── cache.py (Redis JSON 缓存)
```

### 数据流向总览

```
用户提问
   │
   ▼
POST /chat/stream (带 Bearer Token)
   │
   ▼
chat_with_agent_stream()
   ├── 创建 asyncio.Queue (统一输出)
   ├── 设置 _RagStepProxy (emit_rag_step → queue)
   ├── 启动 _agent_worker 后台任务
   │     └── agent.astream() → 逐 token 入队
   │
   └── 主循环：await queue.get() → yield SSE
         ▲
         └── RAG 工具在线程池执行
             emit_rag_step() → call_soon_threadsafe → 入队

RAG 工具执行流程:
search_knowledge_base(query)
   │
   ▼
rag_pipeline.invoke({question: query})
   │
   ├─→ retrieve_initial (L3 叶子召回)
   │     ├── embedding_service.get_all_embeddings(query)
   │     ├── milvus_manager.hybrid_retrieve(dense + sparse)
   │     ├── _rerank_documents (Jina API 精排)
   │     └── _auto_merge_documents (L3→L2→L1)
   │
   ├─→ grade_documents (LLM 相关性评分 yes/no)
   │     └── no → rewrite_question
   │
   ├─→ rewrite_question (Step-Back/HyDE/Complex 路由)
   │     ├── _generate_step_back_question
   │     ├── _answer_step_back_question
   │     └── generate_hypothetical_document
   │
   └─→ retrieve_expanded (重写后二次召回)
         └── 去重 → 返回 rag_trace
```

### 存储结构

#### PostgreSQL 表
| 表名 | 用途 | 关键字段 |
|------|------|---------|
| users | 用户 | id, username, password_hash (pbkdf2_sha256), role, created_at |
| chat_sessions | 会话 | id, user_id, session_id, metadata_json, updated_at |
| chat_messages | 消息 | id, session_ref_id, message_type, content, rag_trace (JSON) |
| parent_chunks | 父块文档 | chunk_id (PK), text, filename, chunk_level, parent_chunk_id, root_chunk_id |

#### Redis Key 前缀
| Key 模式 | 用途 | TTL |
|---------|------|-----|
| supermew:chat_messages:{user}:{session} | 会话消息缓存 | 300s |
| supermew:chat_sessions:{user} | 会话列表缓存 | 300s |
| supermew:parent_chunk:{chunk_id} | 父块内容缓存 | 300s |

#### Milvus 集合 Schema
| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT64 (主键) | 自增 |
| dense_embedding | FLOAT_VECTOR (1024) | BAAI/bge-m3 稠密向量 |
| sparse_embedding | SPARSE_FLOAT_VECTOR | BM25 稀疏向量 |
| text | VARCHAR (2000) | 分块文本 |
| filename, file_type, file_path | VARCHAR | 文件元数据 |
| page_number, chunk_idx | INT64 | 位置索引 |
| chunk_id, parent_chunk_id, root_chunk_id | VARCHAR (512) | 层级关联 |
| chunk_level | INT64 | 1/2/3 (L1/L2/L3) |

#### BM25 状态文件 (`data/bm25_state.json`)
```json
{
  "version": 1,
  "total_docs": 1234,
  "sum_token_len": 567890,
  "vocab": {"中文": 0, "english": 1, ...},
  "doc_freq": {"中文": 10, "english": 5, ...}
}
```
- `vocab`: 词 → 稀疏向量下标（只增不减，避免维度冲突）
- `doc_freq`: 词 → 包含该词的文档数（用于 IDF 计算）
- `total_docs`: 总文档数（每个 L3 chunk 视为一篇文档）
- `sum_token_len`: 总 token 长度（用于 BM25 文档长度归一化）

---

## 核心流程

### 1. 流式聊天 (Streaming Chat)

```python
# api.py: chat_stream_endpoint
POST /chat/stream
Request: {"message": "...", "session_id": "..."}
Response: StreamingResponse(text/event-stream)

SSE 事件格式:
data: {"type": "content", "content": "token..."}
data: {"type": "rag_step", "step": {"icon": "🔍", "label": "正在检索知识库..."}}
data: {"type": "trace", "rag_trace": {...}}
data: {"type": "error", "content": "..."}
data: [DONE]
```

**关键实现**:
1. `asyncio.Queue` 统一聚合内容 token 和 RAG 步骤
2. `_RagStepProxy` 将 `emit_rag_step()` 调用转为 `queue.put_nowait()`
3. `loop.call_soon_threadsafe()` 实现跨线程（ThreadPoolExecutor → asyncio Loop）调度
4. 前端 `AbortController.abort()` 触发 `GeneratorExit` → `agent_task.cancel()`

### 2. 文档上传与向量化

```python
# api.py: upload_document
POST /documents/upload (multipart/form-data)
仅限 admin 角色

流程:
1. 检查同名文件 → 存在则先删除旧 chunk
   - 从 Milvus 分页查询全部 chunk 文本
   - embedding_service.increment_remove_documents(texts) 扣减 BM25 统计
   - milvus_manager.delete(f'filename == "{filename}"')
   - parent_chunk_store.delete_by_filename(filename)

2. 保存文件到 data/documents/

3. document_loader.load_document()
   - 按页加载 PDF/Word/Excel
   - _split_page_to_three_levels(): L1(1200) → L2(600) → L3(300)
   - 生成 chunk_id: "{filename}::p{page}::l{level}::{idx}"

4. parent_chunk_store.upsert_documents(L1 + L2)
   - PostgreSQL 插入 + Redis 缓存

5. milvus_writer.write_documents(L3)
   - embedding_service.increment_add_documents(texts) 增加 BM25 统计
   - 生成 dense + sparse 向量
   - Milvus insert
```

### 3. 混合检索 (Hybrid Search)

```python
# rag_utils.py: retrieve_documents
1. 密集向量 + 稀疏向量生成
   dense_emb = embedding_service.embed_query(query)
   sparse_emb = embedding_service.get_sparse_embedding(query)
     ├─ tokenize: 中文单字 + 英文单词
     └─ BM25: score = IDF * (tf * (k1+1)) / (tf + k1*(1-b+b*doc_len/avg))

2. Milvus hybrid_retrieve
   dense_search = AnnSearchRequest(dense_emb, HNSW, IP)
   sparse_search = AnnSearchRequest(sparse_emb, SPARSE_INVERTED_INDEX, IP)
   results = hybrid_search([dense_search, sparse_search], ranker=RRFRanker(k=60))

3. 降级策略
   hybrid 失败 → dense_retrieve (仅稠密)
   dense 失败 → 返回空 []

4. Rerank (可选)
   POST {rerank_host}/v1/rerank
   {"model": "...", "query": "...", "documents": [...], "top_n": 5}

5. Auto-merging
   - 按 parent_chunk_id 分组
   - 同组子块 >= threshold (默认 2) → 替换为父块文本
   - 两段合并：L3→L2, L2→L1
```

### 4. RAG 状态机 (LangGraph)

```
Entry: retrieve_initial
   │
   ├─→ 召回 L3 叶子块 (top_k=5, candidate_k=15)
   ├─→ Rerank 精排
   └─→ Auto-merging 合并
       │
       ▼
   grade_documents
       │
       ├─ yes ──→ END (生成回答)
       │
       └─ no ──→ rewrite_question
                   │
                   ├─ Step-Back (具体 → 抽象)
                   ├─ HyDE (生成假设文档)
                   └─ Complex (两者组合)
                       │
                       ▼
                   retrieve_expanded
                       │
                       └─→ END
```

---

## 开发指南

### 添加新工具 (Tool)

```python
# 1. backend/tools.py
from langchain_core.tools import tool

@tool("new_tool_name")
def new_tool(arg1: str, arg2: int = 10) -> str:
    """工具的描述，会告诉 LLM 什么时候该用它"""
    # 实现逻辑
    return "结果字符串"

# 2. backend/agent.py - create_agent_instance
agent = create_agent(
    model=model,
    tools=[get_current_weather, search_knowledge_base, new_tool],  # 添加
    system_prompt="...可以调用 new_tool 来..."  # 可能需要更新
)

# 3. (可选) 如果工具需要 RAG 步骤可视化
from tools import emit_rag_step
emit_rag_step("🔧", "正在调用新工具...", f"参数：{arg1}")
```

### 修改 RAG 检索策略

```python
# backend/rag_utils.py

# 调整候选召回数
def retrieve_documents(query: str, top_k: int = 5) -> Dict:
    candidate_k = max(top_k * 3, top_k)  # 修改系数

# 调整 Auto-merging 阈值
AUTO_MERGE_THRESHOLD = int(os.getenv("AUTO_MERGE_THRESHOLD", "2"))  # .env 配置

# 修改 Rerank 参数
def _rerank_documents(query: str, docs: List[dict], top_k: int):
    payload = {
        "top_n": min(top_k, len(docs)),  # 调整精排数量
        ...
    }
```

### 调整分块大小

```python
# backend/document_loader.py
def __init__(self, chunk_size: int = 500, chunk_overlap: int = 50):
    # 三级分块尺寸
    level_1_size = max(1200, chunk_size * 2)   # L1 父块
    level_2_size = max(600, chunk_size)        # L2 中间块
    level_3_size = max(300, chunk_size // 2)   # L3 叶子块（向量化）
```

### 环境变量配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `ARK_API_KEY` | - | 字节火山 Ark API Key |
| `MODEL` | - | 主模型（用于生成/Step-Back/HyDE） |
| `GRADE_MODEL` | `gpt-4.1` | 相关性评分模型 |
| `BASE_URL` | - | LLM API 端点 |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | 本地稠密向量模型 |
| `DENSE_EMBEDDING_DIM` | `1024` | 必须与 Milvus 集合维度一致 |
| `RERANK_MODEL` | - | Jina Rerank 模型名 |
| `RERANK_BINDING_HOST` | - | Jina API 端点 |
| `AUTO_MERGE_THRESHOLD` | `2` | Auto-merging 触发阈值 |
| `LEAF_RETRIEVE_LEVEL` | `3` | 检索时叶子层级过滤 |

### 运行测试

```bash
# 测试 Embedding 服务
uv run python test_embedding.py

# 评估 RAG 效果
uv run python test_langsmith_eval.py
```

---

## 故障排查

### Milvus 连接失败
```bash
# 检查 Docker 容器状态
docker compose ps

# 查看 Milvus 日志
docker compose logs -f standalone

# 健康检查
curl http://localhost:9091/healthz
```

### BM25 统计与 Milvus 不同步
症状：稀疏检索结果异常，或删除文档后 IDF 计算错误

解决：
1. 检查 `data/bm25_state.json` 的 `total_docs` 与 Milvus 中 chunk 数量是否一致
2. 如不一致，清空 Milvus 集合后重新上传文档：
```bash
# 通过 API 删除所有文档
curl -X DELETE /documents/{filename} -H "Authorization: Bearer ..."

# 或直接重建集合（会丢失所有向量）
python -c "from milvus_client import MilvusManager; m = MilvusManager(); m.drop_collection()"
```

### RAG 步骤不推送（前端一直显示"正在思考中..."）
检查点：
1. `tools.py::_RAG_STEP_LOOP` 是否在 `set_rag_step_queue()` 中正确捕获
2. `emit_rag_step()` 是否有异常（try-except 吞掉错误）
3. 前端 `script.js` 中 `ragSteps` 数组是否初始化（第 206 行）

### 429 限流错误
后端会转换为 HTTP 429 返回：
```
上游模型服务触发限流/额度限制（429）。请检查账号额度/模型状态。
```
前端需提示用户检查 API Key 额度。

### 密码哈希兼容性问题
- 新用户：PBKDF2-SHA256 (`pbkdf2_sha256$310000$<salt_b64>$<digest_b64>`)
- 历史用户：bcrypt (`$2$...` 或 `$bcrypt...`)
- `auth.py::verify_password()` 自动检测前缀并选择对应验证逻辑

---

## 附录：端口速查

| 服务 | 端口 | 用途 |
|------|------|------|
| FastAPI | 8000 | Web 服务/API 文档 |
| PostgreSQL | 5432 | 关系数据库 |
| Redis | 6379 | 缓存 |
| Milvus | 19530 | 向量检索 |
| Milvus Health | 9091 | 健康检查 |
| MinIO API | 9000 | 对象存储（Milvus 依赖） |
| MinIO Console | 9001 | MinIO 管理界面 |
| Attu | 8080 | Milvus GUI 管理工具 |
| etcd | 2379 | 分布式 KV（Milvus 依赖） |
