## GraalVM 

不看了,跳过了


## 新一代的垃圾回收器

### 垃圾回收器的演进

* PS+PO
* ParNew + CMS
* G1

新GC:
* Shenandoah : 
* ZGC :


### Shenandoah 
Shenandoah 是由Red Hat开发的一款低延迟的垃圾收集器，Shenandoah 并发执行大部分 GC 工作，包括并发的整理，堆大小对STW的时间基本没有影响。

### ZGC
GC 是一种可扩展的低延迟垃圾回收器。ZGC 在垃圾回收过程中，STW的时间不会超过一毫秒，适合需要低延迟的应用。支持几百兆到16TB 的堆大小，堆大小对STW的时间基本没有影响。

ZGC降低了停顿时间，能降低接口的最大耗时，提升用户体验。但是吞吐量不佳，所以如果Java服务比较关注QPS（每秒的查询次数）那么G1是比较不错的选择。

``` bash
-XX:+UseZGC -XX:+ZGenerational #分代  
-XX:+UseZGC   # 非分代
```

- 动设置年轻代大小，无需设置-Xmn参数。
- 需要设置的参数：
      -Xmx 值 最大堆内存大小
- 可以设置的参数：
      -XX:SoftMaxHeapSize=值


ZGC 中可以使用Linux的Huge Page大页技术优化性能，提升吞吐量、降低延迟。
注意：安装过程需要 root 权限，所以ZGC默认没有开启此功能


操作步骤：

1、计算所需页数，Linux x86架构中大页大小为2MB，根据所需堆内存的大小估算大页数量。比如堆空间需要16G，预留2G（JVM需要额外的一些非堆空间），那么页数就是18G / 2MB = 9216。

2、配置系统的大页池以具有所需的页数（需要root权限）：

$ echo 9216 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

3、添加参数-XX:+UseLargePages 启动程序进行测试




## Java Agent库

### Java工具
常见的Java工具有以下几类：
1. 诊断类工具，如Arthas、VisualVM等。
2. 开发类工具，如Idea、Eclipse。
3. APM应用性能监测工具，如Skywalking、Zipkin等。
4. 热部署工具，如Jrebel等。

#### Java Agent技术
Java Agent技术是JDK提供的用来编写Java工具的技术，使用这种技术生成一种特殊的jar包，这种jar包可以让Java程序运行其中的代码。
* 静态加载模式: 适合APM等性能检测系统
	* 编写premain方法
	* java -javaagent:./test-agent.jar
	* 在主线程执行之前
	* 会自动执行
* 动态加载模式:
	* agentmain
	* 连接java进程
	* 额外线程执行


##### 静态模式示例

加上插件
``` xml
<build>  
    <plugins>  
        <plugin>  
            <groupId>org.apache.maven.plugins</groupId>  
            <artifactId>maven-assembly-plugin</artifactId>  
            <configuration>                <descriptorRefs>  
                    <descriptorRef>jar-with-dependencies</descriptorRef>  
                </descriptorRefs>  
                <archive>                    <manifestFile>src/main/resources/MANIFEST.MF</manifestFile>  
                </archive>  
            </configuration>  
        </plugin>  
    </plugins>  
</build>
```

配置文件`MANIFEST.MF`
```
Manifest-Version: 1.0  
Premain-Class: com.hzy.AgentMain  
Agent-Class: com.hzy.AgentMain  
Can-Redefine-Classes: true  
Can-Retransform-Classes: true  
Can-Set-Native-Method-Prefix: true
```

写代码
```java
public class AgentMain {  
    public static void premain(String[] args, Instrumentation inst) {  
        System.out.println("premain 执行了");  
    }  
    public static void agentmain(String[] args, Instrumentation inst) {  
        System.out.println("agentmain 执行了");  
    }  
}
```

执行
```bash
java -jar -javaagent:./aaa.jar my.jar
```

##### 动态示例

大体相同,只有代码不同
```java
 
public class AttachMain {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        VirtualMachine vm = VirtualMachine.attach("24200");
        vm.loadAgent("D:\\jvm-java-agent\\target\\itheima-jvm-java-agent-jar-with-dependencies.jar");
    }
}
```


### 实现一个简化的Arthas
* 查看内存信息
* 生成堆内存快照
* 打印栈信息
* 打印类加载器
* 打印类的源码
* 打印方法执行的参数和耗时

#### JMX技术-ManagementFactory

