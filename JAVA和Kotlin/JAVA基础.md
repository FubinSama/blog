# JAVA基础

## Integer

使用自动装箱方式创建一个Integer对象时，当数值在-128 ~127时，会将创建的 Integer 对象缓存起来，当下次再出现该数值时，直接从缓存中取出对应的Integer对象。

```JAVA
Integer x = 3; // 自动装箱
Integer y = 3; // 自动装箱，因为在-128 ~127内，所以使用了缓存的对象
System.out.println(x == y); // true：因为y使用的是创建x时的缓存，所以x和y地址相同true
Integer a = new Integer(3); // 强制new一个新对象
Integer b = new Integer(3); // 强制new一个新对象
System.out.println(a == b); //false：因为a和b都是强制new的新对象，所以地址不同
System.out.println(a == x); //false：因为a是强制new的新对象，所以和x地址不同
Integer m = 300; // 自动装箱，因为不在-128 ~127内，所以没有缓存
Integer n = 300; // 自动装箱，因为不在-128 ~127内，所以没有缓存
System.out.println(m == n); // false：因为不会缓存，所以m和n都是新创建一个对象，所以地址不同
System.out.println(m == 300); // true：使用了自动拆箱，相当于判断`int m = 300; m == 300`
System.out.println(a.equals(b));//true
```

## String

在 JAVA 语言中有8种基本类型和一种比较特殊的类型String。这些类型为了使他们在运行过程中速度更快，更节省内存，都提供了一种常量池的概念。常量池就类似一个JAVA系统级别提供的缓存。

8种基本类型的常量池都是系统协调的，String类型的常量池比较特殊。它的主要使用方法有两种：

直接使用双引号声明出来的String对象会直接存储在常量池中。
如果不是用双引号声明的String对象，可以使用String提供的`intern`方法。`intern`方法会从字符串常量池中查询当前字符串是否存在，若不存在就会将当前字符串放入常量池中。

```JAVA
String s = new String("1");
s.intern();
String s2 = "1";
System.out.println(s == s2);

String s3 = new String("1") + new String("1");
s3.intern();
String s4 = "11";
System.out.println(s3 == s4);
```

JDK6时常量池存储在老年代，而对象存储在堆区，所以值为`false`、`false`，JDK7以后值为`false`、`true`。详见【"https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html"】。其实我也不太懂为什么。

## 浮点数

浮点数，基本数据类型不能直接用`==`，包装类型不能直接用`equals`。这是因为浮点数采用2进制的科学记数法，无法严格表示10进制数，本身可能会有精度丢失，因为一般用两数之差小于`1e-6`或`1e-9`来表示相等（这个在学c语言时，应该经常被強调）。

为了防止出现精度丢失，我们可以使用`BigDecimal`，其内部使用string来存储，使用`compareTo`来比较大小。注意在做除法时要指定舍入策略和精度，防止除不尽导致异常。

```JAVA
float a = 1.0f - 0.9f;
float b = 0.9f - 0.8f;
System.out.println(a);// 0.100000024
System.out.println(b);// 0.099999964
System.out.println(a == b);// false

BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
BigDecimal c = new BigDecimal("0.8");
BigDecimal x = a.subtract(b);// 0.1
BigDecimal y = b.subtract(c);// 0.1
System.out.println(x.equals(y));// true

BigDecimal a = new BigDecimal("1.0")
BigDecimal b = new BigDecimal("1.00")
System.out.println(x.equals(y));// false，注意equals也关心精度：1.0和1.00不相等
System.out.println(a.compareTo(b));// 0
```

## Arrays.asList()

> Returns `a fixed-size list backed` by the specified array. (Changes to the returned list "write through" to the array.) `This method acts as bridge between array-based and collection-based APIs`, in combination with `Collection.toArray`. The returned list is serializable and implements RandomAccess.
> This method also provides a convenient way to create a fixed-size list initialized to contain several elements:`List<String> stooges = Arrays.asList("Larry", "Moe", "Curly");`

该方法实际上返回的是`java.util.Arrays$ArrayList`类的对象，作用是充当基于数组和基于集合的 API 之间的桥梁。因而其只是对array做了一层包装，本质上还是一个定长数组，而不是`java.util.ArrayList`这样的变长列表。

想要将一个数组转换为`java.util.Array`对象：

```JAVA
String[] arr = {"a", "b", "c"};
List list1 = new ArrayList<>(Arrays.asList(arr));
// java8 stream api的方式
List list2 = Arrays.stream(arr).collect(Collectors.toList());
```

## fail-fast

`java.util`包下的集合类都是非线程安全的，其迭代器同样是非线程安全的。

其迭代器为了实现简单，不允许在用迭代器遍历一个集合对象时，直接对集合对象进行修改操作（增加、删除、修改）。其引入了一种快速失败的机制，当检测到违背了上述原则时，会抛出`Concurrent Modification Exception`。

基本原理是：集合中有一个`modCount`字段专门用来记录该集合被修改的次数，每次调用修改操作，都会导致该字段值加一。而创建迭代器时，会用一个`expectedModCount`字段来记录迭代器创建时的`modCount`值。每当迭代器使用`next()`方法获取下一个元素时，都会调用`checkForComodification()`方法来检测modCount变量是否为expectedmodCount值，从而判断迭代器创建后集合是否被修改过。

## 嵌套类

外部类只允许使用`public`或`package-private`(即不加任何修饰语)修饰。而内部类可以使用`public`、`protected`、`package-private`、`private`组合`static`来修饰。

>嵌套类分为两类：非静态和静态。非静态嵌套类称为内部类。声明为静态的嵌套类称为静态嵌套类。

