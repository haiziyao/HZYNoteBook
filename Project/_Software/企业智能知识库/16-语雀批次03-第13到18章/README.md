# 语雀第 13～18 章：文件、文档、导入、个人工作区与分享

> 资料主线：语雀第 13～18 章；源码用于核对当前实现。本文不重复 JWT/登录认证，也不展开普通 CRUD 的逐行解释。

## 0. 这一批的主线

这 6 章围绕“文档从进入系统到被使用”的一条链路：

```text
上传文件/导入 Markdown
        ↓
文件二进制 → RustFS（S3 兼容对象存储）
文件元数据 → kb_file MySQL
文档元数据 → kb_document MySQL
文档正文   → MongoDB document_content
        ↓
草稿/编辑/发布
        ↓
收藏、最近访问、PDF 导出、分享链接
```

当前源码还有两条后台线路：

```text
发布/归档 → CompletableFuture + Feign → RAG、KAG、ES 索引
文件格式转换 → RabbitMQ transcode.exchange → 转码消费者
```

统一响应外壳是 `com.knowledge.base.common.result.Result<T>`：

```json
{"code":200,"message":"操作成功","data":{},"timestamp":1720000000000}
```

下面示例中的雪花 ID 只表示格式，不代表真实数据。

---

## 1. 不能忽略的设计

### 1.1 文件服务：二进制与元数据分离

| 设计点 | 对应类 | 为什么重要 |
|---|---|---|
| RustFS 存二进制，MySQL 存文件元数据 | `kb-file/.../FileServiceImpl`、`FileInfo` | 数据库不承载大文件；元数据可分页、可检索、可审计 |
| 存储后端抽象 | `FileStorage`、`RustFileStorage`、`FileStorageFactory` | 业务层不绑定 RustFS，将来可替换本地盘、S3、MinIO |
| 配置驱动选择实现 | `FileStorageProperties`、`RustFsS3Config` | `file.storage.type=rustfs` 决定注入哪一个实现；S3 客户端统一封装 |
| SHA-256 去重/秒传 | `FileServiceImpl.calculateFileHash`、`file_hash` 字段 | 同一文件可复用已有对象，降低存储和上传成本 |
| 断点续传 | `RustFileStorage` 的 `UploadSession`、`initResumableUpload/uploadChunk/mergeChunks` | 大文件失败后只重传分片；会话状态目前保存在进程内，集群部署需外置 |
| 删除顺序 | `FileServiceImpl.deleteFile` | 先更新 MySQL 逻辑状态，再删除对象；失败时要考虑孤儿对象和补偿任务 |
| 转码异步化 | `kb-file/config/RabbitMQConfig`、`TranscodeConsumer` | HTTP 请求只创建任务，实际 FFmpeg 转码由消费者执行 |

### 1.2 文档：MySQL 元数据 + MongoDB 正文

| 设计点 | 对应类 | 面试要点 |
|---|---|---|
| `Document.content` 被 `@TableField(exist=false)` 忽略 | `Document` | MP 不会把正文写入 MySQL；正文由 `DocumentContentService` 处理 |
| MySQL 只保存 `content_id` | `Document`、`DocumentContent` | 两库通过业务 `documentId`/Mongo `_id` 关联，避免把大文本塞进关系库 |
| 正文更新原文档，不盲目新建 | `DocumentContentServiceImpl.updateContent` | 保留 Mongo `_id`，递增 `version`；新文档才 `saveContent` |
| 内容处理集中化 | `DocumentContentServiceImpl.processContent`、`MarkdownProcessService` | 统一做 Markdown 缩进规范化、外部图片转本站文件 URL、摘要生成 |
| 主事务与 Mongo 不是同一事务边界 | `DocumentServiceImpl.create/update` | MySQL 回滚不等于 Mongo 回滚；代码通过异常处理、补偿/重试降低跨库不一致 |
| 发布后索引异步触发 | `DocumentServiceImpl.triggerRagReindex/triggerKAGBuild/triggerSearchIndex` | 用户先得到发布结果，索引稍后构建；失败不阻塞主业务但必须可重试 |

### 1.3 导入与草稿：先解析再上传

