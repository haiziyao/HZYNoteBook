# T-RBAC：既要有角色，也要正在处理任务

这里的 T-RBAC 指 **Task-Role-Based Access Control**，也就是把任务和角色结合起来的访问控制。

先记一句话：

> 只有“正在处理这个任务”还不够，用户还必须具备该步骤要求的角色。

## 一、先看一个真实例子

系统中有一笔金额很大的付款任务：

```text
任务 5001：支付报销单 1001
当前状态：PAYMENT
当前处理人：李四
```

系统规定：

```text
PAYMENT 状态下，只有 finance 角色才能执行 PAY。
```

李四是处理人，而且李四拥有财务角色：

```text
李四是任务 5001 的处理人
+ 李四拥有 finance 角色
+ PAYMENT 状态允许 finance 执行 PAY
= 允许付款
```

如果把任务错误地分给一个普通员工，他虽然是处理人，但没有财务角色，仍然不能付款。

## 二、为什么 TBAC 之后还需要角色

只使用 TBAC 时，判断可能是：

```text
是当前处理人，就能操作。
```

但有些任务非常敏感：

```text
普通员工不能因为任务被错误分配，就获得付款权限；
财务人员也不能跑去处理一个没有分给自己的付款任务。
```

T-RBAC 同时设置两道门：

```text
第一道门：你是不是这个任务的处理人？
第二道门：你有没有这个步骤要求的角色？
```

两道门都通过才允许。

## 三、最简单需要保存哪几类信息

T-RBAC 是一种思想，不强制规定必须有固定数量的表。

为了把关系写清楚，可以使用六张表：

```text
sys_user               用户
sys_role               角色
sys_user_role          用户拥有什么角色
task_instance          任务及当前状态
task_assignee          任务交给了谁
task_role_action       某状态允许哪个角色做什么
```

## 四、先看实际数据

### 1. 用户、角色和用户角色关系

`sys_user`：

| id | username |
| ---: | --- |
| 1 | zhangsan |
| 2 | lisi |

`sys_role`：

| id | role_name | role_key |
| ---: | --- | --- |
| 10 | 普通员工 | employee |
| 20 | 财务人员 | finance |

`sys_user_role`：

| user_id | role_id |
| ---: | ---: |
| 1 | 10 |
| 2 | 20 |

这表示：

```text
张三是普通员工
李四是财务人员
```

### 2. 任务和任务处理人

`task_instance`：

| id | task_type | current_state | status |
| ---: | --- | --- | --- |
| 5001 | EXPENSE | PAYMENT | RUNNING |

`task_assignee`：

| task_id | user_id |
| ---: | ---: |
| 5001 | 2 |

这两行表示：

```text
报销任务 5001 正在付款阶段，并且交给李四处理。
```

### 3. 任务状态、角色和操作的规则

`task_role_action`：

| task_type | task_state | role_id | action |
| --- | --- | ---: | --- |
| EXPENSE | MANAGER_REVIEW | 10 | SUBMIT_COMMENT |
| EXPENSE | PAYMENT | 20 | PAY |

第二行翻译成人话：

```text
报销任务处于 PAYMENT 状态时，财务角色可以执行 PAY。
```

## 五、六张表怎么连起来

这次实际上有两条链，最后汇合到同一个用户和任务上。

用户角色链：

```text
李四 → sys_user_role → finance 角色
```

任务处理链：

```text
李四 → task_assignee → 任务 5001 → PAYMENT 状态
```

规则链：

```text
PAYMENT 状态 + finance 角色 → 允许 PAY
```

三条信息合起来：

```text
李四负责这个任务
AND 李四是财务
AND 这个状态允许财务付款
= 允许
```

## 六、一次请求怎么判断

李四请求对任务 5001 执行 `PAY`：

```text
1. 任务 5001 是否正在运行？是。
2. 李四是否是它的处理人？是。
3. 李四拥有哪些角色？finance。
4. PAYMENT 状态是否允许 finance 执行 PAY？允许。
5. 两种身份都满足，放行。
```

如果张三是任务处理人：

```text
处理人检查通过
角色检查失败，因为张三只有 employee
= 拒绝
```

如果李四不是处理人：

```text
角色检查通过
处理人检查失败
= 拒绝
```

## 七、最简单的建表 SQL

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

CREATE TABLE sys_user_role (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE task_instance (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    task_type VARCHAR(50) NOT NULL,
    business_id BIGINT NOT NULL,
    current_state VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL
);

CREATE TABLE task_assignee (
    task_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    PRIMARY KEY (task_id, user_id)
);

CREATE TABLE task_role_action (
    task_type VARCHAR(50) NOT NULL,
    task_state VARCHAR(50) NOT NULL,
    role_id BIGINT NOT NULL,
    action VARCHAR(50) NOT NULL,
    PRIMARY KEY (task_type, task_state, role_id, action)
);
```

## 八、插入示例数据

```sql
INSERT INTO sys_user (id, username) VALUES
(1, 'zhangsan'),
(2, 'lisi');

INSERT INTO sys_role (id, role_name, role_key) VALUES
(10, '普通员工', 'employee'),
(20, '财务人员', 'finance');

INSERT INTO sys_user_role (user_id, role_id) VALUES
(1, 10),
(2, 20);

INSERT INTO task_instance
    (id, task_type, business_id, current_state, status)
VALUES
    (5001, 'EXPENSE', 1001, 'PAYMENT', 'RUNNING');

INSERT INTO task_assignee (task_id, user_id)
VALUES (5001, 2);

INSERT INTO task_role_action
    (task_type, task_state, role_id, action)
VALUES
    ('EXPENSE', 'MANAGER_REVIEW', 10, 'SUBMIT_COMMENT'),
    ('EXPENSE', 'PAYMENT', 20, 'PAY');
```

## 九、最核心的查询 SQL

判断用户 2 能否对任务 5001 执行 `PAY`：

```sql
SELECT 1
FROM task_instance ti
JOIN task_assignee ta
  ON ti.id = ta.task_id
JOIN sys_user_role ur
  ON ta.user_id = ur.user_id
JOIN task_role_action tra
  ON ti.task_type = tra.task_type
 AND ti.current_state = tra.task_state
 AND ur.role_id = tra.role_id
WHERE ti.id = 5001
  AND ti.status = 'RUNNING'
  AND ta.user_id = 2
  AND tra.action = 'PAY'
LIMIT 1;
```

这条 SQL 同时证明三件事：

```text
用户是处理人；
用户拥有合适角色；
当前任务状态允许这个角色执行目标操作。
```

## 十、T-RBAC 适用场景

* 财务审批和付款流程。
* 医院诊疗任务：既是当前经治医生，又具备医生角色。
* 生产任务：既被分配到工单，又持有设备操作资格。
* 高风险工单：既是处理人，又是安全管理员。

## 十一、高级扩展

可以继续加入：

* 角色只能在特定时间激活。
* 同一用户不能同时担任申请人和审批人。
* 金额不同，需要的角色不同。
* 会签时要求多个不同角色共同完成。
* 临时代理角色到期自动失效。

这些都建立在同一个基础之上：任务条件和角色条件必须同时满足。

## 十二、面试时怎么说

> T-RBAC 把任务权限和角色权限结合起来。用户既要是当前任务的处理人，也要拥有当前任务步骤要求的角色。数据库中一边保存用户角色关系，一边保存任务处理人关系，再用任务状态、角色和操作规则把两边连接起来，鉴权时同时满足才放行。

