# JDK源码阅读

## 小技巧

###  vmSymbols.hpp

这里面定义了虚拟机基本的符号表，可以在这里找到关键的符号获取方法，并以此进行搜索。

### {JAVA类名}.c

> 如：`Thread.c`，`Object.c`

其中会声明`static JNINativeMethod methods[]`数组，该数组记录了Java的`native`方法和JVM对应入口的映射关系

### jvm.cpp

里面包含所有的java native方法的入口，其方法声明如下：

```cpp
JVM_ENTRY(void, JVM_MonitorNotify(JNIEnv* env, jobject handle))
  JVMWrapper("{java native方法的映射名}");
... // 函数执行体
JVM_END
```