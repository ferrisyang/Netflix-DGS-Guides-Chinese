# 错误处理

通常 GraphQL 通过一个在 response 中的 `errors` 结构块，来支持错误报告。Responses 可以同时包含数据部分和错误部分，比如可以返回正确处理的字段数据和发生错误的字段信息。对应发生错误的字段将会被设置为 null，并且将会在 `errors` 结构块中添加错误信息。

DGS 框架有一个开箱即用的异常处理器，根据本页面的 `Error Specification` 章节的描述来工作。这个处理器处理来自 data fetcher 的异常。任意的 `RuntimeException` 将会被转成 `INTERNAL` 类型的 `GraphQLError`。对于一些指定的异常类型，会使用一个指定 GraphQL 错误类型。

| 异常类型 | GraphQL 错误类型 | 描述 |
| :--- | :--- | :--- |
| `AccessDeniedException` | `PERMISSION_DENIED` | 当一个 `@Secured` 检查异常 |
| `DgsEntityNotFoundException` | `NOT_FOUND` | 当请求的实体（基于查询参数）找不到的时候，由开发者手动抛出异常 |

## 映射自定义异常

映射应用指定异常成为带有明确意思异常，返回给客户端是非常有用的。你可以通过注册一个 `DataFetcherExceptionHandler` 来做到。请确认扩展或者代理到 `DefaultDataFetcherExceptionHandler` 类，这是框架的默认异常处理器。如果你没有扩展/代理到这个类的话，你将会失去框架内建的异常处理器。

以下是一个自定义的异常处理实现。

```java
@Component
public class CustomDataFetchingExceptionHandler implements DataFetcherExceptionHandler {
    private final DefaultDataFetcherExceptionHandler defaultHandler = new DefaultDataFetcherExceptionHandler();

    @Override
    public DataFetcherExceptionHandlerResult onException(DataFetcherExceptionHandlerParameters handlerParameters) {
        if(handlerParameters.getException() instanceof MyException) {
            Map<String, Object> debugInfo = new HashMap<>();
            debugInfo.put("somefield", "somevalue");

            GraphQLError graphqlError = TypedGraphQLError.INTERNAL.message("This custom thing went wrong!")
                    .debugInfo(debugInfo)
                    .path(handlerParameters.getPath()).build();
            return DataFetcherExceptionHandlerResult.newResult()
                    .error(graphqlError)
                    .build();
        } else {
            return defaultHandler.onException(handlerParameters);
        }
    }
}
```

下面的 data fercher 抛出 `MyException`。

```java
@DgsComponent
public class HelloDataFetcher {
    @DgsData(parentType = "Query", field = "hello")
    @DgsEnableDataFetcherInstrumentation(false)
    public String hello(DataFetchingEnvironment dfe) {

        throw new MyException();
    }
}
```

response 查询 `hello` 字段的结果

```scheme
{
  "errors": [
    {
      "message": "This custom thing went wrong!",
      "locations": [],
      "path": [
        "hello"
      ],
      "extensions": {
        "errorType": "INTERNAL",
        "debugInfo": {
          "somefield": "somevalue"
        }
      }
    }
  ],
  "data": {
    "hello": null
  }
}
```

## 指定异常

这里有两组我们在 GraphQL 中典型遇到的错误：

1. **综合错误**：指非可预见的错误，而且不能期望终端用户能够修复。这类的错误通常适用于多种类型和字段。这样的错误将会在 GraphQL Response 中的 `errors` 数组中出现。
2. **作为数据的错误**：指能够给与终端用于以指导的错误。（例如：“标题不能使用在你的国家” 或者 “你的账户被锁定了”）。这类错误是典型指出详细用法和在特定的字段或者特定的子集中使用。这些错误是 GraphQL schema 的一部分。

### GraphQLError 接口

GraphQL 的说明提供了一个错误结构的迷你指导。它只有一个必须的，没有定义格式的 String 类型的 `message` 字段。在 Studio Edge 中我们应该有一个清晰的，更有表现力的约定。以下是我们可以使用的定义：

