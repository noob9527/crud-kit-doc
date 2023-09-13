# Guide

## Entity 映射
- one to one
    ```kotlin
    class User : IntEntity() {
        companion object : EntityCompanion<User>()
        @Relation
        var user_detail: UserDetail? = null
    }
    class UserDetail : IntEntity() {
        companion object : EntityCompanion<UserDetail>()
        @Column
        var user_id: Int = 0

        // 注意这里我们会默认使用上面的 user_id 来作为引用属性，你也可以通过 @Relation(reference = "user_id") 来手动指定引用
        @Relation
        var user: User? = null
    }
    ```

- one to many
    ```kotlin
    class Post : IntEntity() {
        companion object : EntityCompanion<Post>()
        @Relation
        var comments: List<PostComment>? = null
    }
    ```

- many to one
    ```kotlin
    class PostComment : IntEntity() {
        companion object : EntityCompanion<PostComment>()
        @Column
        var post_id: Int = 0

        // 注意这里我们会默认使用上面的 post_id 来作为引用属性，你也可以通过 @Relation(reference = "post_id") 来手动指定引用
        @Relation
        var post: Post? = null
    }
    ```

- many to many
    多对多映射一般通过中间表实现，我们没有针对这种场景做专门的注解，建议的处理方式是建中间实体，然后继续使用 1 to n, n to 1 的方式进行映射。

## @Relation 注解
我们使用 @Relation 注解的 reference 与 backReference 来定义两个类之间的关系．拿上面 Post 与 PostComment 的例子来说，如果使用 **伪 SQL** 来描述该关系即为
```
Post join PostComment on Post.reference = PostComment.backReference
```
如果在使用 `@Relation` 注解时，既没有指定 reference, 也没有指定 backReference．则我们会试图通过属性名来推断一个默认值．
- 默认 reference\
  e.g. 如果我们在 PostComment 中有一个 post 属性，则我们会在 PostComment 类中查找名为 "post_id" 或者 "postId" 的属性作为 reference．
- 默认 backReference\
  e.g. 在 Post 中有一个 comments 属性，并且找不到对应的 reference，则我们会在 PostComment 类中查找类型为 `Post` 并且有 `@Relation` 注解的属性，如果这样的属性只有一个．我们则认为它的 reference 就是我们的 backReference.

一旦我们确定了 reference 与 backReference 其中之一，则另一个默认为引用的类的 id 属性．拿上面的例子来说，"PostComment::post" 的 reference 是 `PostComment::post_id` ，因此 Post 与 PostComment 的关系可以描述为
```
PostComment join Post on PostComment.post_id = Post.id
```
有时我们也需要同时指定 reference 和 backReference. 比如说如果我们有 `Payment` 和 `Invoice` 两个类，它们之间通过 `order_id` 属性关联，则可能的映射方式如下
```kotlin
class Payment : IntEntity() {
    companion object : EntityCompanion<Payment>()
    @Column
    var order_id: Int = 0

    @Relation(reference = "order_id", backReference = "order_id")
    var invoice: Invoice? = null
}
class Invoice : IntEntity() {
    companion object : EntityCompanion<Invoice>()
    @Column
    var order_id: Int = 0

    @Relation(reference = "order_id", backReference = "order_id")
    var payment: Payment? = null
}
```

## "POJO" 映射 Entity
即使不是 Entity, 也可以通过 `@Relation` 来引用其他 Entity
```kotlin
data class Whatever(
    val user_id: Int,
) {
    @Relation
    var user: User? = null
}
```
上面的 `Whatever` 类只是普通的 "POJO"， 可以通过 `RelationJoinHandler` 来自动 join 引用的 user
```kotlin
val whatever = Whatever(user_id = 1)
RelationJoinHandler.join(whatever, Includes.setOf(Whatever::user))
```
> 这里如果你打算定义一个 "WhateverRepository", 可以直接继承 `com.cpvsn.crud.orm.repo.AbstractJoinRepository` 来为其提供与 `EntityRepository` 相同的 join 方法．

