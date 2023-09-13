# crud-kit 与 Spring 声明式事务的一些问题

[跳转至结论](#Conclusion)

以如下 entity 声明为例
```kotlin
class Post : IntEntity() {
    companion object : EntityCompanion<Post>()

    @Relation
    var comments: List<PostComment>? = null
}

class PostComment : IntEntity() {
    companion object : EntityCompanion<PostComment>()

    @Column
    var post_id: Int = 0
    @Relation
    var post: Post? = null
}
```
如果我们希望在删除 post 时，先删除所有 comments，并且需要在同一个事务中执行这两项操作，我们可以使用 Spring 的声明式事务。比较传统的写法如下
```
@Service
class PostService {
    @Transactional
    fun delete(id: Int) {
        val post = get(id, Includes.setOf(Post::comments))
        // delete comments
        post.comments.orEmpty().forEach {
            PostComment.delete(it.id)
        }
        // delete post
        Post.delete(id)
    }
}

@RestController
@RequestMapping(path = ["/posts"])
class PostApi {
    @Autowire
    private lateinit var postService: PostService

    ```kotlin
    @DeleteMapping("/{id}")
    fun delete(
            @PathVariable id: Int
    ) {
        return postService.delete(id)
    }
}
```
在我的测试中，传统的写法下使用 crud-kit（无论是否使用 mybatis）与 Spring 声明式事务是完全兼容的。有时，我们可能希望在调用处直接使用 `Post.delete(id)` 而不是 `postService.delete(id)`，一个可能的方式是覆写 `AbstractJdbcRepository`，并通过 `@Repository` 或者 `@Service` 将其加入 Spring Context.
```kotlin
@Repository
class PostRepository : AbstractJdbcRepository<Int, Post>() {
    @Transactional // this works
    override fun save(entity: Post) {
        // some other code
        return super.save(entity)
    }

    @Transactional  // this does not work
    override fun delete(id: Int) {
        val post = get(id, Includes.setOf(Post::comments))
        post.comments.orEmpty().forEach {
            PostComment.delete(it.id)
        }
        super.delete(id)
    }
}
```
这样当我们调用 `Post.delete(id)` 时，会自动调用我们覆写的代码。** 但是因为 [Spring Core 与 Kotlin 的一个兼容问题](https://github.com/spring-projects/spring-framework/issues/26585)，上面的代码中 `save` 方法会在事务中执行，但 `delete` 方法不会。**。具体来说，如果我们覆写的方法带有会被 Kotlin 编译成 Java primitive type 的类型的参数(在我们的例子中，目前主要针对 `delete` 方法，其它带 `Int` 参数的方法都是查询方法)，则该方法上的 `@Transactional` 注解不会生效，目前临时的解决办法有两种：
1. 直接在类上加 `@Transactional` 注解，这样做的缺点是一些不需要事务，甚至于框架内部的一些不需要访问数据库的方法也会在事务中执行，同时也无法在方法级别调整 `@Transactional` 的属性(e.g. propagation, isolation etc)。
1. 在这样的方法上改为使用 "Programmatic" Transactional Management (`org.springframework.transaction.support.TransactionTemplate`)

## Conclusion
在我的测试中
1. 在 Kotlin 代码中，`AbstractJdbcRepository` 的子类的带有 `Int`（或 `Long`, `Double` 等会被 Kotlin 编译成 Java primitive type 的类型） 参数的方法上加的 `@Transactional` 注解不会生效，建议改用 `TransactionTemplate`
1. 如果使用 Spring 进行事务管理，crud-kit 提供的两种数据库访问方式（spring-jdbc 与 mybatis）在事务管理方面是兼容的。

对于使用 Spring 声明式事务的一些常见陷阱，可以参考 [Common Pitfalls of Declarative Transaction Management in Spring](http://blog.staynoob.cn/post/2019/02/common-pitfalls-of-declarative-transaction-management-in-spring/)

### Appendixes 2 Remarks
考虑 crud-kit-spring-boot-start 是否要通过类似于如下机制，将所有 `EntityRepository` 加入 Spring Context? 让 Spring 为所有 `EntityRepository` 生成代理类？
```kotlin
// this code is only for remark, I have not try it, highly likely it does not work as my expectation.
@Component
class BeanRegistry {
    @Autowired
    private lateinit var autowireCapableBeanFactory: AutowireCapableBeanFactory

    @Autowired
    private lateinit var beanFactory: ConfigurableBeanFactory

    @Autowired
    private lateinit var dataSource: DataSource
    @PostConstruct
    fun postConstruct() {
        Reflections().getSubTypesOf(Entity::class.java)
                .forEach { clazz ->
                    val repo = JdbcRepository::class.primaryConstructor!!.call(clazz.kotlin, dataSource)
                    val beanName = "${clazz.simpleName}Repository"
                    val proxy = autowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization(repo, beanName)
                    beanFactory.registerSingleton(beanName, proxy)
                }
    }
}
```