| 字段 | 类型 | 描述 |  |
| :--- | :--- | :--- | :--- |
| `message`\(non-nullable\) | `String!` | 描述一个错误的字符串，为开发者提供一个可以理解的错误 |  |
| `locations` | `[Location]` | 代码定位数组，每个定位将会是 Key 为 `line` 和 `column` 的 `Map` , 从1开始可以描述关联元素的自然数。 |  |
| `path` | \`\[String | Int\]\` | 如果错误是关联了一个或者多个在响应中的字段时，这个字段将会在响应中描述错误字段的路径（可以让客户端找到是否有一个真的 `null` 结果还是因为运行时异常才为 `null` ） |
| `extensions` | `[TypedError]` | [查看下面的 “The TypedError Interface”](08-error-handling.md#TypedError%20接口) |  |

```kotlin
"""
Error format as defined in GraphQL Spec
"""
interface GraphQLError {
    message: String! // Required by GraphQL Spec
    locations: [Location] // See GraphQL Spec
    path: [String | Int] // See GraphQL Spec
    extensions: TypedError
}
```

更多信息请参考 [the GraphQL specification: Errors](http://spec.graphql.org/June2018/#sec-Errors).

#### TypedError 接口

Studio Edge 定义以下 `TypedError` :

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `errorType`\(non-nullable\) | `ErrorType!` | 枚举化的错误代码，就是一个很粗的错误描述，足够客户端进行分支逻辑。 |
| `errorDetail` | `ErrorDetail` | 一个可以提供错误细节的枚举，包括它特殊的原因（因为并不能有据可查，所以需要可供更改的一个主要内容） |
| `origin` | `String` | 记录错误来源的名称（为了实例化 backend service，DGS，gateway，client library, 或者 client app 的名称） |
| `debugInfo` | `DebugInfo` | 如果请求包含了一个想要 Debug 信息的标志，这个字段将会用额外的信息显示它（就像一个 stack trace 或者来自上游服务器额外的报告） |
| `debugUri` | `String` | 一个页面的 URI，包含了有可能对 debug 错误有帮助的额外信息（应该是一个有序错误的一般页面，或者是一个有错误详细情况的特别说明页面） |

```kotlin
interface TypedError {
    """
    An error code from the ErrorType enumeration.
    An errorType is a fairly coarse characterization
    of an error that should be sufficient for client
    side branching logic.
    """
    errorType: ErrorType!

    """
    The ErrorDetail is an optional field which will
    provide more fine grained information on the error
    condition. This allows the ErrorType enumeration to
    be small and mostly static so that application branching
    logic can depend on it. The ErrorDetail provides a
    more specific cause for the error. This enumeration
    will be much larger and likely change/grow over time.
    """
    errorDetail: ErrorDetail

    """
    Indicates the source that issued the error. For example, could
    be a backend service name, a domain graph service name, or a
    gateway. In the case of client code throwing the error, this
    may be a client library name, or the client app name.
    """
    origin: String

    """
    Optionally provided based on request flag
    Could include e.g. stacktrace or info from
    upstream service
    """
    debugInfo: DebugInfo

    """
    Http URI to a page detailing additional
    information that could be used to debug
    the error. This information may be general
    to the class of error or specific to this
    particular instance of the error.
    """
    debugUri: String
}
```

#### 错误类型枚举

以下表格显示了可用的 `ErrorType` `enum` 值：

| 类型 | 描述 | HTTP 模拟 |
| :--- | :--- | :--- |
| `BAD_REQUEST` | 表示请求错误。重复同样的请求并不能解决问题。很有可能是查询或者参数不能被反序列化造成的。 | 400 Bad Request |
| `FAILED_PRECONDITION` | 表示因为系统操作不可用，操作执行被拒绝。比如，在删除文件夹的时候，这个文件夹并不是空的，则  `rmdir` 操作在一个非空的文件夹上等等。如果客户端能够在一旦得到失败，不等系统修复，立刻重试的时候，请使用  `UNAVAILABLE` . | 400 Bad Request, or 500 Internal Server Error |
| `INTERNAL` | 表示发生了不可预见的内部异常：打破了下游系统提供的常量。这个错误为很严重的错误而保留。 | 500 Internal Server Error |
| `NOT_FOUND` | 可能请求了不存在的资源（比如：错误的资源ID），或者一个资源以前存在，而现在不存在（比如：缓存过期）。提醒服务端开发者：如果一个用户实体类的请求被拒绝可以使用 `NOT_FOUND`，就像首次展示的特性或者未公开的允许列表。如果用户本身的类被拒绝，就像基于用户的权限，应该使用 `PERMISSION_DENIED`。 | 404 Not Found |
| `PERMISSION_DENIED` | 表示没有足够的权限执行这个特定的请求。`PERMISSION_DENIED` 不应被用于因为资源或配额消耗殆尽的拒绝，也不应被用于权限相关的拒绝（应该使用 `UNAUTHENTICATED` 代替）。这个错误不应暗示一个请求是否有效或者请求的资源是否存在或者满足其他前置条件。 | 403 Forbidden |
| `UNAUTHENTICATED` | 表示这个请求没有有效的权限，但是这个路径是需要权限的。 | 401 Unauthorized |
| `UNAVAILABLE` | 表示当前服务并不可用。这更像是一个临时的，有可能过一会重试一下就正常了。 | 503 Unavailable |
| `UNKNOWN` | 可能返回这个错误，比如，当从另一个地址空间收到了一个错误，对于当前的地址空间并不知道这个错误。API 引起了这个错误，但是不能返回足够的错误信息。如果客户端遇到一个不明  `errorType`，将会理解为 `UNKNOWN`。不明错误不应触发特殊的行为。它们应该与`INTERNAL` 一样被同等对待。 | 520 Unknown Error |

> HTTP 模拟仅仅是提供了一个粗糙的映射，是一个快速的概念解释，通过HTTP指定的模拟值来判断错误。

