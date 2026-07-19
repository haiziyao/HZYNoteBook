---
title: JVM-基础篇
date: 2026-07-12
tags: [Java, JVM]
description: Java 虚拟机组成 类加载器 字节码与运行时数据区域基础笔记
color: cyan
---
## 什么是JVM
JVM: Java Virtual Machine , 运行java.class 文件

JVM 三大特性
* 解释运行: jvm蒋字节码实时解释为机器码然后运行
* 内存管理: 自动分配内存,GC
* 即时编译: 
常见JVM

| 名称                        | 作者      | 支持版本                | 社区活跃度（github star） | 特性                                                      | 适用场景               |
| ------------------------- | ------- | ------------------- | ------------------ | ------------------------------------------------------- | ------------------ |
| HotSpot (Oracle JDK版)     | Oracle  | 所有版本                | 高(闭源)              | 使用最广泛，稳定可靠，社区活跃JIT支持Oracle JDK默认虚拟机                     | 默认                 |
| HotSpot (Open JDK版)       | Oracle  | 所有版本                | 中(16.1k)           | 同上开源，Open JDK默认虚拟机                                      | 默认对JDK有二次开发需求      |
| GraalVM                   | Oracle  | 11, 17,19企业版支持8     | 高（18.7k）           | 多语言支持高性能、JIT、AOT支持                                      | 微服务、云原生架构需要多语言混合编程 |
| Dragonwell JDK龙井          | Alibaba | 标准版 8,11,17扩展版11,17 | 低(3.9k)            | 基于OpenJDK的增强高性能、bug修复、安全性提升JWarmup、ElasticHeap、Wisp特性支持 | 电商、物流、金融领域对性能要求比较高 |
| Eclipse OpenJ9 (原 IBM J9) | IBM     | 8,11,17,19,20       | 低(3.1k)            | 高性能、可扩展JIT、AOT特性支持                                      | 微服务、云原生架构          |

## 字节码

JVM的组成
* ClassLoader
* JVM内存管理(运行时的数据区域)
* 执行引擎(JIT,解释器,GC)
* 本地接口(native)


字节码文件的组成:
* 基础信息: 魔数,版本号,标识符,父类和接口信息
	* 魔数: 0xcafebabe
	* 主版本号: 版本号 - 44
	* 其他基础信息
	* 
* 常量池:  避免相同内容重复定义,节省空间
* 字段:
* 方法: 存放**字节码指令**
``` java
//这里简单说一下, i++ 和 ++i 的异同
// 操作数栈   局部变量
// i = i++;  (i 初始为 0 )
load 1;   // 把i装在到操作数栈    操作数栈: i=0  局部变量 i =0;
inc 1 by 1; // 把局部变量i 自增   操作数栈: i=0  局部变量 i =1;
store 1; // 把操作数放在局部变量第一个的位置  操作数栈: i=0  局部变量 i = 0;

// i = ++i;  (i 初始为 0 )
inc 1 by 1; // 把局部变量i 自增   操作数栈: i=0  局部变量 i =1;
load 1;   // 把i装在到操作数栈    操作数栈: i=1  局部变量 i =1;
store 1; // 把操作数放在局部变量第一个的位置  操作数栈: i=0  局部变量 i = 1;

// 由此可见:  运算符自增的前置或者后置,主要是改变load 和 inc
```
* 属性: 主要是类的属性, 源码的文件名,内部类
* 玩转字节码常用工具
	* javap
	* javaclasslib 
	* Arthas
	* dump
	* jad



## 类的生命周期
* 加载
* 连接: 验证, 准备, 解析
* 初始化
* 使用
* 卸载


### 加载
加载(Loading)阶段第一步是**类加载器根据类的全限定名通过不同的渠道**以二进制流的方式获取字节码信息，程序员可以使用Java代码拓展的不同的渠道。
- 从本地磁盘上获取文件
- 运行时通过动态代理生成，比如Spring框架
- Applet技术通过网络获取字节码文件

