

>这章节分为两部分
>线程池的使用和Juc.util包下工具类的使用



## 线程池


### 手搓线程池(todo:这里需要复习手搓一下)


### ThreadPoolExecutor

`线程池状态`: int 的高3位标识状态,低29位表示线程数量

- running: 111
- shutdown: 000
- stop: 001
- tidying: 010
- terminated: 011

`构造方法`: 必须熟记

```java
public ThreadPoolExecutor(
			int corePoolSize,  // 最多保留的线程数量
			int maximumPoolSize, // 最大线程数目
			long keepAliveTime,  // 生存时间
			TimeUnit unit, // 时间单位
			BlockingQueue<Runnable> workQueue, //阻塞队列
			ThreadFactory threadfactory, //线程工程
			RejectedExecutionHandler handler // 拒绝策略
)
```

- 刚开始没有线程,有任务来了,就创建线程去执行任务
- 线程数目达到了corePoolSize,其他task进入阻塞队列
- 如果队列是有界队列,当任务数量大于队列大小,会创建maximumPoolSize-corePoolSize数量个线程来救急
- 如果线程达到maximumPoolSize仍然有新任务,执行拒绝策略
	- AbortPolicy: 抛出异常,默认行为
	- CallerRunsPolicy: 调用者执行任务
	- DiscardPolicy: 直接放弃
	- DiscardOldestPolicy: 放弃最早的,取而代之
	- Dubbo: 记录日志,抛异常
	- Netty: 创建一个新线程执行任务
	- ActiveMQ: 超时等待60s
	- PinPoint: 使用拒绝策略链,逐一尝试各个策略
- 如果救急线程在 keepAliveTime* TimeUnit之后没有任务做,就会销毁

`newFixedThreadPool`
```java
public static ExecutorService newFixedThreadPool(int nThreads){
	return new ThreadPoolExecutor(nThreads,nThreads,0L,TimeUnit.MILLSECONDS,
					new LinkedBlockingQueue<Runnable>())
}
```


>评价 适用于任务量已知，相对耗时的任务


`newCachedThreadPool`
```java
public static ExecutorService newCachedThreadPool(){
	return new ThreadPoolExecutor(0,Integer.MAX_VALUE,60L,
						TimeUnit.SECONDS,
					new SynchronousQueue<Runnable>())
}
```

SynchronousQueue是没有容量的,必须得有一个线程来取才能放进去

>评价 整个线程池表现为线程数会根据任务量不断增长，没有上限，当任务执行完毕，空闲 1分钟后释放线 程。 适合任务数比较密集，但每个任务执行时间较短的情况


`newSingleThreadExecutor`
```java
public static ExecutorService newSingleThreadPool(){
	return new ThreadPoolExecutor(1,1,0L,TimeUnit.MILLISECONDS,
					new LinkedBlockingQueue<Runnable>())
}
```


>希望多个任务排队执行。线程数固定为 1，任务数多于 1 时，会放入无界队列排队。任务执行完毕，这唯一的线程 也不会被释放。

为啥要用线程池,不用一个线程直接完成,
因为线程池可以保障线程炸了再补充一个



`执行任务`

```java
void execute(Runnable command);   //执行任务

<T> Future<T> submit(Callable<T> task); //提交,并拿到future

<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;  //提交一堆,拿到所有future

<T> T invokeAny(Collection<? extends Callable<T> tasks) throws InterruptedException, ExecutionException; // 提交一堆,只返回最快执行完的第一个,其他全取消

<T> T invokeAny(Collection<? extends Callable<T> tasks
			long timeout, TimeUnit unit) 
			throws InterruptedException, ExecutionException; // 提交一堆,只返回最快执行完的第一个,其他全取消,带超时时间
```

`关闭线程池`

```java
// shutdown
/* 线程池状态变为 SHUTDOWN 
 不会接收新任务 
 但已提交任务会执行完
  此方法不会阻塞调用线程的执行 */ 
void shutdown();
       
       
List<Runnable> shutdownNow();
/*
状态变为 STOP
不会接受新任务
会将队列的任务返回
interrupt 中断正在执行的任务
*/


boolean isShutdown();// 不在running就返回true

boolean isTerinated();

boolean awaitTermination(long timeout,TimeUnit unit) throw InterruptedExeception;

```


`Work Thread`
让有限的工作线程（Worker Thread）来轮流异步处理无限多的任务。也可以将其归类为分工模式，它的典型实现 就是线程池，也体现了经典设计模式中的享元模式。


`任务调度线程池`

在『任务调度线程池』功能加入之前，可以使用 java.util.Timer 来实现定时功能，Timer 的优点在于简单易用，但 由于所有任务都是由同一个线程来调度，因此所有任务都是串行执行的，同一时间只能有一个任务在执行，前一个 任务的延迟或异常都将会影响到之后的任务。


普通线程池解决的是：

> 任务现在提交，现在尽快执行。

任务调度线程池解决的是：

> 任务先提交，但等 5 秒、每天一次，或者每隔 1 秒再执行。


```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(2);
executor.schedule(()->{},1000,TimeUnit.MILLSECONDS);

ScheduledExecutorSerivice pool = Executors.newScheduledThreadPool(1);
pool.scheduleAtFixedRate(()->{},1,1,TimeUnit.SECONDS);
pool.scheduleWithFixedDelay(();
```


fixedRate：按照计划开始时间计算
fixedDelay：按照上次结束时间计算





`fork+join`
提交给 Fork/Join 线程池的任务需要继承 RecursiveTask（有返回值）或 RecursiveAction（没有返回值）

```java
//这里可以写一个fork+join+ForkJoinPool 的实列
```

## Java.Util.Concurrent

### AQS原理

