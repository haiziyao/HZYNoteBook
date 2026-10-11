# 文档对应章节：网关统一响应与异常封装

本章以项目文章《增加网关服务对接口返回值和异常做统一封装》为主，源码只用于确认实现位置。

## 文章定义的整体链路

```text
客户端请求
→ Spring Cloud Gateway
→ GlobalLogFilter 记录请求
→ 路由到后端微服务
→ Controller / Service
→ kb-common GlobalExceptionHandler 处理业务异常
→ UnifiedResponseFilter 统一响应体
→ CorsResponseHeaderFilter 去重 CORS 头
→ GlobalLogFilter 记录响应和耗时
→ 客户端
```

## 本次先掌握的 5 个核心类

### `Result<T>`

统一返回结构：`code`、`message`、`data`、`timestamp`。静态工厂方法统一生成成功、失败和布尔状态响应，前端只需按一种格式解析。

### `ResultCode`

集中管理 HTTP/业务响应码，避免 Controller 中散落数字。新增业务错误时扩展枚举即可。

### `BusinessException`

业务层主动抛出的运行时异常，携带业务码和业务消息。Service 只表达“业务失败”，不负责拼装 HTTP 响应。

### `GlobalExceptionHandler`

位于 `kb-common`，捕获业务服务中的异常并转换为 `Result`。文章特别设计了 `INNER-REQUEST` 判断：服务间 Feign 调用和外部客户端调用可以采用不同异常处理策略。

### `FeignInterceptorConfig`

为 Feign 发出的服务间请求自动增加：

```text
INNER-REQUEST: true
```

下游 `GlobalExceptionHandler` 可以据此识别内部调用，而不是依赖客户端手工传 Header。

## 网关侧 5 个实现点

- `GlobalLogFilter`：记录方法、URI、远程地址、状态码和耗时。
- `UnifiedResponseFilter`：装饰响应体，仅处理 JSON；已包含 `code/message` 的响应不重复包装，否则包装为 `Result`。
- `GatewayGlobalExceptionHandler`：处理网关连接超时、连接失败和响应状态异常，分别返回 504、502 等统一错误。
- `GatewayWebFluxConfig`：将 WebFlux 最大内存缓冲配置为 16MB，保证响应体拦截处理可用。
- `CorsResponseHeaderFilter`：删除下游重复 CORS 头，避免浏览器报重复响应头错误。

## 面试重点

统一响应不只是为了“好维护”，还为了固定前端契约、隐藏内部异常细节、区分内部/外部调用，并在网关集中记录链路日志。

## 文章中的注意点

- 网关使用 WebFlux，响应体是异步数据流，不能按传统 Servlet 方式直接读取。
- `UnifiedResponseFilter` 需要释放原始 `DataBuffer`，并重新设置 `Content-Length`。
- JSON 流式响应、文件下载和 WebSocket 不应被普通 JSON 包装逻辑误处理。
- CORS 应统一由网关管理；文章明确提醒不要让 `CorsWebFilter` 与 `globalcors` 重复配置。
- 当前源码中的 `GatewayConfig.AuthFilter` 仍是“放行所有请求”的占位实现；本章文章重点不是把它当成完整鉴权方案。