| 设计点 | 对应类 | 不能忽略的原因 |
|---|---|---|
| 先解析文件，再调用 Feign 上传 | `DocumentServiceImpl.uploadAndCreateDocument` | Feign 消费 `MultipartFile` 输入流；先上传可能导致解析阶段无法再次读取 |
| 外部图片由后端代理下载 | `FileUploadServiceImpl.uploadImageFromUrl`、`MarkdownProcessService` | 避免浏览器跨域、外链失效和不可控域名；内部域名可跳过重复上传 |
| 导入默认是草稿 | `uploadAndCreateDocument`、`AutoSaveDTO` | 导入/自动保存不应直接对外发布；发布走单独权限和审核配置 |
| 自动保存不触发索引 | `DocumentController.autoSaveDocument` | 编辑中的半成品不能进入 RAG/KAG/ES；快照异步写 Mongo |
| 大整数 ID 前端用字符串 | `BatchExportRequest.documentIds`、前端服务 | JavaScript `Number` 会丢失 64 位雪花 ID 精度 |

### 1.4 分类与用户管理：CRUD 之外的边界

| 设计点 | 对应类 | 风险 |
|---|---|---|
| 分类是递归树 | `Category`、`CategoryServiceImpl`、`CategoryController` | 移动分类时必须防止把节点移到自身或其子孙节点下面，避免环 |
| 树与平铺列表分开 | `getCategoryTree` / `getAllCategories` | 树用于导航，平铺用于选择器和批量操作；不要让前端自己猜层级 |
| 用户角色与直接权限并存 | `UserController`、`UserServiceImpl`、`UserRoleMapper` | `getUserPermissions` 必须合并角色权限和直接授权，删除/覆盖时要明确边界 |
| 状态筛选在服务层落实 | `UserController.pageUsers` | 禁用用户、已删除用户不能因为分页接口遗漏条件而泄露 |

### 1.5 收藏与最近访问：一致性策略不同

| 功能 | 对应类 | 设计 |
|---|---|---|
| 收藏 | `UserFavoriteServiceImpl` | 记录与 `document.favoriteCount` 在同一 MySQL 事务；新增幂等，取消用物理删除；唯一键 `(user_id, document_id, deleted)` 防重复 |
| 最近访问 | `DocumentAccessServiceImpl`、`DocumentServiceImpl.viewDocument` | 删除旧记录再插入新记录，保持一条最新记录；`CompletableFuture.runAsync` 异步，记录失败不能影响浏览 |
| 查询避免 N+1 | `UserFavoriteMapper.getUserFavorites`、`DocumentAccessMapper.selectRecentAccessByUserId` | 用 LEFT JOIN 一次取标题、摘要、分类、作者等展示字段 |
| 未登录访问 | `DocumentAccessServiceImpl` | 没有用户上下文时跳过记录，不把“记录失败”变成“文档打不开” |

源码差异：语雀文章说明访问记录使用自增 ID，但当前 `DocumentAccess` 使用 `@TableId(type = IdType.ASSIGN_ID)`；以当前源码为准，面试时应主动指出这个差异。

### 1.6 PDF 与分享：输出格式和访问边界

| 设计点 | 对应类 | 面试表达 |
|---|---|---|
| PDF 直接返回字节流 | `PdfExportServiceImpl`、`DocumentController.downloadDocumentPdf` | 后端读 Mongo 正文，PDFBox 渲染后设置 `Content-Disposition` 下载 |
| 中文字体与动态分页 | `PdfExportServiceImpl` | PDFBox 默认字体不支持中文；需要加载字体、测量 CJK 宽度、自动换行和换页 |
| 分享 ID 与数据库主键分离 | `DocumentShareServiceImpl.generateShareId`、`DocumentShare.shareId` | 雪花 `id` 是内部主键，12 位 `shareId` 是外部不可预测访问 Key |
| 密码不存明文 | `DocumentShareServiceImpl` | 当前使用 MD5 摘要比对；生产系统更推荐带盐慢哈希（如 BCrypt/Argon2） |
| 过期/次数/删除三重校验 | `getShareById`、`verifyShareAccess`、`accessShare` | 仅返回分享信息不增加次数；真正访问时校验并递增 `accessCount` |
| 公开接口绕过登录但不能绕过业务校验 | `ShareController` + 网关白名单 | `/share/**` 放在 `ProtectedRoute` 外；网关只放行路径，密码/过期/次数仍由服务判断 |

---

## 2. 主要接口与输入输出实例

所有 JSON 接口都包在 `Result<T>` 中。认证接口是否需要登录以源码上的 `@PreAuthorize` 和网关配置为准；公开分享访问是例外。

### 2.1 第 13 章：文件上传（`kb-file`）

#### 核心接口

