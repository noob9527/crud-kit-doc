# Installation
### via maven
```xml
<!-- for non spring boot user -->
<dependency>
    <groupId>com.cpvsn</groupId>
    <artifactId>crud-kit</artifactId>
    <version>${version}</version>
</dependency>
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
    // for non spring boot user
    compile("com.cpvsn:crud-kit:${version}")
    // for spring boot user
    compile("com.cpvsn:crud-kit-spring-boot-starter:${version}")
    // ...
}
```
