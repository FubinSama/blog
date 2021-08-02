# 关于IDEA用到的各种bug及解决方案

## 读取properties文件，导致中文乱码

1. 选择File->Settings->Editor->FileEncodings，将Global Encoding、Project Encoding、Default encoding for properties files全部设置为UTF-8，并将Transparent native-to-ascii conversion打钩
2. 将原properties文件内容复制到文本编辑器，然后删除该文件，并重建，将内容复制回来，重启项目

## IDEA编译maven的Kotlin项目会强制去.m2中查找jar包

目前没找到正确的配置方式。只能采用为.m2/repository创建软连接到真实的仓库的方式

## IDEA创建的Web项目会发现有中文乱码问题

1. 设置数据库编码为UTF8
2. 设置H5模板使用UTF8
3. 对request进行必要的编码设置
4. 设置Tomcat为UTF8，这里指的是更改启动参数。（当然SpringBoot使用内置Tomcat，不会有该问题）

## IDEA远程调试

1. 启动代码时使用`java --agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar xxx.jar`
2. idea在项目上选择增加运行方法->remote，配置相应的主机和端口号
3. 在需要调试的位置添加断点，并点击调试运行
4. 执行项目的某种操作

ps：请保证5050（或其他指定的端口）在服务器上已打开，不会被防火墙拦截掉
