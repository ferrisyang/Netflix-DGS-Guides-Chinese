# Mutations

DGS 框架支持与 data fetcher 同样的构造器来支持 Mutations，使用 `@DgsData` 注解。下面是一个 mutation 简单的样例：

```scheme
type Mutation {
    addRating(title: String, stars: Int):Rating
}

type Rating {
    avgStars: Float
}
```

```java
@DgsComponent
public class RatingMutation {
    @DgsData(parentType = "Mutation", field = "addRating")
    public Rating addRating(DataFetchingEnvironment dataFetchingEnvironment) {
        int stars = dataFetchingEnvironment.getArgument("stars");

        if(stars < 1) {
            throw new IllegalArgumentException("Stars must be 1-5");
        }

        String title = dataFetchingEnvironment.getArgument("title");
        System.out.println("Rated " + title + " with " + stars + " stars") ;

        return new Rating(stars);
    }
}
```

注意上面的代码，通过调用 `DataFetchingEnvironment.getArgument` 方法为 Mutation 获取输入数据，正如在 data fetcher 使用的那样。

## 输入类型

在上面的样例中，输入参数有两个标准 scalar 类型。你也可以使用复杂类型，你需要在 schema 中作为 `input` 来定义这些。一个 `input` 类型除非有一些 [some extra rules](https://graphql.org/learn/schema/#input-types) ，否则应该与 GraphQL 中的 `type` 保持一致。

根据 GraphQL 的说明，一个输入类型应该作为 `Map` 传输给 data fetcher。意思是说 `DataFetchingEnvironment.getArgument` 作为输入类型来说，应该是一个 `Map`，并不是你认为的 Java/Kotlin 陈述的那样。框架对于此有一个方便的机制，我们将会在后面讲述。首先我们先看一下一个直接使用 DataFetchingEnvironment 的样例。

```scheme
type Mutation {
    addRating(input: RatingInput):Rating
}

input RatingInput {
    title: String,
    stars: Int
}

type Rating {
    avgStars: Float
}
```

```java
@DgsComponent
public class RatingMutation {
    @DgsData(parentType = "Mutation", field = "addRating")
    public Rating addRating(DataFetchingEnvironment dataFetchingEnvironment) {

        Map<String,Object> input = dataFetchingEnvironment.getArgument("input");
        RatingInput ratingInput = new ObjectMapper().convertValue(input, RatingInput.class);

        System.out.println("Rated " + ratingInput.getTitle() + " with " + ratingInput.getStars() + " stars") ;

        return new Rating(ratingInput.getStars());
    }
}

class RatingInput {
    private String title;
    private int stars;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public int getStars() {
        return stars;
    }

    public void setStars(int stars) {
        this.stars = stars;
    }
}
```

## 作为 data fetcher 方法的输入参数

框架让获得输入参数变得容易。你可以指定参数作为 data fetcher 的方法参数。

```java
@DgsComponent
public class RatingMutation {
    @DgsData(parentType = "Mutation", field = "addRating")
    public Rating addRating(@InputArgument("input") RatingInput ratingInput) {
        //No need for custom parsing anymore!
        System.out.println("Rated " + ratingInput.getTitle() + " with " + ratingInput.getStars() + " stars") ;

        return new Rating(ratingInput.getStars());
    }
}
```

`@InputArgument` 注解对于指定输入参数的名称来说非常重要，因为参数的顺序不固定。如果没有注解，框架将会尝试使用参数的名称，但这是代码通过 [specific compiler settings](https://docs.oracle.com/javase/tutorial/reflect/member/methodparameterreflection.html) 进行编译后的唯一可能性。输入参数的类型可以被一个 `DataFetchingEnvironment` 参数组合。

```java
@DgsComponent
public class RatingMutation {
    @DgsData(parentType = "Mutation", field = "addRating")
    public Rating addRating(@InputArgument("input") RatingInput ratingInput, DataFetchingEnvironment dfe) {
        //No need for custom parsing anymore!
        System.out.println("Rated " + ratingInput.getTitle() + " with " + ratingInput.getStars() + " stars") ;
        System.out.println("DataFetchingEnvironment: " + dfe.getArgument(ratingInput));

        return new Rating(ratingInput.getStars());
    }
}
```

## Kotlin 数据类型

在 Kotlin 中，你可以用 Data Classes 来描述输入类型。然而，请确认它的字段是 `var` 或者为每个构造参数添加了 `@JsonProperty`，并且使用 `jacksonObjectMapper()` 去创建一个 Kotlin-compatible Jackson 映射。

```kotlin
data class RatingInput(var title: String, var stars: Int)
```

