# 嵌套 data fetchers

通常，为嵌套字段，datafetcher 需要从它的父级对象中获取它需要的属性数据。

比如下面的 Schema 例子：

```scheme
type Query {
  shows: [Show]
}

type Show {
  # The showId may or may not be there, depending on the scenario.
  showId: ID
  title: String
  reviews: [Review]
}

type Review {
  starRating: Int
}
```

假设我们的后端已经有了从数据存储获取 Shows 和 Reviews 的方法。注意这个例子，`getShows` 这个方法并不返回 reviews。`getReviewsForShow` 这个方法通过 show id 为一个 show 加载 reviews。

```java
interface ShowsService {
  List<Show> getShows(); //Does not include reviews
  List<Review> getReviewsForShow(int showId);   
}
```

这种情况下，你可能需要两个 datafetchers，一个为 shows，另一个为 reviews。这里有实现 datafetcher 的不同选项，每个 datafetcher 都有根据场景的 pros 和 cons。我们将讨论不同的场景和选项。



## 简单场景 - 使用 getSource

在例子中的 Schema，`Show` 类型中有 `showId`。这样可以很容易的在另一个 datafetcher 中加载 `reviews`。`DataFetcherEnvironment` 有一个 `getSource()` 方法可以返回父加载的数据。

```java
@DgsData(parentType = "Query", field = "shows")
List<Show> shows() {
  return showsService.getShows();
}

@DgsData(parentType = "Show", field = "reviews")
List<Review> reviews(DgsDataFetchingEnvironment dfe) {
  Show show = dfe.getSource();
  return showsService.getReviewsForShow(show.getShowId());
} 
```

这是最简单和最普遍的场景，但只可以使用在 `showid` 字段在 `Show` 类型中可用的情况。



## 没有 showId - 使用一个内部类型

有时你不想在 Schema 中暴露 `showId` 字段，或者我们的类型因为其他某些原因没有设置这个字段。例如，在 1:1 和 N:1 的时候，它并不会作为一个Key，在Java模型中来塑造关系。不论原因如何，这种场景下，我们不会在 `Show` 中拥有 `showId` 。

如果我们从 schema 中删除 `showId` 并且使用代码生成器，那么生成出来的 `Show` 类将不会拥有 `showId` 字段。没有 `showId` 字段将会让加载 reviews 变得有一些复杂，因为不能通过使用 `getSource()` 从 `Show` 中获得 `showId`。

`getShowsForService(int showId)` 方法表明内部（可能在 datastore 里），一个 show 会有一个 id。在这样的场景下，对比去暴露ID，我们可能有一个对 `Show` 不同的内部实现。上面提到的例子中，我们管这样被 `ShowService` 返回的类型，叫  `InternalShow` 类型。

```java
interface ShowsService {
  List<InternalShow> getShows(); //Does not include reviews
  List<Review> getReviewsForShow(int showId);   
}

class InternalShow {
  int showId;
  Sring title;

  // getters and setters
}
```

然而，这个在 GraphQL Schema 里的 `Show` 类型，没有 `showId`。

```scheme
type Show {
  title: String
  reviews: [Review]
}
```

好消息是，你可以在你的内部实例中设置一些字段，而并不是在 schema 或者查询中。框架将会创建一个 response 来应对这样额外的数据。

我们将会创建一个额外的 `ShowWithId` 包装类，用来扩展或者撰写（生成）`Show` 类型，并且添加一个 `showId` 字段。

```java
class ShowWithId {
  String showId;
  Show show;

  //Delegate all show fields
  String getTitle() {
    return show.getTitle();
  }

  static ShowWithId fromInternalShow(InternalShow internal) {
    //Create Show instance and store id.
  }
  ....
}
```

这个 `shows` datafetcher 应该返回这个包装类，来仅仅代替 `Show` 类型。

```java
@DgsData(parentType = "Query", field = "shows")
List<Show> shows() {
  return showsService.getShows().stream()
    .map(ShowWithId::fromInternalShow)
    .collect(Collectors.toList());
}
```

如说，这个额外的字段并不会影响发给客户端的响应。



## 没有 showId - 使用 local context

