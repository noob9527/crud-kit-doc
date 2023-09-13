# TsId API

## [TsId](articles/tsid-primary-key)
`crud-kit` 中的 `TsId` 接口提供对 TSID 的抽象：
```kotlin
interface TsId : Comparable<TsId> {
    val stringValue: String
    val longValue: Long
    // ...
}
```
其中，stringValue 返回 TSID encode 后的字符串，默认实现使用 [Crockford's base32](https://www.crockford.com/base32.html) encode.  
```kotlin
val tsid = TsIds.create()
println(tsid.longValue)                         // => 3141577245813259572
println(tsid.stringValue)                       // => 2Q68ZVGXJCF9M
```
使用 TsId 的伴生对象，通过 stringValue 或 longValue 获取 TsId 实例：
```kotlin
val tsid1 = TsId.from(3141577245813259572)
val tsid2 = TsId.from("2Q68ZVGXJCF9M")
```

## @GeneratedTsId
定义 Entity 时，使用 `@GeneratedTsId` 来标注需要生成 TSID 的属性，e.g.
```kotlin
class Sample : StrEntity() {
    companion object : EntityCompanion<String, Sample>()

    @Column
    @GeneratedTsId
    override var id: String = ""

    @Column
    @GeneratedTsId
    var long_id: Long = 0
}
```
@GeneratedTsId 可以用在 String 或 Long 类型的属性上，在 Entity 保存时，会自动调用 `TsIds.create()` 生成一个 TsId 并保存对应的 stringValue 或 longValue. @GeneratedTsId 只会在属性当前值为 null, "", 0L 的时候创建新的 TsId，举例来说：
```kotlin
val sample = Sample()
sample.long_id = 1
Sample.save(sample)
// 因为 id当前值为空字符串，所以 @GeneratedTsId 会为其生成新的 TSID
println(sample.id)  // => "2Q68ZVGXJCF9M"
// @GeneratedTsId 认为 long_id 已经赋值了合法的 TSID，不会生成新的 TSID
println(sample.long_id) // => 1
```

## TsIdFactory
如果想要自定义 TsId 的生成参数，就需要自己创建 TsIdFactory 对象：
```kotlin
val factory = TsIdFactory.create(
    nodeBits = 10,
    node = 42,
    // used to reset the counter when the millisecond changes.
    randomFunction = { ThreadLocalRandom.current().nextInt() },
    // an instant that represents the custom epoch. default to "2000-01-01T00:00:00.000Z"
    customEpoch = Instant.parse("2000-01-01T00:00:00.000Z"),
)
```
其中 `customEpoch` 参数决定了 TsId 中 time component 对应的 "0 时刻"，通过 "0 时刻" 和 TsId 中的 time component，就可以得知一个 TsId 具体的生成时刻：
```kotlin
val factory1 = TsIdFactory.create(
    customEpoch = Instant.parse("2000-01-01T00:00:00.000Z"),
)
val factory2 = TsIdFactory.create(
    customEpoch = Instant.parse("2010-01-01T00:00:00.000Z"),
)
val tsid = TsId.from(0)
val instant1 = factory1.getInstant(tsid)
val instant2 = factory2.getInstant(tsid)
println(instant1)  // => 2000-01-01T00:00:00.000Z
println(instant2)  // => 2010-01-01T00:00:00.000Z
```

## TsIds
TsIds 是单例的 TsIdFactory 对象，`@GeneratedTsId` 就是通过 `TsIds.create()` 来生成 TsId 的。应用中如果想要使用与 `@GeneratedTsId` 相同的参数来生成 TsId，可以直接调用 `TsIds.create()`。如果想要使用自己创建的 TsIdFactory，可以在程序启动时调用：
```kotlin
val factory = TsIdFactory.create(node = 42)
TsIds.setFactory(factory)
```

## TsIdCriteria
有时我们会想要查询创建时间在某个时间段的数据，为此我们会定义一个 `created_at` 列：
```kotlin
class Sample: Entity<String>() {
    companion object : EntityCompanion<String, Sample>() 
    @Column
    override var id: String = ""
    @Column
    override var created_at: Instant = Instant.now()

    data class Query(
        @Criteria.Gte
        val created_at_gte: Instant? = null,
    ) : CriteriaQuery<Sample>()
}
```
为了保证查询效率，我们可能还需要为 `created_at` 列定义索引。如果我们使用 TSID 作为主键的话，因为 TSID 中本身就带有生成时间信息，我们可以直接以 id 的生成时间范围作为查询条件，因为 id 是主键，不需要为此创建额外的索引：
```kotlin
class Sample: Entity<String>() {
    companion object : EntityCompanion<String, Sample>()
    @Column
    override var id: String = ""

    data class Query(
        @TsIdCriteria.TimeGte
        val id_time_gte: Instant? = null,
    ) : CriteriaQuery<Sample>()
}
```

!> 前提是表中生成的 TsId 和当前的 TsIds 有相同的 `customEpoch`

## Spring Boot Integration
`crud-kit-spring-boot-starter` 会读取配置，创建 TsIdFactory，并设置 TsIds 使用对应的 TsIdFactory，下面是一个简单的配置 demo：
```yaml
# in application.yml
crud-kit:
  ts-id:
    node-bits: 10
    node: 42
    custom-epoch: '2010-01-01T00:00:00.000Z'
```
