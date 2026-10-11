
>由于synchronized 这种锁太 硬核了
>我们更好的方式就是实现没有锁的并发

### CAS 代替加锁

- 无锁情况下，即使重试失败，线程始终在高速运行，没有停歇，而 synchronized 会让线程在没有获得锁的时 候，发生上下文切换，进入阻塞。
- 打个比喻 线程就好像高速跑道上的赛车，高速运行时，速度超快，一旦发生上下文切换，就好比赛车要减速、熄火， 等被唤醒又得重新打火、启动、加速... 恢复到高速运行，代价比较大 
- 但无锁情况下，因为线程要保持运行，需要额外 CPU 的支持，CPU 在这里就好比高速跑道，没有额外的跑 道，线程想高速运行也无从谈起，虽然不会进入阻塞，但由于没有分到时间片，仍然会进入可运行状态，还 是会导致上下文切换。

> CAS的前提必须保证可见性


### 原子类


- AtomicBoolean
- AtomicInteger
- AtomicLong

原子引用

- AtomicReference
- AtomicMarkableReference
- AtomicStampedReference

CAS无法解决ABA问题,但是也可以解决,就是加上一个版本号

- AtomicStampedReference: 带版本号的原子操作

如果我们不关心被改了几次,只关心有没有被改过

- AtomicMarkableReference

原子数组

- AtomicIntegerArray
- AtomicLongArray
- AtomicReferenceArray

字段更新器

>这个其实是神来一笔,为啥这么说
>比如, User 里面有一个 count需要时原子的
>那么我们有两种方法实现
>一种是使用AtomicInteger
>一种是用AtomicIntegerFieldUpdater
>如果要创建10000个User对象,方法一需要10000个Atomic对象
>而下面可以只有一个对象

- AtomicReferenceFieldUpdater
- AtomicIntegerFieldUpdater
- AtomicLongFieldUpdater

原子累加器

|对比项|`AtomicLong`|`LongAdder`|
|---|---|---|
|内部数据|一个共享变量|一个基础值和多个分段值|
|更新方式|所有线程竞争同一位置|尽量分散到不同位置|
|低并发性能|很好|很好，但结构稍复杂|
|高并发累加|竞争较激烈|通常吞吐量更高|
|获取结果|直接读取一个值|对各个分段求和|
|读取是否严格原子快照|是|不是|
|适合场景|精确状态控制|统计、计数、监控指标|


##### todo:欣赏LongAdder的实现


unsafe对象