# MyBatis

> 一个不错的Mybatis源码阅读系列：`https://juejin.cn/post/6844904127462375438`

MyBatis 是一款优秀的**持久层框架**，它支持自定义 SQL、存储过程以及高级映射。MyBatis **免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作**。MyBatis 可以**通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录**。

## 入门

```java
final String resource = "com/wfb/learn/mybatis/t1/mybatis-config.xml"; // mybatis的配置文件路径
InputStream inputStream = Resources.getResourceAsStream(resource);
// ① 通过SqlSessionFactoryBuilder和配置文件流创建SqlSessionFactory
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream); 
// ② 通过SqlSessionFactory创建SqlSession
try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
    // ③ 通过SqlSession创建相对应的Mapper对象，并调用相关的sql语句。
    final BlogMapper mapper = sqlSession.getMapper(BlogMapper.class);
    final Blog blog = mapper.selectBlog(101);
}
```

①：调用的是`SqlSessionFactoryBuilder#build(InputStream inputStream)`，实际上调用的是`SqlSessionFactoryBuilder#build(InputStream inputStream, String environment, Properties properties)`，可以看到主要步骤是：

1. 通过`inputStream`、`environment`、`properties`构建`XMLConfigBuilder`对象`parser`。
2. 调用`parser.parse()`方法生成`org.apache.ibatis.session.Configuration`配置对象config
3. `return new DefaultSqlSessionFactory(config);`

②：调用的是`DefaultSqlSessionFactory#openSession()`方法，即执行了`return new DefaultSqlSession(configuration, executor, autoCommit);`，其中`executor`和`autoCommit`依赖于配置。`executor`默认为`SimpleExecutor`，`autoCommit`默认为`false`。

③：调用的是`DefaultSqlSession#getMapper(Class<T> type)`，即`MapperRegistry#getMapper(Class<T> type, SqlSession sqlSession)`方法。可以发现基本逻辑是创建了一个`MapperProxyFactory`，并调用了其`newInstance`方法。即**使用`java.lang.reflect.Proxy#newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)`方法创建了`type`接口的代理类**。通过`MapperProxy#invoke(Object proxy, Method method, Object[] args)` -> `MapperProxy#cachedInvoker(Method method)` -> `return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));` -> `MapperMethod#execute(SqlSession sqlSession, Object[] args)`路径，可以找到该代理类的基本逻辑。即：根据`command.getType()`的类型选择调用不同的`sqlSession`方法。如：`INSERT`对应调用的`SimpleExecutor#doUpdate(MappedStatement ms, Object parameter)`方法。最终会定位到`SimpleExecutor#prepareStatement(StatementHandler handler, Log statementLog)`，再往内走，就会发现实际上是使用的`jdk`自带的`jdbc`的`prepareStatement`来执行的sql。

### 作用域（Scope）和生命周期

#### SqlSessionFactoryBuilder

这个类可以被实例化、使用和丢弃，一旦创建了 SqlSessionFactory，就不再需要它了。 因此 **SqlSessionFactoryBuilder 实例的最佳作用域是方法作用域**（也就是局部方法变量）。 你可以重用 SqlSessionFactoryBuilder 来创建多个 SqlSessionFactory 实例，但最好还是不要一直保留着它，以保证所有的 XML 解析资源可以被释放给更重要的事情。

#### SqlSessionFactory

SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例。 使用 SqlSessionFactory 的最佳实践是在应用运行期间不要重复创建多次，多次重建 SqlSessionFactory 被视为一种代码“坏习惯”。因此 **SqlSessionFactory 的最佳作用域是应用作用域**。 有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式。

#### SqlSession

