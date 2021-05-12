# Java GraphQL 客户端



## 用法

DGS 框架提供一个 GraphQL 客户端，它可以用于从一个 GraphQL endpoint 来获取数据。客户端有两个组件，每个都可以自用或者联合一起用。

* GraphQLClient - 一个 HTTP 客户端包装，它可以提供简单的 GraphQL 响应解析。
* 查询 API codegen - 生成类型安全的查询生成器



## HTTP 客户端包装

GraphQL 客户端包装了任何 HTTP 客户端和提供了简单的 GraphQL 响应解析。客户端用于连接各种 GraphQL endpoint（即使不是使用 DGS 框架实现的），但提供了额外的方便用于解析 Gateway 和 DGS 响应。包括对  [Errors Spec](08-Error Handling.md) 的支持。

使用这个客户端，需要创建一个 `DefaultGraphQLClient` 实例。

```java
GraphQLClient client = new DefaultGraphQLClient(url);
```

`url` 是你想要调用的 endpoint server url。这个 url 将会传递给下面的讨论的 callback。

使用 `GraphQLClient` 执行一个查询。`executeQuery` 方法有4个参数：

1. 请求字符串
2. 一个请求变量的可选Map
3. 一个可选的操作名称
4. 一个 `RequestExecutor` 实例，一般为一个 Lambda 表达式

因为在 Netflix 大量使用了 HTTP 客户端，而 GraphQLClient 又不与任何具体的 HTTP 客户端实现挂钩。使用了任意 HTTP 客户端（RestTemplate，RestClient，OKHTTP，...）。操作名在记录查询 log 的时候使用，或者你想要在一个请求重进行多个查询的时候使用。开发者有责任为做一个真实的 HTTP 调用去实现一个 `RequestExecutor`。`RequestExecutor` 接收 `url`，`headers` Map 和 request `body` 参数，并且应该返回一个 `HttpResponse` 的实例。基于 HTTP 响应，GraphQLClient 解析响应和提供简单接口来访问数据和错误。下面的例子使用了 `RestTemplate`。

```java
private RestTemplate dgsRestTemplate;

private static final String URL = "http://someserver/graphql";

private static final String QUERY = "{\n" +
            "  ticks(first: %d, after:%d){\n" +
            "    edges {\n" +
            "      node {\n" +
            "        route {\n" +
            "          name\n" +
            "          grade\n" +
            "          pitches\n" +
            "          location\n" +
            "        }\n" +
            "        \n" +
            "        userStars\n" +
            "      }\n" +
            "    }\n" +
            "  }\n" +
            "}";

public List<TicksConnection> getData() {
    DefaultGraphQLClient graphQLClient = new DefaultGraphQLClient(URL);
    GraphQLResponse response = graphQLClient.executeQuery(query, new HashMap<>(), "TicksQuery", (url, headers, body) -> {
        /**
         * The requestHeaders providers headers typically required to call a GraphQL endpoint, including the Accept and Content-Type headers.
         * To use RestTemplate, the requestHeaders need to be transformed into Spring's HttpHeaders.
         */
        HttpHeaders requestHeaders = new HttpHeaders();
        headers.forEach(requestHeaders::put);

        /**
         * Use RestTemplate to call the GraphQL service. 
         * The response type should simply be String, because the parsing will be done by the GraphQLClient.
         */
        ResponseEntity<String> exchange = dgsRestTemplate.exchange(url, HttpMethod.POST, new HttpEntity(body, requestHeaders), String.class);

        /**
         * Return a HttpResponse, which contains the HTTP status code and response body (as a String).
         * The way to get these depend on the HTTP client.
         */
        return new HttpResponse(exchange.getStatusCodeValue(), exchange.getBody());
    }); 

    TicksConnection ticks = response.extractValueAsObject("ticks", TicksConnection.class);
    return ticks;
}
```

 `GraphQLClient` 通过多种方式提供了解析和获取数据和错误的方法。获取更全面的方法支持列表，请参考  `GraphQLClient` JavaDoc。