## 按需加载属性
在 Entity 中定义的任何无法直接映射到表中的某一列的属性，都会被视作按需加载属性。repository 中几乎所有查询接口都有 include 参数，用于指定本次查询需要加载哪些属性。
```kotlin
Post.findAll(query, include = setOf("comments"))
```
上面调用说明这次查询需要加载 Post 中的 comments 属性，include 允许嵌套
```kotlin
Post.findAll(query, include = setOf("comments", "comments.user"))
```
上面调用说明这次查询需要加载 Post 中的 comments 与对应 comments 中的 user 属性。下面用 json 表示查询结果
```
[
    {
        "id": 1,
        "name": "post of user 1",
        "comments": [
            {
                "id": 1,
                "post_id": 1,
                "user_id": 1,
                "content": "comment of user 1",
                "user": {
                    "id": 1,
                    "name": "user1"
                }
            },
            {
                "id": 2,
                "post_id": 1,
                "user_id": 2,
                "content": "comment of user 2",
                "user": {
                    "id": 2,
                    "name": "user2"
                }
            }
        ]
    },
    // ...
]
```
有过 Hibernate 使用经历的话，可能会担心 N+1 Query 问题，简单来说，在不用 "fetch join" 优化的情况下，对于上面的查询，hibernate 需要执行的 SQL 数量与查询结果的数量成正比（甚至可能更糟）。而我们这里只会执行3条 SQL，可能的控制台输出如下
```
==>  Preparing: SELECT post.* FROM post WHERE (1=1)
==>  Preparing: SELECT post_comment.* FROM post_comment WHERE (1=1) AND (post_id in (1,2))
==>  Preparing: SELECT user.* FROM user WHERE (1=1) AND (id in (1,2,3))
```
上面的例子中，setOf("comments", "comments.user") 可以简写成 setOf("comments.user") 框架会自动将其展开。我们更推荐使用属性引用的写法，即：
```kotlin
Includes.setOf(Post::comments dot PostComment::user)
```
这样更便于重构代码。


## include 任意属性
并不限于关联实体，有时我们需要按需加载的属性无法直接通过“外键”引用，或者不在同一数据源，或者只是计算开销比较大。e.g.
```kotlin
    class Post : IntEntity() {
        companion object : EntityCompanion<Post>()
        // 外部数据 url
        @Column
        var resource_url: String = ""
        @Relation
        var comments: List<PostComment>? = null
        // 引用任意外部数据
        var resource: ByteArray? = null
        // 计算开销较大的某个属性
        var heavy_computation_result: Double? = null
    }
```
在上面的例子中，我们自然不会像处理“关系实体”一样，自动为你获取对应的 resource 或是计算 heavy_computation_result 属性。你需要在对应的 repository 覆写 `handleJoin` 方法。e.g.
```kotlin
@Repository
class PostRepository: AbstractJdbcRepository<Int, Post>() {
    override fun handleJoin(list: List<Post>, include: Set<String>): List<Post> {
        // 注意这里不要忘记调用 super 方法，我们在该方法中 join “关系实体” 属性。
        val list = super.handleJoin(list, include)

        if(Post::resouce.name in include) {
            list.forEach {
                it.resource = fetchResource(it.resource_url)
            }
        }
        if(Post::heavy_computation_result.name in include) {
            list.forEach {
                it.heavy_computation_result = heavyCompute(it)
            }
        }

        return list
    }

    private fun heavyCompute(post: Post): Double {
        // ...
    }

    private fun fetchResource(url: String): ByteArray {
        // ...
    }
}
```


## 声明式条件查询
如果你之前有过 Spring Data Jpa/Jdbc 等项目使用经历的话，这些库会提供一个非常有意思的功能，下面是 Spring Data Jpa 的一个官方示例
```java
public interface UserRepository extends Repository<User, Long> {
  List<User> findByEmailAddressAndLastname(String emailAddress, String lastname);
}
```
如果我们需要写一个根据 User 的 emailAddress 和 lastname 属性查找 User 的方法，只需要继承它提供的 Repository 接口，并使用对应的名称定义方法就可以了，不需要自己写 SQL 或是使用一些 DSL。但这种方式也有一定的缺点，它往往需要我们定义大量功能类似的方法，并且方法名还特别长，下面也是官方的例子。
```java
public interface PersonRepository extends Repository<User, Long> {

  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname);

  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  List<Person> findByAddress_ZipCode(ZipCode zipCode);
}
```
在实际场景中有可能比上面更加夸张，有时我们一张表就需要定义数十个查询字段，这样上面的方法就变得不可行了。对此，我们采用的是如下的 API。
```kotlin
fun findAll(query: Query? = null): List<T>
```
你可以根据需要定义 Query 类，比如说
```kotlin
data class PersonQuery(
    @Criteria.Ids
    val ids: Set<Int>? = null,
    @Criteria.In
    val emailAddress_in: Set<String>? = null,
    @Criteria.Eq
    val emailAddress: String? = null,
    @Criteria.Eq
    val firstname: String? = null,
    @Criteria.Like
    val firstname_like: String? = null,
    @Criteria.Eq
    val lastname: String? = null,
    @Criteria.Eq
    val address_zip_code: String? = null,
    @Criteria.Gte
    val create_at_gte: Instant? = null,
): CriteriaQuery<Person>
```
理论上你可以在 PersonQuery 中定义任意多个属性，或者干脆定义多个不同的 "Query" 类，比如 PersonQuery1, PersonQuery2。 可能的调用查询的方式如下
```kotlin
val persons = Person.findAll(PersonQuery(
    firstname = "foo",
    emailAddress_in = setOf("foo@gmail.com", "bar@gmail.com"),
))
```

> 更多可用的查询条件参考 [Criteria 源码](./crud-kit/src/main/kotlin/com/cpvsn/crud/query/Criteria.kt)

!> 默认情况下所有 "Criteria.*" 会以 `AND` 连接，如果需要 `OR` 条件，可以使用 `Criteria.Or` 注解，参见 [Complex Query](articles/complex-query.md).

