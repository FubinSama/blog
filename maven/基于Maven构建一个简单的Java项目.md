# 基于Maven构建简单的Java项目

- [基于Maven构建简单的Java项目](#基于maven构建简单的java项目)
  - [手动构建](#手动构建)
    - [1. 创建项目结构](#1-创建项目结构)
    - [2. 完善maven配置](#2-完善maven配置)
    - [3. 编码](#3-编码)
    - [4. 打包项目，生成jar包](#4-打包项目生成jar包)
    - [5. 执行jar文件](#5-执行jar文件)
    - [6. 添加单元测试](#6-添加单元测试)
  - [基于maven原型构建一个web项目](#基于maven原型构建一个web项目)

## 手动构建

### 1. 创建项目结构

项目结构如下所示：

- gswm
  - src
    - main
      - java
  - pom.xml

在linux可以使用如下`shell`命令：

```SHELL
mkdir -p gswm/src/main/java;
cd gswm;
cat /dev/null > pom.xml
```

### 2. 完善maven配置

```SHELL
vim pom.xml
```

在`pom.xml`文件中添加如下内容，详见[pom文件详解](maven/pom文件详解)查看pom文件介绍：

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
</project>
```

### 3. 编码

在`src/main/java`下编写程序代码。

如下步骤创建一个简单的`HelloWord`程序

```SHELL
vim src/main/java/HelloWorld.java
```

在`HelloWord.java`下填入如下内容

```Java
public class HelloWorld {
    public void sayHello() {
        System.out.println("Hello World");
    }
    public static void main(String[] args) {
        HelloWorld hw = new HelloWorld();
        hw.sayHello();
    }
}
```

### 4. 打包项目，生成jar包

在项目根目录（即gswm路径）下执行`mvn package`。

ps：第一次构建项目时，maven会自动从远程仓库中下载本地仓库不存在的项目依赖。因而，第一次构建可能会很慢。

### 5. 执行jar文件

```SHELL
java -cp target/gswm-1.0.0-SNAPSHOT.jar HelloWorld
```

### 6. 添加单元测试

创建单元测试文件夹

```SHELL
mkdir -p src/test/java
```

修改`pom.xml`文件，添加`JUnit`依赖

```SHELL
vim pom.xml
```

将`pom.xml`文件修改为如下：

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

    <!-- 新加的依赖 -->
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

</project>
```

在`src/test/java`下添加相应的单元测试代码。如下示例展示了调用`HelloWord`的`sayHello`方法的单元测试。

```SHELL
vim src/test/java/HelloWorldTest.java
```

添加如下代码：

```Java
import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import org.junit.After;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;

public class HelloWorldTest {
    private final ByteArrayOutputStream outStream = new ByteArrayOutputStream();
    @Before
    public void setUp() {
        System.setOut(new PrintStream(outStream));
    }
    @Test
    public void testSayHello() {
        HelloWorld hw = new HelloWorld();
        hw.sayHello();
        Assert.assertEquals("Hello World", outStream.toString());
    }
    @After
    public void cleanUp() {
        System.setOut(null);
    }
}
```

重新执行`mvn package`命令，可以发现maven在打包前执行了单元测试，并报告了执行结果

## 基于maven原型构建一个web项目

使用`mvn archetype:generate -DarchetypeArtifactId=maven-archetype-­webapp`利用交互模式创建项目：

```SHELL
Define value for property 'groupId': : com.apress.gswmbook
Define value for property 'artifactId': : gswm-web
Define value for property 'version': 1.0-SNAPSHOT: :  
<<Hit Enter>>
Define value for property 'package': com.apress.gswmbook: : war
Confirm the properties configuration:
groupId: com.apress.gswmbook
artifactId: gswm-web
version: 1.0-SNAPSHOT
package: war
Y: <<Hit Enter>>
```

刚创建的web项目的`pom.xml`文件中，只添加了`Junit`依赖。为了能够使用maven运行，我们需要嵌入式容器（如：Tomcat、Jetty）插件。下面是添加`Jetty`插件的步骤：

```SHELL
cd gswm-web
vim pom.xml
```

在`<build />`元素中添加`plugins`

```XML
<build>
    <finalName>gswm-web</finalName>
    <plugins>
        <plugin>
            <groupId>org.eclipse.jetty</groupId>
            <artifactId>jetty-maven-plugin</artifactId>
            <version>9.4.12.RC2</version>
        </plugin>
    </plugins>
</build>
```

之后，我们可以使用`mvn jetty:run`运行`jetty`插件的`run`目标，从而启动该web项目。浏览器访问`http://localhost:8080/`可以看到`Hello World`欢迎页