| 方法 | 路径 | 输入 | 输出/作用 |
|---|---|---|---|
| POST | `/files/upload` | `multipart file` + `fileType/accessLevel/teamId` | 返回 `FileInfoVO`，包含 `id/fileUrl/previewUrl/fileHash/storageType` 等 |
| POST | `/files/upload/batch` | `multipart files[]` + DTO | 返回 `FileInfoVO[]`；批量结果整体返回 |
| GET | `/files/download/{fileId}` | 路径中的文件 ID，可带扩展名 | 二进制流；响应 `Content-Disposition: attachment` |
| GET | `/files/{fileId}` | `fileId` | 返回文件元数据 `FileInfoVO` |
| POST | `/files/page` | `originalName/fileType/uploaderId/accessLevel/current/size` | `PageResult<FileInfoVO>` |
| DELETE | `/files/{fileId}` | `fileId` | `Result<Boolean>`；删除元数据并尝试删除 RustFS 对象 |
| DELETE | `/files/batch` | `[123,456]` | 批量删除结果 |
| GET | `/files/preview/{fileId}` | `fileId` | 返回预览 URL；带 `/**` 的变体直接返回文件流 |
| POST | `/files/convert-url` | `imageUrl` | 下载外链图片并返回 `UrlConvertResponse` |
| POST | `/files/batch-convert` | `string[]` URL | 返回成功/失败映射和计数 |
| POST | `/files/convert/{fileId}` | `targetFormat` | 返回新文件 ID；发送 RabbitMQ 转码消息 |
| GET | `/files/stream/{fileId}/master.m3u8` | `fileId` | HLS 播放列表流 |
| GET | `/files/stream/{fileId}/{segment}.ts` | `fileId/segment` | HLS TS 分片流 |
| GET | `/files/thumbnail/{fileId}` | `fileId` | 视频缩略图二进制流 |

上传示例：

```http
POST /api/file/files/upload
Content-Type: multipart/form-data

file=@cover.png
fileType=IMAGE
accessLevel=2
teamId=10001
```

```json
{"code":200,"message":"操作成功","data":{"id":315091148181999616,"originalName":"cover.png","fileSize":50214,"fileSizeReadable":"49.04 KB","mimeType":"image/png","fileUrl":"/files/download/315091148181999616","previewUrl":"/files/preview/315091148181999616","storageType":"RUSTFS","downloadCount":0}}
```

分页示例：

```json
{"current":1,"size":10,"originalName":"cover","fileType":"IMAGE","accessLevel":2}
```

```json
{"code":200,"data":{"records":[{"id":315091148181999616,"originalName":"cover.png"}],"total":1,"current":1,"size":10}}
```

外链转存示例：

```http
POST /api/file/files/convert-url?imageUrl=https%3A%2F%2Fcdn.example.com%2Fa.png
```

```json
{"code":200,"data":{"originalUrl":"https://cdn.example.com/a.png","convertedUrl":"/files/preview/315091148181999616"}}
```

> 语雀第 13 章重点是图片上传和 RustFS；当前源码额外提供视频 HLS、缩略图、格式转换、断点续传。不要把扩展接口误认为文章当时已经实现的最小范围。

### 2.2 第 14 章：文档查询、创建、删除

| 方法 | 路径 | 输入 | 输出/作用 |
|---|---|---|---|
| POST | `/documents` | `DocumentDTO` | 返回新文档雪花 ID；元数据写 MySQL，正文写 Mongo |
| PUT | `/documents` | `DocumentDTO.id + 要更新字段` | 返回 `Boolean`；正文更新 Mongo，元数据更新 MySQL |
| PATCH | `/documents/{documentId}/summary` | `{ "summary":"..." }` | 仅更新摘要，最长 500 字 |
| GET | `/documents/{documentId}` | `documentId` | `DocumentVO` 详情；可带 Mongo 正文 |
| GET | `/documents/{documentId}/view` | `documentId` | 详情 + 浏览次数增加 + 异步最近访问记录 |
| GET | `/documents/page` | `current,size,categoryId,teamId,keyword,status,sortBy,sortOrder,authorId` | `IPage<DocumentVO>`；关键词主要查 MySQL 字段 |
| GET | `/documents/{documentId}/neighbors` | `documentId` | 上一篇/下一篇 `DocumentNeighborVO` |
| DELETE | `/documents/{documentId}` | `documentId` | 逻辑删除文档，并异步清理索引/图谱 |
| PUT | `/documents/{documentId}/publish` | `documentId` | 按 `system.requireApproval` 决定提交审核或直接发布 |
| PUT | `/documents/{documentId}/archive` | `documentId` | 归档并异步删除 RAG/KAG/ES 索引 |