类加载器在加载完类之后，Java虚拟机会将字节码中的信息保存到方法区中，方法区中生成一个InstanceKlass对象，保存类的所有信息，里边还包含实现特定功能比如多态的信息。
* 基本信息
* 常量池
* 字段
* 方法
* 虚方法表
Java虚拟机同时会在堆上生成与方法区中数据类似的java.lang.Class对象，作用是在Java代码中去获取类的信息以及存储静态字段的数据（JDK8及之后）。


### 连接
验证
1. 文件格式验证
2. 元信息验证，例如类必须有父类（super不能为空）。
3. 验证程序执行指令的语义，比如方法内的指令执行中跳转到不正确的位置。
4. 符号引用验证，例如是否访问了其他类中private的方法等。

准备: 准备阶段为静态变量（static）分配内存并设置初值，每一种基本数据类型和引用数据类型都有其初值。

|             |              |
| ----------- | ------------ |
| **数据类型**    | **初始值**      |
| **int**     | **0**        |
| **long**    | **0L**       |
| **short**   | **0**        |
| **char**    | **‘\u0000’** |
| **byte**    | **0**        |
| **boolean** | **false**    |
| **double**  | **0.0**      |
| **引用数据类型**  | **null**     |

> final修饰的基本数据类型的静态变量，准备阶段直接会将代码中的值进行赋值。


解析: 解析阶段主要是将常量池中的**符号引用**替换为**直接引用**，符号引用就是在字节码文件中使用编号来访问常量池中的内容。


### 初始化
执行 **静态代码块** 和 **类初始化(clinit)**


以下几种方式会导致类的初始化：

1.访问一个类的静态变量或者静态方法，注意变量是final修饰的并且等号右边是常量不会触发初始化。

2.调用Class.forName(String className)。

3.new一个该类的对象时。

4.执行Main方法的当前类。

添加-XX:+TraceClassLoading 参数可以打印出加载并初始化的类


```Java
public class Test1 {
    public static void main(String[] args) {
        System.out.println("A");
        new Test1();
        new Test1();
    }

    public Test1(){
        System.out.println("B");
    }

    {
        System.out.println("C");
    }

    static {
        System.out.println("D");
    }
}

// 先初始化类 Test1-> D 
// 执行main -> A
// new Test1(), 由于已经初始化了,所以 直接走构造方法 -> CBCB
// 这里 C 为什么会输出呢,是因为,代码块会被编译到 每个构造方法的前面
```


clinit不会执行的几种情况
如下几种情况是不会进行初始化指令执行的：
1.无静态代码块且无静态变量赋值语句。

2.有静态变量的声明，但是没有赋值语句。

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=YWZhYzg5ZTFiY2JiMTMwOThhNTA4MzM3ZjdhY2Y2MGNfQm5DUndVeHRCSTJiVWc1Tjl6bG9SN0ozZGxJZDVDTThfVG9rZW46UXNjNWJSeWVtb2tOUUN4MFN2UmN4RFh6bmNjXzE3ODQwMDcyMTY6MTc4NDAxMDgxNl9WNA&add_watermark=true&scene_type=CCM)

3.静态变量的定义使用final关键字，这类变量会在准备阶段直接进行初始化。

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=MzFjODZiYzYxYTU3YzIxYWM2Mjg4NjBlYzg0MmVjOTJfT3JrVWxCc0hXWEZURnRoRFRSR2pOQ3puWVNhR2ZZOFBfVG9rZW46R2FkS2JPUXRVb2RHQmh4aTA0VGNQeWNlblNlXzE3ODQwMDcyMTY6MTc4NDAxMDgxNl9WNA&add_watermark=true&scene_type=CCM)


```Java
public class Demo01 {
    public static void main(String[] args) {
        //new B02();
        System.out.println(B02.a);
    }
}

class A02{
    static int a = 0;
    static {
        a = 1;
    }
}

class B02 extends A02{
    static {
        a = 2;
    }
}

// 访问 属性 的时候,只会触发声明此字段的类的初始化
```

```Java
public class Test2 {
    public static void main(String[] args) {
        Test2_A[] arr = new Test2_A[10];

    }
}

class Test2_A {
    static {
        System.out.println("Test2 A的静态代码块运行");
    }
}

// 初始化数组 只是 初始化了一堆 空指针,所以不会触发初始化
```

