
## 线程

### 创建线程
```java
// 方法1
Thread thread = new Thread("t1"){  
    public void run(){  
            }  
};  
thread.start();

//方法2:
Thread thread2 = new Thread(new Runnable(){
	public void run(){
	}
});
thread2.start();

// 也可以
Runnable task2 = ()->log.debug("hello");
new Thread(task2,"t3").start();



// 方法3
FutureTask<Interger> task = new FutureTask<>(()->{
	log.debug("hello");
	return 3;
});
new Thread(task3, "t3").start();

 // 主线程阻塞，同步等待 task 执行完毕的结果 
Integer result = task3.get();
 log.debug("结果是:{}", result);

```

### 线程常见方法
```java
start(); // 启动线程,并执行run()方法.让线程进入就绪阶段
run(); // 线程启动后自动调用的方法,我们就是通过重写run()方法实现代码逻辑
join(); // 等待线程结束
join(Long n); //等 n 毫秒
getId();
getName();
setName(String);
getPriority();
setPriority(int);  // 在没有使用操作系统线程的Java早期版本中，线程优先级可能很有用。不过现在不要再使用线程优先级了

getState(); //  获取线程状态
// java中用6个enum表示  NEW,RUNNABLE,BLOCKED,WAITING,TIMED_WAITING,TERMINATED

isInterrupted(); //判断是否被打断,不会清除打断标记
interrupted(); //判断是否被打断,清除标记


interrupt(); //打断线程
// 如果线程在 sleep,wait,join 会导致 InterruptedException,清除标记
// 如果线程正在运行,会设置标记
// park线程被打断,也会被设置打断标记

currentThread(); //获取当前正在执行的线程
sleep(long n); // 休眠
yield(); // 让出对cpu的使用
```


### 打断

打断 sleep , wait ,join 的线程,会清空打断标记
打断征程运行的线程,不会清空打断标记
打断park线程,不会清空打断状态
如果打断标记是true,不会触发park(并且park不会消费打断标记,park实现原理是一个授权标记)
也就是说提前使用unpark也会使得park失效



### 主线程与守护线程
有一种特殊的线程叫做守护线程，只要其它 非守护线程运行结束了，即使守护线程的代码没有执行完，也会强制结束。


##  线程状态

### 操作系统五层状态

- 初始: 语言层面创建线程,未与操作系统关联
- 可运行: 线程被创建
- 运行: 获取CPU时间片
- 阻塞: 
- 终止: 线程完毕

### Java Api 六种状态
- new: 被创建,没调用start()
- Runnable: 这个状态覆盖了操作系统层面的三个状态
	- 运行
	- 阻塞
	- 可运行
- Blocked: 
- Waited:
- Time_waited
- Terminated: 线程代码运行结束

## 虚拟线程

- 更加轻量级
- 资源消耗更少
- 上下文切换开销低
- 改善阻塞操作的处理
- 简化并发编程
- 提升性能

### 原理介绍(先欠着)

### 使用说明

```java
// 方法1
Thread.startVirtualThread(()->{
	sout("hello");
});

// 方法2
Thread.ofVirtual()  
        .name("test")  
        .uncaughtExceptionHandler(((t, e) -> System.out.println(t)))  
        .start(()->{  
            System.out.println("hello");  
        });
        
        
// 方法3
	Thread hello = Thread.ofVirtual().unstarted(() -> {  
    System.out.println("hello");  
});  
hello.start();
```

对于线程池的话,不太建议使用虚拟线程池,因为几乎不常用.........