# RabbitMQ 记录：审核通知与操作日志

## 重要线路

```text
业务事件
→ Exchange
→ RoutingKey
→ Queue
→ Consumer
→ MySQL 持久化
→ WebSocket（如在线）
```

## 操作日志

`@OperationLog` 标注 Controller 方法；`OperationLogAspect` 在切面中采集操作人、模块、请求方法、URI、参数和结果，发布到 RabbitMQ。`OperationLogMQListener` 消费后直接用 `JdbcTemplate` 写入 `kb_operation_log`。

结论：这是 `kb-foundation` 内的消息消费者，不是独立日志微服务。

## 审核通知

审核提交、通过、驳回等事件先写通知表，再通过消息/推送发送给审核员或用户。在线用户即时收到 WebSocket；离线用户下次上线通过未读消息查询补偿。

## 必须关注

- 消息重复：消费者应使用业务 ID 或事件 ID幂等。
- 消息丢失：生产确认、持久化队列、失败重试/死信。
- 先落库后通知：避免通知已发但数据库没有记录。
- 不要把 RabbitMQ 当作数据库；它负责传输和削峰。