```Java
public class Test4 {
    public static void main(String[] args) {
        System.out.println(Test4_A.a);
    }
}

class Test4_A {
    public static final int a = Integer.valueOf(1);

    static {
        System.out.println("Test3 A的静态代码块运行");
    }
}
```


## 类加载器 ClassLoader 

### 启动类加载器 (Bootstrap ClassLoader)
``` bash
-Xbootclasspath/a:D:/myjar/a.jar
```


### 扩展类加载器 和 应用程序加载器
- 扩展类加载器和应用程序类加载器都是JDK中提供的、使用Java编写的类加载器。
    
- 它们的源码都位于sun.misc.Launcher中，是一个静态内部类。继承自URLClassLoader。具备通过目录或者指定jar包将字节码文件加载到内存中。

继承关系图如下：

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=OTRlYTE5ZDI2ZTQ2NDE1OWZhYzZmOTUwODYxNmI5MDBfOFRLTGpSajJCNHRLTVdFc3l6bmJ0NUNoVHV2bG0xeFhfVG9rZW46Q3BKVmJOVTdXb295QUF4ZmJoaGM0ZlRIblRlXzE3ODQwMDkyNzI6MTc4NDAxMjg3Ml9WNA&add_watermark=true&scene_type=CCM)

- ClassLoader类定义了具体的行为模式，简单来说就是先从本地或者网络获得字节码信息，然后调用虚拟机底层的方法创建方法区和堆上的对象。这样的好处就是让子类只需要去实现如何获取字节码信息这部分代码。
    
- SecureClassLoader提供了证书机制，提升了安全性。
    
- URLClassLoader提供了根据URL获取目录下或者指定jar包进行加载，获取字节码的数据。
    
- 扩展类加载器和应用程序类加载器继承自URLClassLoader，获得了上述的三种能力。

扩展类加载器（Extension Class Loader）是JDK中提供的、使用Java编写的类加载器。默认加载Java安装目录/jre/lib/ext下的类文件。

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=MDg3Yjk3YzdlZTk4NmY4NmI5Njk1ZjllNWUzZjQwYzdfeFVuajd5dWFTYzFmZ241a083QnVPYUtFbUxsenQ2YkZfVG9rZW46VkxaUmJROXVib0NrM2h4OVdZdGNWbWZJbnJoXzE3ODQwMDkzMjQ6MTc4NDAxMjkyNF9WNA&add_watermark=true&scene_type=CCM)通过扩展类加载器去加载用户jar包：

- **放入/jre/lib/ext下进行扩展**。不推荐，尽可能不要去更改JDK安装目录中的内容。
    
- **使用参数进行扩展使用参数进行扩展**。推荐，使用-Djava.ext.dirs=jar包目录 进行扩展,这种方式会覆盖掉原始目录，可以用;(windows):(macos/linux)追加上原始目录

### 应用程序加载器

应用程序类加载器会加载classpath下的类文件，默认加载的是项目中的类以及通过maven引入的第三方jar包中的类。



## 双亲委派机制
* 向上查找是否加载过
* 向下尝试加载

详细流程：

每个类加载器都有一个父类加载器。父类加载器的关系如下，启动类加载器没有父类加载器：

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=NGExYjRmYjE0YzMwMjZjOTUzY2JjMTBkNDgxN2YzMDJfRDZTWmRUaXk2d0x0dE40ZWFDVnlkRkgyMVJudTZxS0ZfVG9rZW46RTloVmJBN3Rpb1hkSTJ4bGs3eGNWSG9rbm1oXzE3ODQwMTIxODQ6MTc4NDAxNTc4NF9WNA&add_watermark=true&scene_type=CCM)

- 当一个类加载器去加载某个类的时候，会自底向上查找是否加载过，如果加载过就直接返回，如果一直到最顶层的类加载器都没有加载，再由顶向下进行加载。
    
- 应用程序类加载器的父类加载器是扩展类加载器，扩展类加载器的父类加载器是启动类加载器。
    
- 双亲委派机制的好处有两点：第一是避免恶意代码替换JDK中的核心类库，比如java.lang.String，确保核心类库的完整性和安全性。第二是避免一个类重复地被加载。


## 打破双亲委派机制

### 自定义类加载器


