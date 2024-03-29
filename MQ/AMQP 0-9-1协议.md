# AMQP 0-9-1协议

AMQP(Advanced Message Queuing Protocol)是一个消息协议。它使符合标准的客户端应用程序能够与符合标准的消息中间件代理进行通信。

消息传递代理从发布者（发布它们的应用程序，也称为生产者）接收消息并将它们路由到消费者（处理它们的应用程序）。
由于它是一种网络协议，发布者、消费者和代理都可以驻留在不同的机器上。

## AMQP 0-9-1 模型简介

AMQP 0-9-1 模型的世界观如下：消息发布到exchanges，通常被比作邮局或邮箱。然后，交换使用称为绑定的规则将消息副本分发到队列。然后，代理要么将消息传递给订阅队列的消费者，要么消费者按需从队列中获取/拉取消息。

![hello-world-example-routing](./imgs/hello-world-example-routing.png)

发布消息时，发布者可以指定各种消息属性（消息元数据）。这些元数据中的一些可能被代理使用，但是，其余的对代理完全不透明，并且仅由接收消息的应用程序使用。

网络不可靠，应用程序可能无法处理消息，因此 AMQP 0-9-1 模型具有`消息确认的概念(acknowledgements)`：当消息传递给消费者时，消费者会自动或在应用程序开发人员手动操作后尽可能快的通知代理确认消息。当使用消息确认时，代理只有在收到该消息（或消息组）的确认通知时才会从队列中完全删除该消息。

在某些情况下，例如，当消息无法路由时，消息可能会返回给发布者、丢弃，或者如果代理实现了扩展，则将消息放入所谓的“死信队列”。发布者通过使用某些参数发布消息来选择如何处理此类情况。

队列(Queue)、交换机(Exchange)和绑定(binding)统称为 AMQP 实体。

## AMQP 0-9-1 是一种可编程协议

AMQP 0-9-1 是一种可编程协议，从某种意义上说，AMQP 0-9-1 实体和路由方案主要由应用程序本身定义，而不是代理管理员。因此，它为声明队列和交换机实体、定义它们之间的绑定、消费者订阅队列等的操作做了规定。

这为应用程序开发人员提供了很大的自由，但也要求他们能够意识到潜在的定义冲突。在实践中，定义冲突很少见，通常表现为配置错误。

应用程序声明他们需要的 AMQP 0-9-1 实体，定义必要的路由方案，并且可以选择在不再使用 AMQP 0-9-1 实体时删除它们。

## 交换机（Exchange）和其类型

交换机是一种 AMQP 0-9-1 实体，消息会发送到交换机。交换机接收消息并将其路由到零个或多个队列中。使用的路由算法取决于交换类型和称为绑定的规则。 AMQP 0-9-1 代理提供四种交换类型：

交换机类型｜默认的预定义的名字
---|---
Direct exchange|(空字符串) 或者 amq.direct
Fanout exchange|amq.fanout
Topic exchange|amq.topic
Headers exchange|amq.match(在RabbitMQ中是amq.headers)

除了交换类型之外，交换还声明了许多属性，其中最重要的是：

- Name
- Durability：代理重启后，该交换机是否还存在
- Auto-delete：在该交换机上所有队列都解绑后，是否删除它
- Arguments：可选，由插件和特定于代理的功能使用

ps：交换机可以是持久的（Durability）或短暂的。持久交换机在代理重启后仍然存在，而临时交换机则不会（当代理重新上线时必须重新声明它们）。并非所有场景和用例都要求交换是持久的。

### 默认的交换机

默认交换机是代理预先声明的没有名称（空字符串）的直连交换机（Direct exchange）。它有一个特殊的属性，使它对简单的应用程序非常有用：创建的每个队列都会使用与队列名称相同的路由键自动绑定到它。