#### 概述
AbstractQueueSynchronizer 是阻塞式锁和相关的同步器工具的框架
- getState()
- setState()
- compareAndSetState-cas机制设置state状态
- 独占模式是只有一个线程能够访问,共享是多个线程都可以访问
- FIFO等待队列,类似Monitor的EntryList
- 条件变量实现等待唤醒,支持多个条件变量

- tryAcquire
- tryRelease
- tryAcquireShared
- tryReleaseShared
- isHeldExclusively


#### 不可重入锁实现
通过继承AQS实现一个不可重入锁

```java
public class MySync extends AbstractQueuedSynchronizer {  
  
    @Override  
    protected boolean tryAcquire(int arg) {  
        if(arg == 1){  
            if(compareAndSetState(0,1)){  
                setExclusiveOwnerThread(Thread.currentThread());  
                return true;  
            }  
        }  
        return false;  
    }  
  
    @Override  
    protected boolean tryRelease(int arg) {  
        if(arg == 1){  
            if (compareAndSetState(1,0)){  
                setExclusiveOwnerThread(null);  
                return true;  
            }  
        }  
        return false;  
    }  
  
    @Override  
    protected boolean isHeldExclusively() {  
        return getState() == 1;  
    }  
}
```

#### 自定义锁
```java
class MyLock implements Lock {  
	
	// 用上面的自定义队列同步器
    static MySync sync = new MySync();  
  
    @Override  
    public void lock() {  
        sync.acquire(1);  
    }  
  
    @Override  
    public void lockInterruptibly() throws InterruptedException {  
        sync.acquireInterruptibly(1);  
    }  
  
    @Override  
    public boolean tryLock() {  
        return sync.tryAcquire(1);  
    }  
  
    @Override  
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {  
        return sync.tryAcquireNanos(1,unit.toNanos(time));  
    }  
  
    @Override  
    public void unlock() {  
        sync.release(1);  
    }  
  
    @Override  
    public Condition newCondition() {  
        return sync.new ConditionObject();  
    }  
}
```



### ReentrantLock原理


#### 非公平锁

```java
public ReentrantLock(){
	sync = new NonfairSync();
}
```

- 没有竞争
	- setExclusiveOwnerThread == currentThread()
- 有竞争(Thread0占据锁)
	- CAS 修改state,失败
	- tryAcquire(), 失败
	- addWaiter,构造Node队列
		- 队列head->dummy(哑元)->Node(Thread)->tail
	- 当前线程进入acquireQueued逻辑
		- 在死循环中不断获取锁,失败后park阻塞
		- 如果自己是真实node的第一个,再次tryAcquire,失败
		- 进入shouldParkAfterFailedAcquire,将前去waitStatus改为-1,返回false
	- shouldParkAfterFailedAcquire返回结果,再次tryAcquire,失败
		- 由于此次waitStatus已经是-1,这次直接返回true
	- 进入parkAndCheckInterrupt 
- 其他线程也同时来竞争,同上
- Thread0释放锁
- 当前队列不是null,head的waitStatus == -1 ,进入unparkSuccessor
- 找到最近的一个node,unpark恢复
- Thread-1加锁成功
	- exclusiveOwnerThread为当前线程,state = 1
	- 收回这个Thread-Node
- 队列里面的Thread-1在准备拿锁的同时,又来了个Thread-4,他没进队列,直接来拿锁了,导致Thread-1失败
	- Thread-1继续上面的有竞争的逻辑,Thread-4走加锁成功的逻辑

#### 可重入原理

```java
// 主要就是通过state保存的值可大于1来保证

final boolean nonfairTryAcquire(int acquires){
	final Thread current = Thread.currentThread();
	int c = getState();
	if(c==0){
		if(CAS){
			setExclusiveOwnerThread(current);
		}
		return true;
	}else if(current == getExclusiveOwnerThread()){
		int nextc = c + acquires;
		if (nextc < 0) // overflow throw new Error("Maximum lock count exceeded");
		setState(nextc); 
		return true;
	}
	return false;
}

//对于 tryRelease来说也差不多,不屑了 
```


#### 可打断原理(难以理解)

##### 不可打断模式
在此模式下，即使它被打断，仍会驻留在 AQS 队列中，一直要等到获得锁后方能得知自己被打断了



##### 可打断模式


#### 公平锁


#### 条件变量

每个条件变量其实就对应着一个等待队列，其实现类是 ConditionObject


### 读写锁

```java
ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
ReentrantReadWriteLock.ReadLock = rw.readLock();
ReentrantReadWriteLock.WriteLock = rw.writeLock();
```

- 读锁不支持条件变量 
- 重入时升级不支持：即持有读锁的情况下去获取写锁，会导致获取写锁永久等待
- 重入时降级支持：即持有写锁的情况下去获取读锁

#### 原理



#### 源码



#### StampedLock

该类自 JDK 8 加入，是为了进一步优化读性能，它的特点是在使用读锁、写锁时都必须配合【戳】使用 加解读锁



### Semaphore

>相当于提前定义了一堆许可证,只有拿到许可证才能执行




### CountdownLatch

用来进行线程同步协作，等待所有线程完成倒计时。 其中构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数减一




### CyclicBarrier

循环栅栏，用来进行线程协作，等待线程满足某个计数。构造时设置『计数个数』，每个线程执 行到某个需要“同步”的时刻调用 await() 方法进行等待，当等待的线程数满足『计数个数』时，继续执行

```java
CyclicBarrier cb = new CyclicBarrier(2);

		cb.await();
		cb.await();
		
// 只有到达固定数量,才会能够俩人都启动
```

### ConcurrentHashMap


### BlockingQueue


### ConcurrentLinkedQueue


### CopyOnWriteArrayList