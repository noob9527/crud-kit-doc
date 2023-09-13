# Mybatis Integration

## 引入依赖
### via maven
```xml
<dependency>
    <groupId>com.cpvsn</groupId>
    <artifactId>crud-kit-mybatis</artifactId>
    <version>${version}</version>
</dependency>
```

### via gradle
```groovy
dependencies {
    compile("com.cpvsn:crud-kit-mybatis:${version}")
    // ...
}
```

## 定义 mapper 和 repository
```kotlin
@Mapper
@Component
interface PostMapper : EntityAutoIncMapper<Int, Post>

@Repository
class PostRepository(
        override val dao: PostMapper
) : AbstractRepository<Int, Post>()
```
mapper 与 repository 都是可扩展的，mapper 和普通的 mybatis 项目一样，可以用 xml 或者 annotation 为 mapper 添加新的方法。

!> 以上用法仅适用于 Spring boot 项目且引用了，`crud-kit-spring-boot-starter` 依赖。

