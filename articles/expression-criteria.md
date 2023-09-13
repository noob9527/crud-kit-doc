# 声明式查询 API 的局限与 @Criteria.Expr
目前的 CriteriaQuery API 对开发效率确实有非常大的提升，但是它的缺点是不够灵活．考虑下面的场景
```kotlin
class Post : IntEntity() {
    companion object : EntityCompanion<Post>()

    @Column
    var create_by_id: Int? = null
    @Column
    var update_by_id: Int? = null
    @Relation
    var create_by: User? = null
    @Relation
    var update_by: User? = null
}

class User : IntEntity() {
    companion object : EntityCompanion<User>()
    @Column
    var name: String = ""
}
```
如果我们需要查询所有由 John Snow 创建或修改的文章，可能的 SQL 描述是
```sql
select post.* from post
left join user user1 on post.create_by_id = user.id
left join user user2 on post.update_by_id = user.id
where user1.name = 'John Snow' or user2.name = 'John Snow'
```
此时我们应该如果定义查询类？从之前的连表查询示例中，我们知道当我们需要 join 时，可以如下声明
```kotlin
data class PostQuery(
    @Criteria.Join
    val create_by: UserQuery? = null,
    @Criteria.Join
    val update_by: UserQuery? = null,
) : CriteriaQuery<Post>()

data class UserQuery(
    @Criteria.Eq
    val name: String? = null,
) : CriteriaQuery<User>()
```
这样一来，我们可以分别通过 `PostQuery(create_by = UserQuery(name = "John Snow"))` 与 `PostQuery(update_by = UserQuery(name = "John Snow"))` 来查到创建与更新用户为 John Snow 的 post. 但如何使用 or 来连接这两个条件呢？ 我目前并没有想到一个非常完美的解决办法，只好尝试使用＂万能＂的 `@Criteria.Expr`．可是我们又该如何在表达式中引用 user 表呢？
```kotlin
data class PostQuery(
    @Criteria.Expr(expr = "???")
    val user_name: String? = null,
    @Criteria.Join
    val create_by: UserQuery? = null,
    @Criteria.Join
    val update_by: UserQuery? = null,
) : CriteriaQuery<Post>()
```
很明显的是，我们无法直接用表名，因为这里 user 表被 join 了两次．我目前想到的方案是，允许在 expr 中使用占位符来引用对应的 join 属性，然后在生成 SQL 时将其替换成最终使用的别名，具体的定义方式如下：
```kotlin
data class PostQuery(
    @Criteria.Expr(expr = "{this.create_by}.name = #{value} or {this.update_by}.name = #{value}")
    val user_name: String? = null,
    @Criteria.Join
    val create_by: UserQuery? = null,
    @Criteria.Join
    val update_by: UserQuery? = null,
) : CriteriaQuery<Post>()
```
这里的占位符依然允许嵌套，举例来说如果 UserQuery 也有一个 Join 属性
```kotlin
data class UserQuery(
    @Criteria.Join
    val another_user: UserQuery? = null,
) : CriteriaQuery<User>()
```
那么我们可以使用如下的方式引用
```kotlin
@Criteria.Expr(expr = "{this.create_by.another_user}.name = #{value}")
```
!> 需要注意的是，这里 `{this.create_by}.name` 这个表达式中的 "name" 指的是实际的列名，跟 UserQuery 是否存在一个叫 name 的属性没有任何关系．

目前为止，这是我能够相到用来处理复杂查询的最佳方案了，虽然并不十分让人满意．另一个可能的解决办法是，我们可以给 `@Criteria.Join` 添加一个别名属性，`@Criteria.Join(alias="user1")`，之后就可以在表达式中引用了 `@Criteria.Expr(expr = "user1.name = #{value} or user2.name = #{value}")`．但由于 PostQuery 可以被其他查询类复用，我们依然无法保证 "user1" 在此次查询中的唯一性．因此我最终没有考虑这个方案．

## Criteria.Expr 的缺点
字符串表达式给予了 CriteriaQuery API 极大的灵活性，但其自身也有着无法克服的缺陷，包括但不限于：

> - no compile time type check
> - poor API compatibility. (You will be suffer from changing API time to time)
> - poor SQL compatibility. (Migrate to another database can be significantly harder)
> - vulnerable to SQL injection.
> - potentially poor SQL performance

且即便带着这么多的缺点，`@Criteria.Expr` 用户依然无法通过 `@Criteria.Expr` 实现一个可以自由组合 and 与 or 条件的复杂查询。继续拿上面的例子来说，如果希望查询由"用户A"创建或者由"用户B"修改的文章，单靠一个带有 `@Criteria.Expr` 注解的属性就无法实现了。

!> 在新版本的 crud-kit 中，我们引入了新的 `@Criteria.Or`, `@Criteria.And` API, 用于弥补 `@Criteria.Expr` 的缺陷, 详情参见[Complex Query](../articles/complex-query.md)

