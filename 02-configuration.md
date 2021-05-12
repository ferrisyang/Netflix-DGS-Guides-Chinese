# 配置

## 配置 GraphQL Schema 文件的路径

你可以通过 `dgs.graphql.schema-locations` 属性来配置 GraphQL Schema 文件的路径。默认程序将会尝试通过 _Classpath_ 来读取 `schema` 的路径，比如使用 `classpath*:schema/**/*.graphql*`。我们一起看这个例子，比如我们想将这个路径从 `schema` 改成 `graphql-schemas`，你按照下面来定义你的配置：

```yaml
dgs:
    graphql:
        schema-locations:
            - classpath*:graphql-schemas/**/*.graphql*
```

现在，如果你想为 GraphQL Schema 增加额外的文件路径。例如：`graphql-experimental-schemas`

```yaml
dgs:
    graphql:
        schema-locations:
            - classpath*:graphql-schemas/**/*.graphql*
            - classpath*:graphql-experimental-schemas/**/*.graphql*
```

