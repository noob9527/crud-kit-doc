# Complex Query API (experimental)

## Criteria.Expr 的缺陷
一直一来，CriteriaQuery API 都很难处理复杂的条件组合查询。过去，如果要实现一个简单的 or 逻辑，我们需要使用 Criteria.Expr 来实现，`Criteria.Expr` API自身有着各种各样的缺点，详情参见[声明式查询 API 的局限与 @Criteria.Expr](../articles/expression-criteria.md)，且 Criteria.Expr 依然无法应对更加"动态"的复杂查询。举例来说，如果我们有一个 User 类，定义如下：
```kotlin
class User: IntEntity {
    companion object: EntityCompanion<Int, User>()

    @Column
    var first_name: String? = null
    @Colum
    var middle_name: String? = null
    @Column
    var last_name: String? = null
    @Column
    var email: String? = null
}
```
如果用户想要实现关键词查询，返回 first_name, middle_name, last_name 任一属性匹配的记录，我们可能需要定义如下查询类：
```kotlin
data class Query1(
    @Criteria.Expr("""
        {this}.first_name = #{value}
        or {this}.middle_name = #{value}
        or {this}.last_name = #{value}
    """)
    val keyword: String? = null,
): CriteriaQuery<User>()
```
如果此时另一个用户同样需要关键词查询，且其还需要返回 email 匹配的记录，该 `keyword` 就不适用了，只能为其单独再定义一个参数：
```kotlin
data class Query2(
    @Criteria.Expr("""
        {this}.first_name = #{value}
        or {this}.middle_name = #{value}
        or {this}.last_name = #{value}
    """)
    val keyword: String? = null,
    @Criteria.Expr("""
        {this}.first_name = #{value}
        or {this}.middle_name = #{value}
        or {this}.last_name = #{value}
        or {this}.email = #{value}
    """)
    val keyword2: String? = null,
): CriteriaQuery<User>()
```
这样如果用户希望自由组合 keyword 需要匹配的属性，我们就得为其声明多个查询参数。此外如果需要生成如下 sql:
```sql
select * from user where (first_name='John' and last_name='Snow') or email='john.snow@gmail.com'
```
需要额外使用辅助参数实现, 查询类会变得更加复杂，且难以理解，可能的定义方式如下:
```kotlin
/**
 * 难以理解的 Query 类，不要试图看懂下面的代码
 */
data class Query3(
    val combined_param1_aux_first_name: String? = null,
    val combined_param1_aux_last_name: String? = null,
    val combined_param1_aux_email: String? = null,
): CriteriaQuery<User>() {
    @Criteria.Expr("""
        ({this}.first_name = #{combined_param1_aux_first_name}
        and {this}.last_name = #{combined_param1_aux_last_name})
        or {this}.email = #{combined_param1_aux_email}
    """)
    val combined_param_1: Boolean?
        get() = if(this.combined_param1_aux_first_name == null 
            || this.combined_param1_aux_last_name == null
            || this.combined_param1_aux_email == null) null else true
}

fun findByCombinedParam(
    first_name: String,
    last_name: String,
    email: String,
): List<User>{
    return User.findAll(query = Query3(
        combined_param1_aux_first_name = first_name,
        combined_param1_aux_last_name = last_name,
        combined_param1_aux_email = email,
    ))
}
```

