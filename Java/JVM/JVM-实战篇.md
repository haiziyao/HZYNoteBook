## 内存调优

`内存泄漏(Memory Leak)`:
* 内存泄漏无法完全避免,但是持续的内存泄漏会导致`内存溢出`
* 常见场景:
	* 大型的Java后端应用中，在处理用户的请求之后，没有及时将用户的数据删除
	* 分布式任务调度系统如Elastic-job、Quartz等进行任务调度时，被调度的Java应用在调度任务结束中出现了内存泄漏，最终导致多次调度之后内存溢出。
### 解决内存溢出
* 监控发现问题
* 诊断原因
* 修复问题
* 测试验证

``` bash
top 
# load average: 0.1(1分钟) 0.1(5min) 0.1(10min) 负载值
#  RES(常驻内存)  SHR(共享内存)
```

![[Pasted image 20260719155827.png]]


#### VisualVM
VisualVM是多功能合一的Java故障排除工具并且他是一款可视化工具，整合了命令行 JDK 工具和轻量级分析功能，功能非常强大。
``` bash
-Djava.rmi.server.hostname=服务器ip地址
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9122
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false
```


#### Arthas
Arthas 是一款线上监控诊断产品，通过全局视角实时查看应用 load、内存、gc、线程的状态信息，并能在不修改应用代码的情况下，对业务问题进行诊断，包括查看方法调用的出入参、异常，监测方法执行耗时，类加载信息等，大大提升线上问题排查效率。

`Arthas tunnel` 管理微服务架构


```Properties
arthas:
  #tunnel地址，目前是部署在同一台服务器，正式环境需要拆分
  tunnel-server: ws://localhost:7777/ws
  #tunnel显示的应用名称，直接使用应用名
  app-name: ${spring.application.name}
  #arthas http访问的端口和远程连接的端口
  http-port: 8888
  telnet-port: 9999
```

``` bash
nohup java -jar -Darthas.enable-detail-pages=true arthas-tunel-server.3.7.1-farjar.jar &
# 启动 arthas tunnel 并打开web页面

nohup java -jar 
-Dserver.port=tomcat端口号
-Darthas.http-port=arthas的http端口号
-Darthas.telnet-port=arthas的telnet端口号端口号
& 
```

#### Prometheus+Grafana
Prometheus+Grafana是企业中运维常用的监控方案，其中Prometheus用来采集系统或者应用的相关数据，同时具备告警功能。Grafana可以将Prometheus采集到的数据以可视化的方式进行展示。

**优点：**
- 支持系统级别和应用级别的监控，比如linux操作系统、Redis、MySQL、Java进程。
- 支持告警并允许自定义告警指标，通过邮件、短信等方式尽早通知相关人员进行处理
**缺点：**
环境搭建较为复杂，一般由运维人员完成

阿里云一键搭建 : 
1. POM依赖
```XML
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>

    <exclusions><!-- 去掉springboot默认配置 -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
2. 配置
```Properties
management:
  endpoint:
    metrics:
      enabled: true #支持metrics
    prometheus:
      enabled: true #支持Prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: jvm-test #实例名采集
  endpoints:
    web:
      exposure:
        include: '*' #开放所有端口
