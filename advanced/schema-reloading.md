# 热加载 schemas

DGS 框架能够很好的与一些工具配合使用，比如 JRebel。在巨大的有很多依赖的 Spring Boot 代码库中，它可以在开发中花费很少的时间来重启项目。等待项目启动将会打乱开发的工作流程。

## 为热加载开启开发模式

像 JRebel 这样的工具允许热加载代码。你可以让代码在不用重启应用的情况下，进行更新编译，并在运行的应用看到变化。活跃开发一个 DGS 经常会包含 schema 变化和连接 datafetcher。一些初始化需要获取发生的变更。开箱即用的 DGS 框架，为了尽可能的在产品中有效率，缓存了初始化，所以初始化仅仅出现在项目启动的时候。

我们可以在开发中以开发者模式配置 DGS 框架，每次 request 都会重新初始化 schema。你可以用以下三种方式开启开发者模式：

1. 设置  `dgs.reload`  配置属性为 `true`  （例如：在 `application.yml`）
2. 启用 `laptop` 文档
3. 当重载的时候，完全控制实现你自己的 `ReloadIndicator` bean。当在 [dynamic schemas](dynamic-schemas.md) 的情形下，这很有用。

