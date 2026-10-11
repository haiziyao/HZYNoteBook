# 一篇文档审核通过后，RAG、KAG 和普通搜索到底是怎么建出来的？

> 本文以项目当前源码为准，通过一篇具体文档，从上传、审核通过，一直走到 RAG 向量索引、KAG 知识图谱和普通 ES 搜索索引构建完成。

## 一、先给出结论：审核通过后产生了三份派生数据

假设用户上传了一篇文档：

```markdown
# Redis连接池超时排查指南

## 常见原因
连接池耗尽、连接超时配置过小、TCP连接被中间网络设备回收。

## Spring Boot配置
Lettuce连接池需要配置最大连接数、最大等待时间和连接超时。

## 排查流程
先检查Redis服务状态，再检查连接池指标，最后排查网络KeepAlive。
```

本文用以下虚构标识串联流程：

- `documentId = 1900000000000000001`
- `fileId = 1900000000000000101`
- `contentId = 66f0a1b2c3d4e5f678901234`

文档审核通过后，系统会尝试构建三份不同的派生数据：

| 分支 | 最终存储 | 数据粒度 | 主要用途 |
|---|---|---|---|
| RAG 索引 | Elasticsearch `kb_chunk` | 一个 Chunk 一条 | 语义检索、Chunk 级 BM25、RRF 融合和 RAG 问答 |
| KAG 图谱 | Neo4j | Document、Chunk、Entity 节点及关系 | 通过实体和关系扩展关联知识 |
| 普通搜索索引 | Elasticsearch `kb_document` | 一篇文档一条 | 文档级关键词搜索、过滤、高亮和列表展示 |

它们不是“同一份索引复制三遍”，而是面向三种查询模式的三种数据结构。

## 二、在建索引前，原始文档已经存在哪里？

### 1. 上传接口先解析文件

用户调用：

```http
POST /documents/upload/parse
Content-Type: multipart/form-data

file = Redis连接池超时排查指南.pdf
```

`DocumentController.uploadAndParseDocument()` 进入 `DocumentServiceImpl.uploadAndCreateDocument()`。这里的顺序是：

1. `kb-document` 用 `FileParserService` 把 PDF、Word、Markdown 等文件解析成文本。
2. 通过 OpenFeign 调用 `kb-file` 上传原始文件。
3. 根据文件名生成标题，构造 `DocumentDTO`。
4. 创建文档时强制设置 `status = 0`，因此刚上传只是草稿，不会立即启动 RAG/KAG/普通搜索三条索引链路。

这一点非常关键：**上传成功 ≠ 审核成功 ≠ 索引构建成功。**

### 2. RustFS 保存原始二进制文件

`kb-file` 的 `FileServiceImpl.uploadFile()` 会：

1. 验证文件。
2. 计算文件哈希，用于秒传判断。
3. 查询 `kb_file` 库的 `tb_file`。如果相同 `file_hash` 且状态有效，直接返回已有文件记录。
4. 没有命中时，调用 `RustFileStorage.upload()`，内部通过 S3 SDK 的 `putObject()` 把原始文件写入 RustFS。
5. 将文件 ID、原始名称、哈希、大小、类型、存储路径和访问级别等元数据写入 MySQL `tb_file`。

RustFS 里保存的是原始 PDF 或 Word 等文件，它不是 RAG 向量库。

### 3. MySQL 保存文档元数据

`DocumentServiceImpl.createDocument()` 先向 `kb_document` 库的 `kb_document` 表插入文档元数据，包括：

- 雪花 ID 生成的 `id`
- `title`、`summary`
- `category_id`、`team_id`、`tags`
- `author_id`、`author_name`
- `status`、`is_public`
- `file_path`、`file_size`、`file_extension`
- 浏览、点赞、评论计数
- `content_id`：后续用来指向 MongoDB 正文

MySQL 的这条记录是文档的“业务主记录”，审核状态和发布状态都以它为准。

### 4. MongoDB 保存可检索的解析后正文

`DocumentContentServiceImpl.saveContent()` 把解析后的 Markdown 正文写入 MongoDB 集合 `document_content`，一条数据大致为：

```json
{
  "_id": "66f0a1b2c3d4e5f678901234",
  "documentId": 1900000000000000001,
  "content": "# Redis连接池超时排查指南\n\n## 常见原因\n...",
  "contentLength": 168,
  "contentSummary": "...",
  "version": 1,
  "deleted": false
}
```

