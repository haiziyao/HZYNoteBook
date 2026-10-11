


### 日期转换

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
// 上面那个类是线程不安全的,容易出现问题
//


// java8之后,推出了
DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");
//线程安全,因为用到了  不可变
```

> 简单来说,就是这样,上面那个类,在对象中保存了很多状态
> 比如year month day,是作为属性
> 那么在多线程下就有共享的问题
> 对于下面那个,用的都是临时变量,在方法中.所以没有问题



### 不可变设计

##### String

##### final

##### 保护性拷贝



### 享元模式

>wikipedia： A flyweight is an object that minimizes memory usage by sharing as much data as possible with other similar objects

#### 包装类

- Boolean
- Byte
- Short
- Integer
- Long
- Character
等都提供了valueOf()方法

例如Long的valueOf()会缓存 -128~127的Long对象,在这个范围内会重用对象,超出范围才会新建Long对象