创建示例：

```json
{
  "title":"Spring Boot 使用指南",
  "summary":"介绍 Spring Boot 基础用法",
  "content":"# 开始\n\n这是正文。",
  "documentType":1,
  "categoryId":1001,
  "teamId":2001,
  "tags":"Java,Spring Boot",
  "status":0,
  "isPublic":1,
  "allowComment":1
}
```

```json
{"code":200,"message":"创建文档成功","data":315091148181999616}
```

分页查询示例：

```http
GET /api/document/documents/page?current=1&size=10&categoryId=1001&keyword=Spring&status=1&sortBy=updatedAt&sortOrder=desc
```

```json
{"code":200,"data":{"records":[{"id":315091148181999616,"title":"Spring Boot 使用指南","status":1,"contentLength":20,"categoryName":"Java"}],"total":1,"current":1,"size":10}}
```

### 2.3 第 15 章：分类与用户

#### 分类接口（`/categories`）

| 方法 | 路径 | 输入 | 输出/作用 |
|---|---|---|---|
| POST | `/categories` | `{name,description,parentId,sortOrder,icon}` | 返回分类 ID |
| PUT | `/categories` | `{id,name,...}` | 更新分类 |
| DELETE | `/categories/{categoryId}` | 路径 ID | 删除分类；服务层应检查子分类/文档引用 |
| GET | `/categories/{categoryId}` | 路径 ID | 分类详情 |
| GET | `/categories/tree` | 无 | 递归树 `CategoryVO[]` |
| GET | `/categories/children/{parentId}` | 父 ID | 直接子节点 |
| PUT | `/categories/{categoryId}/move` | `newParentId` | 调整层级，必须防循环 |
| GET | `/categories/list` | 无 | 平铺分类列表 |

```json
POST /api/document/categories
{"name":"后端开发","description":"Java 相关","parentId":0,"sortOrder":1,"icon":"java"}
```

```json
{"code":200,"message":"创建分类成功","data":1001}
```

```json
{"code":200,"data":[{"id":1001,"name":"后端开发","children":[{"id":1002,"name":"Spring"}]}]}
```

#### 用户接口（`kb-user-auth` 的 `/users`）

| 方法 | 路径 | 输入 | 输出/作用 |
|---|---|---|---|
| POST | `/users` | `UserDTO`：username、password、realName、email、phone、status 等 | 返回用户 ID |
| PUT | `/users` | `id + 可更新字段` | 更新用户 |
| DELETE | `/users/{userId}` | 路径 ID | 删除用户 |
| GET | `/users/{userId}` | 路径 ID | `UserVO` |
| GET | `/users/page` | `current,size,keyword,role,status` | 用户分页 |
| PUT | `/users/{userId}/password/reset` | `newPassword` | 管理员重置密码 |
| PUT | `/users/password/change` | `oldPassword,newPassword` | 当前用户改密 |
| POST | `/users/{userId}/roles` | `[roleId...]` | 覆盖/分配角色关联 |
| GET | `/users/{userId}/roles` | 路径 ID | 角色 ID 列表 |
| POST | `/users/{userId}/permissions` | `[permissionId...]` | 直接分配权限 |
| GET | `/users/{userId}/permissions` | 路径 ID | 合并后的权限编码 |
| GET | `/users/me/stats` | 登录上下文 | 当前用户文档/浏览/点赞/评论统计 |

创建用户示例：

```json
{"username":"zhangsan","password":"123456","realName":"张三","email":"zhangsan@example.com","status":1}
```

```json
{"code":200,"message":"创建用户成功","data":10001}
```

分配角色示例：

```http
POST /api/user/users/10001/roles
[10,20]
```

```json
{"code":200,"message":"角色分配成功","data":true}
```

### 2.4 第 16 章：导入文档与草稿箱

#### 文件上传/图片转存（`kb-document` 的 `FileUploadController`）

| 方法 | 路径 | 输入 | 输出 |
|---|---|---|---|
| POST | `/files/upload` | `multipart file` | `FileUploadResponse{url,fileName,fileSize,fileSizeReadable}` |
| POST | `/files/upload-from-url` | `imageUrl` | `ImageConvertResponse{originalUrl,convertedUrl}` |
| POST | `/files/batch-upload` | `files[]` | `BatchUploadResponse{fileUrls,successCount,failureCount}` |
| POST | `/files/convert-url` | `imageUrl` | 单图外链转存结果 |
| POST | `/files/batch-convert` | URL 数组 | 成功/失败映射 |

