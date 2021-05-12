# Interfaces and Unions

无论何时在 schema 中使用 interface 类型或 union 类型，都必须注册类型解析器。Interface 类型 和 Union 类型在 [GraphQL documentation](https://graphql.org/learn/schema/#interfaces) 中有说明。

例如，下面的 schema 定义了一个 Movie 接口类型，它具有两个不同的具体对象类型实现。

```scheme
type Query {
    movies: [Movie]
}

interface Movie {
    title: String
}

type ScaryMovie implements Movie {
    title: String
    gory: Boolean
    scareFactor: Int
}

type ActionMovie implements Movie {
    title: String
    nrOfExplosions: Int
}
```

注册以下 datafetcher 以返回一个 movie 列表。datafetcher 返回了一个 Movie 类型的组合。

```java
@DgsComponent
public class MovieDataFetcher {
    @DgsData(parentType = "Query", field = "movies")
    public List<Movie> movies() {
        return Lists.newArrayList(
                new ActionMovie("Crouching Tiger", 0),
                new ActionMovie("Black hawk down", 10),
                new ScaryMovie("American Horror Story", true, 10),
                new ScaryMovie("Love Death + Robots", false, 4)
            );
    }
}
```

GraphQL 运行时需要知道 `ActionMovie` 的Java实例表示 `ActionMovie` 的 GraphQL 类型。这种映射是 `TypeResolver` 的职责。

> 小技巧：如果一个 Java 类型的名字与 GraphQL 中的类型名字定义一样，那么 DGS 框架将会自动创建一个 `TypeResolver`。并不需要添加任何代码。

## 注册一个 Type Resolver

如果 Java 中类型的名称和 GraphQL 类型名称不一样，你需要提供一个 `TypeResolver`。一个 Type resolver 可以帮助框架从具体的 Java 类型映射到 schema 中的正确对象类型。

使用 `@DgsTypeResolver` 注解去注册一个 type resolver。这个注解有一个 `name` 属性；将其设置为 \[GraphQL\] schema 中的 interface 类型或 union 类型的名称。resolver 接受一个 Java interface 类型的对象，并返回一个 String，该 String 是 schema 中定义的实例的具体对象类型。下面是一个针对 `Movie` 接口类型的 type resolver：

```java
@DgsTypeResolver(name = "Movie")
public String resolveMovie(Movie movie) {
    if(movie instanceof ScaryMovie) {
        return "ScaryMovie";
    } else if(movie instanceof ActionMovie) {
        return "ActionMovie";
    } else {
        throw new RuntimeException("Invalid type: " + movie.getClass().getName() + " found in MovieTypeResolver");
    }
}
```

你可以添加 `@DgsTypeResolver` 给任何一个 `@DgsComponent` 类型。这意味着您可以将 type resolver 与负责返回此类型数据的 datafetcher 保持在同一个类中，或者您可以为它创建一个单独的类。

