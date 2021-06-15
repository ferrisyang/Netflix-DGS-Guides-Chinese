# 安全

## 通过 @Secured 进行细粒度的访问控制

DGS 框架使用了众所周知的 `@Secured` 注解集成了 Spring Security。Spring Security 自身可以通过很多方式配置，本文档不在赘述。一旦 Spring Security 被设置，你可以为你的 data fetcher 使用 `@Secured`，就像你在 SpringMVC 中为一个 REST Controller 那样。

```java
@DgsComponent
public class SecurityExampleFetchers {
    @DgsData(parentType = "Query", field = "hello")
    public String hello() {
        return "Hello to everyone";
    }      

    @Secured("admin")
    @DgsData(parentType = "Query", field = "secureGroup")
    public String secureGroup() {
        return "Hello to admins only";
    }
}
```