```Java
public class AgentDemo {

    /**
     * 参数添加模式 启动java主程序时添加 -javaangent:agent路径
     * @param agentArgs
     * @param inst
     */
    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("java agent执行了...");
    }

    /**
     * attach 挂载模式 java主程序运行之后，随时可以将agent挂载上去
     */

    //-XX:+UseSerialGC -Xmx1g -Xms512m
    public static void agentmain(String agentArgs, Instrumentation inst) {
        //打印内存的使用情况
        memory();
    }

    //获取内存信息
    private static void memory(){
        List<MemoryPoolMXBean> memoryPoolMXBeans = ManagementFactory.getMemoryPoolMXBeans();

        System.out.println("堆内存：");
        //获取堆内存
        getMemoryInfo(memoryPoolMXBeans, MemoryType.HEAP);

        //获取非堆内存
        System.out.println("非堆内存：");
        getMemoryInfo(memoryPoolMXBeans, MemoryType.NON_HEAP);

        //nio使用的直接内存
        try{
            @SuppressWarnings("rawtypes")
            Class bufferPoolMXBeanClass = Class.forName("java.lang.management.BufferPoolMXBean");
            @SuppressWarnings("unchecked")
            List<BufferPoolMXBean> bufferPoolMXBeans = ManagementFactory.getPlatformMXBeans(bufferPoolMXBeanClass);
            for (BufferPoolMXBean mbean : bufferPoolMXBeans) {
                StringBuilder sb = new StringBuilder();
                sb
                        .append("name:")
                        .append(mbean.getName())

                        .append(" used:")
                        .append(mbean.getMemoryUsed()/ 1024 / 1024)
                        .append("m")

                        .append(" max:")
                        .append(mbean.getTotalCapacity() / 1024 / 1024)
                        .append("m");

                System.out.println(sb);
            }
        }catch (Exception e){
            System.out.println(e);
        }

    }

    private static void getMemoryInfo(List<MemoryPoolMXBean> memoryPoolMXBeans, MemoryType heap) {
        memoryPoolMXBeans.stream().filter(x -> x.getType().equals(heap))
                .forEach(x -> {
                    StringBuilder sb = new StringBuilder();
                    sb
                            .append("name:")
                            .append(x.getName())

                            .append(" used:")
                            .append(x.getUsage().getUsed() / 1024 / 1024)
                            .append("m")

                            .append(" max:")
                            .append(x.getUsage().getMax() / 1024 / 1024)
                            .append("m")

                            .append(" committed:")
                            .append(x.getUsage().getCommitted() / 1024 / 1024)
                            .append("m");

                    System.out.println(sb);
                });
    }

    public static void main(String[] args) {
        memory();
    }
}
```


创建堆内存快照

``` java
public static void heapDump(){  
    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");  
    HotSpotDiagnosticMXBean hotSpotDiagnosticMXBean = ManagementFactory.getPlatformMXBean(HotSpotDiagnosticMXBean.class);  
    try {  
        hotSpotDiagnosticMXBean.dumpHeap(simpleDateFormat.format(new Date())+".hprof", true);  
    } catch (IOException e) {  
        throw new RuntimeException(e);  
    }  
  
}
```

打印栈信息

```Java
package com.itheima.jvm.javaagent.demo03;

import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;

public class ThreadCommand {

    public static void printStackInfo(){
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] infos = threadMXBean.dumpAllThreads(threadMXBean.isObjectMonitorUsageSupported(),
                threadMXBean.isSynchronizerUsageSupported());
        for (ThreadInfo info : infos) {
            StringBuilder stringBuilder = new StringBuilder();
            stringBuilder.append("name:")
                    .append(info.getThreadName())
                    .append(" threadId:")
                    .append(info.getThreadId())
                    .append(" state:")
                    .append(info.getThreadState())
            ;
            System.out.println(stringBuilder);

            StackTraceElement[] stackTrace = info.getStackTrace();
            for (StackTraceElement stackTraceElement : stackTrace) {
                System.out.println(stackTraceElement.toString());
            }

            System.out.println();
        }
    }



    public static void main(String[] args) {
        printStackInfo();
    }
}
```


打印类加载器

Java Agent中可以获得Java虚拟机提供的Instumentation对象：

该对象有以下几个作用： 
1、redefine，重新设置类的字节码信息。 
2、retransform，根据现有类的字节码信息进行增强。 
3、获取所有已加载的类信息。


