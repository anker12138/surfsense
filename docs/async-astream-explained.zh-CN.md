# 异步、协程、async/await、astream 完全解析

> 以 SurfSense 中 `qna_agent_graph.astream` 如何逐步获取 `final_answer` 为例

---

## 一、从一个问题出发

在 [`surfsense_backend/app/agents/researcher/nodes.py`](../surfsense_backend/app/agents/researcher/nodes.py) 里有这样一段代码：

```python
async for _chunk_type, chunk in qna_agent_graph.astream(
    qna_state, qna_config, stream_mode=["values"]
):
    if "final_answer" in chunk:
        new_content = chunk["final_answer"]
        delta = new_content[len(complete_content):]
        complete_content = new_content
        writer({"yield_value": streaming_service.format_text_chunk(delta)})
```

**问题：`qna_agent_graph.astream` 是如何"逐步"吐出数据的？`final_answer` 又是怎么从 LLM 一路传到这里的？**

要回答这个问题，需要先把几个基础概念搞清楚。

---

## 二、同步 vs 异步：为什么要有异步

### 同步（阻塞）

```python
def 泡茶():
    烧水()        # 等水烧开，什么都不能做
    放茶叶()
    等待浸泡()    # 等 3 分钟，什么都不能做
    return 茶
```

线程在 `烧水()` 期间什么也做不了，就是**阻塞**。

### 异步（非阻塞）

```python
async def 泡茶():
    await 烧水()    # "我先去干别的，水开了通知我"
    放茶叶()
    await 等待浸泡() # "我先去干别的，时间到了通知我"
    return 茶
```

`await` 的语义是：**把控制权交还给事件循环，让事件循环去执行其他任务，等这个 I/O 操作完成后再回来继续**。

对 Web 后端来说意义重大：烧水（等数据库/等 LLM 响应）时，服务器可以同时处理其他用户的请求，而不是傻等。

---

## 三、协程（Coroutine）

用 `async def` 定义的函数就是**协程函数**，调用它得到的是一个**协程对象**，不会立即执行。

```python
async def say_hello():
    print("hello")

coro = say_hello()   # 此时什么都没发生，coro 只是一个协程对象
await coro           # 真正执行，打印 "hello"
```
协程底层是依赖什么机制实现的呢？ 线程调度吗？ asyncio是怎样实现的协程调度的呢？
写一个简单的c语言或python代码说明底层机制


async的函数 如果把 await调用，编译后执行async函数的运行指令就会改为定时检测事件是否完成如果完成了就继续执行，如果没有完成就继续执行其他的协程，等到完成了再切回来继续执行这个协程，这样就实现了异步非阻塞的效果。

## 协程的底层机制

### Python asyncio 的事件循环实现

协程并非由线程调度器驱动，而是由**事件循环（Event Loop）** 统一管理。以下是简化的工作原理：

#### Python 伪代码示例

```python
# 简化的事件循环实现
class SimpleEventLoop:
    def __init__(self):
        self.ready_queue = []      # 就绪的协程
        self.waiting_tasks = {}    # 等待中的任务
    
    def run_until_complete(self, coro):
        """运行协程直到完成"""
        task = Task(coro)
        self.ready_queue.append(task)
        
        while self.ready_queue or self.waiting_tasks:
            # Step 1: 执行就绪协程
            while self.ready_queue:
                task = self.ready_queue.pop(0)
                try:
                    # 驱动协程执行，直到遇到 await
                    next_await = task.coro.send(None)
                except StopIteration as e:
                    # 协程执行完毕
                    return e.value
                else:
                    # 协程暂停在 await 处，登记到等待队列
                    self.waiting_tasks[task] = next_await
            
            # Step 2: 检查等待中的 I/O 是否完成（poll）
            for task, io_obj in list(self.waiting_tasks.items()):
                if io_obj.is_ready():
                    # I/O 完成，恢复协程
                    self.waiting_tasks.pop(task)
                    self.ready_queue.append(task)

# 协程函数
async def fetch_data():
    result = await db.query()  # 暂停在这里
    return result

# 运行
loop = SimpleEventLoop()
loop.run_until_complete(fetch_data())
```

#### 执行时序图

```
协程实例 fetch_data()
    │
    ├─ Task.send(None)  执行至第一个 await
    │    │
    │    ├─> await db.query()
    │    │        ↓
    │    │    【暂停】协程保存执行状态（栈、局部变量等）
    │    │        ↓
    │    │    登记到 waiting_tasks（等待数据库响应）
    │    │        ↓
    │    │    事件循环转向处理其他就绪协程 ←── 这就是"异步"的核心
    │    └─────────────────────────────┐
    │                                  │
    ├─ 事件循环 poll 检测                  │（100ms 后数据库返回结果）
    │    │ I/O 完成 ✓                    │
    │    │ 将 fetch_data Task 放回 ready  │
    │    └───────────────────────────┬──┘
    │                                │
    ├─ Task.send(db_result)  恢复执行│
    │    │  从暂停点继续             │
    │    └─> result = db_result      │
    │        return result           │
    │        StopIteration ──────────┘
```

### C 语言示例：栈保存机制

