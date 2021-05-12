# 代码生成器

DGS 代码生成器插件在基于你的 Domain Graph Service 的 GraphQL Schema 文件的工程编译的时候，生成代码。插件将会生成以下内容：

* Data types for types, input types, enums and interfaces
* 一个 `DgsConstants` 类，包含了类型和字段的名称
* data fetcher 的样例
* 描述你查询 API 所需要的安全的类型



## 快速开始

代码生成器已经被集成在了编译中。有一个可用的 Gradle 的插件，并且最近将会有一个社区版本的 Maven 插件 [available](https://github.com/deweyjose/graphqlcodegen)。

启用插件，添加以下内容来更新你项目的 `build.gradle` 文件：

```groovy
// Using plugins DSL
plugins {
    id "com.netflix.dgs.codegen" version "[REPLACE_WITH_CODEGEN_PLUGIN_VERSION]"
}
```

否则，你可以在你的 buildscript 中设置 classpath 依赖：

```groovy
buildscript {
   dependencies{
      classpath 'com.netflix.graphql.dgs.codegen:graphql-dgs-codegen-gradle:[REPLACE_WITH_CODEGEN_PLUGIN_VERSION]'
   }
}

apply plugin: 'com.netflix.dgs.codegen'
```

下一步，你需要添加如下的 Task 配置：

```groovy
generateJava{
   schemaPaths = ["${projectDir}/src/main/resources/schema"] // List of directories containing schema files
   packageName = 'com.example.packagename' // The package name to use to generate sources
   generateClient = true // Enable generating the type safe query API
}
```

> 注：请使用最新的插件版本，参考 [here](https://github.com/Netflix/dgs-codegen/releases)

作为你项目 build 的一部分，插件添加了一个 `generateJava` Gradle task。 `generateJava` 将会在项目的 `build/generated` 文件夹中生成代码。注意在 Kotlin 项目中，`generateJava` 任务将会默认生成代码（是的，名称是被混淆的）。这个文件夹将会自动添加成为项目的 classpath。类型作为包的一部分，被 `packageName.types` 指定，*packageName* 的值作为在你的 `build.gradle` 文件中配置设定。请确认你的项目源码根据生成的代码使用了指定的包名。

`generateJava` 在 `build/generated-examples` 里生成 data fetcher。

> 注：generateJava 不会将生成的 data fetchers 添加到你项目的源码中。这些 fetchers 主要作为后面你必须实现的基础样板代码。

你可以通过不同的，没有在插件的 `schemaPaths` 中指定的 schema 目录，将部分 schema 从代码生成器中排除。



### 解决 ”Could not initialize class graphql.parser.antlr.GraphqlLexer“ 问题

Gradle 的插件系统为所有插件使用的是扁平的 classpath，这很容易陷入 classpath 冲突问题。Codegen plugin 有一个依赖就是 ANTLR，这是一个可能被其他插件所不兼容的依赖。当你遇到类似 `Could not initialize class graphql.parser.antlr.GraphqlLexer` 错误的时候，这就是典型的 classpath 冲突问题。如果发生了这样的问题，请在你的 build 脚本中修改插件的顺序。ANTLR通常是向后兼容的，而不是向前兼容的。

如果是一个多模块的项目，你需要在 root build 文件中声明 Codegen plugin，但是不需要 apply 这个插件：

```groovy
plugins {
    id("com.netflix.dgs.codegen") version "[REPLACE_WITH_CODEGEN_PLUGIN_VERSION]" apply false

    //other plugins
}
```

在需要使用插件的模块中，你需要再次在 plugins block 中指定 Plugin，但是这次不需要版本号。

```groovy
plugins {
    id("com.netflix.dgs.codegen")
}
```

如果你使用的是旧版本的 `buildscript`  语法，那么你需要在根 `buildscript` 中添加插件依赖，然后仅仅在模块中 `apply` 。



### 映射已经存在的类型

Codegen 尝试生成能够在 Schema 中找到的每个类型，很少有例外。

1. Basic scalar types - 将会相应映射成 Java/Kotlin 类型（String，Integer 等）
2. Date and time types - 将会相应映射成 `java.time`  类型
3. PageInfo 和 RelayPageInfo - 将会映射成  `graphql.relay` 类型
4. 在 `typeMapping`  配置的类型对应映射

当你有想用自定义类型来替代一个确定的类型，你需要使用 `typeMapping` 来配置插件。这个 `typeMapping` 配置是一个 `Map` 类型，其中每一个 Key 就是一个 GraphQL 类型，每一个值是一个 Java/Kotlin 的全类型路径。

```groovy
generateJava{
   typeMapping = ["MyGraphQLType": "com.mypackage.MyJavaType"]
}
```



## 生成客户端 API

代码生成器也可以创建客户端 API 类。你可以使用这些类，从一个 GraphQL 端点使用 Java 或者在 JUnit 中使用 `QueryExecutor` 来查询数据。这个 Java GraphQL 客户端对 server-to-server 之间的通信来说，是很有用的。一个 GraphQL Java 客户端作为框架的一部分 [available](13-Java GraphQL Client.md) 。

代码生成器为每个 Query 和 Mutation 字段创建一个 `field-nameGraphQLQuery`。`*GraphQLQuery` 查询类包含每个参数字段。代码生成插件为 Query 或者 Mutation 返回的每个类型创建一个 `*ProjectionRoot`。一个 Projection 是一个指定了有哪些字段需要返回的 Builder 类。

以下是一个使用生成 API 的样例：

```java
GraphQLQueryRequest graphQLQueryRequest =
        new GraphQLQueryRequest(
            new TicksGraphQLQuery.Builder()
                .first(first)
                .after(after)
                .build(),
            new TicksConnectionProjection()
                .edges()
                    .node()
                        .date()
                        .route()
                            .name()
                            .votes()
                                .starRating()
                                .parent()
                            .grade());
```

这个 API 基于以下的 Schema 生成。`edges` 和 `node`  类型是因为 schema 使用了分页。API 允许以一个流畅的风格写查询，与用 String 写一个查询的感受类似，但增加了代码自动完成和类型安全的好处。

```groovy
type Query @extends {
    ticks(first: Int, after: Int, allowCached: Boolean): TicksConnection
}

type Tick {
    id: ID
    route: Route
    date: LocalDate
    userStars: Int
    userRating: String
    leadStyle: LeadStyle
    comments: String
}

type Votes {
    starRating: Float
    nrOfVotes: Int
}

type Route {
    routeId: ID
    name: String
    grade: String
    style: Style
    pitches: Int
    votes: Votes
    location: [String]
}

type TicksConnection {
    edges: [TickEdge]
}

type TickEdge {
    cursor: String!
    node: Tick
}
```



### 为外部服务生成 Query API

像上面那样生成一个 Query API，对测试你自己 DGS 服务非常有用。当与另一个 GraphQL 服务进行交互的时候，你的代码作为一个客户端，统一的类型 API 也同样很有用。这是使用 [DGS Client](13-Java GraphQL Client.md) 的典型用法。

当你为你自己的 Schema 和一个内部的 Schema 使用代码生成器的时候，你可能想要不同的代码生成配置。建议在你的项目中，分别创建不同的模块，一个包含外部服务的 Schema，另一个仅为了生成 Query API 而包含 codegen 配置文件。以下是一个仅生成一个 Query API 的配置样例。

```groovy
generateJava {
    schemaPaths = ["${projectDir}/composed-schema.graphqls"]
    packageName = "some.other.service"
    generateClient = true
    generateDataTypes = false
    skipEntityQueries = true
    includeQueries = ["hello"]
    includeMutations = [""]
    shortProjectionNames = true
    maxProjectionDepth = 2
}
```



## 配置代码生成器

代码生成器拥有很多配置项。以下的表格显示了 Gradle 配置项目，在命令行中也同样有效，Maven 也可以。

| 配置项                     | 描述                                                         | 默认值                             |
| :------------------------- | :----------------------------------------------------------- | :--------------------------------- |
| schemaPaths                | 包含 Schema 的文件或者文件夹                                 | src/main/resources/schema          |
| packageName                | 生成代码的基础包名称                                         |                                    |
| subPackageNameClient       | 生成查询API的子包名称                                        | client                             |
| subPackageNameDatafetchers | 生成 datafetcher 的子包名称                                  | datafetchers                       |
| subPackageNameTypes        | 生成数据类型的子包名称                                       | types                              |
| language                   | `java` 或 `kotlin`                                           | Autodetected from project          |
| typeMapping                | 类型映射，Key 是 GraphQL 类型，值是 Java 类型的全包名        |                                    |
| generateBoxedTypes         | 总是为基本类型使用封装类                                     | false (封装类只用于可能为空的情况) |
| generateClient             | 生成一个查询API                                              | false                              |
| generateDataTypes          | 生成数据类型。用于仅仅生成查询API的情况。如果 `generateClient` 是 TRUE 时，输入类型将会一直生成. | true                               |
| generateInterfaces         | 为数据类生成接口。如果你想要在你的 datafetcher 中，用更多的上下文和接口来替换数据类型的时候，扩展生成的 POJO是很有用的。 | false                              |
| generatedSourcesDir        | 为 Gradle 创建路径                                           | build                              |
| outputDir                  | 在 `generatedSourcesDir` 以下生成子文件夹路径                | generated                          |
| exampleOutputDir           | 生成 datafetcher 样例代码的路径                              | generated-examples                 |
| includeQueries             | 仅仅为已经存在的查询字段，生成对应的查询API                  | All queries defined in schema      |
| includeMutations           | 仅仅为已经存在的变更字段，生成对应的查询API                  | All mutations defined in schema    |
| skipEntityQueries          | 禁止生成联合类型查询                                         | false                              |
| shortProjectionNames       | projection 类型的名称简写。这些类型对开发者来说是不可见的。  | false                              |
| maxProjectionDepth         | 生成代码的最大 projection 深度。适用于（联合）Schema 嵌套的情况。 | 10                                 |