```
3. 通过地址：`ip地址:端口号/actuator/prometheus`访问之后可以看到jvm相关的指标数据。
4、创建阿里云Prometheus实例

5、选择ECS服务

6、在自己的ECS服务器上找到网络和交换机

7、选择对应的网络：

填写内容，与ECS里边的网络设置保持一致

8、选中新的实例，选择MicroMeter

9、给ECS添加标签；

10、填写内容，注意ECS的标签

11、点击大盘就可以看到指标了

12、指标内容:
 

### 1.2.2 堆内存状况的对比
* 正常情况
	- 处理业务时会出现上下起伏，业务对象频繁创建内存会升高，触发MinorGC之后内存会降下来。
	- 手动执行FULL GC之后，内存大小会骤降，而且每次降完之后的大小是接近的。
	- 长时间观察内存曲线应该是在一个范围内。
- 内存泄漏
	- 处于持续增长的情况，即使Minor GC也不能把大部分对象回收
	- 手动FULL GC之后的内存量每一次都在增长
	- 长时间观察内存曲线持续增长


### 内存泄漏案例:

#### 代码不规范
* equals()和hashCode()导致
	* 没有重写上面这俩方法,导致在放入集合中的时候被判定为不同对象
	* 解决办法: 记得每次一个类重写这个方法就好了
* 内部类引用外部类
	* 比如我们通过`new Outer().new Inner()`创建对象,我们只保留内部对象(内部对象对外部对象有引用),这样就会产生内存溢出
	* 解决方法: 改成静态内部类就行了
	* 感觉这个有点......几乎用不到
	* 还有一种情况是`匿名内部类`,他会自己持有外部对象,所以无法被回收
* ThreadLocal的错误使用
	* ThreadLocal 如果是new出来的,不需要remove也能被回收
	* 如果使用的线程池,不调用remove,就会产生内存泄漏
* String 的 intern 方法
	* JDK6中字符串常量池位于堆内存中的Perm Gen永久代中，如果不同字符串的intern方法被大量调用，字符串常量池会不停的变大超过永久代内存上限之后就会产生内存溢出问题。
* 静态字段保存对象
	* 尽量减少对象长时间保存再静态变量中,不使用的时候就设置为null
	* 单例模式尽量使用懒加载
	* Spring Bean中不要长期存放大对象,如果要作为缓存,就加上过期时间
	* `@Lazy`注解懒加载
* 资源没有正常关闭

#### 并发请求问题

其实就是并发量太大了,导致内存溢出了
我们这里就学会使用Jmeter就行了


### 诊断
``` bash
-XX:+HeapDumpOnOutOfMemoryError  # 发生内存溢出,自动保存hprof内存快照
-XX:HeapDumpPath=<path>  # 指定输出路径
```

可以使用MAT打开`hprof`文件,然后查看分析报告

>MAT如何检测内存泄漏的?

1. 将`引用链`转`支配树`
2. `深堆` 与 `浅堆`
	1. `浅堆`: 
	2. `深堆`: 


> 如果没有内存泄漏,如何拿到hprof文件呢?

java原生命令 和 Arthas命令
``` bash
jmap -dump:live,format=b,file=/user/local/jvm/dump/jmap.hprof 进程ID