例如：当您声明一个名为“search-indexing-online”的队列时，AMQP 0-9-1 代理将使用“search-indexing-online”作为路由键（有时称为绑定键）将其绑定到默认交换器。因此，使用路由键“search-indexing-online”发布到默认交换的消息将被路由到队列“search-indexing-online”。换句话说，默认的交换使看起来可以直接将消息传递到队列，即使从技术上讲不是这样的。

### Direct Exchange

Direct Exchange根据消息路由键将消息传递到队列。直连交换机（Direct Exchange）是消息的单播路由的理想选择（尽管它们也可用于多播路由，只要多个队列绑定同一个路由键即可实现）。下面是它的工作原理：

1. 一个队列通过一个路由键K绑定到交换机
2. 当一个新消息带着路由键R到达直连交换机，该交换机会将其路由到K（队列绑定时的路由键）==R（消息申明的路由键）的队列

直连交换机通常用于以循环方式在多个workers（同一应用程序的实例）之间分配任务。

请注意：在 AMQP 0-9-1 中，消息在消费者之间而不是队列之间进行负载平衡。

![exchange-direct](./imgs/exchange-direct.png)

如上图所示，只要某队列路由键是消息中声明的路由键，直连交换机就会将一份副本交给该队列。负载均衡会在绑定到同一队列的多个消费者之间进行，默认策略是轮询。

### Fanout Exchange

扇出交换机（Fanout Exchange）将消息路由到绑定到它的所有队列，并且忽略路由键。如果 N 个队列绑定到一个扇出交换器，则当一条新消息发布到该交换机时，该消息的副本将传递到所有 N 个队列。扇出交换是消息广播路由的理想选择。

因为扇出交换机向绑定到它的每个队列传递消息的副本，所以它的用例非常相似：

- 大型多人在线游戏(MMO)可以将其用于排行榜更新或其它全局事件
- 体育新闻网站可以使用它近乎实时地向移动客户端分发分数更新
- 分布式系统可以使用它广播各种状态和配置更新
- 群李安可以使用它在参与者之间分发消息（不过AMQP没有内建的出席的概念，所以这种场景下XMPP是个更好的选择）

![exchange-fanout](./imgs/exchange-fanout.png)

### Topic Exchange

主题交换机（Topic Exchange）基于消息路由键与用于将队列绑定到交换的模式之间的匹配将消息路由到一个或多个队列。主题交换类型通常用于实现各种发布/订阅模式变体。主题交换通常用于消息的多播路由。

主题交换机有一套非常广泛的用例。每当一个问题涉及到多个消费者/应用程序有选择地选择他们想要接收的消息类型时，就应该考虑使用主题交换机。

用例：

- 分发与特定地理位置相关的数据，例如，销售点
- 后台任务处理由多个工作者完成，每个工作者都能处理特定的任务集
- 股票价格更新（和其他类型的金融数据更新）。
- 涉及分类或标签的新闻更新（例如，只针对某项运动或团队）。
- 在云中协调不同种类的服务
- 分布式架构/操作系统的软件构建或打包，每个构建器只能处理一个架构或操作系统

### Headers Exchange

报头交换是机为多个属性的路由设计的，这些属性更容易被表达为报头而不是路由密钥。头信息交换忽略了路由密钥属性。相反，用于路由的属性取自报头属性。如果报头的值等于绑定时指定的值，则认为该消息是匹配的。

有可能将一个队列绑定到一个头信息交换中，使用一个以上的头信息进行匹配。在这种情况下，代理需要应用开发者提供更多的信息，即它应该考虑有任何一个头匹配的消息，还是所有的头，这就是 "x-match"绑定参数的作用。当 "x-match "参数被设置为 "any "时，只要有一个匹配的报头值就足够了。如果将 "x-match "设置为 "all"，则规定所有的值都必须匹配。

头信息交换机可以被看作是 "直接交换机的加强版"。它们可以被用作使用头信息进行路由的直接交换机，其中路由键不必是一个字符串；它可以是一个整数或一个哈希（字典）。

请注意，以`x-`开头的标题将不会被用来评估匹配。

## 队列
