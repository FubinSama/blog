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
