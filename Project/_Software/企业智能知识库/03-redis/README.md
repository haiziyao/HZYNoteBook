# Redis 记录：配置缓存与热点数据

## 系统配置缓存

关键类：`kb-common/config/SystemConfigCache`、`kb-foundation/service/impl/SystemConfigServiceImpl`。

```text
Redis Hash
key: kb:system:config
field: 配置键，例如 system.requireApproval
value: 配置值字符串
```

读取：业务服务优先从 `SystemConfigCache` 取值；启动时 `ApplicationReadyEvent` 从 MySQL 全量加载；修改配置时同步更新 MySQL 和 Redis。

## 配置用途

- `system.requireApproval`：决定发布直接成功还是进入审核。
- `file.upload.max.size`：文件大小限制。
- `file.upload.allowed.types`：文件类型白名单。
- `websocket.enabled`：是否启用实时推送。

## 设计风险

- 若事务提交前就刷新 Redis，MySQL 回滚会造成短暂脏缓存。
- `loadAll` 先删 Hash 再批量写入，多实例并发刷新时可能出现短暂空缓存。
- 缓存不是最终数据源，Redis 丢失后必须能从 MySQL 重建。

## 面试表达

> 配置采用 Cache-Aside 的变体：MySQL 是持久化真源，Redis Hash 提供低延迟读取；启动加载和修改同步保证常态一致，异常情况下通过重载恢复。

