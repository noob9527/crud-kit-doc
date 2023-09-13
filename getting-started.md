# Getting Started
## 配置 Crud
```kotlin
// 这里假设你知道如何获取 DataSource (or you can just google java DataSource)
// val dataSource: javax.sql.DataSource = ...
Crud.dataSource(dataSource)
```

!> 如果是 Spring boot 项目且引用了，`crud-kit-spring-boot-starter` 依赖则不需要这一步，[CrudKitAutoConfiguration](../crud-kit-spring-boot-starter/src/main/kotlin/com/cpvsn/crud/autoconfigure/MybatisKitAutoConfiguration.kt) 会处理这部分逻辑

## 定义实体类
```kotlin
class Post: Entity<Int>() {
    companion object : EntityCompanion<Post>()

    @Column(insertable = false, updatable = false)
    override var id: Int = 0

    @Column
    var name: String = ""
}
```
!> 注意实体类必须有 id 属性，这里建议直接继承 IntEntity.

## 增删改查
接下来就可以直接调用对应的 CRUD 方法了，框架会通过反射生成对应的 SQL 。
```kotlin
// crud via companion instance
Post.save(post)
Post.update(post)
Post.find(id)
Post.delete(id)
```

## 条件查询
定义查询类
```kotlin
data class PostQuery(
        @Criteria.Eq
        val id: Int? = null,
        @Criteria.Like
        val name_like: String? = null
) : CriteriaQuery<Post>()
```
开始查询
```kotlin
val query = PostQuery(id = 1, name_like = "%foo%")
val list = Post.find(query)

// console output example
// ==> Preparing: SELECT post.* FROM post WHERE (1=1) AND (post.id = ? AND post.name like ?)
// ==> Parameters: 1(Integer), %user%(String)
```
> 关于 "Query" 的更多信息，参考[声明式条件查询](guide.md?id=声明式条件查询)

## 按需加载
给 Post 添加一对多关系
```kotlin
class Post : IntEntity() {
    companion object : EntityCompanion<Post>()

    @Column
    var name: String = ""

    @Relation
    var comments: List<PostComment>? = null
}

class PostComment : IntEntity() {
    companion object : EntityCompanion<PostComment>()

    @Column
    var post_id: Int = 0
    @Column
    var user_id: Int = 0
    @Column
    var content: String = ""

    // 注意这里默认会使用 post_id 或者 postId 作为引用属性
    // 可以使用 @Relation(reference="post_id") 显式的指定引用属性
    @Relation
    var post: Post? = null
    @Relation
    var user: User? = null
}
```
默认当调用 `Post.findAll(query)` 时，不会查询 `comments`，如果需要同时查询 `comments` 需要显式的 "include" 对应的属性名，e.g.:
```kotlin
Post.findAll(query, include = setOf("comments"))
```
include 支持“级联”，如果想要同时查询每条评论的 "user"，可以调用
```kotlin
Post.findAll(query, include = setOf("comments.user"))
```
实践中推荐使用属性引用的方式指定 include 属性，即
```kotlin
Post.findAll(query, include = Includes.setOf(
    Post::comments dot PostComment::user
))
```
这样当需要修改属性名时，可以方便的使用 IDE 重构功能，从而不影响既有代码。

> 关于 "include" 的更多信息，参考[按需加载属性](guide.md?id=按需加载属性)
