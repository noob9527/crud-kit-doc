# Spring Boot Integration

## Installation
### via maven
```xml
<!-- for spring boot user -->
<dependency>
    <groupId>com.cpvsn</groupId>
    <artifactId>crud-kit-spring-boot-starter</artifactId>
    <version>${version}</version>
</dependency>
```

### via gradle
```groovy
dependencies {
    // for spring boot user
    compile("com.cpvsn:crud-kit-spring-boot-starter:${version}")
    // ...
}
```

## Auto Configuration
如果 Spring 项目有单一数据源，我们会自动使用该数据源。e.g.
```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/crud-kit-demo?zeroDateTimeBehavior=convertToNull&useSSL=false
    username: root
    password: root
```

另外，在 Spring 项目中，如果用户自己为某个 Entity 实现了 `EntityRepository` 接口，并且注册在 ApplicationContext 中，我们会将 companion instance 中对应的方法调用 forward 到该实现上。举例来说，假设我们有如下 Repository 实现
```kotlin
@Repository
class PostRepository : AbstractJdbcRepository<Int, Post>() {
    override fun save(entity: Post): Post {
        println("gotcha")
        return super.save(entity)
    }
}
```
则在调用 `Post.save(post)` 时，会使用上面的 save 方法。如果用户为某个 Entity 定义了多个 `EntityRepository` 实现类，并且没有使用 `@Primary` 注解指定要使用的实现，则会在程序启动时抛出 `org.springframework.beans.factory.NoUniqueBeanDefinitionException` 异常。

## ArgumentResolvers
为了能够在 Spring Mvc 项目中更好的使用我们的查询 API，我们会自动注册一些 ArgumentResolvers.
- SortArgumentResolver\
  e.g.
  ```kotlin
  @GetMapping
  fun list(
          @SortDefault(orders = [
              SortDefault.OrderDefault("id", direction = Sort.Direction.DESC)
          ])
          sort: Sort
  ): List<Post> {
      return Post.listAll(sort)
  }
  ```
  request:
  ```
  http://localhost/posts?sort=user.name asc,id desc
  ```
  解析结果等价于
  ```kotlin
  val sort = Sort.by(
          Sort.Order("user.name", Sort.Direction.ASC),
          Sort.Order("id", Sort.Direction.DESC),
  )
  ```
  如果请求时未指定 sort 参数，则会使用 `@SortDefault` 指定的默认值
- PageRequestArgumentResolver\
  e.g.
  ```kotlin
  @GetMapping
  fun list(
          @SortDefault(orders = [
              SortDefault.OrderDefault("id", direction = Sort.Direction.DESC)
          ])
          @PageRequestDefault(page = 1, size = 20)
          pageRequest: PageRequest
  ): Page<Post> {
      return Post.list(pageRequest)
  }
  ```
  request:
  ```
  http://localhost/posts?page=2&size=10&sort=id
  ```
  解析结果等价于
  ```kotlin
  val pageRequest = PageRequest(
          page = 2,
          size = 10,
          sort = Sort.by(Sort.Order("id"))
  )
  ```
  如果请求时未指定对应的参数，则会使用 `@PageRequestDefault` 指定的默认值
- IncludeArgumentResolver\
  e.g.
  ```kotlin
  @GetMapping
  fun list(
          include: Includes?
  ): List<Post> {
      return Post.findAll(include = include.orEmpty()
  }
  ```
  request:
  ```
  http://localhost/posts?include=user,comments.user
  ```
  解析结果等价于
  ```kotlin
  val include = Includes.setOf("user", "comments.user")
  ```
- CriteriaQueryArgumentResolver\
  Spring Mvc 默认的 ArgumentResolvers 无法正确处理类似于 `Set`, `UserQuery` 这样的类型。e.g.
  ```kotlin
  data class PostQuery(
          @Criteria.Ids
          val ids: Set<Int>? = null,
          @Criteria.Join
          val user: UserQuery? = null,
  ) : CriteriaQuery<Post>()

  data class UserQuery(
          @Criteria.Like
          val name_like: String? = null,
  ) : CriteriaQuery<User>()

  @GetMapping
  fun list(
          query: PostQuery,
  ): List<Post> {
      return Post.findAll(query)
  }
  ```
  request:
  ```
  http://localhost/posts?ids=1,2&user.name_like=%foo%
  ```
  解析结果等价于
  ```kotlin
  val query = PostQuery(
          ids = setOf(1, 2),
          user = UserQuery(name_like = "%foo%")
  )
  ```
- TypedQueryArgumentResolver\
  `CriteriaQueryArgumentResolver` 默认只对 `CriteriaQuery` 的子类生效，有时我们希望使用与 `CriteriaQuery` 相同的解析方式来解析任意 class，这时可以使用 `@TypedRequestParam` 注解，TypedQueryArgumentResolver 对使用该注解的参数生效。
  e.g.
  ```kotlin
  data class SampleRequest1(
          val ids: Set<Int>
  )

  @TypedRequestParam
  data class SampleRequest2(
          val ids: Set<Int>
  )

  @GetMapping
  fun whatever(
        @TypedRequestParam
        request1: SampleRequest1,
        request2: SampleRequest2,
  ) {
      println(request1)
      println(request2)
  }
  ```

