
我们解释一下,为什么叫做`管程`
其实就是`Monitor`的翻译
这章节主要讲java如何通过Monitor进行共享资源的访问


`临界区`:一段代码如果存在对共享资源的多线程读写操作,那么久称为临界区
`竞态条件`: 临界区如果因为执行序列不同而导致结果无法预测,就成为竞态条件



### synchronized

简单来说,这个通过对象锁进行并发控制

这里其实没有太多内容,分清楚几个概念就好了
- 锁对象
- 锁实例

线程八锁:其实就是考争抢锁的先后顺序而有不同状态


#### 极端问题
- 变量访问范围也会使得并发问题有不同的状态
- 方法修饰符也会对并发问题有影响
	- 如果方法是public,万一被子类重写了,父类在用的时候不知道这是一个子类对象,直接并发问题



### 常见线程安全类

- String
- Integer
- StringBuffer
- Random
- Vector
- Hashtable
- Java.util.concurrent

对于某些类来说,我们说他是并发安全
只说明:
- 他们的方法是原子的
- 但是方法的组合不是原子的

>对于String,Integer这些来说
>不可变类是因为其内部的状态不可变,所以线程安全

他们类似与,如果你想改变我的值,我就new一个新对象给你返回,你就和我没有关系了.



### 例子

这里有一堆例子,时看时新吧


## Monitor-管程

### java对象头

- 普通对象
- 数组对象


我们主要在意对象头的`Mark Word`

### Synchronized原理

java对象会关联一个Montinor

Monitor
- WaitSet
	- 一堆Thread
- EntryList
	- 一个Blocked队列,全是Thread
- Owner
	- 指向Thread

> 这个真的非常简单好理解
> Owner就是目前正在执行的,就是执行synchronized(obj)的线程,
> EntryList就是其他没拿到锁,就得等(默认非公平)
> WaitSet是获得过锁,但是不满足进入Waiting状态的线程


#### 轻量级锁

- Thread创建Lock Record锁记录
- 锁记录中Obj ref指向锁对象
- CAS替换 锁记录的 地址和 obj的mark word
	- 成功:正常加锁
	- 失败:
		- 其他线程持有:锁膨胀
		- 自己持有: 锁重入,再添加一条锁记录,内容为null
- 退出:
	- 锁记录如果的值是mark word,cas交换恢复
		- 成功: 表示轻量级锁解锁
		- 失败: 已经升级为重量级锁了,换流程
	- 是null,重置这条锁记录,表示重入锁减1


#### 锁膨胀

* 另一个线程加锁,锁膨胀,给obj申请Monitor
* obj指向重量级锁,自己进入EntryList Blocked
* 此时当前线程有锁记录(有obj指针),记录了mark word,当前线程释放锁会产生cas失败
* 找到obj,那么他一定有Monitor
* 找到Monitor,讲Owner设置为null,唤醒Blocked



#### 自旋优化
重量级锁竞争的时候,可以使用自旋来优化,可以避免阻塞

- 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。 
- 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会 高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能。 
- Java 7 之后不能控制是否开启自旋功能


#### 偏向锁 (这里还有一堆关于撤销的东西,先不看了)
Java 6 中引入了偏向锁来做进一步优化：只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现 这个线程 ID 是自己的就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有


#### 锁消除
#### 锁粗化



### wait notify 原理
- Owner 线程发现条件不满足，调用 wait 方法，即可进入 WaitSet 变为 WAITING 状态 
- BLOCKED 和 WAITING 的线程都处于阻塞状态，不占用 CPU 时间片 
- BLOCKED 线程会在 Owner 线程释放锁时唤醒 
- WAITING 线程会在 Owner 线程调用 notify 或 notifyAll 时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入 EntryList 重新竞争

#### wait notify正确姿势
- sleep是Thread的方法,wait是Obj的方法
- wait需要和synchronized一起使用
- sleep时候不会释放对象锁
- wait会释放锁

#### 很多例子(不赘述)


#### 同步模式之保护性暂停(单列,需要自己重写)
#### 异步模式之生产者消费者

### Join 原理 

```java
thread.join();
//等价于
synchronized(thread){
	while(thread.isAlive()){
		thread.wait(0);
	}
}
```


### park unpark原理

每个线程都有自己的Parker对象
- _counter_
- _cond_
- _mutex_


- 调用park()方法
- 检查counter
	- counter == 0;获得mutex锁
		- 进入cond条件变量阻塞
		- 设置counter = 0;
	- counter == 1,无需阻塞直接继续运行
- 调用unpark(),设置counter = 1;
- 唤醒cond中的线程
- 线程恢复运行
- 设置couter = 0;


### 线程状态转换

- new -> runnable: 调用start()
- runnable <-> waiting
	- obj.wait()
	- obj.notify()  obj.notifyAll()  obj.interrupt()
- runnable <-> waiting
	- t.join()
	- interrupt()
- runnable <-> waiting
	- LockSupport.park()
	- LockSupport.unpark()  interrupt()
- runnable <-> timed_wating
	- obj.wait(long n)
	- obj.notify()  obj.notifyAll()  obj.interrupt()
- runnable <-> timed_waiting
	- t.join(long n)
	- interrupt()
- runnable <-> timed-waiting
	- Thread.sleep(long n)
- runnable <-> timed_waiting
	- LockSupport.parkNanos(long nanos)
	- LockSupport.unpark()  interrupt()
- runnable <-> blocked
	- synchroniezed(obj) 竞争锁失败,进入到blocked  
	- 同步代码块结束,回到runnable重新竞争
- runnable <-> terminated
	- 所有代码完毕结束


### 其他

#### 多把锁
为不同情况准备不同的锁,减小锁的细粒度

#### 活跃性

##### 死锁
有这样的情况：一个线程需要同时获取多把锁，这时就容易发生死锁


##### 定位死锁
jconsole
jps+jstack


##### 饥饿
由于优先级比较低,一直拿不到锁


### ReentranLock(这个比较重要,我们放在juc.util里面)
相对于 synchronized 它具备如下特点 
- 可中断 
	- 需要设置 lock.lockInterruptibly()
- 可以设置超时时间 
	- lock.tryLock()
- 可以设置为公平锁 
	- 默认不公平
	- 公平锁在创建的时候传入ReentrantLock lock = new ReentrantLock(true);
- 支持多个条件变量 
	- lock.newCondition()
	- condition.await()  condition.signal()
- 与 synchronized 一样，都支持可重入