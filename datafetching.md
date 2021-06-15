# Data fetching

在 [getting started guide](getting-started.md)，我们介绍了如何使用 `@DgsData` 注解来创建一个 data fetcher。在本章节中，我们看一下关于 datafercher 的细节。

## @DgsData, @DgsQuery, @DgsMutation 和 @DgsSubscription 注解

你在一个 Java/Kotlin 方法上，使用 `@DgsData` 注解来让它成为一个 datafetcher。这个方法必须在一个被 `@DgsComponent` 注解修饰的 class 中。这个 `@DgsData` 注解有两个参数：

| 参数 | 描述 |
| :--- | :--- |
| `parentType` | 表示 `field` 字段的类型 |
| `field` | 表示 datafercher 对应的字段 |

例如：我们有以下 Schema：

```scheme
type Query {
   shows: [Show]
}

type Show {
  title: String
  actors: [Actor]
}
```

我们可以用一个 datafetcher 来实现这个 schema。

```java
@DgsComponent
public class ShowDataFetcher {

   @DgsData(parentType = "Query", field = "shows")
   public List<Show> shows() {

       //Load shows from a database and return the list of Show objects
       return shows;
   }
}
```

如果没有设置参数 `field` ，则默认会将方法名称当做 field 名称。 `@DgsQuery`, `@DgsMutation` 以及 `@DgsSubscription` 注解是定义 datafetchers 在 `Query`, `Mutation` 和 `Subscription` 类型的快速方式。与下面的定义方式效果一样。

```java
@DgsData(parentType = "Query", field = "shows")
public List<Show> shows() { .... }

// The "field" argument is omitted. It uses the method name as the field name.
@DgsData(parentType = "Query")
public List<Show> shows() { .... }

// The parentType is "Query", the field name is derived from the method name.
@DgsQuery
public List<Show> shows() { .... }

// The parentType is "Query", the field name is explicitly specified.
@DgsQuery(field = "shows")
public List<Show> shows() { .... }
```

注意一个 datafetcher 是如何返回了一个复杂的对象或者对象的集合。你不需要为每个字段分别创建 datafetcher。框架将会把查询中定义的字段组织好一起返回。比如如下的查询：

```scheme
{
    shows {
        title
    }
}
```

尽管我们返回了例子中包含的 `title` 和 `actors` 字段的 Show 对象，但 `actors` 字段将会在响应发出之前被清除掉。

## Child datafetchers

前一个例子中假设你要从数据库中通过一条查询来获取 `Show` 的列表。用户并不关心通过 GraphQL 查询中都包含了什么字段；加载 shows 的消耗是一样的。假如指定了返回的字段，是否会增加消耗呢？比如，在获取 show 的同时，想要加载 `actors` 的信息？如果因为 `actors` 这个字段没有返回，而做额外的查询，是不明智的。

如下情形，这就是为了这个很“昂贵”的字段，而创建了1个额外的 datafetcher。

```java
@DgsData(parentType = "Query", field = "shows")
public List<Show> shows() {

    //Load shows, which doesn't include "actors"
    return shows;
}

@DgsData(parentType = "Show", field = "actors")
public List<Actor> actors(DgsDataFetchingEnvironment dfe) {

   Show show = dfe.getSource();
   actorsService.forShow(show.getId());
   return actors;
}
```

这个 `actors` 的 datafetcher 仅仅在查询中包含了这个字段时获得执行。这个 `actors` datafetcher 也介绍了一个新的概念 -- `DgsDataFetchingEnvironment`。`DgsDataFetchingEnvironment` 提供了一个访问 `context`，查询本身，data loaders，和 `source` 对象的方式。其中 `source` 对象包括了这个字段。例如：当前这个 `source` 就是一个 `Show` 对象，你可以利用 show 的 identifier 来进行 actors 的查询。

值得注意的是 `shows` datafetcher 返回了 `Show` 列表，然而 `actors` datafetcher 仅仅是给单个的 show 返回 actors。 框架将会为了每个 `Show` 去执行 `actors` datafetcher 的查询。如果说这个 `actors` 的查询是来自于一个数据库，那这将会引发 N+1 的问题。为了解决 N+1 的问题，你需要用 [data loaders](data-loaders.md)。

注：嵌套 datafetcher 和在两个关联的 datafetcher 之间传输 context，有更复杂的情形。请参考更高阶的用法 -- [nested datafetchers guide](advanced/context-passing.md)。

## 使用 @InputArgument

在 GraphQL 查询中有1个或更多参数的情况是很普遍的。根据 GraphQL 的说明，一个参数可能如下：

* 一个输入类型
* 一个 scalar
* 一个枚举

其他类型，比如输出类型，集合或者接口，都是不能当做 输入类型 处理的。

你可以在一个 datafetcher 方法参数中，使用 @InputArgument 注解来获得输入参数。框架内置 Jackson 对参数进行正确的类型转换。

```text
type Query {
    shows(title: String, filter: ShowFilter): [Show]
}

input ShowFilter {
   director: String
   genre: ShowGenre
}

enum ShowGenre {
   commedy, action, horror
}
```