heapdump --live /user/local/jvm/dump/arthas.hprof
```

> hprof文件太大了,下载太慢了,怎么办

直接在服务器下载  `Mat-linux`

``` bash
./ParseHeapDump.sh 快照文件路径 org.eclipse.mat.api:suspects org.eclipse.mat.api:overview org.eclipse.mat.api:top_components
```


#### 案例1-分页查询文章接口的内存溢出

这里就是拿到hprof文件,然后找到了是哪里的问题:
* MAT分析是Mysql返回存在大量的ResultSet
* 支配树是否能发现包含的数据有一些特殊的标识
* 到线程里面看看HandlerMethod方法
* 接着去找引用关系
* 通过描述信息就能看到接口

问题根源:

>文章微服务中的分页接口没有限制最大单次访问条数，并且单个文章对象占用的内存量较大，在业务高峰期并发量较大时这部分从数据库获取到内存之后会占用大量的内存空间。

解决方法:
* 限制单词最大访问条数(代码问题)
* 分页接口只展示文章列表,不要返回内容(功能问题)
* 在高峰期对微服务进行限流保护



#### 案例2-Mybatis导致的内存溢出

问题根源: Mybatis在使用foreach进行sql拼接时，会在内存中创建对象，如果foreach处理的数组或者集合元素个数过多，会占用大量的内存空间。

解决方法: 


#### 案例3-导出大文件内存溢出

解决思路: 通过各种方式减少大文件开销
* 分批次
* 反正就是使用各种方法

#### 案例4-ThreadLocal使用时占用大量内存

* 建立了线程池,但是ThreadLocal里面的数据没有remove()
* 在Interceptor中,postHandler()在service报错的时候不会执行,得把remove放在afterCompletion()中

#### 案例5-文章内容审核接口的内存问题

`方法一`:
使用超级大的线程池

存在问题：

1、线程池参数设置不当，会导致大量线程的创建或者队列中保存大量的数据。

2、任务没有持久化，一旦走线程池的拒绝策略或者服务宕机、服务器掉电等情况很有可能会丢失任务。


`方法二`
生产者消费者,用阻塞队列

存在问题：

1、队列参数设置不正确，会保存大量的数据。

2、实现复杂，需要自行实现持久化的机制，否则数据会丢失。

`方法三`
MQ第三方组件

问题根源和解决思路：

在项目中如果要使用异步进行业务处理，或者实现生产者 – 消费者的模型，如果在Java代码中实现，会占用大量的内存去保存中间数据。

尽量使用Mq消息队列，可以很好地将中间数据单独进行保存，不会占用Java的内存。同时也可以将生产者和消费者拆分成不同的微服务。


### 在线定位问题

诊断问题有两种方法，之前我们介绍的是第一种：

- 生成内存快照并分析。
	* 优点：通过完整的内存快照准确地判断出问题产生的原因
	* 缺点：内存较大时，生成内存快照较慢，这个过程中会影响用户的使用;通过MAT分析内存快照，至少要准备1.5 – 2倍大小的内存空间
- 在线定位问题
	  - 优点：无需生成内存快照，整个过程对用户的影响较小
	  - 缺点：无法查看到详细的内存信息;需要通过arthas或者btrace工具调测发现问题产生的原因，需要具备一定的经验

`Arthas stack` 在线定位步骤

1. 使用jmap -histo:live 进程ID > 文件名 命令将内存中存活对象以直方图的形式保存到文件中，这个过程会影响用户的时间，但是时间比较短暂。
2. 分析内存占用最多的对象，一般这些对象就是造成内存泄  
3. 使用arthas的stack命令，追踪对象创建的方法被调用的调用路径，找到对象创建的根源。也可以使用btrace工具编写脚本追踪方法执行的过程。
>stack 全类名 -n 查看条数

`btrace` 在线定位

>允许使用定制化java脚本进行定位信息



## GC 调优

* 通用JVM参数的设置
* 特定垃圾回收器的Jvm参数的设置
* 解决由频繁的FULLGC引起的程序性能问题

### 核心指标

* 吞吐量: 
	* 业务吞吐量: 指在一段时间内,程序需要完成的业务数量
	* 垃圾回收吞吐量
	* 优化手段:
		* 优化业务执行性能,减少单次业务的执行时间
		* 优化垃圾回收吞吐量(code_time/(code_time+GC_time))
* 延时 (Latency)
	* Latency = GC + 业务时间
 * 内存使用量


### 调优方法

* 监控工具发现问题
* 诊断问题
* 修复问题
* 测试验证

#### 工具
* `jstat (jdk 自带工具)`

> jstat -gc 进程ID 每次统计ms间隔 统计次数
> C代表Capacity容量，U代表Used使用量
S – 幸存者区，E – 伊甸园区，O – 老年代，M – 元空间
YGC、YGT：年轻代GC次数和GC耗时（单位：秒）
FGC、FGCT：Full GC次数和Full GC耗时
GCT：GC总耗时

优点：
  操作简单
  无额外的软件安装
缺点：
  无法精确到GC产生的时间，只能用于判断GC是否存在问题

* `visualVM`
VisualVm中提供了一款Visual GC插件，实时监控Java进程的堆内存结构、堆内存变化趋势以及垃圾回收时间的变化趋势。同时还可以监控对象晋升的直方图。
优点：
  适合开发使用，能直观的看到堆内存和GC的变化趋势
缺点：
  对程序运行性能有一定影响
  生产环境程序员一般没有权限进行操作

* `Prometheus + Grafana`

优点：
  支持系统级别和应用级别的监控，比如linux操作系统、Redis、MySQL、Java进程。
  支持告警并允许自定义告警指标，通过邮件、短信等方式尽早通知相关人员进行处理
缺点：
  环境搭建较为复杂，一般由运维人员完成


#### GC日志
> 通过GC日志，可以更好的看到垃圾回收细节上的数据，同时也可以根据每款垃圾回收器的不同特点更好地发现存在的问题。

使用方法（JDK 8及以下）：`-XX:+PrintGCDetails -Xloggc:文件名`
使用方法（JDK 9+）：`-Xlog:gc*:file=文件名`


* GC Viewer
> java -jar gcviewer_1.3.4.jar 日志.log

* GC Easy

#### 常见的GC模式

* 正常情况: 锯齿状 (缓存对象较少)
	* 对象创建之后内存上升，一旦发生垃圾回收之后下降到底部，并且每次下降之后的内存大小接近，存留的对象较少。
* 缓存对象过多: 锯齿状 (缓存对象较多)
* 内存泄漏:  锯齿状 (缓存对象持续增长)
* 持续的FullGC 
	* 在某个时间点产生多次Full GC，CPU使用率同时飙高，用户请求基本无法处理。一段时间之后恢复正常。
* 元空间不足,导致持续FullGC

#### 解决GC的手段
##### 优化 JVM 参数
``` bash
-Xmx 
-Xms 
```

在设置堆内存的时候
可用堆空间 = 剩余可用空间 - 元空间 - 冗余剩余
建议 -Xmx 和 -Xms 一样大
1. 运行时性能更好
2. 可用性问题
3. 启动速度更快


```bash
-XX:MaxMetaspaceSize
-XX:MetaspaceSize
```
-XX:MaxMetaspaceSize=值 参数指的是最大元空间大小，默认值比较大，如果出现元空间内存泄漏会让操作系统可用内存不可控，建议根据测试情况设置最大值，一般设置为256m。

-XX:MetaspaceSize=值 参数指的是到达这个值之后会触发FULLGC（网上很多文章的初始元空间大小是错误的），后续什么时候再触发JVM会自行计算。如果设置为和MaxMetaspaceSize一样大，就不会FULLGC，但是对象也无法回收。
这个值:建议不设置,或者设置较小一点


``` bash
-Xss # 虚拟机器栈大小
```
如果我们不指定栈的大小，JVM 将创建一个具有默认大小的栈。大小取决于操作系统和计算机的体系结构。

比如Linux x86 64位 ： 1MB，如果不需要用到这么大的栈内存，完全可以将此值调小节省内存空间，合理值为256k – 1m之间。

使用：-Xss256k

``` bash
# 不建议设置的参数
-Xmn  # 年轻代大小, G1的话不用管,有自动调整

