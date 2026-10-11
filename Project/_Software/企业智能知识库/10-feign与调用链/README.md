# OpenFeign 记录：跨服务 API 清单

## 文档服务调用 AI

### `RagFeignClient`

- `POST /rag/reindex/{docId}`：重建单文档 RAG chunk/向量索引。
- `DELETE /rag/reindex/{docId}`：删除文档 RAG 索引。

### `KAGFeignClient`

- `POST /kag/build/{docId}`：构建单文档实体关系图谱。
- `DELETE /kag/build/{docId}`：删除该文档图谱数据。

## 文档服务调用搜索

### `SearchIndexFeignClient`

- `POST /index/document`：新增或更新 ES 文档索引。
- `DELETE /index/document/{documentId}`：删除 ES 文档索引。

## Feign 面试表达

OpenFeign 是声明式 HTTP 客户端：接口方法、HTTP 方法和路径写在 Java 注解中，运行时由代理发起请求；服务名交给 Nacos 解析，LoadBalancer 负责实例选择。要配合超时、重试、降级和幂等，不能把网络调用当成本地方法调用。

## 失败影响

- RAG/KAG 失败：文档仍可处于已发布，但 AI 能力暂不可用，需要异步重试。
- ES 失败：文档主数据仍在 MySQL，但搜索结果暂时缺失。
- 任何远程调用都要记录 traceId、耗时和错误原因。

