# 创建一个简单的maven模型(archetype)

maven提供了多种方式去创建一个新的原型，下面，我们演示如何通过一个存在的maven项目生成原型

- [创建一个简单的maven模型(archetype)](#创建一个简单的maven模型archetype)
  - [使用一个存在的项目生成一个maven原型](#使用一个存在的项目生成一个maven原型)

## 使用一个存在的项目生成一个maven原型

让我们先创建一个创建一个原型项目用来作为创建maven原型的种子，该原型是一个使用`Servlet 4.0`的web原型

```SHELL
mvn archetype:generate -DarchetypeArtifactId=maven-archetype-webapp -DgroupId=com.apress.gswmbook -DartifactId=gswm-web-prototype -DinteractiveMode=false
```

执行如上命令后，我们可以创建一个名为`gswm-web-prototype`的简单web项目。之后，修改`web.xml`文件

```SHELL
cd gswm-web-prototype
vim src/main/webapp/WEB-INF/web.xml
```

将该文件内容改为如下所示：

```XML
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="4.0" xmlns="http://xmlns.jcp.org/xml/ns/
javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd">
    <display-name>Archetype Created Web Application</display-­name>
</web-app>
```

之后，我们修改`pom.xml`文件，加入`servlet 4.0`的依赖和使用嵌入式容器启动项目的插件（此处使用的嵌入式容器为Jetty）：

```SHELL
vim pom.xml
```

```XML
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.apress.gswmbook</groupId>
    <artifactId>gswm-web</artifactId>
    <packaging>war</packaging>
    <version>1.0-SNAPSHOT</version>
    <name>gswm-web Maven Webapp</name>
    <url>http://maven.apache.org</url>
    <dependencies>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.1</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
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
</project>
```

最后，我们在项目中创建`java`和`test`目录，并添加一个路由为`/status`，并会返回`200`的servlet.

```SHELL
mkdir -p src/main/java
mkdir -p src/test/java
mkdir -p src/test/resources

mkdir -p src/main/java/com/apress/gswmbook/web/servlet
vim src/main/java/com/apress/gswmbook/web/servlet/AppStatusServlet.java
```

```Java
package com.apress.gswmbook.web.servlet;
import javax.servlet.annotation.WebServlet;
import javax.servlet.∗;
import javax.servlet.http.∗;
import java.io.∗;
@WebServlet("/status")
public class AppStatusServlet extends HttpServlet {
    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        PrintWriter writer = response.getWriter();
        writer.println("OK");
        response.setStatus(response.SC_OK);
    }
}
```

最后，我们通过执行`archetype`插件的`create-from-project`目标，来从该项目生成我们的maven原型，其命令为：`mvn archetype:create-from-project`

如果执行完成，我们可以在最后看到`Archetype created in ${project_home}/target/generated-sources/archetype`的提示（其中`${project_home}`），`${project_home}/target/generated-sources/archetype`即为我们新生成的原型。

为了对生成原型进行更改，实现我们的需求，我们将`archetype`中需要的部分移动到一个新的文件夹（项目）中

```SHELL
mkdir gswm-web-archetype
cp -r archetype/src gswm-web-archetype/
cp archetype/pom.xml gswm-web-archetype/
cd gswm-web-archetype
```

修改`${project_home}/src/main/resources/archetype-resources`下的`pom.xml`文件，将`<finalName/>`属性修改为`${artifactId}`,其命令为`vim src/main/resources/archetype-resources/pom.xml`

从原型创建项目时，maven会提示用户输入程序包名称。它将在新创建的项目的`src/main/java`文件夹下创建与包相对应的目录。然后，它将原型的`archetype-resources/src/main/java`文件夹下的所有内容移动到该包中。

因为，我们希望`AppStatusServlet`在子包`web.servlet`下，所以，我们需要创建`web/servlet`目录，并将`AppStatusServlet.java`移动到该目录，然后打开``AppStatusServlet.java`，将`${package}`改为`${package}.web.servlet`

```SHELL
cd src/main/resources/archetype-resources/src/main/java
mkdir -p web/servlet
mv AppStatusServlet.java web/servlet/
vim web/servlet/AppStatusServlet.java
```

最后，我们在`gswm-web-archetype`目录下执行`mvn clean install`在本地仓库安装该原型

我们可以使用`mvn archetype:generate -DarchetypeCatalog=local`来让`archetype`插件只给出本地原型。

```SHELL
Choose archetype:
1: local -> com.apress.gswmbook:gswm-web-archetype (gswm-web-archetype)
Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): : 1
Define value for property 'groupId': com.apress.gswmbook
Define value for property 'artifactId': test-project
Define value for property 'version' 1.0-SNAPSHOT: : <<Hit Enter>>
Define value for property 'package' com.apress.gswmbook: : <<Hit Enter>>
Confirm properties configuration:
groupId: com.apress.gswmbook
artifactId: test-project
version: 1.0-SNAPSHOT
package: com.apress.gswmbook
 Y: : <<Hit Enter>>
```
