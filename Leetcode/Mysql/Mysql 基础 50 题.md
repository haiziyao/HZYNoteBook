
---
title: Mysql 基础 50 题
date: 2026-06-22
tags: [数据库, SQL]
description: LeetCode SQL 50 题练习笔记
color: cyan
---

#### [1581 进店却未交易的顾客](https://leetcode.cn/problems/customer-who-visited-but-did-not-make-any-transactions?envType=study-plan-v2&envId=sql-free-50)

分析一下,这道题主要来借机会讲一下各种方法优缺点.

你会怎么做这道题?使用 连接过滤 还是使用not in过滤

* NOT IN 
``` sql
select customer_id, count(*) count_no_trans
from Visits
where visit_id not in (
    select visit_id
    from Transactions
)
group by customer_id;
```

这种方式就是易于理解,但是有大坑,如果 子查询存在 NULL ,
那么就会出现  NOT IN (0,1,NULL)

> 我们来简单解释一下Mysql 的NULL 
> 在WHERE id NOT IN (0,1,NULL)里,执行链路是:
> id <> 0 && id <> 1 && id <> NULL 
> 假设id = 3 结果是  true && true && NULL 
> 最终结果是 NULL
> where 会过滤掉 false 和 NULL
> 也就是说,只要 NOT IN ( ) 中出现NULL,你就永远查不出东西出来

**结论: IN 可以使用, NOT IN 慎用**

* NOT EXITS

对于初学者来说, 相关子查询可能有点没怎么见过. 

比如我们见过 where id  in (select  ... ),这是子查询

相关子查询 指 当前查询与 子查询通过 where 过滤字段 相关联 
子查询依赖外层查询的一行，不能脱离外层单独理解。

或许也可以这么说,子查询是有先后概念的,先查子,后面复用子查询

但是相关子查询,是每次都要进行一次子 
``` sql
select v.customer_id, count(*) count_no_trans
from Visits v
where not exists (
    select 1
    from Transactions t
    where t.visit_id = v.visit_id
)
group by v.customer_id;
```

> select 1 这里没有啥含义,用  select * 是相同的效果, 

对于 exists 的作用,读者可以再去AI了解一下, 简单来说 exists 只判断有没有行
比如
>select * from A where exists (select 1);
>的结果是全集
 

*  JOIN 
``` sql
select v.customer_id, count(*) count_no_trans
from Visits v
left join Transactions t
    on v.visit_id = t.visit_id
where t.transaction_id is null
group by v.customer_id;
```

这种写法就很常规了,

>不过这里有个有趣的事情,如果一个字段被定义为NOT NULL,在进行left/right join 的时候,
有些字段是要补NULL的,那么 被定义NOT NULL 的字段会补吗?

**总结: 推荐使用 exist, 不容易出错,也不需要考虑 连接运算产生的巨大中间结果.
但是对于现代优化器,都会有优化的,执行计划可能和数据定义的相同,所以使用什么方式无所谓了


#### 问题2: 如何选用 相关子查询 或者 连接  

``` sql
select s.machine_id,
       round(avg(e.timestamp - s.timestamp), 3) processing_time
from Activity s
join Activity e
    on s.machine_id = e.machine_id
   and s.process_id = e.process_id
   and s.activity_type = 'start'
   and e.activity_type = 'end'
group by s.machine_id;
```

``` sql
select s.machine_id machine_id , round(AVG(e.timestamp-s.timestamp),3) processing_time from
(select * from Activity where activity_type='start') s
join
(select * from Activity where activity_type='end') e
on s.machine_id = e.machine_id and s.process_id = e.process_id
group by s.machine_id
```

> 这两种写法到底该选哪一种?
> 没有答案. 实际场景需要自己动手去 explain 看看执行计划



### Join 的 on 和 using

[一道连接题](https://leetcode.cn/problems/confirmation-rate?envType=study-plan-v2&envId=sql-free-50)

> 其实这道题很简单,但是我从这道题 注意到了一个小细节 

``` sql
select s.user_id,Round(IFNULL(AVG(c.action='confirmed'),0),2) confirmation_rate
from Signups s left join Confirmations c
on s.user_id = c.user_id
group by s.user_id
```

直接看代码其实就行,主要就是两个表的连接,问题在
> select `s.user_id`  写成 `user_id`  行不行 ? 
> 解答: 不行. 

* join  on: 请记住, on 只是两个表 关联的一种关系,不会自动进行合并操作
>  比如可能出现 s.user_id = 1 但是 c.user_id = null 的操作, 因为这里是 left join
>  请记住, join on 只会进行一个判断是否要放在同一行,只是做匹配
* join using
>  如果使用 using 的话,就可以使用 关联 id 进行 一个  自然连接的操作,这是 会进行合并的


> 可以打开上面那个链接,去动手试试 把 s.user_id 改为 c.user_id 或者 user_id 看看结果