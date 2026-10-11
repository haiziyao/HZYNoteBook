# RBAC：用户拿角色，角色拿权限

RBAC 全称是 **Role-Based Access Control**，中文叫基于角色的访问控制。

先记一句话：

> 不直接给每个用户配置权限，而是先把权限交给角色，再把角色分给用户。

## 一、先看一个真实例子

公司有三名员工：

```text
张三：普通员工
李四：财务人员
王五：财务主管
```

系统有这些操作：

```text
报销申请
报销查看
报销审核
报销付款
```

如果逐个给人配权限，人一多会很麻烦。RBAC 的做法是先准备角色：

```text
员工角色：报销申请、报销查看
财务角色：报销查看、报销付款
财务主管角色：报销查看、报销审核、报销付款
```

再给用户角色：

```text
张三 → 员工角色
李四 → 财务角色
王五 → 财务主管角色
```

于是李四能够付款，不是因为系统直接给李四配置了付款权限，而是因为：

```text
李四拥有财务角色
财务角色拥有报销付款权限
所以李四拥有报销付款权限
```

## 二、为什么经典设计是五张表

RBAC 是一种思想，不强制规定必须有固定数量的表。五张表只是最经典、最容易说清楚的设计。

但经典关系中有三种东西：

```text
用户
角色
权限
```

还存在两个多对多关系：

```text
一个用户可以有多个角色，一个角色也可以给多个用户
一个角色可以有多个权限，一个权限也可以给多个角色
```

所以常用五张表：

```text
sys_user              用户
sys_role              角色
sys_permission        权限
sys_user_role         用户和角色的关系
sys_role_permission   角色和权限的关系
```

项目里经常用 `sys_menu` 同时保存菜单和权限标识，此时 `sys_permission` 就是 `sys_menu`，`sys_role_permission` 就可能叫 `sys_role_menu`。

## 三、先看五张表中的实际数据

### 1. 用户表

| id | username |
| ---: | --- |
| 1 | zhangsan |
| 2 | lisi |
| 3 | wangwu |

### 2. 角色表

| id | role_name | role_key |
| ---: | --- | --- |
| 10 | 员工 | employee |
| 20 | 财务 | finance |
| 30 | 财务主管 | finance_manager |

### 3. 权限表

| id | permission_name | permission_key |
| ---: | --- | --- |
| 101 | 报销申请 | expense:create |
| 102 | 报销查看 | expense:read |
| 103 | 报销审核 | expense:approve |
| 104 | 报销付款 | expense:pay |

`permission_key` 就是代码中常见的权限字符串，例如：

```java
@PreAuthorize("hasAuthority('expense:pay')")
```

### 4. 用户角色关系表

| user_id | role_id |
| ---: | ---: |
| 1 | 10 |
| 2 | 20 |
| 3 | 30 |

把每一行读成中文：

```text
用户 1 张三拥有角色 10 员工
用户 2 李四拥有角色 20 财务
用户 3 王五拥有角色 30 财务主管
```

### 5. 角色权限关系表

| role_id | permission_id |
| ---: | ---: |
| 20 | 102 |
| 20 | 104 |
| 30 | 102 |
| 30 | 103 |
| 30 | 104 |

把前两行读成中文：

```text
角色 20 财务拥有权限 102 报销查看
角色 20 财务拥有权限 104 报销付款
```

## 四、五张表到底怎么连起来

以查询李四的权限为例：

```text
sys_user
李四，user_id = 2
        │
        │ sys_user_role.user_id
        ▼
sys_user_role
user_id = 2，role_id = 20
        │
        │ sys_role.id
        ▼
sys_role
role_id = 20，财务角色
        │
        │ sys_role_permission.role_id
        ▼
sys_role_permission
role_id = 20，permission_id = 102、104
        │
        │ sys_permission.id
        ▼
sys_permission
expense:read、expense:pay
```

整条链用一句话表达：

```text
李四 → 财务角色 → 报销查看、报销付款权限
```

两张关系表不是多余的，它们专门保存“谁和谁有关系”。

## 五、一次请求怎么判断

李四请求付款，接口要求：

```text
expense:pay
```