MongoDB 生成的 `_id` 会作为 `content_id` 回写到 MySQL `kb_document`。之后 RAG 和 KAG 消费者通过 `documentId` 请求文档详情时，`kb-document` 会先查 MySQL，再根据 `content_id` 从 MongoDB 取出正文，组装后返回。

因此，几种存储的职责是：

| 存储 | 保存什么 | 是否是审核后才保存 |
|---|---|---|
| RustFS | 原始 PDF、Word 等二进制对象 | 否，上传阶段已保存 |
| MySQL `tb_file` | 原始文件元数据 | 否 |
| MySQL `kb_document` | 文档业务元数据和 `content_id` | 否，审核通过时再更新状态和发布时间 |
| MongoDB `document_content` | 解析后的完整正文 | 否 |
| ES `kb_chunk` | 分块、分块元数据和向量 | 是，发布后异步构建 |
| ES `kb_document` | 文档级搜索副本 | 是，发布后异步同步 |
| Neo4j | 文档、Chunk、实体节点和图关系 | 是，发布后异步构建 |

## 三、审核通过是三条构建链路的起点

用户主动发布文档后，文档先进入待审核状态。审核员调用：

```http
POST /review/tasks/{taskId}/review
Content-Type: application/json

{
  "status": "approved",
  "comment": "内容准确，允许发布"
}
```

`DocumentReviewServiceImpl.approveReview()` 会：

1. 将审核记录更新为通过。
2. 将 MySQL 文档状态更新为 `PUBLISHED`，并写入发布时间。
3. 连续调用三个触发方法：

```java
triggerRagReindex(document.getId());
triggerKAGBuild(document.getId());
triggerSearchIndex(document, null);
```

三个方法都先用 `CompletableFuture.runAsync(..., asyncTaskExecutor)` 与审核主请求解耦。所以用户不必等待分块、Embedding和建图完成才收到“审核通过”的结果。

但这三条分支的后续方式不一样：

```text
                           ┌─ RAG：Feign → kb-ai → RabbitMQ → Chunking → Embedding → ES kb_chunk
审核通过 → 线程池 ─┼─ KAG：Feign → kb-ai → RabbitMQ → Chunking → LLM抽取 → Neo4j
                           └─ 普通搜索：Feign → kb-search → ES kb_document
```

## 四、RAG 索引是怎么构建的？

### 第一步：文档服务通过 Feign 通知 `kb-ai`

`triggerRagReindex(documentId)` 调用 `RagFeignClient`：

```http
POST /rag/reindex/1900000000000000001
```

`kb-ai` 的 `RagReindexController.reindexByDoc()` 收到请求后，调用 `ReindexServiceImpl.reindexByDocId()`。

注意：这一步仍然没有分块，也没有把全文通过 HTTP 传给 `kb-ai`，只传了 `documentId`。

### 第二步：`kb-ai` 把 RAG 任务发入 RabbitMQ

`ReindexServiceImpl` 生成 UUID 任务 ID，构造的消息类似：

```json
{
  "taskId": "rag-task-001",
  "type": "BY_DOC_IDS",
  "documentIds": [1900000000000000001]
}
```

RabbitMQ 拓扑为：

| 元素 | 名称 |
|---|---|
| 交换机类型 | TopicExchange |
| Exchange | `rag.reindex.exchange` |
| Routing Key | `rag.reindex.by_ids` |
| Queue | `rag.reindex.queue` |
| 监听方法 | `ReindexConsumer.handleReindex()` |
| ACK 方式 | `MANUAL` |

配置中还有两个路由键：

- `rag.reindex.all`：重建全部已发布文档。
- `rag.reindex.delete`：删除指定文档的 RAG Chunk 索引。

三个路由键都绑定到同一个 `rag.reindex.queue`。因此它虽然使用 TopicExchange，但这条业务并不是“广播给多个队列”，而是用不同 Routing Key 区分同一个工作队列里的任务类型。

为什么消息中只发 ID，不发整篇正文？

- 避免 RabbitMQ 承载大文本。
- 避免正文更新后，队列中还保留过期副本。
- 消费时重新获取当前最新文档。