-XX:SurvivorRatio #伊甸园区和幸存者区的大小比例，默认值为8
-XX:MaxTenuringThreshold #大晋升阈值，年龄大于此值之后，会进入老年代

-XX:+DisableExplicitGC   #禁止在代码中使用System.gc()
-XX:+HeapDumpOnOutOfMemoryError   # 发生OutOfMemoryError错误时，自动生成hprof内存快照文件
```


```Java
// 总结优化模板
-Xms1g
-Xmx1g
-Xss256k
-XX:MaxMetaspaceSize=512m 
-XX:+DisableExplicitGC-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/opt/logs/my-service.hprof-XX:+PrintGCDetails 
-XX:+PrintGCDateStamps 
-Xloggc:文件路径
/*
DK9及之后gc日志输出修改为 -Xlog:gc*:file=文件名
堆内存大小和栈内存大小根据实际情况灵活调整。
*/
```
##### 减少对象产生
>省略,见前文


##### 更换垃圾回收器
`JDK 8`的GC组合:
1. ParNew + CMS : 保证延时
2. PS + PO : 尽量保证吞吐量
3. G1 : (不测)

`JDK 11` 默认就是 G1 了
G1 : 效果好于 ParNew+CMS 和 PS+PO

##### 优化垃圾回收器参数

CMS的并发模式失败（concurrent mode failure）现象。由于CMS的垃圾清理线程和用户线程是并行进行的，如果在并发清理的过程中老年代的空间不足以容纳放入老年代的对象，会产生并发模式失败。


### 案例实战

多想多看,时看时新


## 性能调优

### 需要解决哪些性能问题
1. CPU占用了过高问题, 达到 100%
2. 单个服务处理时间特别长
3. 程序启动之后运行正常,运行一段时间后无法处理任何请求

### 方法论
线程转储（Thread Dump）提供了对所有运行中的线程当前状态的快照。线程转储可以通过jstack、visualvm等工具获取。其中包含了线程名、优先级、线程ID、线程状态、线程栈信息等等内容，可以用来解决CPU占用率高、死锁等问题。

``` bash
jps # 获得进程ID
jstack 进程ID  > file_path
```

名称: 我们给线程设置合理名字,就知道是哪个线程
优先级: 线程优先级
Java ID(tid): JVM线程的唯一ID
本地 ID(nid): 操作系统分配给线程的唯一ID
状态: 线程状态
栈追踪: 显示整个方法的栈信息 

### 案例

#### CPU 占用 100%
``` bash
top -c 
top -p 进程ID # 按下大写H,就能看到线程了
jstack 进程ID > /usr/local/file/
printf `%x\n` 进程ID # 10进制转16进制
```


#### 接口响应时间很长
在程序运行过程中，发现有几个接口的响应时间特别长，需要快速定位到是哪一个方法的代码执行过程中出现了性能问题。

使用Arthas的trace方法可以显示调用链每个方法的耗时命令：

`trace 类名 方法名`

- 添加 `--skipJDKMethod false` 参数可以输出JDK核心包中的方法及耗时。
- 添加 ‘#cost > 毫秒值’ 参数，只会显示耗时超过该毫秒值的调用。
- 添加 `–n 数值` 参数，最多显示该数值条数的数据。
- 所有监控都结束之后，输入`stop`结束监控，重置arthas增强的对象。

``` bash
java -jar arthas

