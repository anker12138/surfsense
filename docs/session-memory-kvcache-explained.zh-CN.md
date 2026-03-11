# 会话管理、对话历史、LLM 无记忆与 KV Cache 深度解析

> 以 SurfSense 项目的真实代码为例

---

## 一、SurfSense 如何切分"会话"

### 数据模型层

数据库里有一张 `chats` 表，每一行就是一个独立会话（[`surfsense_backend/app/db.py`](../surfsense_backend/app/db.py)）：

```python
class Chat(BaseModel, TimestampMixin):
    __tablename__ = "chats"

    type       = Column(SQLAlchemyEnum(ChatType), ...)  # 类型：QNA 等
    title      = Column(String, ...)                    # 用第一条用户消息当标题
    messages   = Column(JSON, ...)                      # ← 完整对话记录，JSON 数组
    state_version = Column(BigInteger, default=1)

    search_space_id = Column(Integer, ForeignKey("searchspaces.id"), ...)
```

**一个 `Chat` 行 = 一个会话**，`messages` 里存储该会话内所有的 user / assistant 消息。

### URL 路由层

前端通过 URL `path` 区分会话：

```
/dashboard/{search_space_id}/researcher/{chat_id}
```

- `chat_id` 不存在 → 新会话
- `chat_id` 存在 → 继续历史会话

用户发出第一条消息时，前端先调 `POST /chats` 创建数据库记录，拿到 `chat_id`，
然后跳转到新 URL，此后该 `chat_id` 就锁定了这个会话。

---

## 二、对话历史是怎么流转的

### 每次请求的完整链路

```
前端（useChat / @ai-sdk/react）
│
│  每次发送新消息时，把 messages 数组（包含历史）整体发给后端
│
▼
POST /api/v1/chat
body = {
  messages: [
    { role: "user",      content: "第1轮问题" },
    { role: "assistant", content: "第1轮回答" },
    { role: "user",      content: "第2轮问题" },  ← 最新消息
  ]
}
```

### 后端处理（[`chats_routes.py`](../surfsense_backend/app/routes/chats_routes.py)）

```python
# 最后一条是本次用户问题
user_query = messages[-1]["content"]

# 其余全部是历史，转为 LangChain 消息对象
langchain_chat_history = []
for message in messages[:-1]:          # ← 历史 = 除最后一条之外的所有消息
    if message["role"] == "user":
        langchain_chat_history.append(HumanMessage(content=message["content"]))
    elif message["role"] == "assistant":
        langchain_chat_history.append(AIMessage(content=message["content"]))
```

历史被传进 Agent 的 `State.chat_history`，在两个地方使用：
1. **改写查询**（`reformulate_user_query` 节点）——把历史上下文带入，让改写后的查询更精准
2. **生成回答**（`answer_question` 节点）——把历史拼进 System Prompt，让 LLM"知道"之前聊了什么

### 回答完成后持久化

当最新一条 assistant 消息出现时，前端自动调 `PUT /chats/{id}` 把最新 `messages` 数组（含新一轮对话）存回数据库：

```typescript
// researcher/page.tsx
useEffect(() => {
    if (handler.messages[last]?.role === "assistant") {
        updateChat({ messages: handler.messages, ... });
    }
}, [handler.messages, handler.status]);
```

**结论**：历史不在后端内存里，每次请求都是无状态的——前端把完整历史带来，用完就走。

---

## 三、每次提问都重新检索数据库，是否浪费？

### 现状：是的，每次都全量检索

每次用户提问，Agent 都会执行完整的检索流程：

```
用户提问
  │
  ▼
reformulate_user_query  ← LLM 改写查询（1次 LLM 调用）
  │
  ▼
fetch_relevant_documents
  │ 对每个 connector 执行：向量搜索 + 全文搜索 + RRF 融合
  └─ YOUTUBE_VIDEO / EXTENSION / FILE / SLACK / NOTION / ... (各自独立搜索)
  │
  ▼
rerank_documents        ← 重排序
  │
  ▼
answer_question         ← LLM 生成回答
```

