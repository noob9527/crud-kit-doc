# crud-kit 不同 DAO 实现的性能对比

如下结果大致反应了目前(crud version 0.0.25)不同 DAO 实现的性能差异
```
benchmarks summary:
Benchmark                                                  (size)   Mode  Cnt     Score      Error  Units
DaoSelectBenchmark1.mybatisSelect                             100  thrpt    3  1814.658 ± 1006.685  ops/s
DaoSelectBenchmark1.mybatisSelect                            1000  thrpt    3   284.625 ±  205.869  ops/s
DaoSelectBenchmark1.mybatisSelect                            5000  thrpt    3    70.280 ±   16.917  ops/s
DaoSelectBenchmark1.mybatisSelect                           10000  thrpt    3    36.536 ±    6.162  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect     100  thrpt    3   500.832 ±  186.166  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect    1000  thrpt    3    57.643 ±    9.260  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect    5000  thrpt    3    11.969 ±    1.004  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect   10000  thrpt    3     6.141 ±    0.975  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect        100  thrpt    3  2306.251 ±  385.795  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect       1000  thrpt    3   379.092 ±  121.754  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect       5000  thrpt    3    99.545 ±   47.884  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect      10000  thrpt    3    51.175 ±   21.522  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect           100  thrpt    3  1330.139 ±  608.778  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect          1000  thrpt    3   200.230 ±  148.556  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect          5000  thrpt    3    46.662 ±    3.039  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect         10000  thrpt    3    23.801 ±    2.408  ops/s
```
其中 size 指的是结果集条数，Spring JDBC 实现测试了三个不同的 `RowMapper`
- springJdbcBeanPropertyRowMapperSelect 是 Spring JDBC 原生的 BeanPropertyRowMapper
- springJdbcDedicatedRowMapperSelect 是根据测试用的 BenchSample Entity 类定制的 RowMapper
- springJdbcEntityRowMapperSelect 使用的是 crud-kit 目前默认使用的 RowMapper

### Conclusion
Spring JDBC + 定制 RowMapper 性能最好，但采用这个实现会大大影响 crud-kit 库的简便性，并且需要暴露 Spring JDBC 实现细节，因此我也不打算提供 API 让使用者定制 RowMapper．Spring JDBC + Spring JDBC 自带的 BeanPropertyRowMapper 性能最差，并且该实现目前(spring-jdbc 5.2.3.RELEASE)不支持 Java 8 的时间类型，且不支持属性命名以 "is" 开头．因此我们无法使用．最终 crud-kit 默认使用 Spring JDBC + `com.cpvsn.crud.jdbc.internal.EntityRowMapper` 实现．使用者可以根据具体 Entity 的使用场景考虑使用 Mybatis 实现．

