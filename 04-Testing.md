# 测试

DGS 框架允许你写轻量级的测试，可以针对运行需求，部分启动框架。



## 例子

在写测试之前，请确认启用了 JUnit。如果你是通过 Spring Initializr 来创建的工程项目，那么这个配置是默认准备好的。

Gradle：

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()
}
```

Gradle Kotlin：

```kotlin
tasks.withType<Test> {
    useJUnitPlatform()
}
```

Maven：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

用以下的内容创建测试类，用来测试来自 [getting started](01-Getting Started.md) 的 `ShowsDatafetcher`。

Java：

```java
import com.netflix.graphql.dgs.DgsQueryExecutor;
import com.netflix.graphql.dgs.autoconfig.DgsAutoConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;


@SpringBootTest(classes = {DgsAutoConfiguration.class, ShowsDatafetcher.class})
class ShowsDatafetcherTest {

    @Autowired
    DgsQueryExecutor dgsQueryExecutor;

    @Test
    void shows() {
        List<String> titles = dgsQueryExecutor.executeAndExtractJsonPath(
                " { shows { title releaseYear }}",
                "data.shows[*].title");

        assertThat(titles).contains("Ozark");
    }
}
```

Kotlin：

```kotlin
import com.netflix.graphql.dgs.DgsQueryExecutor
import com.netflix.graphql.dgs.autoconfig.DgsAutoConfiguration
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest

@SpringBootTest(classes = [DgsAutoConfiguration::class, ShowsDataFetcher::class])
class ShowsDataFetcherTest {

    @Autowired
    lateinit var dgsQueryExecutor: DgsQueryExecutor

    @Test
    fun shows() {
        val titles : List<String> = dgsQueryExecutor.executeAndExtractJsonPath("""
            {
                shows {
                    title
                    releaseYear
                }
            }
        """.trimIndent(), "data.shows[*].title")

        assertThat(titles).contains("Ozark")
    }
}
```

使用 `@SpringBootTest` 注解将其变成一个 Spring 测试。如果你没有明确指定 `classes` ，Spring 将会启动在 classpath 中的所有组件。这样对一个比较小的项目而言，还可以接受，但是对于一个拥有很多“消耗很大”组件的项目来说，明确的添加我们所需要的测试类，将会加速测试。这样的话，我们需要将 DGS 框架配置类 `DgsAutoConfiguration` 以及需要测试的类 `ShowsDatafetcher` 添加进来。

为了执行查询，我们需要在测试中注入 `DgsQueryExecutor`。这个接口有很多可以执行查询并返回结果的方法。它将在 `/graphql` 端点上执行准确的查询代码，你不需要在测试中去处理 HTTP 相关的内容。这个 `DgsQueryExecutor` 方法接收 JSON paths，以至于你可以更容易准确的从响应中抽取你需要的数据。`DgsQueryExecutor` 也包含了一些方法（例如：`executeAndExtractJsonPathAsObject`），通过底层的 Jackson，将结果反序列化成 Java 对象。开源组件  [JsonPath library](https://github.com/json-path/JsonPath) 将为 JSON paths 提供支撑。

写更多的测试，比如验证使用了`titleFilter` 的 `ShowsDatafetcher` 的行为。你可以在 IDE 中执行这些测试，或者像其他 JUnit 测试一样，通过 Gradle/Maven 来执行。



## 为测试创建 GraphQL Queries

之前的例子中，我们手写了查询字符串。这对于很小很直接的查询是足够简洁的。然而，构造一个长查询字符串是非常让人厌烦的，尤其是在没有多行字符串支持的 Java 中。正因为此，我们可以使用 [GraphQLQueryRequest](13-Java GraphQL Client.md) 与生成用于构建请求所需要的类的 [code generation](06-Code Generation.md) 插件组合去创建一个 GraphQL request。这将是一个方便且类型安全的构建查询的方式。

请按照 [here](13-Java GraphQL Client.md#type-safe-query-api) 的指导搭建代码生成器，用于生成创建查询所必须的类。

现在，我们可以使用 `GraphQLQueryRequest` 去创建查询并且使用 `GraphQLResponse` 来抽取响应数据。

Java：

```java
@Test
public void showsWithQueryApi() {
    GraphQLQueryRequest graphQLQueryRequest = new GraphQLQueryRequest(
            new ShowsGraphQLQuery.Builder().titleFilter("Oz").build(),
            new ShowsProjectionRoot().title()
    );

    List<String> titles = dgsQueryExecutor.executeAndExtractJsonPath(graphQLQueryRequest.serialize(), "data.shows[*].title");
    assertThat(titles).containsExactly("Ozark");
}
```

Kotlin：

```kotlin
@Test
fun showsWithQueryApi() {
    val graphQLQueryRequest = GraphQLQueryRequest(
        ShowsGraphQLQuery.Builder()
            .titleFilter("Oz")
            .build(),
        ShowsProjectionRoot().title())

    val titles = dgsQueryExecutor.executeAndExtractJsonPath<List<String>>(graphQLQueryRequest.serialize(), "data.shows[*].title")
    assertThat(titles).containsExactly("Ozark")
}
```

作为 [graphql-client module](13-Java GraphQL Client.md) 一部分的 `GraphQLQueryRequest` 已经可以用于创建查询字符串，并且可以分别包装在响应中。你可以参考 GraphQLClient JavaDoc 获取更多支持方法的细节。



## 在测试中 Mocking 外部服务调用

一个 data fetcher 连接外部服务（比如：数据库 或者 gRPC 服务）是非常正常的情况。如果需要将这样的情况包含在测试中，将会增加以下两个问题：

1. 潜在因素：当你的业务包含很多外部请求的时候，测试将会执行得很慢。
2. 疑问：你的代码有 Bug 了？还是外部系统出现异常了？

大多数情况，我们最好 Mock 这些外部调用。Spring 已经有了一个非常好的支撑，就是 @Mockbean 注解，我们可以在 DGS 的测试中利用这个注解。



### 例子

我们更新一下  `Shows`  的那个例子，让它从外部数据源中加载数据，而不是之前那样返回一个固定列表。这个例子的目的仅仅是将一个固定列表改成了一个新的带有  `@Service` 注解的类。data fetcher 需要更新注入这个 `ShowsService`。

Java：

```java
public interface ShowsService {
    List<Show> shows();
}

