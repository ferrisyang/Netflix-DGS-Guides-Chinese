# Data Fetching Context

在 [GraphQL] Java 中的每个 datafetcher 都有一个 context（上下文）。一个 datafetcher 通过调用  [`DataFetchingEnvironment.getContext()`](https://javadoc.io/doc/com.graphql-java/graphql-java/12.0/graphql/schema/DataFetchingEnvironment.html#getContext--) 来获得它的 context。 这是将 Request context 传递给 datafetcher 和 dataloader 的一种常见机制。DGS 框架有自己的  [`DgsContext`](https://github.com/Netflix/dgs-framework/blob/master/graphql-dgs/src/main/kotlin/com/netflix/graphql/dgs/context/DgsContext.kt) 实现，它用于日志检测以及其他用途。它的设计方式是，您可以使用自己的自定义 context 对其进行扩展。

为了创建一个自定义的 context，实现 `DgsCustomContextBuilder` 类型的 Spring Bean。撰写 `build()` 方法，以便它创建一个表示自定义 Context 对象的类型实例：

```java
@Component
public class MyContextBuilder implements DgsCustomContextBuilder<MyContext> {
    @Override
    public MyContext build() {
        return new MyContext();
    }
}

public class MyContext {
    private final String customState = "Custom state!";

    public String getCustomState() {
        return customState;
    }
}
```

datafetcher 现在可以通过调用 `getCustomContext()` 方法来获取 context。

```java
@DgsData(parentType = "Query", field = "withContext")
public String withContext(DataFetchingEnvironment dfe) {
    MyContext customContext = DgsContext.getCustomContext(dfe);
    return customContext.getCustomState();
}
```

同样的，自定义的 context 可也用于 DataLoader。

```java
@DgsDataLoader(name = "exampleLoaderWithContext")
public class ExampleLoaderWithContext implements BatchLoaderWithContext<String, String> {
    @Override
    public CompletionStage<List<String>> load(List<String> keys, BatchLoaderEnvironment environment) {

        MyContext context = DgsContext.getCustomContext(environment);

        return CompletableFuture.supplyAsync(() -> keys.stream().map(key -> context.getCustomState() + " " + key).collect(Collectors.toList()));
    }
}
```

