# RAG 改进设想与 Reranker 方案

这份记录保存对当前 RAG 实现的改进设想，后续如要让大模型改造代码，应以此文件为需求参考。

## 一、当前实现的问题

1. **所有请求强制走 RAG**：用户说“你好”也会执行 Embedding、ES 检索，浪费延迟和模型资源。
2. **RAG 检索不是 Tool**：知识库检索和普通聊天耦合在固定流程中，缺少按需调用能力。
3. **逐条 LLM 重排序成本过高**：20 个候选就产生约 20 次聊天模型请求；每次独立评分，缺少横向比较，容易出现大量相同分数，结果还会受模型随机性影响。
4. **RRF 权重固定**：BM25 和向量排名默认等贡献，没有根据业务调节关键词与语义搜索的权重。
5. **Redis Embedding 缓存收益有限**：`embedBatch` 正常路径没有读写缓存，24 小时 TTL 缺少明确业务依据；Redis 不参与 ES 向量检索。
6. **整篇文档删除后重建**：大文档更新时会重复分片、重复 Embedding 和重复写 ES。
7. **无安全的无结果兜底**：RAG 无结果时直接把用户问题交给普通 LLM，可能产生没有企业资料依据的回答。

## 二、目标架构

```text
用户问题
  -> 轻量路由（规则/小模型/Flash）
       ├─ 普通闲聊 -> 直接调用聊天模型
       └─ 企业知识问题 -> 调用 searchKnowledgeBase Tool
                              -> Embedding
                              -> ES 混合检索
                              -> Reranker
                              -> 返回 Top-K chunk
                              -> 强模型生成最终答案
```

### Tool / MCP 设计

- 当前 `kb-ai` 内部优先使用 Function Tool：实现简单、网络层少。
- 如果未来多个 Agent 或外部客户端需要复用，再把检索能力封装为 MCP Server。
- Tool 只负责决定“是否查知识库”；权限必须由服务端强制执行，不能交给模型决定。
- Tool 返回结构化结果，例如 `found`、`results`、`reason`。
- `found=false` 时，系统应直接说明“根据现有知识库暂时无法回答”，或向模型注入禁止编造的约束；不能无提示地自由回答企业事实。

## 三、模型分工

| 阶段 | 推荐组件 | 说明 |
|---|---|---|
| 意图路由 | 规则、轻量模型或 Flash | 判断是否需要知识库 |
| 文本向量化 | Embedding 专用模型 | 不使用聊天模型代替 |
| 初步召回 | Elasticsearch | BM25 + 向量检索，不调用 LLM |
| 重排序 | 专用 Reranker，或 Flash | 批量比较候选，不使用强模型逐条打分 |
| 最终回答 | DeepSeek Pro 等强模型 | 负责理解、综合和表达 |

## 四、Reranker 改造要求

### 当前方案（不推荐）

```text
20 个候选 chunk
  -> 20 次独立 LLM 请求
  -> 每次返回 1～10 分
  -> 按分数排序
```

### 推荐方案 A：批量 Listwise LLM Rerank

一次请求放入问题和多个候选，让模型统一比较并返回排序后的 ID 和分数：

```json
{
  "query": "公司报销标准是什么？",
  "candidates": [
    {"id": "c1", "content": "……"},
    {"id": "c2", "content": "……"}
  ]
}
```

要求模型只返回结构化 JSON：

```json
{
  "ranking": [
    {"id": "c2", "score": 9},
    {"id": "c1", "score": 6}
  ]
}
```

设置低温度或 `temperature=0`，校验 ID 不重复且分数合法。候选过多时先用 RRF 取前 10～20 条，再进行批量重排。

### 推荐方案 B：专用 Cross-Encoder Reranker

```text
query + chunk
  -> Reranker 模型
  -> 相关性分数
```

它专门训练“问题与文本片段相关性”，通常比聊天 LLM 更快、更便宜、更稳定。优先级高于用 Flash 做 LLM 重排。

### 推荐方案 C：不使用 LLM，直接加权 RRF

```text
finalScore = α × BM25_RRF + β × Vector_RRF
```

例如企业制度类资料关键词很重要，可以设置：

```text
α = 0.7
β = 0.3
```

权重应通过离线评测集调参，而不是固定假设 1:1。

## 五、增量索引方案

对每个最终 chunk 计算内容 Hash，并保存到 ES 或索引清单：

```text
document_id + heading + chunk_content -> content_hash
```

更新时：

```text
Hash 相同 -> 复用原 embedding，不重新调用模型
Hash 不同 -> 重新 Embedding 并更新 ES
旧 Hash 消失 -> 删除对应旧 chunk
```

注意：内容 Hash 相同不代表整条 ES 文档完全不用更新。如果标题、权限、发布状态或团队归属变化，仍需要更新元数据。

仅保存 `document_id -> chunk_id` 列表不能完整解决增量更新，因为重新分片后 chunk 边界可能变化。更关键的是稳定的 chunk Hash 或稳定的分片规则。

## 六、Redis 处理建议

- Redis 不是向量检索库，ES 负责向量距离计算和 Top-K。
- 如果不实现基于 Hash 的增量 Embedding，可以移除 Embedding 长缓存，或只保留短 TTL 的失败重试缓存。
- 如果保留缓存，应把模型名、模型版本、向量维度和文本 Hash 纳入 Key，避免模型升级后读取旧向量。
- 批量 Embedding 要先查缓存，未命中部分再批量请求模型，并将新结果回写缓存；否则缓存无法服务主链路。

## 七、建议的降级顺序

```text
路由失败 -> 按业务规则决定是否调用知识库
Embedding 失败 -> 关键词 BM25（或直接返回知识库暂不可用）
向量搜索失败 -> 使用 BM25 结果
Reranker 失败 -> 使用加权 RRF 结果
无任何检索结果 -> 返回“知识库暂时无法回答”，禁止自由编造企业事实
最终强模型失败 -> 使用明确错误提示或轻量模型兜底
```

## 八、改造优先级

1. 先将 RAG 抽成 `searchKnowledgeBase` Tool，避免闲聊强制检索。
2. 将逐条 LLM 重排序改为批量 Listwise，或接入专用 Cross-Encoder。
3. 将 RRF 改为可配置加权 RRF，并建立离线问题集调参。
4. 明确无结果和检索异常时的安全兜底，禁止无依据回答。
5. 再实现 chunk Hash 增量索引；确认有大文档或频繁编辑需求后，再决定是否保留 Redis Embedding 缓存。