### 第三步：RAG 消费者重新拉取正文

`ReindexConsumer` 监听 `rag.reindex.queue`。它收到任务后：

1. 向 Redis `rag:reindex:progress:{taskId}` 写入任务进度。
2. 通过 `DocumentFeignClient.getDocument(documentId)` 调用 `kb-document` 获取文档详情。
3. `kb-document` 先查 MySQL 元数据，再根据 `content_id` 从 MongoDB 取出完整正文。
4. 消费者拿到标题、正文、分类、作者、团队、发布状态等数据。

这里就回答了“正文从哪里来”：**RabbitMQ 只传 `documentId`，消费者再通过文档服务间接从 MongoDB 拉正文。**

### 第四步：先删除这篇文档的旧 Chunk

消费者调用：

```java
vectorIndexService.deleteByDocId(documentId);
```

它在 ES `kb_chunk` 中按 `document_id` 删除这篇文档的所有旧分块，然后再重新构建。这是“全量替换单篇文档 Chunk”，不是根据文本差异做增量更新。

### 第五步：此时才真正进行 Chunking

`ReindexConsumer` 调用 `ChunkingServiceImpl.chunk()`。项目默认配置是：

- 目标 Chunk 大小：512 token。
- Chunk Overlap：64 token。
- 源码没有使用真实 Tokenizer，而是估算 `1 token ≈ 2 个字符`，因此默认最大大约是 1024 字符，overlap 大约是 128 字符。

分块算法按以下层次处理：

1. 先按 Markdown 标题 `#`、`##`、`###` 等切分 Section。
2. Section 内按空行拆成段落。
3. 多个小段落可以合并，只要合并后不超过最大长度。
4. 单个段落太长时，按句号、问号、感叹号或换行切分。
5. 单句仍然过长时，才按字符数硬切。
6. 同一 Section 中的后续 Chunk 带上前一 Chunk 尾部的 overlap，减少切断处的语义丢失。
7. 每个 Chunk 前面加上它的章节标题。

上面这篇短文在默认配置下，可以形成类似三个 Chunk：

```text
Chunk 0
heading = 常见原因
content = 常见原因 + 连接池耗尽、连接超时配置过小……

Chunk 1
heading = Spring Boot配置
content = Spring Boot配置 + Lettuce连接池需要配置……

Chunk 2
heading = 排查流程
content = 排查流程 + 先检查Redis服务状态……
```

实际数量由文本长度、标题和空行分布决定。每个 Chunk 的 `chunkId` 由 UUID 生成。

### 第六步：Chunking 完成后才做 Embedding

消费者从每个 Chunk 取出 `content`：

```java
List<String> texts = chunks.stream()
        .map(DocumentChunk::getContent)
        .toList();

List<float[]> embeddings = embeddingService.embedBatch(texts);
```

`EmbeddingServiceImpl` 通过 LangChain4j 的 `EmbeddingModel` 调用 OpenAI 兼容的 Embedding API，将每个 Chunk 转换为默认 1024 维向量。

时序必须说准：

```text
拉取完整正文 → Chunking → 获得多个 Chunk → 对每个 Chunk 做 Embedding → 写 ES
```

Embedding 缓存的 Key 是：

```text
rag:emb:{MD5(chunk文本)}
```

相同 Chunk 再次建索引时可以复用 Redis 中的向量，减少模型调用和成本。但需要注意，批量 `embedBatch()` 的主路径直接调用 `embedAll()`；这个缓存主要在单条 `embed()` 及批量失败降级到单条时生效，不能笼统地说“每次批量 Embedding 都先命中 Redis”。

### 第七步：将 Chunk 和向量批量写入 ES `kb_chunk`

`VectorIndexServiceImpl.indexChunks()` 通过 ES Bulk API 写入。每个 Chunk 一条 ES 文档，包含：

```json
{
  "chunk_id": "6d36...",
  "document_id": 1900000000000000001,
  "document_title": "Redis连接池超时排查指南",
  "content": "排查流程\n\n先检查Redis服务状态……",
  "heading": "排查流程",
  "chunk_index": 2,
  "total_chunks": 3,
  "category_id": 1001,
  "author_id": 2001,
  "team_id": 3001,
  "doc_status": 1,
  "publish_time": "2026-09-06T10:00:00",
  "indexed_at": "2026-09-06T10:00:03",
  "embedding": [0.021, -0.037, 0.014]
}
```