默认情况下，我们会根据 Query 类中的属性名和 "操作符" 来推断其在 Entity 中对应的属性，进一步根据 `@Column` 注解来判断其在表中的列名 ，比如说
```
class Person: IntEntity() {
    companion object : EntityCompanion<Person>()
    @Column
    var firstname: String? = null

    @Column("email") // 指明其在表中的列名是 email
    var emailAddress: String? = null
}

data class PersonQuery(
    @Criteria.Like
    val firstname_like: String? = null,  // 对应 firstname 属性，使用 "firstname" 作为列名
    @Criteria.In
    val emailAddress_in: Set<String>? = null, // 对应 emailAddress 属性，使用 "email" 作为列名
): CriteriaQuery<Person>
```
你也可以为查询条件指定对应的列名, e.g.
```kotlin
@Criteria.Contains(columnName = "firstname")
val keyword: String? = null
```

## 连表查询
在实际场景中，我们经常需要根据其他表中的字段进行过滤，比如说查找所有名字为 foo 的 user 创建的 post。因此我们可以使用嵌套的方式定义 Query 类.
```kotlin
data class PostQuery(
    @Criteria.Join
    val user: UserQuery? = null,
) : CriteriaQuery<Post>()

data class UserQuery(
    @Criteria.Eq
    val name: String? = null,
) : CriteriaQuery<User>()
```
可能的调用代码如下
```kotlin
Post.findAll(PostQuery(user = UserQuery(
    name = "foo"
)))
```
上面的代码的实现基于我们的 Post 实体中存在对应的 Relation 属性，即
```kotlin
class Post: IntEntity() {
    companion object : EntityCompanion<Post>()
    @Column
    var user_id: Int? = null
    @Relation
    var user: User? = null
}
```
这样我们才知道应该使用 user_id 来作为 "on" 的条件，才可以自动生成类似于 `join user on post.user_id = user.id` 的 SQL 代码（实际场景中由于我们支持多次 join 同一张表，因此我们会为 user 使用不同的 alias，生成的 SQL 也更加复杂）。如果我们的 Entity 中并没有对应的 Relation 属性，或者 on 的条件更复杂，我们也可以自己指定 on 条件，比如：
```
data class PostQuery(
    @Criteria.Join(on = "{this}.id = {that}.post_id and {that}.delete_at is null")
    val comments: PostCommentQuery? = null,
) : CriteriaQuery<Post>()
```
注意上面的例子中，我们建议使用 {this} 和 {that} 占位符，而不是直接使用 `post.id = post_comment.post_id`，是因为我们需要支持多次 join 同一张表，因此不建议固定表的 alias。你也许会想，这里我们 user 表明显只会 join 一次，但我们需要处理 Query 多层嵌套的情况，也许不知不觉中其他类也需要 join 同样的表，考虑下面的情况。
```kotlin
data class PostQuery(
    @Criteria.Join
    val user: UserQuery? = null,
    @Criteria.Join
    val comments: PostCommentQuery? = null,
) : CriteriaQuery<Post>()

data class UserQuery(
    @Criteria.Eq
    val name: String? = null,
) : CriteriaQuery<User>()

data class PostCommentQuery(
    @Criteria.Join
    val user: UserQuery? = null,
) : CriteriaQuery<PostComment>()
```
如果我们需要查询 name 为 "foo" 的用户发的 post，并且要求该 post 被 name 为 "bar" 的用户评论过。可能的调用代码如下
```kotlin
Post.findAll(
    user = UserQuery(
        name = "foo"
    ),
    comments = PostCommentQuery(
        user = UserQuery(
            name = "bar"
        ),
    ),
)
```
这里如果我们不为 user 生成不同的 alias，则可能产生如下错误的 SQL:
```
select * from post
left join user on post.user_id = user.id
left join post_comment on post_comment.post_id = post.id
left join user on post_comment.user_id = user.id
where user.name = "foo" and user.name = "bar"
```

## @Criteria.Expr 连表查询 (Experimental)
有时我们需要混用 or 连接词与连表查询，这样上面的 API 就处理不了了，举例来说如果我们需要查询 "foo" 发表的或评论过的文章，我们需要使用如下的方式定义：
```kotlin
data class PostQuery(
    @Criteria.Expr(expr = "{this.user}.name = #{value} or {this.comments.user}.name = #{value}")
    val post_or_comment_user_name: String?,
    @Criteria.Join
    val user: UserQuery? = null,
    @Criteria.Join
    val comments: PostCommentQuery? = null,
) : CriteriaQuery<Post>()
```
注意这里我们需要使用 {this.user} 占位符来引用 user 属性 join 的表，而不是直接使用 user.name，理由跟之前一样，我们会对同一张表使用不同的别名．对于这种写法的由来在 [声明式查询 API 的局限与 @Criteria.Expr](articles/expression-criteria.md) 中有进一步的解释．

## 自定义注解
TBD

