# ABAC：把权限写成一组条件

ABAC 全称是 **Attribute-Based Access Control**，中文叫基于属性的访问控制。

先记一句话：

> 收集用户、资源、操作和当前环境的属性，然后像执行 `if` 一样判断条件。

## 一、先看一个真实例子

公司有一份财务文档。系统规定：

```text
如果：
1. 用户和文档属于同一个部门；
2. 用户等级不低于文档要求等级；
3. 当前时间处于 09:00 到 18:00；
4. 用户执行的是 READ；
那么允许查看。
```

把它写成程序就是：

```java
if (user.departmentId == document.departmentId
        && user.level >= document.requiredLevel
        && currentTime在工作时间内
        && action.equals("READ")) {
    允许访问;
}
```

这就是 ABAC。它的核心不是某一张表，而是“用属性组成判断条件”。

## 二、什么叫属性

属性就是描述一个东西的字段。

```text
用户属性：部门、职位、等级、地区
资源属性：所属部门、保密等级、资源类型、创建人
操作属性：READ、UPDATE、DELETE、EXPORT
环境属性：当前时间、IP 地址、设备、所在网络
```

一次判断可以同时使用这些属性。

## 三、最简单需要保存哪几类信息

ABAC 是一种思想，不强制规定必须有固定数量的表。

在这个例子里，持久数据可以直接放在两张业务表中：

```text
sys_user：保存用户部门和等级
document：保存文档部门和要求等级
```

当前时间来自服务器，请求操作来自接口，不一定需要存进数据库。

### 1. 用户数据

| id | username | department_id | user_level |
| ---: | --- | ---: | ---: |
| 1 | zhangsan | 10 | 3 |
| 2 | lisi | 20 | 2 |

逐行翻译：

```text
张三属于部门 10，等级是 3
李四属于部门 20，等级是 2
```

### 2. 文档数据

| id | document_name | department_id | required_level |
| ---: | --- | ---: | ---: |
| 1001 | 财务预算 | 10 | 2 |

这一行表示：

```text
财务预算属于部门 10，至少需要等级 2 才能查看。
```

## 四、这些信息是怎么关联的

ABAC 不一定需要中间表。系统在请求到来时把各处的属性拿到一起比较：

```text
当前用户                       当前文档
department_id = 10   对比      department_id = 10
user_level = 3       对比      required_level = 2

当前请求                       当前环境
action = READ         对比      time = 10:30
```

然后把全部结果放进一条规则：

```text
同部门 AND 等级足够 AND 是读取操作 AND 在工作时间
```

全部成立才允许。

## 五、一次请求怎么判断

张三在上午 10:30 请求查看文档 1001：

```text
张三部门 10 = 文档部门 10             成立
张三等级 3 >= 文档要求等级 2           成立
操作是 READ                            成立
10:30 在 09:00 到 18:00 之间           成立

四个条件全部成立，所以允许。
```

李四请求同一文档：

```text
李四部门 20 != 文档部门 10             不成立

只要必要条件有一个不成立，就拒绝。
```

## 六、最简单的建表 SQL

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(64) NOT NULL,
    department_id BIGINT NOT NULL,
    user_level INT NOT NULL
);

CREATE TABLE document (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    document_name VARCHAR(100) NOT NULL,
    department_id BIGINT NOT NULL,
    required_level INT NOT NULL
);
```

## 七、插入示例数据

```sql
INSERT INTO sys_user
    (id, username, department_id, user_level)
VALUES
    (1, 'zhangsan', 10, 3),
    (2, 'lisi', 20, 2);

INSERT INTO document
    (id, document_name, department_id, required_level)
VALUES
    (1001, '财务预算', 10, 2);
```

## 八、最核心的查询 SQL

假设接口本身就是 READ，并且 Java 已经判断当前处于工作时间，那么数据库只比较用户和文档属性：

```sql
SELECT 1
FROM sys_user u
JOIN document d ON d.id = 1001
WHERE u.id = 1
  AND u.department_id = d.department_id
  AND u.user_level >= d.required_level
LIMIT 1;
```

如果想把工作时间也写进 MySQL 示例：

```sql
SELECT 1
FROM sys_user u
JOIN document d ON d.id = 1001
WHERE u.id = 1
  AND u.department_id = d.department_id
  AND u.user_level >= d.required_level
  AND CURRENT_TIME BETWEEN '09:00:00' AND '18:00:00'
LIMIT 1;
```

真实项目通常在业务代码或策略引擎中组合这些条件，不必强行把所有判断都塞进一条 SQL。

## 九、ABAC 适用场景

* 规则会用到部门、级别、地区、时间等多个条件。
* 同一角色在不同环境下权限不同。
* 多租户系统要求用户和资源的 `tenant_id` 一致。
* 数据权限需要限制“只能看本部门、本人或本地区”。
* 零信任场景还要检查设备和网络风险。

## 十、高级扩展：把规则本身也保存起来

基础学习中，规则直接写在 Java 代码里最容易理解。

当规则非常多、需要动态修改时，可以增加策略表：

```text
policy_rule
```

例如保存：

```text
资源类型：DOCUMENT
操作：READ
条件：user.departmentId == resource.departmentId
      && user.level >= resource.requiredLevel
```

然后交给策略引擎执行。但这会涉及表达式解析、规则冲突和安全校验，复杂度明显更高。

不要把“必须有一张万能属性表”当成 ABAC。属性完全可以就是现有业务表里的字段。

## 十一、面试时怎么说

> ABAC 是根据属性组合条件来决定权限。属性可以来自用户、资源、操作和环境，例如同部门、用户等级足够并且处于工作时间才允许读取。它不规定固定表数，简单实现可以直接使用用户表和资源表中的字段，在请求到来时像执行一组 if 条件一样判断。

