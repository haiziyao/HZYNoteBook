# 语雀批次 04：第 19～20 章（AI 助手与 RAG）

> 学习范围：语雀第 19 章“增加 AI 助手功能”、第 20 章“增加 RAG 知识库功能”。
> 本笔记以文章设计为主，并用当前源码核对实现差异。示例中的地址、密钥均已省略。

## 一句话主线

第 19 章先提供普通 AI 对话能力；第 20 章在此基础上，把企业文档切块、向量化并写入 Elasticsearch，查询时使用 BM25 + 向量检索，再经过 RRF 融合和可选重排序，把相关片段注入 Prompt，最终生成带引用的回答。索引构建通过 RabbitMQ 异步执行，文档服务通过 Feign 触发，整体采用最终一致性。

## 1. 模块与依赖关系

```text
客户端
  -> gateway (/api/ai/**)
  -> kb-ai（8086）
       ├─ MySQL：会话、消息、反馈、任务记录
       ├─ Redis：Embedding 缓存、重建进度
       ├─ Elasticsearch：kb_chunk 分块与向量索引
       ├─ RabbitMQ：异步重建索引
       ├─ OpenFeign -> kb-document：获取文档正文/元数据
       └─ LangChain4j -> Qwen/DeepSeek 等 OpenAI 兼容模型
```

Nacos 负责服务注册与发现，Feign 负责跨微服务调用；它们是调用链基础设施，不直接产生 RAG 结果。

## 2. 第 19 章：AI 助手（RAG 的入口）

### 主要接口

| 方法 | 路径 | 作用 |
|---|---|---|
| GET | `/chat/models` | 查询当前可用模型 |
| POST | `/chat` | 普通同步对话，返回完整答案 |
| POST | `/chat/stream` | 普通流式对话，以 SSE 推送答案片段 |
| POST | `/conversation` | 创建会话 |
| GET | `/conversation/list` | 查询当前用户会话列表 |
| GET | `/conversation/{id}` | 查询会话详情和消息 |
| DELETE | `/conversation/{id}` | 删除会话（按当前实现策略处理关联消息） |
| POST | `/feedback` | 对 AI 回复点赞/点踩（控制器：`AiFeedbackController`） |

同步对话示例：

```json
// POST /chat
{
  "conversationId": 10001,
  "message": "什么是向量检索？",
  "model": "qwen-plus"
}
```

```json
{
  "conversationId": 10001,
  "messageId": 20001,
  "content": "向量检索是……",
  "tokens": 86,
  "title": "向量检索",
  "citations": [],
  "fromKnowledgeBase": false
}
```

第 19 章面试表达：`kb-ai` 把模型调用、会话持久化和 SSE 封装为独立微服务；普通对话不依赖知识库，RAG 对话是后续增强入口。

## 3. 第 20 章：RAG 主链路

### 3.1 文档入库与分块

涉及类：

- `ChunkingService` / `ChunkingServiceImpl`（`kb-ai/rag/service`）：按 Markdown 标题、空行和句子拆分文本；按约 512 token 合并，重叠约 64 token，并保存标题上下文。
- `DocumentChunk`（`kb-ai/rag/entity`）：表示一个待向量化的文本块，包含 `chunkId`、`chunkIndex`、`totalChunks` 等信息。

为什么放在 `kb-ai`：分块是 RAG 索引前处理，与模型和检索策略强相关；文档服务只负责文档业务元数据和正文管理。

关键权衡：块过大导致召回不准、Prompt 过长；过小造成上下文碎片化；overlap 用来避免边界信息丢失。

### 3.2 向量嵌入

涉及类：

- `EmbeddingService` / `EmbeddingServiceImpl`：调用 LangChain4j `EmbeddingModel`，提供 `embed` 和 `embedBatch`。
- `EmbeddingConfig`：在 RAG 开启且模型配置存在时创建 `EmbeddingModel` Bean，当前按 1024 维处理。

Redis 缓存键为 `rag:emb:{MD5(text)}`，默认 TTL 24 小时。批量接口失败时退化为逐条调用；Redis 不可用时跳过缓存；模型不可用时当前实现返回 1024 维零向量，服务能继续但检索质量会明显下降。

### 3.3 Elasticsearch 索引与混合检索

涉及类：

