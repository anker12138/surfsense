# SurfSense 文件支持与检索机制分析

更新时间：2026-03-11

## 1. 项目当前支持哪些类型的文件/内容

从实现看，系统有三层“支持”概念：

1. 前端可上传类型（按 ETL 服务动态切换）
2. 后端处理分支（文本/音频/其他文档）
3. 真正能成功解析的范围（取决于当前 ETL 引擎能力）

- 提问：ETL是什么？   ETL 是 Extract（提取）、Transform（转换）、Load（加载）的缩写，指的是数据处理流程，用于将数据从源系统提取出来，经过转换处理后加载到目标系统中。

### 1.1 前端上传层（UI 限制）

前端上传组件会根据 NEXT_PUBLIC_ETL_SERVICE 决定 accept 列表。

- 文件：surfsense_web/components/sources/DocumentUploadTab.tsx:31
- LlamaCloud 类型分支：surfsense_web/components/sources/DocumentUploadTab.tsx:43
- Docling 类型分支：surfsense_web/components/sources/DocumentUploadTab.tsx:76
- 默认（Unstructured）类型分支：同文件 else 分支
  - 这三种类型是分别是什么 怎么选择
    - Unstructured：开源的文档解析库，支持多种格式，适合本地部署和小规模使用
    - LlamaCloud：商业文档解析服务，号称支持更复杂的文档结构和更高的解析质量，适合对解析效果有较高要求的用户
    - Docling：另一种商业文档解析服务，特点和定位与 LlamaCloud 类似，用户可以根据价格、性能和解析效果等因素选择
- 单文件上限 50MB：surfsense_web/components/sources/DocumentUploadTab.tsx:133

可见前端放开的格式包含：

- 文档类：pdf/doc/docx/ppt/pptx/xls/xlsx/rtf/xml/epub/csv/tsv/html 等
- 文本类：md/markdown/txt
- 图片类：jpg/jpeg/png/gif/bmp/tiff/webp/svg 等
- 音频类：mp3/mpeg/mpga/mp4/m4a/wav/webm

### 1.2 后端处理层（实际任务分流）

后端文件上传入口不会在路由层做严格扩展名白名单校验，收到文件后交给 Celery 异步处理：

- 文件上传入口：surfsense_backend/app/routes/documents_routes.py:105
- 参数为 UploadFile 列表：surfsense_backend/app/routes/documents_routes.py:107
- 异步任务触发：surfsense_backend/app/routes/documents_routes.py:150

后台任务分流逻辑：

- Markdown/文本直读：.md/.markdown/.txt
  - surfsense_backend/app/tasks/document_processors/file_processors.py:460
- 音频转写：.mp3/.mp4/.mpeg/.mpga/.m4a/.wav/.webm
  - surfsense_backend/app/tasks/document_processors/file_processors.py:513
- 其他文档走 ETL（UNSTRUCTURED / LLAMACLOUD / DOCLING）
  - surfsense_backend/app/tasks/document_processors/file_processors.py:691
  - surfsense_backend/app/tasks/document_processors/file_processors.py:784
  - surfsense_backend/app/tasks/document_processors/file_processors.py:924

### 1.3 页面估算层（间接反映支持范围）
PageLimitService 不是用来解析文件的，而是用来做“用户页数额度控制”的。

它的核心职责是：

1. 在真正调用 ETL 服务之前，先估算文件大概有多少页
2. 判断当前用户剩余额度是否足够
3. 如果不够，直接拒绝处理，避免触发昂贵的 ETL 解析
4. 如果处理成功，再把 pages_used 更新到用户账户上

代码位置：

- 页数额度服务：surfsense_backend/app/services/page_limit_service.py
- 调用入口：surfsense_backend/app/tasks/document_processors/file_processors.py

不同 ETL 的估算方式不同：

- PDF：优先直接读取真实页数
- Unstructured：优先根据 element metadata 里的 page_number 估算
- LlamaCloud：根据返回 markdown 文档数量或内容长度估算
- Docling：根据内容长度估算
- 可本地部署

可以把它理解成“进加工厂前的额度闸机”，不是加工厂本身。

## 2. 是否有数据库存储用户保存的文件

有，但需要把“存文件”拆开理解。

这个项目真正存进 PostgreSQL 的，不是普通上传文件的原始二进制本体，而是“处理后的文本、摘要、切块、元数据和向量”。

核心结构是 PostgreSQL（含 pgvector）中的 documents + chunks 两层模型。

补充说明：

- 在自托管场景下，PostgreSQL 通常部署在用户自己的环境里
- 从代码实现上看，上传文件会先写到临时文件，再交给 Celery 异步处理
- 处理完成后临时文件会删除
- 没有看到专门用于保存原始上传文件路径或二进制内容的 FILE 文档字段

文件上传入口：

- surfsense_backend/app/routes/documents_routes.py

文档处理主流程：

- surfsense_backend/app/tasks/document_processors/file_processors.py

