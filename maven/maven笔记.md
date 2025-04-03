# maven笔记

## 基本概念

Maven 本质上是一个插件执行框架；所有工作均由插件完成。

插件`Plugin`是由一到多个`Goal`组成的。如`mvn clean:clean`执行`clean`插件的`clean`目标。

插件主要包含两类：

- Build插件：通过`<build/>`元素配置
- Reporting插件：通过`<reporting/>`元素配置

maven的运行是围绕着构建（`build`）的生命周期（`lifecycle`）的概念来进行的，有三个内建的构建生命周期：`default`，`clean`，`site`。而构建生命周期由阶段（`phase`）组成。

`default`构建生命周期主要包含如下阶段：

- `validate`
- `compile`
- `test`
- `package`
- `verify`
- `install`
- `deploy`

这些生命周期阶段（加上此处未显示的其他生命周期阶段）按顺序执行以完成`default`生命周期。

而`clean`和`site`主要由同名的命令行可调用阶段组成。一般通过`mvn clean deploy`按序执行一到多个阶段。

构建阶段是由插件目标组成的。插件目标代表一个特定任务（比构建阶段更精细），有助于项目的构建和管理。它可能绑定到零个或多个构建阶段。未绑定到任何构建阶段的目标可以通过直接调用在构建生命周期之外执行。

用连字符命名的阶段（pre-*、post-* 或 process-*）通常不直接从命令行调用。这些阶段对构建进行排序，产生在构建之外无用的中间结果。在调用集成测试的情况下，环境可能会处于挂起状态。

maven内置了生命周期阶段绑定的默认目标，详见[Introduction to the Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#lifecycle-reference)