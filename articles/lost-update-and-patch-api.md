# 丢失更新问题与 patch API

## "全量更新" vs "部分更新"
截至写这篇文章时，我们只提供如下更新 Entity 的 API
```kotlin
fun update(entity: T): T
```
底层的实现也非常简单，我们根据 entity 的属性来生成一条 update sql, 可能的代码如下
```kotlin
fun updateSql(): String {
    val props = properties.filter { it.updatable }
    return """
            UPDATE $tableQualifiedName
            SET ${props.joinToString(",") { "`${it.columnName}` = #{${it.propertyName}}" }}
            WHERE `$idColumnName` = #{id}}
        """.trimIndent()
}
```
假设我们有一张调查问卷表 Questionnaire. 问卷包含两个问题：
```kotlin
class Questionnaire: IntEntity() {
    companion object : EntityCompanion<Questionnaire>()
    @Column
    var answer1: String? = null
    @Column
    var answer2: String? = null
    // ...
}
```
当用户回答问题时，我们调用 update 接口，将用户回答的结果保存在 `answer1`, `answer2` 字段．
```kotlin
@RestController
@RequestMapping(path = ["/questionnaire"])
class QuestionnaireApi: IntEntity() {
    @PutMapping("/{id}")
    fun update(
            @PathVariable id: Int,
            @RequestBody
            entity: Questionnaire
    ): Questionnaire {
        return Questionnaire.update(entity)
    }
    // ...
}
```
假设该问卷可以由多个用户协同完成，用户 A, B 同时获取了空的问卷，并且他们打算分别回答 1, 2 两个问题，于是他们可能发起如下请求：
```bash
# 用户 A 提交
curl --location --request PUT 'host/questionnaire/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "answer1": "user A answer",
    "answer2": null
}'

# 用户 B 提交
curl --location --request PUT 'host/questionnaire/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "answer1": null,
    "answer2": "user B answer"
}'
```
由于我们使用了＂全量更新＂，所以无论 A, B 用户请求的先后顺序如何，他们中间有一个人的答案会被另一个人覆盖成 null 值．这就是典型的丢失更新场景．因此，对于更新实体，我认为更理想的实现方式是，用户 A, B 分别只提交他们想修改的字段，即：
```bash
# 用户 A 提交
curl --location --request PUT 'host/questionnaire/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "answer1": "user A answer",
}'

# 用户 B 提交
curl --location --request PUT 'host/questionnaire/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "answer2": "user B answer"
}'
```
然后在生成 update sql 时，只修改用户 JSON 中提到的字段：
```kotlin
fun updateSql(): String {
    val props = properties.filter { it.updatable }
                    .filter {
                        TODO("这里我们需要某种机制过滤出用户想更新的字段")
                    }
    return """
            UPDATE $tableQualifiedName
            SET ${props.joinToString(",") { "`${it.columnName}` = #{${it.propertyName}}" }}
            WHERE `$idColumnName` = #{id}}
        """.trimIndent()
}
```
换句话说，我们希望使用＂部分更新＂而不是＂全量更新＂．

!> **注意！！！**这里只是大体上来说，对于丢失更新现象，我认为使用部分更新的方案优于全量更新，但这并不能从根本上杜绝丢失更新问题．对于成熟的处理方案，可以参考[丢失更新](https://blog.staynoob.cn/post/2019/05/the-good-old-transaction/#%E4%B8%A2%E5%A4%B1%E6%9B%B4%E6%96%B0-Lost-Updates)．

## "部分更新" 的实现
摆在我们面前的第一个问题是，如何确定哪些是用户想要更新的字段．由于 Java 中 null 值的特性，我们很容易可以想到，在生成 update sql 值，可以忽略所有值为 null 的字段．但这将导致用户无法将一个字段值更新为 null．继续以上面的例子来说，如果用户 A 希望重置自己的回答，可能会发出如下请求：
```bash
curl --location --request PUT 'host/questionnaire/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "answer1": null,
}'
```
此时用户 A 想达到的效果是，将自己的回答重新设置为 null，同时不去修改 answer2 的值．而上面的办法显然无法处理这一请求．因为上面的 JSON parse 完以后，将得到一个 answer1, answer2 都是 null 的实体对象，因此我们无法区分哪个字段需要被更新为 null, 而哪个字段不需要出现在 update 语句中．一个比较直观的解决方案是，给 update API 添加一个参数，用于指明需要更新哪些字段：
```kotlin
fun update(entity: T, fields: Set<String>): T
```
为了让使用者更容易区分＂全量更新＂与＂部分更新＂的概念，这里我们将方法名改成了 patch．同时，为了简化调用代码，我们创建了一个 Patch interface．最终的 API 如下
```kotlin
interface Patch<T> {
    val entity: T
    val fields: Set<String>
}

// ...
fun patch(patch: Patch<T>)
```
调用方式如下：
```kotlin
Questionnaire.patch(Patch.of(entity, setOf(Questionnaire::answer1.name)))
```
可能的生成 SQL 代码如下：
```kotlin
fun patchSql(fields: Set<String>): String {
    val props = properties.filter { it.updatable }
                    .filter {
                        it.propertyName in fields
                    }
    return """
            UPDATE $tableQualifiedName
            SET ${props.joinToString(",") { "`${it.columnName}` = #{${it.propertyName}}" }}
            WHERE `$idColumnName` = #{id}}
        """.trimIndent()
}
```

## 依据 JSON 创建 "Patch"
下一个问题是，如何根据用户提供的 JSON，得到 fields 参数？一个比较简单的办法是，将 JSON parse 成 map．然后取它的 keys. 如果使用 Jackson 库描述该过程即为：
```kotlin
val fields = ObjectMapper().readValue<Map<String, Any?>>(json).keys
```
当然，在实际操作中还有一些细节需要处理．比如说一些 JSON 库允许通过注解的方式为属性指定序列化后的 name．以 Jackson 举例，假如我们使用如下方式定义 Entity:
```kotlin
class Questionnaire: IntEntity() {
    companion object : EntityCompanion<Questionnaire>()
    @Column
    @JsonProperty(name = "answer_1")
    var answer1: String? = null
    @Column
    var answer2: String? = null
    // ...
}
```
这样一来，由于 `val fields = ObjectMapper().readValue<Map<String, Any?>>(json).keys` 这样的方式获取的 fields 不会考虑到 Questionnaire 实体中的 Jackson 注解．因此得到的 fields 是 `setOf("answer_1")`，这就导致 answer1 并不会被更新．这种情况下，客户端不得不使用如下 json，才能够成功更新 answer1 字段
```json5
{
  // 由 Jackson 反序列化到 answer1 字段
  "answer_1": "user A answer",
  // 这样 fields 中才有 "answer1" 这个元素
  "answer1": "user A answer"
}
```
显然，这是我们无法接受的．为了解决这样的问题，我们提供了 `JacksonPatch` 辅助类，我们修改了 `JacksonPatch` 的反序列化实现，在获取 fields 时处理 Entity 属性上可能的 `@JsonIgnore` 与 `@JsonProperty` 注解以解决上面的问题． 具体的使用方式如下：
```kotlin
@RestController
@RequestMapping(path = ["/questionnaire"])
class QuestionnaireApi: IntEntity() {
    @PutMapping("/{id}")
    fun patch(
            @PathVariable id: Int,
            @RequestBody
            patch: JacksonPatch<Post>,
    ): Questionnaire {
        return Questionnaire.patchThenGet(patch)
    }
    // ...
}
```
