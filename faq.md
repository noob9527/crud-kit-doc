# FAQ

## 如何在日志中输出生成的 SQL 语句?
目前建议通过 datasource 代理的方式输出 SQL 日志, 可以考虑在 p6spy, datasource-proxy 中二选一．举例来说，如果要使用 p6spy，直接在项目中引入如下依赖即可
```groovy
implementation("com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.6.2")
```

参考:
- [p6spy](https://github.com/p6spy/p6spy)
- [datasource-proxy](https://github.com/ttddyy/datasource-proxy)
- [spring-boot-data-source-decorator](https://github.com/gavlyukovskiy/spring-boot-data-source-decorator)
