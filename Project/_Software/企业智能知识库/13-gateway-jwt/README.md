# 源码补充：网关启动与 JWT 请求链路

> 本文件是源码核对补充，不作为语雀/项目文章章节的主线。章节主线请先阅读 `14-gateway-response/README.md`。

本章只分析 4 个文件：`GatewayApplication`、`GatewayConfig`、`AuthGlobalFilter`、`JwtTokenUtil`。

## 1. GatewayApplication

文件：`kb-gateway/.../GatewayApplication.java`

- 解决问题：启动网关 Spring Boot 应用。
- 为什么在网关：网关是独立进程，不能加载数据库相关自动配置。
- 关键点：排除 `DataSourceAutoConfiguration`；只扫描 `gateway` 和 `common` 包，并排除公共模块里的 `CorsConfig`，避免 CORS 配置重复。
- 调用链：`main` → `SpringApplication.run` → 创建 Gateway、Filter、Route 等 Bean → 监听 8080。
- 面试表达：网关不连接业务数据库，启动时排除数据源自动配置，避免无意义的数据库初始化。
- 坑：`@ComponentScan` 范围过大可能把业务 Bean 带入网关；CORS 同时由多个 Filter 配置可能导致跨域和 WebSocket 握手异常。

## 2. GatewayConfig

文件：`kb-gateway/.../config/GatewayConfig.java`

- 解决问题：注册全局过滤器。
- 关键点：内部 `AuthFilter` 的 `filter` 当前直接放行所有请求，返回值为 `-100`。
- 依赖关系：实现 Spring Cloud Gateway 的 `GlobalFilter` 和 `Ordered`。
- 调用链：网关收到请求 → 按 order 执行过滤器 → 当前过滤器放行 → 下一个过滤器/路由。
- 面试表达：这是全局过滤器扩展点，但当前类里的认证逻辑还是占位实现，真正解析 JWT 的是 `AuthGlobalFilter`。
- 坑：项目同时存在一个“放行型” `GatewayConfig.AuthFilter` 和一个真正解析 JWT 的 `AuthGlobalFilter`，容易让维护者误以为认证已经在前者完成；应明确 Bean 职责和执行顺序。

## 3. AuthGlobalFilter

文件：`kb-gateway/.../filter/AuthGlobalFilter.java`

- 解决问题：在网关统一解析 JWT，并把用户 ID传给下游，避免每个服务重复解析。
- Token 来源：优先 `Authorization: Bearer <token>`，其次 Cookie `access_token`。
- 公开路径：登录、注册、文档、Actuator、WebSocket 握手等路径按 `shouldSkip` 放行。
- 调用链：

```text
请求进入网关
→ 判断是否公开接口
→ 提取 Header/Cookie Token
→ JwtTokenUtil 验签并解析 subject
→ 添加 X-User-Id 请求头
→ 转发下游服务
```

- order：`-200`，早于 `GlobalLogFilter` 的日志记录。
- 面试表达：网关本地完成 JWT 解析，把身份上下文通过请求头传递给下游，减少远程认证调用。
- 重要风险：当前 Token 缺失或解析失败时会继续放行，并不直接返回 401；因此必须确认下游安全层会拦截未认证请求。内部服务不能裸奔，否则客户端可能伪造 `X-User-Id`。

## 4. JwtTokenUtil

文件：`kb-common/.../utils/JwtTokenUtil.java`

- 解决问题：统一生成、签名、解析、验证和刷新 JWT。
- 依赖：`JwtConfig` 提供密钥、签发者和过期时间；JJWT 负责签名和验签。
- Token 内容：`subject=userId`，并附带 `userId`、`type=access`、用户名、头像、签发时间、过期时间、issuer。
- 调用链：用户登录 → `generateAccessToken` → 返回客户端 → 网关 `parseToken` → 验签 → 取 `subject` → 注入 `X-User-Id`。
- 面试表达：JWT 的签名保证内容未被篡改，过期时间控制有效期；服务无需保存 Session 即可校验身份。
- 坑：刷新 Token 时应校验 Token 类型，当前实现主要解析 subject；密钥必须在各服务保持一致且不能写入公开笔记；JWT 无法天然解决主动注销和权限即时变更。

## 本章结论

```text
服务启动时注册网关 Bean
→ 请求进入 AuthGlobalFilter
→ 本地验签 JWT
→ 写入 X-User-Id
→ 网关路由到下游服务
```

这里的“认证”和“身份传递”要区分：`AuthGlobalFilter` 负责解析和传递，是否拒绝未认证请求还要看下游安全配置。
