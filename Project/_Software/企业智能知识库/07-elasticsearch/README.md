# Elasticsearch 记录：全文索引与 RAG 向量块

## 两类索引

### 文档索引（`kb-search`）

`POST /index/document` 写入标题、摘要、正文片段、分类、标签、状态、作者和计数等字段；`DELETE /index/document/{documentId}` 删除索引文档。用于普通全文/高级条件搜索。

### RAG chunk 索引（`kb-ai`）

文档正文切块后生成 embedding，写入 `kb_chunk`。`POST /rag/search` 返回相关文本块，`/rag/chat` 再把文本块作为上下文交给大模型。

## 发布后的索引链路

```text
审核通过
→ SearchIndexFeignClient 同步文档索引
→ RagFeignClient 异步切块/向量化
→ 用户可全文搜索或进行 RAG 对话
```

## 一致性与认证

ES 是派生数据，不是 MySQL 真源；同步失败需要重试或全量重建。共享环境返回 HTTP 401 说明服务可达但需要认证，笔记不保存认证信息。