```Java
public Class<?> loadClass(String name)
类加载的入口，提供了双亲委派机制。内部会调用findClass   bn

protected Class<?> findClass(String name)
由类加载器子类实现,获取二进制数据调用defineClass ，比如URLClassLoader会根据文件路径去获取类文件中的二进制数据。重要

protected final Class<?> defineClass(String name, byte[] b, int off, int len)
做一些类名的校验，然后调用虚拟机底层的方法将字节码信息加载到虚拟机内存中

protected final void resolveClass(Class<?> c)
执行类生命周期中的连接阶段
```



```Java
package classloader.broken;//package com.itheima.jvm.chapter02.classloader.broken;

import org.apache.commons.io.IOUtils;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.security.ProtectionDomain;
import java.util.regex.Matcher;

/**
 * 打破双亲委派机制 - 自定义类加载器
 */

public class BreakClassLoader1 extends ClassLoader {

    private String basePath;
    private final static String FILE_EXT = ".class";

    //设置加载目录
    public void setBasePath(String basePath) {
        this.basePath = basePath;
    }

    //使用commons io 从指定目录下加载文件
    private byte[] loadClassData(String name)  {
        try {
            String tempName = name.replaceAll("\\.", Matcher.quoteReplacement(File.separator));
            FileInputStream fis = new FileInputStream(basePath + tempName + FILE_EXT);
            try {
                return IOUtils.toByteArray(fis);
            } finally {
                IOUtils.closeQuietly(fis);
            }

        } catch (Exception e) {
            System.out.println("自定义类加载器加载失败，错误原因：" + e.getMessage());
            return null;
        }
    }

    //重写loadClass方法
    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        //如果是java包下，还是走双亲委派机制
        if(name.startsWith("java.")){
            return super.loadClass(name);
        }
        //从磁盘中指定目录下加载
        byte[] data = loadClassData(name);
        //调用虚拟机底层方法，方法区和堆区创建对象
        return defineClass(name, data, 0, data.length);
    }

    public static void main(String[] args) throws ClassNotFoundException, InstantiationException, IllegalAccessException, IOException {
        //第一个自定义类加载器对象
        BreakClassLoader1 classLoader1 = new BreakClassLoader1();
        classLoader1.setBasePath("D:\\lib\\");

        Class<?> clazz1 = classLoader1.loadClass("com.itheima.my.A");
         //第二个自定义类加载器对象
        BreakClassLoader1 classLoader2 = new BreakClassLoader1();
        classLoader2.setBasePath("D:\\lib\\");

        Class<?> clazz2 = classLoader2.loadClass("com.itheima.my.A");

        System.out.println(clazz1 == clazz2);

        Thread.currentThread().setContextClassLoader(classLoader1);

        System.out.println(Thread.currentThread().getContextClassLoader());

        System.in.read();
     }
}
```


默认情况下自定义类加载器的父类加载器是应用程序类加载器.

两个自定义类加载器加载相同限定名的类不会冲突，在同一个Java虚拟机中，只有相同类加载器+相同的类限定名才会被认为是同一个类。

### 线程上下文类加载器

``` java

Thread.currentThread().setContextClassLoader(classLoader);
Thread.currentThread().getContextClassLoader(classLoader);  // 不设置,默认就是 AppClassLoader
```

#### JDBC 案例 与 SPI ( 这里 没听懂,省略了)



### OSGi框架的类加载器


#### 案例: 使用 arthas不停机解决线上问题

``` bash
jad --source-only com.hzy.example > /opt/jvm/example.java  #  反编译
vim my_code.java  # 修改代码
sc -d com.hzy.example  # 查找类加载器的hash码
mc -c 21b8d17c /opt/jvm/example.java -d /opt/jvm # 编译生成.class
retransform /opt/java/com/hzy/example.class 
```


## 运行时数据区

* 线程共享
	* 方法区
	* 堆
* 线程不共享
	* 程序计数器
	* Java虚拟机栈
	* 本地方法栈

### 程序计数器
程序计数器（Program Counter Register）也叫PC寄存器，每个线程会通过程序计数器记录当前要执行的的字节码指令的地址。  

### Java虚拟机栈
采用栈的数据结构来管理方法调用中的基本数据，先进后出（First In Last Out）,每一个方法的调用使用一个栈帧（Stack Frame）来保存。

