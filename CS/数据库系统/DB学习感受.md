---
title: DB学习感受
date: 2026-06-22
tags: [数据库, 课程反馈]
description: 数据库课程学习感受与教学建议
color: cyan
---

# 建议将DB这门课提前
DB过于重要,放在大三下真的很不舒服.

# 学习上面我所遇到的困难
(在修读这门课之前已经学习过Mysql各种基础,中级知识)
在学习的过程中,其实我仍在一些方面比较吃力:
### 1.sql 语句
对于Some, All  ,where id > All()这些语句我之前不太常用 ,有点忘记
我当时在听课的时候听概念感觉有点"痛苦",主要是概念性语言不太好理解.
其实当时就想打开电脑,找个数据库, 跑一下这个sql语句,我认为我就会明白在干什么.
我记得当时sql这一章节的课时不短,我强烈建议能够加入些实践性的东西,更有利于理解SQL的一些语句的操作
如果说更激烈一些的想法就是: SQL这一章节就作为一节实验课,准备好DB数据,然后把各种sql语言列出来,呈现下面这种形式
```
请执行下面这些sql,查看结果,并用自己的语言写出这句话在干什么
select id from table ...
你的理解: (写出你自己的理解)
练习题: 查找table中的name
```
> 我认为这样一节实践课,掌握Sql语言不成什么问题 
> 而且这样的实验也有一些好处,比如:

 #### [1581 进店却未交易的顾客](https://leetcode.cn/problems/customer-who-visited-but-did-not-make-any-transactions?envType=study-plan-v2&envId=sql-free-50)

请分析一下各种方法优缺点.

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
* NOT EXITS
相关子查询
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

*  JOIN 
``` sql
select v.customer_id, count(*) count_no_trans
from Visits v
left join Transactions t
    on v.visit_id = t.visit_id
where t.transaction_id is null
group by v.customer_id;
```

通过一些合理的题目设置,让同学自己去探究一下各种sql语句底层执行逻辑或者优劣.

> 上面的想法有些激进或者可能不太适应目前的教学情况,但是我还是比较推荐这种形式的,
> 当前时代获取知识的途径太多了,我觉得老师以后的职责更在于方向的指定而不是知识的传授.
> 这也是我为什么倡导实践的原因吧
## 2.概念讲解
在逻辑蕴含那一章节,我还是感觉学习起来挺吃力的(也可能是我自己的原因)
就是在概念讲解的时候,我几乎无法理解概念,很难理解
我不知道其他同学怎么想,但我对概念确实无感,我更偏向于那种先通过例子展示,再提炼出概念.
直接理解概念还是太痛苦了.

在讲解1NF,2NF,3NF那一章节,实质就是在对"增删改异常,数据冗余"做优化.
但是在工程中,我们往往会接受冗余或其他方法,故意违背3NF.
所以其实我挺想了解在哪些些场景,我们刻意降为1NF,哪些场景刻意降为2NF.
是基于哪些方面的考虑(因为老师讲12306的例子很有趣,所以我也很期待在这些方面也有一些实际例子)

## 3.课程安排
其实我一开始选这门课的意图就在于: 在一些实际场景中,我缺少数据库设计的能力,数据库优化能力,维持数据库高可用能力.
但是经过课程的学习,我感觉我只是学会了"ER图的一般画法",了解索引,了解事务ACID,了解并发,了解...

其实我十分建议把一些好理解的,易于学习的知识点的时长压缩一下,更注重后面的一些东西.
做一些对应的实验,比如:
> 一个Sql服务,用多线程同时执行一些不同操作
> 在sql语句没执行完之前,手动关机Sql服务
> 让同学们分析redo log, undo log,恢复数据库,理解redo log / undo log

总结成一句话就是,简单的知识简单过,有难度的知识通过实践理解掌握

# 困境
目前AI发展太快,我也不知道对于课堂的学习到底是什么样子,
大家仿佛都已经 进入"功利性"的困境, 用AI解放自己的时间.