| 方法                 | 描述                                                         | 例子                                                         |
| :------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| getData              | 获取一个数据Map                                              | `Map<String,Object> data = response.getData()`               |
| dataAsObject         | 使用 Jackson Object Mapper，提供一个class来解析数据          | `TickResponse data = response.dataAsObject(TicksResponse.class)` |
| extractValue         | 使用 JsonPath 抽取值。返回类型应该是你期望的类型，但也依赖于 Json 结构。对 JSON Object 来说，只是返回了一个 Map。虽然看起来类型安全，其实并不是。最适用且简单的类型就是 String，Int 等，以及 这些类型的 List。 | `List<String> name = response.extractValue("movies[*].originalTitle")` |
| extractValueAsObject | 使用 JSONPath 抽取值，并且反序列化成给定的 class             | `Ticks ticks = response.extractValueAsObject("ticks", Ticks.class)` |
| extractValueAsObject | 使用 JSONPath 抽取值，并且反序列化成给定的 TypeRef. 用于 一个确定泛型的 Map 和 List | `List<Route> routes = response.extractValueAsObject("ticks.edges[*].node.route", new TypeRef<List<Route>>(){})` |
| getRequestDetails    | 抽取一个 `RequestDetails` 对象。只在当 requestDetails 在查询中作为 Request，并且对应 Gateway 的时候有效。 | RequestDetails requestDetails = `response.getRequestDetails()` |
| getParsed            | 为后面的 JSONPath 进程解析一个 `DocumentContext`             | `response.getDocumentContext()`                              |



### Errors

GraphQLClient 不仅会检查 HTTP 级别的错误（基于 response 状态码），而且也会检查在 GraphQL response 中的 `errors` 块。GraphQLClient 兼容使用在 Gateway 和 DGS 中的  [Errors Spec](08-Error Handling.md) ，并且比较容易抽取像 ErrorType 的错误信息。

例如，下面 GraphQL response，GraphQLClient 让你更容易获取 ErrorType 和 ErrorDetail 字段。主要 `ErrorType` 是一个被  [Errors Spec](08-Error Handling.md)  所指定的枚举。

```json
{
  "errors": [
    {
      "message": "java.lang.RuntimeException: test",
      "locations": [],
      "path": [
        "hello"
      ],
      "extensions": {
        "errorType": "BAD_REQUEST",
        "errorDetail": "FIELD_NOT_FOUND"
      }
    }
  ],
  "data": {
    "hello": null
  }
}
```

```java
assertThat(graphQLResponse.errors.get(0).extensions.errorType).isEqualTo(ErrorType.BAD_REQUEST)
assertThat(graphQLResponse.errors.get(0).extensions.errorDetail).isEqualTo("FIELD_NOT_FOUND")
```



## Type safe Query API

基于一个 GraphQL schema，可以为 Java/Kotlin 生成出来一个类型安全的 query API。生成的 API 是一个规划好字段的（字段选择器），可以让你构建一个 GraphQL 请求的 builder 风格的 API。因为在 schema 有变化的时候代码被重新生成，这将会有利于在查询的时候抓取错误。因为 Java 不支持多行 String（当前为止）它也会为指定的请求赋予更多的可读方式。

如果你已经有一个 DGS，并且想要为这个 DGS 生成一个客户端（例如：为了测试目的）这个客户端生成仅仅是在 [Codegen configuration](06-Code Generation.md) 中的一个额外属性。请设置以下内容在你的 `build.gradle`。

```groovy
buildscript {
   dependencies{
      classpath 'netflix:graphql-dgs-codegen-gradle:latest.release'
   }
}

apply plugin: 'codegen-gradle-plugin'

generateJava{
   packageName = 'com.example.packagename' // The package name to use to generate sources
   generateClient = true
}
```

代码将会在编译的时候生成。生成的代码在 `build/generated` 路径。

随着正确的配置 codegen，当编译项目的时候，将会生成一个 builder 风格的 API。如上面同样的查询例子，使用生成的 builder API 创建查询。

