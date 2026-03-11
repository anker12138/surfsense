# SurfSense 项目 LLM 调用位置全景分析

> 本文档系统梳理了 SurfSense 后端 (`surfsense_backend`) 中所有调用 LLM 的位置，说明每处调用完成的任务、输入内容，以及输出的"沉默作用"（即输出不直接返回给用户，而是作为下游流程的中间结果）。

---

## 概览

| # | 文件 | 函数 / 节点 | 调用方式 | 所用 LLM 角色 | 触发场景 |
|---|------|------------|---------|-------------|---------|
| 1 | `app/services/llm_service.py` | `validate_llm_config()` | `llm.ainvoke()` | 任意 | 保存/更新 LLM 配置时验证 |
| 2 | `app/services/query_service.py` | `QueryService.reformulate_query_with_chat_history()` | `llm.agenerate()` | **Strategic LLM** | 用户发起对话，有历史记录时 |
| 3 | `app/agents/researcher/qna_agent/nodes.py` | `answer_question()` | `llm.ainvoke()` | **Fast LLM** | QnA 工作流生成回答 |
| 4 | `app/agents/researcher/nodes.py` | `generate_further_questions()` | `llm.ainvoke()` | **Fast LLM** | 对话结束后生成推荐追问 |
| 5 | `app/agents/podcaster/nodes.py` | `create_podcast_transcript()` | `llm.ainvoke()` | **Long Context LLM** | 用户触发播客生成任务 |
| 6 | `app/services/docling_service.py` | `process_large_document_summary()` 小文档直接路径 | `summary_chain.ainvoke()` | **Long Context LLM** | 文档摄入（小文档 ≤10万字符） |
| 7 | `app/services/docling_service.py` | `process_large_document_summary()` 大文档分块路径 | `chunk_chain.ainvoke()` × N | **Long Context LLM** | 文档摄入（大文档，每块约 8000 tokens） |
| 8 | `app/services/docling_service.py` | `process_large_document_summary()` 大文档合并路径 | `combine_chain.ainvoke()` | **Long Context LLM** | 大文档所有块摘要生成后合并 |
| 9 | `app/utils/document_converters.py` | `generate_document_summary()` | `summary_chain.ainvoke()` | **Long Context LLM** | 文档摄入（非 Docling 路径） |
| 10 | `app/tasks/document_processors/file_processors.py` | `process_file_in_background()` → `atranscription()` | LiteLLM `atranscription()` | **STT 模型**（非对话 LLM） | 音频文件上传时语音转文字 |
| 11 | `app/agents/podcaster/nodes.py` | `create_merged_podcast_audio()` → `aspeech()` | LiteLLM `aspeech()` | **TTS 模型**（非对话 LLM） | 播客生成任务合成音频 |

---

## 详细说明

### 1. LLM 配置验证 — `validate_llm_config()`