- `ElasticsearchConfig`：创建 `RestClient`、`ElasticsearchTransport`、`ElasticsearchClient`，支持认证连接。
- `VectorIndexService` / `VectorIndexServiceImpl`：维护 `kb_chunk` 索引，提供批量写入、按文档删除、索引创建和混合搜索。
- `KbChunkDoc`：ES 文档结构，包含文档信息、正文、权限维度、发布时间和 `embedding`。

`embedding` 是 `dense_vector`，维度 1024，余弦相似度。关键词侧使用 BM25，字段主要是 `content`、`document_title`，中文分词使用 IK。向量侧当前源码用 `script_score + cosineSimilarity`；文章描述的 ES 8.x/HNSW 是设计目标，学习时以源码实际实现为准。

混合检索：

```text
BM25 结果 + 向量结果
       -> RRF 融合（score = Σ 1 / (C + rank)，默认 C=60）
       -> 候选集
       -> 可选 LLM 重排序（1～10 分）
       -> Top-K
```

### 3.4 检索编排

`RagRetrievalService` / `RagRetrievalServiceImpl` 负责把嵌入、ES 检索、RRF 和重排序串起来。流程是：确保索引存在 → 生成查询向量 → BM25 + kNN/余弦检索 → 取候选（通常 `max(topK*2, hybridTopK)`）→ 可选 LLM 重排序 → 返回 `RagSearchResultVO`。

注意：`RagSearchRequestDTO` 虽然有 `filterDocIds`、`categoryId`，但当前 `RagSearchController` 只传递 `query`、`topK`、`enableRerank`，这两个过滤字段实际上没有生效，属于接口已声明但链路未接通的问题。

### 3.5 Prompt 与回答

`RagChatService` / `RagChatServiceImpl`：

1. 创建或校验会话；
2. 调用检索服务得到参考片段；
3. 组装约束性 Prompt（只依据资料回答、无资料要明确说明、引用 `[1]` `[2]`）；
4. 调用 LLM；
5. 解析引用，保存用户消息和 AI 消息到 MySQL，更新 Token 统计；
6. 返回 `ChatResponseVO`。

流式接口创建 30 分钟 `SseEmitter`，检索和 Prompt 在请求线程完成，模型调用、消息保存和 SSE 发送在线程池异步执行，发送 `message`、`done`、`error` 事件。检索失败时当前实现会降级为普通 LLM 对话，需在产品上明确是否允许这种“无知识库回答”。

## 4. RAG 接口清单与输入输出示例

### 对话

```text
POST /rag/chat
POST /rag/chat/stream
```

输入：

```json
{
  "conversationId": 10001,
  "message": "公司报销标准是什么？",
  "topK": 5,
  "enableRerank": true
}
```

同步输出（示意）：

```json
{
  "conversationId": 10001,
  "messageId": 20002,
  "content": "根据《财务制度》……[1]",
  "tokens": 173,
  "fromKnowledgeBase": true,
  "citations": [
    {
      "index": 1,
      "documentId": 30001,
      "documentTitle": "财务制度",
      "excerpt": "差旅报销标准……",
      "relevanceScore": 0.91
    }
  ]
}
```

流式输出是 SSE：多次 `event: message` 携带文本片段，最后 `event: done`；异常使用 `event: error`。

### 检索

```text
POST /rag/search
```

输入：

```json
{
  "query": "报销标准",
  "topK": 5,
  "enableRerank": true
}
```

输出（示意）：

```json
[
  {
    "documentId": 30001,
    "documentTitle": "财务制度",
    "content": "差旅报销标准……",
    "score": 0.91,
    "chunkIndex": 2
  }
]
```

### 索引管理

```text
POST   /rag/reindex/{docId}       重建单篇文档
POST   /rag/reindex/batch         按文档 ID 批量重建
POST   /rag/reindex/all           全量重建
DELETE /rag/reindex/{docId}       删除单篇文档索引
GET    /rag/reindex/progress/{taskId} 查询异步任务进度
```

重建接口立即返回任务 ID（示意）：

```json
{"taskId":"ridx-20260829-001","status":"QUEUED"}
```

进度查询（示意）：

```json
{"taskId":"ridx-20260829-001","status":"RUNNING","total":100,"success":72,"failed":1}
```

## 5. 异步重建与中间件记录

### RabbitMQ

涉及类：`RabbitConfig`、`ReindexMessage`、`ReindexServiceImpl`、`ReindexConsumer`。

