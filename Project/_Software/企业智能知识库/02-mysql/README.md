# MySQL 记录：结构化数据与事务边界

## 重要模块

- `kb-user-auth`：用户、角色、权限。
- `kb-document`：文档元数据、分类、标签关系、版本、评论/点赞关联。
- `kb-foundation`：系统配置、字典、通知、操作日志。
- `kb-file`：文件元数据。
- `kb-statistics`：统计数据。

## 不能忽略的设计

### BaseEntity

`@TableId(type = IdType.INPUT)` 表示 ID 由应用侧 `SnowflakeIdGenerator` 生成；`@TableField(fill = ...)` 交给 `MetaObjectHandler` 自动填充创建/更新时间和操作人；`@TableLogic` 开启逻辑删除。

### 事务边界

一个服务内的 MySQL 写操作可以使用 `@Transactional` 保证原子性；但 MySQL 与 MongoDB、ES、Neo4j、RabbitMQ 之间不是同一事务，采用异步、重试、补偿和幂等实现最终一致。

### 逻辑删除坑

MP 查询通常自动过滤 `deleted=1` 的记录；物理删除和跨表统计要特别确认是否需要显式包含已删除数据。数据库默认值、MP 全局配置和实体字段含义必须一致。

## 普通 CRUD

字典、标签、分类名称、评论查询等可以只掌握 Controller → Service → Mapper → MySQL 的标准链路；重点放在唯一索引、父子关系、计数器和事务。

## 点赞与评论的实际实现

当前项目的点赞不是 Redis 方案，而是 MySQL `tb_like` 记录：目标 ID、用户 ID、目标类型组成唯一性约束；点赞记录写入后同步递增文档或评论的 `like_count`，取消点赞则删除记录并递减计数。文档点赞还会发布统计事件到 RabbitMQ。

`reply_count` 是冗余计数，用于快速显示和分页，不是维持评论树结构的唯一条件；树结构主要依赖 `parent_id/root_id`。评论点赞代码存在“先查询再插入”的并发窗口，最终仍应依赖数据库唯一索引兜底。
