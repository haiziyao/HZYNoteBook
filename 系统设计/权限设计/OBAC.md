# OBAC：看你和这个业务对象是什么关系

这里的 OBAC 指 **Object-Based Access Control**，中文可以理解为基于对象的访问控制。

先记一句话：

> 系统根据“用户和这个业务对象的关系”，再结合对象当前状态，决定用户能做什么。

## 一、先看一个真实例子

电商系统中有一张订单：

```text
订单 1001
买家：张三
卖家：李四
状态：PENDING_PAYMENT（待付款）
```

系统规定：

```text
待付款时，买家可以付款或取消订单；
待付款时，卖家可以查看，但不能替买家付款；
已发货后，买家不能再取消订单。
```

张三能取消订单，不是因为他拥有一个笼统的 `order:cancel` 权限，而是因为：

```text
张三和订单 1001 的关系是 BUYER
+ 订单当前状态是 PENDING_PAYMENT
+ BUYER 在这个状态下允许 CANCEL
= 张三可以取消订单
```

## 二、什么叫“对象”

对象就是系统中需要被操作的一笔具体业务数据，例如：

```text
某一张订单
某一个项目
某一份合同
某一个病例
某一个工单
```

OBAC 不只是问“你有没有订单管理权限”，还会问：

```text
你和这一张订单是什么关系？
这张订单现在处于什么状态？
```

## 三、最简单需要保存哪几类信息

OBAC 是一种思想，不强制规定必须有固定数量的表。

为了把通用关系展示清楚，可以使用四张表：

```text
sys_user                 用户
business_object          业务对象及其状态
object_relation          用户和对象的关系
object_relation_action   某种关系在某状态能执行什么
```

## 四、先看四张表中的实际数据

### 1. 用户表

| id | username |
| ---: | --- |
| 1 | zhangsan |
| 2 | lisi |
| 3 | wangwu |

### 2. 业务对象表

| id | object_type | business_id | object_state |
| ---: | --- | ---: | --- |
| 501 | ORDER | 1001 | PENDING_PAYMENT |

这一行表示：

```text
权限系统中的对象 501，代表业务订单 1001，
它当前处于待付款状态。
```

如果系统只控制订单，也可以直接在订单表中保存状态，不一定非要增加通用的 `business_object` 表。

### 3. 对象关系表

| object_id | user_id | relation_type |
| ---: | ---: | --- |
| 501 | 1 | BUYER |
| 501 | 2 | SELLER |
| 501 | 3 | CUSTOMER_SERVICE |

逐行翻译：

```text
张三是对象 501 的买家
李四是对象 501 的卖家
王五是对象 501 的客服
```

### 4. 关系操作规则表

| object_type | object_state | relation_type | action |
| --- | --- | --- | --- |
| ORDER | PENDING_PAYMENT | BUYER | PAY |
| ORDER | PENDING_PAYMENT | BUYER | CANCEL |
| ORDER | PENDING_PAYMENT | SELLER | VIEW |
| ORDER | SHIPPED | BUYER | CONFIRM_RECEIPT |

第二行翻译成人话：

```text
订单处于待付款状态时，订单买家可以取消。
```

## 五、四张表怎么连起来

```text
sys_user
张三，id = 1
        │
        │ object_relation.user_id
        ▼
object_relation
object_id = 501，relation = BUYER
        │
        │ business_object.id
        ▼
business_object
ORDER，state = PENDING_PAYMENT
        │
        │ 类型 + 状态 + 关系同时匹配
        ▼
object_relation_action
BUYER 可以 PAY、CANCEL
```

整条链就是：

```text
张三是订单 1001 的买家；订单正在待付款；
规则规定待付款订单的买家可以取消。
```

## 六、一次请求怎么判断

张三请求取消订单 1001：

```text
userId = 1
objectType = ORDER
businessId = 1001
action = CANCEL
```

系统依次判断：

```text
1. 找到代表订单 1001 的对象 501。
2. 查到订单状态是 PENDING_PAYMENT。
3. 查到张三与对象的关系是 BUYER。
4. 查询规则：BUYER 在 PENDING_PAYMENT 能否 CANCEL？能。
5. 允许。
```

