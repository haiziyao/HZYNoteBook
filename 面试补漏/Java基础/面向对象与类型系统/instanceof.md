# `instanceof`：运行时类型判断

> 来源：[小红书-电话面-2026-8-31晚上11点](../../../面经/2026-9-面试/小红书-电话面-2026-8-31晚上11点.md)  
> 首次整理：2026-09-01　|　掌握状态：待复习

## 一、题目清单

1. Java 中 `instanceof` 关键字有什么作用？

## 二、题目与解答

### 1. Java 中 `instanceof` 关键字有什么作用？

答案：`instanceof` 判断对象在运行时是否属于指定类型，或属于该类型的子类、实现类，返回 `boolean`。

```java
if (value instanceof String s) {
    System.out.println(s.length());
}
```

- `null instanceof String` 结果为 `false`。
- 它判断类型关系，不是比较对象内容；内容相等使用 `equals`。
- `getClass()` 通常要求精确类型相同，而 `instanceof` 接受子类关系。
- Java 16+ 的模式匹配可以在判断成功后直接绑定变量。
- 过度使用 `instanceof` 可能说明可以改用多态分派。

## 追问清单

- `instanceof` 与 `getClass()` 有什么区别？
- 为什么对 `null` 判断不会抛异常？