```http
POST /api/document/files/upload
Content-Type: multipart/form-data

file=@manual.md
```

```json
{"code":200,"data":{"url":"/files/download/315...","fileName":"manual.md","fileSize":2048,"fileSizeReadable":"2.00 KB"}}
```

#### 文档导入、自动保存

| 方法 | 路径 | 输入 | 输出/作用 |
|---|---|---|---|
| POST | `/documents/upload` | `multipart file` | 仅上传文档文件，返回文件路径 |
| POST | `/documents/upload/parse` | `multipart file` | 解析 pdf/docx/xlsx/pptx/txt/md，上传源文件，创建草稿，返回 `documentId/title/fileUrl/contentLength/contentPreview` |
| POST | `/documents/autosave` | `AutoSaveDTO{id?,title?,content?,summary?,categoryId?,teamId?,tags?}` | 创建或更新草稿，返回文档 ID；不触发索引 |
| PUT | `/documents/autosave/dismiss` | 无 | 当前用户确认放弃自动保存草稿 |
| GET | `/documents/{documentId}/autosave-history` | `current,size` | 自动保存历史分页 |
| GET | `/documents/{documentId}/autosave-history/{snapshotId}` | Mongo 快照 ID | 返回快照完整 Markdown |

导入示例：

```http
POST /api/document/documents/upload/parse
Content-Type: multipart/form-data

file=@spring-guide.md
```

```json
{"code":200,"message":"文件解析并创建文档成功","data":{"documentId":315091148181999616,"title":"spring-guide","fileUrl":"/files/download/315...","fileSize":4096,"contentLength":1880,"contentPreview":"# Spring Guide\n..."}}
```

自动保存示例：

```json
{"id":null,"title":"","content":"# 尚未完成","summary":"编辑中","categoryId":1001,"tags":"draft"}
```

```json
{"code":200,"message":"自动保存成功","data":315091148181999616}
```

### 2.5 第 17 章：收藏与最近访问

#### 收藏接口

| 方法 | 路径 | 输入 | 输出 |
|---|---|---|---|
| POST | `/favorite/toggle/{documentId}` | 路径 ID | `true=已收藏，false=已取消` |
| POST | `/favorite/add/{documentId}` | 路径 ID | 幂等添加，返回成功 |
| DELETE | `/favorite/remove/{documentId}` | 路径 ID | 物理删除收藏并减少计数 |
| GET | `/favorite/check/{documentId}` | 路径 ID | 是否已收藏 |
| GET | `/favorite/list` | 当前用户上下文 | `UserFavoriteVO[]` |
| GET | `/favorite/count/{documentId}` | 路径 ID | 收藏数量 |

```http
POST /api/document/favorite/toggle/315091148181999616
```

```json
{"code":200,"data":true}
```

收藏列表返回示例：

```json
{"code":200,"data":[{"id":8001,"documentId":315091148181999616,"documentTitle":"Spring Boot 使用指南","documentCategoryName":"后端开发","documentAuthorName":"张三","documentStatus":1,"favoriteTime":"2026-08-29T10:00:00"}]}
```

#### 最近访问接口

| 方法 | 路径 | 输入 | 输出 |
|---|---|---|---|
| POST | `/access/record` | `documentId,documentTitle` 请求参数 | 记录当前用户访问；重复访问会刷新时间 |
| GET | `/access/recent` | `limit`（默认 20） | `DocumentAccessVO[]`，按访问时间倒序 |
| DELETE | `/access/remove/{documentId}` | 路径 ID | 删除当前用户该文档记录 |
| DELETE | `/access/clear` | 无 | 清空当前用户全部访问记录 |

```http
POST /api/document/access/record?documentId=315091148181999616&documentTitle=Spring%20Boot%20%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97
```

```json
{"code":200,"data":true}
```

```json
{"code":200,"data":[{"id":9001,"documentId":315091148181999616,"documentTitle":"Spring Boot 使用指南","summary":"介绍基础用法","categoryName":"后端开发","authorName":"张三","accessTime":"2026-08-29T10:02:00","status":1}]}
```

### 2.6 第 18 章：PDF 导出与文档分享

#### PDF 接口

| 方法 | 路径 | 输入 | 输出 |
|---|---|---|---|
| GET | `/documents/{documentId}/export-pdf` | 文档 ID | 返回 PDF 导出 URL 字符串 |
| GET | `/documents/{documentId}/download-pdf` | 文档 ID | `application/pdf` 二进制流，浏览器下载 |
| POST | `/documents/batch-export` | `{documentIds:["315..."],format:"pdf"|"markdown"}` | ZIP 二进制流 |