示例向量为缩写，实际维度取决于配置，当前默认为 1024。

到这里，RAG 索引构建完成。随后消费者更新 Redis 进度并手动 ACK。

### RAG 索引建好后怎么用？

用户询问“Lettuce 连接池超时怎么排查”时：

1. 对用户问题做 Query Embedding。
2. 在 `kb_chunk` 中并行执行 BM25 关键词检索和向量余弦相似度检索。
3. 用 RRF 合并两份排名。
4. 可选用 LLM 对候选 Chunk 重排序。
5. 取 Top-K Chunk 构造 Prompt，再让 LLM 生成答案和引用。

当前源码所谓的“kNN”是 `match_all + script_score + cosineSimilarity` 全库计分，不是 HNSW 近似最近邻检索。面试时不要自动把它包装成 HNSW。

## 五、KAG 什么时候解耦，发到哪个队列？

### 第一步：文档服务通知 `kb-ai`

RAG 分支启动的同时，`triggerKAGBuild(documentId)` 在另一个线程池任务中调用：

```http
POST /kag/build/1900000000000000001
```

`KAGReindexController` 将请求交给 `GraphBuildServiceImpl.publishBuildTask()`。

### 第二步：将 KAG 建图任务发入 RabbitMQ

消息示例：

```json
{
  "taskId": "kag-task-001",
  "type": "BUILD_BY_DOC_IDS",
  "documentIds": [1900000000000000001]
}
```

RabbitMQ 拓扑为：

| 元素 | 名称 |
|---|---|
| 交换机类型 | TopicExchange |
| Exchange | `kag.graph.exchange` |
| Routing Key | `kag.graph.build.by_ids` |
| Queue | `kag.graph.build.queue` |
| 监听方法 | `KAGReindexConsumer.handleBuild()` |
| ACK 方式 | `MANUAL` |

另外两个路由键是：

- `kag.graph.build.all`：构建所有已发布文档。
- `kag.graph.delete`：删除指定文档的图数据。

三个路由键同样绑定到一个 `kag.graph.build.queue`。

项目还做了一个局部降级：如果 `rabbitTemplate.convertAndSend()` 当场抛异常，`publishBuildTask()` 会在当前 `kb-ai` 请求中改为同步 `buildForDocument()`。这只覆盖“发送时抛异常”，不能等价为完整的 MQ 可靠性保证。

### 第三步：KAG 消费者重新拉正文，并再做一次 Chunking

`KAGReindexConsumer` 消费后调用 `GraphBuildServiceImpl.buildForDocument()`。它与 RAG 类似：

1. 通过 Feign 调用 `kb-document` 获取文档详情。
2. 间接从 MongoDB 获取完整正文。
3. 如果配置了 `clear-before-build = true`，先删除这篇文档已有的图谱节点和关系。
4. 创建 `KnowledgeDocument` 节点。
5. 调用同一个 `ChunkingServiceImpl` 重新对正文分块。

这意味着当前项目**不是审核通过后统一分块一次，然后把同一批 Chunk 分发给 RAG 和 KAG**。实际上是 RAG 消费者分一次，KAG 消费者又分一次。

因为 `chunkId` 是每次调用分块方法时随机生成的 UUID，所以同一段语义文本在 ES 和 Neo4j 中通常会有不同的 `chunkId`。

### 第四步：在 Neo4j 中创建文档和 Chunk 节点

对于示例文档，先创建：

```text
(:KnowledgeDocument {
  docId: 1900000000000000001,
  title: "Redis连接池超时排查指南",
  summary: "...",
  categoryId: 1001,
  authorId: 2001,
  status: 1
})
```

然后为每个分块创建 `DocumentChunk` 节点。Chunk 节点不只保存 ID，还保存完整的分块文本：

```text
(:DocumentChunk {
  chunkId: "kag-chunk-uuid-001",
  docId: 1900000000000000001,
  content: "常见原因\n\n连接池耗尽、连接超时配置过小……",
  heading: "常见原因",
  chunkIndex: 0,
  totalChunks: 3
})
```

并建立：

```text
(KnowledgeDocument)-[:HAS_CHUNK]->(DocumentChunk)
```

