
## LangGraph 的 `astream` 原理


- llm.ainvoke() 是等待LLM生成完整回答后才返回，得到一个完整的AIMessage对象
- llm.astream() 是边生成边返回，每当LLM生成一个或几个token就会yield一个AIMessageChunk对象
- graph.astream()返回的是一个异步迭代器
     - 当一个node执行完毕并把结果合并到state后，graph.astream()会yield出当前完整state的快照
- for 异步迭代器 = 反复调用 __anext__()，
- async  for  异步迭代器 = 异步的反复调用 __anext__()
- async for _chunk_type, chunk in qna_agent_graph.astream(...)
    - 等于 异步的反复读取 graph节点的state快照

### Graph 是什么

LangGraph 把若干**节点函数**（node）和**有向边**（edge）组成一张有向无环图（DAG）。

QnA 子图的结构：

```
__start__
    │
    ▼
rerank_documents          ← 对文档重排序
    │
    ▼
answer_question           ← 调用 LLM 回答问题
    │
    ▼
__end__
```

每个节点函数都是一个 `async def`，接收 `state` 和 `config`，返回一个 dict（用于更新 state）。

### `astream` 的执行机制

```python
async for _chunk_type, chunk in graph.astream(state, config, stream_mode=["values"]):
    ...
```

`stream_mode=["values"]` 表示：**每当一个节点执行完毕并把返回值合并到 state 后，立即 `yield` 出当前完整 state 的快照**。
执行时序如下：
节点之间如果没有依赖关系，可以并行执行；有依赖关系的节点会等待前置节点完成后才开始执行。每当一个节点完成并更新 state 后，`astream` 就会 yield 出当前 state 的快照。

```
graph.astream() 开始
│
├─ 执行 rerank_documents 节点
│     await DatabaseQuery / RerankerService
│     返回 {"reranked_documents": [...]}
│     ── state 更新 ──>  yield (chunk_type, {"reranked_documents": [...]})  ←── 外层 async for 收到第 1 个 chunk
│
├─ 执行 answer_question 节点
│     await llm.ainvoke(messages)   ← 等待 LLM 返回完整回答
│     返回 {"final_answer": "...完整文本..."}
│     ── state 更新 ──>  yield (chunk_type, {"reranked_documents": [...], "final_answer": "..."})  ←── 外层 async for 收到第 2 个 chunk
│
└─ 图执行结束，astream 结束
```

**关键点**：`stream_mode=["values"]` 是**节点粒度的流式**，不是 LLM token 粒度的流式。  
每个节点跑完才 yield 一次，而不是 LLM 每生成一个 token 就 yield。

---

## 七、`answer_question` 的答案是怎么"走"到 `astream` 的

### Step 1：`answer_question` 节点调用 LLM（`ainvoke`，阻塞等待完整响应）

```python
# qna_agent/nodes.py  answer_question()
response = await llm.ainvoke(messages_with_chat_history)
final_answer = response.content          # 完整字符串，一次性拿到
return {"final_answer": final_answer}    # 写回 state
```

> **注意**：这里用的是 `ainvoke`（不是 `astream`），所以 LLM 把完整回答生成完才返回。  
> `final_answer` 到这里是一个完整的字符串，不是分 token 到达的。

### Step 2：LangGraph 把返回值合并进 state，然后 yield

LangGraph 在 `answer_question` 节点返回后：
1. 把 `{"final_answer": "..."}` 合并进 `State`
2. 通过 `astream` 的内部异步生成器 `yield` 出当前完整 state

```
answer_question()  返回 dict
        │
        ▼
LangGraph 框架合并 state
        │
        ▼
astream 内部 yield 出 state 快照
        │
        ▼
外层 async for 收到 chunk
```

### Step 3：外层 `handle_qna_workflow` 提取 delta

```python
async for _chunk_type, chunk in qna_agent_graph.astream(...):
    if "final_answer" in chunk:
        new_content = chunk["final_answer"]
        # 虽然 final_answer 是一次性到达的，但用 delta 方式处理是防御性编码
        # 以防未来改成 token 流式时不需要修改外层逻辑
        delta = new_content[len(complete_content):]
        complete_content = new_content
        writer({"yield_value": streaming_service.format_text_chunk(delta)})
```

**delta 计算的意义**：因为 `stream_mode=["values"]` 每次 yield 的是完整 state，  
如果同一个 field 在多次 yield 中都出现，用 `new[len(old):]` 可以只取新增部分，避免重复输出。  
（在当前实现里 `final_answer` 只在最后那次 yield 中出现，所以 delta = 完整内容。）

---

## 八、完整调用链全景图

```
用户发起请求（HTTP）
        │
        ▼
FastAPI 路由（async def）
        │  await
        ▼
handle_qna_workflow（主图节点，async def）
        │  async for chunk in
        ▼
qna_agent_graph.astream(stream_mode=["values"])
        │  异步迭代，节点粒度 yield
        ├─────────────────────────────────────────┐
        │  节点1：rerank_documents                 │
        │    await reranker_service.rerank(...)    │
        │    yield state{"reranked_documents":[]}  │  ← chunk 1
        │                                          │
        │  节点2：answer_question                   │
        │    await llm.ainvoke(messages)           │
        │    yield state{"final_answer":"..."}     │  ← chunk 2
        └─────────────────────────────────────────┘
        │
        ▼  读取 chunk 2 中的 final_answer
writer(format_text_chunk(delta))
        │
        ▼
HTTP 流式响应（SSE / chunked）推送到前端
```

---

## 、`ainvoke` vs `astream`（LLM 层面）

| | `ainvoke` | `astream` |
|---|---|---|
| **使用场景** | 需要完整响应后再继续处理 | 需要边生成边展示（流式打字效果） |
| **返回** | `await` 得到完整 `AIMessage` | `async for` 逐 token 得到 `AIMessageChunk` |
| **当前代码** | `answer_question` 用此方式 | `astream` 是 LangGraph 图层面的，与 LLM 层面的 astream 不同 |

如果想实现**真正的 token 级流式**（打字效果），`answer_question` 里需要改为：

```python
# token 级流式（当前未启用）
async for token_chunk in llm.astream(messages):
    # 每生成一个或几个 token 就会进来一次
    writer({"yield_value": streaming_service.format_text_chunk(token_chunk.content)})
```


## 关键结论

1. **`astream` 是节点粒度的流式，不是 token 粒度的流式。**  
   `final_answer` 是 `answer_question` 节点跑完后一次性 yield 出来的。

2. **`answer_question` 里用的是 `ainvoke`**，等 LLM 生成完整回答才返回。  
   因此前端看到的是"回答突然全部出现"，而不是逐字打出。

3. **如果想要逐 token 的打字流效果**，需要把 `ainvoke` 换成 `llm.astream`，  
   并在节点内直接通过 `writer` 推送每个 token 的 delta（绕过 LangGraph 的 state 流式机制）。

4. **`async for ... in graph.astream(...)` 能"逐步"读取的本质**：  
   是事件循环在等待每个节点 I/O 时切换协程，节点结束就 yield 一个快照，  
   而不是因为 LLM 在逐 token 输出。