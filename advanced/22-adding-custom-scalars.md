# 添加自定义 Scalars

在 DGS 框架中添加自定义 scalar 类型很容易：创建一个实现 `graphql.schema.Coercing` 接口的类，并用 `@DgsScalar` 注释它。还要确保这个 scalar 类型也定义在 GraphQL schema 中。

例如，这是一个简单的 `LocalDateTime` 实现：

```java
@DgsScalar(name="DateTime")
public class DateTimeScalar implements Coercing<LocalDateTime, String> {
    @Override
    public String serialize(Object dataFetcherResult) throws CoercingSerializeException {
        if (dataFetcherResult instanceof LocalDateTime) {
            return ((LocalDateTime) dataFetcherResult).format(DateTimeFormatter.ISO_DATE_TIME);
        } else {
            throw new CoercingSerializeException("Not a valid DateTime");
        }
    }

    @Override
    public LocalDateTime parseValue(Object input) throws CoercingParseValueException {
        return LocalDateTime.parse(input.toString(), DateTimeFormatter.ISO_DATE_TIME);
    }

    @Override
    public LocalDateTime parseLiteral(Object input) throws CoercingParseLiteralException {
        if (input instanceof StringValue) {
            return LocalDateTime.parse(((StringValue) input).getValue(), DateTimeFormatter.ISO_DATE_TIME);
        }

        throw new CoercingParseLiteralException("Value is not a valid ISO date time");
    }
}
```

Schema：

```scheme
scalar DateTime
```

## 注册自定义 Scalars

在 `graphql-java` 的最新的版本（&gt; v15.0）中，[some scalars](https://github.com/graphql-java/graphql-java-extended-scalars)，特别是 `Long` scalar，默认是不可用的。这些是非标准 scalar，客户端（例如：JavaScript）很难可靠地处理它们。由于已弃用，您将需要显式地添加它们，为此您有几个选项。

> Tip
>
> 全部的 scalars 支持列表，请参见 [graphql-java-extended-scalars](https://github.com/graphql-java/graphql-java-extended-scalars) 项目页。在这里，您还将看到两种 schema 中使用的 scalars 的示例以及示例 query。

### 通过 graphql-dgs-extended-scalars 自动注册 Scalar 扩展

DGS 框架，从版本 3.9.2 开始，有 `graphql-dgs-extended-scalars` 模块。此模块提供 _auto-configuration_，它可以将定义在 `com.graphql-java:graphql-java-extended-scalars` 中的 scalar extension 自动注册。使用它你可以：

1. 在你的 build 中添加 `com.netflix.graphql.dgs:graphql-dgs-extended-scalars` 依赖。如果你使用的是 [DGS BOM](10-using-the-platform-bom.md)，你不需要为它指定一个版本，BOM会推荐一个。
2. 在你的 schema 定义 scalar

   [extended scalars doc](https://github.com/graphql-java/graphql-java-extended-scalars) 上可用的其他映射

`graphql-java-extended-scalars` 模块提供了一些你可以关闭注册的把手。

| 属性 | 描述 |
| :--- | :--- |
| dgs.graphql.extensions.scalars.time-dates.enabled | 如果设置成 `false`, 将不会注册 DateTime, Date, 和 Time scalar extensions. |
| dgs.graphql.extensions.scalars.objects.enabled | 如果设置成 `false`, 将不会注册 Object, Json, Url, 和 Locale scalar extensions. |
| dgs.graphql.extensions.scalars.numbers.enabled | 如果设置成 `false`, 将不会注册所有的 numeric scalar extensions 比如 PositiveInt, NegativeInt, 等等. |
| dgs.graphql.extensions.scalars.chars.enabled | 如果设置成`false`, 将不会注册 GraphQLChar extension. |
| dgs.graphql.extensions.scalars.enabled | 如果设置成 `false`, 将关闭自动注册上面的所有类型. |

> 重要
>
> 你是否正在使用 [code generation Gradle Plugin](../06-code-generation.md) ？
>
> `graphql-java-extended-scalar` 模块不会修改此类插件的行为。您需要显式定义 _type mappings_。例如，假设我们想同时使用 `Url` 和 `PositiveInt` Scalars。您必须将下面的映射添加到构建文件中。
>
> Gradle：
>
> ```groovy
> generateJava {
>     typeMapping = [
>         "Url" : "java.net.URL",
>         "PositiveInt" : "java.lang.Integer"
>     ]
> }
> ```
>
> Gradle Kotlin：
>
> ```kotlin
> generateJava {
>     typeMapping = mutableMapOf(
>         "Url" to "java.net.URL",
>         "PositiveInt" to "java.lang.Integer"
>     )
> }
> ```

### 通过 DgsRuntimeWiring 注册 Scalar Extensions

你也可以手动注册 Scalar Extensions。你需要：

1. 在你的 build 中添加 `com.graphql-java:graphql-java-extended-scalars` 依赖。如果你使用的是 [DGS BOM](10-using-the-platform-bom.md)，你不需要为它指定一个版本，BOM会推荐一个。
2. 在你的 schema 定义 scalar
3. 注册 scalar

这是一个怎样注册的例子：

Schema：

```scheme
scalar Long
```

你可以在 DGS 框架中手动注册 `Long` scalar：

```java
@DgsComponent
public class LongScalarRegistration {
    @DgsRuntimeWiring
    public RuntimeWiring.Builder addScalar(RuntimeWiring.Builder builder) {
        return builder.scalar(Scalars.GraphQLLong);
    }
}
```

