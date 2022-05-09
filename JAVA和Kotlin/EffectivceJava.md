# Effective Java笔记

- [Effective Java笔记](#effective-java笔记)
  - [创建和销毁对象](#创建和销毁对象)
    - [用静态工厂方法代替构造器](#用静态工厂方法代替构造器)
    - [遇到多个构造器参数时，考虑使用build模式](#遇到多个构造器参数时考虑使用build模式)
    - [用私有构造器或者枚举类型强化Singleton属性](#用私有构造器或者枚举类型强化singleton属性)
    - [通过构造器私有化强化不可实例化的能力](#通过构造器私有化强化不可实例化的能力)
    - [优先考虑依赖注入来引用资源](#优先考虑依赖注入来引用资源)
    - [避免创建不必要的对象](#避免创建不必要的对象)
    - [消除过期的对象引用](#消除过期的对象引用)
    - [避免使用finalizer和cleaner方法](#避免使用finalizer和cleaner方法)
    - [对于实现了AutoCloseable接口的类，try-with-ressources优先于try-finally](#对于实现了autocloseable接口的类try-with-ressources优先于try-finally)
  - [对所有对象都通用的方法](#对所有对象都通用的方法)
    - [重写equals时请遵守通用约定](#重写equals时请遵守通用约定)
    - [重写equals时总要重写hashCode](#重写equals时总要重写hashcode)
    - [始终要重写toString](#始终要重写tostring)
    - [谨慎的重写clone方法](#谨慎的重写clone方法)

## 创建和销毁对象

### 用静态工厂方法代替构造器

好处：

1. 见名知其意
2. 实例受控（即将选择复用旧对象还是创建新对象的能力还给了类本身，即单例模式、享元模式）
3. 返回原返回类型的任何子类型对象（将选择new的对象的权利返回给了类）
4. 根据参数选择返回的类， 并对客户端无感知（如：EnumSet）
5. 将客户端与其实现解耦，如：JDBC使用的服务提供者框架

坏处：

1. 因为只需要private的构造器，所以静态工厂方法的类可能无法被子类化
2. 程序员很难发现它们

常用的命名方式：

1. `from`，类型转换，如：`Date d = Date.from(instant);`
2. `of`，聚合，如：`Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);`
3. `valueOf`，更繁琐的类型转换或聚合，如：`BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);`
4. `getInstance`或`instance`，返回实例，如：`StackWalker luke = StackWalker.getInstance(options);`
5. `newInstance`，新创建并返回实例，如：`Object newArray = Arrays.newInstance(classObject, arrayLen);`
6. `getType`或`type`，返回Type类型的实例，如：`FileStore fs = Files.getFileStore(path)`
7. `newType`，新创建并返回Type类型的实例，如：`FileStore fs = Files.getFileStore(path)`

### 遇到多个构造器参数时，考虑使用build模式

参数多而杂时，推荐使用build模式，将构建实例的复杂性交给build

### 用私有构造器或者枚举类型强化Singleton属性

即Singleton类的构造器最好全部private

### 通过构造器私有化强化不可实例化的能力

即类似Math这种工具类要private构造器，推荐写上该类不允许实例化的注释，如：Don't let anyone instantiate this class

### 优先考虑依赖注入来引用资源

如果一个类依赖于其它资源，推荐使用依赖注入而不是将其写死置于工具类或者Singleton类中。

### 避免创建不必要的对象

1. 优先调用类的静态工厂方法而不是构造器来获取实例。因为静态工厂方法会管理实例的生命周期，从而防止创建不必要的对象。
2. 对于需要调用多次的正则匹配，显式的创建Pattern并缓存（从而缓存下生成的NFA状态图），而不是每次都调用`s.mathes(ps)`。
3. 对于不可变对象和已知不会被修改的对象，尽量重用。（如：`java.time.*`下的各个类的实例对象）
4. 适配器模式只是生成一个原对象的视图对象，当原对象不变时，它就不会有变化，所以要对视图对象重用（如Map接口的keySet接口生成的Set视图对象）
5. 优先使用基本类型而不是装箱类型。因为自动装箱意味着对象创建。
6. 不要试图使用对象池来维护小对象。实例化小对象的开销会小于使用对象池对它进行维护的成本

### 消除过期的对象引用

1. 只要类时自己管理内存，就应该警惕内存泄漏问题
2. 内存泄漏的另一个来源时缓存
3. 内存泄漏的第三个常见来源时监听器和其它回调

### 避免使用finalizer和cleaner方法

finalizer方法（Java9之后使用cleaner方法代替了它）只能用来充当安全网。如：`FileInputStream`、`FileOutputStream`、`ThreadPoolExecutor`等类所示。

### 对于实现了AutoCloseable接口的类，try-with-ressources优先于try-finally

## 对所有对象都通用的方法

### 重写equals时请遵守通用约定

如果类的每个实例都只与它本身相等，那么就不需要重写`equals`方法。具体场景如下：

1. 类的每个实例本质上都是唯一的。例如：`Thread`
2. 类没有必要提供“逻辑相等”的测试功能。例如：`java.util.regex.Pattern`类正常来说不会用到`equals`方法
3. 超类已经重写了`equals`方法，该逻辑对本类也适用，就没有必要再重写`equals`方法了
4. 类是私有的，或者是包级私有的，可以确定它的`equals`方法永远不会被调用

如果类具有自己特有的“逻辑相等”概念（通常该类是一种“值类”，即该类是一个表示值的类，如：`Integer`、`String`），而且超类还没有重写`equals`，那么就应该重写`equals`方法。
不过，可以通过“实例受控”在创建时确保“每个值至多存在一个对象”（如：枚举类）的“值类”无需重写`equals`方法。

在JDK中，`java.lang.Object#equals`方法上通过注释写明了“equals”方法的重写规范，如下所示：

The equals method implements an equivalence relation on non-null object references:

- It is reflexive: for any non-null reference value x, x.equals(x) should return true.
- It is symmetric: for any non-null reference values x and y, x.equals(y) should return true if and only if y.equals(x) returns true.
- It is transitive: for any non-null reference values x, y, and z, if x.equals(y) returns true and y.equals(z) returns true, then x.equals(z) should return true.
- It is consistent: for any non-null reference values x and y, multiple invocations of x.equals(y) consistently return true or consistently - return false, provided no information used in equals comparisons on the objects is modified.
- For any non-null reference value x, x.equals(null) should return false.

The equals method for class Object implements the most discriminating possible equivalence relation on objects; that is, for any non-null reference values x and y, this method returns true if and only if x and y refer to the same object (x == y has the value true).
Note that it is generally necessary to override the hashCode method whenever this method is overridden, so as to maintain the general contract for the hashCode method, which states that equal objects must have equal hash codes.

即：`equals`方法实现了等价关系，其属性如下：

- 自反性：对于任何非null的引用值x，`x.equals(x)`必须返回`true`。
- 对称性：对于任何非null的引用值x和y，当且仅当`x.equals(y)==true`时，`y.equals(x)==true`。
- 传递性：对于任何非null的引用值x、y和z，如果`x.equals(y)==true`，且`y.equals(z) == true`，则`x.equals(z) == true`。
- 一致性：对于任何非null的引用值x和y，只要`equals`的比较操作在对象中所有的信息没有被修改，多次调用`x.equals(y)`就会一致的返回`true`，或者一致的返回`false`。
- 对于任何非null的值x，`x.equals(null)`一定返回`false`。

重写`equals`方法时，有如下设计技巧：

- 使用`instanceOf`而不是`getClass()`来实现`equals`方法。这样更符合“里式替换原则”。
- 遵循“符合优于继承”的设计原则，因为我们无法在扩展实例化类（这意味着对于抽象类，这种无法实例化的父类，没有这种障碍）的同时，即增加新的值组件，又保留`equals`约定。（Java类库中的`java.sql.Timestamp`对`java.util.Date`进行了扩展，并增加了值组件`nanoseconds`，从而打破了`equals`规范，这被视为一个反面教材）。
- 不要是`equals`方法依赖于不可靠的资源，不然会很容易违反一致性原则。如：`java.net.URL`的`equals`方法依赖于对URL中主机IP地址的比较。将一个主机名转变成IP地址可能需要访问网络，随着时间的推移，就不能确保会产生相同的结果，即有可能IP地址发生了改变。可以看到在`java.net.URL#equals`方法上的注释上，写着：`Note: The defined behavior for {@code equals} is known to be inconsistent with virtual hosting in HTTP.`

综上所述，实现高质量`equals`方法有如下诀窍：

1. 使用`==`操作检查“参数是否为这个对象的引用”，这是一种性能优化。如果比较操作很昂贵，就值得这么做。
2. 使用`instanceOf`操作符检查“参数是否为正确的类型”。
3. 把参数转换成正确的类型。即之前`instanceOf`测试的类型。
4. 对于该类中的每一个“关键”域，检查参数中的域是否与该对象中对应的域相匹配。
5. 重写`equals`时总要重写`hashCode`。
6. 不要企图让`equals`过于智能。
7. 不要将`equals`方法的参数声明中的`Object`类型替换为其它类型。

### 重写equals时总要重写hashCode

The general contract of hashCode is:

- Whenever it is invoked on the same object more than once during an execution of a Java application, the hashCode method must consistently return the same integer, provided no information used in equals comparisons on the object is modified. This integer need not remain consistent from one execution of an application to another execution of the same application.
- If two objects are equal according to the equals(Object) method, then calling the hashCode method on each of the two objects must produce the same integer result.
- It is not required that if two objects are unequal according to the equals(Object) method, then calling the hashCode method on each of the two objects must produce distinct integer results. However, the programmer should be aware that producing distinct integer results for unequal objects may improve the performance of hash tables.

As much as is reasonably practical, the hashCode method defined by class Object does return distinct integers for distinct objects. (This is typically implemented by converting the internal address of the object into an integer, but this implementation technique is not required by the Java™ programming language.)

即：

1. 在应用程序的执行期间，只要对象的`equals`方法的比较操作所用到的信息没有被修改，那么对同一个对象的多次调用，`hashCode`必须始终返回同一个值。在一个应用程序与另一个程序的执行过程中，执行`hashCode`方法所返回的值可以不一致。
2. 如果两个对象根据`equals(Object)`方法比较是相等的，那么调用这两个对象中的`hashCode`方法都必须产生同样的整数结果。
3. 如果两个对象根据`equals(Object)`方法比较是不想等的，那么调用这两个对象中的`hashCode`方法，则不一定要求`hashCode`必须产生不同的结果。但是程序员应该知道，给不想等的对象产生截然不同的整数结果，有可能提高散列表的性能。

如果重写`equals`而不重写`hashCode`方法，则很容易违反第2条规范。相等的对象必须有相等的散列值。

可以调用`java.util.Objects.hash(Object... values)`，或者参考它自己实现。

- 在散列码的计算过程中，可以把衍生域（derived field）排除在外。
- 如果一个类是不可变的，并且计算散列码的开销也比较大，就应该考虑把散列码缓存在对象内部，而不是每次请求的时候都重新计算散列码。
- 不要试图从散列码计算中排除掉一个对象的关键域来提高性能。
- 不要向`String`、`Integer`等类一样为其`hashCode`函数通过方法注释指明了实现方式，这会使得你无法再更改其实现，因为调用方可能只是按注释写明的算法而不是`hashCode`本身的作用去编写的调用代码。

### 始终要重写toString

`Object`的`toString`方法有如下注释：

Returns a string representation of the object. In general, the toString method returns a string that "textually represents" this object. The result should be a concise but informative representation that is easy for a person to read. It is recommended that all subclasses override this method.

其建议子类全部重写`toString`方法，被返回的字符串应该是一个“简介单信息丰富，并且易于阅读的表达形式”。

- 在实际应用中，`toString`方法应该返回对象中包含的所有值得关注的信息。如果对象太大，或者对象中包含的状态信息难以用字符串来表达，则应该返回一个摘要信息。
- 对于值类，建议在文档中指定返回值的格式。通常最好再提供一个相匹配的静态工厂或者构造器，以便程序员可以很容易地在对象及其字符串表示法之间来回转换。Java平台类库中的许多值都采用了这种做法，比如：`BigInteger`、`BigDecimal`和绝大多数的基本类型包装类。
- 无论是否决定指定格式，都应该在文档中明确地表明你的意图。如写明：The exact details of the representation are unspecified and subject to change，表明当前toString方法还没有规范的返回值格式，当前的格式只代表当前的定义，后面可能会出现修改。
- 在静态工具类中编写`toString`方法是没有意义的。也不要在大多数枚举类型中编写`toString`方法，因为Java已经提供了非常完美的方法。在所有的子类共享的通用字符串表示法的抽象类中，一定要编写一个`toString`方法。例如：大多数集合实现中的`toString`方法都继承自抽象的集合类。

### 谨慎的重写clone方法