数据库模型：

- 文档类型枚举：surfsense_backend/app/db.py
- 文档主表模型：surfsense_backend/app/db.py
- 分块表模型：surfsense_backend/app/db.py

- 文档类型枚举：surfsense_backend/app/db.py:37
- 文档主表模型：surfsense_backend/app/db.py:334
- 分块表模型：surfsense_backend/app/db.py:366

Document 关键字段：

- document_type（来源类型）
  - surfsense_backend/app/db.py:338
- document_metadata（来源特定元数据 JSON）
  - surfsense_backend/app/db.py:339
- content（文档主内容，很多场景是摘要）
  - surfsense_backend/app/db.py:341
- embedding（向量）
  - surfsense_backend/app/db.py:344
- search_space_id（按知识空间隔离）
  - surfsense_backend/app/db.py:357
- chunks 关系
  - surfsense_backend/app/db.py:361

Chunk 关键字段：

- content（可检索分块文本）
  - surfsense_backend/app/db.py:369
- embedding（分块向量）
  - surfsense_backend/app/db.py:370

### 2.1 一个 md 文件上传后会发生什么

md、markdown、txt 会走“直接读取文本”分支：

- 读取原始文本内容
- 生成 unique_identifier_hash，用于判断这是不是同一个来源的文档
- 生成 content_hash，用于判断内容有没有变化
- 调用长上下文 LLM 生成摘要
- 对摘要生成文档级 embedding
- 对原文切块，每个 chunk 生成 embedding
- 转成 blocknote_document，供编辑器使用
- 最终写入一条 documents 记录和多条 chunks 记录

相关代码：

- 文本分流：surfsense_backend/app/tasks/document_processors/file_processors.py
- Markdown 处理器：surfsense_backend/app/tasks/document_processors/markdown_processor.py
- 摘要、切块、向量：surfsense_backend/app/utils/document_converters.py

最终结果可以概括为：

- documents.content：存摘要或结构化总结
- documents.embedding：存摘要的向量
- chunks.content：存正文切块
- chunks.embedding：存每个正文块的向量
- blocknote_document：存编辑器用的 JSON 表示

### 2.2 一个 mp4 文件上传后会发生什么

当前实现里，mp4 并不是按“视频多模态分析”处理，而是被归到“音频转写”分支：

- 先对 mp4 做语音转写
- 生成一段文本，如 “Transcription of xxx.mp4”
- 然后把这段转写文本当作 Markdown 文本继续处理
- 后续步骤和 md 文件一致：摘要、切块、向量化、入库

相关代码：

- 音频/媒体分流：surfsense_backend/app/tasks/document_processors/file_processors.py
- 本地转写服务：surfsense_backend/app/services/stt_service.py

这意味着当前项目对 mp4 的主要存储结果是：

- 转写后的文本
- 转写文本的摘要 
- 转写文本的 chunks
- 文档级和 chunk 级向量

不是：

- 原始 mp4 二进制
- 视频帧特征
- 独立的视频模态向量

### 2.3 PostgreSQL 里到底存不存原始文件

从当前 FILE 上传代码看，结论是：

- 不存原始文件二进制到 documents/chunks
- 也没有专门的 FILE 原始文件路径字段
- 主要只存处理后的文本、摘要、metadata 和 embedding

需要注意一个容易混淆的点：

- Podcast 表里有 file_location 字段
- 但那是播客音频文件的位置，不是普通上传文件的存储字段

### 2.4 不同模态的数据会分别存储吗

从当前实现看，主策略是“统一归一化为文本后再存储和检索”。

具体来说：

- md、txt：直接作为文本
- pdf、docx、ppt、图片等：先通过 **ETL** 转成 Markdown 或文本，再按文本存储
- mp3、mp4、wav 等：先转写成文本，再按文本存储
- YouTube：主要围绕转录文本和元数据进行检索，不是直接检索视频本体

所以当前不是“每种模态一套独立向量库”，而是：

- 用 document_type 区分来源类型
- 用统一的 documents/chunks + embedding 结构承载检索内容

## 3. 不同类型内容如何组织存储到数据库

统一模式：

1. 原始内容先归一化为文本/Markdown
2. 生成摘要与 embedding（写入 documents.content / documents.embedding）
3. 切块并生成 chunk embedding（写入 chunks）
4. metadata 按来源写入 JSON
5. 用 unique_identifier_hash + content_hash 做去重/变更检测

### 3.1 文件上传（FILE）

FILE 文档写入时会记录 document_type=FILE，并写入 metadata、blocknote、chunks。
- 生成摘要：surfsense_backend/app/tasks/document_processors/file_processors.py:96
- 创建 chunks：surfsense_backend/app/tasks/document_processors/file_processors.py:101
- 写入 document_type=FILE：surfsense_backend/app/tasks/document_processors/file_processors.py:136

说明：

- documents.content 通常存摘要/结构化总结
- chunks.content 保存更细粒度正文块，检索主要依赖 chunks