### Appendix
测试使用的代码在 [`com.cpvsn.crud.DaoSelectBenchmark1`](crud-kit-test/src/benchmarks/kotlin/com/cpvsn/crud/DaoSelectBenchmark1.kt)，测试完整的控制台输出如下:
```
benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.mybatisSelect | size=100

Warm-up 1: 1119.856 ops/s
Warm-up 2: 1723.091 ops/s
Iteration 1: 1755.774 ops/s
Iteration 2: 1823.022 ops/s
Iteration 3: 1865.178 ops/s

1814.658 ±(99.9%) 1006.685 ops/s [Average]
  (min, avg, max) = (1755.774, 1814.658, 1865.178), stdev = 55.180
  CI (99.9%): [807.974, 2821.343] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.mybatisSelect | size=1000

Warm-up 1: 186.422 ops/s
Warm-up 2: 268.058 ops/s
Iteration 1: 273.068 ops/s
Iteration 2: 285.190 ops/s
Iteration 3: 295.616 ops/s

284.625 ±(99.9%) 205.869 ops/s [Average]
  (min, avg, max) = (273.068, 284.625, 295.616), stdev = 11.284
  CI (99.9%): [78.755, 490.494] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.mybatisSelect | size=5000

Warm-up 1: 55.790 ops/s
Warm-up 2: 68.780 ops/s
Iteration 1: 69.931 ops/s
Iteration 2: 69.578 ops/s
Iteration 3: 71.331 ops/s

70.280 ±(99.9%) 16.917 ops/s [Average]
  (min, avg, max) = (69.578, 70.280, 71.331), stdev = 0.927
  CI (99.9%): [53.363, 87.197] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.mybatisSelect | size=10000

Warm-up 1: 28.826 ops/s
Warm-up 2: 36.009 ops/s
Iteration 1: 36.353 ops/s
Iteration 2: 36.330 ops/s
Iteration 3: 36.926 ops/s

36.536 ±(99.9%) 6.162 ops/s [Average]
  (min, avg, max) = (36.330, 36.536, 36.926), stdev = 0.338
  CI (99.9%): [30.375, 42.698] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect | size=100

Warm-up 1: 330.736 ops/s
Warm-up 2: 467.919 ops/s
Iteration 1: 489.109 ops/s
Iteration 2: 507.723 ops/s
Iteration 3: 505.663 ops/s

500.832 ±(99.9%) 186.166 ops/s [Average]
  (min, avg, max) = (489.109, 500.832, 507.723), stdev = 10.204
  CI (99.9%): [314.665, 686.998] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect | size=1000

Warm-up 1: 44.189 ops/s
Warm-up 2: 56.097 ops/s
Iteration 1: 57.507 ops/s
Iteration 2: 57.217 ops/s
Iteration 3: 58.204 ops/s

57.643 ±(99.9%) 9.260 ops/s [Average]
  (min, avg, max) = (57.217, 57.643, 58.204), stdev = 0.508
  CI (99.9%): [48.383, 66.903] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect | size=5000

Warm-up 1: 8.977 ops/s
Warm-up 2: 11.617 ops/s
Iteration 1: 12.004 ops/s
Iteration 2: 11.905 ops/s
Iteration 3: 11.997 ops/s

11.969 ±(99.9%) 1.004 ops/s [Average]
  (min, avg, max) = (11.905, 11.969, 12.004), stdev = 0.055
  CI (99.9%): [10.964, 12.973] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect | size=10000

Warm-up 1: 4.479 ops/s
Warm-up 2: 5.982 ops/s
Iteration 1: 6.085 ops/s
Iteration 2: 6.192 ops/s
Iteration 3: 6.145 ops/s

6.141 ±(99.9%) 0.975 ops/s [Average]
  (min, avg, max) = (6.085, 6.141, 6.192), stdev = 0.053
  CI (99.9%): [5.165, 7.116] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect | size=100

Warm-up 1: 1635.516 ops/s
Warm-up 2: 2165.398 ops/s
Iteration 1: 2319.375 ops/s
Iteration 2: 2281.857 ops/s
Iteration 3: 2317.523 ops/s

2306.251 ±(99.9%) 385.795 ops/s [Average]
  (min, avg, max) = (2281.857, 2306.251, 2319.375), stdev = 21.147
  CI (99.9%): [1920.457, 2692.046] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect | size=1000

Warm-up 1: 309.530 ops/s
Warm-up 2: 355.393 ops/s
Iteration 1: 383.494 ops/s
Iteration 2: 371.414 ops/s
Iteration 3: 382.369 ops/s

379.092 ±(99.9%) 121.754 ops/s [Average]
  (min, avg, max) = (371.414, 379.092, 383.494), stdev = 6.674
  CI (99.9%): [257.338, 500.847] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect | size=5000

Warm-up 1: 75.738 ops/s
Warm-up 2: 94.747 ops/s
Iteration 1: 97.423 ops/s
Iteration 2: 98.730 ops/s
Iteration 3: 102.480 ops/s

99.545 ±(99.9%) 47.884 ops/s [Average]
  (min, avg, max) = (97.423, 99.545, 102.480), stdev = 2.625
  CI (99.9%): [51.660, 147.429] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect | size=10000

Warm-up 1: 40.873 ops/s
Warm-up 2: 49.615 ops/s
Iteration 1: 50.104 ops/s
Iteration 2: 50.982 ops/s
Iteration 3: 52.440 ops/s

51.175 ±(99.9%) 21.522 ops/s [Average]
  (min, avg, max) = (50.104, 51.175, 52.440), stdev = 1.180
  CI (99.9%): [29.653, 72.697] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcEntityRowMapperSelect | size=100

Warm-up 1: 1049.463 ops/s
Warm-up 2: 1328.365 ops/s
Iteration 1: 1357.630 ops/s
Iteration 2: 1293.013 ops/s
Iteration 3: 1339.775 ops/s

1330.139 ±(99.9%) 608.778 ops/s [Average]
  (min, avg, max) = (1293.013, 1330.139, 1357.630), stdev = 33.369
  CI (99.9%): [721.361, 1938.918] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcEntityRowMapperSelect | size=1000

Warm-up 1: 155.677 ops/s
Warm-up 2: 198.190 ops/s
Iteration 1: 190.893 ops/s
Iteration 2: 203.932 ops/s
Iteration 3: 205.863 ops/s

200.230 ±(99.9%) 148.556 ops/s [Average]
  (min, avg, max) = (190.893, 200.230, 205.863), stdev = 8.143
  CI (99.9%): [51.673, 348.786] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcEntityRowMapperSelect | size=5000

Warm-up 1: 37.437 ops/s
Warm-up 2: 45.973 ops/s
Iteration 1: 46.507 ops/s
Iteration 2: 46.641 ops/s
Iteration 3: 46.838 ops/s

46.662 ±(99.9%) 3.039 ops/s [Average]
  (min, avg, max) = (46.507, 46.662, 46.838), stdev = 0.167
  CI (99.9%): [43.623, 49.701] (assumes normal distribution)

benchmarks: com.cpvsn.crud.DaoSelectBenchmark1.springJdbcEntityRowMapperSelect | size=10000

Warm-up 1: 19.339 ops/s
Warm-up 2: 23.741 ops/s
Iteration 1: 23.951 ops/s
Iteration 2: 23.700 ops/s
Iteration 3: 23.753 ops/s

23.801 ±(99.9%) 2.408 ops/s [Average]
  (min, avg, max) = (23.700, 23.801, 23.951), stdev = 0.132
  CI (99.9%): [21.393, 26.210] (assumes normal distribution)


benchmarks summary:
Benchmark                                                  (size)   Mode  Cnt     Score      Error  Units
DaoSelectBenchmark1.mybatisSelect                             100  thrpt    3  1814.658 ± 1006.685  ops/s
DaoSelectBenchmark1.mybatisSelect                            1000  thrpt    3   284.625 ±  205.869  ops/s
DaoSelectBenchmark1.mybatisSelect                            5000  thrpt    3    70.280 ±   16.917  ops/s
DaoSelectBenchmark1.mybatisSelect                           10000  thrpt    3    36.536 ±    6.162  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect     100  thrpt    3   500.832 ±  186.166  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect    1000  thrpt    3    57.643 ±    9.260  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect    5000  thrpt    3    11.969 ±    1.004  ops/s
DaoSelectBenchmark1.springJdbcBeanPropertyRowMapperSelect   10000  thrpt    3     6.141 ±    0.975  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect        100  thrpt    3  2306.251 ±  385.795  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect       1000  thrpt    3   379.092 ±  121.754  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect       5000  thrpt    3    99.545 ±   47.884  ops/s
DaoSelectBenchmark1.springJdbcDedicatedRowMapperSelect      10000  thrpt    3    51.175 ±   21.522  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect           100  thrpt    3  1330.139 ±  608.778  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect          1000  thrpt    3   200.230 ±  148.556  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect          5000  thrpt    3    46.662 ±    3.039  ops/s
DaoSelectBenchmark1.springJdbcEntityRowMapperSelect         10000  thrpt    3    23.801 ±    2.408  ops/s
```