```http
GET /api/document/documents/315091148181999616/download-pdf
```

```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename*=UTF-8''Spring%20Boot.pdf
```

批量导出示例：

```json
{"documentIds":["315091148181999616","315091148181999617"],"format":"pdf"}
```

输出为 `application/zip`，不是 JSON。

#### 认证分享接口

| 方法 | 路径 | 输入 | 输出/作用 |
|---|---|---|---|
| POST | `/documents/share` | `ShareDTO` | 创建分享，返回 `ShareVO` |
| GET | `/documents/share/{shareId}` | 分享 ID | 当前用户查看分享详情 |
| POST | `/documents/share/{shareId}/access` | `password?` | 校验并递增访问次数，返回文档 ID |
| GET | `/documents/{documentId}/shares` | 文档 ID | 该文档有效分享列表 |
| GET | `/documents/share/my` | 当前用户 | 我的分享列表 |
| PUT | `/documents/share/{shareId}` | 过期、密码、次数、描述 | 更新分享设置 |
| DELETE | `/documents/share/{shareId}` | 分享 ID | 状态置为已删除 |

创建分享示例：

```json
{
  "documentId":315091148181999616,
  "shareType":1,
  "expireType":2,
  "expireTime":"2026-09-01T23:59:59",
  "accessLimit":10,
  "requirePassword":1,
  "password":"abc123",
  "description":"给面试小组阅读"
}
```

```json
{"code":200,"data":{"shareId":"a1b2c3d4e5f6","shareUrl":"/share/a1b2c3d4e5f6","documentId":315091148181999616,"title":"Spring Boot 使用指南","expireType":2,"expireTime":"2026-09-01T23:59:59","expired":false,"requirePassword":true,"accessCount":0}}
```

#### 公开分享接口（无需登录）

| 方法 | 路径 | 输入 | 输出/作用 |
|---|---|---|---|
| GET | `/share/{shareId}` | 分享 ID | 获取分享基本信息，不增加次数；隐藏 `documentId` |
| POST | `/share/{shareId}/verify` | `password?` | 只验证密码/过期/次数，不增加次数 |
| POST | `/share/{shareId}/access` | `password?` | 验证并增加次数，返回完整 `DocumentVO` |

```http
POST /api/document/share/a1b2c3d4e5f6/verify?password=abc123
```

```json
{"code":200,"data":true}
```

```http
POST /api/document/share/a1b2c3d4e5f6/access?password=abc123
```

```json
{"code":200,"data":{"id":315091148181999616,"title":"Spring Boot 使用指南","content":"# 开始...","status":1}}
```

---

## 3. 中间件/模块归属记录

| 中间件/基础设施 | 关键模块/类 | 主要用途 |
|---|---|---|
| MySQL `kb_file` | `FileInfo`、`FileMapper`、`FileServiceImpl` | 文件元数据、哈希、访问级别、下载次数、转码状态 |
| RustFS（S3 API） | `RustFileStorage`、`RustFsS3Config` | 文件二进制、图片、视频、HLS 分片 |
| MySQL `kb_document` | `Document`、`DocumentMapper`、`DocumentShare`、`UserFavorite`、`DocumentAccess`、`Category` | 文档元数据、分类、分享、收藏、访问记录 |
| MongoDB | `DocumentContent`、`DocumentContentRepository`、`AutoSaveHistory` | Markdown 正文、内容摘要、版本号、自动保存快照 |
| RabbitMQ | `kb-file/config/RabbitMQConfig`、`TranscodeConsumer`、`StatisticsEventPublisher` | 文件转码异步任务、统计事件；当前文件转码使用 DirectExchange `transcode.exchange` + routing key `transcode` |
| Nacos + OpenFeign | `FileServiceFeignClient`、`RagFeignClient`、`KAGFeignClient`、`SearchIndexFeignClient`、`GraphFeignClient` | 服务注册发现和跨微服务调用；导入时调用文件服务，发布后调用 RAG/KAG/ES |
| Elasticsearch | `SearchIndexFeignClient` 对应 kb-search | 文档标题、摘要、部分正文和分类等搜索索引；源码异步写入/删除 |
| Neo4j/KAG | `KAGFeignClient`、`GraphFeignClient` | 文档实体/关系图谱构建、删除和脏节点清理 |
| 本地内存/线程池 | `CompletableFuture`、`ThreadPoolTaskExecutor`、`avatarCache` | 异步访问记录、索引触发、头像缓存；多实例下不能把它当持久队列 |
| 网关 | `kb-gateway` 路由与白名单 | `/api/document/**`、`/api/file/**` 转发；公开 `/share/**` 需白名单但仍由业务层校验 |

