# 创建项目站点和Javadoc报告

- [创建项目站点和Javadoc报告](#创建项目站点和javadoc报告)
  - [基于maven创建项目站点](#基于maven创建项目站点)
    - [通过配置`pom.xml`文件，对site进行配置](#通过配置pomxml文件对site进行配置)
    - [高级配置](#高级配置)
  - [生成项目的Javadoc报告](#生成项目的javadoc报告)
    - [生成单元测试报告](#生成单元测试报告)
    - [生成代码覆盖率报告](#生成代码覆盖率报告)
    - [生成SpotBugs报告](#生成spotbugs报告)

## 基于maven创建项目站点

### 通过配置`pom.xml`文件，对site进行配置

我们以为`gswm`项目(详见[基于Maven构建简单的Java项目](maven/基于Maven构建一个简单的Java项目)生成site站点为例，介绍如何使用`maven site`生命周期创建项目的站点

首先，我们修改`maven-site-plugin`和`maven-project-info-reports-plugin`插件的版本。为此，我们要在`pom.xml`的`<build />`元素中加入相应的配置：

```SHELL
cd gswm
vim pom.xml
```

```XML
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-site-plugin</artifactId>
            <version>3.8.2</version>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-project-info-reports-plugin</artifactId>
            <version>3.1.1</version>
        </plugin>
    </plugins>
</build>
```

之后，我们通过执行`mvn clean site`生命周期生成项目的站点。生成的站点在`${project_dir}/target/site`目录下。我们可以通过在浏览器下运行`index.html`文件，打开该站点。

maven允许我们在`pom.xml`文件中为site配置更多的信息。如：`<description />`元素用来配置`About`中展示的信息;`<mailingLists />`元素用来配置`Mailing Lists`;`<licenses />`元素用来配置`Licenses`。

```SHELL
vim pom.xml
```

```XML
<description>
    This project acts as a starter project for the Introducing
    Maven book (http://www.apress.com/9781484208427) published
    by Apress.
</description>

<mailingLists>
    <mailingList>
        <name>GSWM Developer List</name>
        <subscribe>gswm-dev-subscribe@apress.com</subscribe>
        <unsubscribe>gswm-dev-unsubscribe@apress.com</unsubscribe>
        <post>developer@apress.com</post>
    </mailingList>
</mailingLists>

<licenses>
    <license>
        <name>Apache License, Version 2.0</name>
        <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
    </license>
</licenses>
```

之后，重新执行`mvn clean site`，发现生成的site中包含了`pom.xml`文件中的信息

### 高级配置

为了防止`pom.xml`文件规模的膨胀和难以维护，maven提供了site站点的高级配置方式，允许您在适当命名的`src/site`文件夹下指定用于网站生成的内容和配置。
下面是一个简单的site设置的目录结构：

- site
  - apt
    - index.apt
  - site.xml

`site.xml`文件，也称为站点描述符，用于自定义生成的站点。`apt`文件夹包含以`APT(Almost Plain Text)`格式编写的网站内容，[APT](http://maven.apache.org/doxia/references/apt-format.html)格式允许以类似于纯文本的语法创建文档。除了APT，Maven还支持其他格式，例如`FML`，`Xdoc`和Markdown。

maven提供了多个原型来让我们自动生成站点结构。我们可以通过如下命令使用maven原型为`gswm`项目生成站点高级配置的目录结构。

```SHELL
cd gswm
mvn archetype:generate -DarchetypeGroupId=org.apache.maven.archetypes -DarchetypeArtifactId=maven-archetype-site-simple -DarchetypeVersion=1.4 -DgroupId=com.apress.gswmbook -DartifactId=gswm -Dversion=1.0.0-SNAPSHOT
```

正确执行该命令后，在`gswm/src`下将看到`site`目录。

执行以下命令，修改`index.apt`文件的内容，从而修改`About`页的相关介绍：

```SHELL
cd src/site
vim apt/index.apt
# 该apt文件包含4部分(以-----分隔)，分别为：文档标题、作者、日期和项目描述。
```

```XML
 -----
 Getting Started with Maven(Generate by Advance Configure)
 -----
 WFB
 -----
 09-27-2020
 -----

Maven Site for your project

 Congratulations! If you are looking at this page then you have successfully generated a
 template site employing the simple site archetype and you have run:
  
+-----+

mvn site

+-----+
```

之后通过`cd ../..`回到项目的根路径，执行`maven clean site`重新生成站点，然后打开`target/site/index.html`后，可以发现`About`页的内容发生了改变。

另外，可以通过修改`site.xml`文件来自定义生成的网站，例如更改标题和覆盖默认导航以及外观。

如下所示的步骤，可以自定义生成的站点的图标：

```SHELL
cd src/site
mkdir -p resources/images #创建一个存储图片资源的文件夹
mv ${某个路径下的某图片} resources/images/company.jpg #将需要设置为网站图片的文件放到images目录下
vim site.xml
```

```XML
<bannerLeft>
    <name>Apress</name>
    <src>images/company.png</src>
    <href>http://apress.com</href>
</bannerLeft>
```

之后通过`cd ../..`回到项目的根路径，执行`maven clean site`重新生成站点，然后打开`target/site/index.html`后，可以发现`About`页左上方的图片变成了我们设置的图片

## 生成项目的Javadoc报告

maven提供了一个Javadoc插件，用来使用Javadoc工具去生成Javadocs报告。

集成Javadoc插件仅需要在`pom.xml`文件的`<reporting/>`元素中声明它。当生成`site`时，pom的`reporting`元素
中声明的插件将会被执行。

```XML
<project>
    <!-- Content removed for brevity -->
    <reporting>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <version>3.2.0</version>
            </plugin>
        </plugins>
    </reporting>
</project>
```

执行`mvn clean site`并打开`target/site/index.html`，可以发现程序的左侧侧边栏出现了`Reporting`导航，并可以查看到程序的Javadoc报告。

### 生成单元测试报告

maven在每次build时，默认会执行所有的测试。任何测试失败都会导致build失败。Maven提供了Surefire插件，该插件为运行由JUnit或TestNG等框架创建的测试提供了统一的界面，它也会生成多种形式的执行结果，如：XML、HTML。Surefire插件的配置方式与pom文件报告部分中的Javadoc插件相同。

```XML
<project>
    <!-- Content removed for brevity -->
    <reporting>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-report-plugin</artifactId>
                <version>2.17</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jxr-plugin</artifactId>
                <version>2.3</version>
            </plugin>
        </plugins>
    </reporting>
</project>
```

### 生成代码覆盖率报告

JaCoCo（开源）和Atlassian的Clover是两种流行的Java代码覆盖工具。

```XML
<project>
    <build>
        <plugins>
        <!-- Content removed for brevity -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.4</version>
            <executions>
                <execution>
                    <id>jacoco-init</id>
                    <goals>
                        <!--  
                            prepare-agent目标设置一个指向JaCoCo运行时环境的属性,运行单元测试时，该变量作为VM参数传递。
                        -->
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>jacoco-report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>
            </executions>
            </plugin>
        </plugins>
    <build>
</project>
```

重新执行`mvn clean site`后，打开`${project_path}/target/site/jacoco/index.html`可以看到项目的测试覆盖性报告

### 生成SpotBugs报告

SpotBugs是用于检测Java代码缺陷的工具。它使用静态分析来检测错误模式，例如无限递归循环和空指针引用。

```XML
<project>
    <!-- Content removed for brevity -->
    <reporting>
        <plugins>
            <plugin>
                <groupId>com.github.spotbugs</groupId>
                <artifactId>spotbugs-maven-plugin</artifactId>
                <version>3.1.12</version>
            </plugin>
        </plugins>
    </reporting>
</project>
```

执行`mvn clean site`并打开`target/site/index.html`，可以发现程序的左侧侧边栏出现了`Reporting`导航，并可以查看到程序的Javadoc报告。
