# 联合测试

[Federation](https://www.apollographql.com/docs/federation/) 允许你扩展或者引用在一个 Graph 中已存在的类型。DGS 根据 DGS 拥有的 schema 完成部分查询，而 Gateway 负责从其他 DGS 获取数据。

## 不通过 Gateway 来测试联合查询

通过复制 Gateway 将发送给 DGS 的查询格式，可以独立测试 DGS 的联合查询。这并不涉及 Gateway，因此 DGS 不负责查询响应中将混合的部分。如果希望验证 DGS 能够返回适当的数据以响应联合查询，那么这种技术非常有用。

我们一起看一个例子，schema 扩展了已经在另一个 DGS 中定义的 `Movie` 类型。

```scheme
type Movie @key(fields: "movieId")  @extends {
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

现在你想要验证你的 DGS 能够通过根据 `movieId` 字段的混合了 `script` 字段来实现 Movie 查询。通常，Gateway 将会以以下格式发送一个 [\_entities](https://www.apollographql.com/docs/apollo-server/federation/federation-spec/#resolve-requests-for-entities) 请求：

```scheme
 query ($representations: [_Any!]!) {
        _entities(representations: $representations) {
            ... on Movie {
                movieId
                script { title }
        }}}
```

`representations` 输入是一个变量Map，这个 map 包含了设置了 `Movie` 的 `__typename` 字段以及 `movieId` 为值。例如 `12345`。

你现在可以通过手动创建 query 的方式设置一个 [Query Executor](https://netflix.github.io/dgs/query-execution-testing/) 来测试，或者你也可以使用 [client code generation](https://netflix.github.io/dgs/advanced/java-client/#type-safe-query-api) 提供的`Entities Query Builder API` 来生成这个联合 query。

这里有一个使用手动创建 `_entities` query 的方式查询 `Movie` 的例子：

```java
@Test
    void federatedMovieQuery() throws IOException {
         String query = "query ($representations: [_Any!]!) {" +
              "_entities(representations: $representations) {" +
                  "... on Movie {" +
                      "movieId " +
                      "script { title }" +
         "}}}";

        Map<String, Object> variables = new HashMap<>();
        Map<String,Object> representation = new HashMap<>();
        representation.put("__typename", "Movie");
        representation.put("movieId", 1);
        variables.put("representations", List.of(representation));

        DocumentContext context = queryExecutor.executeAndGetDocumentContext(query, variables);
        GraphQLResponse response = new GraphQLResponse(context.jsonString());
        Movie movie = response.extractValueAsObject("data._entities[0]", Movie.class);
        assertThat(movie.getScript().getTitle()).isEqualTo("Top Secret");
    }
```

### Using the Entities Query Builder API

或者，您可以使用 [EntitiesGraphQLQuery](https://netflix.github.io/dgs/advanced/java-client/#building-federated-queries) 来构建 graphql 请求，并结合 [code generation](https://netflix.github.io/dgs/generating-code-from-schema/) 插件来生成使用请求构建器所需的类，从而生成联合查询。这提供了一种方便的类型安全的方法来构建查询。

要设置 code generation 以生成构建查询所需的类，请遵循[这里](13-java-graphql-client.md#type-safe-query-api)的说明。

需要在 `build.gradle` 中添加 `com.netflix.graphql.dgs:graphql-dgs-client:latest.release` 依赖。

现在，我们可以编写一个使用 `EntitiesGraphQLQuery`、`GraphQLQueryRequest` 和 `EntitiesProjectionRoot` 来构建查询的测试。最后，还可以使用 `GraphQLResponse` 提取响应。

配置如下：

```java
@Test
    void federatedMovieQueryAPI() throws IOException {
        // constructs the _entities query with variable $representations containing a 
        // movie representation that represents { __typename: "Movie"  movieId: 12345 }
        EntitiesGraphQLQuery entitiesQuery = new EntitiesGraphQLQuery.Builder()
                    .addRepresentationAsVariable(
                            MovieRepresentation.newBuilder().movieId(1122).build()
                    )
                    .build();
        // sets up the query and the field selection set using the EntitiesProjectionRoot
        GraphQLQueryRequest request = new GraphQLQueryRequest(
                    entitiesQuery,
                    new EntitiesProjectionRoot().onMovie().movieId().script().title());

        String query  = request.serialize();
        // pass in the constructed _entities query with the variable map containing representations
        DocumentContext context = queryExecutor.executeAndGetDocumentContext(query, entitiesQuery.getVariables());

        GraphQLResponse response = new GraphQLResponse(context.jsonString());
        Movie movie = response.extractValueAsObject("data._entities[0]", Movie.class);
        assertThat(movie.getScript().getTitle()).isEqualTo("Top Secret");
    }
```