@Service
public class ShowsServiceImpl implements ShowsService {
    @Override
    public List<Show> shows() {
        return List.of(
            new Show("Stranger Things", 2016),
            new Show("Ozark", 2017),
            new Show("The Crown", 2016),
            new Show("Dead to Me", 2019),
            new Show("Orange is the New Black", 2013)
        );
    }
}
```

Kotlin：

```kotlin
interface ShowsService {
    fun shows(): List<ShowsDataFetcher.Show>
}

@Service
class BasicShowsService : ShowsService {
    override fun shows(): List<ShowsDataFetcher.Show> {
        return listOf(
            ShowsDataFetcher.Show("Stranger Things", 2016),
            ShowsDataFetcher.Show("Ozark", 2017),
            ShowsDataFetcher.Show("The Crown", 2016),
            ShowsDataFetcher.Show("Dead to Me", 2019),
            ShowsDataFetcher.Show("Orange is the New Black", 2013)
        )
    }
}

@DgsComponent
class ShowsDataFetcher {
    @Autowired
    lateinit var showsService: ShowsService

    @DgsData(parentType = "Query", field = "shows")
    fun shows(@InputArgument("titleFilter") titleFilter: String?): List<Show> {
        return if (titleFilter != null) {
            showsService.shows().filter { it.title.contains(titleFilter) }
        } else {
            showsService.shows()
        }
    }
}
```

目前来说，现在这个例子的数据还是来源于内存，想象一下，这个 Service 真的去调用了一个外部的数据存储。我们一起尝试在测试中 Mock 这个 Service。

Java：

```java
@SpringBootTest(classes = {DgsAutoConfiguration.class, ShowsDataFetcher.class})
public class ShowsDataFetcherTests {

    @Autowired
    DgsQueryExecutor dgsQueryExecutor;

    @MockBean
    ShowsService showsService;

    @BeforeEach
    public void before() {
        Mockito.when(showsService.shows()).thenAnswer(invocation -> List.of(new Show("mock title", 2020)));
    }

    @Test
    public void showsWithQueryApi() {
        GraphQLQueryRequest graphQLQueryRequest = new GraphQLQueryRequest(
                new ShowsGraphQLQuery.Builder().build(),
                new ShowsProjectionRoot().title()
        );

        List<String> titles = dgsQueryExecutor.executeAndExtractJsonPath(graphQLQueryRequest.serialize(), "data.shows[*].title");
        assertThat(titles).containsExactly("mock title");
    }
}
```

Kotlin：

```kotlin
@SpringBootTest(classes = [DgsAutoConfiguration::class, ShowsDataFetcher::class])
class ShowsDataFetcherTest {

    @Autowired
    lateinit var
    dgsQueryExecutor:DgsQueryExecutor

    @MockBean
    lateinit var
    showsService:ShowsService

    @BeforeEach

    fun before() {
        Mockito.`when`(showsService.shows()).thenAnswer {
            listOf(ShowsDataFetcher.Show("mock title", 2020))
        }
    }

    @Test
    fun shows() {
        val titles :List<String> =dgsQueryExecutor.executeAndExtractJsonPath("""
                    {
                        shows {
                            title
                            releaseYear
                        }
                    }
                """.trimIndent(), "data.shows[*].title")

        assertThat(titles).contains("mock title")
    }
}
```



## 测试异常

至今为止，你写的都是正常测试。失败场景也很容易测试。我们使用之前 Mocked 例子来强制抛出一个异常。

Java：

```java
@Test
void showsWithException() {
Mockito.when(showsService.shows()).thenThrow(new RuntimeException("nothing to see here"));
    ExecutionResult result = dgsQueryExecutor.execute(" { shows { title releaseYear }}");
    assertThat(result.getErrors()).isNotEmpty();
    assertThat(result.getErrors().get(0).getMessage()).isEqualTo("java.lang.RuntimeException: nothing to see here");
}
```

Kotlin：

```kotlin
@Test
fun showsWithException() {
    Mockito.`when`(showsService.shows()).thenThrow(RuntimeException("nothing to see here"))

    val result = dgsQueryExecutor.execute("""
        {
            shows {
                title
                releaseYear
            }
        }
    """.trimIndent())

    assertThat(result.errors).isNotEmpty
    assertThat(result.errors[0].message).isEqualTo("java.lang.RuntimeException: nothing to see here")
}
```

当在执行查询的时候，发生了错误，这些错误将会包装在一个 `QueryException` 中。你可以很容易的查看这些错误。`QueryException` 中的 `message` 关联了所有错误。你可以通过 `getErrors()` 方法来进一步的检查错误。