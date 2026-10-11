# TBAC：因为正在处理这个任务，所以临时获得权限

TBAC 全称是 **Task-Based Access Control**，中文叫基于任务的访问控制。

先记一句话：

> 权限跟着任务走：谁正在处理任务，谁才能执行任务当前阶段允许的操作。

## 一、先看一个真实例子

张三提交了一张报销单，系统为它创建了一个审批任务：

```text
任务 5001：处理报销单 1001
当前状态：FINANCE_REVIEW（财务审核）
当前处理人：李四
```

系统规定财务审核阶段可以做：

```text
VIEW       查看
APPROVE    通过
REJECT     驳回
```

于是李四能审核这张报销单，是因为：

```text
李四是任务 5001 的当前处理人
+ 任务当前处于财务审核
+ 财务审核状态允许 APPROVE
= 李四可以执行审核通过
```

任务结束或转给王五以后，李四的这项权限也随之消失。

## 二、它控制的不是“永远能做什么”

RBAC 常说：

```text
李四是财务，所以有 expense:approve 权限。
```

TBAC 更关心：

```text
李四现在是不是这一个任务的处理人？
这个任务现在走到了哪一步？
这一步允许做什么？
```

所以 TBAC 的权限通常是临时的、与任务实例绑定的。

## 三、最简单需要保存哪几类信息

TBAC 是一种思想，不强制规定必须有固定数量的表。

这个例子最少需要知道：

```text
1. 系统里有谁
2. 有哪些正在运行的任务，任务现在是什么状态
3. 谁负责哪个任务
4. 每种任务状态允许什么操作
```

可以用四张表：

```text
sys_user
task_instance
task_assignee
task_state_action
```

## 四、先看四张表中的实际数据

### 1. 用户表

| id | username |
| ---: | --- |
| 1 | zhangsan |
| 2 | lisi |
| 3 | wangwu |

### 2. 任务实例表

| id | business_type | business_id | current_state | status |
| ---: | --- | ---: | --- | --- |
| 5001 | EXPENSE | 1001 | FINANCE_REVIEW | RUNNING |

这一行翻译成人话：

```text
任务 5001 正在处理报销单 1001，
现在走到财务审核阶段，而且任务还没有结束。
```

`business_id=1001` 指向真正的报销单。任务不是报销单本身，而是“处理这张报销单的流程”。

### 3. 任务处理人表

| task_id | user_id |
| ---: | ---: |
| 5001 | 2 |

这一行表示：

```text
任务 5001 当前交给用户 2 李四处理。
```

### 4. 状态允许操作表

| task_type | task_state | action |
| --- | --- | --- |
| EXPENSE | FINANCE_REVIEW | VIEW |
| EXPENSE | FINANCE_REVIEW | APPROVE |
| EXPENSE | FINANCE_REVIEW | REJECT |
| EXPENSE | PAID | VIEW |

前三行表示：

```text
报销任务处于财务审核状态时，可以查看、通过或驳回。
```

最后一行表示：

```text
报销任务付款完成后只能查看，不能再审核。
```

## 五、四张表到底怎么连起来

```text
sys_user
李四，id = 2
        │
        │ task_assignee.user_id
        ▼
task_assignee
task_id = 5001
        │
        │ task_instance.id
        ▼
task_instance
type = EXPENSE，state = FINANCE_REVIEW
        │
        │ 类型和状态同时匹配
        ▼
task_state_action
允许 VIEW、APPROVE、REJECT
```

整条关系读成人话：

```text
李四负责任务 5001；任务 5001 是财务审核状态；
财务审核状态允许 APPROVE；所以李四可以通过。
```

## 六、一次请求怎么判断

李四请求通过报销任务 5001：

```text
userId = 2
taskId = 5001
action = APPROVE
```

系统依次检查：

```text
1. 任务 5001 是否处于 RUNNING？是。
2. 用户 2 是否是任务 5001 的处理人？是。
3. 任务当前状态是什么？FINANCE_REVIEW。
4. FINANCE_REVIEW 是否允许 APPROVE？允许。
5. 所有条件成立，放行。
```

如果王五发起同样请求，第 2 步失败。如果任务已经进入 `PAID`，第 4 步失败。

## 七、最简单的建表 SQL

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(64) NOT NULL
);

CREATE TABLE task_instance (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    task_type VARCHAR(50) NOT NULL,
    business_type VARCHAR(50) NOT NULL,
    business_id BIGINT NOT NULL,
    current_state VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL
);

CREATE TABLE task_assignee (
    task_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    PRIMARY KEY (task_id, user_id)
);

CREATE TABLE task_state_action (
    task_type VARCHAR(50) NOT NULL,
    task_state VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    PRIMARY KEY (task_type, task_state, action)
);
```

## 八、插入示例数据

```sql
INSERT INTO sys_user (id, username) VALUES
(1, 'zhangsan'),
(2, 'lisi'),
(3, 'wangwu');

INSERT INTO task_instance
    (id, task_type, business_type, business_id, current_state, status)
VALUES
    (5001, 'EXPENSE', 'EXPENSE_ORDER', 1001,
     'FINANCE_REVIEW', 'RUNNING');

INSERT INTO task_assignee (task_id, user_id)
VALUES (5001, 2);

INSERT INTO task_state_action
    (task_type, task_state, action)
VALUES
    ('EXPENSE', 'FINANCE_REVIEW', 'VIEW'),
    ('EXPENSE', 'FINANCE_REVIEW', 'APPROVE'),
    ('EXPENSE', 'FINANCE_REVIEW', 'REJECT'),
    ('EXPENSE', 'PAID', 'VIEW');
```

## 九、最核心的查询 SQL

判断用户 2 能否对任务 5001 执行 `APPROVE`：

```sql
SELECT 1
FROM task_instance ti
JOIN task_assignee ta
  ON ti.id = ta.task_id
JOIN task_state_action tsa
  ON ti.task_type = tsa.task_type
 AND ti.current_state = tsa.task_state
WHERE ti.id = 5001
  AND ti.status = 'RUNNING'
  AND ta.user_id = 2
  AND tsa.action = 'APPROVE'
LIMIT 1;
```

SQL 的意思是：

```text
找到运行中的任务 5001，
确认用户 2 是处理人，
再确认任务当前状态允许 APPROVE。
```

## 十、TBAC 适用场景

* 请假、报销和采购审批。
* 工单流转。
* 医疗诊疗流程。
* 订单履约和售后流程。
* 任何“权限随流程阶段变化”的系统。

## 十一、高级扩展

真实工作流还可能增加：

* 候选人：任务尚未被领取时，多个人都可以领取。
* 会签：多个人都处理后才能进入下一步。
* 转办：当前处理人把任务交给别人。
* 委托：用户请假期间自动交给代理人。
* `task_history`：记录每次状态变化和操作人。
* 条件流转：金额大于 10 万时增加总经理审批。

这些是工作流能力。TBAC 最核心的三项仍然是：处理人、任务状态、状态允许的操作。

## 十二、面试时怎么说

> TBAC 是让权限跟随任务实例和任务状态变化。最简单需要任务表记录当前状态、处理人表记录谁负责该任务、状态操作表记录每个状态允许什么操作。鉴权时同时检查任务是否运行中、当前用户是不是处理人，以及当前状态是否允许目标操作。