### 3.1 面试时的一句话主链路

> 用户导入或创建文档时，文件先由文件服务保存到 RustFS，文件元数据和文档元数据写 MySQL，Markdown 正文写 MongoDB；文档发布后通过线程池异步调用 RAG、KAG、ES 建索引。浏览行为异步落 MySQL，收藏在 MySQL 事务内维护计数；PDF 从 MongoDB 读取正文用 PDFBox 渲染，分享则把权限状态保存在 `kb_document_share`，公开访问由白名单接口进入但仍执行密码、过期和次数校验。

## 4. 文章与当前源码的差异清单

1. 语雀第 13 章主要实现图片上传；当前 `kb-file` 已扩展视频流、缩略图、格式转换和断点续传。
2. 语雀第 14/16 章的示例表把状态写成 `draft/published` 字符串；当前 `Document` 使用 `Integer`：`0=草稿、1=发布、2=归档`。
3. 语雀示例中 `kb_document.content` 是正文列；当前源码用 `@TableField(exist=false)` 忽略该列，正文以 MongoDB 为准。
4. 语雀说明访问记录使用自增 ID；当前 `DocumentAccess` 使用 MP `ASSIGN_ID`。
5. 当前源码同时存在 `kb-file.FileController` 与 `kb-document.FileUploadController` 的 `/files/*`；它们属于不同微服务，网关前缀必须区分，否则容易误路由。
6. 当前 `UserFavorite` 继承 `BaseEntity` 但服务通过自定义 `physicalDelete` 取消收藏；阅读时要以 Service/Mapper 实际 SQL 为准，不只看实体上的 `@TableLogic`。
7. 分享密码当前用 MD5；这是文章实现点，生产环境应升级为带盐慢哈希并增加限流、防暴力破解。

## 5. 面试回答模板

- **为什么正文放 MongoDB？** 大文本读写与元数据查询职责不同，MySQL 保留可索引、可关联的元数据，Mongo 保存 Markdown 正文；用 `content_id`/`documentId` 关联，减少耦合。
- **为什么导入先解析再上传？** `MultipartFile` 输入流可能被 Feign 消费，先解析可保证后续仍能拿到文本；解析为空直接失败，避免产生无正文文档。
- **为什么最近访问异步？** 它是非核心体验数据，不能让记录失败拖慢详情接口；主链路返回后异步更新，失败记录日志并可补偿。
- **如何保证收藏不重复？** 数据库唯一键 + Service 幂等判断 + 事务内更新收藏计数。
- **公开分享安全吗？** 网关白名单只解决“能进来”，服务仍校验分享状态、过期时间、次数、密码；真正内容接口不直接暴露数据库 ID。
- **发布后索引失败怎么办？** 主数据已经提交，索引调用异步且失败不回滚发布；应记录失败任务并提供重试/重建接口，接受最终一致性。

## 6. 文件服务深挖：你真正需要掌握的实现

### 6.1 RustFS 后，MySQL 元数据是怎么来的？

不是 RustFS 返回一条“元数据记录”让 MySQL 保存，而是应用层自己组装：

```text
MultipartFile
  ├─ originalFilename / size / contentType
  ├─ SHA-256 流式计算 → fileHash
  ├─ 扩展名 + MIME → fileType（DOCUMENT/IMAGE/VIDEO/AUDIO）
  └─ fileHash + 日期 + 扩展名 → relativePath
             ↓
RustFileStorage.upload(inputStream, relativePath, size)
             ↓ 成功
FileServiceImpl.buildFileInfo(...)
             ↓
MySQL kb_file.file_info INSERT
```

`FileInfo` 主要保存：原始名、系统存储名、RustFS 相对路径、大小、文件类型、MIME、哈希、存储类型、上传者、访问级别、状态、下载次数。上传成功后，若是音视频，`MediaService.probeMediaInfo` 再调用 FFprobe 得到 `duration/resolution/bitrate`，回写同一条 MySQL 记录。

面试可以这样说：

> 对象存储负责保存字节，文件服务负责做校验、哈希、路径规划和元数据落库；因此 MySQL 元数据是业务层计算出来的，不是 RustFS 自动写入的。

### 6.2 去重和秒传

