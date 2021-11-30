# JVM常用命令和选项

## JVM dump

在配置了`-XX:+HeapDumpOnOutOfMemoryError`后，当发生内存益处时，会自动把dump存储为文件，文件后缀为`.hprof`。

可以使用`jhat -J-Xmx512M *.hprof`分析该文件，分析完成后，可以在控制台看到

```shell
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

此时，打开`http://localhost:7000/`即可查看该分析报告。

可以看到`All Classes including platform`以包为单位，展示了每个类的情况，对于最后的`Other Queries`，我们可以看到还有如下选项：

```text
All classes excluding platform
Show all members of the rootset
Show instance counts for all classes (including platform) - 每个类的实例数量，从多到少排序
Show instance counts for all classes (excluding platform) - 每个类的实例数量，从多到少排序
Show heap histogram
Show finalizer summary
Execute Object Query Language (OQL) query
```

其中`including platform`表示包含平台包，即包含JDK; 而`excluding platform`是排除JDK后。

## 查看版本号

```java
java -version
```

```shell
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
```

## 查看Java命令默认的命令行参数

```java
java -XX:+PrintCommandLineFlags
```

```shell
-XX:InitialHeapSize=258850880 -XX:MaxHeapSize=4141614080 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC
```

1. InitialHeapSize：初始堆大小
2. MaxHeapSize：最大堆大小
3. UseCompressedClassPointers：开启`kclass point`指针压缩
4. UseCompressedOops：开启对象指针压缩
5. UseParallelGC：使用`ParallelGC`，同样意味着年老代默认使用`ParallelOldGC`

## 查看GC细节

```java
java -XX:PrintGCDetails
```

```shell
Heap
 PSYoungGen      total 74240K, used 3840K [0x000000076db80000, 0x0000000772e00000, 0x00000007c0000000)
  eden space 64000K, 6% used [0x000000076db80000,0x000000076df401a0,0x0000000771a00000)
  from space 10240K, 0% used [0x0000000772400000,0x0000000772400000,0x0000000772e00000)
  to   space 10240K, 0% used [0x0000000771a00000,0x0000000771a00000,0x0000000772400000)
 ParOldGen       total 169472K, used 0K [0x00000006c9200000, 0x00000006d3780000, 0x000000076db80000)
  object space 169472K, 0% used [0x00000006c9200000,0x00000006c9200000,0x00000006d3780000)
 Metaspace       used 3414K, capacity 4554K, committed 4864K, reserved 1056768K
  class space    used 368K, capacity 386K, committed 512K, reserved 1048576K
```

1. 年轻代（PSYoungGen）
   1. 伊甸区（eden）
   2. 存活区
      1. from
      2. to
2. 年老代（ParOldGen）
3. 元空间（Metaspace）

## 查看完整的参数细节

```java
java -XX:+PrintFlagsFinal
```

```shell
...
     bool UseParallelGC                            := true                                {product}
     bool UseParallelOldGC                          = true                                {product}
...
```

表明jdk8默认使用`ParallelGC`和`ParallelOldGC`进行GC。其中，`UseParallelGC`是`CommandLineFlags`，是`-XX:+UseParallelGC`指定的，而`UseParallelOldGC`是默认行为。