每个线程都应该有它自己的 SqlSession 实例。**SqlSession 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域**。 绝对**不能将 SqlSession 实例的引用放在一个类的静态域，甚至一个类的实例变量也不行。 也绝不能将 SqlSession 实例的引用放在任何类型的托管作用域中，比如 Servlet 框架中的 HttpSession**。 如果你现在正在使用一种 Web 框架，考虑将 SqlSession 放在一个和 HTTP request相似的作用域中。 换句话说，每次收到 HTTP 请求，就可以打开一个 SqlSession，返回一个响应后，就关闭它。 这个关闭操作很重要，为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 finally 块中。 下面的示例就是一个确保 SqlSession 关闭的标准模式：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
    // 你的应用逻辑代码
}
```

#### 映射器实例

映射器是一些绑定映射语句的接口。映射器接口的实例是从 SqlSession 中获得的。虽然**从技术层面上来讲，任何映射器实例的最大作用域与请求它们的 SqlSession 相同。但方法作用域才是映射器实例的最合适的作用域**。 也就是说，映射器实例应该在调用它们的方法中被获取，使用完毕之后即可丢弃。 映射器实例并不需要被显式地关闭。尽管在整个请求作用域保留映射器实例不会有什么问题，但是你很快会发现，在这个作用域上管理太多像 SqlSession 的资源会让你忙不过来。 因此，最好将映射器放在方法作用域内。就像下面的例子一样：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
    BlogMapper mapper = session.getMapper(BlogMapper.class);
    // 你的应用逻辑代码
}
```

## 配置

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <!--
    这是配置文件的基本模板，完整配置详见：https://mybatis.org/mybatis-3/zh/configuration.html
    -->

    <!--
    如果一个属性在不只一个地方进行了配置，那么，MyBatis 将按照下面的顺序来加载：
    1. 首先读取在 properties 元素体内指定的属性。
        即：<property name="username" value="${mysql_username}"/>
    2. 然后根据 properties 元素中的 resource 属性读取类路径下属性文件，或根据 url 属性指定的路径读取属性文件，并覆盖之前读取过的同名属性。
        即：<properties resource="./mybatis-config.properties">
    3. 最后读取作为方法参数传递的属性，并覆盖之前读取过的同名属性。
        即：`SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, props);`中的props属性
    -->
    <properties resource="./mybatis-config.properties">
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/test"/>
        <property name="username" value="username"/>
        <property name="password" value="password"/>
        <!-- 启用默认值特性，默认值的分隔符为：':' -->
        <property name="org.apache.ibatis.parsing.PropertyParser.enable-default-value" value="true"/>
        <!--
        修改默认值的分隔符为：'?:'
        <property name="org.apache.ibatis.parsing.PropertyParser.default-value-separator" value="?:"/>
        -->
    </properties>

    <!--
    这是 MyBatis 中极为重要的调整设置，它们会改变 MyBatis 的运行时行为
    -->
    <settings>
        <setting name="useGeneratedKeys" value="true"/>
    </settings>

    <!--
    类型别名可为 Java 类型设置一个缩写名字。 它仅用于 XML 配置，意在降低冗余的全限定类名书写。
    -->
    <typeAliases>
        <typeAlias alias="Author" type="com.wfb.learn.mybatis.t2.Author"/>
        <!--
        也可以指定一个包名，MyBatis 会在包名下面搜索需要的 Java Bean，比如：
        <package name="domain.blog"/>
        每一个在包 domain.blog 中的 Java Bean，在没有注解的情况下，会使用 Bean 的首字母小写的非限定类名来作为它的别名。
        比如 domain.blog.Author 的别名为 author；若有注解，则别名为其注解值。
        ```java
        @Alias("author")
        public class Author {
            ...
        }
        ```
        -->
    </typeAliases>

    <!--
    MyBatis 在设置预处理语句（PreparedStatement）中的参数或从结果集中取出一个值时， 都会用类型处理器将获取到的值以合适的方式转换成 Java 类型。
    -->
    <typeHandlers>
        <!--
        <typeHandler handler="org.mybatis.example.ExampleTypeHandler"/>
        -->
    </typeHandlers>

    <!--
    MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：
    1. Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
    2. ParameterHandler (getParameterObject, setParameters)
    3. ResultSetHandler (handleResultSets, handleOutputParameters)
    4. StatementHandler (prepare, parameterize, batch, update, query)
    -->
    <plugins>
        <plugin interceptor=""/>
    </plugins>

    <environments default="development"> <!--默认使用的环境 ID-->
        <environment id="development"> <!--每个 environment 元素定义的环境 ID-->
            <transactionManager type="JDBC"/> <!--事务管理器的配置,在 MyBatis 中有两种类型的事务管理器（也就是 type="[JDBC|MANAGED]")-->
            <dataSource type="POOLED"> <!--数据源的配置,有三种内建的数据源类型（也就是 type="[UNPOOLED|POOLED|JNDI]"）-->
                <!-- 占位符功能，如果driver未被指定，则使用默认值com.mysql.cj.jdbc.Driver -->
                <property name="driver" value="${driver:com.mysql.cj.jdbc.Driver}"/>
                <property name="url" value="${url}"/>
                <property name="username" value="${username}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>

    <!--
    MyBatis 可以根据不同的数据库厂商执行不同的语句，这种多厂商的支持是基于映射语句中的 databaseId 属性。
    -->
    <databaseIdProvider type="DB_VENDOR">
        <property name="SQL Server" value="sqlserver"/>
        <property name="DB2" value="db2"/>
        <property name="Oracle" value="oracle" />
    </databaseIdProvider>

    <!--
    映射器（mappers）
    -->
    <mappers>
        <mapper resource="com/wfb/learn/mybatis/t1/BlogMapper.xml"/>
    </mappers>