我们可以按照如下方法签名写一个 datafetcher：

```java
@DgsData(parentType = "Query", field = "shows")
public List<Show> shows(@InputArgument String title, @InputArgument ShowFilter filter)
```

如果参数名称与方法参数名称不匹配，我们可以在 `@InputArgument` 中指定 `name` 参数。

### 输入参数在 Kotlin 中为空的情况

如果我们使用的是 Kotlin，我们必须注意输入参数可以为空的情形。如果 schema 定义了一个输入参数可以为空，那么代码必须用一个可以为空的类型来映射。如果用一个非空的类型接收了一个空值，那么 Kotlin 将会抛出异常。

例如：

```scheme
# name is a nullable input argument
hello(name: String): String
```

你必须将 datafetcher 写成如下这样：

```kotlin
fun hello(@InputArgument hello: String?)
```

在 Java 中不需要担心这个，类型总是可以为空。你只是需要在代码中做好空判断就好。

### 与 lists 搭配使用 @InputArgument

输入参数也可以是个 list。如果输入的参数是 list，你必须在 `@InputArgument` 中严格的指定类型：

```scheme
type Query {
    hello(people:[Person]): String
}
```

```java
public String hello(@InputArgument(collectionType = Person.class) List<Person> people)
```

### 与 Optional 搭配使用 @InputArgument

输入参数经常在 Schema 中被定义成一个 Optional 形式。你的 datafetcher 代码需要对参数进行空检查。或者你可以将参数改成 Optional 的形式。

```java
public List<Show> shows(@InputArgument(collectionType = ShowFilter.class) Optional<ShowFilter> filter)
```

你需要像用在 lists 那样，当使用复杂类型时，通过 `collectionType` 参数来提供这个复杂类型的类型。如果这个参数没有提供，那么值将会是 `Optional.empty()`。这将取决于你是否使用 `Optional`。

## 代码生成器常量

直到当前 `@DgsData` 的例子为止，我们在参数 `parentType` 和 `field` 填写的都是 string 类型的值。如果你用了 [code generation](generating-code-from-schema.md) ，那么你可以用生成的常量来替换。代码生成器将会创建一个名为 `DgsConstants` 的 Class，在这个 class 中将会有每个定义在 Schema 中的数据类型以及字段的常量定义。使用这样的常量，我们可以将一个 datafetcher 写成如下样子：

```scheme
type Query {
    shows: [Show]
}
```

```java
@DgsData(parentType = DgsConstants.QUERY_TYPE, field = DgsConstants.QUERY.Shows)
public List<Show> shows() {}
```

使用常量的好处是，你可以在编译期间，检查 schema 与 datafetcher 名称是否一致性。

## @RequestHeader 和 @RequestParam

有时候你需要在一个 datafetcher 中获取 HTTP headers 或者其他 request 内容。你可以很容易的通过 `@RequestHeader` 注解来获取 HTTP header 里的值。这个 `@RequestHeader` 注解与其他 Spring WebMVC 的注解使用方式一样。

```java
public String hello(@RequestHeader String host)
```

在技术上，headers 是值的 lists。如果设置了多个值，那么你可以用 List 当做参数类型来获取它们。否则，这些值将会串联成一个 String。

同样的，你可以使用 `@RequestParam` 来获取请求参数。 `@RequestHeader` 和 `@RequestParam` 都支持设定 `defaultValue` 和 `required` 参数。如果一个 `@RequestHeader` 或者 `@RequestParam` 被设定为 `required`， 并且没有设定 `defaultValue` 和没有提供默认值，那么将会抛出一个 `DgsInvalidInputArgumentException` 异常。

## 使用 DgsRequestData

或者说，你可以从 datafetcher context 中获得 `DgsRequestData` 对象。`DgsRequestData` 对象中，拥有作为 `HttpHeaders` 的 HTTP Headers，以及作为 `WebRequest` 的请求自身。这两个类型都来自于 Spring Web。依赖于你的运行时环境，你可以将 `WebRequest` 进行转换，例如转成一个 `ServletWebRequest` 对象。

```java
@DgsData(parentType = "Query", field = "serverName")
public String serverName(DgsDataFetchingEnvironment dfe) {
     DgsRequestData requestData =  DgsContext.getRequestData(dfe);
     return ((ServletWebRequest)requestData.getWebRequest()).getRequest().getServerName();
}
```

类似 `@InputArgument`，可以将 header 或者 参数以一个 `Optional` 的方式包起来。

## 使用上下文（context）

在之前的章节中，`DgsRequestData` 对象被描述成 datafetching _context_ 的一部分。你可以通过创建一个 `DgsCustomContextBuilder` 的方式为 datafetcher 自定义 context。

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

一个 data fetcher 可以通过调用 `getCustomContext()` 方法来获得 context：

```java
@DgsData(parentType = "Query", field = "withContext")
public String withContext(DataFetchingEnvironment dfe) {
    MyContext customContext = DgsContext.getCustomContext(dfe);
    return customContext.getCustomState();
}
```

同样的，自定义的 context 也可以用于一个 DataLoader 中。

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

