# MongoDB 记录：正文与内容版本

## 关键类

`kb-document/service/impl/DocumentContentServiceImpl`、`DocumentContent`。

## 数据关系

```text
MySQL kb_document.id      = 文档元数据 ID（Long）
MySQL kb_document.content_id = MongoDB document_content._id（String）
MongoDB document_content.documentId = MySQL 文档 ID（索引）
MongoDB document_content.content = Markdown 正文
MongoDB document_content.version = 内容版本
```

## 主要接口

- `saveContent(documentId, content)`：新建正文，返回 Mongo `_id`。
- `updateContent(documentId, content)`：按 `documentId` 找到正文并更新。
- `getContentById(contentId)`：按 Mongo `_id` 查询正文。

## 为什么不共用 ID？

技术上可以共用，但独立 `content_id` 将元数据身份和正文存储身份解耦，便于正文版本、快照、迁移和独立生命周期。代价是跨库无统一事务，必须处理 MySQL 成功而 Mongo 失败的情况。

## 面试坑点

- 不要说 Mongo 只保存“附件”；这里保存的是文档真实正文。
- `documentId` 建索引后，业务更新主要按文档 ID定位，不代表 `_id` 没有意义。
- 大文本读取、解析和索引构建要考虑超时、重试和幂等。