Java虚拟机栈的栈帧中主要包含三方面的内容：

- 局部变量表，局部变量表的作用是在运行过程中存放所有的局部变量
	* Nr.
	* 起始PC
	* 长度
	* 序号
	* 名字
- 操作数栈，操作数栈是栈帧中虚拟机在执行指令过程中用来存放临时数据的一块区域
    。他是一种栈式的数据结构，如果一条指令将一个值压入操作数栈，则后面的指令可以弹出并使用该值。在编译期就可以确定操作数栈的最大深度，从而在执行时正确的分配内存大小。
- 帧数据，帧数据主要包含动态链接、方法出口、异常表的引用
	* 动态链接
	* 方法出口
	* 异常表


栈溢出:Java虚拟机栈如果栈帧过多，占用内存超过栈内存可以分配的最大大小就会出现内存溢出。Java虚拟机栈内存溢出时会出现StackOverflowError的错误。

``` bash
-Xss1048576 
-Xss1024K      
-Xss1m
-Xss1g
```

本地方法栈
Java虚拟机栈存储了Java方法调用时的栈帧，而本地方法栈存储的是native本地方法的栈帧。


### 堆内存
创建出来的对象都存在于堆上。栈上的局部变量表中，可以存放堆上对象的引用。静态变量也可以存放堆对象的引用，通过静态变量就可以实现对象在线程之间共享。

堆内存大小是有上限的，当对象一直向堆中放入对象达到上限之后，就会抛出OutOfMemory错误。

`dashboard -i 刷新率`

* used: 已经使用
* total: 可用堆内存
* max: 最大堆内存

如果不设置任何的虚拟机参数，max默认是系统内存的1/4，total默认是系统内存的1/64。在实际应用中一般都需要设置total和max的值。

``` bash
-Xms6291456
-Xms6144k
-Xms6m
-Xmx83886080
-Xmx81920k
-Xmx80m
```

为什么arthas中显示的heap堆大小与设置的值不一样呢？

arthas中的heap堆内存使用了JMX技术中内存获取方式，这种方式与垃圾回收器有关，计算的是可以分配对象的内存，而不是整个内存。

Java服务端程序开发时，建议将-Xmx和-Xms设置为相同的值

### 方法区
* 类的元信息
	* 基本信息
	* 常量池
	* 字段
	* 方法
	* 虚方法表
* 运行时常量池
* 字符串常量池


```Java
/**
 * 字符串常量池案例
 */
public class Demo2 {
    public static void main(String[] args) {
        String a = "1";
        String b = "2";
        String c = "12";
        String d = a + b;
        System.out.println(c == d);
        //不同,原因在于,String + String 的底层实际上是 return new StringBuilder();
        // 也就是创建了一个新的对象,一个堆上,一个是字符串常量池
    }
}

 
package chapter03.stringtable;

/**
 * 字符串常量池案例
 */
public class Demo3 {
    public static void main(String[] args) {
        String a = "1";
        String b = "2";
        String c = "12";
        String d = "1" + "2";
        System.out.println(c == d);
    }
}
 
```

``` java
public class TestIntern {  
    public static void main(String[] args) {  
        String s1 = new StringBuilder().append("think").append("123").toString();  
	  
        System.out.println(s1.intern() == s1);  // Ture 
  
        String s2 = new StringBuilder().append("ja").append("va").toString();  
  
        System.out.println(s2.intern() == s2);  // false
    }  
}

// 这个版本的intern(),如果在堆中,就会在常量池中存一个指向堆的索引
// 上面俩结果不同是因为,上面常量池存堆的索引,下面时因为"java"在程序运行本来就被自动创建了
// 所以常量池中就不是指向堆的索引了
```

#### 面试题：静态变量存储在哪里呢？

- JDK6及之前的版本中，静态变量是存放在方法区中的，也就是永久代。
    
- JDK7及之后的版本中，静态变量是存放在堆中的Class对象中，脱离了永久代。具体源码可参考虚拟机源码：BytecodeInterpreter针对putstatic指令的处理。


### 直接内存
直接内存（Direct Memory）并不在《Java虚拟机规范》中存在，所以并不属于Java运行时的内存区域。

