# 网关gaetway

注意引入gateway不要引入阻塞的spring-boot-starter-web库。因为gateway依赖的web-flux会与其发生冲突。

三大核心概念

- 路由(Route)：构建网关的基本模板，由一系列的断言和过滤组成。
- 断言(Predicate)：判断该路由是否匹配要求
- 过滤(Filter)：指的是Spring框架中的GatewayFilter的实例，使用过滤器，可以在请求被路由前和后对请求进行修改。

## 断言工厂：RoutePredicateFactory

Spring Cloud 网关 匹配routes作为SPring WebFlux的HandlerMapping的基础设置。该网关引入了许多内置的断言工厂。这些断言工厂分别去匹配不同的http请求属性。可以通过配置多个路由断言工厂来实现逻辑与的组合断言。

其主要实现如下：

### AfterRoutePredicateFactory

只有在时间超过配置的这个时间后，路由请求才会被执行

### BeforeRoutePredicateFactory

只有在时间在配置的这个时间之前，路由请求才会被执行

### BetweenRoutePredicateFactory

只有在时间在配置的这个时间范围内，路由请求才会被执行

### CookieRoutePredicateFactory

需要两个参数，一个是Cookie name，一个是正则表达式。
路由规则会通过获取对应的Cookie name的值和正则表达式去匹配，匹配上就路由，匹配不上就不执行

### HeaderRoutePredicateFactory

两个参数：一个是属性名称，一个是正则表达式

### HostRoutePredicateFactory

接受一组参数，一组匹配的域名列表。模板之间用`,`分隔，每个模板是一个用`.`号作为分隔符的ant表达式。它通过参数中的主机地址(即Host请求头)作为匹配规则。

### MethodRoutePredicateFactory

请求方法，比如：GET、POST

### PathRoutePredicateFactory

请求路径

### QueryRoutePredicateFactory

请求参数。两个参数：请求key和其值的正则表达式

### RemoteAddrRoutePredicateFactory

## 过滤

### 单个路由的过滤：GatewayFilter

### 全局过滤：GLobalFilter
