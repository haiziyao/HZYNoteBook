# Nacos 记录：注册发现与配置中心

## 用途

- 服务启动后注册实例，如 `kb-document`、`kb-ai`。
- Feign 根据服务名发现地址，不把 IP 写死在业务代码中。
- 管理部分 Spring 配置，支持环境隔离和动态刷新。

## 调用示例

```text
RagFeignClient(name = kb-ai)
→ Nacos 找到 kb-ai 实例
→ LoadBalancer 选择实例
→ HTTP POST /rag/reindex/{docId}
```

## 面试坑点

- Nacos 是控制面，不是业务数据库。
- 注册成功不等于业务接口一定健康，还要看健康检查、超时和降级。
- 本地 profile 的数据库密码属于敏感配置，笔记中不记录具体值。