在 JDK 1.4 中引入了 NIO 机制，使用了直接内存，主要为了解决以下两个问题:

1、Java堆中的对象如果不再使用要回收，回收时会影响对象的创建和使用。

2、IO操作比如读文件，需要先把文件读入直接内存（缓冲区）再把数据复制到Java堆中。

现在直接放入直接内存即可，同时Java堆上维护直接内存的引用，减少了数据复制的开销。写文件也是类似的思路。


## 垃圾回收(Garbage Collection)
- 自动垃圾回收，自动根据对象是否使用由虚拟机来回收对象
    - 优点：降低程序员实现难度、降低对象回收bug的可能性
    - 缺点：程序员无法控制内存回收的及时性
- 手动垃圾回收，由程序员编程实现对象的删除
    - 优点：回收及时性高，由程序员把控回收的时机
    - 缺点：编写不当容易出现悬空指针、重复释放、内存泄漏等问题


### 方法区回收(不常用)
1. 此类所有实例对象都已经被回收，在堆中不存在任何该类的实例对象以及子类对象。
2. 加载该类的类加载器已经被回收。
3. 该类对应的 java.lang.Class 对象没有在任何地方被引用。
### 堆回收
`引用计数法` (java没有使用)
引用计数法会为每个对象维护一个引用计数器，当对象被引用时加1，取消引用时减1。
* 缺点: 
	1.每次引用和取消引用都需要维护计数器，对系统性能会有一定的影响
	2.存在循环引用问题，所谓循环引用就是当A引用B，B同时引用A时会出现对象无法回收的问题。

>-verbose:gc 

`可达性分析法`
Java使用的是可达性分析算法来判断对象是否可以被回收。可达性分析将对象分为两类：垃圾回收的根对象（GC Root）和普通对象，对象与对象之间存在引用关系。
 
`GC Root对象`
* Thread对象, 引用线程栈帧中的方法参数,局部变量
* 系统类加载器的java.lang.Class对象, 引用类中的静态变量
* 监视器对象,用来保存同步锁synchronized关键字持有的对象。
* 本地方法调用时使用的全局对象。


`强引用`
`软引用`:
* 如果一个对象只有软引用关联到它，当程序内存不足时，就会将软引用中的数据进行回收。
```Java
/**
 * 软引用案例3 - 引用队列使用
 */
public class SoftReferenceDemo3 {

    public static void main(String[] args) throws IOException {

        ArrayList<SoftReference> softReferences = new ArrayList<>();
        ReferenceQueue<byte[]> queues = new ReferenceQueue<byte[]>();
        for (int i = 0; i < 10; i++) {
            byte[] bytes = new byte[1024 * 1024 * 100];
            SoftReference studentRef = new SoftReference<byte[]>(bytes,queues);
            softReferences.add(studentRef);
            //软引用在内存不足的时候会删除软引用的堆内存
            //但是软引用对象没有被删除
	        //java提供了一种方法会自动把软引用对象放入一个队列
	        //然后我们自行删除
        }

        SoftReference<byte[]> ref = null;
        int count = 0;
        while ((ref = (SoftReference<byte[]>) queues.poll()) != null) {
            count++;
        }
        System.out.println(count);

    }
}
```

`弱引用`
弱引用的整体机制和软引用基本一致，区别在于弱引用包含的对象在垃圾回收时，不管内存够不够都会直接被回收。在JDK 1.2版之后提供了WeakReference类来实现弱引用，弱引用主要在ThreadLocal中使用。


`虚引用`
虚引用也叫幽灵引用/幻影引用，不能通过虚引用对象获取到包含的对象。虚引用唯一的用途是当对象被垃圾回收器回收时可以接收到对应的通知。Java中使用PhantomReference实现了虚引用，直接内存中为了及时知道直接内存对象不再使用，从而回收内存，使用了虚引用来实现。

`终结器引用`
终结器引用指的是在对象需要被回收时，终结器引用会关联对象并放置在Finalizer类中的引用队列中，在稍后由一条由FinalizerThread线程从队列中获取对象，然后执行对象的finalize方法，在对象第二次被回收时，该对象才真正的被回收。在这个过程中可以在finalize方法中再将自身对象使用强引用关联上，但是不建议这样做。