没有任何检索结果缓存跨 turn 复用。

### 为什么不缓存？

这其实是有意为之的权衡，理由充分：

| 考量 | 说明 |
|------|------|
| **查询语义不同** | 即使第 2 问和第 1 问话题相近，改写后的查询词是不同的；缓存命中率低 |
| **精准性优先** | 每次重新检索确保 LLM 看到和本次问题最相关的文档，而不是复用上轮文档 |
| **实现简单** | 避免缓存失效、缓存污染等工程复杂性 |
| **知识库动态变化** | 用户随时可能新增文档，缓存旧结果会漏掉新内容 |

### 如果想优化

- **语义缓存**：对查询做 embedding，如果两次查询语义距离 < 阈值，复用上次检索结果（Redis 或内存做 LRU）
- **按 turn 传递 `reranked_documents`**：如用户追问同一主题，可在状态里携带上轮文档，跳过重新检索
- **Connector 级别限速**：对外部 API 类 connector（Tavily、Linkup 等）做请求去重

---

## 四、LLM 没有记忆——记忆完全由应用层管理

### LLM 本质上是无状态函数

```
LLM = f(input_tokens) → output_tokens
```

每次调用 `llm.ainvoke(messages)` 都是一次全新的前向推理，**完全独立**，没有任何上一次调用的残留。

### SurfSense 如何"模拟"记忆

把历史对话拼进提示词，欺骗 LLM "好像"记得之前的内容：

```python
# qna_agent/nodes.py  answer_question()
system_prompt = _format_system_prompt(
    full_citation_prompt_template,
    chat_history_str,   # ← 历史对话被序列化成字符串，嵌入 System Prompt
    language,
)

messages_with_chat_history = [
    SystemMessage(content=system_prompt),   # 含历史
    HumanMessage(content=human_message_content),  # 当前问题
]

response = await llm.ainvoke(messages_with_chat_history)
```

**本质是**：把历史当作普通文本 token 传进去，LLM 在 attention 机制下"看到"了历史，
并在生成时把历史当作上下文参考。

### 代价与上限

- **Token 消耗随历史增长**：对话轮数越多，每次请求的 prompt 越长，费用越高
- **Context Window 限制**：历史过长时必须截断（SurfSense 通过 `optimize_documents_for_token_limit` 处理文档，但历史本身没有自动截断）
- **精度损耗**：极长上下文里 LLM 对早期内容的注意力会衰减（中间位置信息丢失，"Lost in the Middle" 现象）

---

## 五、KV Cache：同一 LLM 连续调用能复用上下文吗？

这是最值得深究的问题。

### 什么是 KV Cache

Transformer 推理时，每个 token 都要计算 Key（K）和 Value（V）矩阵。
对于已经计算过的 token 序列，可以把 K/V 缓存起来，下次推理如果前缀相同，直接复用，
不需要重新计算这部分的注意力。

```
第 1 次推理：[System Prompt | 历史1 | 问题1]
              ──────────────────────────────
              全部需要计算 K/V，存入 Cache

第 2 次推理：[System Prompt | 历史1 | 答案1 | 问题2]
              ─────────────────────
              这部分前缀和第 1 次完全相同 → 直接复用缓存
                                           ───────────
                                           只需计算新增部分
```

这不仅加速推理，也节省计算成本（部分 API 提供商对缓存命中 token 收取更低费用）。

### SurfSense 的情况：**无法复用 KV Cache**

原因：SurfSense 采用"外部 LLM API"（OpenAI、Anthropic、本地 Ollama 等），
每次调用都是一个**独立的 HTTP 请求**。

```
HTTP 请求 1：POST /completions
  体：[System Prompt | 历史1 | 问题1]
  ↓
LLM API 推理，在服务器上计算 K/V，返回答案
  ↓
HTTP 连接关闭，服务器端 K/V Cache 是否保留取决于 API 服务商

HTTP 请求 2：POST /completions（新的 TCP 连接）
  体：[System Prompt | 历史1 | 答案1 | 问题2]
  ↓
是否命中 KV Cache 完全取决于 API 服务商是否实现了"Prompt Cache"
```