订单发货后状态变成 `SHIPPED`。即使张三仍然是买家，由于该状态没有取消规则，也会被拒绝。

## 七、最简单的建表 SQL

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(64) NOT NULL
);

CREATE TABLE business_object (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    object_type VARCHAR(50) NOT NULL,
    business_id BIGINT NOT NULL,
    object_state VARCHAR(50) NOT NULL,
    UNIQUE KEY uk_object (object_type, business_id)
);

CREATE TABLE object_relation (
    object_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    relation_type VARCHAR(50) NOT NULL,
    PRIMARY KEY (object_id, user_id, relation_type)
);

CREATE TABLE object_relation_action (
    object_type VARCHAR(50) NOT NULL,
    object_state VARCHAR(50) NOT NULL,
    relation_type VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    PRIMARY KEY (
        object_type,
        object_state,
        relation_type,
        action
    )
);
```

## 八、插入示例数据

```sql
INSERT INTO sys_user (id, username) VALUES
(1, 'zhangsan'),
(2, 'lisi'),
(3, 'wangwu');

INSERT INTO business_object
    (id, object_type, business_id, object_state)
VALUES
    (501, 'ORDER', 1001, 'PENDING_PAYMENT');

INSERT INTO object_relation
    (object_id, user_id, relation_type)
VALUES
    (501, 1, 'BUYER'),
    (501, 2, 'SELLER'),
    (501, 3, 'CUSTOMER_SERVICE');

INSERT INTO object_relation_action
    (object_type, object_state, relation_type, action)
VALUES
    ('ORDER', 'PENDING_PAYMENT', 'BUYER', 'PAY'),
    ('ORDER', 'PENDING_PAYMENT', 'BUYER', 'CANCEL'),
    ('ORDER', 'PENDING_PAYMENT', 'SELLER', 'VIEW'),
    ('ORDER', 'SHIPPED', 'BUYER', 'CONFIRM_RECEIPT');
```

## 九、最核心的查询 SQL

判断用户 1 能否取消订单 1001：

```sql
SELECT 1
FROM business_object bo
JOIN object_relation rel
  ON bo.id = rel.object_id
JOIN object_relation_action rule
  ON bo.object_type = rule.object_type
 AND bo.object_state = rule.object_state
 AND rel.relation_type = rule.relation_type
WHERE bo.object_type = 'ORDER'
  AND bo.business_id = 1001
  AND rel.user_id = 1
  AND rule.action = 'CANCEL'
LIMIT 1;
```

把 SQL 翻译成人话：

```text
找到订单 1001，
找到用户 1 与该订单的关系，
再检查“订单类型 + 当前状态 + 用户关系”是否允许 CANCEL。
```

## 十、OBAC 和 ACL 有什么不同

ACL 可能直接保存：

```text
张三可以取消订单 1001
```

OBAC 保存的是：

```text
张三是订单 1001 的买家
规则规定：待付款订单的买家可以取消
```

ACL 保存最终权限；OBAC 保存业务关系，然后根据对象状态和规则推导权限。

## 十一、OBAC 适用场景

* 电商订单中的买家、卖家、客服。
* 项目中的所有者、成员、访客。
* 合同中的甲方、乙方、审批人。
* 医疗系统中的患者、主治医生、会诊医生。
* 社交系统中的作者、关注者、好友。

## 十二、高级扩展

在简单业务中，不需要四张通用表。例如订单本身已有 `buyer_id`、`seller_id`、`status`，完全可以直接查询订单表：

```sql
SELECT 1
FROM orders
WHERE id = 1001
  AND buyer_id = 1
  AND status = 'PENDING_PAYMENT'
LIMIT 1;
```

通用关系表适合对象类型多、关系种类多的系统，但会增加查询和维护成本。选择表结构时应先服从业务，而不是为了模型名强行抽象。

## 十三、面试时怎么说

> OBAC 是根据用户与具体业务对象的关系，并结合对象状态来判断权限。例如用户是订单买家，且订单处于待付款状态，规则才允许取消。可以保存对象表、用户对象关系表和关系状态操作规则表；简单系统也可以直接使用订单表里的 buyer_id 和 status 判断。