### 垃圾回收算法
* 阶段
	* 标记阶段
	* 清除阶段
* 算法
	* 标记-清除算法
	* 复制算法
	* 标记-整理算法
	* 分代GC
* 评价标准
	* 吞吐量
	* 最大暂停时间
	* 堆使用效率

`标记清除算法`
* 过程:
	* 标记阶段
	* 清除阶段
* 优点:
	* 维护标志位就行,比较简单
* 缺点:
	* 内存碎片化

`复制算法`
* 过程
	* From 和 To 两块空间
	* GC时候把存活的放在To空间
	* 清除From空间
	* 互换
* 优点
	* 吞吐量高
	* 不会碎片化
* 缺点
	* 内存利用率低

`标记整理算法`
* 过程
	* 标记存活
	* 整理到连续片段
* 优点
	* 内存利用率高
	* 不会碎片化
* 缺点
	* 效率不高

`分代GC`
 现代优秀的垃圾回收算法，会将上述描述的垃圾回收算法组合进行使用
* 内存规划
	* 新生代
		* 伊甸园
		* 幸存区
			* S0
			* S1
	* 老年代
* 过程
	* 创建对象,在伊甸园分配内存
	* 伊甸园区满了,触发Minor GC,使用From To算法复制到 幸村区
	* 持续From To 算法(对每个对象维护一个年龄字段,每次移动都年龄加1)
	* 大年龄的放入老年代
	* 老年代满了就触发 Full GC
	* Full GC之后仍然放不进去对象,抛出 Out Of Memory


### 垃圾回收器
为什么分代GC算法要把堆分成年轻代和老年代？首先我们要知道堆内存中对象的特性：

- 系统中的大部分对象，都是创建出来之后很快就不再使用可以被回收，比如用户获取订单数据，订单数据返回给用户之后就可以释放了。
    
- 老年代中会存放长期存活的对象，比如Spring的大部分bean对象，在程序启动之后就不会被回收了。
    
- 在虚拟机的默认设置中，新生代大小要远小于老年代的大小


分代GC算法将堆分成年轻代和老年代主要原因有：

1、可以通过调整年轻代和老年代的比例来适应不同类型的应用程序，提高内存的利用率和性能。

2、新生代和老年代使用不同的垃圾回收算法，新生代一般选择复制算法，老年代可以选择标记-清除和标记-整理算法，由程序员来选择灵活度较高。

3、分代的设计中允许只回收新生代（minor gc），如果能满足对象分配的要求就不需要对整个堆进行回收(full gc),STW时间就会减少。

由于垃圾回收器分为年轻代和老年代，除了G1之外其他垃圾回收器必须成对组合进行使用。

具体的关系图如下：