```Java

public class ClassCommand {

    //获取所有类加载器
    private static Set<ClassLoader> getAllClassLoader(Instrumentation inst){
        HashSet<ClassLoader> classLoaders = new HashSet<>();
        Class[] allLoadedClasses = inst.getAllLoadedClasses();
        for (Class clazz : allLoadedClasses) {
            ClassLoader classLoader = clazz.getClassLoader();
            classLoaders.add(classLoader);
        }

        return classLoaders;
    }

    public static void printAllClassLoader(Instrumentation inst){
        Set<ClassLoader> allClassLoader = getAllClassLoader(inst);
        String result = allClassLoader.stream().map(x -> {
            if (x ==null) {
                return "BootStrapClassLoader";
            } else {
                return x.toString();
            }
        }).distinct().sorted(String::compareTo).collect(Collectors.joining(","));
        System.out.println(result);
    }
}
```

打印类的源码

```XML
<dependency>
    <groupId>org.jd</groupId>
    <artifactId>jd-core</artifactId>
    <version>1.1.3</version>
</dependency>
```

```Java
//获取类信息
public static void printClass(Instrumentation inst){
    Scanner scanner = new Scanner(System.in);
    System.out.println("请输入类名:");
    String next = scanner.next();
    Class[] allLoadedClasses = inst.getAllLoadedClasses();
    System.out.println("要查找的类名是:" + next);
    //匹配类名
    for (Class clazz : allLoadedClasses) {
        if(clazz.getName().equals(next)){
            System.out.println("找到了类,类加载器为:" + clazz.getClassLoader());
            ClassFileTransformer transformer = new ClassFileTransformer() {
                @Override
                public byte[] transform(Module module, ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
                    ClassFileToJavaSourceDecompiler classFileToJavaSourceDecompiler = new ClassFileToJavaSourceDecompiler();

                    Printer printer = new Printer() {
                        protected static final String TAB = "  ";
                        protected static final String NEWLINE = "\n";

                        protected int indentationCount = 0;
                        protected StringBuilder sb = new StringBuilder();

                        @Override public String toString() { return sb.toString(); }

                        @Override public void start(int maxLineNumber, int majorVersion, int minorVersion) {}
                        @Override public void end() {
                            System.out.println(sb.toString());
                        }

                        @Override public void printText(String text) { sb.append(text); }
                        @Override public void printNumericConstant(String constant) { sb.append(constant); }
                        @Override public void printStringConstant(String constant, String ownerInternalName) { sb.append(constant); }
                        @Override public void printKeyword(String keyword) { sb.append(keyword); }
                        @Override public void printDeclaration(int type, String internalTypeName, String name, String descriptor) { sb.append(name); }
                        @Override public void printReference(int type, String internalTypeName, String name, String descriptor, String ownerInternalName) { sb.append(name); }

                        @Override public void indent() { this.indentationCount++; }
                        @Override public void unindent() { this.indentationCount--; }

                        @Override public void startLine(int lineNumber) { for (int i=0; i<indentationCount; i++) sb.append(TAB); }
                        @Override public void endLine() { sb.append(NEWLINE); }
                        @Override public void extraLine(int count) { while (count-- > 0) sb.append(NEWLINE); }

                        @Override public void startMarker(int type) {}
                        @Override public void endMarker(int type) {}
                    };

                    try {
                        classFileToJavaSourceDecompiler.decompile(new Loader() {
                            @Override
                            public boolean canLoad(String s) {
                                return false;
                            }

                            @Override
                            public byte[] load(String s) throws LoaderException {
                                return classfileBuffer;
                            }
                        },printer,className);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    //System.out.println(new String(classfileBuffer));
                    return ClassFileTransformer.super.transform(module, loader, className, classBeingRedefined, protectionDomain, classfileBuffer);
                }
            };

            inst.addTransformer(transformer,true);
            try {
                inst.retransformClasses(clazz);
            } catch (UnmodifiableClassException e) {
                e.printStackTrace();
            }finally {
                inst.removeTransformer(transformer);
            }

        }
    }
}
```


打印方法执行的参数和耗时

使用java agent来实现这种方法,比使用AOP增强的优势大很多

`ASM`字节码增强

```XML
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm</artifactId>
    <version>9.6</version>
</dependency>
```

听天书了,因为都是各种Api是否了解,知道.

`ByteBuddy`

这些我们都听个响就行了


### APM系统
Application performance monitor 

听个响