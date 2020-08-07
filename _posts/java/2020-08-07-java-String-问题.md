# java String


## String 的字面量

先看如下例子：

主类 `test.pkg.Test` 和 `test.pkg.Other`

```java
package test.pkg;
class Test {
    public static void main(String[] args) {
        String hello = "Hello", lo = "lo";
        System.out.println(hello == "Hello");
        System.out.println(Other.hello == hello);
        System.out.println(other.Other.hello == hello);
        System.out.println(hello == ("Hel"+"lo"));
        System.out.println(hello == ("Hel"+lo));
        System.out.println(hello == ("Hel"+lo).intern());
    }
}
class Other { static String hello = "Hello"; }
```

`other.Other` 类

```java
package other;
public class Other { public static String hello = "Hello"; }
```

请问 main 方法的打印结果是什么？ 

[答案 (java se specs: String Literals)](https://docs.oracle.com/javase/specs/jls/se11/html/jls-3.html#jls-3.10.5)

TLDR；
> This example illustrates six points:
> * String literals in the same class and package represent references to the same String object (§4.3.1).
> * String literals in different classes in the same package represent references to the same String object.
> * String literals in different classes in different packages likewise represent references to the same String object.
> * Strings concatenated from constant expressions (§15.28) are computed at compile time and then treated as if they were literals.
> * Strings computed by concatenation at run time are newly created and therefore distinct.
> * The result of explicitly interning a computed string is the same String object as any pre-existing string literal with the same contents.

相同的String 字面量，在同一个类下面，在不同类下面，在不同包下面，都引用同一个String 对象。

从静态表达式连接的字符串值在编译期间会计算出来，并且将其当作字面量来对待，即，在class 文件中 `str = "a" + "b"` 会被编译成 `str = "ab"`。

运行时期间进行的String连接计算是新创建的，因此是各自独立的

显式地获取一个String计算值的intern 的操作，即 `someStr.intern()`，会从字符串常量池中获取一个字面量相等的String对象引用，注意如果值没有被提前创建，那么`intern()`会新创建一个String对象然后添加在字符串常量池中来引用。


## String 的创建

__直接赋值__

此方式在方法区中字符串常量池中创建对象
```java 
String str = "helloworld";
```

__构造器__

此方式在堆内存创建对象

```java
String str = new String();
```

## String str = new String("abc") 创建了多少个实例？

这个问题其实是不严谨的，但面试一般会遇到，所以我们要补充来说明。

>"类的加载和执行要分开来讲：创建了两个
>1. 当加载类时，"abc"被创建并驻留在了字符创常量池中（如果先前加载中没有创建驻留过）。
>2. 当执行此句时，因为"abc"对应的String实例已经存在于字符串常量池中，所以JVM会将此实例复制到会在堆（heap）中并返回引用地址。”


通过字节码我们可以看到：

源码：

```java 
String str = new String("abc")
```
字节码：

```
    Code:
       0: new           #2                  // class java/lang/String
       3: dup
       4: ldc           #3                  // String abc
       6: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)
       9: astore_1
      10: return
```

执行时仅(#2)创建了一个对象。

