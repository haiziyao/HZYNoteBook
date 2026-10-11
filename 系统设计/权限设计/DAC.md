# DAC：东西的主人决定给谁使用

DAC 全称是 **Discretionary Access Control**，中文叫自主访问控制。

先只记一句话：

> 每个资源都有主人，主人可以自己使用，也可以把某种权限授予别人。

“自主”两个字指的就是：资源所有者可以自主决定把权限给谁。

## 一、先看一个真实例子

张三创建了一篇文档：

```text
文档 1001：《项目计划书》
主人：张三
```

张三想让李四帮忙修改，于是进行了授权：

```text
张三把文档 1001 的 UPDATE 权限授给李四
```

现在：

```text
张三是主人，所以可以操作自己的文档
李四不是主人，但张三给了他 UPDATE，所以李四可以修改
王五既不是主人，也没有被授权，所以王五不能修改
```

这就是 DAC。

## 二、它和 ACL 到底有什么区别

ACL 只强调资源旁边有一张名单：

```text
李四可以修改文档 1001
```

DAC 还要回答：

```text
这篇文档是谁的？
这条权限是谁授予李四的？
授予者有没有资格授权？
```

因此可以先这样理解：

> DAC = 带有“资源主人和授权行为”的 ACL。

## 三、最简单需要保存哪几类信息

DAC 是一种思想，不强制规定必须有固定数量的表。

只保护文档时，最简单可以保存三类信息：

```text
1. 系统里有谁
2. 文档是什么、主人是谁
3. 主人把什么操作授给了谁
```

可以使用三张表：

```text
sys_user
document
document_grant
```

### 1. 用户表

| id | username |
| ---: | --- |
| 1 | zhangsan |
| 2 | lisi |
| 3 | wangwu |

### 2. 文档表

| id | document_name | owner_id |
| ---: | --- | ---: |
| 1001 | 项目计划书 | 1 |

这一行翻译成人话：

```text
owner_id=1
= 用户 1 张三是文档 1001 的主人
```

### 3. 授权记录表

| document_id | grantor_id | grantee_id | action |
| ---: | ---: | ---: | --- |
| 1001 | 1 | 2 | READ |
| 1001 | 1 | 2 | UPDATE |

把每一行读成中文：

```text
文档 1001 + 授权人 1 + 被授权人 2 + READ
= 张三允许李四查看项目计划书

文档 1001 + 授权人 1 + 被授权人 2 + UPDATE
= 张三允许李四修改项目计划书
```

`grantor_id` 是给出权限的人，`grantee_id` 是得到权限的人。

## 四、三张表是怎么连起来的

```text
sys_user.id = 1（张三）
        │
        │ document.owner_id
        ▼
document.id = 1001（项目计划书）
        │
        │ document_grant.document_id
        ▼
document_grant
        │
        │ grantee_id = 2
        ▼
sys_user.id = 2（李四）
```

关系可以读成一句完整的话：

```text
文档 1001 属于张三，张三把它的 UPDATE 权限给了李四。
```

## 五、一次请求怎么判断

李四请求修改文档 1001：

```text
userId = 2
documentId = 1001
action = UPDATE
```

系统分两步判断：

```text
第一步：用户 2 是不是文档主人？
不是，因为 owner_id 是 1。

第二步：主人是否给用户 2 授过 UPDATE？
是，document_grant 中存在这条记录。

最终：允许。
```

王五请求修改时：

```text
不是主人
+ 没有 UPDATE 授权
= 拒绝
```

最简单的判断公式是：

```text
是资源主人 OR 得到过相应授权 = 允许
否则 = 拒绝
```

## 六、最简单的建表 SQL

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(64) NOT NULL
);

CREATE TABLE document (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    document_name VARCHAR(100) NOT NULL,
    owner_id BIGINT NOT NULL
);

CREATE TABLE document_grant (
    document_id BIGINT NOT NULL,
    grantor_id BIGINT NOT NULL,
    grantee_id BIGINT NOT NULL,
    action VARCHAR(50) NOT NULL,
    PRIMARY KEY (document_id, grantee_id, action)
);
```

## 七、插入示例数据

```sql
INSERT INTO sys_user (id, username) VALUES
(1, 'zhangsan'),
(2, 'lisi'),
(3, 'wangwu');

INSERT INTO document (id, document_name, owner_id)
VALUES (1001, '项目计划书', 1);

INSERT INTO document_grant
    (document_id, grantor_id, grantee_id, action)
VALUES
    (1001, 1, 2, 'READ'),
    (1001, 1, 2, 'UPDATE');
```

## 八、最核心的查询 SQL

判断用户 2 能否修改文档 1001：

```sql
SELECT 1
FROM document d
WHERE d.id = 1001
  AND (
      d.owner_id = 2
      OR EXISTS (
          SELECT 1
          FROM document_grant dg
          WHERE dg.document_id = d.id
            AND dg.grantee_id = 2
            AND dg.action = 'UPDATE'
      )
  )
LIMIT 1;
```

这段 SQL 不要整段硬背，把它读成人话：

```text
找到文档 1001，并且满足下面任意一个条件：
1. owner_id 是当前用户；
2. 授权表里存在“当前用户拥有 UPDATE”的记录。
```

## 九、DAC 适用场景

* 网盘文件的创建者分享文件。
* 在线文档的所有者邀请协作者。
* Linux 文件所有者设置读、写、执行权限。
* 项目创建者把项目管理权交给其他成员。

DAC 适合“资源有明确主人，而且主人可以决定分享”的场景。

## 十、后续可以增加的能力

基础版本先理解“主人授权”即可。真实系统还可能增加：

* `expires_at`：授权在什么时间过期。
* `can_grant`：得到权限的人能否继续转授。
* 撤销授权：主人删除 `document_grant` 记录。
* 转移所有权：修改 `document.owner_id`。
* 授权审计：记录授权时间和操作人。

转授会产生一个新问题：李四把权限给王五后，张三撤销李四的权限，王五的权限要不要一起消失？这是 DAC 的高级授权链问题，不影响基础模型的理解。

## 十一、面试时怎么说

> DAC 是由资源所有者自主决定权限的模型。资源表需要记录 owner_id，授权表记录资源、授权人、被授权人和操作。请求到来时，先判断当前用户是不是资源主人；如果不是，再检查是否存在相应授权。它常用于文件分享和在线文档协作。

