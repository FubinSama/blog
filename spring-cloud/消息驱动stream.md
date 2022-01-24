# 消息驱动stream

屏蔽底层消息中间件的差异，降低切换成本，统一消息的编程模型。

## 定义

Spring Cloud Stream是一个构建消息驱动微服务的框架。

应用程序通过inputs或者outputs来与Spring Cloud Stream中binder对象交互。
通过我们配置来binding，而Spring Cloud Stream的binding对象负责与消息中间件交互。
所以，我们只需要搞清楚如何与Spring Cloud Stream交互就可以方便使用消息驱动的方式。

通过使用Spring Integration来连接消息代理中间件以实现消息事件驱动。
Spring Cloud Stream为一些供应商的消息中间件产品提供了个性化的自动化配置实现，引用了发布-订阅、消费组、分区三个核心概念。

目前官方只支持RabbitMQ、Kafka。第三方进行了其它组件的整合。

## 分组

在Stream中处于同一个group中的多个消费者是竞争关系，就能够保证消息只会被其中一个应用消费一次。
不同组之间是可以重复消费的（即消息会全量发送到不同的组）。

对于rabbitmq来说，所为的分组就是应用是否绑定在了同一个队列上。（Stream会自动创建队列绑定到声明的交换机上，交换机会广播所有的队列。）
如：`studyExchange.anonymous.cZyIBTn3SoCLxZWetxMqfQ`和`studyExchange.anonymous.jfhfQGLWT2WgtddXdIH2SA`这两个队列（都绑定在studyExchange这个交换机上）意味着两个应用不在同一个组。（不指定分组时会自动为每个应用生成一个`anonymous.随机序号`作为组，如上面所示）