**文件**: [`app/services/llm_service.py`](../surfsense_backend/app/services/llm_service.py#L129)

**触发时机**: 用户通过 API 创建或更新 LLM 配置（`POST /llm-configs`、`PUT /llm-configs/{id}`）时，后端先调用此函数验证配置可用性。

**调用方式**:
```python
response = await llm.ainvoke([test_message])
```

**输入**:
- `test_message = HumanMessage(content="Hello!")`（单条测试消息）
- LLM 实例由候选配置（provider、model、api_key、api_base、litellm_params）实时构建

**输出的沉默作用**:
输出（`response`）**仅用于判断调用是否成功**，不做任何内容使用。若抛出异常则向 API 返回 422 错误；若成功则配置被持久化到数据库。输出内容本身被完全丢弃。

---

### 2. 用户查询重构 — `QueryService.reformulate_query_with_chat_history()`

**文件**: [`app/services/query_service.py`](../surfsense_backend/app/services/query_service.py#L16)

**触发时机**: `researcher` 图的 `reformulate_user_query` 节点执行时，当对话历史不为空（多轮对话场景）。

**调用方式**:
```python
response = await llm.agenerate(messages=[[system_message, human_message]])
reformulated_query = response.generations[0][0].text.strip()
```

**输入**:

| 消息类型 | 内容要点 |
|---------|---------|
| `SystemMessage` | 当前日期、查询优化专家角色设定、对话历史（chat history）、查询重构指南（增强具体性、消歧、扩展关键词、分解复杂问题、保持用户意图）、约束：仅输出重构后的查询字符串 |
| `HumanMessage` | `"Reformulate this query for better research results: {user_query}"` |

所用 LLM：**Strategic LLM**（用户为该搜索空间配置的"战略级"模型）。

**输出的沉默作用**:
输出的重构查询字符串（`reformulated_query`）**不展示给用户**，直接作为后续多连接器向量/全文检索的查询参数传入 `fetch_relevant_documents()`，用于提升文档检索召回率和精度。

---

### 3. Q&A 问答生成 — `answer_question()`

**文件**: [`app/agents/researcher/qna_agent/nodes.py`](../surfsense_backend/app/agents/researcher/qna_agent/nodes.py#L283)

**触发时机**: QnA 工作流（`handle_qna_workflow` 节点调用 `qna_agent_graph.astream()`）中的核心生成节点。

**调用方式**:
```python
response = await llm.ainvoke(messages_with_chat_history)
final_answer = response.content
```

**输入**:

| 消息类型 | 内容要点 |
|---------|---------|
| `SystemMessage` | QnA 基础提示（角色、引用格式 `[citation:source_id]`、语言指令）+ 引用规则（可选，按搜索空间配置）+ 自定义指令（可选，用户设定）+ 对话历史 |
| `HumanMessage` | 检索到的文档内容（格式化为 `<sources>` 块，经过 token 优化裁剪）+ 用户问题 |

- 所用 LLM：**Fast LLM**
- 文档先经过 `RerankerService` 重排序，再按 token 上限优化截断后传入

**输出的沉默作用**:
`final_answer` 字符串通过 `StreamWriter` 以增量 delta 形式流式推送到前端，并存储到 `State.final_written_report`，最终写入对话历史（`Chat.messages`）持久化到数据库。输出内容**是用户可见的最终回答**，但 LLM 本身的调用过程（中间 token、生成过程）对用户不可见。

---

### 4. 推荐追问生成 — `generate_further_questions()`

**文件**: [`app/agents/researcher/nodes.py`](../surfsense_backend/app/agents/researcher/nodes.py#L1637)

**触发时机**: Researcher 图的末端节点，在生成主回答后自动执行。

**调用方式**:
```python
response = await llm.ainvoke(messages)
```

**输入**:

| 消息类型 | 内容要点 |
|---------|---------|
| `SystemMessage` | `get_further_questions_system_prompt()` — 当前日期、角色定义（专注生成上下文相关追问）、输入说明（chat_history + available_documents）、输出格式（JSON，含 `further_questions` 数组）、分析指引 |
| `HumanMessage` | 完整对话历史（XML 格式）+ 可检索到的文档列表（metadata + content 片段） |

- 所用 LLM：**Fast LLM**

**输出的沉默作用**:
LLM 响应为 JSON 字符串，经解析后提取 `further_questions` 列表。这些问题**不作为回答内容展示**，而是作为 UI 可点击的"推荐追问"卡片推送到前端，存储在对话状态中以便用户快速继续提问。

---

### 5. 播客文稿生成 — `create_podcast_transcript()`

**文件**: [`app/agents/podcaster/nodes.py`](../surfsense_backend/app/agents/podcaster/nodes.py#L53)

**触发时机**: Celery 后台任务 `podcast_tasks.py` 调用播客器图（`podcaster_graph.ainvoke()`）时执行。

**调用方式**:
```python
llm_response = await llm.ainvoke(messages)
podcast_transcript = PodcastTranscripts.model_validate(json.loads(llm_response.content))
```

**输入**:

| 消息类型 | 内容要点 |
|---------|---------|
| `SystemMessage` | `get_podcast_generation_prompt(user_prompt)` — 播客脚本生成指令、对话风格要求、两位主持人设定、自定义用户提示 |
| `HumanMessage` | `<source_content>{state.source_content}</source_content>` — 知识源原始内容（可能是多个文档的合并文本） |

- 所用 LLM：**Long Context LLM**（需要处理大量源内容）

**输出的沉默作用**:
LLM 输出为 JSON 格式播客稿（`PodcastTranscripts`），包含每段对话的 `speaker_id` 和 `dialog`。该文稿**不直接展示给用户**，而是作为 `create_merged_podcast_audio()` 节点的输入，驱动 TTS 逐段合成音频，最终合并为可下载的 MP3 播客文件。

---

### 6. 文档摘要生成（直接路径）— 小文档

**文件**: [`app/services/docling_service.py`](../surfsense_backend/app/services/docling_service.py#L250)

**触发时机**: 文档摄入流程，文档内容 ≤ 100,000 字符时走直接路径。

**调用方式**:
```python
summary_chain = SUMMARY_PROMPT_TEMPLATE | llm
result = await summary_chain.ainvoke({"document": content})
```

**输入**:
- **Prompt**: `SUMMARY_PROMPT_TEMPLATE`（专业分析师角色、准确性/客观性/全面性原则、Markdown 格式输出规范）
- **`{document}`**: 完整文档内容

**输出的沉默作用**:
`result.content` 作为文档的 `summary_content`，**不直接展示给用户**，而是：
1. 存储为 `Document.content`（文档级向量存储的语义内容）
2. 生成 embedding 后存入 `Document.embedding`（用于向量相似度检索）

该摘要是后续 RAG 检索"文档级索引"的核心内容。

---

### 7. 文档摘要生成（分块路径）— 大文档每块

**文件**: [`app/services/docling_service.py`](../surfsense_backend/app/services/docling_service.py#L308)

**触发时机**: 文档内容 > 100,000 字符时，使用 `RecursiveChunker + OverlapRefinery` 将文档切分为约 8000 token 的块，对每块单独调用 LLM。

**调用方式**:
```python
chunk_chain = chunk_template | llm
chunk_result = await chunk_chain.ainvoke({
    "chunk": chunk.text,
    "chunk_number": i,
    "total_chunks": total_chunks,
})
```

**输入**:
- **Prompt**: 内联 `chunk_template` — 当前块编号/总块数、要求为该块生成综合摘要
- **`{chunk}`**: 单个文档块文本（含 10% 重叠上下文）

**输出的沉默作用**:
每块摘要（`chunk_summary`）累积到 `chunk_summaries` 列表，**不直接使用**，全部传入下一步合并调用（调用 #8）生成最终文档摘要。

---

### 8. 文档摘要生成（合并路径）— 大文档汇总

**文件**: [`app/services/docling_service.py`](../surfsense_backend/app/services/docling_service.py#L350)

**触发时机**: 大文档所有块的摘要生成完毕后，执行最终合并步骤。

**调用方式**:
```python
combine_chain = combine_template | llm
final_result = await combine_chain.ainvoke({
    "summaries": combined_summaries,
    "document_title": document_title,
})
```

**输入**:
- **Prompt**: `combine_template` — 合并多个章节摘要为统一文档摘要的指令（逻辑连贯、去重、全面覆盖）
- **`{summaries}`**: 所有块摘要拼接（含 `=== Section N ===` 分隔符）
- **`{document_title}`**: 文档标题

**输出的沉默作用**:
`final_summary` 与调用 #6 的输出用途相同：存储为 `Document.content` 并生成 embedding，是大文档的最终"语义代表"，用于文档级 RAG 检索。

---

### 9. 通用文档摘要生成 — `generate_document_summary()`

**文件**: [`app/utils/document_converters.py`](../surfsense_backend/app/utils/document_converters.py#L123)

**触发时机**: 除 Docling 路径之外的文档摄入路径（Unstructured、LlamaCloud、Markdown 文件、各类 connector 文档同步），以及 Docling 摘要回退。

**调用方式**:
```python
summary_chain = SUMMARY_PROMPT_TEMPLATE | user_llm
summary_result = await summary_chain.ainvoke({"document": content_with_metadata})
```

**输入**:
- **Prompt**: `SUMMARY_PROMPT_TEMPLATE`（同调用 #6）
- **`{document}`**:
  ```xml
  <DOCUMENT>
    <DOCUMENT_METADATA>
      {document_metadata}  <!-- 文件名、ETL服务类型、文档类型等 -->
    </DOCUMENT_METADATA>
    <DOCUMENT_CONTENT>
      {optimized_content}  <!-- 按模型上下文窗口裁剪后的文档内容 -->
    </DOCUMENT_CONTENT>
  </DOCUMENT>
  ```
- 内容在传入前已通过 `optimize_content_for_context_window()` 进行 token 优化

**输出的沉默作用**:
- `summary_content` → 与元数据合并为 `enhanced_summary_content`（含 `# DOCUMENT METADATA` 和 `# DOCUMENT SUMMARY` 章节）
- 存储为 `Document.content`（数据库持久化，用于 RAG 文档级检索和用户"文档详情"视图展示）
- `summary_embedding` → 存储为 `Document.embedding`（向量索引，用于余弦相似度检索）

---

### 10. 音频语音转文字（STT）— `atranscription()`

**文件**: [`app/tasks/document_processors/file_processors.py`](../surfsense_backend/app/tasks/document_processors/file_processors.py#L572)

**触发时机**: 用户上传音频文件（`.mp3/.mp4/.mpeg/.mpga/.m4a/.wav/.webm`）时，若配置了外部 STT 服务（非 `local/` 前缀）则调用此路径。

> ⚠️ 注：本调用使用 LiteLLM 的 STT API（如 OpenAI Whisper），**不是对话型 LLM 调用**，但属于 LiteLLM 统一接口下的模型调用。

**调用方式**:
```python
transcription_response = await atranscription(
    model=app_config.STT_SERVICE,       # e.g. "whisper-1"
    file=audio_file,                    # 二进制音频数据
    api_key=app_config.STT_SERVICE_API_KEY,
    api_base=app_config.STT_SERVICE_API_BASE,  # 可选
)
transcribed_text = transcription_response.get("text", "")
```

**输入**:
- 音频文件二进制流（支持 mp3/mp4/wav/webm 等格式）
- STT 模型名称与 API 配置（来自全局 `app_config.STT_SERVICE` 等环境变量）

**输出的沉默作用**:
`transcribed_text` 添加标题前缀后，**作为 Markdown 文本**传入 `add_received_markdown_file_document()`，经过与普通文档相同的摘要生成（调用 #9）和 embedding 流程后存储到数据库。用户在文档列表中看到的是已转文字、已摘要化的文档，而非原始音频或转录原文。

---

### 11. 播客音频合成（TTS）— `aspeech()`

**文件**: [`app/agents/podcaster/nodes.py`](../surfsense_backend/app/agents/podcaster/nodes.py#L163)

**触发时机**: 播客器图的 `create_merged_podcast_audio` 节点，在文稿生成（调用 #5）之后执行，对每段对话**并发**调用。

> ⚠️ 注：本调用使用 LiteLLM 的 TTS API（如 OpenAI TTS），**不是对话型 LLM 调用**。

**调用方式**:
```python
response = await aspeech(
    model=app_config.TTS_SERVICE,         # e.g. "tts-1"
    voice=voice,                          # 根据 speaker_id 选择声音
    input=dialog,                         # 单段对话文本
    api_key=app_config.TTS_SERVICE_API_KEY,
    api_base=app_config.TTS_SERVICE_API_BASE,  # 可选
    max_retries=2,
    timeout=600,
)
```

**输入**:
- 单条播客对话文本（`dialog` 字段，来自 LLM 生成的 `PodcastTranscriptEntry`）
- 声音 ID（由 `speaker_id` 映射，不同主持人使用不同音色）
- TTS 模型配置（来自 `app_config.TTS_SERVICE` 等）

**输出的沉默作用**:
每段音频字节流（`response.content`）保存为临时 MP3/WAV 文件，全部段完成后用 FFmpeg 串联合并为最终播客 MP3 文件，存储路径记录到 `State.final_podcast_file_path`，最终由 Celery 任务将文件路径写入数据库，用户可通过 API 下载完整播客。各段中间音频文件在合并后自动删除，对用户不可见。

---

## LLM 角色分工汇总

SurfSense 为每个搜索空间（`SearchSpace`）配置三种不同角色的 LLM：

| LLM 角色 | 用途 | 特点要求 |
|---------|------|---------|
| **Fast LLM** | 查询重构（#2）、Q&A 问答（#3）、追问生成（#4）| 低延迟，快速响应；支持流式输出 |
| **Strategic LLM** | （代码注释中规划用于复杂规划任务，当前实际用于查询重构）| 复杂推理能力强 |
| **Long Context LLM** | 播客文稿（#5）、文档摘要（#6~#9） | 大上下文窗口（≥32K tokens）；用于处理长文档 |

此外还有两种非对话 LLM（通过 LiteLLM 统一接口调用）：
- **STT 模型**（如 Whisper）：音频转文字（#10）
- **TTS 模型**（如 OpenAI TTS / 本地 Kokoro）：文字转音频（#11）

---

## 输入/输出流向示意

```
用户提问
  │
  ▼
[#2 QueryService] 查询重构
  │ 输出：优化后的搜索字符串（沉默）
  ▼
多连接器检索（pgvector + 全文搜索）
  │ 输出：文档 chunks 列表
  ▼
[RerankerService] 文档重排序
  │ 输出：按相关性排序的文档列表（沉默）
  ▼
[#3 answer_question] Q&A 生成
  │ 输出：含引用的 Markdown 回答 → 流式返回用户
  ▼
[#4 generate_further_questions] 追问生成
  │ 输出：JSON 追问列表（沉默）→ 前端展示为推荐按钮

文档上传
  │
  ├─(音频文件)──[#10 atranscription]→ 转录文本（沉默）
  │                                       │
  └─(其他文件)                           │
       │                                  │
       ▼                                  ▼
  [#6~#9 generate_document_summary] 文档摘要（沉默）
       │ 输出：摘要文本 → Document.content（DB）
       │        摘要 embedding → Document.embedding（DB）
       ▼
  RAG 向量索引就绪

播客生成请求
  │
  ▼
[#5 create_podcast_transcript] 文稿生成（沉默）
  │ 输出：JSON 对话脚本
  ▼
[#11 aspeech] TTS 音频合成（沉默，并发执行）
  │ 输出：各段 MP3 片段（临时文件，沉默）
  ▼
FFmpeg 合并 → 最终播客 MP3 → 用户下载
```

---

*文档生成日期：2026-03-11*
