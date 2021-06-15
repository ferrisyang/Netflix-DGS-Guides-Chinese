# 配置

### 核心属性

| 名称                            | 类型     | 默认值                              | 描述                              |
| :------------------------------ | :------- | :---------------------------------- | :-------------------------------- |
| dgs.graphql.path                | String   | `"/graphql"`                        | 到将服务GraphQL请求的端点的路径。 |
| dgs.graphql.schema-json.enabled | Boolean  | `true`                              | 启用 schema-json endpoint 功能.   |
| dgs.graphql.schema-json.path    | String   | `"/schema.json"`                    | 不带斜杠的schema-json端点的路径。 |
| dgs.graphql.schema-locations    | [String] | `"classpath*:schema/**/*.graphql*"` | GraphQL schema 文件的位置。       |
| dgs.graphql.graphiql.enabled    | Boolean  | `true`                              | 启用 GraphiQL 功能.               |
| dgs.graphql.graphiql.path       | String   | `"/graphiql"`                       | 不带斜杠的GraphiQL端点的路径。    |



#### 例如：配置 GraphQL Schema 文件的路径

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



### DGS 扩展标量: graphql-dgs-extended-scalars

| 名称                                              | 类型    | 默认值 | 描述                                                         |
| :------------------------------------------------ | :------ | :----- | :----------------------------------------------------------- |
| dgs.graphql.extensions.scalars.enabled            | Boolean | `true` | 注册了用于DGS框架的 graphql-java-extended-scalars 中的标量扩展。 |
| dgs.graphql.extensions.scalars.chars.enabled      | Boolean | `true` | 将注册 GraphQLChar 扩展。                                    |
| dgs.graphql.extensions.scalars.numbers.enabled    | Boolean | `true` | 将注册所有数字标量扩展，如 PositiveInt、NegativeInt 等。     |
| dgs.graphql.extensions.scalars.objects.enabled    | Boolean | `true` | 将注册 Object、Json、Url 和区域设置标量扩展。                |
| dgs.graphql.extensions.scalars.time-dates.enabled | Boolean | `true` | 将注册 DateTime、Date和 Time 标量扩展。                      |



### DGS 指标: graphql-dgs-spring-boot-micrometer

| Name                                                         | Type     | Default | Description                                                  |
| :----------------------------------------------------------- | :------- | :------ | :----------------------------------------------------------- |
| management.metrics.dgs-graphql.enabled                       | Boolean  | `true`  | 允许 DGS 的 GraphQL指标，通过 micrometer。                   |
| management.metrics.dgs-graphql.instrumentation.enabled       | Boolean  | `true`  | 启用 DGS 的 GraphQL's 基础  instrumentation; 输出 `gql.query`, `gql.resolver`, 和 `gql.error` . |
| management.metrics.dgs-graphql.data-loader-instrumentation.enabled | Boolean  | `true`  | 为 DataLoader 启用 DGS' instrumentation; 输出 `gql.dataLoader` . |
| management.metrics.dgs-graphql.tag-customizers.outcome.enabled | Boolean  | `true`  | 启用DGS的GraphQL结果标记定制器。这将会添加一个 SUCCESS 或 ERROR 的 OUTCOME tag 输出 gql. |
| management.metrics.dgs-graphql.query-signature.enabled       | Boolean  | `true`  | 启用 DGS 的 `QuerySignatureRepository`; 如果可用的指标将被标记为 `gql.query.sig.hash`. |
| management.metrics.dgs-graphql.query-signature.caching.enabled | Boolean  | `true`  | 启用 DGS' `QuerySignature` 缓存；如果设置为false，则每次请求时都会计算签名。 |
| management.metrics.dgs-graphql.tags.limiter.limit            | Integer  | 100     | 将应用于此标记的限制。这个限制的解释取决于基数限制器本身。   |
| management.metrics.dgs-graphql.autotime.percentiles          | [Double] | []      | DGS Micrometer Timers 百分比, 例如. `[0.95, 0.99, 0.50]`.[^1] |
| management.metrics.dgs-graphql.autotime.percentiles-histogram | Boolean  | `false` | 允许发布 DGS Micrometer Timers 的百分比直方图. [^1]          |



> 提示：
>
> 您可以配置百分位数，并启用百分位数直方图，直接通过 Spring Boot 中开箱即用的 per-meter 自定义。例如，为所有 `gql.*` 启用百分比直方图，你可以设置以下属性:
>
> ```properties
> management.metrics.distribution.percentiles-histogram.gql=true
> ```
>
> 更多信息，请参考 [Spring Boot's Per Meter Properties](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator.metrics.customizing.per-meter-properties)



[^1]: [Spring Boot's Per Meter Properties](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator.metrics.customizing.per-meter-properties) 可用于配置百分位数和直方图，开箱即用。