```JAVA
public class OuterClass {
    
    private String a = "OuterClass a";
    private static String b = "OuterClass static b";

    private void test1() {
        System.out.println("OuterClass test1");
    }

    private static void test2() {
        System.out.println("OuterClass static test2");
    }

    class InnerClass {

        private String a = "InnerClass a";
//         private static String b = "InnerClass static b"; // 内部类OuterClass.InnerClass中的静态声明非法

        private void test1() {
            System.out.println(a); // InnerClass a
            System.out.println(b); // InnerClass static b
            System.out.println(OuterClass.this.a); // OuterClass a
            System.out.println(OuterClass.b); // OuterClass static b
            OuterClass.this.test1(); // OuterClass test1
            OuterClass.test2(); // OuterClass static test2
        }

        private void test2() {}
    }

    static class StaticNestedClass {
        private String a = "StaticNestedClass a";
        private static String b = "StaticNestedClass static b";

        private static void test1() {
//             System.out.println(a); // 无法从静态上下文中引用非静态 变量 a
            System.out.println(b); // StaticNestedClass static b
//             System.out.println(OuterClass.this.a); // 无法从静态上下文中引用非静态 变量 this
            System.out.println(OuterClass.b); // StaticNestedClass static b
//             OuterClass.this.test1(); // 无法从静态上下文中引用非静态 变量 this
            OuterClass.test2(); // StaticNestedClass static test2
        }

        private void test2() {}
    }

    public static void main(String[] args) {
        OuterClass outClass = new OuterClass();
        InnerClass innerClass = outClass.new InnerClass(); // 内部类必须使用外部类实例`.new`创建，可以访问外部类的所有访问修饰符的实例字段、方法和静态字段、类方法
        innerClass.test1();
        innerClass.test2();
        StaticNestedClass staticNestedClass = new StaticNestedClass(); // 内部静态类的实例创建方式同外部类，但是只能访问外部类的所有作用域的静态字段、方法
        staticNestedClass.test1();
        staticNestedClass.test2();
    }
}
```

## 局部类

```java

```

## 类加载机制

> 详见java8语言规范：`https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html`

类加载的基本步骤：

1. 加载：将类的二进制表示加载到内存中
2. 链接：验证（验证类的二进制表示格式正确）、准备（进行类的空间分配，并初始化为默认值，为jvm所需的额外数据结构分配内存和初始化）、解析（将符号引用转化为直接引用，这个步骤是可选的，取决于虚拟机）
3. 初始化（类的初始化包括执行其静态初始化程序和类中声明的静态字段（类变量）的初始化程序，接口的初始化包括执行接口中声明的字段（常量）的初始化程序）

类或接口类型 T 将在以下任何一项第一次出现之前立即初始化：

1. `T`是一个类，并创建了一个`T`的实例。
2. 调用由`T`声明的静态方法。
3. 分配了一个由`T`声明的静态字段。
4. 使用了由`T`声明的静态字段，并且该字段不是常量变量。
5. `T`是一个顶级类，并且执行在词法上嵌套在`T`中的`assert`语句。

> 注意：子类的初始化意味着父类必须要先初始化。子接口的初始化不会导致父接口被初始化。
> 使用类或者接口的常量变量（即`static final`修饰的变量）不会导致初始化。（接口的方法默认是`public static`，属性默认是`public static final`，即调用调用接口里面的属性不会引起接口初始化）
> 单纯的声明一个`T`类型的变量，如：`T t = null;`不会导致初始化。
> 对静态字段的引用（第 8.3.1.1 节）只会初始化实际声明它的类或接口，即使它可能通过子类、子接口或实现接口的类的名称来引用。
> 在类`Class`和包`java.lang.reflect`中调用某些反射方法也会导致类或接口初始化。

### 类初始化的细节

Java 虚拟机的实现负责通过使用以下过程来处理同步和递归初始化。
该过程假定 Class 对象已经过验证和准备，并且 Class 对象包含指示以下四种情况之一的状态：

1. 这个 Class 对象已经过验证和准备，但没有初始化。
2. 这个 Class 对象正在由某个特定的线程`T`初始化。
3. 这个 Class 对象已经完全初始化并可以使用了。
4. 此 Class 对象处于错误状态，可能是因为尝试初始化但失败。

对于每个类或接口`C`，都有一个唯一的初始化锁`LC`。从`C`到`LC`的映射由`Java`虚拟机实现决定。初始化`C`的过程如下：



## 二进制兼容性

更改成员或构造函数的声明访问以允许较少的访问可能会破坏与预先存在的二进制文件的兼容性，从而导致在解析这些二进制文件时引发链接错误。但是，在链接完成后，再将父类更改为更大访问性，并重新编译父类却不会产生错误。如：

```shell
mkdir points;
vim points/Point.java
```

```java
package points;
public class Point {
    public int x, y;
    protected void print() {
        System.out.println("(" + x + "," + y + ")");
    }
}
```

```shell
vim Test.java
```

```java
class Test extends points.Point {
    public static void main(String[] args) {
        Test t = new Test();
        t.print();
    }
    protected void print() { 
        System.out.println("Test"); 
    }
}
```

```shell
javac Test.java
java Test
# 会正常输出Test
# 现在将Point的print方法改为public，并重新编译
cd points
rm -f Point.class
vim Point.java
```

```java
class Test extends points.Point {
    public static void main(String[] args) {
        Test t = new Test();
        t.print();
    }

    // 将protected改为public
    public void print() { 
        System.out.println("Test"); 
    }
}
```

```shell
javac Point.java
cd ..
java Test
# 仍然会正常输出Test
```