### 3.2 去重和更新是怎么做的

项目不是每次上传都盲目新建文档，而是结合两个哈希字段做判断：

- unique_identifier_hash：表示“这个来源对象是谁”
- content_hash：表示“当前内容是不是变了”

这也是为什么扩展网页、YouTube、文件上传都可以支持“重复导入但只在内容变化时更新”。

## 4. Agent 会自动检索本地数据库并提取相关内容吗

结论：会，但检索哪些来源、是否把某些文档强制加入上下文，取决于请求里的 selected_connectors 和 document_ids_to_add_in_context。

聊天入口：

- surfsense_backend/app/routes/chats_routes.py
- surfsense_backend/app/tasks/stream_connector_search_results.py

### 4.1 selected_connectors 是什么

selected_connectors 是前端显式传进来的“要搜索哪些来源”的列表，不是 Agent 自己临时分析后自动决定的。

当前 researcher 节点实际支持的 connector 类型包括：

- FILE：用户上传文件
- EXTENSION：浏览器扩展采集的网页内容
- CRAWLED_URL：爬虫抓取网页
- YOUTUBE_VIDEO：YouTube 转录内容
- SLACK_CONNECTOR
- NOTION_CONNECTOR
- GITHUB_CONNECTOR
- LINEAR_CONNECTOR
- DISCORD_CONNECTOR
- JIRA_CONNECTOR
- CONFLUENCE_CONNECTOR
- CLICKUP_CONNECTOR
- GOOGLE_CALENDAR_CONNECTOR
- GOOGLE_GMAIL_CONNECTOR
- AIRTABLE_CONNECTOR
- LUMA_CONNECTOR
- ELASTICSEARCH_CONNECTOR
- TAVILY_API
- SEARXNG_API
- LINKUP_API
- BAIDU_SEARCH_API

## 5. 联网搜索返回结果如何处理

### 5.1 联网搜索结果会执行哪些处理

当前实现并不是“把联网结果先入库再检索”，而是：

1. 直接调用外部搜索 API
2. 读取返回的 title、content 或 snippet、url、score 等字段
3. 把结果包装成 source 对象和 document 字典
4. 直接参与本次回答的上下文构建

### 5.2 联网搜索结果是多模态还是纯文本

从当前上下文构建逻辑看，进入问答流程的是“文本化结果”，其中的视频 图片 会做相应的文本化处理吗：

### 5.3 联网搜索结果会不会存数据库

当前代码路径下，联网搜索结果不会写入 PostgreSQL documents/chunks 表。

它们是“即取即用”的上下文材料，只存在于当前回答流程里。
它们只是被包装成临时 document 对象，直接供 LLM 回答使用。



## 6. 什么时候基于 documents 表进行检索，什么时候基于 chunks 表进行检索，检索后怎样打分排序，最后怎样添加到上下文

### 6.1 什么时候查 chunks，什么时候查 documents

由请求参数 search_mode  决定。

默认值是 CHUNKS。search_mode怎么定义？智能体会自己修改吗 不能

代码位置：

- 参数校验：surfsense_backend/app/utils/validators.py
- Chunk 检索器：surfsense_backend/app/retriver/chunks_hybrid_search.py
- Document 检索器：surfsense_backend/app/retriver/documents_hybrid_search.py

两种模式的差别是：

1. CHUNKS
  - 直接在 chunks 表上搜索
  - 适合更细粒度命中
  - 默认就是这个模式

2. DOCUMENTS
  - 先在 documents 表上搜索
  - 命中文档后，再把该文档的 chunks 拉出来返回
  - 所以它不是简单返回摘要，而是“先按文档筛，再展开回 chunk”



### 6.2 检索结果怎样打分排序

本地数据库检索走的是“混合搜索”：

1. 向量检索
  - 把 query 转成 embedding
  - 用 pgvector 比较语义距离

2. 全文检索
  - 用 PostgreSQL 的 to_tsvector 和 plainto_tsquery 做关键词搜索
  - 用 ts_rank_cd 做文本相关性排序

3. 倒数排名融合 RRF
  - 把向量检索和全文检索的结果融合
  - 得到最终 score

4. 可选 rerank
  - 如果开启 reranker，再按用户问题对候选上下文重新排序一次
  - 没开就保留原始顺序


### 6.3 检索结果最后怎样进入上下文
被整理后的 document 列表。

处理顺序大致是：

1. 先收集 user-selected documents
2. 再收集 connector 检索到的 documents
3. 合并为 all_documents
4. 可选经过 reranker 二次重排
5. 根据 token 限制裁剪文档数量和长度
6. 通过 format_documents_section 渲染成提示词里的 documents_text
7. 与用户问题一起送给 LLM

对应代码：

- researcher 主流程：surfsense_backend/app/agents/researcher/nodes.py
- qna rerank 与回答：surfsense_backend/app/agents/researcher/qna_agent/nodes.py



