# 使用平台 BOM

使用 Platform Bill of Materials (BOM)（平台物料清单）

不管 Gradle[1] 还是 Maven[2]，都定义了一种机制，开发者可以利用属于同一个框架中的一致 Version 依赖项，或者依赖项的伞形结构，来让它们一起很好的工作。这样可以很好的解决版本冲突和帮你指出哪个依赖项的版本可以很好的一起工作。

让我们来看一个场景，并且假定你同时使用 `graphql-dgs-spring-boot-starter` 和 `graphql-dgs-subscriptions-websockets-autoconfigure`。如果没有使用 platform/BOM，你不得不去为每个依赖项的定义版本；除非版本是明确指明的，否则未来它们就又会有了分歧。如果你有一个多模块的工程，每个模块使用不同的DGS 框架依赖项的时候，例如，`graphql-dgs-client`，人工指定依赖项的版本将会很困难。**如果你使用platform/BOM**，那么你只需要定义 DGS 框架**版本一次**，它将会确定所有 DGS 框架的依赖项使用同样的版本。

DGS 框架的一个场景是，我们有两个不同的 BOM 定义，`graphql-dgs-platform-dependencies` 和 `graphql-dgs-platform`。后者仅为 DGS 模块定义版本对齐，而前者还为 DGS 框架的依赖项定义版本，如Spring、Jackson 和 Kotlin。



## 使用 Platform/BOM?

我们一起看一个例子，并假设我们想要使用 DGS 框架 3.10.2...

Gradle：

```groovy
repositories {
    mavenCentral()
}

dependencies {
    // DGS BOM/platform dependency. This is the only place you set version of DGS
    implementation(platform('com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:3.10.2'))

    // DGS dependencies. We don't have to specify a version here!
    implementation 'com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter'
    implementation 'com.netflix.graphql.dgs:graphql-dgs-subscriptions-websockets-autoconfigure'

    //Additional Jackson dependency. We don't need to specify a version, because Jackson is part of the BOM/platform definition.
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-joda'

    //Other dependencies...
}
```

Gradle Kotlin：

```kotlin
repositories {
    mavenCentral()
}

dependencies {
    //DGS BOM/platform dependency. This is the only place you set version of DGS
    implementation(platform("com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:3.10.2"))

    //DGS dependencies. We don't have to specify a version here!
    implementation("com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter")
    implementation("com.netflix.graphql.dgs:graphql-dgs-subscriptions-websockets-autoconfigure")

    //Additional Jackson dependency. We don't need to specify a version, because Jackson is part of the BOM/platform definition.
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-joda")

    //Other dependencies...
}
```

Maven：

```xml
<dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.netflix.graphql.dgs</groupId>
        <artifactId>graphql-dgs-platform-dependencies</artifactId>
        <!-- The DGS BOM/platform dependency. This is the only place you set version of DGS -->
        <version>3.10.2</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
</dependencyManagement>
<dependencies>
    <!-- DGS dependencies. We don't have to specify a version here! -->
    <dependency>
        <groupId>com.netflix.graphql.dgs</groupId>
        <artifactId>graphql-dgs-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>com.netflix.graphql.dgs</groupId>
        <artifactId>graphql-dgs-subscriptions-websockets-autoconfigure</artifactId>
    </dependency>
    <!-- Additional Jackson dependency. We don't need to specify a version, because Jackson is part of the BOM/platform definition. -->
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-joda</artifactId>
    </dependency>
    <!-- Other dependencies -->
</dependencies>
```

注意在**平台依赖上，仅仅需要指定版本**，并不是在 `graphql-dgs-spring-boot-starter` 和 `graphql-dgs-subscriptions-websockets-autoconfigure`。BOM 将对齐所有指定的 DGS 依赖，换句话说，使用相同的版本。额外，自从我们使用了 `graphql-dgs-platform-dependencies`，对于一些依赖，我们可以使用 DGS 为我们选择的一些依赖包的版本，比如 Jackson。

> 注
>
> 推荐使用平台版本。版本可以被用户覆盖，或者被你用的其他平台覆盖（比如 Spring dependency-management plugin）。

---

1. Gradle 通过 [Java Platform](https://docs.gradle.org/current/userguide/java_platform_plugin.html) 支持这个特性，请查看章节描述怎样 [consume a Java platform](https://docs.gradle.org/current/userguide/java_platform_plugin.html#sec:java_platform_consumption)。
2. Maven 通过 [BOM](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#bill-of-materials-bom-poms) 支持这个特性，注意 BOM 通过 `dependencyManagement` 块使用。

