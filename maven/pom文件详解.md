# pom文件详解

## maven为pom文件提供的隐式属性

maven暴露它的pom（Project Object Model）属性到`project.`前缀。如：可以使用`${project.artifactId}`来取得`<artifactId></artifactId>`中设置的值

如果想访问`settings.xml`中定义的属性，可以使用`settings.`前缀。

如果想访问环境变量中定义的属性，可以使用`env.`前缀。如：`env.PATH`可以获取`PATH`环境变量中的值。

## 在maven中自定义属性

可以在pom文件的`<properties></properties>`中定义用户自定义属性。如下所示：

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <!-- 该项目的GAV -->
    <groupId>com.apress.gswmbook</groupId>
    <artifactId>gswm</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <!-- 项目的打包类型 -->
    <packaging>jar</packaging>
    <!-- 项目名 -->
    <name>gswm</name>
    <!-- 项目描述 -->
    <description>Getting Started with Maven</description>

    <properties>
        <junit.version>4.12</junit.version>
    </properties>

    <!-- 开发者信息 -->
    <developers>
    <developer>
        <id>wfb</id>
        <name>Fubin Wang</name>
        <email>fubinsama@qq.com</email>
        <properties>
            <active>true</active>
        </properties>
    </developer>
    </developers>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

</project>
```

## 配置插件(plugin)的行为

以compiler插件为例，让其将项目编译为Java8，而不是默认的Java6

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <!-- Project details omitted for brevity -->
    <dependencies>
        <!-- Dependency details omitted for brevity -->
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

ps：`<build />`元素有一个`<finalName />`属性，可以用它来定义生成的artifact的文件名。默认情况下是：`<<project_artifiact_id>>-<<project_version>>`