### 第五步：对每个 Chunk 调用 LLM 抽取实体和关系

KAG 不对 Chunk 做 Embedding。它调用 `ExtractionService.extract()`，让 LLM 从每个 Chunk 中抽取结构化知识。

对于“Spring Boot 配置”分块，LLM 可能返回类似：

```json
{
  "entities": [
    {"name": "Spring Boot", "type": "FRAMEWORK"},
    {"name": "Lettuce", "type": "CLIENT"},
    {"name": "Redis", "type": "DATABASE"}
  ],
  "relations": [
    {"source": "Spring Boot", "relation": "USES", "target": "Lettuce", "weight": 0.95},
    {"source": "Lettuce", "relation": "CONNECTS_TO", "target": "Redis", "weight": 0.98}
  ]
}
```

这只是说明数据形状的示例，真实抽取结果由模型和 Prompt 决定。

### 第六步：实体归一化后写入 Neo4j

`EntityNormalizer` 会做三层处理：

1. 合并当前批次中的同名实体。
2. 检查 Neo4j 中已有同名实体或匹配别名的实体。
3. 按编辑距离合并高度相似的名称。

然后创建或合并 `KnowledgeEntity` 节点，并建立两类关系：

```text
(DocumentChunk)-[:MENTIONS]->(KnowledgeEntity)
(KnowledgeEntity)-[:RELATED_TO {relation: "USES", weight: 0.95}]->(KnowledgeEntity)
```

当前源码对实体之间的 Neo4j Relationship Type 统一使用 `RELATED_TO`。`USES`、`CONNECTS_TO` 等具体语义保存在关系的 `relation` 属性中，不是真正的多种 Neo4j Relationship Type。

建图完成后，`kb-ai` 还会通知 `kb-graph` 清除图谱 Redis 缓存，使前端下次查询看到新图。最后 KAG 消费者手动 ACK。

## 六、普通搜索索引是怎么做的？

普通搜索分支最容易被误解。它的真实链路是：

```text
DocumentReviewServiceImpl.triggerSearchIndex()
→ CompletableFuture
→ SearchIndexFeignClient.indexDocument(docData)
→ kb-search POST /index/document
→ SearchServiceImpl.indexDocumentData()
→ ElasticsearchOperations.save()
→ ES kb_document
```

这条链路：

- **不经过 RabbitMQ。**
- **不分 Chunk。**
- **不做 Embedding。**
- 一篇文档在 `kb_document` 中只有一条 ES 记录。

设计上准备保存的字段包括：

- 文档 ID、标题、摘要和部分正文
- 分类和标签
- 作者和团队
- 文档状态和公开状态
- 浏览、点赞、评论数
- 发布、创建和更新时间

`kb-search` 普通搜索时，在 `kb_document` 上使用 `multi_match`：

```text
title^2 + summary^1.5 + content
```

同时过滤 `docStatus = 1`，并为标题、摘要和正文返回高亮片段。

### 当前审核通过路径存在一个实际缺陷

`approveReview()` 调用的是：

```java
triggerSearchIndex(document, null);
```

此时 `document` 是 MySQL 实体，而它的 `content` 字段被 `@TableField(exist = false)` 忽略，完整正文实际在 MongoDB。`buildSearchIndexData()` 又没有在此处根据 `content_id` 回查 MongoDB。

所以，通过“审核通过”触发的这次 `kb_document` 写入，很可能只有标题、摘要和元数据，没有正文。而创建已发布文档或更新已发布文档的其他路径如果显式传入了正文，才会截取前 1000 个字符写入普通搜索索引。

这会导致普通搜索对“只出现在正文、没出现在标题或摘要”的词召回不全。合理修复方向是在构造普通搜索数据时根据 `content_id` 获取 MongoDB 正文，或统一交给 `kb-search` 根据 `documentId` 回拉详情。本文只解释问题，没有修改源码。

## 七、`kb_document` 和 `kb_chunk` 都在 ES，为什么还要分开？

| 对比项 | `kb_document` | `kb_chunk` |
|---|---|---|
| 建立者 | `kb-search` | `kb-ai` |
| 粒度 | 文档级 | Chunk 级 |
| 是否有向量 | 否 | 有 |
| 是否用 RabbitMQ | 否 | 是 |
| 是否 Chunking | 否 | 是 |
| 关键词检索 | 标题、摘要、正文的文档级 BM25 | Chunk 正文和标题的 Chunk 级 BM25 |
| 用途 | 搜索页列表、过滤、高亮 | RAG 召回、语义检索、混合检索 |

