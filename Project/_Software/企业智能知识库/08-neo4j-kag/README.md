# Neo4j / KAG 记录：实体关系和图谱检索

## 构建接口

- `POST /kag/build/{docId}`：构建单篇文档图谱。
- `POST /kag/build/batch`：批量构建。
- `POST /kag/build/all`：全量构建。
- `DELETE /kag/build/{docId}`：删除文档相关图谱数据。

## 构建过程

```text
正文
→ 文本分块
→ LLM/规则抽取实体与关系
→ 实体归一化
→ Neo4j 节点/关系
```

典型概念链：`KnowledgeDocument → DocumentChunk → KnowledgeEntity`。

## 查询接口

- `POST /kag/search`：返回实体、关系、图谱上下文或推理路径，不直接等于最终回答。
- `POST /kag/chat`、`/kag/chat/stream`：图谱检索结果作为大模型上下文。

## 与 `kb-graph` 的区别

当前文档发布通过 Feign 调 `kb-ai` 的 KAG 构建接口；`kb-graph` 更偏图谱查询、展示和清理。不要把“构建图谱”和“查询图谱”混成一个服务职责。

