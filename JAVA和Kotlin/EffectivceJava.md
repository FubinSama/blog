# Effective Java笔记

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

###