一篇上万字的文档如果只生成一个向量，主题会被过度平均，很难精准命中局部细节，因此 RAG 需要 `kb_chunk`。普通搜索页需要按文档展示标题、作者、分类、计数等信息，因此又需要 `kb_document`。

项目的“混合智能搜索”是调用 `kb-ai` 查 `kb_chunk`，获得 Chunk 级混合检索结果，然后在 `kb-search` 按 `documentId` 聚合，并用 `kb_document` 补充分类、作者、浏览量等文档元数据。

## 八、三条链路的完整时序

```text
1. 用户上传 Redis连接池超时排查指南.pdf
   ├─ kb-document 解析出 Markdown 正文
   ├─ kb-file 计算哈希，检查秒传
   ├─ RustFS 保存原始文件
   ├─ MySQL tb_file 保存文件元数据
   ├─ MySQL kb_document 保存文档元数据，状态=草稿
   └─ MongoDB document_content 保存完整解析正文

2. 用户主动发布，文档进入待审核

3. 审核员审核通过
   ├─ MySQL审核记录=通过
   └─ MySQL文档状态=PUBLISHED

4. 线程池并行触发三条派生数据链路
   │
   ├─ RAG
   │  ├─ Feign通知 kb-ai
   │  ├─ rag.reindex.exchange
   │  ├─ rag.reindex.by_ids
   │  ├─ rag.reindex.queue
   │  ├─ 消费者按ID拉正文
   │  ├─ 删旧Chunk
   │  ├─ Chunking
   │  ├─ Embedding
   │  └─ Bulk写入 ES kb_chunk
   │
   ├─ KAG
   │  ├─ Feign通知 kb-ai
   │  ├─ kag.graph.exchange
   │  ├─ kag.graph.build.by_ids
   │  ├─ kag.graph.build.queue
   │  ├─ 消费者按ID拉正文
   │  ├─ KAG自己再做一次Chunking
   │  ├─ 创建Document和Chunk节点
   │  ├─ LLM抽取实体和关系
   │  ├─ 实体归一化
   │  ├─ 写入Neo4j节点和关系
   │  └─ 清除图谱Redis缓存
   │
   └─ 普通搜索
      ├─ Feign直接调用 kb-search
      ├─ 不走RabbitMQ
      ├─ 不分块，不Embedding
      └─ 写入 ES kb_document
```

## 九、这套设计的一致性和失败语义

### 1. 审核通过是业务事实，三份索引是派生数据

RAG、KAG 和普通 ES 分支都在异步任务中捕获异常，失败时只记录日志，不回滚已审核通过的文档。

这体现的是：

```text
业务主数据优先提交 + 派生索引最终一致
```

好处是审核员不会因为 Embedding 服务或 Neo4j 短暂故障而长时间卡住。代价是文档可能已经显示为已发布，但暂时搜不到、问不到或看不到图谱。

### 2. RabbitMQ 的可靠性并不完整

RAG 和 KAG 队列都设置了 `x-dead-letter-exchange = 原交换机名 + .dlx`，但当前源码没有真正声明对应的死信交换机和死信队列。消费失败时使用：

```java
basicNack(deliveryTag, false, false)
```

即不重新入队。如果运行环境没有额外创建 DLX/DLQ，失败消息不会有可用的死信队列承接。

当前也没有看到完整的统一补偿调度器来根据发布文档状态对三份派生数据做对账和重建。所以面试时可以说“项目采用最终一致的思路”，但不能说“已经实现了完整的最终一致保障”。

### 3. 三个异步任务可能早于审核事务提交

`approveReview()` 有 Spring `@Transactional`，但它在事务方法内就启动了 `CompletableFuture`。异步线程有可能在 MySQL 事务真正提交前请求文档详情，从而读到旧状态。

更稳妥的做法是在事务提交后发布领域事件，例如使用 `@TransactionalEventListener(phase = AFTER_COMMIT)`，再由事件处理器投递 RAG/KAG/搜索构建任务。

### 4. RAG 和 KAG 重复拉取、重复分块