1. `calculateFileHash` 以 8KB 缓冲区流式计算 SHA-256，避免大文件一次性读入内存。
2. 用 `file_hash + status=1` 查 `kb_file`。
3. 找到已存在文件时直接返回旧 `FileInfoVO`，不重新上传对象，这就是秒传。
4. 没找到时按 `yyyy/MM/dd/hash前两位/hash.扩展名` 生成路径，再上传对象和插入元数据。

注意：秒传依赖“哈希相同即内容相同”的假设；生产环境还应加并发唯一约束、租户/权限校验，避免用户拿到无权访问的旧文件 URL。

### 6.3 断点续传

当前实现采用 S3 Multipart Upload：

```text
initResumableUpload
  → RustFS createMultipartUpload → uploadId
  → sessionId 映射 uploadId、路径、总大小、分片数
uploadChunk(sessionId, chunkIndex)
  → S3 uploadPart(partNumber=chunkIndex+1)
getUploadedChunks(sessionId)
  → S3 listParts 查询已上传分片
mergeChunks(sessionId)
  → completeMultipartUpload → 删除会话
```

实现上的两个面试风险：

- `uploadSessions` 是 `ConcurrentHashMap` 内存数据，服务重启或多实例时会丢失；生产应把会话放 Redis/数据库，或让客户端直接持有 S3 uploadId。
- `mergeChunks` 中 ETag 使用了简化值 `"etag-" + chunkIndex`，真实 S3 必须使用上传分片时返回的 ETag，否则合并可能失败；这是当前源码需要改进的点。

### 6.4 删除顺序、孤儿对象和补偿任务

当前代码的顺序是：

```text
查 MySQL FileInfo
  → status=0（业务软删除）并 updateById
  → 尝试 storage.delete(filePath)
```

这不是跨系统原子事务，因此会出现两类不一致：

| 情况 | 结果 | 处理思路 |
|---|---|---|
| MySQL 已标记删除，RustFS 删除失败 | 孤儿对象：用户不可见但占空间 | 记录失败任务，定时扫描 `status=0` 且对象仍存在的记录重试删除 |
| RustFS 已删除，MySQL 更新失败 | 元数据指向不存在对象 | 事务回滚只能回滚 MySQL，不能恢复对象；需要操作日志/补偿，或采用“删除中”状态 |

更稳妥的生产方案是：数据库事务只写 `deleting`/`deleted` 状态和待处理事件，后台任务幂等删除对象，成功后再标记完成；定时任务按状态重试，达到次数后告警。当前源码已经在失败时记录日志，但还没有完整的持久化补偿表，这是需要知道的设计缺口。

### 6.5 转码到底在干什么？

转码不是普通文档上传必经步骤，只针对音视频：

```text
POST /files/convert/{fileId}?targetFormat=hls
  → 校验文件是音视频
  → RabbitTemplate 发 TranscodeMessage(fileId,targetFormat)
  → file_info.transcode_status=PENDING
  → TranscodeConsumer 消费消息
  → FFmpeg 生成 HLS master.m3u8 + *.ts 分片/缩略图
  → 更新转码状态和播放地址
```

目的不是“改变文件内容让它能保存”，而是把 MP4 等源文件转换成浏览器可边下边播的 HLS 多码流/分片格式，并提取媒体信息。RabbitMQ 让 FFmpeg 这种耗时任务脱离 HTTP 请求线程；失败可以重试，前端通过 `transcodeStatus` 查询进度。

## 7. 学习取舍建议（针对主线）

| 部分 | 建议 | 最低掌握线 |
|---|---|---|
| 1.3 导入/上传逻辑 | **不能完全跳过，但不用抠每行代码** | `multipart`、先解析再上传、Feign 调文件服务、外链图片后端转存、导入默认草稿、失败降级 |
| 1.4 分类/用户 | 可以暂时略读 | 分类树移动防环；用户角色/直接权限；状态和权限边界 |
| 1.5 收藏/最近访问 | 主线可跳过实现细节 | 收藏唯一键+事务；最近访问异步且失败不影响浏览 |
| 1.6 PDF/分享 | PDF 渲染细节可跳过；分享安全不能完全跳过 | PDF 是 Mongo→PDFBox→字节流；分享要记住 shareId、密码、过期、次数、公开白名单 |

结论：如果你的目标是先吃透“文档主线”，优先级是 **文件上传基础链路 > 文档 MySQL/Mongo 双存储 > 导入草稿 > 发布后异步索引**。分类、收藏、PDF 排版可以后置；分享的权限校验保留面试级理解即可。
