# SystemInfoPlugin插件开发示例

首先参看[基于maven构建一个简单的Java项目](./基于Maven构建一个简单的Java项目.md)，创建一个名为`gswm-maven-plugin`的maven项目

项目结构如下所示：

- gswm-maven-plugin
  - src
    - main
      - java
        - com
          - apress
            - plugins
              - SystemInfoMojo.java
  - pom.xml

linux下的创建命令如下：

```SHELL
mkdir -p gswm-maven-plugin/src/main/java/com/apress/plugins
cd gswm-maven-plugin
cat /dev/null >> pom.xml
cat /dev/null >> src/main/java/com/apress/plugins/SystemInfoMojo.java
```

在`pom.xml`文件中设置打包方式为`maven-plugin`，添加`maven-plugin-api`和`maven-plugin-annotations`依赖用来支持插件开发，添加`commons-lang3`依赖用来获取系统信息。

`pom.xml`文件内容如下：

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.apress.plugins</groupId>
    <artifactId>gswm-maven-plugin</artifactId>
    <version>1.0.0</version>
    <packaging>maven-plugin</packaging>
    <description>System Info Plugin</description>
    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-plugin-api</artifactId>
            <version>3.6.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.plugin-tools</groupId>
            <artifactId>maven-plugin-annotations</artifactId>
            <version>3.6.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.9</version>
        </dependency>
    </dependencies>
    <!-- Use the latest version of Plugin -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-plugin-plugin</artifactId>
                <version>3.6.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

下一步，我们需要创建一个名为`systeminfo`的goal。它的创建逻辑在`src/main/java/com/apress/plugins/SystemInfoMojo.java`文件中定义：

```SHELL
vim src/main/java/com/apress/plugins/SystemInfoMojo.java
```

```Java
package com.apress.plugins;
import org.apache.commons.lang3.SystemUtils;
import org.apache.maven.plugin.AbstractMojo;
import org.apache.maven.plugin.MojoExecutionException;
import org.apache.maven.plugin.MojoFailureException;
import org.apache.maven.plugins.annotations.Mojo;

// @Mojo声明该类是一个MOJO,该注解的name属性用来指定该goal的名称
@Mojo( name = "systeminfo")
public class SystemInfoMojo extends AbstractMojo {

    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {
        getLog().info( "Java Home: " + SystemUtils.JAVA_HOME );
        getLog().info( "Java Version: "+ SystemUtils.JAVA_VERSION);
        getLog().info( "OS Name: " + SystemUtils.OS_NAME );
        getLog().info( "OS Version: " + SystemUtils.OS_VERSION );
        getLog().info("User Name: " + SystemUtils.USER_NAME );
    }
}
```

最后，我们使用`mvn install`将该插件安装到本地maven仓库。

我们有两种方式使用该插件：

1. 直接使用`mvn com.apress.plugins:gswm-maven-plugin:systeminfo`来运行该插件
2. 在某个项目的`pom.xml`中指定某个phase执行该插件。如下所示：

修改[基于Maven构建简单的Java项目](./基于Maven构建一个简单的Java项目.md)中`gswm`项目的`pom.xml`文件，为其增加`build`元素。

```XML
<build>
    <plugins>
        <plugin>
            <groupId>com.apress.plugins</groupId>
            <artifactId>gswm-maven-plugin</artifactId>
            <version>1.0.0</version>
            <executions>
                <execution>
                <phase>validate</phase>
                <goals>
                    <goal>systeminfo</goal>
                </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

在`gswm`项目中执行`validate`(命令为：`mvn validate`)或其之后的过程，如：`compile`(命令为：`mvn compile`)，均会看到`systeminfo`目标的打印结果。