```c
// 协程的核心：保存执行状态
typedef struct {
    void *stack_ptr;           // 栈指针（栈顶）
    void *frame_ptr;           // 帧指针（当前函数栈帧）
    int state;                 // 暂停点标记
} Coroutine;

// 协程切换伪代码
void coroutine_yield(Coroutine *coro) {
    // 保存当前 CPU 状态
    // - 栈指针 ESP
    // - 帧指针 EBP  
    // - 指令指针 EIP（暂停位置）
    __asm__ ("movl %%esp, %0" : "=r"(coro->stack_ptr));
    __asm__ ("movl %%ebp, %0" : "=r"(coro->frame_ptr));
    
    // 事件循环转向其他协程
    event_loop_schedule_other();
}

void coroutine_resume(Coroutine *coro) {
    // 恢复 CPU 状态
    __asm__ ("movl %0, %%esp" : : "r"(coro->stack_ptr));
    __asm__ ("movl %0, %%ebp" : : "r"(coro->frame_ptr));
    
    // 从保存的指令指针继续执行
    __asm__ ("jmp *%0" : : "r"(coro->eip));
}
```

#### 栈关键点

- **每个协程有独立的栈**（或共用，取决于实现）
- **`await` 时保存 SP/BP/IP**，事件循环切换到其他协程的栈
- **I/O 完成后恢复这些寄存器**，CPU 继续从暂停位置执行
- **无需操作系统线程调度**，完全在用户态（asyncio）

### 对比：线程 vs 协程

| 特性 | 线程 | 协程 |
|------|------|------|
| **调度** | 操作系统内核（抢占式） | 事件循环（协作式） |
| **上下文切换** | 自动，任何时刻 | 手动，仅在 `await` 时 |
| **内存成本** | ~1MB（栈） | ~1KB（保存状态） |
| **数量** | 数千个 | 数百万个 |
| **同步原语** | Lock、Condition 等 | asyncio.Lock、Event 等 |

---




协程的核心特性：
- **可暂停**：遇到 `await` 时暂停，把控制权让出去
- **可恢复**：等待的事情完成后，事件循环把控制权还回来，从暂停点继续
- **轻量**：不是线程，创建成千上万个协程成本极低

```
协程函数  ──调用──>  协程对象（暂停的）  ──await──>  执行 / 暂停 / 恢复
```

---

## 四、`async` 和 `await` 的关系

| 关键字 | 作用 | 比喻 |
|--------|------|------|
| `async def` | 声明这个函数是一个协程函数 | 标记"这个任务可以挂起" |
| `await expr` | 挂起当前协程，等 `expr` 完成后返回其结果 | 挂一个"等完了叫我"的牌子，然后去干别的 |

**规则：`await` 只能出现在 `async def` 内部。**

```python
async def fetch_data():
    # await 等待 DB 查询，同时事件循环可以处理其他协程
    result = await db.execute(query)
    return result
```

---

## 五、普通迭代器 vs 异步迭代器

### 普通迭代器（同步）

```python
for item in [1, 2, 3]:
    print(item)
```

每次调用 `__next__()` 立即返回下一个值，**不能暂停**。

### 异步迭代器

```python
async for chunk in some_async_generator():
    print(chunk)
```

每次迭代都会调用 `__anext__()`，这是一个**可等待对象**（awaitable）。  
事件循环可以在每次 `await __anext__()` 时去处理其他协程。  
当异步生成器 `yield` 出一个值，`__anext__()` 的等待结束，迭代继续。

**`astream` 返回的就是一个异步迭代器。**

---

## 六、LangGraph 的 `astream` 原理

### 6.1 Graph 是什么

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

### 6.2 `astream` 的执行机制

```python
async for _chunk_type, chunk in graph.astream(state, config, stream_mode=["values"]):
    ...
```

`stream_mode=["values"]` 表示：**每当一个节点执行完毕并把返回值合并到 state 后，立即 `yield` 出当前完整 state 的快照**。

执行时序如下：

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

## 九、`ainvoke` vs `astream`（LLM 层面）

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

---

## 十、概念汇总

```
协程函数（async def）
    ↓ 调用
协程对象
    ↓ await / 放入事件循环
事件循环驱动执行
    ↓ 遇到 await I/O
暂停当前协程，执行其他协程
    ↓ I/O 完成
恢复协程，继续执行

异步迭代器（实现 __aiter__ / __anext__）
    ↓ async for
每次迭代 await __anext__()，可在等待期间运行其他协程
    ↓ 生成器内部 yield
调用方取到一个值，循环体执行一次

LangGraph astream
= 把图的每个节点执行封装成异步生成器
= 每个节点完成 → yield 一次 state 快照
= stream_mode=["values"] → 每次 yield 完整 state
```

---

## 十一、关键结论

1. **`astream` 是节点粒度的流式，不是 token 粒度的流式。**  
   `final_answer` 是 `answer_question` 节点跑完后一次性 yield 出来的。

2. **`answer_question` 里用的是 `ainvoke`**，等 LLM 生成完整回答才返回。  
   因此前端看到的是"回答突然全部出现"，而不是逐字打出。

3. **如果想要逐 token 的打字流效果**，需要把 `ainvoke` 换成 `llm.astream`，  
   并在节点内直接通过 `writer` 推送每个 token 的 delta（绕过 LangGraph 的 state 流式机制）。

4. **`async for ... in graph.astream(...)` 能"逐步"读取的本质**：  
   是事件循环在等待每个节点 I/O 时切换协程，节点结束就 yield 一个快照，  
   而不是因为 LLM 在逐 token 输出。
