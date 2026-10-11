# ACL：给资源贴一张访问名单

ACL 全称是 **Access Control List**，中文叫访问控制列表。

先不要记表名。只记住一句话：

> 每一个具体资源旁边，都贴着一张“谁可以对它做什么”的名单。

## 一、先看一个真实例子

系统里有三个人：

```text
用户 1：admin
用户 2：zhangsan
用户 3：lisi
```

系统里有一篇文档：

```text
文档 1001：《项目计划书》
```

文档旁边的名单是：

```text
张三可以查看
张三可以修改
李四可以查看
```

这张名单就是 ACL。

## 二、最简单需要保存哪三类信息

ACL 是一种思想，不强制规定必须有固定数量的表。它只是需要保存三类信息：

```text
1. 谁：用户
2. 什么东西：文档
3. 谁能对这个东西做什么：权限关系
```

如果系统目前只保护文档，最容易理解的设计就是三张表。

### 1. 用户表

| id | username |
| ---: | --- |
| 1 | admin |
| 2 | zhangsan |
| 3 | lisi |

它回答：系统里有谁？

### 2. 文档表

| id | document_name |
| ---: | --- |
| 1001 | 项目计划书 |
| 1002 | 财务报表 |

它回答：有哪些东西需要保护？

### 3. 文档访问名单

| document_id | user_id | action |
| ---: | ---: | --- |
| 1001 | 2 | READ |
| 1001 | 2 | UPDATE |
| 1001 | 3 | READ |

把每一行直接读成中文：

```text
文档 1001 + 用户 2 + READ
= 张三可以查看项目计划书

文档 1001 + 用户 2 + UPDATE
= 张三可以修改项目计划书

文档 1001 + 用户 3 + READ
= 李四可以查看项目计划书
```

这张关系表就是那张“访问名单”。

## 三、三张表是怎么连起来的

```text
sys_user.id = 2
       │
       │ document_acl.user_id
       ▼
document_acl
       ▲
       │ document_acl.document_id
       │
document.id = 1001
```

说成人话：

```text
user_id 告诉我们是谁
document_id 告诉我们是哪篇文档
action 告诉我们允许做什么
```

## 四、一次请求怎么判断

张三请求修改文档 1001：

```text
userId = 2
documentId = 1001
action = UPDATE
```

系统去名单中找：

```text
document_id=1001、user_id=2、action=UPDATE
```

找到了，所以允许。

如果张三请求删除文档：

```text
document_id=1001、user_id=2、action=DELETE
```

名单中没有这一行，所以拒绝。

最初学习 ACL 时，规则就这么简单：

```text
查到记录 = 允许
查不到记录 = 拒绝
```

## 五、最简单的建表 SQL

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(64) NOT NULL
);

CREATE TABLE document (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    document_name VARCHAR(100) NOT NULL
);

CREATE TABLE document_acl (
    document_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    action VARCHAR(50) NOT NULL,
    PRIMARY KEY (document_id, user_id, action)
);
```

联合主键表示同一个用户对同一篇文档的同一种操作不能重复配置。

## 六、插入示例数据

```sql
INSERT INTO sys_user (id, username) VALUES
(1, 'admin'),
(2, 'zhangsan'),
(3, 'lisi');

INSERT INTO document (id, document_name) VALUES
(1001, '项目计划书'),
(1002, '财务报表');

INSERT INTO document_acl (document_id, user_id, action) VALUES
(1001, 2, 'READ'),
(1001, 2, 'UPDATE'),
(1001, 3, 'READ');
```

## 七、最核心的查询

```sql
SELECT 1
FROM document_acl
WHERE document_id = #{documentId}
  AND user_id = #{userId}
  AND action = #{action}
LIMIT 1;
```

在 MySQL 中判断张三能否修改文档 1001：

```sql
SELECT 1
FROM document_acl
WHERE document_id = 1001
  AND user_id = 2
  AND action = 'UPDATE'
LIMIT 1;
```

返回一行就是允许，没有结果就是拒绝。

## 八、为什么有些设计又出现 `acl_resource`

上面的系统只有文档，所以直接使用 `document_acl` 很清楚。

如果以后还要保护：

```text
文件
订单
项目
报表
```

可能会出现很多表：

```text
document_acl
file_acl
order_acl
project_acl
```

为了统一，才增加通用资源表：

```text
acl_resource(id, resource_type, business_id)
acl_entry(resource_id, user_id, action)
```

例如：

| id | resource_type | business_id |
| ---: | --- | ---: |
| 1 | DOCUMENT | 1001 |
| 2 | FILE | 2001 |

`DOCUMENT + 1001` 仍然表示项目计划书，只是先给它发了一张 ACL 系统内部的“资源身份证”。

通用版本不是 ACL 的必要条件，只是系统资源类型很多时的工程优化。

## 九、ALLOW 和 DENY 是后续扩展

基础版本只有：有记录就允许。

如果需要明确禁止，可以增加 `effect`：

| document_id | user_id | action | effect |
| ---: | ---: | --- | --- |
| 1001 | 2 | UPDATE | ALLOW |
| 1001 | 3 | DELETE | DENY |

常见规则是：

```text
DENY 优先
其次看 ALLOW
都没有则默认拒绝
```

先理解基础名单，再学习冲突规则。

## 十、ACL 适用场景

* 网盘中把某一个文件分享给某个人。
* 在线文档为每篇文档配置协作者。
* 某一条订单额外授权给客服处理。
* 某个项目单独邀请成员。

ACL 擅长控制“这一个具体东西”。它不擅长管理大量通用岗位权限，那通常由 RBAC 处理。

## 十一、面试时怎么说

> ACL 是以具体资源为中心的访问控制。可以把它理解成资源旁边贴了一张名单，记录哪个用户能对该资源执行什么操作。最简单可以使用用户表、资源表和 ACL 关系表。请求到来时根据用户 ID、资源 ID 和 action 查询关系表，查到就允许，查不到就默认拒绝。
