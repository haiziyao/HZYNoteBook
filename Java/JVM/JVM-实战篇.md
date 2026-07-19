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

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=NTQ0Njk4NzU0NjY3MjljZTY0M2M2MzVkZGY2OTJjNWZfWEl0TldvTGszbkhMZXFzejMybVNKRm5hdURMN3ByNFVfVG9rZW46UEZtRmJ4RlZ1b0VrZnR4RDU2WWMzcXdMbkxoXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

5、选择ECS服务

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWI4MDQzNDRmYmZjNGYzZWNjNzQ0NzgxYWY1M2I3M2RfenlvdTRTcEdOVGVwMDRac1NjdVJWaENqcTRvNEdrMnFfVG9rZW46S2pmdWI1T2FTb0NSTEx4dGNOemNBeW9zbkpnXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

6、在自己的ECS服务器上找到网络和交换机

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=YzA2YjNjZjU1Yzc5YTYzYzg1OTY4MWY2MThjYjFhZDFfNWUzYk42clZTdWk5SXlpOThXeXBwRDh1VWt6RkZMWEtfVG9rZW46SjNIdWIyVTVVb2xTWnR4UGdQb2NVWlZLbmxnXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

  

7、选择对应的网络：

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDM2YjYwNGYyNzNiNDVkZTI4NDliN2EzOTFlMzY0MGZfYTNLYlF4TVMxR3ZId2RyWnNPSnNDS3FLaEt4R3VPZFdfVG9rZW46QmNNOWJFYlREb1I5NzR4Q21XNGM1dzFhbkhoXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

填写内容，与ECS里边的网络设置保持一致

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=OWI0NmVjNzhlMmJiZTIwNTEzOWY1ZmExOTEyZGRmZTFfVGVXWXBNUkQxTmJKbENDTTVtVzhoMVNLeFRiNEJ0VXZfVG9rZW46Szh0dmJEQ0djb0pYVVN4bmhuUGNxTlNMbnplXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

8、选中新的实例，选择MicroMeter

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTczMTM4MGI4MTcyNWZhMGM4ODJlOTBjMGJmOTliNDVfd2NpR2RHR1B3YmJmQnF0ZkxXMm9XeTZ0OEw5RHRjeWhfVG9rZW46RWxtdGJlcWpab3FjOTN4MnRBc2NMVUw5bnNiXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=NWNiOGQ4ZjBhNmJkYTJmOWMwYTA2M2YwOTliYjIyNjRfeEI1aGd1T3E0UlZjYkw2UTZuMkJBUW80bnIybHh5N3NfVG9rZW46SURmNmJXeTd2b2k1WHN4UmlEdGNxWHNQbmZjXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

9、给ECS添加标签；

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=MzdhMGM3MzE1OWNmOGE1OGIzNjdmNjNhNzhmMGNmZGNfclhmQWhxeXF6UWQwb1VGcDl6dFVKellPamlFbnlPUHpfVG9rZW46T3paY2JIcmxJb2toVjJ4TWJXUmNnSndLbjdnXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=YWRkOTY2N2I1YzUwMTM1MGZmODAwZmMyYTdmNGZjNzRfRFJpT3oySjVteFhwcWE1bkxKd0RERHpCdEU4SVJ3NzFfVG9rZW46QTlZV2J6V3Rqb3hPY1Z4Ym0zaGM2YndVbnZkXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

10、填写内容，注意ECS的标签

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDk2ZDNjNDlkMGUwZDQyMjNiMzhhNTU4MWFkYjM2MWRfTDlsa3p4clZYTkZadHJnN2M5YU1iYk56RllGWkU4dVZfVG9rZW46QVFCM2JReVVNb1J6QzV4TGpjVmNsVVc0blZnXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

11、点击大盘就可以看到指标了

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=YjJkNmMxMTRkYjc1MzgzNGRjODljMmJiNzFkY2IwNjJfTTlFdXcwV1ZyZjBhc1JzS010Y3VnbHRqRGFGU2xjTU5fVG9rZW46Q0pTM2JVVExZbzhUdFZ4RDFXTWNKQzlObkhnXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

12、指标内容:

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=NmI5YTA2NzdlYmNjMzhjODU1YTk1NGE3MGU3NDIwYmNfNTlYYzlWbWhRek51YkNWOFUwSE1SYk5LY2ZCUUVGM1RfVG9rZW46RTNjNmJKbWw5b1Y2VnB4QzBsNmNZWFpXbmhjXzE3ODQ0NTI2NDE6MTc4NDQ1NjI0MV9WNA&add_watermark=true&scene_type=CCM)

  

### 1.2.2 堆内存状况的对比
* 正常情况
	- 处理业务时会出现上下起伏，业务对象频繁创建内存会升高，触发MinorGC之后内存会降下来。
	- 手动执行FULL GC之后，内存大小会骤降，而且每次降完之后的大小是接近的。
	- 长时间观察内存曲线应该是在一个范围内。
- 内存泄漏
	- 处于持续增长的情况，即使Minor GC也不能把大部分对象回收
	- 手动FULL GC之后的内存量每一次都在增长
	- 长时间观察内存曲线持续增长


内存泄漏案例:
* equals()和hashCode()导致
	* 没有重写上面这俩方法,导致在放入集合中的时候被判定为不同对象
	* 解决办法: 记得每次一个类重写这个方法就好了
* 内部类引用外部类
	* 