# JVM

> https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.1

Java 虚拟机期望几乎所有类型检查都在运行时之前完成（通常由编译器完成），而不必由 Java 虚拟机本身完成。

对于Values of primitive types，Java 虚拟机的指令集使用旨在对特定类型的值进行操作的指令来区分其操作数类型。例如，`iadd`、`ladd`、`fadd`和`badd`都两个数值相加并生成数值结果的JVM指令，但每个指令都专门针对其操作数类型：分别为`int`、`long`、`float` 和`double`。

JVM支持的primitive data types类型包括：*numeric type*，*boolean type*和*returnAddress type*：

- *numeric types*包括*integral types*和*floating-point types*。

  - *integral types*包括：

    - `byte`, whose values are **8-bit signed two's-complement integers**, and whose default value is **zero**。

    - `short`, whose values are **16-bit signed two's-complement integers**, and whose default value is **zero**

    - `int`, whose values are **32-bit signed two's-complement integers**, and whose default value is **zero**

    - `long`, whose values are **64-bit signed two's-complement integers**, and whose default value is **zero**

    - `char`, whose values are **16-bit unsigned integers representing Unicode code points** in the Basic Multilingual Plane, **encoded with UTF-16**, and whose default value is the **null code point (`'\u0000'`)**

  - floating-point types*包括：

    - `float`, whose values are elements of the float value set or, where supported, the **float-extended-exponent value set**, and whose default value is **positive zero**

    - `double`, whose values are elements of the double value set or, where supported, the **double-extended-exponent value set**, and whose default value is **positive zero**

- The values of the `boolean` type encode the truth values `true` and `false`, and the default value is **false**.

- The values of the `returnAddress` type are **pointers to the opcodes of Java Virtual Machine instructions**. Of the primitive types, only the `returnAddress` type is **not directly associated with a Java programming language type**.

对于浮点数，IEEE 754 标准不仅包括正负符号数值，还包括正负零、正无穷大和负无穷大以及特殊的非数字值（以下缩写为“NaN”）。 NaN 值用于表示某些无效运算的结果，例如零除零。

Java 虚拟机的每个实现都需要支持两个标准浮点值集，称为浮点值集和双精度值集。此外，Java虚拟机的实现可以根据其选择支持两个扩展指数浮点值集中的一个或两个，称为浮点扩展指数值集和双精度扩展指数值集。

浮点，浮点扩展 -  exponent，double和double扩展的exponent值集不是类型。实现Java虚拟机始终是正确的，可以使用float值设置的元素来表示float类型的值；但是，在某些情况下，实现可以使用float-extended-Exponent值集的元素。同样，实现使用双值设置的元素代表类型double的值始终是正确的。但是，在某些情况下，实现可以使用双重扩展 -  exponent值集的元素。

**除 NaN 外，浮点值集的值都是有序的。从小到大排列时，分别是负无穷大、负有限值、正负零、正有限值、正无穷大。**

**浮点正零和浮点负零比较相等**，但还有其他操作可以区分它们；例如，**1.0 除以 0.0 会产生正无穷大，但 1.0 除以 -0.0 会产生负无穷大**。

**NaN 是无序的，因此如果其中一个或两个操作数均为 NaN，则数值比较和数值相等测试的值为 false**。特别是，当且仅当该值为 NaN 时，对某个值与其自身的数值相等性进行测试时，该测试的值为 false，即**NaN != NaN**。

**returnAddress 类型由 Java 虚拟机的 jsr、ret 和 jsr_w 指令（§jsr、§ret、§jsr_w）使用。 returnAddress 类型的值是指向 Java 虚拟机指令的操作码的指针。**

虽然Java虚拟机定义了bool类型，但它只提供了非常有限的支持。没有专门用于布尔值操作的 Java 虚拟机指令。相反，**Java 编程语言中对布尔值进行操作的表达式被编译为使用 Java 虚拟机 int 数据类型的值**。

Java 虚拟机直接支持`boolean arrays`。其`newarray`指令允许创建`boolean arrays`。使用字节数组指令`baload`和`bastore`访问和修改`boolean arrays`。而**在 Oracle 的 Java 虚拟机实现中，Java 编程语言中的bool数组被编码为 Java 虚拟机字节数组，每个bool元素使用 8 位**。

**引用类型分为三种：`class types`、`array types`和`interface types`**。它们的值分别是对动态创建的*class instances*、*arrays*或*class instances or arrays that implement interfaces*的引用。