# Async Data Fetching

Data loaders 解决加载数据时的 N+1 问题。

## 对于 N+1 问题的解释

说你要查询 movies 列表，并且每个 movie 包含一些 director 数据。设想一下，Movie 和 Director 实体需要两个不同的服务。在一个纯粹的实现中，为了加载50部影片，那么你需要调用 Director 服务 50 次：每次一个 movie。一共 51 个查询：1个查询是获取 movies 列表，50个查询是为了获取每个 movie 的 director 数据。这很明显不会有很好的性能。

创建一个可以通过一次查询获取所有 directors 列表的查询更有效率。首先 _Director_ 服务必须支持这样的查询，因为服务应该提供获取 Directors 列表的方法。那么在 Movie 服务的 data fetcher 需要更加的智能，去处理 Directors 服务的批量请求。

这就引入了 Data Loader。

## 如果我的服务不支持批量加载怎么办？

如果（在这个例子中）`DirectorServiceClient` 并不提供一个获取 directors 列表的方法，怎么办？如果只提供了通过ID获取单一 director 的方法，怎么办？同样的问题对于 REST 服务也一样：如果没有一个可以获取多个 directors 的接口，怎么办？同样的，为了直接从数据库中加载数据，你必须写一个一次可以获取多个 directors 的查询。如果这些方法不可用，那么需要提供服务来修复！

## 实现一个 Data Loader

对你来说最简单的注册一个 data loader 的方式就是去创建一个类，来实现 `org.dataloader.BatchLoader` 或者 `org.dataloader.MappedBatchLoader` 接口。这个接口是参数化的；它需要一个 Key 的类型和 `BatchLoader` 的结果。例如，如果 Director 的 identifiers 是一个 `String` 类型，那么你需要一个 `org.dataloader.BatchLoader<String, Director>`。你必须用 `@DgsDataLoader` 来给这个类进行注解，以至于框架可以描述注册这个 data loader。

为了实现 `BatchLoader` 接口，你必须实现一个 `CompletionStage<List<V>> load(List<K> keys)` 的方法。

以下就是关于一个数据加载器（Data Loader）从一个虚构的 Director 服务中加载数据的例子:

```java
package com.netflix.graphql.dgs.example.dataLoader;

import com.netflix.graphql.dgs.DgsDataLoader;
import org.dataloader.BatchLoader;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.stream.Collectors;

@DgsDataLoader(name = "directors")
public class DirectorsDataLoader implements BatchLoader<String, Director> {

    @Autowired
    DirectorServiceClient directorServiceClient;

    @Override
    public CompletionStage<List<Director>> load(List<String> keys) {
        return CompletableFuture.supplyAsync(() -> directorServiceClient.loadDirectors(keys));
    }
}
```

Data loader 负责为一个给定的 Key 列表加载数据。在这个例子中，它只传递了一个Key 的列表给拥有的 `Director` 的后端（这可能是一个【GRPC】服务）。然而，你也需要写一个可以从数据库中加载数据的服务。尽管这个例子注册了一个 data loader，除非你去实现一个 data fetcher 去使用它，否则没人会去用它。

## 使用 Try 实现一个 Data Loader

如果你想要在部分数据获取的时候处理异常，你可以从 loader 返回一个 `Try` 对象的列表。查询结果将包含部分成功调用的结果和一个异常情况的错误。

```java
package com.netflix.graphql.dgs.example.dataLoader;

import com.netflix.graphql.dgs.DgsDataLoader;
import org.dataloader.BatchLoader;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.stream.Collectors;

@DgsDataLoader(name = "directors")
public class DirectorsDataLoader implements BatchLoader<String, Try<Director>> {

    @Autowired
    DirectorServiceClient directorServiceClient;

    @Override
    public CompletionStage<List<Try<Director>>> load(List<String> keys) {
        return CompletableFuture.supplyAsync(() -> keys.stream()
            .map(key -> Try.tryCall(() -> directorServiceClient.loadDirectors(keys)))
            .collect(Collectors.toList()));
        }
    }
}
```

## 作为 Lambda 提供

因为 `BatchLoader` 是一个功能接口（一个接口只有一个方法），你也可以作为 Lambda 表达式来提供实现。从技术上来说，这与提供一个类是一样的；它只是另一种写法：

```java
@DgsComponent
public class ExampleBatchLoaderFromField {
    @DgsDataLoader(name = "directors")
    public BatchLoader<String, Director> directorBatchLoader = keys -> CompletableFuture.supplyAsync(() -> directorServiceClient.loadDirectors(keys));
}
```

## MappedBatchLoader

`BatchLoader` 接口为一个 Key 列表创建了一个 Value 列表。你也可以为一个值的 Set 来使用创建了 Key/Value `Map` 的 `MappedBatchLoader` 。如果你不期望所有的 Key 都有一个值对应的话，后者提供了一个更好的选择。就像你注册一个 `BatchLoader` 那样，注册一个 `MappedBatchLoader`。