```java
GraphQLQueryRequest graphQLQueryRequest =
                new GraphQLQueryRequest(
                    new TicksGraphQLQuery.Builder()
                        .first(first)
                        .after(after)
                        .build(),
                    new TicksConnectionProjectionRoot()
                        .edges()
                            .node()
                                .date()
                                .route()
                                    .name()
                                    .votes()
                                        .starRating()
                                        .parent()
                                    .grade());

String query = graphQLQueryRequest.serialize();
```

`GraphQLQueryRequest` 是一个来自 `graphql-dgs-client` 的类。`TicksGraphQLQuery` 和 `TicksConnectionProjectionRoot`  是生成的。在创建查询后，它将会被序列化成一个 String，并且通过 GraphQLClient 来执行。

注意 `edges` and `node`  字段，因为在例子中 schema 使用 Relay pagination。



### 接口的预测

当返回的字段是一个接口时，需要使用一个片段来指定字段的正确类型。

```scheme
type Query @extends {
    script(name: String): Script
}

interface Script {
    title: String
    director: String
    actors: [Actor]
}

type MovieScript implements Script {
    title: String
    director: String
    length: Int
}

type ShowScript implements Script {
    title: String
    director: String
    episodes: Int
}
```

```scheme
query { 
    script(name: "Top Secret") { 
        title 
        ... on MovieScript {
            length
        } 
    } 
}
```

Query builder 很好的支持这样的语法。

```java
 GraphQLQueryRequest graphQLQueryRequest =
    new GraphQLQueryRequest(
        new ScriptGraphQLQuery.Builder()
            .name("Top Secret")
            .build(),
        new ScriptProjectionRoot()
            .title()
            .onMovieScript()
                .length();                    
    );
```



### 创建联合查询

你可以一同使用 `GraphQLQueryRequest` 和 `EntitiesGraphQLQuery` 来生成联合查询。在基于 input schema上，通过基于输入 schema 的 `representations` 的协助，API 提供一个类型安全的方式来创建  [_entities](https://www.apollographql.com/docs/apollo-server/federation/federation-spec/#resolve-requests-for-entities)  查询。`representations` 是作为一个 Map 变量传递进来。每个生成的 `representations` 类都是基于你 schema 中定义的类型来创建的 `key` 字段，一同使用 `__typename`。`EntitiesProjectionRoot` 用于查询特定的类型。

例如，我们一起看一下一个 扩展 Movie 类型的 schema：

```scheme
type Movie @key(fields: "movieId") @extends {
    movieId: Int @external
    script: MovieScript
}

type MovieScript  {
    title: String
    director: String
    actors: [Actor]
}

type Actor {
    name: String
    gender: String
    age: Int
}
```

随着客户端代码的生成，你现在有一个 `MovieRepresentation`，包括 key 字段，例如 `movieId`，和已经被设置成 `Movie` 类型的 `__typename` 字段。现在你可以将做为一个 `representations` 变量的  `EntitiesGraphQLQuery`  添加进每个 representation。你在 `Movie` 表单中，也有一个带有可以查询字段的  `onMovie()` 方法的  `EntitiesProjectionRoot`。最后，你可以将所有作为一个 `GraphQLQueryRequest` 放一起，一同序列化成最终的查询 String。`representations`  变量 map 是在 `EntitiesGraphQLQuery`  上通过 `getVariables` 来提供的。

这是一个之前 schema 的例子：

```java
        EntitiesGraphQLQuery entitiesQuery = new EntitiesGraphQLQuery.Builder()
                    .addRepresentationAsVariable(
                            MovieRepresentation.newBuilder().movieId(1122).build()
                    )
                    .build();
        GraphQLQueryRequest request = new GraphQLQueryRequest(
                    entitiesQuery,
                    new EntitiesProjectionRoot().onMovie().movieId().script().title()
                    );

        String query  = request.serialize();
        Map<String, Object> representations = entitiesQuery.getVariables();
```