</configuration>
```

### XML 映射器

SQL 映射文件只有很少的几个顶级元素（按照应被定义的顺序列出）：

- `cache` – 该命名空间的缓存配置。
- `cache-ref` – 引用其它命名空间的缓存配置。
- `resultMap` – 描述如何从数据库结果集中加载对象，是最复杂也是最强大的元素。
- `sql` – 可被其它语句引用的可重用语句块。
- `insert` – 映射插入语句。
- `update` – 映射更新语句。
- `delete` – 映射删除语句。
- `select` – 映射查询语句。

### select

```xml
<select
  id="selectPerson"
  parameterType="int"
  resultType="hashmap"
  resultMap="personResultMap"
  flushCache="false"
  useCache="true"
  timeout="10"
  fetchSize="256"
  statementType="PREPARED"
  resultSetType="FORWARD_ONLY">
```

- `id`：在命名空间中唯一的标识符，可以被用来引用这条语句。
- `parameterType`：将会传入这条语句的参数的类全限定名或别名。**这个属性是可选的**，因为 MyBatis **可以通过类型处理器（TypeHandler）推断出具体传入语句的参数**，默认值为未设置（unset）。
- `resultType`：期望从这条语句中返回结果的类全限定名或别名。 注意，**如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型**。 resultType 和 resultMap 之间只能同时使用一个。
- `resultMap`：对外部 resultMap 的命名引用。
- `flushCache`：将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：false。
- `useCache`：将其设置为 true 后，将会导致本条语句的结果被二级缓存缓存起来，默认值：对 select 元素为 true。
- `timeout`：这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。
- `fetchSize`：这是一个给驱动的建议值，尝试让驱动程序每次批量返回的结果行数等于这个设置值。 默认值为未设置（unset）（依赖驱动）。
- `statementType`：可选 `STATEMENT`，`PREPARED` 或 `CALLABLE`。这会让 `MyBatis` 分别使用 `Statement`（jdbc执行静态sql），`PreparedStatement`（jdbc预编译sql语句） 或 `CallableStatement`（jdbc调用储存过程），默认值：`PREPARED`。
- `resultSetType`：`FORWARD_ONLY`（结果集的游标只能向下滚动），`SCROLL_SENSITIVE`（返回可滚动的结果集，当数据库变化时，当前结果集同步改变）, `SCROLL_INSENSITIVE`（结果集的游标可以上下移动，当数据库变化时，当前结果集不变） 或 `DEFAULT`（等价于`unset`） 中的一个，默认值为 `unset` （依赖数据库驱动）。在JDBC规范中,resultSetType的默认取值为`TYPE_FORWARD_ONLY`。
- `databaseId`：如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。
- `resultOrdered`：这个设置仅针对嵌套结果 select 语句：如果为 true，将会假设包含了嵌套结果集或是分组，当返回一个主结果行时，就不会产生对前面结果集的引用。 这就使得在获取嵌套结果集的时候不至于内存不够用。默认值：false。详见：test中的`org.apache.ibatis.submitted.nestedresulthandler.NestedResultHandlerTest#testGetPerson`方法。
- `resultSets`：这个设置仅适用于多结果集的情况。它将列出语句执行后返回的结果集并赋予每个结果集一个名称，多个名称之间以逗号分隔。详见：test中的`org.apache.ibatis.submitted.sptests.SPTest#testGetNamesAndItems`方法。