```text
Exchange: rag.reindex.exchange（TopicExchange）
Queue:    rag.reindex.queue（Durable）
Routing:  rag.reindex.all / rag.reindex.by_ids / rag.reindex.delete
```

调用链：索引接口生成 taskId → 发送 JSON 消息 → 消费者 Feign 获取文档 → 删除旧 chunk → 分块 → 批量嵌入 → Bulk 写 ES → 手动 ACK → Redis 更新进度。配置了死信交换机参数，失败消息可进入死信处理。

面试表达：RabbitMQ 用于把“用户请求”和“耗时索引构建”解耦，接口快速返回；代价是最终一致性，因此必须有手动 ACK、失败重试、死信和可观测的任务进度。

### Redis

- `rag:emb:{MD5(text)}`：Embedding 结果缓存，减少重复调用模型和成本。
- `rag:reindex:progress:{taskId}`：保存异步重建进度，供前端轮询。

缓存失效或 Redis 不可用时应能回源/跳过缓存，不能把 Redis 当作唯一数据源。

### MySQL

保存 AI 会话、消息、反馈、Token 统计；它是可追溯业务记录，不存大规模向量正文。RAG 生成回答后先落库，再把结果返回/推送，便于历史查询。

### Elasticsearch

存储可检索的 chunk 和向量，是 RAG 的检索引擎，不是权威文档源。索引丢失可以从文档服务重新构建。共享 ES 当前需要认证，学习时不要对真实环境执行写入、删索引或全量重建。

### OpenFeign、Nacos、LLM

- `DocumentFeignClient`：RAG 消费者按文档 ID 获取正文和元数据。
- Nacos：提供 `kb-document`、`kb-ai` 的注册发现；Feign 只负责调用，不负责服务注册。
- Qwen/DeepSeek：分别承担 Embedding 和 Chat/重排序模型调用；外部模型失败时要有超时、重试、降级策略。

## 6. 文档服务触发点与一致性

`kb-document` 的 `DocumentServiceImpl` 注入 `RagFeignClient`，在以下事件中异步调用 `CompletableFuture.runAsync(..., asyncTaskExecutor)`：发布成功、已发布内容更新、草稿发布 → 重建索引；变为非发布、删除、归档 → 删除索引。

优点：文档主链路不等待 ES/RAG，响应快且服务解耦。风险：MySQL 状态与 ES 索引短时间不一致，或跨服务调用失败。工程上要配合重试、幂等（构建前删除旧 chunk）、失败任务监控和手工/定时重建。

## 7. 不能忽略的设计点（面试版）

1. **分块参数**：`chunkSize=512`、`chunkOverlap=64` 直接影响召回质量和 Prompt 成本。
2. **混合检索**：BM25 擅长关键词，向量擅长语义；RRF 不要求两种分数同量纲，适合融合排名。
3. **重排序成本**：LLM 重排提升精度但增加延迟和费用，只对候选集执行并允许关闭。
4. **索引幂等**：重建前删除同文档旧 chunk，避免重复数据；失败后可安全重试。
5. **最终一致性**：文档库是源，ES 是派生索引；不能把 ES 写成功当作文档发布事务成功。
6. **降级边界**：Embedding 不可用返回零向量会损害质量；RAG 检索失败降级普通对话可能导致无依据回答，生产上应记录并告警。
7. **权限过滤**：索引字段包含 `author_id`、`team_id`、`doc_status` 等，检索时必须真正拼接权限条件；当前 `filterDocIds/categoryId` 未接通要修复或明确不支持。
8. **文章与源码差异**：文章偏 ES 8.x/HNSW 设计，源码实际使用 `script_score + cosineSimilarity`，回答面试题时要区分“目标架构”和“落地实现”。

## 8. 面试时的 30 秒回答

“我们把 AI 能力放在独立的 `kb-ai` 服务。文档发布后，文档服务通过 Feign 异步触发 RAG 重建任务，任务进入 RabbitMQ。消费者从文档服务取正文，按标题和段落切成带 overlap 的 chunk，用 Qwen Embedding 转成 1024 维向量，和元数据一起写入 ES。查询时同时走 BM25 和向量检索，用 RRF 融合，再用 LLM 可选重排，把 Top-K 片段放进 Prompt，生成带引用的答案。会话和消息落 MySQL，Embedding 与任务进度放 Redis。这样主链路响应快，但要通过幂等、重试、死信和重建任务处理最终一致性。”

