# kotlin使用`@ConfigurationProperties`集成`spring-boot-configuration-processor`的配置

除了使用`@Value`单个指定与环境中的属性值绑定外。springboot还提供了一种批量的指定方式`@ConfigurationProperties`注解。

通过该注解的`prefix`属性，指定该类型中的所有字段绑定的属性名前缀。

```Java
@Setter
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
    private String dataFormat;
}
```

`WebMvcProperties`类使用了`@ConfigurationProperties`注解，其中的`dataFormat`字段会自动的绑定环境中的`spring.mvc.data-format`属性（springboot默认使用模糊匹配原则，可以匹配`data_format`、`dataFormat`等多种情况）

spring要求类WebMvcProperties必须是一个pojo，因而kotlin类要对应的能够在编译后生成一个pojo。[springboot官方文档](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/boot-features-kotlin.html)(参见其48.5 @ConfigurationProperties章节)推荐使用`lateinit var`的声明方式。

因而对于上面的例子，可以使用如下实现：

```Kotlin
@ConfigurationProperties(prefix = "spring.mvc")
class WebMvcProperties {
    private lateinit var dataFormat: String
}
```

我们可以使用`spring-boot-configuration-processor`进行自动补全

```Maven
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <version>${org.springframework.boot.version}</version>
</dependency>
```

只需在maven中加入`spring-boot-configuration-processor`依赖，idea就可以使用`target/classes/META-INF/spring-configuration-metadata.json`文件中配置的相关映射对`application.properties`等环境配置文件中的key进行代码提示。

而`target/classes/META-INF/spring-configuration-metadata.json`文件，我们可以依靠kotlin的`kapt`进行生成。

对`pom.xml`中的`kotlin-maven-plugin`插件配置进行修改，在`compile`的execution前加入如下配置。让其kapt插件使用`spring-boot-configuration-processor`进行处理。

```Maven
<execution>
    <id>kapt</id>
    <goals>
        <goal>kapt</goal>
    </goals>
    <configuration>
        <sourceDirs>
            <sourceDir>src/main/kotlin</sourceDir>
            <sourceDir>src/main/java</sourceDir>
        </sourceDirs>
        <annotationProcessorPaths>
            <!-- Specify your annotation processors here. -->
            <annotationProcessorPath>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-configuration-processor</artifactId>
                <version>${org.springframework.boot.version}</version>
            </annotationProcessorPath>
        </annotationProcessorPaths>
    </configuration>
</execution>
```

我们需要使用maven的编译命令对项目进行编译，而不是使用idea编译系统。

kotlin的[官方文档](https://kotlinlang.org/docs/reference/kapt.html)如下所示，说明了这个问题
> Please note that kapt is still not supported for IntelliJ IDEA’s own build system. Launch the build from the “Maven Projects” toolbar whenever you want to re-run the annotation processing.

具体操作步骤为在`Project`视图中右击该项目选择`Run Maven`，选择`compile`或者`install`等会间接调用`compile`的命令，执行后，就会发现`target`中生成了相应的json映射文件。

`spring-boot-configuration-processor`采用的搜索spring组件的方式。因而：必须要是spring容器管理的`@ConfigurationProperties`注解标识的类，才会生成其相关信息到json映射文件中。

为此，我们可以有3种方式把类交给spring容器：

1. 使用`@component`类的注解，将该类直接声明为组件
2. 在某个spring组件的方法上使用`@Bean`注解，该方法返回一个该类的实例对象
3. 在springboot的配置上加入`@EnableConfigurationProperties`注解，使springboot自动扫描全部被`@ConfigurationProperties`声明的类，并将其转化为组件