![](https://lisxpq12rl7.feishu.cn/space/api/box/stream/download/asynccode/?code=NjEyMjZjMTI2MjFjMWEyN2FhZmZlMWVkNDY5YWM5ZWJfYmI3cjdlMDJHUVVYaHdPRWZQUXRHZFQ0QmtiZzE0RWNfVG9rZW46Sm15S2JBVjEyb2gzTlh4ekVkbmNCSUl3bmplXzE3ODQzNjEyNzA6MTc4NDM2NDg3MF9WNA&add_watermark=true&scene_type=CCM)

`年轻代Serial `:单线程串行回收年轻代的垃圾回收器。
* 复制算法
* 优点:
	* 单CPU处理下吞吐量出色
* 缺点:
	* 多CPU下吞吐量不如其他垃圾回收器
	* 堆如果偏大会让用户线程处于长时间的等待
* 使用场景:
	* Java编写的客户端程序或者硬件配置有限的场景
`老年代-SerialOld`:Serial垃圾回收器的老年代版本，采用单线程串行回收
* 标记-整理算法
* 优点:
	* 单CPU处理下吞吐量出色
* 缺点:
	* 多CPU下吞吐量不如其他垃圾回收器，堆如果偏大会让用户线程处于长时间的等待
* 使用场景
	* 与Serial垃圾回收器搭配使用，或者在CMS特殊情况下使用

>-XX:+UseSerialGC 新生代、老年代都使用串行回收器。

`年轻代-ParNew`
ParNew垃圾回收器本质上是对Serial在多CPU下的优化，使用多线程进行垃圾回收
-XX:+UseParNewGC 新生代使用ParNew回收器， 老年代使用串行回收器

`老年代- CMS(Concurrent Mark Sweep)垃圾回收器`
CMS垃圾回收器关注的是系统的暂停时间，允许用户线程和垃圾回收线程在某些步骤中同时执行，减少了用户线程的等待时间。

参数：XX:+UseConcMarkSweepGC

缺点：

1、CMS使用了标记-清除算法，在垃圾收集结束之后会出现大量的内存碎片，CMS会在Full GC时进行碎片的整理。这样会导致用户线程暂停，可以使用-XX:CMSFullGCsBeforeCompaction=N 参数（默认0）调整N次Full GC之后再整理。

2.、无法处理在并发清理过程中产生的“浮动垃圾”，不能做到完全的垃圾回收。

3、如果老年代内存不足无法分配对象，CMS就会退化成Serial Old单线程回收老年代。

`年轻代-Parallel Scavenge垃圾回收器`
Parallel Scavenge是JDK8默认的年轻代垃圾回收器，多线程并行回收，关注的是系统的吞吐量。具备自动调整堆内存大小的特点。


* 年轻代
* 复制算法
* 吞吐量高，而且手动可控。为了提高吞吐量，虚拟机会动态调整堆的参数
* 不能保证单次的停顿时间
* 后台任务，不需要与用户交互，并且容易产生大量的对象。比如：大数据的处理，大文件导出
`老年代-Parallel Old垃圾回收器`
Parallel Old是为Parallel Scavenge收集器设计的老年代版本，利用多线程并发收集。
参数： -XX:+UseParallelGC 或
-XX:+UseParallelOldGC可以使用Parallel Scavenge + Parallel Old这种组合。

* 老年代
* 标记-整理算法
* 并发收集，在多核CPU下效率较高
* 暂停时间会比较长
* 与Parallel Scavenge配套使用

`G1垃圾回收器`
1.支持巨大的堆空间回收，并有较高的吞吐量。
2.支持多CPU并行垃圾回收。
3.允许用户设置最大暂停时间。
JDK9之后强烈建议使用G1垃圾回收器。

G1的整个堆会被划分成多个大小相等的区域，称之为区Region，区域不要求是连续的。分为Eden、Survivor、Old区。Region的大小通过堆空间大小/2048计算得到，也可以通过参数-XX:G1HeapRegionSize=32m指定(其中32m指定region大小为32M)，Region size必须是2的指数幂，取值范围从1M到32M。

* young GC
	* 新创建的对象会存放在Eden区。当G1判断年轻代区不足（max默认60%），无法分配对象时需要回收时会执行Young GC。
	* 标记出Eden和Survivor区域中的存活对象，
	* 根据配置的最大暂停时间选择某些区域将存活对象复制到一个新的Survivor区中（年龄+1），清空这些区域。
	* 后续Young GC时与之前相同，只不过Survivor区中存活对象会被搬运到另一个Survivor区。
	* 当某个存活对象的年龄到达阈值（默认15），将被放入老年代。
	* 部分对象如果大小超过Region的一半，会直接放入老年代，这类老年代被称为Humongous区。比如堆内存是4G，每个Region是2M，只要一个大对象超过了1M就被放入Humongous区，如果对象过大会横跨多个Region。
	* 多次回收之后，会出现很多Old老年代区，此时总堆占有率达到阈值时（-XX:InitiatingHeapOccupancyPercent默认45%）会触发混合回收MixedGC。回收所有年轻代和部分老年代的对象以及大对象区。采用复制算法来完成。
* Mixed GC
	* 回收过程
		* 初始标记
		* 并发标记
		* 最终标记
		* 并发复制
	* G1对老年代的清理会选择存活度最低的区域来进行回收，这样可以保证回收效率最高，这也是G1（Garbage first）名称的由来。
	* 如果清理过程中发现没有足够的空Region存放转移的对象，会出现Full GC。单线程执行标记-整理算法，此时会导致用户线程的暂停。所以尽量保证应该用的堆内存有一定多余的空间。