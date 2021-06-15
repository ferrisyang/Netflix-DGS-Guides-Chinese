# 动态 schemas

我们强烈建议主要使用以 schema 为先的开发模式。大多数 DGSs 都有一个 schema 文件，并使用声明的，基于注解的编程模型去创建 datafetcher。这就是说，在某些情况下，从另一个 source （可能动态的）生成 schema 的场景是必须的。

## 从 Code 创建一个 schema

使用 `@DgsTypeDefinitionRegistry` 注解从代码创建一个 schema。在一个 `@DgsComponent` 类中的内部方法上使用 `@DgsTypeDefinitionRegistry` 注解来提供一个 `TypeDefinitionRegistry`。

`TypeDefinitionRegistry` 是 [graphql-java](https://www.graphql-java.com/) API 的一部分。你可以使用一个 `TypeDefinitionRegistry` 来以编程的方式定义一个 schema。

注意，可以将静态 schema 文件与一个或多个 `DgsTypeDefinitionRegistry` 方法混合使用。结果是一个合并了所有注册类型的 schema。通过这种方式，您可以主要使用 schema-first 的工作流，同时返回`@DgsTypeDefinitionRegistry` 向 schema 添加一些动态部分。

以下是一个 `DgsTypeDefinitionRegistry` 的例子。

```java
@DgsComponent
public class DynamicTypeDefinitions {
    @DgsTypeDefinitionRegistry
    public TypeDefinitionRegistry registry() {
        TypeDefinitionRegistry typeDefinitionRegistry = new TypeDefinitionRegistry();

        ObjectTypeExtensionDefinition query = ObjectTypeExtensionDefinition.newObjectTypeExtensionDefinition().name("Query").fieldDefinition(
                FieldDefinition.newFieldDefinition().name("randomNumber").type(new TypeName("Int")).build()
        ).build();

        typeDefinitionRegistry.add(query);

        return typeDefinitionRegistry;
    }
}
```

这个 `TypeDefinitionRegistry` 在 `Query` 对象类型上创建了一个 `randomNumber` 字段。

## 以编程方式创建 datafetchers

如果你正在动态的创建 schema elements，很可能需要动态的创建 datafetcher。您可以使用 `@DgsCodeRegistry` 注解来以编程的方式添加 datafetcher。带注解的 `@DgsCodeRegistry` 方法有两个参数：

GraphQLCodeRegistry.Builder codeRegistryBuilder TypeDefinitionRegistry registry

方法必须返回修改后的 GraphQLCodeRegistry.Builder

下面是为上一个示例中创建的字段以编程方式创建的 datafetcher 的示例。

```java
@DgsComponent
public class DynamicDataFetcher {
@DgsCodeRegistry
public GraphQLCodeRegistry.Builder registry(GraphQLCodeRegistry.Builder codeRegistryBuilder, TypeDefinitionRegistry registry) {
DataFetcher<Integer> df = (dfe) -> new Random().nextInt();
FieldCoordinates coordinates = FieldCoordinates.coordinates("Query", "randomNumber");

        return codeRegistryBuilder.dataFetcher(coordinates, df);
    }
}
```

## 在运行时更改 schemas

在一些非常罕见的用例中，将在运行时创建 schemas/datafetcher 与动态地重新加载 schema 结合起来是很有帮助的。

可以通过实现自己的 `ReloadSchemaIndicator` 来实现这一点。

您可以使用外部信号（例如：从一个 Message Queue 读取），通过再次执行 `@DgsTypeDefinitionRegistry` 和 `@DgsCodeRegistry`，让框架重新创建 schema。如果这些方法基于外部输入创建 schema，那么系统就可以动态地重新连接它的API。

出于显而易见的原因，对于典型的 api，这不是一种应该使用的方法；通常，稳定的 APIs 是追求目标。

