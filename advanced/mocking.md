# Mocking

这个指南是关于怎样为 datafetcher 提供 Mock data。这里有两个主要原因：

1. 在开发中，提供 UI 团队可以使用的 datafetcher 样例数据，在 schema 设计阶段是非常有用的。
2. 为 UI 团队撰写他们的测试，提供稳定的测试数据

可以提出这样一个参数，即这种 mock data 类型应该存在于 UI 代码中。毕竟是为了他们的测试。然而，通过将其拉入 DGS，数据的所有者可以提供可供许多团队使用的测试数据。这两种情况并不相互排斥。

## \[GraphQL\] Mocking

在 DGS 框架中的 library 支持：

1. 从 mocks 中生成静态数据
2. 为简单类型（比如 String 字段）返回生成的数据

这个 library 是模块化的，因此您可以将其用于各种工作流和用例。

Mock 框架已经是 DGS 框架的一部分。您所需要提供的只是一个或多个 `MockProvider` 实现。`MockProvider` 是一个带有 `Map<String, Object> provide()` 方法的接口。`Map` 中的每个 `key` 都是\[GraphQL\] schema 中的一个字段，可以有好几层。`Map` 中的 `value` 是您希望为此键返回的任何 mock data。

### Example

创建一个 `MockProvider` ，为你在 [getting started tutorial](../getting-started.md) 中创建的 `hello` 字段提供 Mock 数据。

```java
@Component
public class HelloMockProvider implements MockProvider {
    @NotNull
    @Override
    public Map<String, Object> provide() {
        Map<String, Object> mock = new HashMap<>();
        mock.put("hello", "Mocked hello response");
        return mock;
    }
}
```

如果再次运行应用程序并测试 `hello` 查询，您将看到它现在返回 mock 数据。

