# 语雀主线批次二：第 11～14 章

> 前 10 章已经完成，本文件只记录后续批次，不重复前 10 章。内容以语雀文章为主，本地源码用于确认当前版本的实际类名和接口差异。

## 批次范围

```text
11 网关服务统一封装返回值
12 登录和认证（一）
12 登录和认证（二）
13 文件上传
14 文档查询、创建和删除
```

## 1. 第 11 章：网关统一封装

### 文章主旨

网关统一处理返回格式、异常、日志和 CORS，让业务服务专注业务逻辑，前端只适配一种 JSON 响应格式。

### 文章定义的组件

- `kb-common/result/Result`：`code/message/data/timestamp` 统一响应。
- `ResultCode`：集中管理 HTTP 与业务响应码。
- `BusinessException`：业务层抛出的可识别异常。
- `AssertUtil`：参数校验失败时抛 `BusinessException`。
- `GlobalExceptionHandler`：业务服务内异常转成 `Result`。
- `FeignInterceptorConfig`：服务间 Feign 请求增加 `INNER-REQUEST: true`。
- `GlobalLogFilter`：记录请求 URI、远程地址、响应状态和耗时。
- `UnifiedResponseFilter`：拦截 JSON 响应，已是 `code/message` 格式的不重复包装，否则包装成 `Result`。
- `GatewayGlobalExceptionHandler`：处理网关超时、连接失败等异常。
- `CorsResponseHeaderFilter`：去除下游重复 CORS 头。
- `GatewayWebFluxConfig`：WebFlux 响应缓冲上限 16MB。

### 关键线路

```text
客户端
→ Gateway Filter
→ 下游 Controller/Service
→ Common GlobalExceptionHandler
→ UnifiedResponseFilter
→ CORS 头去重
→ 客户端统一响应
```

### 面试重点

统一封装的价值不只是维护方便，还包括固定前端契约、隐藏内部异常细节、区分内部 Feign 调用与外部请求、集中记录链路日志。WebFlux 响应是异步数据流，包装响应时要处理 DataBuffer 释放、Content-Length 和流式/文件响应例外。

## 2. 第 12 章：登录和认证（一）（二）

### 文章主旨

`kb-user-auth` 是安全门户，负责用户名密码登录、JWT 生成与验证、权限控制、用户管理；第二篇主要讲前端如何保存 Token、请求拦截和路由守卫。

### JWT 设计

- HS256（HMAC SHA-256）签名。
- Access Token：文章示例为短期有效（约 2 小时）。
- Refresh Token：文章示例为长期有效（约 7 天）。
- `subject` 保存用户 ID；同时可带用户名、头像、Token 类型、签发/过期时间和 issuer。

### 后端线路

```text
POST /auth/login
→ UserMapper 查询用户
→ 校验密码和状态
→ JwtTokenUtil 生成 Access/Refresh Token
→ 返回 UserVO + Token
```

后续请求：

```text
Authorization: Bearer <access-token>
→ 网关/服务解析并验签
→ UserContextUtil 使用 ThreadLocal 保存当前用户
→ Service 获取当前用户 ID 做权限判断
```

### 前端线路

```text
登录页提交
→ Axios 请求拦截器自动加 Token
→ 路由守卫判断登录状态
→ 过期时刷新 Token 或跳转登录页
```

### 面试重点

JWT 无状态，减少 Session 共享和认证服务频繁查询；代价是主动注销、权限即时变更和 Token 泄露后的吊销更难。Token 应放在安全位置，服务端不能盲目信任客户端传入的用户头。

## 3. 第 13 章：文件上传

### 文章主旨

`kb-file` 提供上传、下载、预览和断点续传，文件内容放 RustFS，文件元数据放 MySQL。

### 中间件与类

- RustFS：兼容 AWS S3 API 的对象存储。
- `FileStorage`：存储抽象接口。
- `RustFileStorage`：RustFS 实现。
- `FileStorageFactory`：按 `StorageType` 选择存储实现。
- `FileService` / `FileServiceImpl`：校验、生成存储名、写元数据、调用存储层。
- `FileInfo`：保存原文件名、存储文件名、路径、大小、MIME、SHA-256、bucket 等元数据。

### 文件线路

```text
multipart 上传
→ 文件类型/大小校验
→ 计算 hash（可用于去重）
→ 生成存储名和对象路径
→ RustFS 写二进制
→ MySQL 写 kb_file 元数据
→ 返回文件 ID/访问地址
```

### 面试重点

对象存储和数据库分离，适合大文件和水平扩展；`FileStorage` 抽象让后续可替换 MinIO、S3 或其他实现。必须考虑对象写成功但 MySQL 失败、MySQL 成功但对象写失败的补偿和孤儿文件清理。

## 4. 第 14 章：文档查询、创建和删除

### 文章主旨

本章实现文档列表查询、创建、删除和状态管理，明确使用：

- MySQL：文档元数据；
- MongoDB：大文本正文；
- MyBatis-Plus：结构化查询和 CRUD；
- 雪花 ID：文档、正文关联记录等分布式唯一 ID；
- `UserContextUtil`：获取当前用户。

### 文档创建线路

```text
Controller 接收 DocumentDTO
→ 校验标题/分类/标签等参数
→ 雪花算法生成文档 ID
→ MySQL 保存元数据
→ MongoDB 保存正文
→ 返回文档 ID/VO
```

### 查询线路

```text
分页参数 + 关键词/分类筛选
→ MySQL 查询文档元数据
→ 按需读取 MongoDB 正文
→ 组装 DocumentVO
```

### 删除线路

文档删除首先作用于 MySQL 逻辑删除；正文、文件和搜索派生数据的清理需要同步或异步补偿，不能只删一张表就认为整个文档已清理。

### 面试重点

文档表是结构化真源，MongoDB 是正文存储；两库没有天然的统一事务。创建、更新、删除都要考虑部分成功、幂等和补偿。文档状态管理要与审核流程、版本、ES/RAG/KAG 派生任务衔接。

## 5. 本批次中间件矩阵

| 文章 | MySQL | MongoDB | Redis | RabbitMQ | RustFS/S3 | ES/Neo4j |
|---|---|---|---|---|---|---|
| 11 网关 | 仅下游使用 | - | 网关依赖可选 | - | - | - |
| 12 认证 | 用户、角色、权限 | - | Token/缓存相关能力 | - | - | - |
| 13 文件 | 文件元数据 | - | 配置/限制读取 | 可扩展异步处理 | 二进制对象 | - |
| 14 文档 | 元数据 | 正文 | 浏览量/热点 | 后续事件 | 文件附件可关联 | 发布后派生索引 |

## 6. 本批次面试检查点

1. 能说出 `Result`、`BusinessException`、`GlobalExceptionHandler`、网关 Filter 的职责边界。
2. 能解释 Access Token 与 Refresh Token 的生命周期差异。
3. 能完整描述文件“对象存储 + 元数据表”的两阶段失败场景。
4. 能完整描述文档“DTO → MySQL + MongoDB → VO”的创建和查询链路。
5. 能指出网关统一封装对 WebFlux 流式响应、CORS 和内部 Feign 请求的特殊处理。

