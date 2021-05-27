# 开始

## 创建一个新的 Spring Boot 应用

DGS 框架建立在 Spring Boot 之上，所以如果你没有一个 Spring Boot 应用的话，需要创建一个新的来开始。Spring Initializr 是一个简单的开始方式。你可以利用 Gradle 或者 Maven，Java 8 版本以上或者Kotlin。我们建议使用 Gradle，因为我们在Gradle上已经有了一个很酷的 [code generation plugin](06-code-generation.md)。

只依赖 Spring Web。

![initializr](.gitbook/assets/initializr.png)

从IDE（推荐使用 Intellij）中打开这个项目。

## 添加 DGS 框架依赖

在你的 Gradle 或者 Maven 配置文件中，添加 `com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter` 依赖，dgs 版本：


=== "Gradle"
    ```groovy
    repositories {
        mavenCentral()
    }

    dependencies {
        implementation "com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter:latest.release"
    }
    ```
=== "Gradle Kotlin"
    ```kotlin
    repositories {
        mavenCentral()
    }

    dependencies {
        implementation("com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter:latest.release")
    }
    ```
=== "Maven"
    ```xml
    <dependency>
        <groupId>com.netflix.graphql.dgs</groupId>
        <artifactId>graphql-dgs-spring-boot-starter</artifactId>
        <!-- Make sure to set the latest framework version! -->
        <version>${dgs.framework.version}</version>
    </dependency>
    ```

!!! caution
    The DGS Framework requires Kotlin 1.4, and does not work with Kotlin 1.3. Older Spring Boot versions may bring in Kotlin 1.3.



> 注: DGS 框架需要 Kotlin 1.4，并且不能工作在 Kotlin 1.3上，Spring Boot 的旧版本可能用的是 Kotlin 1.3.

## 创建一个 Schema

首先从 Schema 开始开发是 DGS 框架的设计理念。框架将会从 `src/main/resources/schema` 文件夹中读取所有的 schema 文件。请创建一个 schema 文件 `src/main/resources/schema/schema.graphqls`。

```scheme
type Query {
    shows(titleFilter: String): [Show]
}

type Show {
    title: String
    releaseYear: Int
}
```

这个 Schema 允许查询 shows 的列表，并可以通过 title 进行筛选。

## 实现一个 Data Fetcher

Data fetcher 负责返回一个查询的数据。创建两个新的 class `example.ShowsDataFetcher` 和 `Show` 以及添加下面的代码。注意，我们有一个 [Codegen plugin](06-code-generation.md) 可以自动生成代码，但是在本指导里，我们手写这些 class。

Java：

```java
@DgsComponent
public class ShowsDatafetcher {

    private final List<Show> shows = List.of(
            new Show("Stranger Things", 2016),
            new Show("Ozark", 2017),
            new Show("The Crown", 2016),
            new Show("Dead to Me", 2019),
            new Show("Orange is the New Black", 2013)
    );

    @DgsQuery
    public List<Show> shows(@InputArgument String titleFilter) {
        if(titleFilter == null) {
            return shows;
        }

        return shows.stream().filter(s -> s.getTitle().contains(titleFilter)).collect(Collectors.toList());
    }
}

public class Show {
    private final String title;
    private final Integer releaseYear   ;

    public Show(String title, Integer releaseYear) {
        this.title = title;
        this.releaseYear = releaseYear;
    }

    public String getTitle() {
        return title;
    }

    public Integer getReleaseYear() {
        return releaseYear;
    }
}
```

Kotlin：

```kotlin
@DgsComponent
class ShowsDataFetcher {
    private val shows = listOf(
        Show("Stranger Things", 2016),
        Show("Ozark", 2017),
        Show("The Crown", 2016),
        Show("Dead to Me", 2019),
        Show("Orange is the New Black", 2013))

    @DgsQuery
    fun shows(@InputArgument titleFilter : String?): List<Show> {
        return if(titleFilter != null) {
            shows.filter { it.title.contains(titleFilter) }
        } else {
            shows
        }
    }

    data class Show(val title: String, val releaseYear: Int)
}
```

这就是需要的所有代码了，现在应用可以测试了！

## 通过 GraphiQL 测试应用

启动应用并打开浏览器到 `http://localhost:8080/graphiql`。GraphiQL 是一个集成在 DGS 框架中开箱即用的查询编辑器。请写如下的查询，并测试结果。

```scheme
{
    shows {
        title
        releaseYear
    }
}
```

请注意，这并不像 REST 那样，你必须在查询中指定你想要返回的字段列表。这就是 GraphQL 带来的好处，但这也对很多的开发者而言，是一个新的挑战。

GraphiQL 编辑器只是个图形界面，在你的应用服务中使用 `/graphql` 这个地址。你现在可以通过这个图形界面来很好的连接后端服务，例如可以使用 [React and the Apollo Client](https://www.apollographql.com/docs/react/)。

## 下一步

现在你已经运行了第一个 GraphQL 服务，我们建议通过以下的步骤来在后面进行改进:

* 使用 [Use the DGS Platform BOM](advanced/10-using-the-platform-bom.md)，组织 DGS 框架的依赖。
* 学习更多的 [datafetchers](03-data-fetching.md)
* 使用 [Gradle CodeGen plugin](06-code-generation.md)，为你自动生成数据类型。
* 在 JUnit 中写 [query tests](04-testing.md)
* 查看 [example projects](https://netflix.github.io/dgs/examples)