系统判断过程：

```text
1. 根据 user_id=2 找到李四的角色：finance。
2. 根据 finance 角色找到权限：expense:read、expense:pay。
3. 请求需要 expense:pay。
4. 李四的权限集合中存在 expense:pay。
5. 允许访问。
```

张三只有员工角色，没有 `expense:pay`，所以请求付款会被拒绝。

## 六、最简单的建表 SQL

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(64) NOT NULL
);

CREATE TABLE sys_role (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    role_name VARCHAR(64) NOT NULL,
    role_key VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE sys_permission (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    permission_name VARCHAR(64) NOT NULL,
    permission_key VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE sys_user_role (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE sys_role_permission (
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, permission_id)
);
```

关系表中的 ID 不应该设置 `AUTO_INCREMENT`。它们的作用是组合两个已有 ID，联合主键已经可以防止重复。

## 七、插入示例数据

```sql
INSERT INTO sys_user (id, username) VALUES
(1, 'zhangsan'),
(2, 'lisi'),
(3, 'wangwu');

INSERT INTO sys_role (id, role_name, role_key) VALUES
(10, '员工', 'employee'),
(20, '财务', 'finance'),
(30, '财务主管', 'finance_manager');

INSERT INTO sys_permission
    (id, permission_name, permission_key)
VALUES
    (101, '报销申请', 'expense:create'),
    (102, '报销查看', 'expense:read'),
    (103, '报销审核', 'expense:approve'),
    (104, '报销付款', 'expense:pay');

INSERT INTO sys_user_role (user_id, role_id) VALUES
(1, 10),
(2, 20),
(3, 30);

INSERT INTO sys_role_permission (role_id, permission_id) VALUES
(10, 101),
(10, 102),
(20, 102),
(20, 104),
(30, 102),
(30, 103),
(30, 104);
```

## 八、查询一个用户拥有的全部权限

```sql
SELECT DISTINCT p.permission_key
FROM sys_user u
JOIN sys_user_role ur
  ON u.id = ur.user_id
JOIN sys_role r
  ON ur.role_id = r.id
JOIN sys_role_permission rp
  ON r.id = rp.role_id
JOIN sys_permission p
  ON rp.permission_id = p.id
WHERE u.id = 2;
```

四次 `JOIN` 是因为从用户走到权限，中间必须依次经过四条连接：

```text
用户 → 用户角色关系 → 角色 → 角色权限关系 → 权限
```

使用 `INNER JOIN` 是因为这里只需要“确实存在完整授权链”的数据。任意一环不存在，就不应该得到权限。

直接判断用户 2 是否有付款权限：

```sql
SELECT 1
FROM sys_user_role ur
JOIN sys_role_permission rp
  ON ur.role_id = rp.role_id
JOIN sys_permission p
  ON rp.permission_id = p.id
WHERE ur.user_id = 2
  AND p.permission_key = 'expense:pay'
LIMIT 1;
```

## 九、RBAC 适用场景

* 后台管理系统。
* 企业岗位权限。
* 电商运营、客服、财务等功能权限。
* Spring Security 中按角色或权限字符串鉴权。

RBAC 擅长解决“大量用户拥有相似岗位权限”的问题。

## 十、高级扩展：RBAC0 到 RBAC3

先掌握上面的基础版本，再记扩展：

```text
RBAC0：用户、角色、权限以及两组多对多关系。
RBAC1：角色可以继承，例如主管继承员工权限。
RBAC2：增加约束，例如互斥角色、人数限制。
RBAC3：RBAC1 + RBAC2，同时支持继承和约束。
```

真实项目还常加：

* 角色和权限的状态字段。
* 租户或部门数据范围。
* 超级管理员直接放行。
* 权限缓存以及角色变更后的缓存失效。

## 十一、面试时怎么说

> RBAC 的核心是用户不直接拥有权限，而是用户拥有角色，角色拥有权限。经典设计有用户、角色、权限三张主体表，再用用户角色表和角色权限表解决两个多对多关系。查询权限时沿着“用户到角色，再从角色到权限”的链路查询，得到权限字符串后完成接口鉴权。