### 三种情况对比

| 部署方式 | KV Cache 能否跨请求复用？ | 说明 |
|----------|--------------------------|------|
| **OpenAI API** | ✅ 部分支持（Prompt Caching） | 前缀超过 1024 token 且稳定时自动命中，cached token 费用是正常的 50% |
| **Anthropic API** | ✅ 支持（Cache Control） | 需显式标记 `cache_control`，最长缓存 5 分钟 |
| **本地 vLLM / Ollama** | ✅ 支持 | vLLM 默认开启 prefix caching；Ollama 也有 context caching |
| **LM Studio / llama.cpp** | ⚠️ 进程内有 | 同一进程内的连续调用可复用，跨进程或重启后失效 |
| **完全无状态 Serverless** | ❌ 不可复用 | 每个请求在新实例上运行 |

### SurfSense 为什么很难命中 KV Cache

即使 API 支持 Prompt Cache，SurfSense 每次请求的 prompt 也会因为：

1. **检索文档不同**：每次 `fetch_relevant_documents` 返回的文档可能不同，
   文档内容被拼进 Human Message，导致 prompt 不完全相同
2. **历史追加**：每轮对话都在历史末尾追加新内容，前缀相同但后缀在变化

**能命中缓存的部分**：System Prompt（含静态 QnA Base Prompt）几乎每次都相同，
这部分如果被 API 服务商缓存，可以节省 System Prompt 部分的计算。

### 如果本地部署（vLLM）

vLLM 使用 **Automatic Prefix Caching（APC）**，对 SurfSense 最友好：

```
turn 1: [system_prompt | chat_history="" | docs_A | question_1]
                 ↑
          这段 token 序列计算后，K/V 存入 block cache

turn 2: [system_prompt | chat_history="Q1+A1" | docs_B | question_2]
         ──────────────
         system_prompt 部分命中缓存（如果 generation config 一致）
         chat_history & docs 部分不同，需要重新计算
```

---

## 六、全景流程图

```
用户发消息（第 N 轮）
│
│  前端：把 messages[0..N-1] (历史) + messages[N] (本问) 一起 POST 给后端
▼
FastAPI /chat 路由
│  从 messages 里切出 chat_history = messages[:-1]
│  user_query = messages[-1]["content"]
▼
LangGraph researcher graph
│
├── reformulate_user_query
│     输入：user_query + chat_history（字符串化）
│     输出：reformulated_query
│     LLM 调用 #1（strategic LLM）
│
└── handle_qna_workflow
      │
      ├── fetch_relevant_documents（每次都执行，无缓存）
      │     用 reformulated_query + user_query 在所有 connector 里检索
      │
      ├── rerank_documents
      │     用 user_query + reformulated_query 作为 query 重排文档
      │
      └── answer_question
            输入：检索文档 + chat_history（嵌入 System Prompt）+ user_query
            LLM 调用 #2（fast LLM）
            ainvoke = 一次性等待完整答案（非 token 流式）

前端收到流式回答，追加到 messages 数组，调 updateChat 存回数据库
```

---

## 七、核心结论

| 问题 | 结论 |
|------|------|
| 会话如何区分？ | 每个 `chat_id`（`chats` 表的一行）是一个独立会话；`messages` JSON 字段存储该会话的全部对话 |
| 历史如何处理？ | 前端每次请求都把完整历史带来；后端无状态，不在内存里保存任何会话状态 |
| 每次都重新检索吗？ | 是的，没有跨 turn 的检索缓存；每次提问都执行完整的向量+全文检索 |
| LLM 的"记忆"从哪里来？ | 完全由应用层注入——历史对话被序列化后塞进 System Prompt，LLM 本身完全无状态 |
| KV Cache 能复用吗？ | **同一次请求内**可以（前缀 attention 不重复计算）；**跨请求**能否复用取决于 API 服务商（OpenAI/Anthropic 支持 Prompt Cache）；本地 vLLM 开启 APC 后 system_prompt 部分能命中；但因为每次文档内容不同，整体命中率有限 |
