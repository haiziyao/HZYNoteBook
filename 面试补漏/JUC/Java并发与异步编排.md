# Java 并发与异步编排

> 来源：[字节-后端一面-2026-8-31](../../面经学习/字节-后端一面-2026-8-31.md)  
> 首次整理：2026-09-01　|　掌握状态：部分掌握

## 一、题目清单

1. 如何保证多个线程访问共享数据时的并发安全？
2. `synchronized` 是什么类型的锁？竞争失败的线程怎么办？
3. 实例方法和静态方法上的 `synchronized` 分别锁什么？
4. 线程池的四种拒绝策略是什么？
5. `CompletableFuture` 如何创建、组合和处理异常？
6. `CompletableFuture` 默认使用什么线程池？

## 二、题目与解答

### 1. 如何保证多个线程访问共享数据时的并发安全？

根据共享状态和操作原子性选择 `synchronized`、`ReentrantLock`、读写锁、原子类/CAS、并发集合、不可变对象或线程封闭。`volatile` 只保证可见性和有序性，不能让 `i++` 这类复合操作自动具备原子性。

### 2. `synchronized` 是什么类型的锁？竞争失败的线程怎么办？

它是 JVM 内置的对象监视器锁，属于悲观锁和可重入锁，公平性不作保证。竞争 monitor 失败的线程通常进入 JVM 管理的竞争队列并处于 `BLOCKED`；这与调用 `wait()` 后的 `WAITING/TIMED_WAITING` 不同。

### 3. 实例方法和静态方法上的 `synchronized` 分别锁什么？

实例方法锁当前对象 `this`；静态方法锁当前类的 `Class` 对象；同步代码块锁括号中指定的对象。

### 4. 线程池的四种拒绝策略是什么？

- `AbortPolicy`：抛出 `RejectedExecutionException`。
- `CallerRunsPolicy`：由提交线程执行，形成反压。
- `DiscardPolicy`：静默丢弃新任务。
- `DiscardOldestPolicy`：丢弃队列头部的旧任务，再尝试提交新任务。

### 5. `CompletableFuture` 如何创建、组合和处理异常？

用 `supplyAsync`/`runAsync` 创建异步任务；`thenApply` 转换结果，`thenCompose` 串联依赖任务，`thenCombine` 合并独立任务，`allOf` 等待全部任务；`exceptionally` 提供兜底值，`handle` 统一处理结果与异常，`whenComplete` 适合日志和监控。是否重试取决于错误是否临时、操作是否幂等，并要限制次数和退避。

### 6. `CompletableFuture` 默认使用什么线程池？

未显式传 Executor 的异步方法通常使用 `ForkJoinPool.commonPool()`；不带 `Async` 的回调通常由完成前置阶段的线程执行。阻塞 I/O、数据库或 RPC 任务应使用隔离的有界线程池。
