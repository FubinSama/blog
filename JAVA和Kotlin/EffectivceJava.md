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
    - [覆盖equals时请遵守通用约定](#覆盖equals时请遵守通用约定)

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

### 覆盖equals时请遵守通用约定

如果类的每个实例都只与它本身相等，那么就不需要覆盖`equals`方法。具体场景如下：

1. 类的每个实例本质上都是唯一的。例如：`Thread`
2. 类没有必要提供“逻辑相等”的测试功能。例如：`java.util.regex.Pattern`类正常来说不会用到`equals`方法
3. 超类已经覆盖了`equals`方法，该逻辑对本类也适用，就没有必要再覆盖`equals`方法了
4. 类是私有的，或者是包级私有的，可以确定它的`equals`方法永远不会被调用