trace 类名 方法名 --skipJDKMethod false '#cost > 1000'

stop
```


在使用trace定位到性能较低的方法之后，使用watch命令监控该方法，可以获得更为详细的方法信息。
命令：
`watch 类名 方法名 ‘{params, returnObj}’ ‘#cost>毫秒值' -x 2`

`‘{params, returnObj}‘` 代表打印参数和返回值。

`-x` 代表打印的结果中如果有嵌套（比如对象里有属性），最多只展开2层。允许设置的最大值为4。


#### 定位偏底层的性能问题

Arthas提供了性能火焰图的命令

使用arthas的profile命令，生成性能监控的火焰图。
命令1： profiler start 开始监控方法执行性能
命令2： profiler stop --format html 以HTML的方式生成火焰图
火焰图中一般找绿色部分Java中栈顶上比较平的部分，很可能就是性能的瓶颈。


#### 线程被耗尽问题

线程耗尽问题，一般是由于执行时间过长，分析方法分成两步：
1、检测是否有死锁产生，无法自动解除的死锁会将线程永远阻塞。
2、如果没有死锁，再使用案例1的打印线程栈的方法检测线程正在执行哪个方法，一般这些大量出现的方法就是慢方法。


### JMH基准测试框架
Java Microbenchmark Harness
[ 官网地址：https://github.com/openjdk/jmh](https://github.com/openjdk/jmh)

```Shell
mvn archetype:generate \
-DinteractiveMode=false \
-DarchetypeGroupId=org.openjdk.jmh \
-DarchetypeArtifactId=jmh-java-benchmark-archetype \
-DgroupId=org.sample \
-DartifactId=test \
-Dversion=1.0


# 运行
mvn verify
# 执行生成后的jar包
# 放到服务器执行
```


```Java
package org.sample;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.results.format.ResultFormatType;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.text.SimpleDateFormat;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Date;
import java.util.concurrent.TimeUnit;

//执行5轮预热，每次持续1秒
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
//执行一次测试
@Fork(value = 1, jvmArgsAppend = {"-Xms1g", "-Xmx1g"})
//显示平均时间，单位纳秒
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class HelloWorldBench {

    @Benchmark
    public int test1() {
        int i = 0;
        i++;
        return i;
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(HelloWorldBench.class.getSimpleName())
                .resultFormat(ResultFormatType.JSON)
                .forks(1)
                .build();

        new Runner(opt).run();
    }
}
```


``` java
 public void  test1() {
        int i = 0;
        i++;
} //这是一个死代码的例子,会出现函数耗时0.几纳秒

 public int   test1(Blackhole bh) {
        int i = 0;
        i++;
        int j = 1;
        bh.consume(i);
        return j;
}
 //添加黑洞
```

### 案例实战
* trace 分析性能瓶颈
* 优化代码,反复使用trace测试性能提升情况
* JMH进行测试
* 对比测试结果