当 schema 类型和内部类型几乎一样的时候，使用包装类将会很好的工作。另一种方法是使用 “local context”。一个 datafetcher 可以返回一个 `DataFetcherResult<T>`，它包括了 `data`，`errors` 和 `localContext`。其中 `data` 和 `errors` 字段是从 datafetcher 中正常返回的数据和错误信息。而 `localContext` 字段则可以包括任何你想要传递给子 datafetcher 的数据。在子 datafetcher 中，可以通过 `DataFetchingEnvironment` 来获得 `localContext` ，如果不被覆写的话，并且可以继续向下一级 datafetcher 传递。

下面的例子就是 `shows` datafetcher 创建一个 `DataFetcherResult`，包含了一个 `Show` 的列表实例（不是内部类）。这个 `localContext` 被设置成一个 map，其中每一个 `show` 当做 key，而 `showId` 被当做了 value。

```java
@DgsData(parentType = "Query", field = "shows")
public DataFetcherResult<List<Show>> shows(@InputArgument("titleFilter") String titleFilter) {
    List<InternalShow> internalShows = getShows(titleFilter);

    List<Show> shows = internalShows.stream()
        .map(s -> Show.newBuilder().title(s.getTitle()).build())
        .collect(Collectors.toList());
        return DataFetcherResult.<List<Show>>newResult()
            .data(shows)
            .localContext(internalShows.stream()
            .collect(Collectors.toMap(s -> Show.newBuilder().title(s.getTitle()).build(), InternalShow::getId)))
            .build();

}

private List<InternalShow> getShows(String titleFilter) {
    if (titleFilter == null) {
        return showsService.shows();
    }

    return showsService.shows().stream().filter(s -> s.getTitle().contains(titleFilter)).collect(Collectors.toList());
}
```

`reviews` datafetcher 现在可以使用 `getSource` 和 `getLocalContext` 组合方法，用于获取一个 show 的 `showId`。

```java
@DgsData(parentType = "Show", field = "reviews")
public CompletableFuture<List<Review>> reviews(DgsDataFetchingEnvironment dfe) {

    Map<Show, Integer> shows = dfe.getLocalContext();

    Show show = dfe.getSource();
    return showsService.getReviewsForShow(shows.get(show));
}
```

这方案的一个好处就是，对比 `getSource` 来看，`localContext` 可以向下一级的 datafetcher 传递。



## 预加载

假设我们内部的数据存储允许我们很有效率的同时加载 shows 和 reviews，比如使用 SQL 的联合查询。这样的话，在 `shows` 的 datafetcher 中预加载 reviews 将会很有效率。在 `show` datafetcher 中，我们可以检查在查询中是否包括了 `reviews` 字段，并且只在有这个字段的时候，才加载 reviews 的数据。依赖于我们使用的 Java/Kotlin 类型，`Show` 类型并不是必须有一个 `reviews` 字段。如果我们使用了 DGS codegen，则将会有这个字段，我们仅仅需要在 `Show` datafetcher 中创建 `Show` 实例的时候，设置 `reviews` 字段即可。如果 `show` datafetcher 返回的类型中没有 `reviews` 字段，我们可以再次使用 `localContext` 来传递 review 数据给 `reviews` datafetcher。下面就是一个使用了 `localContext` 来预加载的例子。

```java
@DgsData(parentType = "Query", field = "shows")
public DataFetcherResult<List<Show>> shows(DataFetchingEnvironment dfe) {
    List<Show> shows = showsService.shows();
    if (dfe.getSelectionSet().contains("reviews")) {

        Map<Integer, List<Review>> reviewsForShows = reviewsService.reviewsForShows(shows.stream().map(Show::getId).collect(Collectors.toList()));

        return DataFetcherResult.<List<Show>>newResult()
            .data(shows)
            .localContext(reviewsForShows)
            .build();
    } else {
        return DataFetcherResult.<List<Show>>newResult().data(shows).build();
    }
}

@DgsData(parentType = "Show", field = "reviews")
public List<Review> reviews(DgsDataFetchingEnvironment dfe) {
    Show show = dfe.getSource();

    //Load the reviews from the pre-loaded localContext.
    Map<Integer, List<Review>> reviewsForShows = dfe.getLocalContext();
    return reviewsForShows.get(show.getId());
}
```

