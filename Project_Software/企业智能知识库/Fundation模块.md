

### Config

#### RedisCacheConfig:
``` java

// 放在类上
@EnableCaching

// 配置缓存器
@Bean
public RedisCacheManager cacheManager(
        RedisConnectionFactory connectionFactory) {
    // 配置 Redis 缓存
}


// 使用

@Cacheable(value = "userCache", key = "#userId")
public User getUser(Long userId) {
    return userMapper.selectById(userId);
}

@CacheEvict(value = "userCache", key = "#userId")
public void updateUser(Long userId) {
    // 修改用户
}
```

#### RabbitMQConfig

```java
// 复习一下6种工作模式

// simple

// work

// fanout

// direct

// topic

	//grpc
```

#### WebScoketConfig(不太好理解,先挂着了)

```java

// 这里主要是Spring中WebSocket的应用,一定要好好学

// 类注解: 开启开启 Spring 的 WebSocket + STOMP 消息代理功能。
@EnableWebSocketMessageBroker
// 实现这个接口  WebSocketMessageBrokerConfigurer

@Override
public void configureMessageBroker(
        MessageBrokerRegistry registry) {
    registry.enableSimpleBroker("/topic", "/queue");
    registry.setApplicationDestinationPrefixes("/app");
    registry.setUserDestinationPrefix("/user/");
}

@Override
public void registerStompEndpoints(
        StompEndpointRegistry registry) {
    registry.addEndpoint("/ws/notification")
            .setAllowedOriginPatterns("*")
            .setHandshakeHandler(new CustomHandshakeHandler())
            .addInterceptors(webSocketHandshakeInterceptor)
            .withSockJS();
}
```


#### BaseEntity

```java
// 主键注解
@TableId(type = IdType.INPUT)  //INPUT主键值由程序自己传入，MyBatis-Plus 不自动生成。
private Long id;


// 字段create-at, 在新增的时候自动填充
@TableField(
    value = "created_at",
    fill = FieldFill.INSERT
)
private LocalDateTime createdAt;

// 新增和更新的时候填充
@TableField(
    value = "updated_at",
    fill = FieldFill.INSERT_UPDATE
)
private LocalDateTime updatedAt;

// 逻辑字段: 开启逻辑删除
@TableLogic
private Integer deleted;
/*  搭配注解使用
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0

*/

```


#### SystemConfigCache

```java
// 这里有个做法
// 对于基建配置,放在nacos
// 对于其他一些配置放在redis中,各个分布式项目都可以读取
// 并支持 CURD
```


#### OperationLogAspect

```java
// 这个注解之前没见过类似用法,主要就是对标注这个注解的都提供AOP
@Around("@annotation(operationLog)")

@Before     // 方法执行前
@After      // 方法结束后，无论成功失败
@AfterReturning // 成功返回后
@AfterThrowing  // 抛异常后
@Around     // 执行前后都能控制，功能最强

//这里就是用了一个注解对 对应方法进行自动 日志增强,还会发到对应mq的交换机


// tips: 注解本身实际上是个接口变种,可以在方法参数中写上注解类,通过自动传入的对象拿到注解参数
```


#### JwtAuthenticationFilter

```java

// 继承基类
public class JwtAuthenticationFilter extends OncePerRequestFilter{}

// /usr/api/* 这样的风格匹配器,我竟然之前没用过
private final AntPathMatcher pathMatcher = new AntPathMatcher();
```


#### ReviewNotificationListener

```java
// 这个类里有设计到消息幂等性的知识(尽管这里没设计)


// 让当前方法成为 RabbitMQ 的消费者，持续监听指定队列；队列中有消息时，Spring 自动调用这个方法。
@RabbitListener(queues = "kb.notification.review.queue")
public void handleReviewEvent(ReviewEventDTO event)


// 在这里顺便讲讲openFeign
@EnableFeignClients(
    basePackages = "com.knowledge.base.foundation.feign"
)// 放在启动类

@Resource  
private UserAuthFeignClient userAuthFeignClient;

// 进入类内部

@FeignClient(
        name = "kb-user-auth",
        url = "${kb-user-auth.url:#{null}}",
        path = "/auth",
        fallbackFactory = UserAuthFeignFallbackFactory.class
)
public interface UserAuthFeignClient{

@PostMapping("/validate")  // POST /auth/validate
Result<TokenValidateVO> validateToken(
        @RequestHeader("Authorization") String authorization,
        @RequestParam(value = "token", required = false) String token);
}
```



#### SpringSecurity

```java

// 开启spring security 的 web 安全功能
@EnableWebSecurity

// 配置安全链
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {}
```