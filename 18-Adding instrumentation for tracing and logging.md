# 添加用于跟踪和日志记录的工具

向 GraphQL API 添加跟踪、指标和日志记录非常有价值。在 Netflix，我们将每个 datafetcher 的跟踪范围和指标发布到分布式跟踪/指标后端，并将日志查询和查询结果发布到日志后端。我们在 Netflix 使用的实现是针对我们的基础设施的，但很容易添加您自己的框架！

DGS 框架内部使用了 [GraphQL Java](https://www.graphql-java.com/)。GraphQL Java 支持 `instrumentation` 概念。在 DGS 框架中，我们可以通过实现 `graphql.execution.instrumentation.Instrumentation` 轻松地添加一个或多个`instrumentation`，并将该类注册为 `@Component`。实现 `Instrumentation` 接口的最简单方法是扩展 `graphql.execution.instrumentation.SimpleInstrumentation`。

下面是一个将每个 datafetcher 的执行时间和总查询执行时间输出到日志的实现示例。您很可能希望将日志输出替换为写入跟踪/度量后端。请注意，代码示例说明了异步 datafetcher。如果我们不这样做，那么异步 datafetcher 的结果将始终为0，因为实际的处理在后面进行。

Java：

```java
@Component
public class ExampleTracingInstrumentation extends SimpleInstrumentation {
    private final static Logger LOGGER = LoggerFactory.getLogger(ExampleTracingInstrumentation.class);

    @Override
    public InstrumentationState createState() {
        return new TracingState();
    }

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(InstrumentationExecutionParameters parameters) {
        TracingState tracingState = parameters.getInstrumentationState();
        tracingState.startTime = System.currentTimeMillis();
        return super.beginExecution(parameters);
    }

    @Override
    public DataFetcher<?> instrumentDataFetcher(DataFetcher<?> dataFetcher, InstrumentationFieldFetchParameters parameters) {
        // We only care about user code
        if(parameters.isTrivialDataFetcher()) {
            return dataFetcher;
        }

        return environment -> {
            long startTime = System.currentTimeMillis();
            Object result = dataFetcher.get(environment);
            if(result instanceof CompletableFuture) {
                ((CompletableFuture<?>) result).whenComplete((r, ex) -> {
                    long totalTime = System.currentTimeMillis() - startTime;
                    LOGGER.info("Async datafetcher {} took {}ms", findDatafetcherTag(parameters), totalTime);
                });
            } else {
                long totalTime = System.currentTimeMillis() - startTime;
                LOGGER.info("Datafetcher {} took {}ms", findDatafetcherTag(parameters), totalTime);
            }

            return result;
        };
    }

    @Override
    public CompletableFuture<ExecutionResult> instrumentExecutionResult(ExecutionResult executionResult, InstrumentationExecutionParameters parameters) {
        TracingState tracingState = parameters.getInstrumentationState();
        long totalTime = System.currentTimeMillis() - tracingState.startTime;
        LOGGER.info("Total execution time: {}ms", totalTime);

        return super.instrumentExecutionResult(executionResult, parameters);
    }

    private String findDatafetcherTag(InstrumentationFieldFetchParameters parameters) {
        GraphQLOutputType type = parameters.getExecutionStepInfo().getParent().getType();
        GraphQLObjectType parent;
        if (type instanceof GraphQLNonNull) {
            parent = (GraphQLObjectType) ((GraphQLNonNull) type).getWrappedType();
        } else {
            parent = (GraphQLObjectType) type;
        }

        return  parent.getName() + "." + parameters.getExecutionStepInfo().getPath().getSegmentName();
    }

    static class TracingState implements InstrumentationState {
        long startTime;
    }
}
```

Kotlin：

```kotlin
@Component
class ExampleTracingInstrumentation: SimpleInstrumentation() {

    val logger : Logger = LoggerFactory.getLogger(ExampleTracingInstrumentation::class.java)

    override fun createState(): InstrumentationState {
        return TraceState()
    }

    override fun beginExecution(parameters: InstrumentationExecutionParameters): InstrumentationContext<ExecutionResult> {
        val state: TraceState = parameters.getInstrumentationState()
        state.traceStartTime = System.currentTimeMillis()

        return super.beginExecution(parameters)
    }

    override fun instrumentDataFetcher(dataFetcher: DataFetcher<*>, parameters: InstrumentationFieldFetchParameters): DataFetcher<*> {
        // We only care about user code
        if(parameters.isTrivialDataFetcher) {
            return dataFetcher
        }

        val dataFetcherName = findDatafetcherTag(parameters)

        return DataFetcher { environment ->
            val startTime = System.currentTimeMillis()
            val result = dataFetcher.get(environment)
            if(result is CompletableFuture<*>) {
                result.whenComplete { _,_ ->
                    val totalTime = System.currentTimeMillis() - startTime
                    logger.info("Async datafetcher '$dataFetcherName' took ${totalTime}ms")
                }
            } else {
                val totalTime = System.currentTimeMillis() - startTime
                logger.info("Datafetcher '$dataFetcherName': ${totalTime}ms")
            }

            result
        }
    }

    override fun instrumentExecutionResult(executionResult: ExecutionResult, parameters: InstrumentationExecutionParameters): CompletableFuture<ExecutionResult> {
        val state: TraceState = parameters.getInstrumentationState()
        val totalTime = System.currentTimeMillis() - state.traceStartTime
        logger.info("Total execution time: ${totalTime}ms")

        return super.instrumentExecutionResult(executionResult, parameters)
    }

    private fun findDatafetcherTag(parameters: InstrumentationFieldFetchParameters): String {
        val type = parameters.executionStepInfo.parent.type
        val parentType = if (type is GraphQLNonNull) {
            type.wrappedType as GraphQLObjectType
        } else {
            type as GraphQLObjectType
        }

        return "${parentType.name}.${parameters.executionStepInfo.path.segmentName}"
    }

    data class TraceState(var traceStartTime: Long = 0): InstrumentationState
}
```

Datafetcher 'Query.shows': 0ms

Total execution time: 3ms



## 开箱即用的度量

- 支持 vi the opt-in `graphql-dgs-spring-boot-micrometer` 模块.
- 提供指定的 GraphQL 度量，比如 `gql.query`, `gql.error`, and `gql.dataLoader`。
-  [Micrometer](https://micrometer.io/) 提供支持，它支持很多后端。

Gradle Groovy：

```groovy
dependencies {
    implementation 'com.netflix.graphql.dgs:graphql-dgs-spring-boot-micrometer:3.+'
}
```

Gradle Kotlin：

```kotlin
dependencies {
    implementation("com.netflix.graphql.dgs:graphql-dgs-spring-boot-micrometer:3.+")
}
```

Maven：

```xml
<dependencies>
    <dependency>
        <groupId>com.netflix.graphql.dgs</groupId>
        <artifactId>graphql-dgs-spring-boot-micrometer</artifactId>
        <version>3.+</version>
    <dependency>
</dependencies>
```

> ⚠️ 警告:
>
> 上面使用的版本只是一个示例。请通过访问  [Release page](https://github.com/Netflix/dgs-framework/releases) 来确认版本号。



### 查询计时器: gql.query

捕获给定的 GraphQL 查询，或者 mutation 的耗时。

**名称:** `gql.query`

**Tags:**

| Tag 名称          | 值                                          | 描述                                                         |
| :---------------- | :------------------------------------------ | :----------------------------------------------------------- |
| `outcome`         | `success` or `failure`                      | 操作结果，在 [ExecutionResult](https://github.com/graphql-java/graphql-java/blob/master/src/main/java/graphql/ExecutionResult.java) 中定义. |
| `queryComplexity` | one in [5, 10, 20, 50, 100, 200, 500, 1000] | 在查询中 node 的总数                                         |

queryComplexity 通常计算为 1 + childComplexity。查询复杂度对于计算查询的代价很有价值，因为它会根据查询的输入参数而变化。计算值表示为一个桶值，以减少度量的基数。

**Example Query:**

```scheme
query {
  viewer {
    repositories(first: 50) {
      edges {
        repository:node {
          name

          issues(first: 10) {
            totalCount
            edges {
              node {
                title
                bodyHTML
              }
            }
          }
        }
      }
    }
  }
}
```

**Example Calculation:**

```scheme
50          = 50 repositories
+
50 x 10     = 500 repository issues

            = 550 total nodes
```



### 错误计数器: gql.error

捕获在执行查询或变更时遇到的 GraphQL 错误的数量。记住，一个 GraphQL 请求可能有多个错误。

**名称:** `gql.error`

**Tags:**

| Tag 名称          | 描述                                                  |
| :---------------- | :---------------------------------------------------- |
| `gql.errorCode`   | GraphQL 错误码，比如 `VALIDATION`, `INTERNAL`, 等等。 |
| `gql.path`        | 错误发生的结果路径。                                  |
| `gql.errorDetail` | 如果有的话，这是包含了更多细节的可选项。              |



### Data Loader 计时器: gql.dataLoader

捕获一批查询的 data loader 调用的运行时间。如果您希望找到可能导致查询性能较差的 data loader，这将非常有用。

**名称:** `gql.dataLoader`

**Tags:**

| Tag 名称              | 描述                                              |
| :-------------------- | :------------------------------------------------ |
| `gql.loaderName`      | 一个 data loader 的名称，可以与实体类型名称一致。 |
| `gql.loaderBatchSize` | 一次批处理中查询的数量                            |



### Data Fetcher 计时器: gql.resolver

捕获每次 data fetcher 调用的运行时间。如果您希望找到可能导致查询性能较差的 data fetcher，这很有用。

> ⚠️ 警告:
>
> 这个度量在以下情况不可用：
>
> * 数据通过一个 Batch Loader 处理的情况
> * Datafetcher 是  [TrivialDataFetcher](https://github.com/graphql-java/graphql-java/blob/master/src/main/java/graphql/TrivialDataFetcher.java)。*trivial DataFetcher*  是将一个对象中的简单Map数据放到一个字段中的 datafetcher。这是直接定义在  `graphql-java`。

**名称:** `gql.resolver`

**Tags:**

| Tag 名称    | 描述                                                         |
| :---------- | :----------------------------------------------------------- |
| `gql.field` | datafetcher 的名称。它可以用`@DgsData` 注解中的  `${parentType}.${field}`  格式指定. |



#### 作为一个计时器的 Data Fetcher 定时器

data fetcher，或 resolver，计时器也可以用作一个计数器。如果以这种方式使用，它将反映每个 data fetcher 的调用次数。如果你想知道哪些  data fetchers 经常被使用，这是很有用的。



### 进一步定制标签

通过提供实现以下功能接口的 *bean*，您可以定制应用于上述指标的标记。

| 接口                         | 描述                                                         |
| :--------------------------- | :----------------------------------------------------------- |
| `DgsContextualTagCustomizer` | 用于添加常见的上下文标记。这些例子可以用来描述部署环境、应用程序概要文件、应用程序版本等 |
| `DgsExecutionTagCustomizer`  | 用于添加指定查询[ExecutionResult](https://github.com/graphql-java/graphql-java/blob/master/src/main/java/graphql/ExecutionResult.java)。例子： [SimpleGqlOutcomeTagCustomizer](https://github.com/Netflix/dgs-framework/blob/master/graphql-dgs-spring-boot-micrometer/src/main/kotlin/com/netflix/graphql/dgs/metrics/micrometer/tagging/SimpleGqlOutcomeTagCustomizer.kt) |
| `DgsFieldFetchTagCustomizer` | 用于添加指定 datafetcher 执行方法。例子：[SimpleGqlOutcomeTagCustomizer](https://github.com/Netflix/dgs-framework/blob/master/graphql-dgs-spring-boot-micrometer/src/main/kotlin/com/netflix/graphql/dgs/metrics/micrometer/tagging/SimpleGqlOutcomeTagCustomizer.kt) |



### 配置

#### management.metrics.dgs-graphql.enabled

开箱即用启用度量。默认值：true

#### management.metrics.dgs-graphql.tag-customizers.outcome.enabled

启用自定义Tag，不论是 `success` 或 `failure`，都会随着反映 GraphQL 结果而标记于 `gql.query` 和 `gql.resolver` timers。注意与 REST 相比，在 GraphQL 中，一个 HTTP OK 并不能充分说明一个响应没有错误。评估一个响应是否成功，我们需要考虑在响应体中是否存在任何 GraphQL 错误。换句话说，即使 GraphQL 响应中有错误，你也可能会获得一个 HTTP 200 的响应。

默认值：true

#### management.metrics.dgs-graphql.data-loader-instrumentation.enabled

开启 data loader 的仪表盘。默认值：true

