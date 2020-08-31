# Java 问题: equals ，== 和 hashcode() 的区别

## `==` 运算符

简单来说, 对于基本类型是值比较，对于引用类型来说是引用比较。

在[java se specification 文档](https://docs.oracle.com/javase/specs/jls/se11/html/jls-15.html#jls-15.21)中对于 == 运算符有如下3种情况的解释:

1. __`==`用于计算 数值间的相等性时，做二进制值判断__

    * long 和 int 做整数相等性判断
        * 值相等则，== 结果为 true, 反之为false
    * float 和 double 做浮点数相等性判断，遵循 IEEE 754的标准
        * 如果其中一个是NaN，那么== 的结果是false
        * 正数0和负数0视为相等的，即 `-0.0 == 0.0` 为 true
        * 其他情况的浮点数值比较视为不相等，特别如正无穷和负无穷，它们只与其本身相等
2. __布尔类型相等性__

    如果 == 的两个布尔值相等，且为基本类型 boolean，或是两布尔值相等，且其中一个是基本类型 boolean 另一个是包装类型 `Boolean`, 则 == 为 true, 反之为 false

同样也是做值比较，但后者由于涉及到基本类型和包装类型间的运算，触发了自动解包(unboxing)。但如果是两个`Boolean`就不同了，涉及到了引用类型的判断：

```java
System.out.println(new Boolean(true) == new Boolean(true));     // false
```

3. __引用类型的判断, 进行对象间比较__

== 操作数包括引用类型(reference type)或是 null 类型的情况

* 如果两个值都是 null，为true
* 如果两个值引用了同一个对象，或者是同一个array，== 的结果是 true

## equals 方法

由于对象继承自 Object, 默认的equals使用了 `Object::equals` 的实现，即判断`this == given`

覆写该方法建议满足：反身相等，对称相等，传递相等，一致性（幂等），可以看看`String::equals`的实现。

java api 建议如果覆写了 `equals()` 需要覆写 `hashCode()` 方法, 以维护 “equals 结果为true, 那么 hashCode() 的结果是相等的” 的准则

## hashCode 方法

hashCode 方法也继承自 Object, 是native方法，使用C++ 实现，返回一串整数，大多数情况是一个独特的散列值，但有的地方会覆写，比如Boolean 的 hashCode 分别返回了true 和 false 的两个固定魔数。

如果两个对象的equals的结果是false, 其hashCode 的结果不一定是不同的。例如"通话"和"重地"两个中文单词。此概念详见“哈希表”的数据结构和算法内容，计算key 的hash值的算法有很多种，但hash值冲突的次数越少，hash table 的性能就越好。例如HashMap, 当对象的hashCode() 值相等，就会分享同一个hash 桶的位置，当equals为true, 那么对象都在hashmap 的同一列上，存放位置相同，那么hashCode就该一样。

```
                  +-----------------------------------+
                  |   |   |   |   |   |   |   |   |   |
  hashTable:      | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
                  |   |   |   |   |   |   |   |   |   |
                  +---------^---------------^---------+
                            |               |
                            |               |
                            |               |
                            |          +----------+
  Hash("Hello") = 2    +---------+     |          |
                       |         |     |  "World" |
                       | "Hello" |     +----------+
  Hash("World") = 6    +----^----+
                            |
                            |
                       +---------+
                       |         |
                       | "Hello" |
                       +---------+
```