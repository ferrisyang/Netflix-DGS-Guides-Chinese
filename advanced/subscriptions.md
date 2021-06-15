# 订阅

GraphQL 订阅用于随时间从服务器接收查询的更新。一个常见的例子是从服务器发送更新通知。

常规的 GraphQL 查询使用一个简单的 \(HTTP\) 请求/响应来执行查询。对于订阅来说，保持了一个开放的链接。当前，我们使用 Websocket 的技术来支持订阅。我们将会在未来添加 SSE 的支持。

## 服务端编程模型

在 DGS 框架中，一个订阅是一个有 `@DgsData` 注解修饰的 datafetcher 实现。不同于普通的 datafetcher，订阅必须返回一个 `org.reactivestreams.Publisher`。

```java
import reactor.core.publisher.Flux;
import org.reactivestreams.Publisher;
⋮

@DgsData(parentType = "Subscription", field = "stocks")
public Publisher<Stock> stocks() {
    return Flux.interval(Duration.ofSeconds(1)).map({ t -> Tick(t.toString()) })
}
```

`Publisher` 接口来自于 Reactive Streams。Flux 是默认的 Spring 实现。

[`SubscriptionDatafetcher.java`](https://github.com/Netflix/dgs-framework/blob/master/graphql-dgs-example-java/src/main/java/com/netflix/graphql/dgs/example/datafetcher/SubscriptionDataFetcher.java) 是一个完整的例子。下一步，必须选择传输实现，这取决于应用程序的部署方式。

## Websockets

订阅的 endpoint 运行在 `/subscriptions` 上。正常的 GraphQL 查询发送到 `/graphql` ，而订阅请求发送到 `/subscriptions`。在 GraphQL 社区里，订阅用的最常见的传输协议就是 WebSockets。

Apollo 定义了一个 [sub-protocol](https://github.com/apollographql/subscriptions-transport-ws/blob/master/PROTOCOL.md)，由客户端库支持，由 DGS 框架实现。为了启用 WebSocket 支持，需要在你的 `build.gradle` 中添加如下模块：

```groovy
implementation 'com.netflix.graphql.dgs:graphql-dgs-subscriptions-websockets-autoconfigure:latest.release'
```

Apollo 客户端通过一个 [link](https://www.apollographql.com/docs/link/links/ws/) 支持 WebSocket。通常，您希望将Apollo Client配置为HTTP链接和WS链接，并根据查询类型将它们 [区分](https://www.apollographql.com/docs/link/composition/#directional-composition) 。