### Criteria.And, Criteria.Or
为了解决上面遇到的问题，我们引入了新的 `Criteria.And`, `Criteria.Or` 注解，下面我们使用新注解定义一个支持复杂查询的 Query 类：
```kotlin
data class ComplexQuery(
    @Criteria.Eq
    var first_name: String? = null,
    @Criteria.Eq
    var middle_name: String? = null,
    @Criteria.Eq
    var last_name: String? = null,

    @Criteria.And
    var and: List<ComplexQuery>? = null,
    @Criteria.Or
    var or: List<ComplexQuery>? = null,
): CriteriaQuery<User>()
```
其中在 `or` list 中的 Query 生成的条件会以 SQL `or` 关键词连接（有点类似于 ElasticSearch Boolean Query 中的 should），`and` 同理, 下面是一些使用 `ComplexQuery` 的例子：
```kotlin
/**
 * 使用 ComplexQuery API 实现上面提到的 Query1 keyword 查询
 * 
 * 等价的 sql 输出: 
 * select * from user where 1=1 and
 * (first_name=:keyword or middle_name:keyword or last_name=:keyword)
 */
fun findByKeyword(keyword: String): List<User>{
    return User.findAll(query = ComplexQuery(
        or = listOf(
            ComplexQuery(first_name = keyword),
            ComplexQuery(middle_name = keyword),
            ComplexQuery(last_name = keyword),
        ),
    ))
}

/**
 * 使用 ComplexQuery API 实现上面提到的 Query2 keyword2 查询
 * 
 * 等价的 sql 输出:
 * select * from user where 1=1 and
 * (first_name=:keyword or middle_name=:keyword or last_name:keyword or email=:keyword)
 */
fun findByKeyword2(keyword: String): List<User>{
    return User.findAll(query = ComplexQuery(
        or = listOf(
            ComplexQuery(first_name = keyword),
            ComplexQuery(middle_name = keyword),
            ComplexQuery(last_name = keyword),
            ComplexQuery(email = keyword),
        ),
    ))
}

/**
 * 使用 ComplexQuery API 实现上面提到的 Query3 combined_param_1 查询
 *
 * 等价的 sql 输出:
 * select * from user where 1=1 and
 * ((first_name=:first_name and last_name=:last_name) or email=:email)
 */
fun findByCombinedParam(
    first_name: String,
    last_name: String,
    email: String,
): List<User>{
    return User.findAll(query = ComplexQuery(
        or = listOf(
            ComplexQuery(
                first_name = first_name,
                last_name = last_name
            ),
            ComplexQuery(email = email),
        ),
    ))
    // 或
    // return User.findAll(query = ComplexQuery(
    //     or = listOf(
    //         ComplexQuery(
    //             // 嵌套使用
    //             and = listOf(
    //                 ComplexQuery(first_name = first_name),
    //                 ComplexQuery(last_name = last_name),
    //             ),
    //         ),
    //         ComplexQuery(email = email),
    //     ),
    // ))
}
```
如此一来，嵌套的使用 `ComplexQuery` 的 `and` 与 `or` 属性就可以定义出任意复杂的条件组合，且不再需要多次 "hardcode" 数据库列名，代码更加易于重构，解决了之前使用 Criteria.Expr 组合条件的所有问题。

### Spring MVC Integration
为了能更加方便集成 crud-kit 与 Spring-MVC, `crud-kit-spring-boot-starter` 包会自动注册一些`ArgumentResolver`, 其中 [CriteriaQueryArgumentResolver](../../crud-kit-spring-boot-starter/src/main/kotlin/com/cpvsn/crud/spring/resolver/CriteriaQueryArgumentResolver.kt) 会自动使用 URL 参数来创建 Query 实例（参见[ArgumentResolvers](../spring-integration.md?id=argumentresolvers)）。以前面的 `ComplexQuery` 为例，Url 参数：
```
?first_name=John&last_name=Snow
```
在经过处理后，会生成如下 Query 实例:
```kotlin
ComplexQuery(
    first_name = "John",
    last_name = "Snow",
)
```
而在新增了 `Criteria.And`, `Criteria.Or` API 后，我们需要一种新的格式，才能让前端用户在 Url 参数中为 "and", "or" 属性赋值。目前，我们采用 `${property_name}-${index}=${value}` 的格式来为对象集合中指定 index 的对象属性赋值，举例来说，如果我们想创建如下 query 实例：
```kotlin
ComplexQuery(
    or = listOf(
        ComplexQuery(first_name = "foo"),
        ComplexQuery(middle_name = "bar"),
        ComplexQuery(last_name = "baz"),
    ),
)
```
则请求的 Url 参数部分应该为
```
?or-0.first_name=foo&or-1.middle_name=bar&or-2.last_name=baz
```
由于这种 url 传参格式可读性并不好，因此建议新的代码使用 JSON 格式的 "Request Body" 来传递查询参数，下面是一个例子：
```json5
{
// other conditions...
//  "first_name": "whatever",
  "or": [
    {"first_name":  "foo"},
    {"middle_name":  "bar"},
    {"last_name":  "baz"}
  ],
//  "and": [
//    {"first_name":  "foo"},
//    {"middle_name":  "bar"},
//    {"last_name":  "baz"}
//  ]
}
```
对应的后端接口定义如下：
```kotlin
@PostMapping("/search")
fun list2(
    @RequestBody query: ComplexQuery,
    extra: Includes?,
    @PageRequestDefault(page = 1, size = 20)
    @SortDefault(
        orders = [
            SortDefault.OrderDefault(
                name = "id",
                direction = Sort.Direction.DESC
            )
        ]
    )
    pageRequest: PageRequest
): Page<User> {
    return User.list(
        query,
        pageRequest,
        extra.orEmpty()
    )
}
```

## Warning
由于 Complex Query API 目前还处于实验阶段，未来上述 API 可能会有修改，如果出现无法解决的兼容问题，该 API 甚至可能被移除。