```java
@DgsDataLoader(name = "directors")
public class DirectorsDataLoader implements MappedBatchLoader<String, Director> {

    @Autowired
    DirectorServiceClient directorServiceClient;

    @Override
    public CompletionStage<Map<String, Director>> load(Set<String> keys) {
        return CompletableFuture.supplyAsync(() -> directorServiceClient.loadDirectors(keys));
    }
}
```

## 使用一个 Data Loader

以下是一个使用数据加载器（Data Loader）加载数据的例子：

```java
@DgsComponent
public class DirectorDataFetcher {

    @DgsData(parentType = "Movie", field = "director")
    public CompletableFuture<Director> director(DataFetchingEnvironment dfe) {

        DataLoader<String, Director> dataLoader = dfe.getDataLoader("directors");
        String id = dfe.getArgument("directorId");

        return dataLoader.load(id);
    }
}
```

上面的代码仅仅是一个普通的 data fetcher。然而，使用 data loader 来替换真正从其他服务或者数据库中获取数据。你可以从 `DataFetchingEnvironment` 里的 `getDataLoader()` 方法获取一个 data loader。这需要你传入一个 String 类型的 data loader 名称。另一个对 data fetcher 的变更是用 `CompletableFuture` 来代替原来实际的返回类型。这将会让框架异步工作，并且是批量处理的一个必须条件。

### 使用 DgsDataFetchingEnvironment

你也可以通过自定义的 `DgsDataFetchingEnvironment` 来以类型安全的方式获取 data loader。`DgsDataFetchingEnvironment` 来自于 `graphql-java` 中 `DataFetchingEnvironment` 的增强版本，并且提供了使用类名称的 `getDataLoader()` 。

```java
@DgsComponent
public class DirectorDataFetcher {

    @DgsData(parentType = "Movie", field = "director")
    public CompletableFuture<Director> director(DgsDataFetchingEnvironment dfe) {

        DataLoader<String, Director> dataLoader = dfe.getDataLoader(DirectorsDataLoader.class);
        String id = dfe.getArgument("directorId");

        return dataLoader.load(id);
    }
}
```

同样的方式也适用于将 `@DgsDataLoader` 当做 Lambda 来代替一个 Class 来定义。如果你在同一个 Class 中定义多个 `@DgsDataLoader` lambda，你不能使用这个特性。建议你通过一个 String 形式的名字来使用 `getDataLoader()` 的方式实现。

注意这里没有关于批处理是怎样工作的逻辑；这些都被框架处理了！框架将组织在加载多个 movie 的同时，加载多个 directors，为 data loader 处理所有的请求，并使用一个 ID list 代替单个 ID 调用 data loader。data loader 已经实现了如何处理 ID list 的方法，避免了 N+1 问题。

## 在 CompletableFuture 中使用 SecurityContextHolder 那样的 Spring 特性

当你写异步 data fetcher 时，代码将会执行在 worker 线程。Spring 内部存储一些 context，例如可以让 SecurityContextHolder 工作的线程。运行在不同线程的的 context 中的内部代码不可用，这使得获取与 request 相关的 Principal 将无法工作。

Spring Boot 对这有一个解决方式：它管理了一个线程池，可以处理这个 context。你可以用以下方法来注入使用：

```java
@Autowired
@DefaultExecutor
private Executor executor;
```

典型的让 data fetcher 变成异步的方法是 `supplyAsync()`，你必须将这个 executor 当做第二个参数，传递给这个方法。

```java
@DgsData(parentType = "Query", field = "list_things")
public CompletableFuture<List<Thing>> resolve(DataFetchingEnvironment environment) {
 return CompletableFuture.supplyAsync(() -> {
    return myService.getThings();
}, executor);
```

## 缓存

批处理是一个避免 N+1 问题的最重要的一方面。然而 Data loader 也支持缓存。如果加载同一个 Key 多次的话，应该只加载一次。比如，如果已经加载一个 movies 列表，并且一些 movies 有相同的 director，那么 director 数据应该只加载一次。

> 在 DGS 1 中缓存是默认关闭的
>
> 第1版的 DGS 框架默认关闭了缓存，但你可以通过 `@DgsDataLoader` 注解打开：

```java
@DgsDataLoader(name = "directors", caching=true)
class DirectorsBatchLoader implements BatchLoader<String, Director> {}
```

在 DGS 框架版本2中，默认开启了缓存，你并不需要修改上面的配置。

## Batch Size

有时候你可能要一次性加载多个元素，但需要限制容量。比如从数据库中加载数据，在 SQL 中使用了 `IN` 关键字，但是可能需要限制 ID 的最大个数。 `@DgsDataLoader` 里有一个 `maxBatchSize` 的属性，你可以配置这个行为。默认是没有最大限制的。

## Data Loader 范围

大多数情况，Data loaders 只连接一个 request。如果连接多个 request，将会引发调试困难。