当前实现简单，两条链路可以独立失败和重建，但也带来：

- 重复调用文档服务和 MongoDB。
- 重复计算 Chunk。
- 两次分块之间如果正文被更新，边界可能不一致。
- ES 和 Neo4j 里的 Chunk UUID 无法直接对齐。

更高级的改造方案可以是：审核事务提交后发布一个文档知识构建事件，一个独立分块阶段生成稳定 Chunk ID 和文本版本，再将 Chunk 构建完成事件分发给 RAG 向量化、KAG 实体抽取和其他下游。这是改进方向，不是当前源码已经实现的逻辑。

## 十、面试时可以如何在两分钟内说清楚？

> 文档上传时，系统先在文档服务解析文本，再把原始文件交给文件服务存入 RustFS，文件元数据存 MySQL `tb_file`。文档业务元数据存 MySQL `kb_document`，解析后的完整正文存 MongoDB，MySQL 通过 `content_id` 关联 MongoDB。刚上传是草稿，只有用户发布并审核通过后，才会启动三条派生数据链路。
>
> RAG 分支通过 Feign 调用 `kb-ai`，再将只包含任务 ID 和文档 ID 的消息发往 TopicExchange `rag.reindex.exchange`，使用 `rag.reindex.by_ids` 路由到 `rag.reindex.queue`。消费者按 ID 从文档服务拉取 MongoDB 正文，先删旧 Chunk，再按 Markdown 标题、段落和句子分块，添加 overlap，然后对每个 Chunk 做 Embedding，把分块文本、元数据和向量 Bulk 写入 ES `kb_chunk`。
>
> KAG 分支类似，消息经 `kag.graph.exchange` 和 `kag.graph.build.by_ids` 进入 `kag.graph.build.queue`。KAG 消费者会自己再拉一次正文并重新分块，在 Neo4j 创建 Document 和 Chunk 节点，建立 `HAS_CHUNK`，再逐 Chunk 调 LLM 抽取实体和关系，经实体归一化后写入 `KnowledgeEntity`、`MENTIONS` 和 `RELATED_TO`。
>
> 普通搜索不走 RabbitMQ，文档服务在线程池中通过 Feign 直接调用 `kb-search`，按一文档一条的粒度写入 ES `kb_document`，用于 BM25、过滤、高亮和文档列表展示。三条链路都不回滚已通过的审核状态，所以它们是基于派生数据最终一致的设计，但当前项目在死信队列、补偿对账和事务提交后发事件方面还有完善空间。

## 十一、关键源码定位

| 职责 | 关键类 |
|---|---|
| 上传、解析、MySQL/MongoDB 文档存储 | `kb-document` 的 `DocumentServiceImpl` |
| MongoDB 正文存取 | `DocumentContentServiceImpl` |
| 原始文件元数据和秒传 | `kb-file` 的 `FileServiceImpl` |
| RustFS S3 对象写入 | `RustFileStorage` |
| 审核通过及三条分支触发 | `DocumentReviewServiceImpl` |
| RAG RabbitMQ 拓扑 | `RabbitConfig` |
| RAG 任务生产和 Redis 进度 | `ReindexServiceImpl` |
| RAG 队列消费与索引编排 | `ReindexConsumer` |
| Chunking | `ChunkingServiceImpl` |
| Embedding | `EmbeddingServiceImpl` |
| ES `kb_chunk` 写入和混合检索 | `VectorIndexServiceImpl` |
| KAG RabbitMQ 拓扑 | `KAGRabbitConfig` |
| KAG 队列消费 | `KAGReindexConsumer` |
| KAG 拉取、分块、抽取和 Neo4j 写入编排 | `GraphBuildServiceImpl` |
| 实体去重和归一化 | `EntityNormalizer` |
| 普通搜索索引写入与查询 | `kb-search` 的 `SearchServiceImpl` |
| ES `kb_document` 字段映射 | `DocumentIndex` |

---

一句话总结：**MySQL 和 MongoDB 保存真实业务数据，RustFS 保存原文件；审核通过后，RAG 把正文变成可做语义检索的 ES Chunk 向量索引，KAG 把正文变成 Neo4j 节点和关系，普通搜索则把文档变成可做 BM25、过滤和高亮的文档级 ES 副本。**
